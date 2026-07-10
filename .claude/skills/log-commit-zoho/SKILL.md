---
name: log-commit-zoho
description: >-
  Runs the demo "log → commit → push → Zoho subtask" routine in one shot. Use
  this whenever the user asks to "run the routine", "log and commit", "log a
  change and create a Zoho task/subtask", or otherwise wants to append an entry
  to claude.log, commit and push it to GitHub, and then open a subtask under
  Zoho task UNT-T48831. Also use for any direct Zoho ask about UNT-T48831
  itself — "show me the task", "list its subtasks", "add a subtask/comment to
  UNT-T48831" — since this skill already has the IDs needed to hit it without
  searching. Trigger even when the user only mentions part of the flow (e.g.
  "commit and make the Zoho ticket") — the steps are meant to run together as
  a single pipeline.
---

# Log, Commit, Push & Zoho Subtask

This skill runs a fixed four-step routine. Do the steps **in order** — each step
depends on the one before it (the Zoho subtask needs the commit link, and the
commit link needs a successful push).

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
subtasks, add a comment, or create a new subtask. Do not call
`get_portals` / `get_projects_list` / `get_tasks_by_*` to rediscover them.

Useful direct calls against the parent task:

- **Get the task itself**: `ZohoProjects_get_task_details` with the IDs above.
- **List its subtasks**: `ZohoProjects_get_tasks_by_project` filtered to
  `parental_info.parent_task_id = 688906000092147305`, or just call
  `get_task_details` and check `association_info.has_subtasks`, then
  `get_tasks_by_project` sorted by `created_time` and match `parental_info`.
- **Add a comment**: `ZohoProjects_add_task_comment` with `task_id
  688906000092147305`.
- **Create a new subtask**: `ZohoProjects_create_a_task` with `project_id
  688906000001071159` and `body.parental_info.parent_task_id =
  "688906000092147305"` (this is what Step 4 below does).

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

Call `mcp__zoho-projects__ZohoProjects_create_a_task` directly with the fixed
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
  call `ZohoProjects_update_a_task` (same `task_id` returned by create) with
  the desired `status.id`. Use `ZohoProjects_get_task_details` on the parent
  task to see the project's status options if the exact "In Progress" id is
  needed, or match the closest in-progress status and note the substitution.

## Finish

Report back a short summary: the log line added, the commit link, and the URL or
ID of the created Zoho subtask so the user can click through.
