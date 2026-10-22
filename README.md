# claude-routine

This repository is **not an application** — it is a Claude Code configuration
and demo repo. Its single purpose is one repeatable routine: append a
timestamped entry to `claude.log`, push that change to GitHub, and then file a
matching subtask in Zoho Projects under task `UNT-T48831`. It exists to
demonstrate — and keep re-verifying — that a Claude Code skill can drive MCP
connectors end to end: every run leaves two auditable traces, a greppable line
in `claude.log` and a Zoho ticket whose description links back to the exact
commit that carried it.

## What the routine does

The routine is one ordered pipeline. Each step depends on the previous one —
the Zoho subtask needs the commit link, and the commit link needs a successful
push — so the steps are never run out of order or on their own.

1. **Log.** Append a single line to `claude.log` at the repo root:
   `[YYYY-MM-DD HH:MM:SS TZ] <message>`. The timestamp comes from the system
   (`date "+%Y-%m-%d %H:%M:%S %Z"`), not from guesswork, and it is saved
   because step 4 reuses it. One entry per line keeps the log greppable.
2. **Commit and push via the GitHub MCP tool.** Call
   `mcp__github__push_files` with the full new contents of `claude.log` and a
   `chore(log): <message>` commit message. The SHA returned by that call is
   the authoritative commit id. `git commit` / `git push` are deliberately
   **not** used.
3. **Sync local to that exact SHA.** Run `git fetch origin <branch>`, then
   `git reset --hard origin/<branch>` **only** when `git status --porcelain`
   shows `claude.log` as the sole pending change. Any other uncommitted work
   means stopping and asking instead.
4. **Create the Zoho subtask.** Call `mcp__zoho-projects__createTask` with the
   pinned IDs from the table below — no discovery calls. The title carries the
   step-1 timestamp; the description carries the commit link, the timestamp
   and the log line.

### Why the GitHub MCP push instead of `git push`

An API push creates the commit on GitHub's side and does not transfer the local
commit object. So a local `git commit` followed by a separate API push produces
*two* commits with identical content but different SHAs (different committer
metadata), and local and remote history diverge. Treating `push_files` as the
source of truth and snapping the local branch onto its SHA in step 3 keeps one
commit, one SHA, and one link — the link that ends up in the Zoho ticket.

The full contract, including the exact tool arguments and the description
template, lives in
[`.claude/skills/log-commit-zoho/SKILL.md`](.claude/skills/log-commit-zoho/SKILL.md).

## Repository layout

| Path | Purpose |
| --- | --- |
| `README.md` | This document. |
| `claude.log` | Append-only log, one timestamped entry per line. |
| `_mcp.json` | MCP server definitions for the two connectors. |
| `.claude/settings.json` | Project permissions. |
| `.claude/skills/log-commit-zoho/SKILL.md` | The active routine. |
| `.claude/skills/commit-push-zoho-task.zip` | Earlier variant, reference. |
| `.gitignore` | Hides the unzipped older variant. |

All seven paths above are tracked by git; there are no untracked files in the
repo. Details:

- `claude.log` is committed empty and fills up one routine run at a time.
- `_mcp.json` declares the `github-new` and `zoho-projects` HTTP connectors
  (see [Prerequisites & setup](#prerequisites--setup)).
- `.claude/settings.json` pre-approves the `mcp__zoho-projects` connector.
- `.claude/skills/log-commit-zoho/SKILL.md` is 206 lines: four ordered steps,
  pinned Zoho IDs, and the trigger phrases in its frontmatter. It is the
  source of truth for the routine.
- `.claude/skills/commit-push-zoho-task.zip` is the archive of the earlier
  generic variant. It was produced by the macOS archiver and also contains
  inert `__MACOSX/._*` metadata entries; they carry no instructions.
- `.gitignore` ignores `.claude/skills/commit-push-zoho-task/`, so the
  *unzipped* older variant is never committed — only its zip is tracked.

## Prerequisites & setup

### Tooling

- `git`, and a shell in which the routine can run `git` and `date`.
- A Claude Code client with MCP support, configured to see the two servers
  below.
- Network access to `api.githubcopilot.com` (GitHub MCP) and to the Zoho MCP
  host.
- A GitHub token with **contents write** access to
  `krunal-simform/claude-routine`, and Zoho Projects access to portal
  `36485097`.

### Environment variables

Two variables, no more. They are referenced by name only — no values live in
this repo.

| Variable | Used by | What it is |
| --- | --- | --- |
| `GITHUB_PAT` | `github-new` | GitHub PAT, contents-write |
| `ZOHO_API` | `zoho-projects` | Zoho MCP endpoint token |

Inside `_mcp.json` they appear only as a header value, `Bearer ${GITHUB_PAT}`,
and as a URL path segment, `.../mcp/${ZOHO_API}/message`.

Export them in the environment your client runs in, using placeholders — never
real values in a tracked file:

```bash
export GITHUB_PAT="<your-github-pat>"
export ZOHO_API="<your-zoho-mcp-token>"
```

### MCP wiring

`_mcp.json` is the reference copy of the two server definitions:

```json
{
  "mcpServers": {
    "github-new": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_PAT}"
      }
    },
    "zoho-projects": {
      "type": "http",
      "url": "https://projects-task-management-929867360.zohomcp.com/mcp/${ZOHO_API}/message"
    }
  }
}
```

Both entries must be registered in your MCP client's own configuration. If
your client reads a different settings file, copy these two server entries
across verbatim — names included, because `github-new` and `zoho-projects` are
what the `mcp__<server>__<tool>` tool names are derived from.

### Clone

```bash
git clone https://github.com/krunal-simform/claude-routine.git
cd claude-routine
```

### Permissions

`.claude/settings.json` pre-approves the `mcp__zoho-projects` connector only.
The `github-new` connector is *not* in that allow-list, so the first GitHub MCP
call of a session may raise a permission prompt — approve it rather than
working around it.

## How to run it

The skill is invoked explicitly: its frontmatter sets
`disable-model-invocation: true`, so it is never auto-selected. Ask for it by
name (`/log-commit-zoho`) or with any of the trigger phrases from its
`description`:

- "run the routine"
- "log and commit"
- "log a change and create a Zoho task/subtask"
- "commit and make the Zoho ticket"

Direct Zoho asks about the parent task are covered too, since the skill already
knows its IDs: "show me the task", "list its subtasks", "add a subtask to
UNT-T48831".

Triggering on *part* of the flow still runs the whole pipeline — "commit and
make the Zoho ticket" logs, pushes, syncs and files the subtask, because the
steps are designed to run together. If no log message is supplied, the skill
asks for a one-line summary (or infers one and confirms it) before touching
anything.

A finished run reports the log line added, the commit link, and the id or URL
of the created Zoho subtask.

### Two skill variants

- **`.claude/skills/log-commit-zoho/SKILL.md` — current.** Pinned four-step
  pipeline with hard-coded Zoho IDs and a GitHub-MCP push. This is the routine.
- **`.claude/skills/commit-push-zoho-task.zip` — reference only.** Earlier
  generic variant, archived as a zip. It is parameterized by `project` /
  `task-list` / optional `parent-task` arguments, discovers IDs at run time
  (`get_portals`, `get_projects_list`, `get_all_project_task_lists`), commits
  with `git commit` + `git push`, and calls older tool names such as
  `create_a_task`; its title and description templates live in
  `references/zoho-task.md` inside the zip.

Only the zip of the older variant survives, and Claude Code does not load a
zipped skill. Nothing in this README depends on it; it is history, not the
active routine.

## Pinned Zoho identifiers

Step 4 uses these values verbatim. No portal, project or task lookup happens at
run time.

| Field | Value |
| --- | --- |
| Portal ID | `36485097` |
| Project ID | `688906000001071159` (`PR-46`, "Unassigned Tasks") |
| Parent task ID | `688906000092147305` (`UNT-T48831`) |
| Parent task list ID | `688906000006396141` |
| Owner (Krunal Patel) | zuid `802509144` / zpuid `688906000048662473` |

The parent task displays as `UNT-T48831`, "Cluade test tasks"; the task list is
"iOS - Ajay Ghodadra's Team". The IDs are passed straight in as `path_variables`
(`portal_id`, `project_id`, `task_id`) for any `mcp__zoho-projects__*` call
about this task. `getAllPortals` / `getAllProjects` / `getTasksByPortal` are
intentionally not called to rediscover them. If this table and `SKILL.md` ever
disagree, `SKILL.md` wins — it is what the routine actually executes.

## Limitations & troubleshooting

The routine stops and reports rather than guessing. That is the design, not a
rough edge.

- **"Add a comment to the task."** The `zoho-projects` connector exposes no
  comment-on-task tool at all. The skill says so instead of improvising; in
  particular it never substitutes `updateTask` or edits the description to
  fake a comment.
- **A pinned ID returns 404 / not-found.** The task may have been moved or
  deleted. Stop and report it; do not silently re-discover a "similar" task.
  The identifiers are pinned on purpose.
- **The GitHub push fails** (no `origin` remote, branch protection, expired or
  under-scoped `GITHUB_PAT`, connector not approved). Stop. A commit URL is
  never fabricated, because the Zoho subtask must link to a commit that really
  exists.
- **Unrelated uncommitted changes at step 3.** `git reset --hard` would throw
  them away, so the routine stops and asks instead of resetting.
- **`claude.log` is behind or in conflict locally.** The fix is exactly what
  step 3 does — fetch and reset to `origin/<branch>`. Run it by hand once your
  other work is committed or stashed, and only when `claude.log` is the sole
  pending change.
- **Task status can't be set on create.** `createTask` applies the project's
  default open status. A different status needs a follow-up `updateTask` with
  the task id that `createTask` returned.
- **Nothing appears in Zoho.** Check that the subtask's `parent_task_id` is
  populated and that the run reported a task id. A push that succeeded without
  a task is a half-finished pipeline and should be reported as such.

## Contributing

Contributions are welcome — this is a small, docs-and-config repo, so the loop
is short.

### Workflow

1. Branch from `master`: `git switch -c feat/my-change master`.
2. Make the change, keeping the diff scoped to one concern.
3. Open a pull request against `krunal-simform/claude-routine` with base
   `master`.
4. In the PR description, say **what changed and how it was verified**. For a
   docs change: which render or lint check you ran. For a routine change: the
   dry-run transcript, or the log line, commit link and Zoho task id from a
   real run.

### Commit messages

Follow [Conventional Commits](https://www.conventionalcommits.org/): `feat:`,
`fix:`, `chore:`, `docs:`. That matches this repo's history, where
`git log --oneline` shows `feat:` and `fix:` subjects, one of them with a body
and a `Co-Authored-By:` trailer. Keep the subject under ~72 characters.

### Changing the routine

- Edit `.claude/skills/log-commit-zoho/SKILL.md` — it is the tracked source of
  truth; there is no other copy to keep in sync.
- Keep the four steps in execution order (log, push via GitHub MCP, sync local,
  Zoho subtask). Reordering breaks the dependency chain: step 4 needs the
  commit link from step 2, and step 2 needs the timestamp from step 1.
- Keep the pinned Zoho identifiers intact. Changing one is an owner-sign-off
  change (see below), not a drive-by edit.
- Re-check the frontmatter. `name`, `description` (where the trigger phrases
  live) and `allowed-tools` must still name real tools on the `zoho-projects`
  connector — for example `mcp__zoho-projects__createTask` and
  `mcp__zoho-projects__updateTask`. The read tools the prose relies on
  (`getTaskDetails`, `getTasksByProject`) must still exist under those names,
  even though they are not in the allow-list.
- The allow-list is broader than what the routine uses: it also carries
  `createProject`, `trashProject`, `createTaskList`, `createPhase`,
  `createProjectIssue` and friends. Trimming it is a deliberate,
  separately-reviewed change, not incidental cleanup.
- Leave `disable-model-invocation: true` alone unless you mean to make the
  skill auto-selectable.

### Secrets and generated files

`GITHUB_PAT`, `ZOHO_API` and any other token stay as `${...}` references in
`_mcp.json`. Never commit real values, tokens, or credential-bearing URLs — not
in code, docs, commit messages or `claude.log`.

`.claude/skills/commit-push-zoho-task/` is gitignored on purpose; only the zip
of that variant is tracked. Don't unzip it into the repo, and leave the zip
itself alone unless you are intentionally replacing it.

### Reviews and ownership

Open the PR and ask `krunal-simform` (the repo owner) for review. Call out
explicitly any change that touches the pinned Zoho identifiers or `_mcp.json`:
those need owner sign-off. Verifying a routine change end to end also requires
Zoho Projects access to portal `36485097` — if you don't have it, say in the PR
how far you got (for example steps 1–3 verified, step 4 unverified) so a
reviewer with access can finish the check.

Questions, or unsure whether a change is in scope? Open an issue on
`krunal-simform/claude-routine`, or ask the owner directly in the PR.
