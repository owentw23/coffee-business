# Coffee business progress and handoff

Updated: 2026-10-11. This file is the authoritative current implementation status. The sequence and gates are in [PROJECT_PLAN.md](PROJECT_PLAN.md).

## Current state

- Stage: 2–3 — spec v2 draft + implementation plans v2. Business: weekday cold brew pre-orders (black/white, milk +S$1 for oat/Rokeby/soy, house beans, 6pm-day-before cutoff, attended handover); weekend balcony shop (menu TBD) with 14-day premium A/B bean cycles (Saturday start, mystery week then reveal) and a counter guessing game.
- Customer side is now our own **website** (menu of the week, info, about, pre-order form); staff side is an **all-in-one staff web app** at `staff.<domain>`. Tally dropped.
- Spec: `docs/superpowers/specs/2026-10-11-operations-automation-design-v2.md` (draft, awaiting owner review). Plans: `docs/superpowers/plans/2026-10-11-roadmap-v2.md` (7 plans) and `2026-10-11-plan-1-foundation-v2.md` (11 tasks, self-reviewed). Old roadmap/Plan 1 marked superseded.
- Owner proposal Word document (`docs/proposal/`) describes v1; needs a v2 refresh.
- Failed or blocked: none. No application code written yet.

## Next action

1. Owner reviews spec v2 and Plan 1 v2 (cold brew no A/B: confirmed).
2. Execute Plan 1 v2 Tasks 1–10 **locally** (subagent-driven recommended); Task 11 (domain + production) deferred by owner.
3. Initial cost projection: `Knowledge/Resources/Cost projection.md` (≈ S$27/month after launch; S$0 while building locally).

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
| 2026-10-11 | Spec v2 revised (website + staff app, cold brew milk options, separate beans); roadmap v2 and Plan 1 v2 written | `docs/superpowers/plans/2026-10-11-*.md`; self-review fixed 4 test/dependency issues | Plan not executed; SQL untested until Task runs |

Historical rationale and session records live under [Knowledge/](Knowledge/START-HERE.md). Update this handoff after every meaningful milestone; keep it concise.
