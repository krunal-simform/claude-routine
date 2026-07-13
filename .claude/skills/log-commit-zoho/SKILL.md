---
name: log-commit-zoho
description: >-
  Runs the demo "log → commit → push → Zoho subtask" routine in one shot. Use
  this whenever the user asks to "run the routine", "log and commit", "log a
  change and create a Zoho task/subtask", or otherwise wants to append an entry
  to claude.log, commit and push it to GitHub, and then open a subtask under
  Zoho task UNT-T48831. Also use for any direct Zoho ask about UNT-T48831
  itself — "show me the task", "list its subtasks", "add a subtask to
  UNT-T48831" — since this skill already has the IDs needed to hit it without
  searching. Trigger even when the user only mentions part of the flow (e.g.
  "commit and make the Zoho ticket") — the steps are meant to run together as
  a single pipeline.
allowed-tools:
  - mcp__zoho-projects__createProject
  - mcp__zoho-projects__updateProject
  - mcp__zoho-projects__trashProject
  - mcp__zoho-projects__restoreProject
  - mcp__zoho-projects__createTaskList
  - mcp__zoho-projects__createDefaultTasklist
  - mcp__zoho-projects__updateTaskList
  - mcp__zoho-projects__createTask
  - mcp__zoho-projects__updateTask
  - mcp__zoho-projects__createPhase
  - mcp__zoho-projects__updatePhase
  - mcp__zoho-projects__createProjectIssue
  - mcp__zoho-projects__updateIssue
disable-model-invocation: true
---

# Log, Commit, Push & Zoho Subtask

This skill runs a fixed routine. Do the steps **in order** — each step depends
on the one before it (the Zoho subtask needs the commit link, and the commit
link needs a successful push).

**Push uses the GitHub MCP tools (`mcp__github__push_files`), not the `git
push` CLI.** The GitHub MCP push creates the commit directly on GitHub via the
API — it does not transfer your local commit object. That means if you commit
locally with `git commit` *and separately* push via the API, you end up with
two different commits with the same content but different SHAs (different
committer metadata), and local/remote history diverges. To avoid that, treat
the GitHub-side commit created by `push_files` as the single source of truth,
then sync your local branch to point at that exact SHA (Step 3) instead of
computing/using a separately-made local commit SHA.

## Fixed Zoho identifiers — no lookup needed

The parent task `UNT-T48831` lives in the Zoho Projects **simform** portal.
These IDs are resolved and stable — use them directly with the
`mcp__zoho-projects__*` tools instead of searching/listing projects or tasks:

| Field                | Value                                                  |
| -------------------- | ------------------------------------------------------- |
| Portal ID            | `36485097`                                             |
| Project ID           | `688906000001071159` (project key `PR-46`, "Unassigned Tasks") |
| Parent task ID       | `688906000092147305` (display key `UNT-T48831`, "Cluade test tasks") |
| Parent task list ID  | `688906000006396141` ("iOS - Ajay Ghodadra's Team")     |
| Owner (Krunal Patel) | zuid `802509144` / zpuid `688906000048662473`           |

These go straight into `path_variables` (`portal_id`, `project_id`, `task_id`)
for any `mcp__zoho-projects__*` call about this task — get details, list
subtasks, or create a new subtask. Do not call `getAllPortals` /
`getAllProjects` / `getTasksByPortal` to rediscover them.

Useful direct calls against the parent task:

- **Get the task itself**: `getTaskDetails` with the IDs above.
- **List its subtasks**: `getTasksByProject` filtered to
  `parental_info.parent_task_id = 688906000092147305`, or just call
  `getTaskDetails` and check `association_info.has_subtasks`, then
  `getTasksByProject` sorted by `created_time` and match `parental_info`.
- **Create a new subtask**: `createTask` with `project_id
  688906000001071159` and `body.parental_info.parent_task_id =
  "688906000092147305"` (this is what Step 4 below does).

Note: this connector has no comment-on-task tool (no `addComment`-style
call in the tool list). If the user asks to "add a comment" to
UNT-T48831, say so rather than improvising a call — don't substitute
`updateTask` or another tool to fake a comment.

If any of these IDs ever come back 404/not-found (e.g. the task was moved or
deleted), stop and tell the user rather than silently searching for a
replacement — the identifiers here are pinned intentionally.

## Inputs

- **Log message**: what to record in `claude.log`. If the user didn't give one,
  ask for a short one-line summary of what was done, or infer it from the
  current change/context and confirm.
- Everything else (the parent task, owner, status) is fixed above.

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

## Step 2 — Commit & push via GitHub MCP

Do **not** run `git commit` / `git push` for this. Instead push `claude.log`
straight to GitHub with `mcp__github__push_files`, which creates the commit
and pushes it in one API call:

1. Get `owner`/`repo` from `git remote get-url origin` (strip `git@github.com:`
   / `https://github.com/` prefixes and any trailing `.git`).
2. Get the target branch with `git branch --show-current`.
3. Call `mcp__github__push_files` with:
   - `owner`, `repo`, `branch` from above
   - `files`: `[{ path: "claude.log", content: "<full contents of claude.log after Step 1's append>" }]`
   - `message`: a clear conventional-style message, e.g. `chore(log): <log message>`
4. From the tool's response, record the **commit SHA it created** — this is
   the authoritative commit id. Do not compute a SHA from local git state; the
   local repo has not been updated yet at this point.

If the call fails (no `origin` remote, branch protection, etc.), stop and tell
the user what's missing rather than fabricating a commit — the Zoho subtask
must contain a real commit URL.

## Step 3 — Sync the local repo to that exact commit

The local working tree is still behind (and may still have the Step 1 edit as
an uncommitted change). Bring local in line with the commit GitHub just
created, so the local HEAD SHA and the remote SHA are identical rather than
two different commits with the same content:

```bash
git fetch origin <branch>
```

Then check `git status --porcelain`. If the only pending change is
`claude.log` (the one this routine just pushed), fast-forward local to match:

```bash
git reset --hard origin/<branch>
```

If there are *other* uncommitted local changes unrelated to this routine,
stop and ask the user how to proceed instead of resetting — don't silently
discard their work.

Capture the details Step 4 needs:

- **Commit SHA**: the SHA returned by `push_files` in Step 2 (equivalently,
  `git rev-parse HEAD` after the sync above — they must match).
- **Commit link**: `https://github.com/<owner>/<repo>/commit/<SHA>`.

## Step 4 — Create the Zoho subtask

Call `mcp__zoho-projects__createTask` directly with the fixed
IDs from the table above — no auth/search step required first (the
`mcp__zoho-projects__*` tools are already connected). Use:

```
path_variables:
  portal_id: "36485097"
  project_id: "688906000001071159"

body:
  name: "Claude routine — <YYYY-MM-DD HH:MM:SS TZ>"   # the Step 1 timestamp
  description: "<see template below>"
  parental_info:
    parent_task_id: "688906000092147305"
  tasklist:
    id: "688906000006396141"
  owners_and_work:
    owners:
      - zpuid: "688906000048662473"
  status: (see notes below)
```

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
- Owner is always Krunal Patel (zpuid `688906000048662473`) — do not look this
  up again.
- `status` is not a free field on create; if you need it set to something
  other than the project's default open status, create the task first, then
  call `updateTask` (same `task_id` returned by create) with
  the desired `status.id`. Use `getTaskDetails` on the parent
  task to see the project's status options if the exact "In Progress" id is
  needed, or match the closest in-progress status and note the substitution.

## Finish

Report back a short summary: the log line added, the commit link, and the URL or
ID of the created Zoho subtask so the user can click through.
