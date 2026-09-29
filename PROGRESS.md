# Coffee business progress and handoff

Updated: 2026-09-30. This file is the authoritative current implementation status. The sequence and gates are in [PROJECT_PLAN.md](PROJECT_PLAN.md).

## Current state

- Stage: 2 — architecture design (brainstorming). Design so far: [Operations automation architecture](Knowledge/Projects/Operations%20automation%20architecture.md).
- Completed: goal and constraints captured (condo-balcony café in Singapore, PayNow/cash, code-first Next.js + Supabase + Telegram + Claude API). Decisions in `Knowledge/Projects/Scope decisions.md`.
- In progress: section 1 approved 2026-09-30 (change: bot never messages customers; partner contacts them). Section 2 approved 2026-09-30 (gone quiet = 14 days). Similar-systems review done; Amendments in `Knowledge/Resources/Similar systems review.md`; owner chose nightly-export backups and automatic payment-proof checking with auto-verify on full pass.
- Failed or blocked: none. No code written.
- Not yet done: design sections 2 (data model; beans/finance/insights logic) and 3 (carousel agent); written spec; implementation plan.

## Next action

1. Present section 3 (carousel agent); then write the spec (confirm review amendments #1–4, #7 at spec review).
2. Write the spec, self-review, and ask the owner to review it before planning.

## Recent evidence

| Date | Claim | Evidence / check | Limits |
|---|---|---|---|
| 2026-09-28 | Project memory template initialized | Starter archive contents copied and placeholders adapted in this project | Business goal and implementation status remain unknown |
| 2026-09-28 | Private GitHub repo with vault included | Initial commit pushed to `owentw23/coffee-business`; `vault-lint` passed; see `Knowledge/Projects/Sessions/2026-09-28-github-repo.md` | No collaborators invited yet |
| 2026-09-28 | Shared Codex and Claude workspace documented | User reports Claude Code has access to the same folder; local checkout tracks `origin/main` | Does not verify Claude runtime access or GitHub account permissions |
| 2026-09-30 | Automation architecture decisions recorded | Owner answers in Claude Code conversation; see `Knowledge/Projects/Sessions/2026-09-30-automation-architecture.md` | Design only; SFA licence is owner-reported |
| 2026-09-30 | Auto-capture hooks configured | Synthetic hook events piped through `scripts/vault-hook.cjs`, exit 0; see `Knowledge/Projects/Sessions/2026-09-30-auto-capture-hooks.md` | Real Stop and SessionEnd events observed 2026-09-29 16:17–16:20 UTC (drafts curated and deleted 2026-09-30) |

Historical rationale and session records live under [Knowledge/](Knowledge/START-HERE.md). Update this handoff after every meaningful milestone; keep it concise.
