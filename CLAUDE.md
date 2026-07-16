This repo is product #64 of Febin's 50-SaaS challenge. Nothing is built yet —
docs/ is the source of truth. Read docs/LLD.md then docs/PLAN.md and execute tasks
in order with TDD.

Stack conventions + the shipped reference implementation live in the private repo
febufenn-cyber/50-saas (contract-reviewer/) — same Worker+Supabase+RLS patterns,
adapted here from per-op credits to a monthly plan gate (see LLD).

The sequence state machine, the idempotent stop-on-payment RPC, and the
templates-not-LLM decision in the LLD are verified — do not re-litigate without
evidence.
