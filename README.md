# remindloop

Polite, escalating invoice-chasing sequences via email/WhatsApp that stop the
moment payment lands.

**Status: planned — not yet built (50-SaaS challenge #64)**

## The problem

Freelancers hate chasing late payers, so they either don't (and eat the loss) or
send one awkward message and give up. A pre-written, tone-escalating sequence that
fires on autopilot — and stops instantly when the invoice is paid — removes both the
awkwardness and the manual tracking.

## Target buyer

Freelancers and small agencies with overdue invoices, especially those already using
(or willing to use) billplume for invoicing.

## Pricing hypothesis

Rs299/mo, flat. 3-day free trial on signup.

## Stack

Cloudflare Worker (TypeScript, assets + `/api/*` in one Worker) fronting Supabase
(Postgres + RLS, magic-link auth). No LLM — reminder copy is pre-written templates
per tone preset, not generated. Monetization is a monthly plan gate, same pattern as
seatsaver/billplume/coldquill in this batch.

## How to continue this build

Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the
ordered build tasks. `CLAUDE.md` points an agent at the shared conventions in the
private reference repo.

## Risks / constraints

- **Sequences are a state machine in Postgres**, not ad-hoc cron logic — a sequence
  must always be in exactly one of a known set of states, and the stop-on-payment
  transition must be atomic and idempotent (a webhook can fire more than once).
- **WhatsApp automation is gated** the same way as seatsaver: proactive nudges need
  Meta's WhatsApp Business Cloud API (verified WABA, BSP, approved templates). Day 1
  ships email-only plus `wa.me` manual-tap links; verify current Meta/BSP terms
  before building.
- **Tone presets are templates, not LLM output.** Deterministic, auditable, no
  per-message generation cost or risk of an off-tone message going out unsupervised.
- **Pairs with billplume** (sibling product in this batch) but must also work
  standalone against manually-entered invoices — don't hard-couple to it.
