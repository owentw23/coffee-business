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
Customers ──► Tally form ─┐ (webhook)             ┌─► Telegram bot ──► Partner (orders ⚠️, beans, digest)
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
Vercel's free Hobby plan forbids commercial use, including a staff-only business dashboard, so it is not used. The staff web app (counter and admin) is a **static single-page app** (React + Vite) on **Cloudflare Pages (free tier)**. All server-side logic runs in **Supabase Edge Functions** in the **Singapore region**. Customers never use our web app: they order through a **Tally form** (owner decision, 2026-09-30).

Speed: the app loads once and then runs on the tablet. Only "save sale" crosses the network, to Supabase in Singapore, and the UI never waits for it.

### Technical architecture (added 2026-09-30)

| Layer | Choice |
|---|---|
| Staff web app | React + Vite + TypeScript, installable PWA. Routes: `/counter` and `/admin` (staff only). No offline mode: home Wi-Fi is always available (owner, 2026-09-30); saves are optimistic with automatic retry. |
| Customer ordering | Tally form (free plan). Its content is generated from the ERP through the Tally API (`PATCH /forms/{id}`). |
| Web hosting | Cloudflare Pages, free tier. Serves static files only. |
| Backend | Supabase: Postgres, Auth, REST API with row-level security, Edge Functions (Deno/TypeScript), Storage, `pg_cron` + `pg_net`. |
| Messaging | Telegram Bot API via webhook, using `grammy`. |
| AI | Anthropic API, called only from Edge Functions: payment-proof field extraction and digest commentary. |
| Marketing hand-off | Google Drive API, via a service account with access to a single `Marketing/` folder. |
| Backups | GitHub Actions: `pg_dump`, encrypted with `age` (public key), uploaded to Cloudflare R2 (free tier). The private key is kept offline by the owner. A weekly job also copies the proofs bucket through Storage's S3-compatible endpoint. |
| CI | GitHub Actions: typecheck, unit and integration tests (local Supabase via CLI) on every push. |

**Server entry points:**
- `tally-webhook`: receives each Tally submission.
  - Verifies Tally's webhook signature.
  - Maps answers to menu items by stable block IDs and **recomputes the price from the ERP**. A mismatch with the Tally total, or an unknown item, flags the order and alerts the owner.
  - Checks the pickup time against the opening schedule.
  - Upserts the customer, inserts the order and assigns an order code.
  - Copies the screenshot from Tally into the private proofs bucket.
  - Responds quickly, then runs the proof check in the background (`EdgeRuntime.waitUntil`): Anthropic vision extraction, the rule checks, the status update, and the Telegram notice.
- `tally-sync`: rebuilds the Tally form from the ERP and pushes it with `PATCH /forms/{id}`.
  - Pickup-day dropdown: the next 7 open days.
  - One conditional time dropdown per day: 15-minute times within that day's hours.
  - Menu: quantity fields for active pre-order items, a calculated total, and the static PayNow QR image.
  - Form status: closed outside opening hours, with a "next opening" message.
  - Triggered when admin saves the menu or hours, by Telegram hour and menu commands, and by a 15-minute job that removes past times. It skips the push when nothing has changed.
- `telegram-webhook`:
  - Verifies the `X-Telegram-Bot-Api-Secret-Token` header.
  - Accepts only allowlisted chat IDs (the partner and the owner).
  - Handles the buttons: Accept anyway, Reject, Ordered as suggested, Arrived, Ready.
  - Handles the hours and menu controls: the morning check-in (Yes / Change hours / Closed today), Pause pre-orders, Close early, Reopen, Upcoming days, and Menu today (sold-out toggles). Each one triggers `tally-sync`.
- `run-job`: a single function that takes a `job` parameter (`bean_forecast`, `weekly_insights`, `digest`, `marketing_brief`, `recurring_expenses`, later `ig_metrics`).
  - Called by `pg_cron` through `pg_net`, carrying a shared job secret stored in Supabase Vault.
  - Every run writes to `job_runs`, and failures alert the owner.
- `record_sale(payload)`: a Postgres function (RPC) called by the counter for each sale.
  - It **snapshots unit price and cost on the server** and inserts the matching `stock_movements`.
  - It is idempotent on the client UUID.

**Auth model:**
- **Customers:** they interact only with Tally. Our only public endpoint is `tally-webhook`, which accepts only requests carrying a valid Tally signature.
- **Staff:** Supabase Auth accounts with roles in a `staff_roles` table: owner = `admin`, partner = `staff`.
  - The counter tablet stays signed in as staff, behind a local PIN lock screen.
  - RLS grants staff read and write on operational tables; admin-only on settings and expenses.
- **Telegram:** a chat-ID allowlist stored in `settings`.
- **Partner's Claude:** no Supabase access. It reads only the Drive `Marketing/` folder.

**Secrets:**
- Supabase secrets: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`, `ANTHROPIC_API_KEY`, `GOOGLE_SERVICE_ACCOUNT_JSON`, `JOB_SECRET`, `TALLY_API_KEY`, `TALLY_SIGNING_SECRET`.
- GitHub secrets: `SUPABASE_DB_URL`, `AGE_PUBLIC_KEY`, `R2_*`.
- Nothing secret is stored in the repo.

**Environments:**
- Local: Supabase CLI plus the Vite dev server, with a separate test Telegram bot.
- Production: one Supabase project and one Pages project.
- Schema changes go only through `supabase/migrations/`.

**Repo layout (planned):**
- `apps/web/`
- `supabase/migrations/`
- `supabase/functions/{tally-webhook,tally-sync,telegram-webhook,run-job,_shared}/`
- `marketing/`: the Cowork task instructions and the brand guide.
- `.github/workflows/{ci,backup}.yml`

## 5. Components

### 5.1 Customer ordering: Tally form (owner decision, 2026-09-30)
Linked from the Instagram bio and the condo chat.

Flow:
1. menu quantities, with a calculated total;
2. pickup day (next 7 open days), then a pickup time within that day's hours;
3. name and mobile, with a one-line PDPA purpose note;
4. "Your total is S$X": the static PayNow QR image; the customer enters the amount and **their mobile number as the reference**;
5. upload a screenshot of the payment;
6. Tally shows "Order confirmed".

**No pickup capacity limits:** any time within opening hours is allowed (owner, 2026-09-30).

**Pre-order menu:** Tally has no per-item add-ons, so variants such as Oat latte or Iced latte are separate items. Admin flags which items appear on pre-order.

**She never edits Tally.** The ERP is the source of truth, and `tally-sync` rewrites the form, so opening hours and menu are changed only in admin or through Telegram.

**Fallback:** if the Tally API cannot handle the calculated total and conditional blocks (spike at build time), she edits Tally by hand; the webhook's price recomputation and hours check catch any drift.

**Privacy:** screenshots also sit on Tally's servers. The form's purpose note covers this.

The customer never interacts with Telegram or with our web app.

### 5.2 Counter tablet PWA (PIN-protected)
- Large buttons for the top 6 drinks, one-tap modifiers, a running total.
- **PayNow** shows the exact-amount QR; **Cash** saves immediately. There is no payment double confirmation.
- A today's-pre-orders panel, sorted by pickup slot, with Ready / Collected / No-show buttons.
- **Optimistic saves:** each sale gets a client-generated UUID and shows "✓ saved" at once. It retries in the background until confirmed and is idempotent on the UUID. No offline mode, because home Wi-Fi is always on.
- **Manual fallback:** a printed static PayNow QR plus a paper tally, entered later through a back-dated entry screen.

### 5.3 Admin (owner and partner)
- Menu, prices and modifiers.
- Recipes, and ingredients with costs.
- Bean catalogue: roaster, bag size, delivery lead time, rest days.
- Expenses, including recurring ones.
- Promotions (the only source of promos for marketing).
- Settings: the default weekly opening hours, day-by-day overrides for the next 7 days, pickup interval (15 minutes), PayNow name and number, and the static PayNow QR image used on the form.
- Reports: weekly and monthly P&L, item margins, insights.
- Optional stock count entry.

### 5.4 Telegram bot (partner and owner only)
- **Pre-order proof results:**
  - **clean proof:** an FYI only;
  - **flagged proof:** a ⚠️ with the reason, the screenshot, and a tap-to-WhatsApp link to the customer. She taps **Accept anyway** or **Reject**, and messages the customer herself.
- **Bean alerts:** the reorder list, plus **Ordered as suggested** and **Arrived** buttons (optional roast date on Arrived).
- **Sunday digest:** exact numbers plus 3–4 lines of AI commentary with one suggestion. Includes the PayNow total for the week, to compare with her bank app.
- **Opening hours and menu:**
  - A morning check-in: "Opening today 8:00–14:00?" with ✅ Yes / ✏️ Change hours / ❌ Closed today.
  - During the day: Pause pre-orders, Close early, Reopen.
  - Upcoming days: close or change any of the next 7 days.
  - Menu today: tap-to-toggle sold-out items.
  - Each change updates the Tally form within seconds and confirms it ("Form updated ✅ Pickup times now 9:00–11:45").
  - If a change affects existing orders, the bot lists them with tap-to-WhatsApp links.
- **Owner-only:** job failure alerts, and alerts for price mismatches between Tally and the ERP.

### 5.5 Payment proof check
1. A small vision model extracts: amount, recipient name, masked number, date/time, reference, transaction ID, bank, and a confidence value.
2. Code compares the extracted fields with the order. All of these must pass:
   - amount = order total;
   - recipient matches the configured PayNow name;
   - payment time within 30 minutes before the form submission;
   - reference contains the customer's mobile number (at least the last 4 digits);
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
| Nightly export | 02:00 (GitHub Action) | `age`-encrypted database dump to Cloudflare R2; also keeps the Free project active |
| Tally time roll | Every 15 minutes during opening hours | `tally-sync`: remove past times, open or close the form |
| Morning check-in | 30 minutes before default opening | Telegram hours prompt |
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
| `menu_items` | name, category, price_cents, active, on_preorder, sold_out_today, sort_rank, tally_block_id (stable) |
| `modifiers` | name, price_delta_cents |
| `ingredients` | name, kind (bean/milk/packaging/other), unit (g/ml/pc), avg_cost_micros. Beans also have: roaster, bag_size_g, lead_time_days, rest_days, waste_allowance_pct |
| `recipe_lines` / `modifier_lines` | menu_item_id or modifier_id, ingredient_id, qty |
| `customers` | name, mobile (unique), created_at. Pre-orders only |
| `orders` | id (client UUID), code, channel (walk_in/pre_order), status, payment_method (paynow/cash), total_cents, tally_total_cents, customer_id, pickup_at, tally_response_id (unique), proof_path, proof_check jsonb, source (e.g. `ig`), created_at |
| `order_items` | order_id, menu_item_id, qty, modifiers, **unit_price_cents and unit_cost_cents snapshot at sale** |
| `stock_movements` | append-only: ingredient_id, qty (±), reason (purchase/sale/waste/count_adjust), ref, at |
| `purchases` | ingredient_id, qty, bags, cost_cents, status (ordered/arrived), roast_date, ordered_at, arrived_at |
| `stock_counts` | ingredient_id, counted_qty, at (optional, drives variance) |
| `expenses` | date, category, amount_cents, note, recurring |
| `promos` | text, starts, ends |
| `weekly_insights` | week_start, payload jsonb (shared by the digest and the marketing brief) |
| `ig_posts` | ig_media_id, posted_at, views, reach, saves, shares (later) |
| `opening_hours` | weekday, opens, closes (the default week) |
| `opening_overrides` | date, closed, opens, closes, paused (day-level exceptions) |
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
- Pickup-time load; no-shows; flagged proofs.
- Repeat rate; **regular = 3+ pre-orders; gone quiet = none in 14 days**.
- Bean variance (when counts exist): <3% ok, 3–5% watch, >5% investigate.
- Later: IG views, reach, saves and shares; weather.

## 8. Reliability
- Idempotent writes: client UUIDs, and Telegram update IDs deduplicated.
- The job audit table records every run, and the owner is alerted on failure.
- Optimistic counter saves with retry, plus the paper fallback. Tally submissions are idempotent on `tally_response_id`.
- Nightly encrypted export with a **monthly restore test** logged in `job_runs`.
- Daily jobs prevent the Supabase Free inactivity pause.

## 9. Security and privacy
- Row-level security on all tables. The only public endpoint is `tally-webhook`, which checks Tally's signature.
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
  - `tally-webhook` with recorded Tally payloads (bad signature, price mismatch, out-of-hours, duplicate response);
  - `tally-sync` form generation from fixture menus and hours (snapshot of the PATCH body);
  - Telegram webhook with recorded updates;
  - counter save retry idempotency.
- **Fixtures:** hand-calculated test weeks for the P&L and forecast; recorded vision outputs for proof checks.
- **Manual before launch:**
  - scan the PayNow QR with the DBS, OCBC and UOB apps;
  - run 20 test orders end to end through a test Tally form and a test bot, including changing hours and toggling sold-out items.
- **Coverage target:** 80%. The TDD workflow applies.

## 11. Dependencies and running cost
| Item | Cost |
|---|---|
| Supabase Free | S$0 |
| Cloudflare R2 (backups, free tier) | S$0 |
| Cloudflare Pages Free | S$0 |
| Tally Free | S$0 |
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
2. **Counter PWA** plus the fallback flow. *Walk-in data capture starts here.*
3. **Pre-orders:** Tally API spike, then `tally-sync`, `tally-webhook`, proof check, and the hours and menu bot flows.
4. **Beans, finance, insights, digest.**
5. **Canva/Cowork spike**, then brief export, task instructions, and a Cowork task set up with the partner.
6. **Pilot week** with residents.

Until step 3 ships, pre-orders can run on a hand-edited Tally form.

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
2. Confirm the hosting: staff app on Cloudflare Pages, not Vercel Hobby (no commercial use) or Vercel Pro (~USD 20/month). Supabase in Singapore. Also confirm Cloudflare R2 as the backup destination.
3. Pending from the partner: brand kit, Instagram Business account, a paid Claude plan.
