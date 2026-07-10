---
name: log-commit-zoho
description: >-
  Runs the demo "log → commit → push → Zoho subtask" routine in one shot. Use
  this whenever the user asks to "run the routine", "log and commit", "log a
  change and create a Zoho task/subtask", or otherwise wants to append an entry
  to claude.log, commit and push it to GitHub, and then open a subtask under
  Zoho task UNT-T48831. Trigger even when the user only mentions part of the
  flow (e.g. "commit and make the Zoho ticket") — the steps are meant to run
  together as a single pipeline.
---

# Log, Commit, Push & Zoho Subtask

This skill runs a fixed four-step routine. Do the steps **in order** — each step
depends on the one before it (the Zoho subtask needs the commit link, and the
commit link needs a successful push).

## Inputs

- **Log message**: what to record in `claude.log`. If the user didn't give one,
  ask for a short one-line summary of what was done, or infer it from the
  current change/context and confirm.
- Everything else (the parent task `UNT-T48831`, owner, status) is fixed below.

## Step 1 — Append to `claude.log`

Append a single timestamped line to `claude.log` at the repo root. Create the
file if it does not exist. Keep one entry per line so the log stays greppable.

Format each entry as:

```
[YYYY-MM-DD HH:MM:SS TZ] <log message>
```

Get the timestamp from the system (`date "+%Y-%m-%d %H:%M:%S %Z"`) rather than
guessing, and **save this exact timestamp** — Step 4 reuses it in the Zoho task
title and description so the log entry and the ticket line up.

## Step 2 — Commit

Stage `claude.log` and commit it. Use a clear conventional-style message, e.g.:

```
chore(log): <log message>
```

If the repo has no commits yet, this first commit is fine. Do not amend or force
anything.

## Step 3 — Push to GitHub

Push the current branch to the `origin` remote:

```bash
git push -u origin HEAD
```

Then capture the details Step 4 needs:

- **Commit SHA**: `git rev-parse HEAD`
- **Remote URL**: `git remote get-url origin`
- **Commit link**: build the GitHub URL from the remote and SHA, i.e.
  `https://github.com/<owner>/<repo>/commit/<SHA>` (normalize `git@github.com:`
  / trailing `.git` remotes into the `https://github.com/...` form).

If push fails because there is no `origin` remote or no upstream, stop and tell
the user what's missing rather than fabricating a link — the Zoho subtask must
contain a real commit URL.

## Step 4 — Create the Zoho subtask

Use the **Zoho Projects MCP** to create a subtask under the parent task. If the
Zoho tools aren't available yet, call
`mcp__claude_ai_Zoho_Projects__authenticate` first, share the URL with the user,
and finish auth before continuing.

Create the subtask with these exact fields:

| Field       | Value                                                                 |
| ----------- | --------------------------------------------------------------------- |
| Parent task | `UNT-T48831`                                                          |
| Title       | `Claude routine — <YYYY-MM-DD HH:MM:SS TZ>` (the Step 1 timestamp)    |
| Owner       | `Krunal Patel`                                                        |
| Status      | `In Progress`                                                         |
| Description | The commit link **and** the timestamp (see template below)            |

Description template:

```
Automated routine run.
Commit: <commit link>
Timestamp: <YYYY-MM-DD HH:MM:SS TZ>
Log entry: <log message>
```

Notes for reliability:

- The title **must contain the date and time** — that's how the demo
  distinguishes runs, so never drop it.
- Match the owner to the Zoho user "Krunal Patel"; if the API needs a user/owner
  ID rather than a name, look it up from the project's users first.
- Status must resolve to the project's "In Progress" status. If the exact label
  differs, pick the closest in-progress status and note the substitution.

## Finish

Report back a short summary: the log line added, the commit link, and the URL or
ID of the created Zoho subtask so the user can click through.
