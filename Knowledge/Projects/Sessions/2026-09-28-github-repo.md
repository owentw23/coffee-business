---
tags: [session, setup, git]
status: evergreen
created: 2026-09-28
modified: 2026-09-28
source: claude-code-session
---
# GitHub repository setup — 2026-09-28

## Request
Set up a GitHub repository for the project, then share it privately with other people.

## Decisions
- Owner chose a **single private repository**, `owentw23/coffee-business`, which contains both the project and the `Knowledge/` vault. Collaborators receive the memory with one clone. See [[Projects/Scope decisions]].
- Rejected: two separate repositories (collaborators could miss the vault) and a submodule (each vault change needs an extra sync commit in the parent).
- The empty, commit-less `Knowledge/.git` was removed; `Knowledge/` was removed from the root `.gitignore`. `Knowledge/.gitignore` still excludes `.obsidian/` and `Inbox/Auto/`.
- Updated the commit instructions in `AGENTS.md`, `CLAUDE.md`, `OBSIDIAN-VAULT-GUIDE.md`, and [[VAULT-RULES]] to point at the project repository. Pushes still require a request.
- Added `README.md` with clone and collaboration steps.

## Checks and limits
- Before committing: `node scripts/vault-lint.cjs` passed, and a secret-pattern scan found no credentials or `.env` files.
- Collaborators have not been invited yet; the owner will do that on GitHub.

## Next action
Invite collaborators (repository **Settings → Collaborators**), then define the business goal in `PROJECT_PLAN.md`.
