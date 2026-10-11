---
tags: [resources, costs, planning]
status: developing
created: 2026-10-11
modified: 2026-10-11
source: price-checks-2026-10-11
---
# Cost projection: systems (initial, 2026-10-11)

Scope: software, hosting, AI and devices for the system in spec v2 (`docs/superpowers/specs/2026-10-11-operations-automation-design-v2.md`). **Excludes** café stock and equipment (beans, milk, bottles, labels, brewing gear). Exchange-rate assumption: **US$1 ≈ S$1.30** (not checked). Sources: [[Resources/Source register]].

## Phase A — build locally (now)

| Item | Cost |
|---|---|
| Supabase local stack (Docker), Vite dev servers, test Telegram bot | **S$0** |
| Claude Code / Codex to build | Existing subscriptions. If one runs out of usage: Claude Pro US$20/month (US$17/month billed annually); Max from US$100/month |
| Build effort (rough) | About 20–30 agent-hours for all 7 plans, plus about 10–15 hours of owner review and setup |

## Phase B — go live (one-time)

| Item | Cost |
|---|---|
| Domain `.com` (Cloudflare Registrar, at cost) | ~US$10–11/year ≈ **S$14/year** (rough; Cloudflare lists no price on its page) |
| Domain `.sg` (alternative) | Higher, roughly S$40–60/year (unchecked); not confirmed on Cloudflare |
| Tablet, if none is spare | Roughly S$250–600 (unchecked) |

## Phase C — running costs (monthly, after launch)

| Item | Plan | Monthly |
|---|---|---|
| Supabase (Singapore) | Free | S$0 (upgrade path: Pro US$25 ≈ S$33) |
| Cloudflare Pages ×2, Turnstile, R2 backups (10 GB free) | Free | S$0 |
| Telegram bot | Free | S$0 |
| GitHub private repo + Actions (2,000 min/month free) | Free | S$0 |
| Google Drive (brief + photos) | Free tier | S$0 |
| Claude API: payment-proof checks (~300/month, Haiku 5.5 at US$0.10/US$0.50 per M tokens) + 4 digests (Sonnet 5.5 at US$2/US$10) | Pay per use | **< US$1 ≈ S$1** |
| Barista's Claude plan (content agent, Cowork + Canva) | Pro | **US$20 ≈ S$26** (US$17 if billed annually) |
| Canva Pro/Teams | Already held | S$0 new |
| **Total** | | **≈ S$27/month** |

**API estimate basis:**
- One screenshot check ≈ 2,000 input tokens (image ≈ width × height / 750; large screenshots are downscaled, so ~1,600 tokens) + 200 output tokens. That costs ≈ US$0.0003 on Haiku 5.5.
- One digest ≈ 5,000 input + 600 output tokens. That costs ≈ US$0.02 on Sonnet 5.5.

## Year-one estimate

≈ S$14 (domain) + 12 × S$27 ≈ **S$340**, plus any extra build-tool subscription months and a tablet if needed.

**Main cost risks:**
- Supabase Pro (+S$33/month) if free limits or pausing become a problem.
- Heavier AI models than planned.
- The Claude plan price changing.
