# Contributing to `claude-routine`

Demo/sandbox scaffolding, not a product. It exercises MCP servers and a Claude
Code skill end to end: append to `claude.log`, push via GitHub MCP, sync local to
that commit, create a Zoho Projects subtask linking it. No build or tests.

## Prerequisites

| Requirement | Notes |
| --- | --- |
| Claude Code CLI | Run it from the repo root so `.claude/` and `_mcp.json` are picked up. |
| `GITHUB_PAT` | GitHub token with write access; sent as `Authorization: Bearer ${GITHUB_PAT}` to `https://api.githubcopilot.com/mcp`. |
| `ZOHO_API` | Zoho MCP token, interpolated into `https://projects-task-management-929867360.zohomcp.com/mcp/${ZOHO_API}/message`; needs access to the pinned portal. |

## Secrets policy

> **Never commit a real token, PAT or Zoho token, in any file.** `_mcp.json` is
> tracked, so a leak there is a public leak the moment it is pushed.

- `_mcp.json` must only ever reference `${GITHUB_PAT}` and `${ZOHO_API}`; those
  placeholders *are* the contract — the values come from the environment.
- If a token is ever committed, rotate it at the provider first, then rewrite
  history — a later commit that deletes the line does not remove it.
- Before committing, read `git diff --cached` for anything credential-shaped.

## Repository layout

| Path | What it is |
| --- | --- |
| `README.md` | Short note about the routine; not the guide. |
| `claude.log` | Append-only routine log; committed, empty by design. |
| `_mcp.json` | Servers `github-new` and `zoho-projects` (both `http`); tracked, keep secret-free. |
| `.claude/settings.json` | `permissions.allow` pre-authorises `mcp__zoho-projects`. |
| `.claude/skills/log-commit-zoho/SKILL.md` | The routine, in four ordered steps; the core artefact. |
| `.claude/skills/commit-push-zoho-task.zip` | Tracked; `.gitignore` only excludes the unzipped dir. |

## Adding or modifying a skill

Create `.claude/skills/<skill-name>/SKILL.md`, where `<skill-name>` is the folder
name (mirroring `log-commit-zoho`). Frontmatter fields that matter:

| Field | Required | Meaning |
| --- | --- | --- |
| `name` | yes | Skill id; matches the folder name. |
| `description` | yes | When the skill applies — what the model reads. |
| `allowed-tools` | yes | Tools it may call; keep in sync with its body. |
| `disable-model-invocation` | no | `true` = manual-only slash command; no auto-invoke. |

`log-commit-zoho` sets `disable-model-invocation: true`: despite its trigger-heavy
`description` it does **not** auto-invoke. Removing that key is a PR-worthy change.

## Adding a new MCP server

1. Add an entry under `mcpServers` in `_mcp.json` (`type`, `url`, optional
   `headers`), using `${VAR}` for every credential; add `mcp__<server>` to
   `permissions.allow` in `.claude/settings.json` if it should run unprompted.
2. Re-validate the JSON (below). The name becomes the tool prefix
   `mcp__<server>__<tool>` that skills list in `allowed-tools`.

## Routine conventions you must not break

One entry per line in `claude.log`, timestamped from
`date "+%Y-%m-%d %H:%M:%S %Z"` and reused verbatim in the Zoho subtask title and
description so log line and ticket line up — never guess it, never drop it:

```
[YYYY-MM-DD HH:MM:SS TZ] <message>
```

Push `claude.log` with `mcp__github__push_files`, not the git CLI: the API-created
commit is the single source of truth. Then `git fetch origin <branch>` and
`git reset --hard origin/<branch>`, only when the sole pending change is
`claude.log` (otherwise stop and ask). A local commit plus an API push gives two
SHAs for identical content — that divergence is the bug this rule prevents.
Messages: conventional (`feat:`, `fix:`); log pushes use `chore(log): <message>`.

Pass these stable IDs straight into `path_variables`, never a `getAllPortals` /
`getAllProjects` / `getTasksByPortal` lookup:

| Field | Value |
| --- | --- |
| Portal ID | `36485097` |
| Project ID | `688906000001071159` (`PR-46`, "Unassigned Tasks") |
| Parent task ID | `688906000092147305` (`UNT-T48831`) |
| Parent task list ID | `688906000006396141` ("iOS - Ajay Ghodadra's Team") |
| Owner zpuid | `688906000048662473` (zuid `802509144`) |

If an ID returns 404, **stop and report it** rather than silently rediscovering
one. The connector has no comment-on-task tool — don't fake one with `updateTask`.

## Validating a change

No build, test runner or linter exists, so validate manually:

- [ ] `python3 -m json.tool` parses both `_mcp.json` and `.claude/settings.json`
      (the `${…}` placeholders keep them valid JSON).
- [ ] Touched `SKILL.md` frontmatter parses and lists `name`, `description`,
      `allowed-tools`; `git diff --cached` shows no credential.
- [ ] One end-to-end run on a scratch branch: the log line, the commit URL and
      the created Zoho subtask (same `YYYY-MM-DD HH:MM:SS TZ`) must agree.
- [ ] After the sync, `git rev-parse HEAD` equals the SHA `push_files` returned.

## Proposing a change

Fork or branch off `master`, commit conventionally (`docs: …`, `feat:`, `fix:`)
and open a pull request against `master`.

Describe the **exact behaviour change**, not just "update skill": e.g. "removes
`disable-model-invocation` so the skill becomes model-invocable", or "adds
`mcp__github` to `permissions.allow` so pushes stop prompting". Do not hand-edit
`claude.log` in a PR; it is written only by a routine run.

## Gotchas

- `.gitignore` only excludes the extracted `.claude/skills/commit-push-zoho-task/`
  directory; the `.zip` beside it is tracked.
- `claude.log` is intentionally committed — never add or reorder entries by hand.
- `_mcp.json` is tracked and must stay secret-free.
- Local and remote SHAs must match after a run; a local `git push` breaks that.
- Don't strip the pinned Zoho IDs "to be clean" — they are pinned on purpose.
