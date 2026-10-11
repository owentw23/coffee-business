# Operations automation — design spec v2

- Date: 2026-10-11 (revised the same day: own website instead of Tally, all-in-one staff web app, cold brew milk options, separate cold brew beans)
- Status: draft for owner review
- Supersedes: `2026-09-30-operations-automation-design.md` (v1). §4 lists every change from v1.
- Rationale and history: `Knowledge/Projects/Scope decisions.md`, `Knowledge/Projects/Operations automation architecture.md`, `Knowledge/Resources/Similar systems review.md`

## 1. The business

A small coffee business run by two owners from a condo in Singapore.

| When | What is sold | How |
|---|---|---|
| **Weekdays (Mon–Fri)** | **Cold brew, pre-order only** | Ordered on the café website by **6pm the day before**; handed over in person at a pickup time |
| **Weekends (Sat–Sun)** | Balcony shop on level 1 (**menu to be decided**) | Walk-ins; sales taken in the staff web app on a tablet |

**Cold brew**
- **Black** (no milk) or **white** (with milk).
- White comes with regular milk by default. **Oat, Rokeby or soy cost +S$1.**
- Cold brew uses its **own house bean(s)**. The better beans are kept for the weekend shop.

**The bean cycle (weekend shop)**
- Every **14 days** a new **pair of premium beans (A and B)** goes on the weekend menu, starting on a **Saturday**.
- **Days 1–7: mystery.** The beans are shown only as "Bean A" and "Bean B" everywhere, including the website.
- **Day 8 (Saturday): reveal.** Names, origin and tasting notes appear.

**The mystery game (at the counter, debut weekend only, days 1–2)**
- A customer who orders a mystery-bean drink may guess its tasting notes **once**. If the first guess is right, **that drink is free**.
- Guesses are free text, **judged by the barista** with the official notes and accepted similar words shown in the staff app.
- **One try per mobile number, per bean, per cycle.**

## 2. Success criteria

The owners only:
1. brew,
2. serve and hand over orders,
3. judge guesses,
4. order beans when told,
5. glance at the numbers,
6. post 3–4 prepared Instagram posts a week.

Specifically:
- Everything about the website (menu, info, about us, pre-order settings) is changed from the staff app, with no code.
- Every sale, pre-order and giveaway is recorded.
- The 6pm brew list needs no counting.
- The mystery/reveal switch happens everywhere automatically.

## 3. Constraints and principles (unchanged from v1)

**Constraints**
- Payments: personal PayNow and cash only. No payment provider, no cards.
- Singapore. SFA licence held; premises settled externally (owner-reported).
- Lean budget: free tiers that allow commercial use; pay-per-use AI. A custom domain is now needed (yearly fee).
- Built by the owner with Claude Code / Codex; the barista is non-technical.
- IRAS: keep records for 5 years.
- PDPA: collect only what is needed, and say why.

**Principles**
1. **SQL computes, AI explains.**
2. **One source of truth** (the Supabase ERP).
3. **The shop never depends on the software** (paper fallback).
4. **She only acts on exceptions and routines.**

## 4. Changes from v1

| Area | v1 | v2 |
|---|---|---|
| Customer ordering | Tally form synced from the ERP | **Café website** with the menu of the week, business info, about us and a **built-in pre-order form** (no Tally) |
| Staff tools | Counter PWA + admin pages | **One all-in-one staff web app** on a subdomain (e.g. `staff.<domain>`): payments, orders, dashboard, and control of the website and form |
| Payment QR for pre-orders | Static QR; customer types the amount | **Exact-amount PayNow QR per order** with the order code (possible again now that the form is ours) |
| Pre-orders | Any drink, during opening hours | **Weekday cold brew only**: black or white, milk choice, cutoff 6pm the day before |
| Counter | Every day | **Weekends only**, plus the **mystery game** |
| Beans | Fixed beans; low-stock alert | **Cold brew house bean:** low-stock alert (v1 logic). **Weekend premium pair:** 14-day A/B cycle with planned ordering |
| Production | None | **6pm brew list** for tomorrow, plus a milk count for bottling |
| Free drinks | None | **Game giveaways:** S$0 revenue, cost still counted |
| Content | Weekly brief | Same, planned around the cycle, with a **reveal embargo** |
| Pickup | (Unattended cooler proposed in another session) | **Attended handover**; unattended pickup set aside |

## 5. Architecture

```
Customers ─► Café website (<domain>) ───────────┐  menu of the week · info · about · pre-order form
              place-order / submit-proof ────────┤  (Edge Functions; Turnstile anti-spam)
Owners ────► Staff web app (staff.<domain>) ─────┼─► Supabase ERP (Singapore)
              counter · orders · dashboard ·     │    • Bean cycle engine (weekend pair, mystery → reveal)
              website & form controls · cycles   │    • Sales, orders, stock, costs
                                                 │    • Scheduled jobs (pg_cron)
                                                 ├─► Telegram bot ─► owners (brew list, pickups, flags, bean orders, digest)
                                                 ├─► Claude API (payment-proof check, digest notes)
                                                 ├─► Nightly encrypted backup ─► Cloudflare R2
                                                 └─► Weekly brief ─► Google Drive ─► Claude Cowork + Canva ─► Instagram (she posts)
```

| Layer | Choice |
|---|---|
| Public website | React + Vite, on **Cloudflare Pages** at the root domain. It reads public data through one function, `site_snapshot()`, and writes only through Edge Functions |
| Staff web app | React + Vite PWA, on **Cloudflare Pages** at `staff.<domain>`. Supabase Auth; owner = admin, barista = staff. Installed on the tablet |
| Backend | **Supabase, Singapore region** (free): Postgres + RLS, Auth, Storage, Edge Functions, `pg_cron` |
| Anti-spam | **Cloudflare Turnstile** (free) on the pre-order form |
| Shared code | One monorepo: `apps/site`, `apps/staff`, `packages/shared` (money, PayNow QR, types) |
| Unchanged | Telegram bot, Anthropic API, GitHub Actions → R2 backups, Cowork + Canva |

## 6. Components

### 6.1 Café website (public)
Pages:
- **Home:** this week's highlights, the next pickup day, an announcement banner.
- **Menu of the week:**
  - weekday cold brew with prices and milk options;
  - the weekend menu with the current bean pair, shown as mystery in week 1 and revealed in week 2;
  - sold-out flags.
- **About us** and **Info:** location, weekday pickup times, weekend hours, contact, Instagram.
- **Pre-order** (6.2).

**All content comes from the ERP**, which the staff app edits:
- page text and photos;
- menu, prices and availability;
- pickup times and closed days;
- weekend hours.

**Embargo:** before day 8, the site never receives the weekend beans' names or notes. `site_snapshot()` enforces this in SQL.

### 6.2 Pre-order form (on the website)
1. Choose cold brew: **Black** or **White**. For White, milk: Regular (default) / Oat (+S$1) / Rokeby (+S$1) / Soy (+S$1). Quantities; running total.
2. **Pickup day:** open weekdays from the earliest day allowed by the 6pm cutoff. The window and the optional daily bottle cap are set in the staff app.
3. **Pickup time:** that day's pickup times.
4. Name and mobile, with a PDPA purpose note.
5. **Turnstile** check, then submit. The `place-order` function:
   - recomputes the price from the ERP;
   - checks the cutoff, the day and any cap;
   - creates the order and an order code (e.g. `CB-0412`).
6. The page shows an **exact-amount PayNow QR** with the order code as the reference.
7. The customer pays and **uploads the screenshot** (`submit-proof`, using a signed upload URL). The page shows "Order confirmed".

**Proof check (unchanged):** a vision model reads the screenshot and code compares the fields:
- amount;
- recipient;
- time;
- reference = order code;
- transaction ID not seen before.

A full pass is auto-verified. A failure is flagged in the staff app and on Telegram, with a tap-to-WhatsApp link.

### 6.3 Staff web app (all-in-one, on the tablet)
| Area | What it does |
|---|---|
| **Counter** (weekends) | Big drink buttons, A/B bean picker, modifiers, exact-amount PayNow QR or cash, optimistic save + retry, **Play guess** (6.4), paper-fallback back-dated entry |
| **Orders** (weekdays) | Today's pickups by time with Collected / No-show; flagged proofs with Accept / Reject; tomorrow's brew list |
| **Dashboard** | Sales, profit per drink, cold brew per weekday, weekend sales by hour, game report, PayNow total for the bank check |
| **Website** | Edit home, about and info text and photos; announcement banner; publish |
| **Menu** | Items, prices, milk and other modifiers, where each item is sold (pre-order / weekend), sold-out toggle |
| **Pre-orders** | Open/close pre-orders, pickup times per weekday, closed days, cutoff time, optional daily cap |
| **Beans and cycles** | Bean catalogue (cold brew and premium), next cycle's pair with names and notes and accepted similar words, purchases, "Arrived" |
| **Settings** (owner only) | PayNow name and number, cold brew recipe numbers, Telegram chat IDs, staff accounts |

Telegram stays as the **push channel**. The staff app is where things are done; Telegram nudges.

### 6.4 Mystery game flow (Counter, debut weekend only)
1. She rings up the mystery drink and taps **Play guess**.
2. She enters the customer's mobile number. The app checks the one-try rule: "Already played Bean A this cycle".
3. She types the guess.
4. The app shows her the official notes and accepted similar words.
5. She taps **✅ Correct** or **❌ Not this time**.
6. **Correct** → the line is a giveaway at S$0, with cost and stock still recorded. **Wrong** → a normal sale.

The guess text is saved for the reveal post. PDPA line: "Your number is only used so everyone gets one guess per bean."

### 6.5 Bean cycle engine (weekend premium pair)
- **Next cycle setup:** in the staff app, the owners pick a bean for slot A and slot B, and enter display names, origin, tasting notes and accepted similar words.
- **Reminders:** Telegram on the Wednesday before ("fill in next pair") and on the Friday before the reveal ("notes check").
- **Saturday 00:05 job:**
  - **day 1:** the new pair becomes current (mystery);
  - **day 8:** reveal.

  The website, menu and staff app follow automatically.
- **Slot resolution:** weekend recipes refer to **slot A/B**. A sale resolves the slot to the cycle's actual bean, so stock and cost are correct.

### 6.6 Brew list and pickups (weekday cold brew)
- **6:00pm Sun–Thu** (Telegram + Orders screen), tomorrow's:
  - bottles;
  - litres;
  - cold brew bean grams and water;
  - **milk count for bottling** (e.g. "White: 5 regular, 2 oat, 1 soy").

  The maths uses settings: bottle ml, ratio (g/L), yield %, spare bottles.
- **Morning:** pickup list by time.
- **At handover:** Collected. Orders not collected by the end of the day → `no_show` with WhatsApp links.

### 6.7 Telegram bot (owners only)
| Message | When |
|---|---|
| Brew list + milk count | 6:00pm Sun–Thu |
| Pickup list | Weekday mornings |
| Payment flag | Only on a failed proof check |
| Weekend check-in ("Open 9:00–14:00 today?") | Sat and Sun mornings |
| Next pair setup / reveal check | Wednesday and Friday before |
| Bean order: cold brew bean low; next premium pair | When calculated |
| Sunday summary + one suggestion | Sunday evening |
| Job failures, price mismatches | To the owner (developer) only |

### 6.8 Content agent (Cowork + Canva, unchanged mechanics)
The weekly brief now includes:
- the cycle stage;
- game stats after the reveal;
- cold brew highlights.

Cycle posts:

| Cycle day | Post |
|---|---|
| Day 1 | "New mystery pair, come guess this weekend" |
| Weekdays | Cold brew pre-order reminders |
| Day 8 | **Reveal:** names, origin, notes, game stats, best guesses with no names |
| Day 12 | Last call + next teaser |

**Rules from v1:**
- real photos only;
- prices only from the brief;
- never name customers;
- inspiration for style only.

**New rule:** never reveal the beans before day 8; the brief withholds them.

### 6.9 Backups and security (unchanged)
- Nightly `age`-encrypted dump to R2, with a monthly restore test.
- RLS on every table.
- Public writes only through `place-order` / `submit-proof`, both behind Turnstile and rate limits.
- Public reads only through `site_snapshot()`.
- Secrets live only in the platforms' secret stores.

## 7. Data model (changes from v1)

| Table | Notes |
|---|---|
| `menu_items` | Adds `sold_on` (`weekday_preorder` / `weekend_counter` / `both`), `uses_bean_slot`, `description`, `sold_out` |
| `modifiers` | Adds `group_name` (e.g. `milk`). Seed: Oat, Rokeby, Soy at +100 cents |
| `menu_item_modifiers` (new) | Which modifiers each item allows (e.g. White cold brew → milk group) |
| `recipe_lines` | Either a fixed `ingredient_id` **or** `chosen_bean = true` (grams of whichever A/B bean the customer picks, resolved at sale), never both |
| `bean_cycles` (new) | `starts_on` (a Saturday), `reveal_on` = +7, `ends_on` = +13 (generated) |
| `cycle_beans` (new) | cycle, slot, bean ingredient, display name, origin, `tasting_notes[]`, `accepted_terms[]` (staff-only) |
| `order_items` | Adds `bean_slot`, `bean_ingredient_id`, `cycle_id`, `is_game_giveaway` |
| `guesses` (new) | cycle, slot, mobile, guess text, correct, order; **unique (cycle, slot, mobile)** |
| `customers` | Pre-order customers and game players (mobile) |
| `site_content` (new) | key (`home`, `about`, `info`, `announcement`), title, body, image path, published |
| `pickup_times` (new), `day_overrides` | Weekday pickup times; closed days, weekend hours, bottle cap |
| `settings` | Cutoff 18:00, bottle_ml, brew_ratio_g_per_l, yield_pct, spare_bottles, PayNow, Telegram allowlist |

Unchanged:
- `ingredients` (beans flagged `cold_brew` or `premium` through a `bean_use` column);
- `orders`;
- `stock_movements`;
- `purchases`;
- `expenses`;
- `weekly_insights`;
- `job_runs`.

## 8. Business logic

- **Brew list:**
  - `bottles = Σ qty + spare`
  - `finished_L = bottles × bottle_ml / 1000`
  - `water_L = finished_L / yield`
  - `beans_g = water_L × ratio`
  - Milk counted per type.
  - Example (placeholder numbers): 8 bottles × 250 ml = 2.0 L → 2.35 L water → 188 g beans at 80 g/L and 85% yield.
- **Cold brew bean:** the v1 low-stock forecast. Order when days left ≤ delivery + rest + 3 safety days; quantity covers 14 days; freshness cap.
- **Premium pair:**
  - Order date = next `starts_on` − (delivery + rest + 2 days).
  - Quantity per slot = the last 2 cycles' average weekend grams × 0.5, adjusted by the last A/B split, plus 10%, rounded to whole bags.
  - Owner estimates are used until 2 cycles of history exist.
- **Finance:**
  - v1 metrics, plus game cost (giveaways at cost), A vs B share, cold brew bottles per weekday, milk option uptake, cutoff misses, no-shows.

## 9. Testing (additions)
- **Unit:**
  - brew-list and milk maths;
  - cutoff (including Friday → Monday);
  - pair order date and quantity;
  - slot resolution across the reveal.
- **Database:**
  - the `site_snapshot()` embargo;
  - one-try uniqueness;
  - a giveaway has zero revenue and keeps its cost;
  - a modifier not allowed on an item is rejected.
- **End to end:**
  - pre-order on the website → screenshot → proof check → brew list → Collected;
  - a staff edit of website text appears on the site.
- **Manual:**
  - PayNow QR on DBS/OCBC/UOB;
  - one full rehearsal cycle with test dates.

## 10. Build order
See `docs/superpowers/plans/2026-10-11-roadmap-v2.md`.
1. Foundation
2. Weekend counter + game + cycles
3. Weekday pre-orders
4. Website and business controls
5. Beans, finance, insights
6. Backups + marketing brief
7. Content agent

Then a pilot: one full 14-day cycle.

## 11. Out of scope (v2)
- Card payments or a payment provider.
- Unattended pickup.
- Delivery.
- Customer accounts and logins.
- Customer notifications.
- Auto-ordering beans.
- Auto-posting to Instagram.
- Loyalty cards.
- AI-generated product images.
- Playing the game through pre-orders.

## 12. Open items
1. Weekend menu.
2. Weekday pickup times and the handover spot.
3. Cold brew bottle size, prices (black and white), ratio and yield.
4. How close a guess must be (set per cycle; the barista decides).
5. **Domain name** (needed for `staff.<domain>`).
6. Carried over: brand kit, IG Business account, the barista's paid Claude plan, Canva/Cowork spike, a refresh of the owner proposal for v2.
7. **Assumption to confirm:** cold brew uses house bean(s) with no A/B choice; the A/B cycle and game are weekend-only.
