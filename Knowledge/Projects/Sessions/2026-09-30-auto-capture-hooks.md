---
tags: [session, tooling, memory]
status: developing
created: 2026-09-30
modified: 2026-09-30
source: owner-conversation-2026-09-30
---
# Automatic vault capture hooks — 2026-09-30 (Claude Code)

## Request
Owner asked why vault memory only saved on request. Cause: `scripts/vault-hook.cjs` existed but no hook config called it; CLAUDE.md only instructs a save at the end of substantive work. Owner chose a two-layer setup.

## Decision
- Raw draft capture: project `.claude/settings.json` runs `scripts/vault-hook.cjs` on `Stop`, `PreCompact` and `SessionEnd`, writing untrusted drafts to `Knowledge/Inbox/Auto/` (git-ignored, skipped by `vault-lint`).
- Curated save: `SessionStart` (also fires after compaction) injects the handoff plus a reminder to curate any pending drafts into `PROGRESS.md` and vault notes, then delete them. A hook script cannot write curated summaries itself, so curation happens at the next session start or when I finish substantive work.

## Evidence
- Piped synthetic Stop, PreCompact, SessionEnd and SessionStart events into the script; all exited 0 and the SessionStart output reported the pending-draft count. `vault-lint` passed.
- Not verified: that Claude Code actually fires the hooks. Settings may need a new session to load; check `Knowledge/Inbox/Auto/` after the next turn.

## Related
[[Projects/Sessions/2026-09-28-shared-workspace]]
