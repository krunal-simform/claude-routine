# Contributing to claude-routine

This repository is a small demo of a single Claude Code routine: one skill that
appends a line to `claude.log`, publishes it through the GitHub MCP server, syncs
the local checkout to the commit that push created, and opens a Zoho Projects
subtask. There is no application code, package manifest, build step, test suite,
CI workflow or `LICENSE` file, so a contribution is normally a small edit to
documentation, configuration or the skill itself. The routine lives in
[`.claude/skills/log-commit-zoho/SKILL.md`](.claude/skills/log-commit-zoho/SKILL.md):
read it before changing anything it describes, and treat it rather than any
summary here as the reference for the procedure.

## Getting started

### Prerequisites

| Requirement | Why you need it |
| --- | --- |
| `git` and a shell with `date` | To clone the repository. The routine's timestamp comes from `date "+%Y-%m-%d %H:%M:%S %Z"`, so `date` must be available. |
| A Claude Code client with MCP support | The routine is a skill that calls MCP tools. No version is pinned in this repository. |
| `GITHUB_PAT` | A GitHub token with contents-write access to this repository. `_mcp.json` sends it as `Authorization: Bearer ${GITHUB_PAT}` to the GitHub MCP server. |
| `ZOHO_API` | A Zoho MCP token with access to the Zoho Projects portal the routine writes to. `_mcp.json` interpolates it into the Zoho MCP server URL. |

Credential values are never committed. `_mcp.json` is tracked and must only ever
contain the `${GITHUB_PAT}` and `${ZOHO_API}` placeholders. If a token is
committed, rotate it at the provider first — deleting the line in a later commit
does not remove it from history.

### Run it locally

```bash
git clone https://github.com/krunal-simform/claude-routine.git
cd claude-routine
```

1. Export `GITHUB_PAT` and `ZOHO_API` in the environment that launches your
   Claude Code client.
2. Register the two server entries from `_mcp.json` — `github-new` and
   `zoho-projects`, both `type: http` — in your own MCP client configuration.
   Nothing loads `_mcp.json` automatically, so use it as the reference.
3. Start Claude Code from the repository root so `.claude/` is picked up.
4. Invoke the skill by name: `/log-commit-zoho`. Its frontmatter sets
   `disable-model-invocation: true`, so it does not auto-invoke — you have to
   ask for it.
5. Confirm the finish report: the log line added, the commit link and the Zoho
   subtask id. A run that reports only some of these failed; say which part.

`.claude/settings.json` pre-approves only `mcp__zoho-projects`, so expect an
approval prompt the first time a GitHub MCP tool is called; do not widen the
committed allow list as part of an unrelated change. There is no install, build
or test command to run: the repository *is* the routine.

## How to contribute

1. **Open an issue first** for anything non-trivial — a change to the routine's
   steps, the `claude.log` format, the MCP configuration or the skill's
   frontmatter. Small fixes such as typos or broken links can go straight to a
   pull request.
2. **Fork** if you are not a collaborator, then clone your fork.
3. **Branch off `master`.** Do not commit to `master` directly, except for the
   `claude.log` commit that a routine run itself creates.
4. Make one logical change, **commit** it, and **push** the branch.
5. **Open a pull request against `master`** describing what changed and why.

There is no CI here, so nothing runs on your pull request. The pull request text
is the verification record — state what you actually ran.

## Branch and commit conventions

- **Short-lived topic branches** named `<type>/<short-slug>`, cut from `master`:
  `feat/…` for a new capability, `fix/…` for a correction, `docs/…` for
  documentation only, `chore/…` for housekeeping that changes neither behaviour
  nor docs.
- **Imperative, specific subject lines** of roughly 72 characters or fewer:
  "add contributing guide", not "added some stuff".
- **Prefix the subject with `type(scope):`** when it helps. The history on
  `master` already uses `feat:` and `fix:`; `docs: add CONTRIBUTING.md` is the
  same shape. Keep the wording consistent within one pull request.
- **`chore(log): …` belongs to the routine.** That is the message shape the
  skill's push step uses when it creates the `claude.log` commit through the
  GitHub API. Never hand-write, amend or force-push such a commit: the local
  checkout is reset onto the SHA the API created, so rewriting one desynchronises
  local and remote history.
- **Automation branches** such as `pex/<ticket>` are created by tooling for task
  work; they are not the convention to copy for your own contributions.

## Pull request expectations

- **Small and focused.** One concern per pull request, with every changed file
  accounted for in the description.
- **A description that answers what and why** — what changed, what prompted it,
  and anything a reviewer would otherwise have to guess.
- **A link to the related issue** (`Closes #NN`) when one exists. If there is
  none, say so explicitly rather than leaving it out silently.
- **How you verified it locally**, naming the concrete check you ran: for docs
  or configuration, the preview or linter used and whether the JSON still parses
  (`python3 -m json.tool _mcp.json`); for a skill change, the log line, commit
  link and Zoho subtask id from a real run, or which of the four steps you could
  not verify.
- **Anything that changes behaviour**, in its own paragraph rather than buried:
  edits to `_mcp.json`, to `.claude/settings.json`, or to `SKILL.md` frontmatter
  (especially `disable-model-invocation`).
- **A maintainer review before merge.** There is no automated gate, so a human
  is the only check.

## Code and documentation style

Match what is already in the repository rather than introducing new tooling.

- **One source of truth for the routine.** Edit
  `.claude/skills/log-commit-zoho/SKILL.md`; do not copy its steps into other
  files. Summarise behaviour in prose and link to the skill.
- **Keep skill frontmatter internally consistent.** `name` must equal the
  skill's directory name, `description` and `allowed-tools` must describe what
  the body actually does, and `disable-model-invocation: true` stays unless
  making the skill model-invocable is the point of your pull request.
- **Markdown house style**, as `SKILL.md` uses it: ATX (`#`) headings, fenced
  code blocks with a language tag (`bash`, `json`), tables for field/value
  lists, repo-relative paths in backticks, relative links between files, and
  prose wrapped near 80 characters.
- **Keep documentation in step with behaviour.** If the routine's steps change,
  `README.md`, `SKILL.md` and the `claude.log` line format must still agree.
- **`claude.log` is generated output.** Append-only, one entry per line, written
  only by a routine run — never hand-edit it, reorder it, or add it to
  `.gitignore`.
- **Never commit a credential value.** Use `${GITHUB_PAT}` and `${ZOHO_API}` for
  environment placeholders and `<angle-bracket>` placeholders for facts you do
  not know. Read `git diff --cached` for anything credential-shaped.

## Reporting bugs and requesting features

Use GitHub Issues: <https://github.com/krunal-simform/claude-routine/issues>.
The repository defines no issue templates, so a plain issue is expected.

A useful bug report contains:

- **What you ran** — the exact prompt or skill invocation, and the repository
  state at the time (branch, and whether `claude.log` had uncommitted changes).
- **What you expected and what happened instead.**
- **Which of the four steps failed** — log append, GitHub MCP push, local sync,
  or Zoho subtask — because each fails differently and they run in order.
- **Your environment** — Claude Code version, operating system and shell.
- **The error text**, or a sanitized tool response.
- **Whether a commit link or a Zoho task id was produced**, even if the run was
  only partly successful.

Never paste tokens, `Authorization` headers, or tool payloads that contain them
into an issue.

For a feature request, describe the use case and what it would change about the
log line or the Zoho task contract, so the effect on the routine is explicit.

## Code of conduct

Be respectful and constructive, assume good faith, and keep discussion focused
on the change at hand. Harassment, personal attacks and discriminatory language
are not welcome in this project's issues, pull requests or reviews. Maintainers
may edit, hide or remove contributions and comments that do not meet this
standard, and may block contributors who persist.

<!-- TODO: link CODE_OF_CONDUCT.md once the Code of Conduct proposed in PR #1
     lands on master. It does not exist on master today, so a relative link
     would 404. -->

## Questions and contact

Ask in a GitHub Issue — that is the project's discussion channel. The repository
owner is `krunal-simform`, who can be @-mentioned in an issue or pull request
when something needs a maintainer's decision. No maintainer email address is
published for this project, so please use an issue rather than waiting for one.
