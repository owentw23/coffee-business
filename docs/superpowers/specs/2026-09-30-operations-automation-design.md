# Operations automation — design spec

- Date: 2026-09-30
- Status: draft for owner review
- Decisions and rationale: `Knowledge/Projects/Scope decisions.md`, `Knowledge/Projects/Operations automation architecture.md`, `Knowledge/Resources/Similar systems review.md`

## 1. Purpose and success criteria

A condo-balcony café in Singapore (walk-ins + pre-orders, PayNow and cash only). The partner (barista) should only need to:

1. make coffee;
2. attend to pre-orders;
3. order beans when alerted;
4. check cost and revenue;
5. post 3–4 AI-prepared Instagram posts a week.

Success means:
- every sale is recorded;
- pre-order payment proofs are checked without her effort unless something looks wrong;
- bean alerts arrive early enough to cover delivery and resting;
- weekly numbers need no spreadsheet work;
- posts are ready to publish after a short chat review.

## 2. Constraints

- Payments: personal PayNow (exact-amount QR) and cash. No payment provider, no cards.
- Licensing and premises: SFA licence held; URA/MCST settled externally (owner-reported).
- Budget: lean. Free tiers where they permit commercial use; pay-per-use AI.
- Builder/maintainer: the owner with Claude Code / Codex. The partner is non-technical.
- Records: IRAS requires business records to be kept for 5 years.
- Privacy: PDPA. Collect only the customer name and mobile, for pre-orders only.

## 3. Principles

1. **SQL computes, AI explains.** Every number used for decisions (stock, reorder quantities, costs, revenue, metrics) is computed deterministically. AI reads screenshots, narrates numbers, and creates marketing content, and nothing else.
2. **One source of truth.** All sales, stock and money live in one Postgres database.
3. **The shop never depends on the software.** A manual fallback always exists.
4. **She only acts on exceptions.** Clean cases are automatic; problems come to her Telegram.

## 4. Architecture

```
Customers ──► Order page ─┐                       ┌─► Telegram bot ──► Partner (orders ⚠️, beans, digest)
                          │                       │                └─► Owner (system alerts)
Partner ────► Counter PWA ├─► Supabase ("ERP") ───┤
Owner/Partner ► Admin ────┘   Postgres · Storage  ├─► Weekly marketing brief ─► Google Drive
                              Edge Functions      │                                   │
                              Cron                └─► Nightly encrypted export        ▼
                                                                    Cowork scheduled task (partner's Claude)
                                                                    + Canva connector → 3–4 posts/week
                                                                    ◄─ partner chats to revise
```

### Hosting decision (correction, 2026-09-30)
Vercel's free Hobby plan forbids commercial use, so it is not used. The web app is a **static single-page app** (React + Vite, installable PWA) on **Cloudflare Pages (free tier)**. All server-side logic runs in **Supabase Edge Functions**, which removes the need for a Next.js server. The owner confirms this at spec review.

## 5. Components

### 5.1 Customer order page (public)
Flow:
1. menu;
2. cart (with modifiers);
3. pickup slot (only slots with remaining capacity);
4. name and mobile, with a one-line PDPA purpose note;
5. exact-amount PayNow QR with the order code (e.g. `BK-0412`) to type as the reference;
6. upload a screenshot of the payment;
7. "Order confirmed".

The customer never interacts with Telegram. The page writes only through an Edge Function; it has no direct table access.

### 5.2 Counter tablet PWA (PIN-protected)
- Large buttons for the top 6 drinks, one-tap modifiers, a running total.
- **PayNow** shows the exact-amount QR; **Cash** saves immediately. There is no payment double confirmation.
- A today's-pre-orders panel, sorted by pickup slot, with Ready / Collected / No-show buttons.
- **Offline-first:** sales are stored locally with client-generated UUIDs and synced idempotently when the connection returns.
- **Manual fallback:** a printed static PayNow QR plus a paper tally, entered later through a back-dated entry screen.

### 5.3 Admin (owner and partner)
- Menu, prices and modifiers.
- Recipes, and ingredients with costs.
- Bean catalogue: roaster, bag size, delivery lead time, rest days.
- Expenses, including recurring ones.
- Promotions (the only source of promos for marketing).
- Settings: hours, slot length and capacity, PayNow name and number.
- Reports: weekly and monthly P&L, item margins, insights.
- Optional stock count entry.

### 5.4 Telegram bot (partner and owner only)
- **Pre-order proof results:**
  - **clean proof:** an FYI only;
  - **flagged proof:** a ⚠️ with the reason, the screenshot, and a tap-to-WhatsApp link to the customer. She taps **Accept anyway** or **Reject**, and messages the customer herself.
- **Bean alerts:** the reorder list, plus **Ordered as suggested** and **Arrived** buttons (optional roast date on Arrived).
- **Sunday digest:** exact numbers plus 3–4 lines of AI commentary with one suggestion. Includes the PayNow total for the week, to compare with her bank app.
- **Owner-only:** job failure alerts.

### 5.5 Payment proof check
1. A small vision model extracts: amount, recipient name, masked number, date/time, reference, transaction ID, bank, and a confidence value.
2. Code compares the extracted fields with the order. All of these must pass:
   - amount = order total;
   - recipient matches the configured PayNow name;
   - time within 30 minutes of order creation;
   - reference contains the order code;
   - transaction ID not seen before;
   - confidence above the threshold.
3. **Full pass → `verified` automatically. Any failure → `flagged` to the partner.**

Limit: this cannot prove funds arrived. The weekly PayNow total check is the backstop.

### 5.6 Scheduled jobs (Supabase Cron + Edge Functions)
All times are Asia/Singapore.

| Job | When | Output |
|---|---|---|
| Bean forecast | Daily, 07:00 | Telegram alert only when needed |
| Recurring expenses | 1st of month | Expense rows |
| Weekly insights | Sunday 17:00 | A `weekly_insights` row |
| Digest | Sunday 17:30 | Telegram message |
| Marketing brief | Sunday 18:00 | Markdown/JSON brief to a Google Drive folder |
| Nightly export | 02:00 (GitHub Action) | Encrypted database dump to private storage; also keeps the Free project active |
| IG metrics pull | Daily (after the IG Business account exists) | Metrics rows |

Every run writes a `job_runs` row. Failures alert the owner.

### 5.7 Content agent (Cowork scheduled task on the partner's Claude account)
- **Inputs, read through connectors:**
  - the weekly marketing brief from Drive: insights, menu with prices, active promos, new beans, recent post performance;
  - the Drive folders `Inspiration/` and `Photos/`, which she can add to from her phone at any time;
  - a brand guide document (brand kit, tone of voice, reference designs).
- **Schedule:** Sunday 20:00, after the brief.
- **The task:**
  1. plans 3–4 posts for the week (day, concept, message, goal);
  2. generates Canva designs (4:5 social posts) for posts that already have suitable photos;
  3. lists specific photo requests for the gaps.
- **Revisions:** she opens the task and chats ("more minimal", "use this photo") and uploads the requested photos, and Claude updates the designs. She posts from Canva or from the exported images.
- **Rules in the task instructions** (versioned in this repo under `marketing/`):
  - prices and items only from the brief;
  - promos only from the brief;
  - never name customers;
  - only her real photos in posts (minor edits only);
  - inspiration is style reference only, never copied.
- **No database access.** Her Claude reads only the brief.
- **Unknowns, tested by a spike before build:**
  - brand consistency on her Canva Pro/Teams plan (third-party sources say brand kit and brand templates via the connector are Enterprise-only);
  - whether Claude can place her own photos into generated designs;
  - a cloud scheduled run with the Canva and Drive connectors.
- **Fallback:** she places photos in Canva herself via "Open in Canva"; consistency comes from reference designs and the brand guide.

## 6. Data model (Postgres)

| Table | Key columns / notes |
|---|---|
| `menu_items` | name, category, price_cents, active, sort_rank |
| `modifiers` | name, price_delta_cents |
| `ingredients` | name, kind (bean/milk/packaging/other), unit (g/ml/pc), avg_cost_micros. Beans also have: roaster, bag_size_g, lead_time_days, rest_days, waste_allowance_pct |
| `recipe_lines` / `modifier_lines` | menu_item_id or modifier_id, ingredient_id, qty |
| `customers` | name, mobile (unique), created_at. Pre-orders only |
| `orders` | id (client UUID), code, channel (walk_in/pre_order), status, payment_method (paynow/cash), total_cents, customer_id, pickup_slot, proof_path, proof_check jsonb, source (e.g. `ig`), created_at |
| `order_items` | order_id, menu_item_id, qty, modifiers, **unit_price_cents and unit_cost_cents snapshot at sale** |
| `stock_movements` | append-only: ingredient_id, qty (±), reason (purchase/sale/waste/count_adjust), ref, at |
| `purchases` | ingredient_id, qty, bags, cost_cents, status (ordered/arrived), roast_date, ordered_at, arrived_at |
| `stock_counts` | ingredient_id, counted_qty, at (optional, drives variance) |
| `expenses` | date, category, amount_cents, note, recurring |
| `promos` | text, starts, ends |
| `weekly_insights` | week_start, payload jsonb (shared by the digest and the marketing brief) |
| `ig_posts` | ig_media_id, posted_at, views, reach, saves, shares (later) |
| `settings`, `job_runs` | config; job audit |

Order statuses:
- pre-order: `submitted` → `verified` or `flagged` → (`verified` or `rejected`) → `ready` → `collected` or `no_show`;
- walk-in: `completed`.

Revenue counts `completed`, `verified`, `ready` and `collected`.

## 7. Business logic

### 7.1 Bean forecast (per bean, daily)
- `daily_use` = theoretical grams used (sales × recipe) averaged over the last 14 opening days; a manual estimate is used until 14 days of history exist.
- `effective_lead` = `lead_time_days` + `rest_days` (default rest 7 days).
- Alert when `stock / daily_use` ≤ `effective_lead` + 3 safety days and there is no open purchase.
- `order_qty` = `daily_use` × (`effective_lead` + 14) − stock − on_order, rounded up to whole bags, **capped at 21 days of use** (freshness: best within ~5 weeks of roast).
- Message: beans sorted by urgency, each with kg, bags, days left, and share of demand.

### 7.2 Finance
- Revenue: counted order statuses.
- COGS: Σ qty × `unit_cost_cents`. Unit cost = recipe × ingredient average cost, with bean cost × (1 + waste allowance, default 5%).
- Gross margin, overall and per item. Net = gross − expenses.
- PayNow total per week, for the bank-app check.

### 7.3 Insights (views → `weekly_insights`)
- Sales by hour and weekday; item mix; average ticket; pre-order share.
- Item margin with a popular-vs-profitable matrix.
- Slot load; no-shows; flagged proofs.
- Repeat rate; **regular = 3+ pre-orders; gone quiet = none in 14 days**.
- Bean variance (when counts exist): <3% ok, 3–5% watch, >5% investigate.
- Later: IG views, reach, saves and shares; weather.

## 8. Reliability
- Idempotent writes: client UUIDs, and Telegram update IDs deduplicated.
- The job audit table records every run, and the owner is alerted on failure.
- Offline counter plus the paper fallback.
- Nightly encrypted export with a **monthly restore test** logged in `job_runs`.
- Daily jobs prevent the Supabase Free inactivity pause.

## 9. Security and privacy
- Row-level security on all tables. The public page can only call the order Edge Function.
- Proofs are stored in a private bucket, reachable only through signed URLs from the bot and admin.
- Secrets live in Supabase / Cloudflare / GitHub secret stores only; `.env` stays git-ignored.
- Customer data is minimal (name and mobile), stated as used only for order contact. Export and deletion on request.
- The partner's Claude sees only the marketing brief, never the database.

## 10. Testing
- **Unit:**
  - bean forecast (including rest days, cap and bag rounding);
  - COGS and margin;
  - proof comparison rules;
  - PayNow QR payload.
- **Integration:**
  - order Edge Function (RLS denies direct writes);
  - Telegram webhook with recorded updates;
  - offline sync idempotency.
- **Fixtures:** hand-calculated test weeks for the P&L and forecast; recorded vision outputs for proof checks.
- **Manual before launch:**
  - scan the PayNow QR with the DBS, OCBC and UOB apps;
  - run 20 test orders end to end on a test bot.
- **Coverage target:** 80%. The TDD workflow applies.

## 11. Dependencies and running cost
| Item | Cost |
|---|---|
| Supabase Free | S$0 |
| Cloudflare Pages Free | S$0 |
| Telegram bot | S$0 |
| GitHub Actions (backup) | S$0 |
| Google Drive | S$0 |
| Anthropic API (proof checks and digest commentary) | est. a few S$/month; confirm pricing at build time |
| Partner's paid Claude plan | subscription |
| Canva Pro/Teams | already held |
| Domain | optional |

Libraries (planned):
- `react`, `vite`, `vite-plugin-pwa`, `@supabase/supabase-js`;
- a PayNow QR library (e.g. `@chewhx/paynowqr`), `qrcode`;
- `grammy`, `@anthropic-ai/sdk`.

## 12. Build order
1. **Foundation:** schema, RLS, admin (menu, recipes, beans), Telegram bot skeleton.
2. **Counter PWA** plus the offline and fallback flows. *Walk-in data capture starts here.*
3. **Pre-orders:** order page, proof check, bot flows.
4. **Beans, finance, insights, digest.**
5. **Canva/Cowork spike**, then brief export, task instructions, and a Cowork task set up with the partner.
6. **Pilot week** with residents.

Until step 3 ships, pre-orders may run on Take App's free plan (50 orders/month).

## 13. Out of scope (v1)
- Payment providers and cards.
- Auto-ordering beans.
- AI-generated product imagery.
- "Ask the ERP" chat.
- Loyalty stamp card.
- Auto-publishing to Instagram.
- Customer notifications.

## 14. Open items at review
1. Confirm review amendments #1 (rest days), #2 (waste allowance and variance bands), #3 (IG metrics), #4 (PayNow QR library and bank tests) and #7 (rush-speed counter and manual fallback). All are included above.
2. Confirm the hosting change: static PWA on Cloudflare Pages + Supabase Edge Functions instead of Next.js on Vercel.
3. Pending from the partner: brand kit, Instagram Business account, a paid Claude plan.
