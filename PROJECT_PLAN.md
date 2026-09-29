# Coffee business roadmap

Created: 2026-09-28. Updated: 2026-09-30. This file is the authoritative plan and sequence.

## Goal and scope

- Goal: Automate operations for a condo-balcony café in Singapore so the partner only makes coffee, attends to pre-orders, orders beans when alerted, checks cost/revenue, and posts a weekly AI-built Instagram carousel. Design: `Knowledge/Projects/Operations automation architecture.md`.
- In scope: customer pre-order page; counter tablet transaction entry; Supabase ERP; Telegram bot (partner inbox + customer notifications); bean reorder alerts; cost/revenue reporting; insights layer; weekly carousel agent using her real photos.
- Out of scope (v1): payment providers or card payments; automatic bean ordering; AI-generated food imagery; "ask the ERP" chat queries; loyalty stamp card; brand concept work (partner owns it).
- Constraints: PayNow and cash only; low running cost; maintained by the owner with Claude Code/Codex.

## Stages and acceptance gates

| Stage | Work | Evidence needed to complete | State |
|---|---|---|---|
| 1 | Define goal, scope, constraints | Owner answers recorded in vault | Done 2026-09-29 |
| 2 | Architecture design and written spec | Sections 1–3 approved; spec committed and owner-reviewed | In progress (section 1 presented) |
| 3 | Implementation plan | Plan document approved by owner | Not started |
| 4 | Foundation: Supabase schema, Next.js app, Telegram bot skeleton | Migrations applied; bot replies; tests pass | Not started |
| 5 | Orders: order page, payment-proof upload, Telegram verify/reject, counter tablet | End-to-end test order verified and rejected on a test bot | Not started |
| 6 | Beans, finance, insights, weekly digest | Forecast and P&L match hand-calculated test data | Not started |
| 7 | Carousel agent | Partner approves a generated carousel from real photos | Not started |
| 8 | Pilot with residents | One week of live use; issues logged | Not started |

## Decision rules

- Numbers used for decisions are computed in SQL; AI only explains them or uses them as inputs.
- The project owner approves scope changes; approvals are recorded in `Knowledge/Projects/Scope decisions.md`.
- Stages 5–7 depend on stage 4; stage 7 depends on the partner's brand kit and on the insights from stage 6.

## Open planning questions

See `Knowledge/Projects/Open questions.md` (premises rules, PayNow proxy, customer Telegram adoption, IG Business account, brand kit).

For current progress and the immediate next action, read [PROGRESS.md](PROGRESS.md).
