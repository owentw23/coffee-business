# Operations automation — design spec v2

- Date: 2026-10-11
- Status: draft for owner review
- Supersedes: `2026-09-30-operations-automation-design.md` (v1). Everything in v1 carries over unless this document changes it. The "Changes from v1" table lists every change.
- Rationale and history: `Knowledge/Projects/Scope decisions.md`, `Knowledge/Projects/Operations automation architecture.md`, `Knowledge/Resources/Similar systems review.md`

## 1. The business

A small coffee business run by two owners from a condo in Singapore.

| When | What is sold | How |
|---|---|---|
| **Weekdays (Mon–Fri)** | **Cold brew, pre-order only** | Customers order on a Tally form by **6pm the day before** and collect at a pickup time; someone hands it over in person |
| **Weekends (Sat–Sun)** | Balcony shop on level 1 (**menu to be decided**) | Walk-in customers at the counter; sales entered on the counter tablet |

**Unique selling point: the bean cycle.**
- Every **14 days** a new **pair of beans (A and B)** starts. Customers choose between them.
- **Week 1 (days 1–7): mystery.** The beans are shown only as "Bean A" and "Bean B".
- **Week 2 (days 8–14): revealed.** Names, origin and tasting notes are shown.
- Cycles **start on a Saturday**. Day 1 is a Saturday, day 8 (reveal) is a Saturday, and day 14 is a Friday.

**The mystery game (counter only).**
- On the **mystery weekend (days 1–2)**, a customer who orders a mystery-bean drink at the counter can guess its tasting notes **once**.
- If the guess is right **on the first try**, that drink is **free**.
- Guesses are free text. The barista judges them, with the official notes on the tablet screen.
- **One try per person, per bean, per cycle**, enforced by mobile number.
- Pre-orders are not part of the game. Weekday cold brew on days 3–7 is still sold as mystery Bean A/B, but it cannot be played.

## 2. Success criteria

The owners only need to:
1. brew,
2. serve and hand over orders,
3. judge guesses,
4. order beans when told,
5. glance at the numbers,
6. post 3–4 prepared Instagram posts a week.

Specifically:
- every sale and every game giveaway is recorded;
- the brew list for tomorrow arrives at 6pm without anyone counting orders;
- the mystery-to-reveal switch happens everywhere automatically;
- the next bean pair is ordered in time to arrive and rest.

## 3. Constraints and principles (unchanged from v1)

**Constraints**
- Payments: personal PayNow and cash only. No payment provider, no cards.
- Singapore. SFA licence held; premises rules settled externally (owner-reported).
- Lean budget: free tiers that allow commercial use, plus pay-per-use AI.
- Built by the owner with Claude Code / Codex; the barista is non-technical.
- IRAS: keep business records for 5 years.
- PDPA: collect only what is needed, and say why.

**Principles**
1. **SQL computes, AI explains.** No AI-generated number is used for decisions.
2. **One source of truth.** Everything lives in the Supabase ERP.
3. **The shop never depends on the software.** A paper fallback always exists.
4. **She only acts on exceptions and routines.** Clean cases are automatic.

## 4. Changes from v1

| Area | v1 | v2 |
|---|---|---|
| Pre-orders | Any drink, during opening hours, 7-day window | **Weekday cold brew only.** Pickup Mon–Fri at set pickup times; order by **6pm the day before**; window limited to the current cycle |
| Counter tablet | Every day | **Weekends only**, plus the **mystery game** flow |
| Opening hours | One schedule for everything | Two schedules: **weekday pickup times** (cold brew) and **weekend shop hours** |
| Beans | Fixed beans; daily low-stock alert | **14-day A/B cycles.** Recipes use bean *slots* A/B; **next-pair ordering** is planned per cycle |
| Production | None | **Nightly brew list** at 6pm for the next day's pickups |
| Free drinks | None | **Game giveaways**: recorded at S$0 revenue, cost still counted |
| Customer identity | Pre-order mobile only | Pre-order mobile **plus** game players' mobile (opt-in to play) |
| Content | Weekly brief, 3–4 posts | Same, but **planned around the cycle**: launch, game results, reveal, last call |
| Weekday pickup | (Another session proposed an unattended cooler) | **Attended**: someone hands it over. Unattended pickup set aside |

## 5. Architecture

```
Weekday customers ─► Tally cold brew form ─┐ (webhook)           ┌─► Telegram bot ─► owners
Weekend customers ─► Counter tablet (+game) ┼─► Supabase ERP ─────┤   (brew list, pickups, flags,
Owners ───────────► Admin dashboard ───────┘   • Bean cycle engine│    bean orders, digest)
                                                • Sales & orders   ├─► Claude API (payment-proof check, digest notes)
                                                • Scheduled jobs   ├─► Nightly encrypted backup → Cloudflare R2
                                                                   └─► Weekly brief → Google Drive
                                                                         → Claude Cowork + Canva → Instagram (she posts)
```

Hosting (unchanged from v1):
- **Staff app:** React + Vite on **Cloudflare Pages** (free).
- **Backend:** **Supabase, Singapore region** (free), including Edge Functions and `pg_cron`.
- **Forms:** Tally (free).
- **Inbox:** Telegram (free).
- **Backups:** GitHub Actions → Cloudflare R2 (free).
- **AI:** Anthropic API (pay per use).
- **Content:** Claude Cowork + Canva on the barista's own Claude account.

## 6. Components

### 6.1 Bean cycle engine (new)
- **Admin "Cycles" page.** For the next cycle, the owners enter for slot A and slot B:
  - which bean (from the bean catalogue);
  - display name and origin;
  - **tasting notes** (e.g. "blueberry, cacao, jasmine");
  - **accepted similar words** for judging (e.g. "berry" for blueberry, "chocolate" for cacao).
- **Reminders:**
  - Telegram nudges on the Wednesday before a cycle starts: "Next pair starts Sat 25 Oct — fill in beans and notes".
  - On Friday evening: "Reveal tomorrow — notes check".
- **Automatic switches (scheduled job, Saturday 00:05):**
  - **Day 1:** the new pair becomes current. Tally, the tablet and the menu show "Bean A" and "Bean B".
  - **Day 8:** reveal. The same places switch to real names and notes, and `tally-sync` updates the form.
- **Slot resolution.** Recipes and order lines refer to **slot A or B**. The engine resolves the slot to the actual bean of the cycle the sale belongs to, so stock and cost hit the right bean.

### 6.2 Weekday cold brew pre-orders (Tally)
Form flow:
1. Bottles of cold brew: quantity of **Bean A** and **Bean B** (mystery names in week 1, real names in week 2). Total calculated.
2. **Pickup day**: weekdays from tomorrow (or later, if past 6pm) up to the last weekday of the current cycle.
3. **Pickup time**: that day's set pickup times.
4. Name and mobile, with a PDPA purpose note.
5. Static PayNow QR plus "Your total is S$X". The customer pays and uses their mobile number as the reference.
6. Upload the payment screenshot → "Order confirmed".

Behind the form (as in v1):
- **`tally-webhook`:**
  - checks Tally's signature;
  - recomputes the price from the ERP;
  - checks the cutoff and pickup time (late or invalid orders are flagged);
  - stores the order with bean slots;
  - copies the screenshot;
  - runs the background AI proof check, which auto-verifies a full pass and otherwise flags to Telegram.
- **`tally-sync`:** the ERP rewrites the form whenever any of these change:
  - prices;
  - pickup times;
  - closed days;
  - cycle start or reveal;
  - the 6pm cutoff rolling to the next day.
- **No pickup capacity limit by default.** An optional per-day bottle cap (e.g. "max 20 bottles Wed") exists in settings. When a day is full, sync removes that day.

### 6.3 Brew list and pickups
- **6:00pm, Sun–Thu (cutoff):** Telegram sends tomorrow's brew list, per slot:
  - bottles;
  - litres of finished cold brew;
  - grams of beans and water.

  Example with an 80 g/L ratio and 85% yield: "Brew tonight → Bean A: 8 bottles = 2.0 L → 188 g beans + 2.35 L water." These numbers are placeholders until the owners set their recipe.
- **The maths uses settings:**
  - bottle size (ml);
  - brew ratio (g beans per L water);
  - yield (% of water recovered after filtering);
  - an optional spare-bottles buffer.
- **Morning of pickup:** Telegram sends the pickup list grouped by time, with name and bottles.
- **At handover:** she taps **Collected** in Telegram (or on the tablet).
- **No-shows:** orders still uncollected at the end of the day become `no_show`. The bot lists them with tap-to-WhatsApp links.

### 6.4 Weekend counter tablet + mystery game
Sales:
- Large buttons for the weekend menu (to be decided); each drink using beans asks **A or B** in one tap.
- Exact-amount PayNow QR or Cash.
- Optimistic save with retry, plus the paper fallback (unchanged from v1).

**Game flow (mystery weekend only; the button is hidden otherwise):**
1. She rings up the mystery drink as usual and taps **Play guess**.
2. She enters the customer's **mobile number**. The tablet checks whether that number has already guessed this bean this cycle. If it has: "Already played Bean A this cycle."
3. She types the customer's guess (free text).
4. The tablet reveals, to her only, the official notes and accepted similar words.
5. She taps **✅ Correct** or **❌ Not this time**.
6. **Correct:** the drink is recorded as a **game giveaway**: S$0 charged, cost and stock still recorded. **Wrong:** a normal sale.

The guess text is saved either way. It feeds the reveal post ("best guesses").

PDPA purpose line on the tablet: "Your number is used only to make sure everyone gets one guess per bean."

### 6.5 Admin dashboard
v1 features remain:
- menu and prices;
- recipes;
- bean catalogue;
- expenses;
- reports;
- PayNow settings.

Added:
- the **Cycles page** (6.1);
- **weekday pickup times** and closed days;
- **weekend shop hours**;
- cold brew settings (bottle size, ratio, yield, optional daily cap);
- a **game report** per cycle.

### 6.6 Telegram bot (owners only)
| Message | When |
|---|---|
| Brew list for tomorrow | 6:00pm Sun–Thu |
| Pickup list | Morning of each pickup day |
| Collected / no-show buttons | During and after pickups |
| Payment flag (with tap-to-WhatsApp) | Only when a proof check fails |
| Weekend check-in: "Shop open 9:00–14:00 today?" | Sat and Sun morning |
| Pause, close a day, sold out | Any time (commands) |
| Next-cycle setup reminder; reveal check | Wednesday and Friday before a cycle starts or reveals |
| Order next bean pair (list + quantities) | On the calculated order date |
| Sunday summary + one suggestion | Sunday evening |
| Job failures, price mismatches | To the owner (developer) only |

### 6.7 Content agent (Cowork + Canva)
Unchanged mechanics from v1:
- the weekly brief goes to Drive;
- a Cowork scheduled task on the barista's Claude account plans 3–4 posts;
- Canva designs them;
- she revises by chatting, then posts.

The brief now includes **cycle context**, so the plan follows the cycle:

| Cycle day | Post idea |
|---|---|
| Day 1 (Sat) | "New mystery pair — come guess this weekend" |
| Day 3 (Mon) | Mystery cold brew is open for pre-orders |
| Day 8 (Sat) | **Reveal**: names, origin, notes, game stats ("9 of 31 guessed right"), best guesses with no names |
| Day 11–12 | Last call for this pair + teaser for the next |

**Rules kept from v1:**
- only real photos;
- prices only from the brief;
- never name customers;
- inspiration is used for style, not copied.

**New rule:** never reveal week-1 beans before day 8. The brief withholds bean names and notes until the reveal.

### 6.8 Payment proof check, backups, security
Unchanged from v1:
- the vision model extracts the fields, code compares them, and a full pass is auto-verified;
- the weekly PayNow total is the backstop;
- nightly `age`-encrypted dump to R2, with a monthly restore test;
- RLS on every table;
- the only public endpoint is the signed Tally webhook;
- secrets are kept only in the platforms' secret stores.

## 7. Data model (changes from v1)

| Table | Change |
|---|---|
| `bean_cycles` (new) | id, `starts_on` (Saturday), `reveal_on` (= +7), `ends_on` (= +13), status (`planned`/`current`/`done`) |
| `cycle_beans` (new) | cycle_id, `slot` (`A`/`B`), ingredient_id (bean), display_name, origin, `tasting_notes text[]`, `accepted_terms text[]`. Notes are readable by staff only. The public brief and form get them only from day 8 |
| `recipe_lines` | Either `ingredient_id` **or** `bean_slot` (`A`/`B`), never both. Bean grams live on the slot line |
| `menu_items` | Adds `sold_on` (`weekday_preorder` / `weekend_counter` / both) and `uses_bean_slot` (boolean) |
| `order_items` | Adds `bean_slot`, `bean_ingredient_id` (resolved at sale), `cycle_id`, `is_game_giveaway`. A giveaway line has `unit_price_cents = 0`; its cost snapshot is kept |
| `guesses` (new) | cycle_id, slot, mobile, guess_text, `is_correct`, order_id, created_at. **Unique (cycle_id, slot, mobile)** = one try |
| `customers` | Also created for game players (mobile required, name optional) |
| `pickup_times` (new) | weekday, time (e.g. Mon 07:30, Mon 08:30) |
| `day_overrides` | Replaces `opening_overrides`: date, closed, weekend hours, optional bottle cap |
| `settings` | Adds cutoff time (18:00), bottle_ml, brew_ratio_g_per_l, yield_pct, spare_bottles |

Unchanged from v1:
- `orders` (channel `pre_order` = weekday cold brew; `walk_in` = weekend);
- `stock_movements`;
- `purchases`;
- `expenses`;
- `weekly_insights`;
- `job_runs`.

Revenue counts paid sales only. Giveaways appear as **"game cost"**: their cost at cost price.

## 8. Business logic

### 8.1 Brew list
For each slot, for tomorrow's verified pre-orders:
- `bottles = Σ qty + spare_bottles`
- `finished_L = bottles × bottle_ml / 1000`
- `water_L = finished_L / yield_pct`
- `beans_g = water_L × brew_ratio_g_per_l`

### 8.2 Next-pair bean order
- **Order date** = next cycle `starts_on` − (`lead_time_days` + `rest_days` + 2 safety days).
- **Quantity per slot** = expected grams for 14 days, rounded up to whole bags:
  - **weekday:** the average cold brew grams per cycle over the last 2 cycles × 0.5 (even A/B split), adjusted by the last cycle's A/B share;
  - **weekend:** the same, using weekend recipe grams;
  - plus 10% buffer.
- Before 2 cycles of history exist, owner-entered estimates are used.
- A mid-cycle low-stock alert is kept. A top-up bag may need resting, so the alert includes the rest days.

### 8.3 Finance and insights
v1 metrics (revenue, cost of goods, margin per item, net, PayNow weekly total) are kept, plus:
- **per cycle:** A vs B share, cold brew bottles per weekday, weekend sales by hour;
- **game:** players, win rate, giveaway cost, repeat visits by game players;
- **pre-orders:** cutoff-miss rate, no-shows.

## 9. Testing (additions to v1)
- **Unit:**
  - brew-list maths;
  - next-pair order date and quantity;
  - slot resolution across the reveal boundary;
  - cutoff logic, including Friday → Monday.
- **Database:**
  - the one-try uniqueness of `guesses`;
  - a giveaway records cost but zero revenue;
  - the week-1 notes are not readable through the brief path.
- **Manual before launch:**
  - one full rehearsal cycle with test dates (launch → game → reveal → next pair) on the test bot and test form.

## 10. Build order (revises the roadmap)
1. **Foundation**, including the cycle schema (`bean_cycles`, `cycle_beans`, slot recipes) and slot-aware `record_sale`. Plan 1 needs a revision before execution.
2. **Weekend counter tablet + mystery game + Cycles admin page.**
3. **Weekday cold brew pre-orders** (Tally sync and webhook, proof check, brew list, pickup list).
4. **Bean planning, finance, insights, digest.**
5. **Backups and marketing brief** (with cycle context and reveal embargo).
6. **Content agent** (Cowork + Canva).

Then a pilot: one full 14-day cycle with residents.

## 11. Out of scope (v2)
- Payment providers and cards.
- Unattended pickup.
- Delivery.
- Customer notifications.
- Auto-ordering beans.
- Auto-posting to Instagram.
- Loyalty cards.
- AI-generated product images.
- Playing the game through pre-orders.

## 12. Open items
1. **Weekend menu**: to be decided. The design supports any drinks, including bean-slot drinks.
2. **Pickup times** for weekdays (e.g. 7:30 and 8:30), plus where the handover happens.
3. **Cold brew numbers:** bottle size, price, brew ratio, yield.
4. **Accepted similar words policy:** how close a guess must be (e.g. "2 of 3 notes" vs "all notes"). This is set per cycle on the Cycles page; the barista has the final say.
5. Carried over from v1: brand kit, Instagram Business account, the barista's paid Claude plan, Canva/Cowork spike.
6. The owner proposal Word document (`docs/proposal/`) describes v1 and needs a v2 refresh.
