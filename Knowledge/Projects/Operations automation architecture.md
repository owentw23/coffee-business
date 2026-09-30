---
tags: [project, architecture, automation]
status: developing
created: 2026-09-30
modified: 2026-09-30
source: owner-conversation-2026-09-29
---
# Operations automation architecture

Design in progress (brainstorming, architectural path). Agreed choices are logged in [[Projects/Scope decisions]]; unresolved items in [[Projects/Open questions]]. Session record: [[Projects/Sessions/2026-09-30-automation-architecture]].

The approved design is explained for the café owner in `docs/proposal/Cafe-Operations-System-Proposal.docx`, built from `docs/proposal/proposal-brief.md` and `docs/proposal/architecture.svg` on 2026-09-30. See [[Projects/Sessions/2026-09-30-owner-proposal]] for the document checks and remaining review.

## Goal

The partner (barista) should only need to: make coffee, attend to pre-orders, order beans when told, check cost/revenue, and post a weekly AI-built Instagram carousel. Everything else is automated.

## Business context (owner-confirmed)

- Physical café on the first-floor balcony of a condo complex in Singapore, plus pre-orders. Customers are mainly residents.
- Payments: PayNow (personal proxy) and cash only. No cards, no payment provider.
- Premises rules (URA/MCST) settled externally.
- SFA licence already obtained (owner statement, 2026-09-29; not independently verified).

## Principle: "SQL computes, AI explains"

Only the carousel needs creative judgment. Bean quantities, costs, revenue, and every metric are computed deterministically in Postgres. AI never produces a number used for decisions; it narrates computed numbers and uses them as inputs.

## Section 1 — overall architecture (approved 2026-09-30 with change)

Code-first stack: Next.js (Vercel) + Supabase (Postgres, Storage, Cron, Edge Functions) + one Telegram bot + Claude API. n8n was considered and rejected for v1 (see Scope decisions).

Components:
1. **Customer order page** (web, also usable as a Telegram Mini App): menu → cart → pickup slot → exact-amount PayNow QR → upload payment screenshot → "Order confirmed". Collects name and mobile number so the partner can contact the customer. (Earlier proposal of a Telegram deep link for customers dropped 2026-09-30; customers never interact with the bot.)
2. **Counter tablet** (same app, PIN-protected): walk-in transaction entry into the ERP. Tap items → exact-amount PayNow QR or Cash → save. No payment double-confirmation. Also lists today's pre-orders by slot with a Ready button.
3. **Admin**: menu/prices, recipes (grams of beans/milk per drink), bean catalogue, expenses, cost/revenue report.
4. **Telegram bot**: partner gets each pre-order with its screenshot and ✅ Verify / ❌ Reject buttons. Reject → order marked rejected and the bot shows the customer's number with a tap-to-WhatsApp link; the partner messages them herself. Verify → nothing sent (page already said confirmed). The bot only talks to the partner (and the owner for system alerts). Partner sends weekly photos to the bot for the carousel.
5. **Jobs**: daily bean forecast (SQL, alert only when needed); weekly cost/revenue digest; weekly carousel agent (the only AI agent).

## Insights layer (proposed 2026-09-29, owner-endorsed carousel link)

Capture from day one (cannot be backfilled): timestamped sales with items, payment method and channel; recipes and ingredient costs; bean purchases; pre-order customer Telegram IDs; pickup slot load; rejected proofs and no-shows; Instagram post insights; tracked IG → order-page link; optional data.gov.sg weather.

Uses: peak hours, item mix, average ticket, **margin per drink**, bean usage variance (theoretical vs purchased), repeat rate and lapsed regulars, carousel reach vs following-week orders.

AI roles:
- Weekly Telegram digest: exact numbers + 3–4 lines of AI commentary and one suggestion.
- **Feeding the carousel agent (owner explicitly asked to keep this, 2026-09-29):** the carousel agent reads the same computed insights (best seller, slow day, new item, regulars) to choose what each week's carousel features.
- Later, not v1: "ask the ERP" in Telegram (read-only text-to-SQL); Telegram loyalty stamp card for walk-ins.

Limits: walk-ins stay anonymous by design; IG insights need a Business/Creator account linked to a Facebook Page.

## Beans (agreed)

Alert only. Based on weekly demand per bean (from recipes × sales), output a list of beans and quantities to reorder ("beans low, reorder ~X kg"). She orders online herself. No automatic ordering.

## Carousel (agreed 2026-09-29; superseded 2026-09-30 by the content agent below)

Original: all imagery is her real photos; AI designs layout (HTML templates rendered in code), weekly single carousel. Superseded because the owner wants a weekly content calendar, Canva-made layouts, photo briefs, and an inspiration library.

## Dependencies

- Accounts: Vercel, Supabase, Telegram bot (@BotFather), Anthropic API key, optional domain; Instagram Business/Creator + Facebook Page for insights.
- Libraries (planned): `next`, `@supabase/supabase-js`, `grammy`, PayNow EMVCo QR generator, `@anthropic-ai/sdk`, `playwright` or `satori` for slide rendering, `sharp`.
- From the partner: brand kit (fonts, colours, logo), menu and recipes, bean suppliers and prices, PayNow number/UEN.

## Remaining design sections

- Section 2: data model; bean forecast, finance, and insights logic.
- Section 3: carousel agent pipeline.
- Then: written spec, self-review, owner review, implementation plan.

## Section 2 — data model and logic (approved 2026-09-30)

Tables: `menu_items`, `modifiers`, `ingredients`, `recipe_lines`, `modifier_lines`, `orders`, `order_items` (price and cost snapshot at sale), `customers` (pre-order name + mobile), `stock_movements` (append-only ledger), `purchases`, `expenses` (incl. recurring), `weekly_insights` (shared by the digest and the carousel agent), `settings`, `job_runs`. Bean forecast, finance and insights are SQL. Regular = 3+ pre-orders; gone quiet = 14 days. Scrutiny and proposed amendments: [[Resources/Similar systems review]].

## Payment proof check (agreed 2026-09-30)

Vision model (small, e.g. Haiku-class) reads the screenshot into fixed fields; deterministic code checks: amount equals order total; recipient matches her PayNow name; timestamp within ~30 min of order; reference contains order code; transaction ID never seen before; extraction confident. Full pass → auto-verified, FYI to partner. Any failure → ⚠️ to partner with the reason and a tap-to-WhatsApp link. Limit: cannot prove funds arrived; weekly PayNow total vs bank app is the backstop.

## Backups (agreed 2026-09-30)

Supabase Free + nightly encrypted database export (scheduled GitHub Action) to private storage; monthly restore test. IRAS requires 5-year record retention.

## Section 3 v2 — content agent via Canva API (2026-09-30; superseded same day by v3)

- **Inputs via Telegram, any time:** inspiration (images, links, screenshots) → inspiration library with AI notes (layout, hook, colour, why it works), style reference only, never copied; text ideas/news → idea backlog; photos → photo library or matched to an open photo brief. Ambiguous image → bot asks 💡 Inspiration / 📸 For a post.
- **Weekly content calendar (Sunday, with digest):** 3–4 posts/week (owner, 2026-09-30). Each post: day, concept, message, goal from `weekly_insights`, format, Canva template, slide copy, caption, hashtags. Partner approves or edits in chat.
- **Photo briefs:** reuse library photos where they match; otherwise specific shot requests with deadline. Vision check of submitted photos against the brief; minor edits only (crop, exposure, white balance, straighten). No generative imagery of products.
- **Rendering in Canva:** 6–10 Canva brand templates designed once by the partner; registered with field limits. Canva Connect Autofill (requires Canva Pro+; owner confirms Pro/Teams) fills text and image fields; export API returns one PNG per page. Vision check for overflow/crops. Delivery the evening before or morning of post day: full-quality files, caption, and an "Open in Canva" link for manual tweaks. Partner taps Posted; IG metrics later.
- **Rules:** prices/items from the menu DB; never name customers; only her real photos in posts; inspiration is style-only.
- **Feasibility spike required before build:** unreviewed Connect integration usable on her account; multi-page template autofill; unattended refresh-token use. Fallback: "fill kit" (template link + text + prepared photos) or Canva Bulk Create CSV.

## Section 3 v3 — Cowork content agent (approved 2026-09-30)

Owner proposed and approved: the content agent runs as a **Claude Cowork scheduled task on the partner's own paid Claude account** with the **Canva connector**. The ERP writes a weekly marketing brief (insights, menu/prices, promos, new beans, post performance) to Google Drive; the task reads it plus Drive `Inspiration/` and `Photos/` folders and a brand guide, plans 3–4 posts, generates Canva designs, lists photo requests, and the partner revises by chatting. Her Claude never gets database access. Supersedes v2 because it removes the Canva API integration and the code-based planning and rendering pipeline. Spike before build: brand consistency on Pro/Teams (third-party sources say connector brand kit/templates are Enterprise-only), placing her photos into designs, cloud scheduled run with the Canva and Drive connectors. Full spec: `docs/superpowers/specs/2026-09-30-operations-automation-design.md`.

## Customer ordering via Tally (agreed 2026-09-30)

Replaces the custom order page. `tally-webhook` (signature check, price recomputation from the ERP, hours check, screenshot copy, background proof check) and `tally-sync` (rebuilds pickup days and times, menu and open/closed status via `PATCH /forms/{id}`) replace `place-order` and `submit-proof`. Hours and menu are controlled in admin or on Telegram. A Tally API spike comes first; fallback is manual Tally edits caught by the webhook checks. Details in the spec, section 5.1.
