---
tags: [resources, research, architecture]
status: developing
created: 2026-09-30
modified: 2026-09-30
source: web-research-2026-09-30
---
# Similar systems review (2026-09-30)

Scrutiny of [[Projects/Operations automation architecture]] against comparable products and practice. Sources are listed in [[Resources/Source register]]. Limit: no credible first-hand write-up of a solo café owner building their own POS/ERP was found; evidence below comes from commercial products, vendor docs and industry guides.

## Findings

1. **Take App (Singapore)** already does pre-orders with PayNow, required payment-proof upload, pickup scheduling and inventory. Free: 50 orders/month. Business: USD 50/month (API/webhooks and customer data export are paid only). Implication: the pre-order flow is a proven pattern; the reason to build is integration (one dataset feeding beans, finance and the carousel), which Take App only allows on the paid plan.
2. **Loyverse** free POS covers recipes ("composite items"), stock and low-stock alerts. API plan terms not confirmed. It would split data across systems.
3. **Theoretical vs actual usage**: under 2–3% variance is normal, 3–5% needs attention, over 5% signals waste, over-portioning or bad recipe data.
4. **Bean resting**: espresso beans usually need 7–14 days' rest after roast and taste best from about week 2–3 to week 5. Reorder timing must include rest days; order size is capped by the freshness window.
5. **Instagram API**: `impressions` was replaced by `views` (deprecated April 2025); carousel views count each image. Use views, reach, saves, shares.
6. **PayNow QR**: several open-source generators support a fixed, non-editable amount (`paynowqr`, `@chewhx/paynowqr`, `@jeremyling/sg-paynow-qr`). Must be tested against real bank apps.
7. **Supabase Free**: pauses after 7 days of inactivity and has no automated backups; Pro (USD 25/month) adds backups.
8. **IRAS**: sole proprietors must keep business records for 5 years.

## Proposed design changes (2026-09-30: #5 replaced by automatic proof check; #6 resolved as nightly export; others pending confirmation at spec review)

- Bean lead time = delivery + rest days; order cap from the freshness window; optional roast date on "Arrived".
- Bean waste allowance in costing; variance bands 3% / 5%.
- IG metrics: views, reach, saves, shares.
- Reuse an existing PayNow QR library; test on major bank apps; show the order code for the transfer reference; flag duplicate screenshot uploads.
- Backups and no-pause (Supabase Pro or a nightly export) because records must be kept 5 years.
- Manual fallback mode (printed QR + later entry) so an outage never stops service; the shop's opening must not depend on the software.
- Counter screen built for rush speed: big buttons for top drinks, one-tap modifiers.

## Customer order form options (researched 2026-09-30)

Owner wants a third-party form instead of a hosted order page. Needs: menu quantities with a calculated total, screenshot upload, webhook to Supabase, free commercial use; per-slot capacity is nice-to-have.

| Tool | Free tier (as found) | Fit |
|---|---|---|
| Tally | Unlimited submissions; uploads 10 MB/file; calculations, webhooks, answer piping, conditional logic free | Best overall; no per-slot capacity (that is TallyCal, a separate product) |
| Fillout | 1,000 responses/month; uploads; calculations; webhooks; scheduling | Strong runner-up; scheduling limits are daily and for one-on-one bookings, not per-slot group capacity |
| Youform | Unlimited submissions; uploads 10 MB; webhooks | Newer Tally clone; calculations not confirmed |
| OpnForm | Unlimited submissions; webhooks | File uploads only on Pro (USD 19/month), so it fails the screenshot requirement on free |
| Zoho Forms | 3 forms, 500 submissions/month | Only option with per-option "choice availability" (real slot capacity); webhook availability on free not confirmed |
| Jotform | 5 forms, 100 submissions/month, 500 stored lifetime | Has limited-quantity options, but free limits too tight |
| Take App | 50 orders/month; PayNow + proof upload built in | API/webhooks paid (USD 50/month), so it cannot feed the ERP for free |

No form tool found that renders a PayNow QR with a per-order amount; the form shows a static QR and the computed total.

