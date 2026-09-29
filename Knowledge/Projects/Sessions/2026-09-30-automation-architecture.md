---
tags: [session, architecture, automation]
status: developing
created: 2026-09-30
modified: 2026-09-30
source: owner-conversation-2026-09-29
---
# Automation architecture brainstorm — 2026-09-29/30 (Claude Code)

## Request
Design an agentic architecture so the partner only makes coffee, handles pre-orders, orders beans, checks cost/revenue, and posts a weekly AI-built IG carousel. List what is needed and dependencies.

## Decisions
Recorded in [[Projects/Scope decisions]]; design detail in [[Projects/Operations automation architecture]].

## Evidence
- Owner answers to clarifying questions in this conversation (format, location, pre-order channel, walk-in capture, carousel imagery, bean automation, notification channel, stack, payment flow).
- Payment provider fees checked 2026-09-29 (Stripe, HitPay pricing pages) before the owner chose no provider; see [[Resources/Source register]].

## Checks
None. Design only; no code written.

## Unresolved
See [[Projects/Open questions]]. IG Business account and brand kit pending from the partner.

## Update 2026-09-30
Owner answered the open questions (premises settled, personal PayNow, IG Business and brand kit later) and approved section 1 with one change: no bot messages to customers. Auto-capture drafts in `Knowledge/Inbox/Auto/` were reviewed: they contained only handoff snapshots, but show real Stop and SessionEnd hook events fired 2026-09-29 16:17–16:20 UTC. Drafts deleted after curation.

## Next action
Present section 2 (data model; beans, finance, insights logic), then section 3 (carousel agent).
