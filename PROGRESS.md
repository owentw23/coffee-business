# Coffee business progress and handoff

Updated: 2026-10-11. This file is the authoritative current implementation status. The sequence and gates are in [PROJECT_PLAN.md](PROJECT_PLAN.md).

## Current state

- Stage: 2 (reopened) — design v2. Business model clarified 2026-10-11: weekday cold brew pre-orders (6pm-day-before cutoff, attended handover), weekend balcony shop (menu TBD), 14-day A/B bean cycles starting Saturdays (week 1 mystery, week 2 revealed), mystery game at the counter on the debut weekend (free-text guess, barista judges, one try per mobile per bean, correct = drink free).
- Spec v2 written: `docs/superpowers/specs/2026-10-11-operations-automation-design-v2.md` (draft, awaiting owner review). v1 spec marked superseded.
- Plans: roadmap and Plan 1 exist but **Plan 1 must be revised** for bean cycles, slot recipes and giveaways before execution.
- Owner proposal Word document (`docs/proposal/`) describes v1; needs a v2 refresh.
- Failed or blocked: none. No application code written yet.
- Pending from owners: spec v2 review; weekend menu; weekday pickup times; cold brew bottle size/price/ratio/yield; brand kit; IG Business account; paid Claude plan.

## Next action

1. Owner reviews spec v2.
2. Revise roadmap + Plan 1 to v2 (about 30–45 minutes of agent work).
3. Refresh the proposal brief for v2.

## Recent evidence

| Date | Claim | Evidence / check | Limits |
|---|---|---|---|
| 2026-09-28 | Project memory template initialized | Starter archive contents copied and placeholders adapted in this project | Business goal and implementation status remain unknown |
| 2026-09-28 | Private GitHub repo with vault included | Initial commit pushed to `owentw23/coffee-business`; `vault-lint` passed; see `Knowledge/Projects/Sessions/2026-09-28-github-repo.md` | No collaborators invited yet |
| 2026-09-28 | Shared Codex and Claude workspace documented | User reports Claude Code has access to the same folder; local checkout tracks `origin/main` | Does not verify Claude runtime access or GitHub account permissions |
| 2026-09-30 | Automation architecture decisions recorded | Owner answers in Claude Code conversation; see `Knowledge/Projects/Sessions/2026-09-30-automation-architecture.md` | Design only; SFA licence is owner-reported |
| 2026-09-30 | Auto-capture hooks configured | Synthetic hook events piped through `scripts/vault-hook.cjs`, exit 0; see `Knowledge/Projects/Sessions/2026-09-30-auto-capture-hooks.md` | Real Stop and SessionEnd events observed 2026-09-29 16:17–16:20 UTC (drafts curated and deleted 2026-09-30) |
| 2026-09-30 | Design spec written | `docs/superpowers/specs/2026-09-30-operations-automation-design.md`; self-reviewed (timezone fix) | Owner-approved 2026-09-30; Canva/Cowork capabilities unverified until spike |
| 2026-09-30 | Roadmap and Plan 1 written | `docs/superpowers/plans/`; self-review fixed workspace ordering, CI build step, test count | Plan not yet executed; SQL and code untested until Task runs |
| 2026-09-30 | Owner proposal Word document created | `docs/proposal/Cafe-Operations-System-Proposal.docx`; 9 A4 pages rendered and visually reviewed; 5 tables, 10 numbered sections, Figure 1; accessibility audit found 0 issues | Prices and service capabilities follow the supplied brief; no independent price or connector verification in this document task |
| 2026-10-11 | Spec v2 drafted for the clarified business model | `docs/superpowers/specs/2026-10-11-operations-automation-design-v2.md`; owner answers on cycle start, cutoff, one-try rule, judging | Not owner-reviewed; brew maths uses placeholder numbers |

Historical rationale and session records live under [Knowledge/](Knowledge/START-HERE.md). Update this handoff after every meaningful milestone; keep it concise.
