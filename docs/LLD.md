# remindloop — Low-Level Design

## Architecture

```
Browser (magic-link auth via supabase-js CDN) -- HTTPS --> Cloudflare Worker
  (single Worker: Workers Assets + /api/* routes)
  |-- GET  /            -> static frontend (public/*)
  |-- GET  /config.js   -> {SUPABASE_URL, SUPABASE_ANON_KEY}
  |-- /api/*             -> handlers.ts, JWT verified via GoTrue /auth/v1/user
  |-- POST /api/webhooks/payment -> stop-on-payment (billplume or Razorpay direct)
  |-- cron (*/15 * * * *) -> dispatch due sequence steps
  v
Supabase Postgres (RLS) + GoTrue auth        Resend (email) + wa.me / WhatsApp
  invoices, sequences, message_log               Cloud API (post-approval)
  templates config (in-repo, not DB)
```

Flow: user adds an invoice (manual entry, or billplume posts one via the shared
webhook) -> user picks a tone preset and starts a sequence -> cron finds sequences
with `next_send_at <= now()` and `status = 'running'`, sends that step's templated
message, advances `current_step`, computes the next `next_send_at`, and marks
`completed` after the last step -> if a payment webhook fires for that invoice at any
point, the sequence transitions straight to `stopped_paid` and the cron skips it.

## Data model

```sql
create table public.profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  plan_active_until timestamptz,
  trial_ends_at timestamptz not null default (now() + interval '3 days'),
  created_at timestamptz not null default now()
);
alter table public.profiles enable row level security;
create policy "read own profile" on public.profiles for select using (auth.uid() = user_id);

create table public.invoices (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  client_name text not null,
  client_email text,
  client_phone text,                 -- E.164, for WhatsApp
  amount_paise bigint not null,
  due_date date not null,
  status text not null default 'active' check (status in ('active','paid','stopped')),
  source text not null default 'manual',   -- 'manual' | 'billplume'
  external_ref text,                       -- billplume invoice id, if linked
  created_at timestamptz not null default now()
);
alter table public.invoices enable row level security;
create policy "owner rw invoices" on public.invoices for all using (auth.uid() = user_id);
create index invoices_external_ref on public.invoices (user_id, external_ref);

create table public.sequences (
  id uuid primary key default gen_random_uuid(),
  invoice_id uuid not null references public.invoices(id) on delete cascade,
  tone_preset text not null check (tone_preset in ('friendly','firm','final')),
  current_step int not null default 0,
  status text not null default 'running'
    check (status in ('running','paused','completed','stopped_paid','stopped_manual')),
  next_send_at timestamptz not null default now(),
  created_at timestamptz not null default now(),
  unique (invoice_id)                -- one active sequence per invoice
);
alter table public.sequences enable row level security;
create policy "owner rw sequences" on public.sequences for all using (
  invoice_id in (select id from public.invoices where user_id = auth.uid())
);

create table public.message_log (
  id uuid primary key default gen_random_uuid(),
  sequence_id uuid not null references public.sequences(id) on delete cascade,
  step_number int not null,
  channel text not null check (channel in ('email','whatsapp')),
  sent_at timestamptz not null default now(),
  status text not null default 'sent' check (status in ('sent','failed'))
);
alter table public.message_log enable row level security;
create policy "owner read message_log" on public.message_log for select using (
  sequence_id in (select s.id from public.sequences s
    join public.invoices i on i.id = s.invoice_id where i.user_id = auth.uid())
);

create or replace function public.stop_sequence_on_payment(p_invoice_id uuid) returns void
language plpgsql security definer set search_path = public as $$
begin
  update invoices set status = 'paid' where id = p_invoice_id and status = 'active';
  update sequences set status = 'stopped_paid'
    where invoice_id = p_invoice_id and status in ('running','paused');
end $$;
revoke execute on function public.stop_sequence_on_payment(uuid) from public, anon, authenticated;
grant execute on function public.stop_sequence_on_payment(uuid) to service_role;

create or replace function public.grant_plan_days(p_user uuid, p_days int) returns void
language plpgsql security definer set search_path = public as $$
begin
  update profiles set plan_active_until =
    greatest(now(), coalesce(plan_active_until, now())) + (p_days || ' days')::interval
  where user_id = p_user;
end $$;
revoke execute on function public.grant_plan_days(uuid, int) from public, anon, authenticated;
grant execute on function public.grant_plan_days(uuid, int) to service_role;
```

`stop_sequence_on_payment` is idempotent by construction (the `status = 'active'` /
`status in ('running','paused')` guards make a repeat webhook call a no-op), which
matters because payment webhooks can and do fire more than once.

Tone presets (`friendly`/`firm`/`final`) map to a fixed step sequence defined in an
in-repo config (`src/templates.ts`), not a DB table — e.g. step 1 at day+3 (friendly),
step 2 at day+7 (firmer), step 3 at day+14 (final notice), each with an email subject
+ body template and a shorter WhatsApp-message variant, using placeholders like
`{{client_name}}`, `{{amount}}`, `{{days_overdue}}`.

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/invoices` | GET/POST | JWT | List / manually add an invoice to chase | 402 if plan/trial expired |
| `/api/invoices/:id/sequence` | POST | JWT | Start a sequence with a tone preset | 409 if a sequence already exists |
| `/api/invoices/:id/sequence` | PATCH | JWT | Pause/resume/stop manually | 404 |
| `/api/webhooks/payment` | POST | shared secret or Razorpay signature | Calls `stop_sequence_on_payment` | 401 bad auth; 200 no-op if already stopped |
| `/api/me` | GET | JWT | Profile + plan/trial status | — |

No LLM route — see below.

## LLM strategy

**N/A in v1.** Tone presets are pre-written templates rendered with placeholder
substitution, not LLM-generated text — deterministic, free to send, and auditable
before it goes out. LLM-personalized wording is a possible v2 stretch, explicitly
out of scope now (YAGNI).

## Frontend pages

`/` login · `/dashboard` invoice list with sequence status and days-overdue ·
`/new` manually add an invoice · `/invoice/:id` sequence detail + message log ·
`/billing` plan status + manual UPI + Razorpay checkout (post-KYC).

## Error handling + credits/refund flow

No credit spend/refund cycle — monthly-plan gate like seatsaver/billplume. The cron
dispatcher processes each due step independently and logs failures to
`message_log(status='failed')` without blocking other sequences; a failed send does
not advance `current_step` so the next cron tick retries it. The stop-on-payment path
is intentionally the highest-priority write: it always wins a race against the cron
dispatcher because both use the same `status in (...)` guard at the DB layer.

## Integrations and launch gates

**WhatsApp Business Cloud API BLOCKS automated nudges** the same way as seatsaver —
needs a verified WABA, a BSP, and approved templates for anything outside the 24h
customer-service window. Day 1 ships email (via Resend or similar) as the only
automated channel, with `wa.me` links offered as a manual-tap fallback for the
WhatsApp step. **billplume integration**: v1 accepts a webhook payload from billplume
(`external_ref` = billplume invoice id) so a billplume user can one-click "chase this"
into remindloop; remindloop still works fully standalone via manual invoice entry.

## Security notes

RLS scopes every table to the owning `auth.uid()` (via a join through `invoices` for
`sequences`/`message_log`); the Worker's service-role key is the only writer. The
payment webhook route authenticates via a shared-secret header (for billplume) or
Razorpay's HMAC signature — never trusts a bare invoice ID with no signature. Contact
fields (`client_email`, `client_phone`) are validated server-side before any send is
scheduled. Secrets (`SUPABASE_SERVICE_ROLE_KEY`, `RESEND_API_KEY`, webhook shared
secret) via `wrangler secret put`; plain vars for the Supabase URL and anon key.

## Out of scope for v1

SMS channel, LLM-personalized message wording, automated WhatsApp sending before
Business API approval, legal-escalation or collections-agency handoff, multi-currency,
custom tone-preset editing (v1 ships exactly three fixed presets), team accounts.
