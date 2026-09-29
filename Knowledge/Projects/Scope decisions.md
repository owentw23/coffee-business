---
tags: [project, decisions]
status: seed
created: 2026-09-28
modified: 2026-09-30
source: project-setup
---
# Scope decisions

Record **agreed** scope choices with date, who decided, rationale, and evidence. Keep proposals explicitly separate. Link to the current [roadmap](../../PROJECT_PLAN.md) rather than copying its status.

## Agreed

- 2026-09-28, project owner: host the project and its memory vault together in one **private** GitHub repository, `owentw23/coffee-business`, and share it by inviting collaborators. Rationale: anyone with access gets the Obsidian memory with a single clone. Evidence: [[Projects/Sessions/2026-09-28-github-repo]].
- 2026-09-29, project owner: first build target is operations automation so the partner only makes coffee, handles pre-orders, orders beans when alerted, checks cost/revenue, and posts a weekly AI-built IG carousel. Design: [[Projects/Operations automation architecture]].
- 2026-09-29, project owner: physical condo-balcony café in Singapore plus pre-orders; PayNow and cash only.
- 2026-09-29, project owner: code-first stack (Next.js + Supabase + Telegram bot + Claude API) over n8n. Rationale: order page, counter tablet, and slide rendering must be code anyway; one repo is testable, versioned, and maintainable by Claude Code/Codex.
- 2026-09-29, project owner: no payment provider (Stripe/HitPay rejected). Personal/business PayNow QR with the exact amount; tablet records walk-in sales as PayNow or Cash without double confirmation.
- 2026-09-29, project owner: pre-orders via a simple order web page; customer uploads a PayNow screenshot, sees "confirmed"; partner verifies in Telegram. ~~On reject the bot messages the customer that proof was not verified.~~ Superseded 2026-09-30: the partner messages rejected customers herself.
- 2026-09-29, project owner: Telegram bot is the partner's single inbox (orders, bean alerts, digests, carousel).
- 2026-09-29, project owner: bean automation is alert-only — list of beans and reorder quantities from weekly demand per bean; she orders online herself.
- 2026-09-29, project owner: carousel uses only her real photos; AI designs layout, text/info slides, scrapbook style, and captions. Partly superseded 2026-09-30: layouts come from Canva templates, and the agent plans a weekly content calendar (see below).
- 2026-09-29, project owner: computed business insights must feed the carousel agent.
- 2026-09-30, project owner: section 1 architecture approved with one change — the bot does not message customers. The partner contacts rejected customers herself, so customers do not need Telegram; the order page collects name and mobile number instead.
- 2026-09-30, project owner: receive payments via personal PayNow (not UEN).
- 2026-09-30, project owner: MCST/URA premises rules are settled externally; not a project concern.
- 2026-09-30, project owner: section 2 data model approved. "Regular" = 3+ pre-orders; "gone quiet" = no order in **14 days** (changed from proposed 21). Partner accepts tapping "Ordered as suggested" / "Arrived" and entering bean prices.
- 2026-09-30, project owner: backups via Supabase Free + nightly encrypted export (option B), not Supabase Pro. Daily jobs keep the project active; monthly restore test.
- 2026-09-30, project owner: payment proofs checked automatically — a vision model extracts amount, recipient, time, reference, transaction ID; code compares with the order. **Full pass = auto-verified** (FYI to partner); only flagged proofs need her action. Weekly PayNow total check remains the backstop against edited screenshots. Supersedes the "partner verifies every proof" step for clean proofs.
- 2026-09-30, pending confirmation at spec review: review amendments #1 (bean rest days in lead time), #2 (waste allowance, 3%/5% variance bands), #3 (IG views/reach/saves/shares), #4 (reuse PayNow QR library, test on bank apps), #7 (rush-speed counter, manual fallback). Owner did not object when they were presented.
- 2026-09-30, project owner: marketing becomes a content agent — weekly calendar of 3–4 posts, each with a concept and a Canva-template layout; agent sends photo briefs when real photos are needed; partner can send inspiration references and text ideas to the agent. Canva Pro/Teams is already available.
