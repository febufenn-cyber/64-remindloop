# remindloop — Build Plan

TDD throughout: write the failing test first, then the code that passes it. Each
task below is sized for one focused agent session.

## T1 — Project scaffold

Files: `wrangler.jsonc`, `package.json`, `tsconfig.json`, `src/worker.ts`,
`public/index.html`, `.dev.vars` (gitignored), `supabase/config.toml`.
Interfaces: `export default { fetch(req, env, ctx) }` returning a static
"not built yet" page.
Test: `test/worker.test.ts` — GET `/` returns 200.
Done: `wrangler dev` serves the page; `npm test` green.

## T2 — Supabase schema + RLS

Files: `supabase/migrations/0001_init.sql` (profiles, invoices, sequences,
message_log, `stop_sequence_on_payment`, `grant_plan_days`, RLS exactly as in
`docs/LLD.md`).
Test: migration applies against a local/test Supabase; RLS test queries as user B
and asserts 0 rows for user A's sequences; `unique(invoice_id)` on sequences
confirmed by attempting a duplicate insert.
Done: migration applies cleanly; both tests pass.

## T3 — Template config + renderer

Files: `src/templates.ts` (tone -> step list, each with `{dayOffset, emailSubject,
emailBody, whatsappBody}` template strings), `src/render.ts`
(`renderTemplate(template, invoice): {subject, body}` doing placeholder
substitution).
Test: `test/render.test.ts` — asserts every placeholder in every template resolves
against a sample invoice with no leftover `{{...}}` in the output.
Done: all three tone presets render cleanly for a fixture invoice.

## T4 — Supa client + sequence state machine

Files: `src/supa.ts` (`Supa.insertInvoice`, `startSequence`, `patchSequence`,
`getDueSteps`, `advanceStep`), `src/handlers.ts`, `src/router.ts`.
Interfaces:
```ts
function nextTransition(current: SequenceStatus, action: "pause"|"resume"|"stop"): SequenceStatus | null;
```
Test: `test/state-machine.test.ts` — table-driven test of every legal and illegal
transition (e.g. `completed -> running` must return `null`/reject).
Done: illegal transitions are rejected at the handler layer, not just the DB check
constraint.

## T5 — Cron dispatcher

Files: `src/cron.ts` (`dispatchDueSteps(supa, now): Promise<{sent: number, failed:
number}>`).
Test: `test/cron.test.ts` — mocked clock + mocked Supa + mocked email client:
asserts a due step sends, advances `current_step`, and computes the correct next
`next_send_at`; asserts a send failure logs to `message_log` without advancing the
step (so the next tick retries).
Done: cron logic fully tested with no live network calls; last step completion sets
`status = 'completed'`.

## T6 — Stop-on-payment webhook

Files: `src/handlers.ts` (`/api/webhooks/payment`), signature/shared-secret
verification.
Test: `test/webhook.test.ts` — asserts a valid webhook calls
`stop_sequence_on_payment`; asserts calling it twice for the same invoice is a safe
no-op (idempotency); asserts bad auth returns 401 without touching the DB.
Done: race between a webhook and the cron dispatcher is proven safe by a test that
fires both against the same fixture invoice.

## T7 — WhatsApp link + billplume webhook intake

Files: `src/whatsapp.ts` (`buildWaMeLink`), `src/handlers.ts`
(`/api/webhooks/billplume-invoice` intake, mapping `external_ref`).
Test: `test/whatsapp.test.ts`, `test/intake.test.ts` — asserts correct `wa.me`
encoding and correct dedup on `external_ref` (posting the same billplume invoice
twice doesn't create two rows).
Done: both flows unit-tested; standalone manual-entry path still works unchanged.

## T8 — Deploy + live smoke test + launch checklist

Files: `scripts/smoke.ts`, `scripts/grant-plan.ts`.
Test: `scripts/smoke.ts` against the deployed Worker — add an invoice, start a
sequence, fire the payment webhook, confirm the sequence is `stopped_paid` and the
cron skips it on the next tick.
Done: `wrangler deploy` succeeds; smoke green; secrets set via `wrangler secret
put`; WhatsApp-gate disclaimer visible in the UI; pricing/trial copy on the billing
page.
