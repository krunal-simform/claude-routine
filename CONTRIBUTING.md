# Contributing

Thanks for helping with this repo. This guide explains how to work in it.

## Purpose and scope

This repo has no application code. It holds a Claude routine: the
`log-commit-zoho` skill appends a timestamped line to `claude.log`, pushes it
to GitHub through the GitHub MCP, and creates a Zoho Projects subtask under
`UNT-T48831`.

Contributions are mostly edits to the skill and its config, not code.

Files that matter:

| File                                      | What it is                                          |
| ----------------------------------------- | --------------------------------------------------- |
| `README.md`                               | One-line description of the routine                 |
| `claude.log`                              | The routine's log file                              |
| `.claude/skills/log-commit-zoho/SKILL.md` | The skill definition                                |
| `.claude/settings.json`                   | Permissions for the `zoho-projects` MCP tools       |
| `_mcp.json`                               | MCP server config for `github-new` and `zoho-projects` |

## Setup

`_mcp.json` defines two HTTP MCP servers:

- `github-new`: the GitHub MCP server. It authenticates with a token
  read from the `GITHUB_PAT` environment variable.
- `zoho-projects`: the Zoho Projects MCP server. Its URL contains the
  `ZOHO_API` environment variable.

Set both environment variables before you start Claude, using your own values:

```bash
export ZOHO_API=<your-zoho-mcp-key>
export GITHUB_PAT=<your-github-token>
```

The skill pushes with the GitHub MCP `push_files` tool, so your GitHub MCP
server must provide it. Note that `_mcp.json` names the server `github-new`
while `SKILL.md` refers to `mcp__github__push_files`. Check that the tool
names in your setup match what the skill expects.

**Never commit credentials or tokens.** That includes a GitHub PAT, the real
`ZOHO_API` value, and any Zoho MCP URL with the key filled in. Keep
`_mcp.json` as `${VAR}` references and use placeholders like the ones above in
docs and examples.

`.claude/settings.json` allows the `mcp__zoho-projects` tools. Don't widen
these permissions as part of an unrelated change.

## Commit conventions

Follow the Conventional Commit style used in the existing history:

- `feat:` for new behaviour, e.g. `feat: add mcp`, or
  `feat: use GitHub MCP for push instead of git CLI in log-commit-zoho skill`.
- `fix:` for corrections, e.g.
  `fix: correct zoho tool names and allow write tools in log-commit-zoho skill`.
- `chore(log):` for the routine's own log commits, e.g.
  `chore(log): <log message>`. These are created by the routine, not by hand.

Use a short, imperative subject.

## Changing the skill (`SKILL.md`)

The skill is at `.claude/skills/log-commit-zoho/SKILL.md`. When you edit it:

- **Keep the step order.** Each step depends on the one before it:
  1. Append a line to `claude.log`.
  2. Commit and push through the GitHub MCP (`push_files`).
  3. Sync the local repo to the commit that GitHub created.
  4. Create the Zoho subtask.
- **Don't use `git push` (or a local `git commit`) for the routine's commit.**
  The GitHub MCP creates the commit on GitHub. Pushing a separate local commit
  as well would give two commits with the same content but different SHAs, and
  local and remote history would diverge.
- **Keep the pinned Zoho IDs in sync with Zoho.** The IDs are in the "Fixed
  Zoho identifiers" table in `SKILL.md`. If one returns 404, stop and tell the
  user instead of searching for a replacement. Don't copy the IDs into other
  docs. Point to the table.
- **The Zoho connector has no add-comment tool.** Don't fake a comment with
  `updateTask` or another tool.
- Keep the `allowed-tools` list in the `SKILL.md` frontmatter consistent with
  `.claude/settings.json`.

## `claude.log` rules

- One entry per line.
- Format: `[YYYY-MM-DD HH:MM:SS TZ] <message>`
- Take the timestamp from the system with `date "+%Y-%m-%d %H:%M:%S %Z"`.
- The routine appends entries. Don't edit or reorder them by hand.

## Branch and PR process

1. Branch from `master`.
2. Make small, focused changes.
3. Open a pull request against `master`.
4. Describe what changed and why. If the change touches the skill, say which
   step it affects.

No review or merge rules are defined for this repo yet. The team can replace
this section with its own rules.
