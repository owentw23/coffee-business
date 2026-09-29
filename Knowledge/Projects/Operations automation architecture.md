---
tags: [project, architecture, automation]
status: developing
created: 2026-09-30
modified: 2026-09-30
source: owner-conversation-2026-09-29
---
# Operations automation architecture

Design in progress (brainstorming, architectural path). Agreed choices are logged in [[Projects/Scope decisions]]; unresolved items in [[Projects/Open questions]]. Session record: [[Projects/Sessions/2026-09-30-automation-architecture]].

## Goal

The partner (barista) should only need to: make coffee, attend to pre-orders, order beans when told, check cost/revenue, and post a weekly AI-built Instagram carousel. Everything else is automated.

## Business context (owner-confirmed)

- Physical café on the first-floor balcony of a condo complex in Singapore, plus pre-orders. Customers are mainly residents.
- Payments: PayNow and cash only. No cards, no payment provider.
- SFA licence already obtained (owner statement, 2026-09-29; not independently verified).

## Principle: "SQL computes, AI explains"

Only the carousel needs creative judgment. Bean quantities, costs, revenue, and every metric are computed deterministically in Postgres. AI never produces a number used for decisions; it narrates computed numbers and uses them as inputs.

## Section 1 — overall architecture (presented 2026-09-29)

Code-first stack: Next.js (Vercel) + Supabase (Postgres, Storage, Cron, Edge Functions) + one Telegram bot + Claude API. n8n was considered and rejected for v1 (see Scope decisions).

Components:
1. **Customer order page** (web, also usable as a Telegram Mini App): menu → cart → pickup slot → exact-amount PayNow QR → upload payment screenshot → "Order confirmed" → one-tap "Get updates on Telegram" deep link (required because a bot can only message users who have started it).
2. **Counter tablet** (same app, PIN-protected): walk-in transaction entry into the ERP. Tap items → exact-amount PayNow QR or Cash → save. No payment double-confirmation. Also lists today's pre-orders by slot with a Ready button.
3. **Admin**: menu/prices, recipes (grams of beans/milk per drink), bean catalogue, expenses, cost/revenue report.
4. **Telegram bot**: partner gets each pre-order with its screenshot and ✅ Verify / ❌ Reject buttons. Reject → bot messages the customer "rejected — proof not verified". Verify → no customer message (page already said confirmed). Partner sends weekly photos to the bot for the carousel.
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

## Carousel (agreed)

All imagery is her real photos. AI only designs layout and base: info posts with text, scrapbook-style compositions, captions and hashtags. No AI-generated food imagery. Delivered to her via Telegram ready to post.

## Dependencies

- Accounts: Vercel, Supabase, Telegram bot (@BotFather), Anthropic API key, optional domain; Instagram Business/Creator + Facebook Page for insights.
- Libraries (planned): `next`, `@supabase/supabase-js`, `grammy`, PayNow EMVCo QR generator, `@anthropic-ai/sdk`, `playwright` or `satori` for slide rendering, `sharp`.
- From the partner: brand kit (fonts, colours, logo), menu and recipes, bean suppliers and prices, PayNow number/UEN.

## Remaining design sections

- Section 2: data model; bean forecast, finance, and insights logic.
- Section 3: carousel agent pipeline.
- Then: written spec, self-review, owner review, implementation plan.
