# claude-routine

A Claude Code demo that runs one fixed routine end to end: append a timestamped
line to [`claude.log`](claude.log), push it to GitHub through the GitHub MCP
server, and open a matching Zoho Projects subtask under `UNT-T48831`.

> Originally described as: *"Claude routine calls — after every routine call
> the claude will add log to claude.log file."*

<!-- TODO: add build/workflow, latest-release and license badges once this repo
has CI, a tagged release and a LICENSE file. None of those exist today, so no
badge is shown. -->

## Features

- **One fixed pipeline** — once started, the skill performs
  log → push → Zoho subtask in order. You start it deliberately (see
  [Usage](#usage)): it declares `disable-model-invocation: true`, so Claude does
  not begin it on its own. The steps are defined in
  [`SKILL.md`](.claude/skills/log-commit-zoho/SKILL.md).
- **Greppable append-only log** — exactly one
  `[YYYY-MM-DD HH:MM:SS TZ] <message>` line per run in `claude.log`.
- **Push through the GitHub API, not `git push`** — `mcp__github__push_files`
  creates the commit on GitHub and the local branch is then synced to that
  exact SHA, so you never end up with two same-content commits with different
  SHAs.
- **Traceable tickets** — the Zoho subtask title carries the same timestamp as
  the log line, and its description links the real commit URL.
- **No lookups per run** — portal, project, parent task, task list and owner
  are pinned in the skill, so the routine never spends calls rediscovering
  them.
- **Honest failure modes** — a failed push or a missing Zoho ID stops the
  routine with a report instead of fabricating a commit or a ticket.

## What this repository contains

| Path | Purpose |
| --- | --- |
| [`claude.log`](claude.log) | Append-only run log. Empty at rest; one line per routine run. |
| [`.claude/skills/log-commit-zoho/SKILL.md`](.claude/skills/log-commit-zoho/SKILL.md) | The routine itself. Source of truth for every step, ID and command below. |
| [`.claude/settings.json`](.claude/settings.json) | Pre-authorises the `mcp__zoho-projects` MCP server for this project. |
| [`_mcp.json`](_mcp.json) | Definitions of the two HTTP MCP servers used: `github-new` and `zoho-projects`. |
| [`.claude/skills/commit-push-zoho-task.zip`](.claude/skills/commit-push-zoho-task.zip) | Packaged copy of an **earlier, superseded** skill (`commit-push-zoho-task` plus `references/zoho-task.md`). Reference only — the live skill is `log-commit-zoho`. |
| [`.gitignore`](.gitignore) | Ignores an unpacked `.claude/skills/commit-push-zoho-task/` directory. |

## How the routine works

The four steps run in order — the Zoho subtask needs the commit link, and the
commit link needs a successful push.

| Step | Action |
| --- | --- |
| 1 | Append `[YYYY-MM-DD HH:MM:SS TZ] <message>` to `claude.log` (create it if missing). The timestamp comes from the system, not from a guess, and is reused in step 4. |
| 2 | Push `claude.log` with `mcp__github__push_files` (the name `SKILL.md` uses — see the [naming caveat](#tool-naming-mcp__github__push_files)) — `owner`/`repo` parsed from `git remote get-url origin`, branch from `git branch --show-current`, message like `chore(log): <message>`. Keep the commit SHA the API returns. |
| 3 | `git fetch origin <branch>`, confirm `git status --porcelain` shows nothing but `claude.log`, then `git reset --hard origin/<branch>` so local `HEAD` equals the SHA from step 2. |
| 4 | Create the Zoho subtask with `mcp__zoho-projects__createTask` under the pinned parent task, titled `Claude routine — <YYYY-MM-DD HH:MM:SS TZ>`, with the commit link in the description. |

**Why the API instead of `git push`:** `push_files` creates the commit
server-side, so a locally made commit and the API commit would be two different
SHAs with identical content. Treating the GitHub commit as the single source of
truth — and syncing local to it in step 3 — keeps history identical on both
sides.

### Tool naming: `mcp__github__push_files`

`SKILL.md` refers to the push tool as `mcp__github__push_files`, but
[`_mcp.json`](_mcp.json) names that server **`github-new`**. Claude Code
addresses MCP tools as `mcp__<server>__<tool>`, so the tool probably surfaces as
`mcp__github-new__push_files` and the name in `SKILL.md` may not resolve. Check
your session's MCP tool list and use the name it actually shows.

<!-- TODO: reconcile the server name in _mcp.json (github-new) with the tool name
SKILL.md calls (mcp__github__push_files) — either rename the server or update
SKILL.md. Also raise as a follow-up: mcp__github__push_files is not in the
allowed-tools frontmatter at all, and step 2 depends on it. -->

## Prerequisites

- **Claude Code**, able to load project skills from `.claude/skills/`.
  <!-- TODO: state the minimum Claude Code version tested. -->
- **A GitHub MCP server** exposing a `push_files` tool. `SKILL.md` addresses it
  as `mcp__github__push_files`, while `_mcp.json` names that server `github-new`
  — see the [tool-naming note](#tool-naming-mcp__github__push_files) above.
- **A Zoho Projects MCP connector** providing the `mcp__zoho-projects__*` tools
  the routine uses. Those split in two groups:
  - **Listed in `allowed-tools`** (13 tools): `createProject`,
    `updateProject`, `trashProject`, `restoreProject`, `createTaskList`,
    `createDefaultTasklist`, `updateTaskList`, `createTask`, `updateTask`,
    `createPhase`, `updatePhase`, `createProjectIssue`, `updateIssue`.
  - **Used in the skill body but not listed in `allowed-tools`**:
    `getTaskDetails` and `getTasksByProject` (the read-only calls to inspect
    `UNT-T48831` and its subtasks).
- **A git clone of this repository** with an `origin` remote and a checked-out
  branch — steps 2 and 3 read both.
- **Credentials** for the two MCP servers: `GITHUB_PAT` and `ZOHO_API` (see
  [Configuration](#configuration)).

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/krunal-simform/claude-routine.git
cd claude-routine
```

### 2. Export the credentials the MCP servers expand

`_mcp.json` references the variables `${GITHUB_PAT}` and `${ZOHO_API}`; export
them in the environment that starts Claude Code. Never commit real values.

```bash
export GITHUB_PAT="<your-github-token>"
export ZOHO_API="<your-zoho-mcp-token>"
```

### 3. Make the MCP server definitions available to Claude Code

The two HTTP server definitions ship in [`_mcp.json`](_mcp.json). They must be
loaded as project MCP configuration for the GitHub and Zoho Projects tools to
exist in the session (`mcp__github-new__*` / `mcp__zoho-projects__*` by the
naming in that file — see [the caveat above](#tool-naming-mcp__github__push_files)).

> [!NOTE]
> The exact loading behaviour of the `_mcp.json` filename is unconfirmed —
> check how your Claude Code install expects project MCP config to be named and
> placed before relying on the routine.
> <!-- TODO: confirm how _mcp.json is loaded (Claude Code conventionally reads
> .mcp.json at the repo root); then either rename the file or document the
> copy/registration step here. -->

### 4. Start the routine explicitly

The skill carries `disable-model-invocation: true`, so **Claude will not start
it from its `description`** — you invoke it yourself. Invoke it by its `name`
field, `log-commit-zoho` (in Claude Code, `/log-commit-zoho`), and then give it
the one-line log message when it asks; `SKILL.md` says to ask for a short
summary when none was supplied.

```text
/log-commit-zoho
→ when prompted for the log message: smoke test: README walkthrough
```

A run that starts correctly appends one line to [`claude.log`](claude.log), so
an unchanged empty `claude.log` after your attempt means the skill never ran.

> [!NOTE]
> If your Claude Code build exposes the skill as a `/slash` command, the exact
> argument syntax for passing the message on the same line is unconfirmed.
> <!-- TODO: confirm the invocation form (bare `/log-commit-zoho` then answer the
> prompt, vs. `/log-commit-zoho <message>`) on a real install, and whether the
> message is passed as an argument. -->

## Configuration

### Environment variables

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `GITHUB_PAT` | string (secret) | Yes | none | Token sent as `Authorization: Bearer` to the GitHub MCP server; must be allowed to push to this repository. |
| `ZOHO_API` | string (secret) | Yes | none | Path segment of the Zoho Projects MCP endpoint URL (`.../mcp/${ZOHO_API}/message`). |

No `.env` file is committed and no values belong in the repository —
`_mcp.json` only contains the `${...}` references.

### Pinned Zoho identifiers

The routine targets one fixed place in Zoho Projects. The identifiers are
hard-coded in [`SKILL.md`](.claude/skills/log-commit-zoho/SKILL.md) so that no
search calls are needed:

| Field | Value |
| --- | --- |
| Portal | `simform` |
| Project | `PR-46` — "Unassigned Tasks" |
| Parent task | `UNT-T48831` — "Cluade test tasks" |
| Parent task list | "iOS - Ajay Ghodadra's Team" |
| Owner | Krunal Patel |

The numeric portal / project / task / task-list / zpuid values are in the
*Fixed Zoho identifiers* table inside `SKILL.md` — edit them there, not here,
and keep the two in sync if the target task moves.

## Usage

### Start the routine

Because of `disable-model-invocation: true`, the routine does **not** start from
a plain-language request — invoke the skill yourself, by its `name`
(`log-commit-zoho`, exposed in Claude Code as `/log-commit-zoho`), then supply
the log message.

Once it is running, these are the kinds of instructions it handles — the
`description` in `SKILL.md` lists them as what the skill covers, not as
auto-triggers:

```text
run the routine
log and commit: <one-line summary of what changed>
add a subtask to UNT-T48831
show me the task UNT-T48831
```

### Commands the routine runs

All of these come from `SKILL.md`:

```bash
date "+%Y-%m-%d %H:%M:%S %Z"     # step 1: log line and ticket title
git remote get-url origin         # step 2: derive owner/repo for push_files
git branch --show-current         # step 2: target branch
git status --porcelain            # step 3: must show only claude.log
git fetch origin <branch>         # step 3
git reset --hard origin/<branch>  # step 3: local HEAD := the SHA GitHub made
git rev-parse HEAD                # step 3: must equal that SHA
```

### What a run produces

| Artefact | Shape |
| --- | --- |
| Log line | `[YYYY-MM-DD HH:MM:SS TZ] <message>` appended to `claude.log` |
| Commit | `chore(log): <message>`, link `https://github.com/<owner>/<repo>/commit/<SHA>` |
| Zoho subtask | Title `Claude routine — <YYYY-MM-DD HH:MM:SS TZ>`, parent `UNT-T48831`, owner Krunal Patel |

Subtask description template:

```text
Automated routine run.
Commit: <commit link>
Timestamp: <YYYY-MM-DD HH:MM:SS TZ>
Log entry: <log message>
```

## Project structure

```text
claude-routine/
├── .claude/
│   ├── settings.json                     # pre-authorises the Zoho MCP server
│   └── skills/
│       ├── log-commit-zoho/
│       │   └── SKILL.md                  # the routine (source of truth)
│       └── commit-push-zoho-task.zip     # packaged copy of the earlier skill
├── .gitignore
├── claude.log                            # append-only run log (empty at rest)
├── _mcp.json                             # MCP server definitions
└── README.md
```

## Development

There is **no build step, no test suite, no linter and no CI** in this
repository — it is a skill definition plus a log file, so verification is a
manual run.

Manual check after editing `SKILL.md` or the MCP config:

1. Run the routine once with an obvious message.
2. `claude.log` gained exactly one new line, prefixed with a
   `[YYYY-MM-DD HH:MM:SS TZ]` timestamp.
3. The commit exists on GitHub at the reported SHA (open the commit link).
4. `git rev-parse HEAD` equals that SHA — local and remote agree.
5. The subtask appears under `UNT-T48831` with the timestamped title and the
   commit link.

When editing the skill:

- Keep the frontmatter keys intact — `name`, `description`, `allowed-tools`
  (one tool per line) and `disable-model-invocation`. A malformed frontmatter
  block stops Claude Code loading the skill.
- `disable-model-invocation: true` is what makes the routine user-invoked only;
  flipping it to `false` changes the instructions above, so document the change.
- Keep the task phrasings in `description` in sync with the [Usage](#usage)
  prompts above.
- Commit messages follow the existing history style: `feat: …`, `fix: …`,
  `chore(log): …`.

## Troubleshooting / FAQ

**Why not just `git commit` + `git push`?**
Because `push_files` creates the commit on GitHub; a separate local commit
would have the same content but a different SHA, and the two histories would
diverge. The API's commit is the truth — step 3 points local at it.

**`push_files` failed (missing `origin`, branch protection, bad token).**
The routine stops and reports what is missing. It must not fabricate a commit:
the Zoho subtask has to carry a real commit URL. Fix the remote or token, then
re-run.

**The push tool is not listed as `mcp__github__push_files` in my session.**
Expected: `_mcp.json` names the server `github-new`, so the tool likely appears
as `mcp__github-new__push_files`. Match the name your MCP tool list shows — see
[the naming caveat](#tool-naming-mcp__github__push_files).

**Is `git reset --hard` here dangerous?**
Only run it when `git status --porcelain` shows nothing but the `claude.log`
change this routine just pushed. If other local changes exist, the routine
stops and asks instead of discarding your work.

**A pinned Zoho ID returns 404.**
Stop and report it — the IDs are pinned on purpose. Do not silently search for
a replacement.

**Can I add a comment to a Zoho task?**
No: this connector has no comment-on-task tool. Say so rather than substituting
`updateTask`.

**Why can't the task be created with a specific status?**
`status` is not a free field on create. Create the task first, then call
`updateTask` with the desired `status.id`, taking the id from the project's
status options.

**The GitHub / Zoho Projects MCP tools are not available.**
The MCP servers are not configured in the session. Check that `_mcp.json` is
loaded as project MCP config and that
[`.claude/settings.json`](.claude/settings.json) pre-authorises
`mcp__zoho-projects`. The GitHub server is named `github-new` in that file, so
look for `mcp__github-new__*` in your tool list.

**Nothing happens when I ask Claude to "run the routine".**
That is by design: `SKILL.md` sets `disable-model-invocation: true`, so Claude
cannot start the skill from its `description`. Start it yourself with
`/log-commit-zoho` (its `name` field). See
[Installation step 4](#4-start-the-routine-explicitly).

## Contributing

- Open an issue describing the change before sending work that alters the
  routine's steps or pinned IDs.
- Branch from `master`; keep one logical change per pull request.
- If you touch `SKILL.md`, keep its frontmatter valid and the invocation
  instructions above accurate.
- Report the manual check from [Development](#development) in your pull request.

<!-- TODO: link CONTRIBUTING.md (and CODE_OF_CONDUCT.md) here once PR #1 and
PR #2 are merged; those files are not on master yet, so a relative link would
404 today. -->

## License

No `LICENSE` file is present in this repository, so **no license is stated** and
default copyright applies. **TODO:** choose a license and add it at the
repository root; then add the license badge and a link here.
