# Proposal brief: Café Operations System (for ChatGPT to write the Word document)

## How to use this brief

Paste this whole file into ChatGPT and attach `architecture.svg` from the same folder. Then ask:

> "Using this brief and the attached diagram, create the Word document (.docx) exactly as specified."

Everything ChatGPT needs is below. **Treat the facts in this brief as fixed.** Do not add features, prices, dates or promises that are not written here.

---

## 1. The job

Write a short, friendly **proposal document in Microsoft Word (.docx)** that explains to the café's owner how the new operations system will work. The design is already approved; this document explains it and invites any last concerns.

- **Reader:** the café owner who runs the counter (the barista). Non-technical. Knows coffee and the business, not software.
- **Goal:** after reading, she understands:
  - what she will do day to day;
  - what the system does for her;
  - what it costs;
  - what she needs to provide;
  - the order in which it will be built.
- **Length:** about 8–10 A4 pages, including a title page.
- **Tone:** warm, plain English, confident, no jargon.
  - Say "you" to the reader.
  - Short sentences.
  - Explain any unavoidable technical word in brackets the first time, e.g. "Telegram bot (an automated chat account)".
- **Spelling:** British/Singapore English (colour, organise). Currency is **S$**.

## 2. Formatting

- **Page:** A4, 2 cm margins, page numbers in the footer ("Café Operations System proposal · page N").
- **Font:** Arial (or similar sans-serif), 11 pt body text.
- **Colours:** headings coffee brown `#4A3426`; subheadings teal `#2F7D6D`; table header rows cream `#F3E9DC`.
- **Headings:** use Word's built-in Heading 1 and Heading 2 styles, so a table of contents works.
- **Lists:** real Word bullets and numbering, not typed characters.
- **Tables:** simple, light borders, header row shaded cream.
- **Page breaks:** start each major section on a new page where it helps readability. Sections 2, 3, 5 and 8 should start on new pages.
- **No emojis** in the document.

## 3. The diagram

Insert the attached **`architecture.svg`** as Figure 1 in section 2, converted to a high-resolution PNG if needed. Place it full width with the caption *"Figure 1. The complete system at a glance."*

If the file cannot be used, recreate the same diagram with four columns from left to right, with arrows flowing left to right in each row:

| WHO | WHAT THEY USE | THE CAFÉ SYSTEM | WHAT COMES OUT |
|---|---|---|---|
| Customers (condo residents) | Pre-order form (Tally · PayNow + screenshot) | One tall box, "Café system (the 'ERP' database)", listing: Sales & pre-orders · Menu & recipes · Bean stock · Costs & profit · Opening hours · Weekly insights. Footnote: "Numbers are calculated, never guessed by AI" | Telegram bot (order & bean alerts, hours & sold-out buttons, Sunday summary) |
| Barista (serving at the counter) | Counter tablet (tap drinks · PayNow or cash) | (same box) | Weekly brief (Google Drive) → Claude + Canva (plans & designs posts) → Instagram (3–4 posts a week) |
| Owners (checking the business) | Admin dashboard (menu · costs · reports) | Under the system box: "AI payment check" (reads each screenshot) and "Nightly backup" (encrypted, kept 5+ years) | Note: "Barista reviews by chatting, then posts" |

Colours: people boxes in cream/brown, the screens they use in light teal, the café system in dark brown with white text, outputs in light blue, safeguards in light grey.

---

## 4. Document outline and content

Write each section from the points given. Keep tables exactly as written; the wording may be polished lightly.

### Title page
- **Title:** Café Operations System
- **Subtitle:** A proposal for running the balcony café with less admin
- **Lines:** "Prepared 30 September 2026" · "Status: design approved; build not yet started"
- **One-sentence summary (larger text):** "You make the coffee, and the system takes the orders, checks payments, tracks beans and money, and prepares your Instagram posts."

### 1. What changes for you
Opening paragraph: her time goes into coffee and customers, not admin. She stays in control; the system does the repetitive work and only asks her when a human is needed.

**What you do** (bullets):
- Make coffee and serve customers.
- Tap the drinks on the counter tablet for walk-in customers.
- Hand over pre-orders at the pickup time.
- Order beans online when the system tells you which ones and how much.
- Glance at the Sunday summary to see sales, costs and profit.
- Review your Instagram posts by chatting with Claude, then post them.

**What the system does for you** (bullets):
- Takes pre-orders through a simple online form linked from Instagram.
- Checks each payment screenshot automatically and only alerts you if something looks wrong.
- Keeps a record of every sale, pre-order and cost in one place.
- Works out when beans are running low, including the resting time fresh beans need.
- Calculates profit per drink and sends you a weekly summary.
- Plans 3–4 Instagram posts a week and designs them in Canva using your real photos.

### 2. How the system works
- One sentence: read the diagram left to right: who uses it, what they use, the café system in the middle that remembers everything, and what comes out.
- **Figure 1** (the diagram).
- Subheading **"The four parts in plain words"**, with one bullet per column:
  - **Who:** customers order ahead; the barista serves at the counter; the owners check how the business is doing.
  - **What they use:** a pre-order form (no app to install), a counter tablet, and an admin dashboard for menu, costs and reports.
  - **The café system:** one private database that remembers every sale, recipe, bean bag and expense, and does all the maths.
  - **What comes out:** Telegram messages to your phone, a weekly brief that Claude uses to plan posts, and ready-to-post Instagram designs.

### 3. What customers experience
**Pre-order (up to 7 days ahead)**, as a numbered list:
1. Tap the order link in the café Instagram bio or the condo chat.
2. Choose drinks. The total is added up automatically.
3. Pick a day and a pickup time within opening hours.
4. Enter name and mobile number.
5. Scan the PayNow QR, pay the exact total, and use their mobile number as the reference.
6. Upload the payment screenshot. They immediately see "Order confirmed".
7. Collect the drink at the chosen time. It is already paid.

Follow with a short paragraph. Behind the scenes, AI reads the screenshot and checks the amount, recipient, time and reference. If everything matches, nothing is needed from her. If not, she gets a Telegram alert with a tap-to-WhatsApp link to message the customer herself.

**Walk-in**, as a numbered list:
1. The customer orders at the balcony counter.
2. You tap the drinks on the tablet. It shows a PayNow QR with the exact amount, or you choose Cash.
3. The sale is saved instantly and the screen clears for the next customer.

### 4. Your week with the system
A table with columns **When | What happens | Your effort**:

| When | What happens | Your effort |
|---|---|---|
| Each morning | Telegram asks "Opening today 8:00–14:00?" The order form updates itself. | One tap |
| During service | Walk-ins on the tablet; pre-orders listed by pickup time. | Normal service |
| Any time | Pause pre-orders, close early, or mark an item sold out from Telegram. | One tap |
| When flagged | A payment screenshot did not match. Message the customer. | Only when needed |
| When beans run low | List of beans and quantities to reorder. Order online, tap "Ordered". | A few minutes |
| When beans arrive | Tap "Arrived" so stock stays accurate. | One tap |
| Sunday evening | Summary of sales, costs and profit, plus one suggestion. Compare the PayNow total with your bank app. | A few minutes |
| Sunday night onward | Claude has planned the week's posts. Send any requested photos, chat to adjust, then post. | Short chat per post |

### 5. The building blocks
A table with columns **Part | What it does | Service used**:

| Part | What it does | Service used |
|---|---|---|
| Pre-order form | Menu, pickup day and time, payment screenshot. Updates itself when hours or menu change. | Tally (free) |
| Counter tablet | Fast drink buttons, exact-amount PayNow QR, cash, today's pre-orders. | Our web app on Cloudflare Pages (free) |
| Admin dashboard | Menu and prices, recipes, beans, expenses, reports. | Our web app on Cloudflare Pages (free) |
| Café system (ERP) | The single record of sales, stock, costs and hours. Runs all scheduled jobs. | Supabase, Singapore (free) |
| Telegram bot | Your inbox: alerts, buttons, weekly summary. Talks only to the owners. | Telegram (free) |
| Payment check | Reads screenshots and compares them with the order. | Claude AI (pay per use) |
| Content assistant | Plans and designs 3–4 posts a week from your photos and ideas. | Claude + Canva, on your Claude account |
| Backups | Encrypted copy of all records every night, kept at least 5 years. | GitHub + Cloudflare R2 (free) |

### 6. Keeping it accurate and safe
Bullets, each with a bold lead phrase:
- **AI never makes up numbers.** Stock, costs, profit and reorder amounts are calculated from real records. AI only reads screenshots, explains the numbers, and helps with posts.
- **Prices are checked twice.** Every pre-order total is recalculated from your menu, so an out-of-date form can never undercharge without being noticed.
- **Payment screenshots are checked, not blindly trusted.** Duplicates and mismatches are flagged. The weekly PayNow total against your bank app catches anything a screenshot cannot prove.
- **Nothing stops service.** If the tablet or internet fails, a printed PayNow QR and a paper tally keep you going; entries are added later.
- **Records are kept safe.** Nightly encrypted backups, tested monthly, cover the 5-year record-keeping rule for Singapore businesses.
- **Customer privacy.** Only name and mobile are collected, only for pre-orders, and only to contact the customer about their order.

### 7. What we need from you
A table with columns **Item | Why it is needed**:

| Item | Why it is needed |
|---|---|
| Menu with prices and recipes | Recipes (grams of beans, millilitres of milk, cups) let the system work out cost and stock. |
| Bean suppliers, bag sizes, prices, delivery time | Needed for bean alerts and costing. |
| Default opening hours | Used for the pre-order form and the morning check-in. |
| Your PayNow name and QR code | Shown on the order form and checked against screenshots. |
| Brand kit (logo, colours, fonts, tone) | Keeps Instagram posts on-brand. |
| Instagram Business account | Lets the system learn which posts perform best (can come later). |
| A paid Claude plan on your account | Runs the weekly content assistant and connects to your Canva. |

### 8. Running costs
Start with one sentence: most services run on free plans that allow business use, and prices were checked in September 2026 and can change.

A table with columns **Item | Monthly cost**:

| Item | Monthly cost |
|---|---|
| Supabase, Cloudflare Pages, Cloudflare R2, Tally, Telegram, Google Drive, GitHub | Free |
| Claude AI for payment checks and weekly summary | A few Singapore dollars (pay per use; confirmed at build time) |
| Claude subscription on your account (content assistant) | Standard paid plan (check current price) |
| Canva Pro or Teams | Already held |
| Custom web address (optional) | About S$15 a year |

### 9. How it will be built
Start with one sentence: the system is built in six stages, each working on its own, so the café benefits early.

A table with columns **# | Stage | What you get**:

| # | Stage | What you get |
|---|---|---|
| 1 | Foundation | The café system, secure logins, menu editor, and the Telegram bot connected to your phone. |
| 2 | Counter tablet | Fast walk-in sales with PayNow QR or cash. Sales records start here. |
| 3 | Pre-orders | The Tally order form that updates itself, automatic payment checks, and hours and sold-out buttons in Telegram. |
| 4 | Beans and money | Bean alerts, profit per drink, the Sunday summary and reports. |
| 5 | Backups and weekly brief | Nightly encrypted backups and the weekly brief for the content assistant. |
| 6 | Content assistant | Claude and Canva planning and designing your posts, set up on your account. |

Close with: after stage 6, a one-week pilot with residents irons out anything awkward in real use. Until stage 3 is ready, pre-orders can run on a manually edited Tally form.

### 10. Next steps
Numbered list:
1. Read this proposal and note anything that does not match how you want to work.
2. Gather the items in section 7, starting with the menu, recipes and bean details.
3. Stage 1 begins. You will be asked to create the Telegram bot and staff logins yourself, so no passwords are shared.

---

## 5. Guardrails for ChatGPT

- **Do not invent:**
  - dates or durations for the build stages (none have been estimated);
  - exact prices for the Claude subscription or AI usage;
  - features not listed here (for example card payments, loyalty cards, delivery, or customer notifications).
- **Payments are PayNow and cash only.** There is no card payment and no payment provider.
- **Customers never use Telegram.** The bot talks only to the owners.
- **AI never generates photos of the drinks.** All post images are the barista's real photos; AI only designs layouts and writes text.
- **Out of scope (do not promise):** automatic bean ordering, automatic Instagram posting, "ask the system" chat questions, loyalty stamp cards.
- **Keep it for the owner, not for developers.** Leave out code, database table names, and API details.

## 6. Final checklist before handing back the .docx

- [ ] 10 numbered sections plus a title page, using Heading 1 and Heading 2 styles.
- [ ] Figure 1 is present, readable, and captioned.
- [ ] All five tables (sections 4, 5, 7, 8 and 9) are present with the exact rows above.
- [ ] Real Word bullets and numbering; no typed "•" characters.
- [ ] A4, 2 cm margins, page numbers in the footer.
- [ ] No emojis, no invented dates or prices, S$ used for currency.
