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
- 2026-09-30, project owner: content agent runs as a Claude Cowork scheduled task on the partner's own paid Claude account with the Canva connector, fed by an ERP-generated weekly marketing brief in Google Drive; she revises by chatting. Supersedes the Canva Connect API design proposed earlier the same day.
- 2026-09-30, proposed by Claude, pending owner confirmation at spec review: host the web app as a static PWA on Cloudflare Pages (free, commercial use allowed) with server logic in Supabase Edge Functions, because Vercel Hobby forbids commercial use.
- 2026-09-30, project owner: customers order through a **Tally form** instead of a hosted order page (Tally chosen over Fillout, Youform, OpnForm, Zoho Forms, Jotform and Take App; see [[Resources/Similar systems review]]). No pickup capacity limits; pre-orders up to **7 days ahead**. She never edits Tally: the ERP rewrites the form through the Tally API when hours or menu change (Telegram morning check-in, pause/close/reopen, sold-out toggles). Customers enter the amount themselves on a static PayNow QR and use their mobile number as the reference.
- 2026-09-30, project owner: counter tablet needs no offline mode (home Wi-Fi always on); optimistic saves with retry plus the paper fallback.
- 2026-09-30, project owner (confirmed same day): staff app hosting on Cloudflare Pages (owner preferred Vercel; Vercel Hobby forbids commercial use, Vercel Pro ~USD 20/month). Supabase region Singapore.
- 2026-09-30, project owner: design spec approved, including review amendments #1–4 and #7, Cloudflare Pages hosting, Supabase Singapore, Cloudflare R2 backups. Next: implementation plans and a Word proposal for the owner.
- 2026-10-11, project owner: **business model clarified.** Weekdays: cold brew sold by pre-order only, handed over in person at a pickup time. Weekends: balcony shop open for walk-ins (weekend menu to be decided). Every 2 weeks a new pair of beans (A and B) that customers choose between; in week 1 the beans are undisclosed, in week 2 they are revealed.
- 2026-10-11, project owner: **mystery-bean game.** During week 1, a customer who guesses the tasting notes of the coffee they ordered on the first try gets it free. Guesses are made in person at the counter, as free text. The pre-order flow is only for weekday cold brew (the game is not part of pre-orders).
- 2026-10-11, project owner: weekday pickup is attended (someone hands it over); the unattended cooler/fridge idea is not pursued for now.
- 2026-10-11, project owner: cycles start on **Saturday** (game runs on the debut weekend, days 1–2; reveal on day 8, also a Saturday); cold brew cutoff **6pm the day before**; one try enforced by **mobile number**; the **barista judges** free-text guesses with the official notes and accepted similar words on screen. Spec v2: `docs/superpowers/specs/2026-10-11-operations-automation-design-v2.md`.
- 2026-10-11, project owner: **own website replaces Tally.** Public café website (menu of the week, business info, about us, built-in pre-order form) plus an **all-in-one staff web app on a subdomain** (payments at the counter, orders, dashboard, and control of the website and form), used on a tablet. Supersedes the Tally decisions of 2026-09-30.
- 2026-10-11, project owner: **cold brew** comes black or white; white has regular milk by default, with **oat, Rokeby or soy at +S$1**. Cold brew uses **different (house) beans**; the better beans are reserved for the weekend shop. Working assumption (to confirm): the A/B mystery cycle and game apply to weekend drinks only.
- 2026-10-11, project owner: confirmed **cold brew has no A/B choice** (house bean only); the A/B mystery cycle and game are weekend-only. **Build and draft locally first**; buy the domain and go to production later (Plan 1 v2 Task 11 deferred).

