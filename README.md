# Claude Routine

A small demo of a Claude "routine": **log → commit → push → Zoho subtask**.

When the routine runs, Claude appends a timestamped line to `claude.log`, pushes that file to GitHub through the GitHub MCP server, and then files a subtask in Zoho Projects that links to the new commit.

This repository has **no application code**. It only holds the configuration Claude needs to run the routine: an MCP server definition, a permissions file, and a skill.

## Repository structure

```text
.
├── README.md                                  # This file
├── claude.log                                 # Append-only log written by the routine (currently empty)
├── _mcp.json                                  # MCP servers: github-new and zoho-projects
├── .gitignore                                 # Ignores an unzipped .claude/skills/commit-push-zoho-task/
└── .claude/
    ├── settings.json                          # Pre-approves all mcp__zoho-projects tools
    └── skills/
        ├── log-commit-zoho/
        │   └── SKILL.md                       # The main skill (the routine itself)
        └── commit-push-zoho-task.zip          # Archived earlier, different skill (see below)
```

| Path | Purpose |
| --- | --- |
| `claude.log` | One line per routine run, in the format described in [`claude.log` format](#claudelog-format). |
| `_mcp.json` | Defines two HTTP MCP servers: `github-new` (`https://api.githubcopilot.com/mcp`, authenticated with `GITHUB_PAT`) and `zoho-projects` (a Zoho MCP URL that contains `ZOHO_API`). |
| `.claude/settings.json` | Allows every tool of the `mcp__zoho-projects` server without a permission prompt. GitHub tools are not listed here. |
| `.claude/skills/log-commit-zoho/SKILL.md` | The skill that implements the four-step routine. |
| `.claude/skills/commit-push-zoho-task.zip` | An earlier, different skill, `commit-push-zoho-task`. It is not used by the routine (see below). |
| `.gitignore` | Contains `.claude/skills/commit-push-zoho-task/`, so an unzipped copy of the archived skill is not committed. |

### About `commit-push-zoho-task.zip`

The zip contains an earlier skill, `commit-push-zoho-task`, plus a `references/zoho-task.md` file (and macOS `__MACOSX` metadata). It differs from the current skill:

- It is invoked as `/commit-push-zoho-task project task-list [parent-task]`.
- It looks up the Zoho portal, project, task list, parent task and assignee at run time instead of using pinned IDs.
- It commits the working-tree changes and pushes with `git`, falling back to the GitHub MCP.

Treat it as an archive. The current behaviour is defined by `log-commit-zoho/SKILL.md`.

## Prerequisites and setup

You need:

- [Claude Code](https://docs.claude.com/en/docs/claude-code) with MCP support.
- A git clone of this repository with an `origin` remote that points to GitHub. The skill reads `owner`/`repo` from `git remote get-url origin`.
- Access to the Zoho Projects MCP connector used by the team.

### Environment variables

`_mcp.json` reads two environment variables. Set both in the shell you start Claude from.

| Variable | Used for | Notes |
| --- | --- | --- |
| `GITHUB_PAT` | Sent as `Authorization: Bearer ${GITHUB_PAT}` to `https://api.githubcopilot.com/mcp` (server `github-new`). | A GitHub personal access token with **repo write access** to this repository. For example, a classic token with the `repo` scope, or a fine-grained token with read/write access to the repository's *Contents*. |
| `ZOHO_API` | Substituted into the `zoho-projects` URL: `https://projects-task-management-929867360.zohomcp.com/mcp/${ZOHO_API}/message`. | The key from your Zoho MCP connector. Treat it like a secret. |

```bash
export GITHUB_PAT=<your-github-token>
export ZOHO_API=<your-zoho-mcp-key>
```

Never commit real values for these variables, and never paste them into this README, `claude.log`, issues or Zoho tasks.

### How `_mcp.json` is picked up

Claude Code automatically loads a project-level file named `.mcp.json`. This repository's file is named `_mcp.json`, so it is **not** loaded automatically by a plain `claude` run. Point Claude at it explicitly:

```bash
claude --mcp-config _mcp.json
```

(If your routine environment uses a different mechanism to register MCP servers, register the same two servers there.) After starting, run `/mcp` in Claude to confirm that `github-new` and `zoho-projects` are connected.

### Permissions

`.claude/settings.json` allows all `mcp__zoho-projects` tools, so Zoho calls run without prompts. It does not allow GitHub MCP tools, so Claude may ask you to approve the `push_files` call the first time.

## Running the routine

1. Export `GITHUB_PAT` and `ZOHO_API` (see above).
2. Start Claude from the repository root with the MCP config loaded:

   ```bash
   claude --mcp-config _mcp.json
   ```

3. Trigger the `log-commit-zoho` skill, for example by typing:

   ```text
   run the routine
   ```

   or invoke it explicitly with `/log-commit-zoho`. Phrases such as "log and commit" or "log a change and create a Zoho subtask" also match the skill's description. The skill has `disable-model-invocation: true`, so Claude will not start it on its own; see [Known limitations](#known-limitations).

4. Give a one-line log message. If you don't, Claude asks for one, or infers one from the current context and asks you to confirm.

### Example result

At the end of a run Claude reports the log line, the commit link and the Zoho subtask. The values below are placeholders:

```text
Log line:  [2025-01-15 10:30:00 UTC] Updated README
Commit:    https://github.com/krunal-simform/claude-routine/commit/<commit-sha>
Zoho:      subtask "Claude routine — 2025-01-15 10:30:00 UTC" under UNT-T48831 (<subtask-id or URL>)
```

## What the routine does

The steps run in order. The Zoho subtask needs the commit link, and the commit link needs a successful push.

1. **Append to `claude.log`.** Claude gets the timestamp from the system with `date "+%Y-%m-%d %H:%M:%S %Z"` and appends one line to `claude.log` (creating the file if needed). It saves that exact timestamp and reuses it in Step 4.
2. **Commit and push via the GitHub MCP.** Claude reads `owner`/`repo` from `git remote get-url origin` and the branch from `git branch --show-current`. It then calls the GitHub MCP `push_files` tool with the full contents of `claude.log` and a message like `chore(log): <log message>`. The commit SHA returned by that call is the authoritative commit id. If the call fails, the routine stops and tells you what is missing instead of making up a commit.
3. **Sync the local repo.** Claude runs `git fetch origin <branch>`. If the only uncommitted change is `claude.log`, it runs `git reset --hard origin/<branch>`, so local HEAD equals the commit GitHub just created. If there are other uncommitted changes, it stops and asks you what to do rather than discarding your work.
4. **Create the Zoho subtask.** Claude calls the Zoho `createTask` tool under parent task `UNT-T48831`:
   - **Title:** `Claude routine — <YYYY-MM-DD HH:MM:SS TZ>`, using the Step 1 timestamp. The timestamp must always be in the title.
   - **Description:**

     ```text
     Automated routine run.
     Commit: <commit link>
     Timestamp: <YYYY-MM-DD HH:MM:SS TZ>
     Log entry: <log message>
     ```

   - **Owner and task list:** fixed to Krunal Patel and the parent's task list (see [Zoho identifiers](#zoho-identifiers)).
   - **Status:** not set on create. If a different status is needed, the skill creates the task first and then calls `updateTask`.

Claude finishes by reporting the log line, the commit link, and the URL or ID of the new Zoho subtask.

### Why the push uses the GitHub MCP and not the git CLI

`push_files` creates the commit on GitHub through the API. It does not upload a local commit object. If Claude committed locally with `git commit` *and* pushed through the API, there would be two commits with the same content but different SHAs (different committer metadata), and local and remote history would diverge. To avoid this, the skill treats the GitHub-side commit as the single source of truth and then resets the local branch to it (Step 3).

## `claude.log` format

One entry per line, so the file stays greppable:

```text
[YYYY-MM-DD HH:MM:SS TZ] <log message>
```

Example:

```text
[2025-01-15 10:30:00 UTC] Updated README
```

## Zoho identifiers

The skill pins the Zoho identifiers instead of looking them up, so it does not call discovery tools such as `getAllPortals` or `getAllProjects`. These are internal IDs, not secrets or API keys.

| Field | Value |
| --- | --- |
| Portal | `simform` (ID `36485097`) |
| Project | `PR-46` "Unassigned Tasks" (ID `688906000001071159`) |
| Parent task | `UNT-T48831` "Cluade test tasks" (ID `688906000092147305`) |
| Parent task list | "iOS - Ajay Ghodadra's Team" (ID `688906000006396141`) |
| Owner | Krunal Patel (zpuid `688906000048662473`) |

If any of these IDs returns a 404 or not-found (for example, the task was moved or deleted), the routine **stops and tells you**. It does not search for a replacement, because the identifiers are pinned on purpose. Update `SKILL.md` deliberately if the target task changes.

## Known limitations

- **No comment-on-task tool.** The Zoho connector has no "add comment" tool. If asked to comment on `UNT-T48831`, the skill says so. It does not fake a comment with `updateTask` or another tool.
- **Explicit invocation only.** `SKILL.md` sets `disable-model-invocation: true`, so Claude will not run the skill on its own. Ask for it by name (`/log-commit-zoho`) or with a request such as "run the routine".
- **Hard-coded Zoho IDs.** The routine only works for the one parent task listed above.
- **GitHub tool naming.** `SKILL.md` refers to the GitHub tool as `mcp__github__push_files`, but `_mcp.json` names the server `github-new`, so the tool is exposed under that server name (for example `mcp__github-new__push_files`). Claude should use the GitHub MCP `push_files` tool from the `github-new` server.
- **Broad `allowed-tools` (optional follow-up).** The `allowed-tools` list in `SKILL.md` includes write tools the routine does not use, such as `createProject`, `trashProject` and `updateIssue`. It is documented here only and has not been changed.

## Troubleshooting

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| Step 2 fails because `owner`/`repo` can't be determined | There is no `origin` remote. | Add one: `git remote add origin <github-url>`, then rerun. |
| `push_files` is rejected | Branch protection or another push rule on the current branch. | The routine stops and does not invent a commit. Switch to an unprotected branch or relax the rule, then rerun. |
| Step 3 stops and asks what to do | There are uncommitted local changes other than `claude.log`. | Commit or stash them, then rerun. The skill never resets over unrelated work. |
| `github-new` fails with 401, or won't connect | `GITHUB_PAT` is unset, expired, or lacks write access to the repository. | Export a valid token and restart Claude. |
| `zoho-projects` won't connect or tools are missing | `ZOHO_API` is unset, so the URL is malformed, or `_mcp.json` was not loaded. | Export `ZOHO_API`, start Claude with `--mcp-config _mcp.json`, and check `/mcp`. |
| Zoho call returns 404 / not found | A pinned ID no longer exists. | The routine stops. See [Zoho identifiers](#zoho-identifiers). |

Environment variables are read when Claude starts, so restart Claude after exporting or changing them.
