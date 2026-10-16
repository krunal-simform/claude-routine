# Contributing

This repository is a small demo of one Claude Code "routine". The tracked
surface is documentation, configuration and a single skill definition, so a
contribution is normally an edit to one of those files. Keep changes small, and
keep them true to what the repository actually contains.

There is no application code, package manifest, build step, test suite, CI
workflow, or LICENSE file in this repository — do not assume any of them when
opening a pull request. If a change genuinely needs one of them, open an issue
first so a maintainer can decide.

## What this repository does

- The routine is the skill `.claude/skills/log-commit-zoho/SKILL.md`. One run
  performs four steps, in order:
  1. append one timestamped line to `claude.log` at the repository root;
  2. publish that file through the GitHub MCP tool `push_files`, which creates
     the commit through the GitHub API rather than with `git commit` /
     `git push`;
  3. reconcile the local checkout to the exact SHA that push returned;
  4. create a subtask under the pinned Zoho Projects parent task `UNT-T48831`.
- A log entry is a single line in the format
  `[YYYY-MM-DD HH:MM:SS TZ] <log message>`. The same timestamp is reused in the
  Zoho subtask title and description so that the log line and the ticket line
  up.
- The skill is user-invoked: its front matter sets
  `disable-model-invocation: true`, so a run starts from an explicit request
  that names the routine rather than from the model selecting it unaided.
- `README.md` states the durable promise:

  > After every routine call the claude will add log to claude.log file.

  If you change the routine's steps, keep `README.md`, `claude.log` and
  `SKILL.md` in agreement.

## Prerequisites and local setup

1. **Claude Code** (CLI or IDE extension). No version is pinned or documented
   here — use your current install.

2. **Clone and branch:**

   ```bash
   git clone https://github.com/krunal-simform/claude-routine.git
   cd claude-routine
   git switch -c docs/your-change
   ```

   Commit to `master` directly only for the routine's own `claude.log` commits.

3. **MCP servers.** `_mcp.json` declares two HTTP servers:

   | Key             | Type   | URL                                                                              | Credential                                   |
   | --------------- | ------ | -------------------------------------------------------------------------------- | -------------------------------------------- |
   | `github-new`    | `http` | `https://api.githubcopilot.com/mcp`                                              | header `Authorization: Bearer ${GITHUB_PAT}` |
   | `zoho-projects` | `http` | `https://projects-task-management-929867360.zohomcp.com/mcp/${ZOHO_API}/message` | none declared                                |

   - Export `GITHUB_PAT` and `ZOHO_API` in whatever environment launches Claude
     Code. Both are load-bearing: with a value missing, that server is unusable.
     On the GitHub side the symptom is a failed push, which the routine treats
     as terminal.
   - Credential *values* are never committed; `${...}` are environment
     placeholders. Never replace a placeholder with a literal value in a commit.
   - Nothing in this repository loads `_mcp.json` (its name is not
     `.mcp.json`), so register these two entries in your own Claude Code MCP
     configuration, using this file as the reference.
   - The server key is the tool-name prefix: `zoho-projects` maps to
     `mcp__zoho-projects__*`, and `github-new` maps to `mcp__github-new__*`.
     Known mismatch, documented rather than fixed: the skill's prose names
     `mcp__github__push_files`, while the only GitHub server declared here is
     keyed `github-new`. If your environment registers the server under a
     different key, every tool name in the skill changes with it.

4. **Permissions.** `.claude/settings.json` currently allow-lists only
   `mcp__zoho-projects`. The GitHub push tool appears neither there nor in the
   skill's `allowed-tools`, so expect an approval prompt at the push step. Do
   not widen the committed allow list as part of a documentation or template
   change.

5. **Smoke check.** In a session, ask the agent to run the routine with a
   one-line message and confirm you get back the log line, the commit link and
   the Zoho subtask URL (the skill's "Finish" report). When experimenting, run
   on a scratch branch, and do not run the routine while you have unrelated
   uncommitted changes: Step 3 stops rather than resetting over them.

## Project layout

| Path                                       | What it is                                                   |
| ------------------------------------------ | ------------------------------------------------------------ |
| `README.md`                                | One-line description of the routine.                         |
| `_mcp.json`                                | The two MCP servers; a reference, not auto-loaded.           |
| `.claude/settings.json`                    | Session permissions; allow-lists `mcp__zoho-projects`.       |
| `.claude/skills/log-commit-zoho/SKILL.md`  | The routine itself (206 lines; front matter on lines 1–29).  |
| `.claude/skills/commit-push-zoho-task.zip` | The predecessor skill, checked in as an archive.             |
| `claude.log`                               | Append-only routine output; tracked on purpose.              |
| `.gitignore`                               | One rule: excludes the unpacked predecessor directory.       |
| `CONTRIBUTING.md`                          | Prerequisites, layout, skill authoring, workflow.            |
| `CODE_OF_CONDUCT.md`                       | Contributor Covenant v2.1; contact is a placeholder.         |
| `.github/`                                 | Pull request and issue templates.                            |

`.gitignore` contains exactly one rule — the *unpacked* predecessor directory
`.claude/skills/commit-push-zoho-task/` — while the `.zip` stays tracked.

## Adding or changing a skill

- **Location.** `.claude/skills/<name>/SKILL.md`, where `<name>` is the token a
  user types. The front-matter `name:` must repeat the directory name; renaming
  one without the other leaves a file that reads fine to a human but no longer
  matches its invocation identity.

- **Front matter.** As used here: `name`, `description` (this repository uses a
  folded `>-` block that lists the trigger phrasings), `allowed-tools` (a YAML
  list of `mcp__<server>__<tool>` entries) and `disable-model-invocation: true`.
  The flag, not the description, decides whether a run is user-invoked or
  model-selected, so keep the two in agreement.

- **Keep `allowed-tools` in step with the body.** Search the prose for tool
  names whenever you edit the list. Removing a tool that a step calls breaks
  that step at run time with no load-time error; adding a tool nobody calls
  silently widens what an invoked run may do.

- **Package shape.** `log-commit-zoho` ships as a plain `SKILL.md`;
  `commit-push-zoho-task` is distributed as a tracked `.zip`. New skills should
  ship unpacked as `.claude/skills/<name>/SKILL.md`. If you do add an archive,
  add its unpacked directory to `.gitignore` the way the existing one is
  handled.

- **Safe-edit checklist.**
  - Rename the directory and the front-matter `name:` together.
  - Edit the pinned Zoho identifiers only in the identifier table in `SKILL.md`
    — it is the single source of truth for the body and for Step 4.
  - Keep the log-line format and the Zoho title/description timestamp identical.
  - Never inline a credential.

## Change workflow

- **Branch per change**, and fork first if you are not a collaborator. Use
  `<type>/<short-slug>` — `docs/`, `feat/`, `fix/`, `chore/` — for example
  `docs/contributing-files`.

- **Small, focused commits.** One logical change per pull request.

- **Commit messages** follow `type(scope): imperative summary`, for example
  `docs(contributing): add contributor docs`. The shape
  `chore(log): <log message>` is the one the *routine itself* uses when it
  creates the `claude.log` commit through the GitHub API — use it only for
  routine-generated log commits, and never hand-amend, re-word or re-push those
  commits. Do not rewrite history on a branch a routine run has already pushed
  to: the local checkout is deliberately reset onto the SHA the API created.

- **Pull requests** target `master` and fill in
  `.github/PULL_REQUEST_TEMPLATE.md`. Say what changed and why, list the files,
  and describe how you verified it — there is no CI, so the pull request text
  is the verification record. Link the related Zoho task by putting the subtask
  key or URL in the pull request; routine runs create subtasks under the parent
  task `UNT-T48831` (project key `PR-46`, "Unassigned Tasks", portal
  `36485097`). If a routine run produced the change, also include the commit
  link `https://github.com/krunal-simform/claude-routine/commit/<SHA>` and the
  exact log line. If a change has no Zoho task, write `<no Zoho task>` rather
  than inventing a reference.

- **Review.** Expect at least one maintainer review before merge. Squash or
  fast-forward, but do not rewrite a branch that already carries a
  routine-created `claude.log` commit.

## Docs and style rules

- Markdown only, ATX (`#`) headings, files at the repository root or under
  `.github/`.
- Wrap prose near 80 characters, as `SKILL.md` does. Use fenced code blocks with
  a language tag (`bash`, `json`, `markdown`), and tables for field/value lists.
- Refer to files by repository-relative path in backticks, and link with
  relative paths (`CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`,
  `.claude/skills/log-commit-zoho/SKILL.md`). No absolute sandbox or machine
  paths.
- Dates and times use `YYYY-MM-DD HH:MM:SS TZ`, matching the log entry format.
- Do not duplicate the routine's steps or identifiers outside `SKILL.md`. Link
  to the file and describe behaviour in prose so that there is one source of
  truth.
- Never commit a credential value. Use `${GITHUB_PAT}` and `${ZOHO_API}` for
  environment placeholders, and `<angle-bracket>` placeholders for unknown facts
  such as contacts.

## Generated output and secrets

- `claude.log` is append-only output produced by the routine and is **tracked on
  purpose**. Every run appends one line and pushes the file, and the
  reconciliation step depends on the commit that push creates. Do not hand-edit,
  reorder, reformat or delete entries, do not add it to `.gitignore`, and expect
  a routine run to create a commit on whichever branch you are on.
- `.claude/skills/commit-push-zoho-task.zip` is checked in for history; the
  unpacked `.claude/skills/commit-push-zoho-task/` directory is the only thing
  `.gitignore` excludes.
- **Secrets.** Never commit tokens, `.env` files or rotated credentials —
  placeholders only. This repository has no secret scanning or other automated
  guard, so if you push a secret, tell a maintainer immediately and rotate it.

## Questions and issues

- Open an issue at <https://github.com/krunal-simform/claude-routine/issues>,
  using the templates in `.github/ISSUE_TEMPLATE/`.
- For questions about the tracked task itself, reference the Zoho Projects
  parent task `UNT-T48831`.
- Maintainer contact: `<maintainer-email>` and `<maintainer-github-handle>`.
  These are explicit placeholders for a maintainer to fill in, not an oversight.

Participation in this repository is covered by our
[Code of Conduct](CODE_OF_CONDUCT.md).
