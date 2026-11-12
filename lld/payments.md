# LLD — `nearpay-payments`

> Version 1.13 (backend resilience audit 2026-11-12: periodic kill-switch read with version guard and divergence alert §7; rails writes without a breaker until kit fixes KIT-2 §10 — §22) · Version 1.12 (step 3.2b, spec 0.3.1: owner-filtered evidence summary in reconciliation — §21.8) · Version 1.11 (step 3.2 implementation decisions §21: invoice-session idempotency and uniqueness, minimum check, test-helper refund addresses, foreign executions, owner-filtered reconciliation) · Version 1.10 (step 3.1, spec 0.3.0: internal invoice sessions §6.3, §20) · Version 1.9 (spec 0.2.1 implementation decisions — §"Spec 0.2.1 contract changes" → "Implementation decisions") · Spec 0.2.1 update (spec 0.2.1: kill-switch read at boot, abandoning orphaned legs, observed fees, refund-leg nonce from rails, new display fields — §"Spec 0.2.1 contract changes") · Version 1.8 (Stage 2 audit fixes §19: payer client IP and link rate limits in Valkey, `max_uses` reservations, `submitting` relay state, late execution states, execution/leg consistency check, kill switch fails closed when unknown, Turnstile test secrets on local/test only) · Version 1.7 (step 2.2 implementation decisions §18: refund budgets and duplicate accounting §5, `sign_intent` nonces, reconciliation day of a statement line, ops-action envelope id, FX pricing, link success page) · Version 1.6 (step 2.1, Stage 2 contracts in `nearpay-spec` 0.2.0: data §4, refund rules §5, API §6 (console reconciliation on `/ops/…` §6.4), events §7, refund algorithm §8.4 (kill switch), jobs §9 (`reconciliation_diff` cases), decisions §17) · Version 1.5 (step 1.4, spec 0.1.5: `PublicCheckoutSession.payment_intent` §6.2, `CheckoutSession.sequence` §6.1, Turnstile test secrets §10) · Version 1.4 (step 1.2 implementation findings folded: data §4, intent and session rules §5, payer API §6.2, consumers §7, session/attempt algorithms §8, jobs §9, integrations §10, configuration §11) · Version 1.3 list filters instead of `/search` (ADR-0018) · Version 1.1 audited (`audits/payments.md`) · HLD §5.1, §7 · Domain model §2–§6 · Flows §2–§4 · Foundations `00-foundations.md`

## 1. Scope

**Owns:** CheckoutSession, PaymentIntent, PaymentAttempt, Refund (+ legs), PaymentLink, Customer, BalanceTransaction (statement), settlement-context read model, payer-facing real-time stream, daily reconciliation.
**Does not own:** quotes, pricing, tags, provider tracking, evidence (rails); relaying signed intents (agentpay); destinations and fee schedules (identity); deposit-instruction co-signing (sentinel).

## 2. Runtime

| Compose service | Command | Replicas | Notes |
|---|---|---|---|
| `payments-api` | `api` | 2 | public `/v1/checkout…`, `/v1/payment_intents…`, `/v1/refunds…`, `/v1/payment_links…`, `/v1/customers…`, `/v1/balance_transactions…`; internal `/internal/*`; SSE |
| `payments-worker` | `worker` | 1 | consumers (rails, identity, compliance), jobs, outbox relay |

## 3. Code layout

```
src/domain/checkout-session/  state + guards (open/expiring/complete/expired)
src/domain/payment-intent/     state + settlement application + overpayment rules
src/domain/payment-attempt/    state mapping from execution states
src/domain/refund/             refundable math, legs
src/domain/payment-link/ customer/ statement/
src/app/                       createSession, createAttempt, submitDepositTx, signalSent, submitSignedIntent,
                               applyExecutionUpdate, expireSession, cancelIntent, reviewDecision,
                               createRefund, prepareRefundLeg, submitRefundTx, cancelRefund, reconcileDay
src/infra/clients/             rails.ts, identity.ts, agentpay.ts, turnstile.ts
src/sse/                       stream endpoint, Valkey pub/sub fan-out
src/consumers/                 rails-executions.ts, identity-context.ts, compliance-reviews.ts
src/workers/                   session-expiry.ts, quoting-recovery.ts, reconciliation.ts, purge.ts
```

## 4. Data

Asset amounts are `numeric(78,0)` (base units can exceed 2^63, e.g. 24-decimal tokens); fiat amounts are `bigint` minor units.

```sql
create table settlement_contexts (            -- read model of identity (consumer + sync fallback)
  account_id uuid not null, livemode boolean not null, context jsonb not null,
  context_version timestamptz not null, refreshed_at timestamptz not null,
  stale boolean not null default false,       -- an identity event arrived: refresh before trusting it
  primary key (account_id, livemode));

create table geo_policy (id smallint primary key check (id = 1), version text not null, blocked_countries text[] not null,
  blocked_regions text[] not null, updated_at timestamptz not null default now());   -- cached compliance policy

create table customers (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  email_hash bytea, email_enc bytea, name_enc bytea, description text, wallets jsonb not null default '[]',
  metadata jsonb not null default '{}', deleted_at timestamptz,
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create index customers_email on customers (account_id, livemode, email_hash);

create table payment_intents (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  status text not null check (status in ('requires_payment','processing','requires_review','succeeded','canceled','expired')),
  amount bigint,                       -- fiat minor units when currency <> 'asset'
  currency text not null,              -- ISO 4217 lower-case or 'asset'
  pricing_asset_id text, pricing_asset_amount numeric(78,0),   -- when currency = 'asset'
  description text, customer_id uuid references customers(id),
  source text not null check (source in ('checkout','invoice','payment_link','api')),
  invoice_id uuid, payment_link_id uuid,
  destination_snapshot jsonb not null, -- {sd_id, asset_id, type, chain, address, attestation_hash, verification, per_payment_cap_usd_micros}
  fee_snapshot jsonb not null,         -- {schedule_id, rules[], provider_split, bearer}
  settings_snapshot jsonb not null,    -- {late_payments, underpay_tolerance_bps}
  settlement_asset_id text not null, settlement_decimals smallint not null, settlement_symbol text not null,   -- from rails asset info at creation
  amount_settled numeric(78,0) not null default 0, amount_refunded numeric(78,0) not null default 0,
  amount_refund_pending numeric(78,0) not null default 0,
  succeeded_attempt_id uuid, succeeded_at timestamptz, late boolean not null default false,
  canceled_at timestamptz, cancellation_reason text, review_reason text check (review_reason in ('short_settlement','verification_held')),
  compliance_review jsonb,             -- {id, status, reason} from evt.compliance.review.* (payment stands; no state change)
  metadata jsonb not null default '{}', sequence integer not null default 0,
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create index pi_account_created on payment_intents (account_id, livemode, created_at desc, id desc);
create index pi_status on payment_intents (account_id, livemode, status);

create table checkout_sessions (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  payment_intent_id uuid not null unique references payment_intents(id),
  status text not null check (status in ('open','expiring','complete','expired')),
  client_secret_hash bytea not null, ui_mode text not null check (ui_mode in ('hosted','embedded')),
  success_url text not null, cancel_url text, expires_at timestamptz not null,
  customer_email_enc bytea, locale text, allowed_origins text[] not null default '{}', -- embedded frame-ancestors
  completed_at timestamptz, late boolean not null default false,
  live_attempts_count smallint not null default 0, metadata jsonb not null default '{}',
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create index cs_expiry on checkout_sessions (expires_at) where status = 'open';

create table payment_attempts (
  id uuid primary key, payment_intent_id uuid not null references payment_intents(id),
  account_id uuid not null, livemode boolean not null,
  method text not null check (method in ('route_quote','intents_transfer')),
  status text not null check (status in ('quoting','awaiting_deposit','deposit_detected','incomplete_deposit','settling',
     'verifying','succeeded','refunded','failed','expired_unpaid','held')),
  visible boolean not null default true,
  origin_asset_id text not null, origin_chain text not null,
  refund_to text not null, refund_type text not null check (refund_type in ('origin_chain','intents')),
  payer_wallet text, payer_ip_country char(2),
  agent boolean not null default false,                -- created by agentpay (internal agent attempt)
  execution_id text,                                   -- rails ex_…
  execution_version integer not null default 0,
  applied_fee_bps integer, route_class text,
  settlement_amount numeric(78,0),                     -- expected output incl. amount tag
  amount_tag numeric(78,0), price_snapshot jsonb,
  deposit jsonb,                                       -- {address, memo, amount_in, min_amount_in, deadline, chain}
  intents_requirement jsonb,                           -- intents_transfer attempts
  deadline timestamptz, settled_amount numeric(78,0), refunded_amount numeric(78,0),
  evidence jsonb, failure_code text, is_overpayment boolean not null default false,
  overpayment_reason text check (overpayment_reason in ('duplicate','after_cancel','after_expiry')),
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create index pa_intent on payment_attempts (payment_intent_id);
create index pa_quoting on payment_attempts (created_at) where status = 'quoting';
create unique index pa_execution on payment_attempts (execution_id) where execution_id is not null;

create table refunds (
  id uuid primary key, payment_intent_id uuid not null references payment_intents(id),
  payment_attempt_id uuid not null references payment_attempts(id),   -- whose settlement is refunded (paying attempt or an overpayment)
  account_id uuid not null, livemode boolean not null,
  amount numeric(78,0) not null,                      -- settlement-asset base units
  basis text not null default 'settled_amount', fee_mode text not null check (fee_mode in ('payer_absorbs','merchant_absorbs')),
  reason text check (reason in ('requested_by_customer','duplicate','fraudulent','overpayment','other')),
  status text not null check (status in ('requires_action','pending','succeeded','failed','canceled')),
  method text not null check (method in ('route_quote','intents_transfer')),
  recipient text not null, recipient_asset_id text not null, recipient_chain text not null,
  recipient_type text not null check (recipient_type in ('origin_chain','intents')), recipient_overridden boolean not null default false,
  source text not null check (source in ('api','dashboard','review_rejection')),
  current_leg_id uuid, created_by text not null, failure_code text, succeeded_at timestamptz, canceled_at timestamptz,
  amount_received numeric(78,0), evidence jsonb, extra_amount_sent numeric(78,0) not null default 0,
  idempotency_key text,                               -- create replays find the row (unique per account/mode)
  metadata jsonb not null default '{}',
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create unique index refunds_idem on refunds (account_id, livemode, idempotency_key) where idempotency_key is not null;
create index refunds_intent on refunds (payment_intent_id);

create table refund_legs (
  id uuid primary key, refund_id uuid not null references refunds(id), number smallint not null,
  method text not null, swap_type text check (swap_type in ('EXACT_INPUT','EXACT_OUTPUT')),
  execution_id text unique, execution_version integer not null default 0,
  status text not null check (status in ('preparing','awaiting_signature','submitted','processing','succeeded','returned','failed','expired','held')),
  next_action jsonb, deadline timestamptz,            -- null while preparing
  amount numeric(78,0) not null, amount_out_expected numeric(78,0), signed_nonce text,
  submitted_tx text, duplicate boolean not null default false, evidence jsonb, failure_code text,
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), unique (refund_id, number));

create table payment_links (
  id uuid primary key, account_id uuid not null, livemode boolean not null, active boolean not null default true,
  amount bigint, currency text not null, pricing_asset_id text, pricing_asset_amount numeric(78,0),
  description text not null, settlement_destination_id uuid, success_url text, max_uses integer, uses integer not null default 0,
  session_expires_in integer not null default 3600, collect_email boolean not null default false,
  metadata jsonb not null default '{}', created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0);

create table balance_transactions (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  type text not null check (type in ('payment','overpayment','refund','fee_adjustment')),
  payment_intent_id uuid, refund_id uuid, payment_attempt_id uuid,
  gross_amount bigint, gross_currency text,
  settled_amount numeric(78,0) not null, settlement_asset_id text not null, settlement_decimals smallint not null, settlement_symbol text not null,
  fee_bps integer, fee_amount_est numeric(78,0), fee_asset_id text,
  evidence jsonb not null, created_at timestamptz not null default now());
create index bt_account_created on balance_transactions (account_id, livemode, created_at desc);
create unique index bt_once on balance_transactions (type, coalesce(payment_attempt_id, refund_id), coalesce(refund_leg, 0));
-- Stage 2 columns: refund_leg smallint, received jsonb ({amount, asset_id, chain}), fee_adjustment jsonb

create table reconciliation_runs (                  -- append-only: a re-run is a new row (INV-9)
  id uuid primary key, account_id uuid not null, livemode boolean not null, day date not null,
  status text not null check (status in ('running','ok','diff','error')), trigger text not null check (trigger in ('schedule','ops')),
  rerun_of uuid references reconciliation_runs(id), totals jsonb not null default '[]', diffs jsonb not null default '[]',
  error text, resolution jsonb,                      -- {classification, note, actor, resolved_at}
  started_at timestamptz not null default now(), completed_at timestamptz);
create unique index rr_running on reconciliation_runs (account_id, livemode, day) where status = 'running';
create index rr_day on reconciliation_runs (day desc, id desc);

-- transitions: checkout_session_transitions, payment_intent_transitions, payment_attempt_transitions, refund_transitions (foundations §7)
```
Stage 1 migration creates only the Stage 1 tables (no `customers`, `refunds`, `refund_legs`, `payment_links`, `reconciliation_runs`; `customer_id` has no foreign key until Stage 2).

**Statement display:** a line shows the exact settled amount including the amount tag, e.g. `2.000042 USDC (NEAR)` (decimals and symbol stored with the intent and the line).

Client secrets: `cs_…_secret_<32 base62>`; stored as SHA-256; compared constant-time.

## 5. State machines (guards and side effects)

**PaymentAttempt** mirrors the execution state from rails (`domain-model.md` §4.3). Payments never infers provider state; it only applies `rails.execution.*` events whose `aggregate.version > execution_version`.

**PaymentIntent**

| From | Trigger | Guard | To | Side effects (same tx) |
|---|---|---|---|---|
| `requires_payment` | attempt → `deposit_detected`/`settling`/`incomplete_deposit` | — | `processing` | event `processing` |
| `processing` | attempt → `refunded`/`failed`/`expired_unpaid` | no other attempt in `deposit_detected…verifying` | `requires_payment` — also when the session is `expiring` (INV-3: an older attempt may still settle); intent and session become `expired` together only once **every** attempt is terminal (session follow-up below) | event `payment_failed` (reason from attempt) |
| `requires_payment`/`processing` | attempt → `succeeded` with `settled ≥ expected × (1 − tolerance)` | — | `succeeded` | `amount_settled`, `succeeded_attempt_id`, `late` if session not `open`; session → `complete`; statement `payment`; events `succeeded`, `checkout.session.completed` |
| `requires_payment`/`processing` | attempt → `succeeded` with short amount | — | `requires_review` (`review_reason='short_settlement'`) | statement `payment` (actual amount); event `requires_review`. **First settlement wins:** that attempt is the intent's settlement; any later success is an overpayment (`duplicate`) |
| any non-terminal | attempt → `held` | — | `requires_review` (`review_reason='verification_held'`) | event `requires_review`. A held verification is **not** a settlement: the next verified success of any attempt pays the intent (`requires_review → succeeded`) |
| `requires_review` | merchant `accept` | — | `succeeded` | session `complete`; events |
| `requires_review` | merchant `reject` | — | `succeeded` stays false → `requires_payment` if session open else `expired` | refund recommended (Stage 2: auto-create draft refund) |
| `succeeded`/`canceled`/`expired`, or `requires_review` after a short settlement | another attempt → `succeeded` | — | unchanged | attempt `is_overpayment`, `overpayment_reason` (`duplicate`, `after_cancel`, `after_expiry`); statement `overpayment`; event `overpaid` |
| `requires_payment` | merchant `cancel` | no attempt in `deposit_detected…verifying` | `canceled` | hide attempts (live but undetected attempts included: a later settlement is an overpayment `after_cancel`); session → `expired`; event `canceled` |
| `requires_payment` | session expiry finalisation | all attempts terminal | `expired` | event (internal) |

**CheckoutSession:** `open → expiring` at `expires_at` (job); `expiring → expired` when every attempt is terminal and the intent is neither succeeded nor `requires_review` (a session whose intent is in review stays `expiring`); `open|expiring → complete` with the intent. Attempt creation requires `open`. Merchant `expire` (domain-model §4.1) is allowed while attempts are live but no deposit was detected: those attempts are hidden, and a later settlement is an overpayment `after_expiry`.

**Intent view:** `fee.total_bps` is the fee of the attempt that paid (or was last quoted), `0` before any quote (the spec field is not nullable).

**Refund:** per `domain-model.md` §4.4; `refundable = amount_settled − amount_refunded − amount_refund_pending` under `select … for update` on the intent; `amount_refund_pending` increments on create (and on `prepare` from `failed`), moves to `amount_refunded` on success, decrements on fail/cancel. A refund of an **overpayment** (`payment_attempt` = an `is_overpayment` attempt) uses that attempt's `settled_amount − Σ its refunds` instead and does not touch the intent's counters. Stage 2 rules (§17):

| From | Trigger | To | Side effects |
|---|---|---|---|
| — | create (intent `succeeded`, or `requires_review` after `reject`) | `requires_action` | leg 1 `preparing` → rails; event `created` (+ `requires_action` when `next_action` is ready) |
| `requires_action` | leg execution → `deposit_detected`/`settling`/`verifying` (or a signed intent relayed) | `pending` | event `pending` |
| `pending` | leg execution → `succeeded` (payer-side evidence, INV-2) | `succeeded` | counters; statement `refund` (per leg); event `succeeded` (notify: `refund.receipt`) |
| `pending`/`requires_action` | leg execution → `refunded` (returned to the merchant) / `failed` | `failed` | pending released; event `failed` |
| `requires_action` | leg deadline passed, nothing submitted | `requires_action` (leg `expired`, `next_action` null) | merchant calls `prepare` (new leg) |
| `failed` | `prepare` | `requires_action` | re-reserve (refundable check) |
| `requires_action` | `cancel` (nothing submitted on the current leg) | `canceled` | pending released; event `canceled` |
| `succeeded` | another leg settles | unchanged | leg `duplicate`, `extra_amount_sent += leg.amount`, statement `refund` line for the leg; `amount_refunded` unchanged (§18 item 1) |
| `canceled`/`failed` | an older leg still settles (merchant had sent) | `succeeded` | INV-3 (truth wins): recorded like a success; needs domain-model §4.4 edge `canceled/failed → succeeded` (requested from brain) |

## 6. API

### 6.1 Merchant API (secret / restricted keys)

| Method & path | Scope | Request → Response | Errors |
|---|---|---|---|
| `POST /v1/checkout/sessions` | `checkout_sessions.write` | `{amount, currency, asset_id?, settlement_destination?, success_url, cancel_url?, expires_in? (300–86400, default 3600), ui_mode?, customer?, customer_email?, description?, allowed_origins?, late_payments?, metadata?}` → session `{id, object, url, client_secret, status, expires_at, payment_intent, amount, currency, livemode}` | `account_not_live`, `destination_not_active`, `amount_below_destination_minimum`, `amount_above_unverified_cap`, `currency_unsupported` |
| `GET /v1/checkout/sessions/{id}` · list | `.read` | session (`expand[]=payment_intent`); `sequence` (spec 0.1.5, also on the bus `CheckoutSessionEvent`) = `checkout_sessions.version` — 1 on create, bumped by every session write (`ledger.updateSession`), equal to the session events' `aggregate.version`; independent of the intent's own `sequence` | |
| `POST /v1/checkout/sessions/{id}/expire` | `.write` | session | `state_conflict` |
| `POST /v1/payment_intents` | `payment_intents.write` | same pricing fields + `return_url` (required), `expires_in?`, `customer?`, no UI → `CreatedPaymentIntent` = intent + `checkout_session` + `client_secret` of a **headless session** (§17) for custom UIs on the payer API (`@nearpay-money/js`) (Stage 2) | as session create + `customer_deleted` |
| `GET /v1/payment_intents/{id}` · list | `.read` | intent with `attempts[]`, `overpayments[]` (+ Stage 2 `customer`, `payment_link`, `refunds[]`, `amount_refund_pending`, `review_decision`); list filters `status`, `created[gte/lte]` (Stage 1) plus `amount[gte\|gt\|lte\|lt]`, `currency`, `asset_id` (pricing asset), `settlement_asset_id`, `source`, `customer`, `payment_link`, `checkout_session`, `metadata[key]` (≤ 10, exact) (Stage 2, ADR-0018) | |
| `POST /v1/payment_intents/{id}/cancel` | `.write` | intent | `state_conflict`, `attempt_in_flight` |
| `POST /v1/payment_intents/{id}/review` | `.write` or session (owner/admin/finance) | `{decision: accept\|reject, reason?, create_refund?}` → intent; `accept` only for `short_settlement` (§17) | `state_conflict`, `review_requires_evidence` |
| `POST /v1/refunds` | `refunds.write` or session (owner/admin/finance) | `{payment_intent, payment_attempt?, amount?, reason?, fee_mode? = payer_absorbs, metadata?}` → refund (`requires_action`, `next_action` or `null` while preparing) | `intent_not_succeeded`, `attempt_not_settled`, `amount_exceeds_refundable`, preparation errors (`screening_blocked`, `screening_unavailable`, `asset_unsupported`, `route_unavailable`, `pricing_unavailable`, `quote_rejected`) |
| `GET /v1/refunds/{id}` · list | `refunds.read` (or session) | list filters `payment_intent`, `status`, `created[…]` | |
| `POST /v1/refunds/{id}/prepare` | `refunds.write` | new leg when the current one expired / was never prepared, or from `failed` → refund | `state_conflict`, `amount_exceeds_refundable`, preparation errors |
| `POST /v1/refunds/{id}/submit_tx` | `refunds.write` | `{tx_hash, sender?}` (`sign_and_send`) or `{signed_intent}` (`sign_intent`, relayed by agentpay) → refund | `state_conflict`, `leg_expired`, `signed_intent_mismatch`, `intent_invalid`, `insufficient_balance`, `rail_unavailable` |
| `POST /v1/refunds/{id}/cancel` | `refunds.write` | only `requires_action`, nothing submitted on the current leg | `state_conflict` |
| `POST /v1/refunds/{id}/recipient` | session, owner, step-up | `{address, reason}`: INV-7 override; fresh screening; new leg | `state_conflict`, `refund_address_invalid`, preparation errors |
| `POST /v1/test_helpers/refunds/{id}/simulate_deposit` | secret test key or session | sandbox `/sim/deposits` (route legs) or `/sim/intents/transfers` (Intents legs) → `202` | `livemode_forbidden`, `state_conflict`, `leg_expired` |
| `POST/GET/PATCH /v1/payment_links…` | `payment_links.*` or session (owner/admin/developer/finance) | create (pricing validated like a session), get, list (`active`), PATCH (`active`, `description`, `success_url`, `max_uses`, `collect_email`, `metadata`; pricing immutable). No delete: deactivate | session-create pricing errors |
| `POST/GET/PATCH/DELETE /v1/customers…` | `customers.*` or session (owner/admin/finance/support) | CRUD (`email`, `name`, `description`, `wallets[]`, `metadata`); list filter `email`; DELETE erases PII and returns `{id, object, deleted: true}` | `customer_deleted` |
| `GET /v1/balance_transactions` | `balance_transactions.read` (or session) | statement lines (`txn_`: type, payment_intent, refund, gross, settled, fee, evidence; Stage 2 `refund_leg`, `received`, `fee_adjustment`); filters `payment_intent`, `type`, `created[gte/lte]` (Stage 1), `refund` (Stage 2) | |

All `POST` honour `Idempotency-Key` (foundations §7).

### 6.2 Payer API (anonymous route + `X-NearPay-Client-Secret`)

| Method & path | Request → Response | Errors |
|---|---|---|
| `GET /v1/checkout/sessions/{id}/public` | → `{id, status, payment_intent, amount, currency, display:{merchant_name_unverified, description}, expires_at, settlement:{asset_id, chain, type}, attempt?}` — `payment_intent` (`pi_…`, spec 0.1.5, required) names the session's intent, so the embedded checkout's `np:complete` needs no SSE `intent` event | `client_secret_invalid` |
| `GET /v1/checkout/sessions/{id}/public/payment_options?chain=&wallet=` | → `{options:[{origin_asset_id, chain, symbol, estimated_amount_in, route_class, min_deadline_s, available}]}` (indicative, from rails with an estimated `amount_out`; options slower than the remaining window are marked `route_too_slow`). No settlement estimate or rails unreachable → empty `options` (the checkout offers the manual path) | |
| `POST /v1/checkout/sessions/{id}/public/attempts` | `{origin_asset_id, refund_to?, payer_wallet?, method: route_quote\|intents_transfer, turnstile_token}` — `route_quote` requires `refund_to` (wallet path: the connected address); `intents_transfer` requires `payer_wallet` (the payer's Intents account; refunds go there) → attempt `{id, status, method, deposit?:{address, memo, amount_in, min_amount_in, deadline, chain}, intents_requirement?:{pay_to, token, amount, fee_to, fee_amount, binding, deadline}, attestation_ticket}` | `session_not_open`, `attempt_in_flight`, `too_many_attempts`, `route_too_slow`, `refund_address_invalid`, `screening_blocked`, `geo_blocked`, `rail_unavailable`, `turnstile_failed` |
| `POST …/attempts/{pa}/deposit_tx` | `{tx_hash}` → `202` | |
| `POST …/attempts/{pa}/sent` | → `202` (deposit signal) | |
| `POST …/attempts/{pa}/signed_intent` | `{signed: MultiPayload}` → `202` (relayed by agentpay) | `intent_invalid`, `insufficient_balance` |
| `GET …/public/stream` | SSE (`text/event-stream`, client secret in header — the SPA uses a fetch-based SSE client because `EventSource` cannot send headers): `event: session` / `attempt` / `intent` snapshots (`session` is the public session above, incl. `payment_intent`); heartbeat 15 s; `retry: 3000` | |
| `POST …/attempts/{pa}/simulate_deposit` | **test mode only** → sandbox `POST /sim/deposits` → `202` | `livemode_forbidden` |
| `POST /v1/test_helpers/payment_attempts/{pa}/advance` (secret test key) | `{to: deposit\|processing\|success\|refund\|fail}` → sandbox `POST /sim/advance` → `202` | `livemode_forbidden` |
| `GET /v1/payment_links/{id}/public` · `POST …/public/sessions` (anonymous, no client secret) | → `PublicPaymentLink` (`active`, pricing, `display`, `settlement` asset/chain/type, `collect_email`) / `{customer_email?, locale?}` → `PaymentLinkSession` `{checkout_session, client_secret, url, expires_at}`; session creation rate-limited per client network, per link and client network, and per link (§19.1), `max_uses` reserved atomically (§19.2); the session is `source: payment_link` with the link's metadata | `link_inactive` (inactive or `uses ≥ max_uses`), `account_not_live`, `destination_not_active`, `currency_unsupported` |

The payer API never returns the merchant's raw destination address; it returns the asset/chain only. The co-signed merchant name and destination fingerprint come from sentinel (ADR-0010).

### 6.3 Internal

| Method & path | Caller (scope) | Purpose |
|---|---|---|
| `POST /internal/checkout_sessions` | billing (`payments.sessions.create`) | hosted invoice sessions (`createInvoiceCheckoutSession`, Stage 3, §20): `source='invoice'`, `invoice_id`, `customer`, `expires_at` 5 min–30 d ahead; returns the session with `client_secret`/`url`; one `open` session per invoice |
| `GET /internal/checkout_sessions/{id}` | billing (`payments.read`) | session in any status, no secret (Stage 3) |
| `POST /internal/checkout_sessions/{id}/expire` | billing (`payments.sessions.create`) | `{reason}`; invoice sessions only; `attempt_in_flight` like the public expire (Stage 3) |
| `GET /internal/payment_intents/{id}` | billing, agentpay (`payments.read`) | full object |
| `POST /internal/checkout_sessions/{id}/agent_attempts` | agentpay (`payments.attempts.create`) | `{payer_wallet}` → creates (or reuses the caller's still-valid, non-expired) `intents_transfer` attempt and returns `{attempt_id, execution_id, account_id, livemode, intents_requirement}`; same guards as §8.2 (limits apply per session) |
| `GET /internal/payment_intents/{id}/receipt-context` | notify (`payments.receipts.read`) | `{customer_email, merchant_display_name, amount, currency, settled, asset, evidence_links}` (PII; never in events) |
| `GET /internal/customers/{id}` | billing (`payments.read`) | customer incl. decrypted email for invoicing |
| `GET /internal/refunds/{id}` | services (`payments.read`) | full refund (Stage 2; the console reads `GET /ops/refunds/{id}`, §6.4) |
| `GET /internal/refunds/{id}/receipt-context` | notify (`payments.receipts.read`) | `RefundReceiptContext` `{customer_email, merchant_display_name, livemode, refund, payment_intent, original_amount/currency, refunded, received, recipient_display, evidence_links, email_receipts}` (Stage 2) |
| compliance `POST /internal/cases` (called by payments, `compliance.cases.write`) | — | payments opens a `reconciliation_diff` case for a `diff`/`error` run (§9) |

### 6.4 Console (`/ops/…`, visibility `ops`, served by payments — Stage 2)

Routed by the gateway only on the console host (dashboard-and-console §2); `@nearpay-internal/service` checks the ops scope, `opsRoles` (`ops`, `ops_admin`) and step-up. Every write is audited with the operator id.

| Method & path | Scope · step-up | Purpose |
|---|---|---|
| `GET /ops/reconciliation_runs` · `/{id}` | `ops:payments.read` | runs newest first; filters `account`, `livemode`, `status`, `day`, `day[gte\|lte]`, `unresolved`, cursor paging (replaces the planned `GET /internal/reconciliation/{account}/{day}`: use `account` + `day`) |
| `POST /ops/reconciliation_runs` | `ops:payments.reconcile` | `{account_id, livemode, day}` → `202` run (`running`); completed UTC days only (`reconciliation_day_open`); one running run per day (`state_conflict`); a re-run records `rerun_of` |
| `POST /ops/reconciliation_runs/{id}/resolution` | `ops:payments.reconcile` · step-up | `{classification, note}` on a `diff`/`error` run (RB-05); corrections are new statement lines, never edits (INV-9) |
| `GET /ops/refunds/{id}` | `ops:payments.read` | full refund with legs (console payment view) |

## 7. Events

| Produced | Payload object |
|---|---|
| `evt.payments.checkout_session.completed` · `.expired` | session |
| `evt.payments.payment_intent.created` · `.processing` · `.succeeded` · `.payment_failed` · `.requires_review` · `.overpaid` · `.canceled` · `.expired` | intent (+ attempts) |
| `evt.payments.refund.created` · `.requires_action` · `.pending` · `.succeeded` · `.failed` · `.canceled` | refund (consumers: webhooks → `refund.<same>` incl. `refund.pending`; notify on `succeeded`) |
Payment-link, customer and statement-line changes emit no events in v1 (no consumer; readable via API).

| Consumed | Handler |
|---|---|
| `evt.rails.execution.updated` / `.settled` / `.refunded` / `.failed` / `.held` | `applyExecutionUpdate` (version-checked) |
| `evt.identity.account.updated`, `evt.identity.settlement_destination.*` | refresh `settlement_contexts` |
| `evt.compliance.review.opened` · `.closed` (subject = payment intent, or `execution` mapped to the intent through the attempt's `execution_id`) | set intent `compliance_review` (no state change; payment stands) |
| `evt.compliance.policy.updated` | refresh cached geo policy (`GET /internal/geo-policy`) |
| `evt.sentinel.kill_switch.engaged` · `.released` (Stage 2) | set the local kill-switch flag (version-checked); while engaged no new live refund leg is prepared (§17 item 16) |

**Kill-switch copy: events plus a periodic read (v1.13, audit PAYMENTS-1).** The event is not the only update path. `payments-api` and `payments-worker` each read sentinel `getKillSwitch` at boot, every 10 s while no state is stored (unknown → live refunds paused, finding 8), and **every 60 s while a state is known** (`kill-switch-sync`). Every write to the `kill_switch` row — event or read — applies only while no row exists or when its version is newer than the stored one, so the copy never moves back to an older version and an older event or read never overrides a newer state. A missed `released` or `engaged` therefore self-heals within a minute. Refund paths never wait for sentinel while the state is known.

After each successful read the copy is compared with sentinel's answer (pure `domain/kill-switch-copy.ts`): a copy that was behind and is repaired by the read logs WARN `ALERT kill_switch_copy_diverged` `cause=copy_behind resolved=true` (INFO instead when sentinel changed < 30 s ago: the event is still in flight); a difference the read cannot repair — `sentinel_behind` (sentinel answered an older version: restored or reset database), `value_overridden` (same version, other state: a hand-edited row), `copy_behind` not applied, `copy_unknown` — logs `ALERT kill_switch_copy_diverged` on a new cause and then every 5 min, and sets `kill_switch_diverged = 1`. A failed re-read while known keeps the copy and logs a rate-limited WARN. Gauges (api and worker): `nearpay_payments_kill_switch_engaged` (1 = live refund legs paused: engaged or unknown), `nearpay_payments_kill_switch_version` (−1 unknown), `nearpay_payments_kill_switch_source_engaged`, `nearpay_payments_kill_switch_source_version` (−1 when unversioned), `nearpay_payments_kill_switch_diverged`, `nearpay_payments_kill_switch_last_read_timestamp_seconds` — deploy's `KillSwitchCopyDiverged` covers payments as well as rails.

## 8. Algorithms

### 8.1 Create checkout session
```
ctx = settlementContext(acct, lm)          // read model; if missing, stale or older than 5 min → sync GET identity (1 s), upsert
if lm and not ctx.account.live_enabled → 403 account_not_live
fiat currency ∉ NP_PRICING_CURRENCIES (Stage 1: usd; asset pricing is always accepted) → 400 currency_unsupported
dest = param ?? ctx.defaults[pricing_asset] ?? ctx.defaults['*'];  dest.status == active else 409 destination_not_active
min = rails.destinationMinimum(dest) (cached 1 h) → compared in USD micros; amount < min → 422
    rails unreachable → check skipped (economic guard only; rails re-checks on quote)
if dest.verification == 'unverified' and (usd(amount) > cap or the price cannot be valued in USD) → 422 amount_above_unverified_cap
asset info (decimals, symbol) of the settlement asset from rails → stored on the intent
if ui_mode == 'embedded' and allowed_origins empty → 400 allowed_origins_required (https origins; http://localhost:* only in test mode)
tx: insert intent(requires_payment, snapshots: destination, fee(schedule+bearer), settings) + session(open, secret hash)
    + transitions + outbox(created ×2) + job session.expire at expires_at (dedupe 'session.expire:<id>')
return session + url = PAY_ORIGIN + '/c/' + cs_id + '#' + client_secret   (fragment never reaches servers)
```

### 8.2 Create attempt (payer)
```
verify client secret; verify Turnstile (skip if Turnstile outage flag → stricter IP limit)
geo: country = CF-IPCountry (+ cf-region-code) forwarded by the gateway; blocked per cached compliance geo policy → 403 geo_blocked (local: no header → allowed)
     policy never loaded and compliance unreachable → 503 screening_unavailable (fail closed)
tx1 (lock session+intent FOR UPDATE):
  session.status == open; intent.status in (requires_payment, processing)
  no attempt in (deposit_detected, incomplete_deposit, settling, verifying) → else 409 attempt_in_flight
  lm and session.live_attempts_count >= 5 → 429 too_many_attempts
  remaining = session.expires_at − now(); if not settings.late_payments and remaining < rails.minDeadline(origin_chain) → 422 route_too_slow
  set previous visible attempts visible=false; insert attempt(quoting); live_attempts_count++
commit
exec = rails.createExecution(idem='attempt:'+id, {kind:'payment', method, account, lm, origin:{asset, chain, refund_to, refund_type},
        destination: intent.destination_snapshot, pricing:{amount,currency|asset}, fee: intent.fee_snapshot,
        deadline_cap: settings.late_payments ? null : session.expires_at, slippage_bps: default})   // 8 s timeout
  error envelope → definitive: attempt failed(failure_code) + return error, except api_error / idempotency_in_progress /
      rate_limited (attempt stays quoting → recovery job); name mapping: deadline_too_short → route_too_slow,
      invalid_request on origin.refund_to → refund_address_invalid
  timeout → leave quoting (recovery job) + 503 rail_unavailable
tx2: attempt ← exec (awaiting_deposit, deposit, settlement_amount, tag, fee bps, route_class, price_snapshot, deadline,
     execution_id, execution_version); publish SSE
ticket = Ed25519(payments_ticket_key, JCS({attempt_id, livemode, deposit_key, exp: now+15 min}))   // rate-gates sentinel (L-SN-03)
return attempt + attestation_ticket
```

### 8.3 Apply execution update (consumer)
```
tx: inbox.consume(evt.id) or skip
    attempt = by execution_id FOR UPDATE; if evt.aggregate.version <= attempt.execution_version → skip
    attempt.status = map(evt.state); copy deposit/evidence/amounts; execution_version = evt.version
    intent = FOR UPDATE; apply intent table §5 (including overpayment branch)
    session follow-up (complete / expiring→expired check)
    statement line if success/overpayment (unique bt_once)
    outbox events; Valkey publish 'cs:<id>' after commit
```

### 8.4 Refund
```
tx: intent FOR UPDATE; status == succeeded, or requires_review with a short settlement being rejected (§17)
    attempt = param payment_attempt ?? intent.succeeded_attempt; attempt has verified settled_amount else 409 attempt_not_settled
    refundable (intent, or the overpayment attempt) >= amount else 422 amount_exceeds_refundable
    recipient = attempt.refund_to + origin asset/chain, type = attempt.refund_type (intents for intents_transfer)  // INV-7
    method = attempt.method == intents_transfer ? 'intents_transfer' : 'route_quote'
    insert refund(requires_action, idempotency_key) + leg 1 (preparing); amount_refund_pending += amount; outbox created
livemode && kill switch engaged (payments consumes evt.sentinel.kill_switch.*; rails also refuses, §8.1 step 0) → 503 rail_unavailable, no leg, no recovery job (create, prepare)
leg = rails POST /internal/refund_executions (idem 'refund:'+re+':leg:'+n, {method, reference:{pi, pa, re, n},
        source: destination_snapshot, recipient, amount, swap_type: fee_mode==payer_absorbs ? EXACT_INPUT : EXACT_OUTPUT})   // 8 s
  definitive error → refund failed(failure_code), pending released, error returned
  timeout / rail_unavailable → 201 with next_action null; job refund.prepare_recovery retries with the same key
next_action (from the execution view):
  source chain_address        → sign_and_send {chain, from: destination address, to: deposit.address, asset_id, token (refund.token),
                                 symbol, decimals, amount: deposit.amount_in, memo, deadline, display}
  source intents_account      → sign_intent: payments builds the Verifier message for refund.transfer
                                 {signer_id, deadline, nonce (fresh, current salt), intents:[transfer(receiver, {token: amount}, memo=binding)]}
                                 and returns nep413 {recipient, nonce, message} + payload (ERC-191 / raw string) + parsed message
tx: leg awaiting_signature(next_action, deadline, execution_id, execution_version); event refund.requires_action
submit_tx {tx_hash}       → rails POST /internal/executions/{ex}/deposit_tx (refund stays requires_action until detection)
submit_tx {signed_intent} → equal to the issued payload (signer, nonce, message) else 422 signed_intent_mismatch;
                            live + kill switch engaged → 503 rail_unavailable (agentpay refuses the relay too, agentpay §8);
                            tx: leg awaiting_signature → submitting (row lock; cancel/prepare/recipient refused while submitting, §19.4)
                            agentpay POST /internal/relay {execution_id, binding, signed, kind:'refund', refund};
                            accepted → leg submitted (refund pending); refused (intent_invalid, insufficient_balance,
                            signed_intent_mismatch) → leg awaiting_signature; unknown outcome → leg stays submitting
execution events (version-checked, like §8.3; matched by refund_legs.execution_id):
  deposit_detected/settling/verifying → leg processing, refund pending
  succeeded (payer-side evidence)     → leg succeeded; refund succeeded (amount_received, evidence); counters; statement 'refund' line
                                         (settled = leg amount in the settlement asset, received = evidence amount; one line per leg)
  refunded / failed                   → leg returned / failed; refund failed (pending released)
  expired_unpaid                      → leg expired; next_action null (prepare again)
  held                                → leg held; refund stays pending (ops case via compliance)
  a second leg succeeding             → duplicate leg (§5)
```

## 9. Jobs

| Kind | Trigger | Behaviour |
|---|---|---|
| `session.expire` | at `expires_at` (dedupe `session.expire:<id>`) | `open → expiring`; hide visible attempts; if no non-terminal attempt and the intent is not in review → `expired` |
| `attempt.quoting_recovery` | every 15 s scan `status='quoting' and created_at < now()-20 s` | re-call rails with same idempotency key; after 2 min → `failed(rail_timeout)` |
| `context.refresh` | on identity events (the consumer marks the context `stale` and enqueues the job) | sync the account's settlement context |
| `context.sweep` | every 5 min | refresh stale contexts and contexts older than 5 min of accounts with `open`/`expiring` sessions (backstop to events) |
| `reconcile.daily` | 02:00 UTC per account with activity | compare statement vs rails `GET /internal/evidence?account&day` (payment and refund executions) per asset: inflow (`payment` + `overpayment` lines vs settled payment executions), outflow (`refund` lines vs settled refund executions), fees (statement estimates vs observed fees when rails reports them); write an append-only run (`rcn_`); a `diff`/`error` run opens a `reconciliation_diff` compliance case through compliance `POST /internal/cases` (`Idempotency-Key: rcn:<run id>`, subject `reconciliation_run`) so it lands in the console case queue; the run itself is read and resolved on `/ops/reconciliation_runs…` (§6.4, RB-05); metric + alert on any diff |
| `refund.prepare_recovery` | every 15 s: legs `preparing` older than 20 s | re-call rails with the same idempotency key; after 2 min → refund `failed(rail_timeout)` |
| `kit.purge` | hourly | idempotency, inbox, published outbox |

## 10. Integrations

| Target | Timeout | Retries | Breaker | Failure |
|---|---|---|---|---|
| rails `POST /internal/executions` (and `refund_executions`, `abandon`) | 8 s | 0 inline (recovery job retries) | no (v1.13 interim, audit PAYMENTS-2: kit 0.3.1's half-open probe can leak and lock every later call out until a restart, KIT-2; restore a breaker after the kit fix) | `503 rail_unavailable` |
| rails indicative options | 2 s | 1 | yes | empty options (manual path) |
| rails deposit tx / signals | 2 s | 1 (idempotent) | yes | `503` |
| sandbox (test mode) | 2 s | 1 | no | `503` |
| rails reads (minimums, chain profiles, assets) | 2 s | 1 | yes | minimum check skipped (§8.1) |
| compliance geo policy | 1 s | 1 | yes | cached policy; never loaded → `503 screening_unavailable` |
| identity settlement context | 1 s | 1 | yes | use cached context ≤ 1 h old; else `503` |
| agentpay `POST /internal/relay` | 5 s | 0 (idempotent on payload hash; client may resubmit) | yes | `503` |
| Turnstile siteverify | 2 s | 1 | yes | outage flag → stricter per-IP limit (F-12). Cloudflare's test secrets (`1x…AA` pass, `2x…AA` fail, `3x…AA` spent token) are answered locally when `NP_ENV` ≠ `server`, as identity does (identity §11), so an offline stack never falls into the outage limit; on the server and for real secrets siteverify is always asked |

## 11. Configuration

`NP_PAY_ORIGIN` (`https://pay.nearpay.localhost`), `NP_TICKET_SIGNING_KEY_FILE` (Ed25519; public key configured in sentinel), `NP_SANDBOX_BASE_URL`, `NP_COMPLIANCE_URL`, `NP_MAX_LIVE_ATTEMPTS` (5), `NP_SESSION_TTL_DEFAULT` (3600), `NP_TURNSTILE_SECRET_FILE` (unset → Turnstile disabled), `NP_UNDERPAY_TOLERANCE_MAX_BPS` (100), `NP_PRICING_CURRENCIES` (Stage 1: `usd`; `currency: asset` is always accepted), `NP_PII_KEK_FILE`, `NP_RAILS_URL`, `NP_AGENTPAY_URL`, `NP_IDENTITY_URL`, `NP_VALKEY_URL` (SSE fan-out `cs:<uuid>` and payment-link rate-limit counters; in-process without it), `NP_LINK_SESSIONS_PER_IP` (10), `NP_LINK_SESSIONS_PER_LINK_CLIENT` (5), `NP_LINK_SESSIONS_PER_LINK` (600) (§19.1).

## 12. Security

- Payer endpoints authenticate by client secret only; responses carry no PII and no raw destination address.
- `allowed_origins` restrict embedding (`frame-ancestors` is set by the checkout CDN per session via a signed parameter; see checkout LLD).
- Every query predicates `account_id` and `livemode`; internal routes check service scopes.
- Customer email/name envelope-encrypted; `email_hash = HMAC(kek, lower(email))` for lookup.

## 13. Observability

`…_attempts_total{method,route_class,outcome}`, `…_checkout_conversion_ratio` (sessions completed / opened, 1 h window), `…_time_to_success_seconds{route_class}` histogram, `…_overpayments_total{reason}`, `…_review_total{reason}`, `…_reconciliation_diffs_total`. Alerts: conversion drop > 30% vs 7-day baseline; any reconciliation diff; attempts stuck in `quoting` > 2 min.

## 14. Failure modes

| Failure | Outcome |
|---|---|
| Duplicate/late rails events | version check → no-op |
| Rails down during attempt | `503 rail_unavailable`; attempt stays `quoting` → recovery |
| Payer pays an older attempt | tracked; first success wins; later → `overpaid` |
| Session expires mid-payment | `expiring` until terminal; late success honoured (INV-3) |
| Identity down | sessions use cached context (≤ 1 h); new accounts cannot create sessions until context is available |

## 15. Tests

- Unit: every row of §5 tables; refundable math; fee bearer output; window rule.
- Property: random interleavings of execution events (incl. duplicates, reordering) and merchant actions never produce `amount_refunded > amount_settled`, never two `succeeded_attempt_id`, never `expired` while an attempt is non-terminal, statement Σ = Σ settled evidence.
- Integration: create session → attempt (rails mocked from spec) → apply events → webhooks events emitted; idempotent replays; SSE delivers updates.
- Sandbox scenarios 00–07 (`api-conventions.md` §8) end-to-end in testbed.

**Unified console audit trail (T-12, step 2.1 integration).** Every console write payments executes (reconciliation run create/resolve, ops refund actions) also publishes `evt.payments.ops_action.recorded` (`OpsActionEvent`, ids only, in the same transaction via the outbox); identity records it in `audit_events` so the console shows one audit trail.

## 16. Stage mapping

| Stage | Scope |
|---|---|
| 1 | sessions (hosted + embedded), intents, attempts (`route_quote`, `intents_transfer`), application of rails events incl. overpayment/late/review(read-only), statement lines and `GET /v1/balance_transactions`, SSE, test mode, agent requirements/attempts |
| 2 | refunds (+ legs, dashboard signing), payment links, customers, `POST /v1/payment_intents`, richer list filters on payment intents (ADR-0018), review accept/reject, reconciliation — contracts in §17 |
| 3 | internal invoice sessions for billing — contracts in §20 |

## 16.1 Stage 3 additions for agent commerce (step 3.1, spec 0.3.0)

- **`payCheckoutSessionTestHelper`** `POST /v1/test_helpers/checkout_sessions/{id}/pay` (test mode only; secret key, `restricted_key:checkout_sessions.write`, or session owner/admin/developer; idempotent) → `202 TestHelperPayment {checkout_session, payment_intent, payment_attempt, outcome}`. Runs §8.2 for the session without Turnstile: `method: route_quote`, `origin_asset_id` = the given one or the first available sandbox option, `refund_to` = the sandbox test address of the origin chain (sandbox §4); then, unless `outcome: none`, sandbox deposit (`amount_in`, default the quoted one) and advance to `success` / `fail` / `refund` (the same steps as `advancePaymentAttempt`). Works for any open session, including invoice sessions (`source: invoice`, §20). Errors as §8.2 (`session_not_open`, `attempt_in_flight`, `too_many_attempts`, `asset_unsupported`, `rail_unavailable`) and `livemode_forbidden`. Backs `nearpay trigger` (sdks-and-tools §3).
- `advancePaymentAttempt` also admits `restricted_key:checkout_sessions.write` and `simulateRefundDeposit` `restricted_key:refunds.write` (the CLI's restricted test key).
- **Executions not owned by payments.** Rails executions created by agentpay (cross-chain x402, rails §20) arrive on the same `evt.rails.execution.*` subjects with `owner: agentpay`: the consumer (§8.3) acks them without effect (no dead letter). Reconciliation calls `listEvidence` with `owner=payments`.

## 17. Stage 2 contracts (step 2.1, `nearpay-spec` 0.2.0)

Spec modules: public `refunds.ts`, `payment-links.ts`, `customers.ts`, `reviews.ts`, additions in `payments.ts`; internal `payments.ts`, refund legs in `rails.ts`, relay in `services.ts`, `sandbox.ts`; events `evt.payments.refund.*`. Decisions made by the contract (not in earlier versions of this LLD):

1. **Refund legs are their own rails route.** `POST /internal/refund_executions` (`CreateRefundExecution`) instead of `kind: refund` on `POST /internal/executions`: the payment request requires pricing and a fee schedule that refunds do not have, and changing that body to a union would not be additive. Reads, deposit tx and events reuse the execution routes. For `kind: refund`, `Execution.origin` is the merchant side (`refund_to` = destination address) and `Execution.settlement` the payer side (tag 0); `Execution.refund` carries `swap_type`, amounts, the source `token` and, for Intents sources, the one `RefundTransfer` to sign (with a binding). `EXACT_OUTPUT` converts `amount` into the recipient asset with rails pricing.
2. **Intents-source route legs are signed intents too.** A merchant whose destination is an Intents account refunds a cross-chain payment through a 1Click `INTENTS` deposit: `next_action` is `sign_intent` with a transfer to the deposit address (memo = binding), relayed by agentpay like an `intents_transfer` leg.
3. **`sign_intent` payload.** Payments builds the Verifier message (fresh nonce per leg) and returns the exact NEP-413 `signMessage` arguments, the exact ERC-191 / raw-ed25519 string and the parsed message; `submit_tx` accepts only that payload (`signed_intent_mismatch`). agentpay validates against `Execution.refund.transfer` (one transfer; signer = merchant) — `relaySignedIntent` gains optional `kind: refund` and `refund`.
4. **Overpayment refunds.** `payment_attempt` on create refunds a specific settled attempt (recipient = that attempt's refund address; refundable per attempt). Needed for scenario `06`.
5. **Duplicate legs.** If two legs of one refund both settle (merchant sent to an expired-then-re-prepared leg and to the new one), the second is `duplicate`, its amount is added to `extra_amount_sent` and gets its own statement line (`refund_leg`). Sandbox scenario `06` on a refund leg reproduces it (sandbox §4).
6. **Create semantics.** Definitive preparation errors are returned and the refund is recorded `failed` (retry with `prepare`); transient rail failures return `201` with `next_action: null` and a recovery job (§9). Refund creation is replay-safe through `refunds.idempotency_key`.
7. **Cancel vs truth.** `cancel` is allowed only before anything was submitted for the current leg; a deposit the merchant still makes to an open leg either returns to the destination (deadline) or, if it settles, moves the refund to `succeeded` (INV-3). This adds edges `canceled → succeeded` and `failed → succeeded` (older leg) to domain-model §4.4 — accepted at integration (domain-model v1.4).
8. **Recipient override (INV-7).** `POST /v1/refunds/{id}/recipient`: session, owner, step-up; same asset/chain/type; rails screens again (`screen: true`) when preparing the new leg.
9. **Review decisions.** `accept` is allowed for `short_settlement` only: a `verification_held` intent has no evidence and accepting it would mark it succeeded without evidence (INV-2) → `review_requires_evidence`. `reject` returns the intent to `requires_payment` (session open) or `expired`, emits `payment_intent.payment_failed` when it returns to `requires_payment`, and by default (`create_refund`) creates a `review_rejection` refund of the settled amount for the merchant to sign. The decision is on the intent as `review_decision`.
10. **`POST /v1/payment_intents` uses a headless session.** The payer API is session-scoped, so an API intent is created with a checkout session (`ui_mode: hosted`, `success_url` = `return_url`) whose `client_secret` is returned once in `CreatedPaymentIntent` (never on `PaymentIntent` or in events).
11. **Payment links** are deactivated, not deleted; `uses` counts succeeded payments; a link with `uses ≥ max_uses` answers `link_inactive`. Payer routes are anonymous (identity §11 allow-list `GET|POST /v1/payment_links/{id}/public/**`) with rate limits; attempts on the resulting session stay Turnstile-protected.
12. **Metadata filters** (`metadata[key]=value`) arrive as flat query keys; spec helper `withMetadataFilter` accepts at most 10 such keys and rejects any other unknown key; OpenAPI documents them as one `deepObject` parameter.
13. **Reconciliation runs** get the `rcn_` prefix (ops object, not public), are append-only (a re-run records `rerun_of`), and a `diff`/`error` run is the RB-05 ops case. "Zero diffs" is demonstrable as a run in status `ok` with `diff_count: 0` (console page, or `GET /ops/reconciliation_runs?account=…&day=…`). Integration decision: the runs are console (`ops`) routes, not `/internal/*` (which the gateway never routes), and a `diff`/`error` run opens a `reconciliation_diff` case in compliance (`createInternalCase`) — compliance owns cases.
14. **Reviews as webhooks.** Compliance reviews are delivered as `review.opened` / `review.closed` with `PublicReview`; a review of an execution is shown as a review of its payment intent (`Review.payment_intent`, filled by compliance from `Execution.reference`, which rails echoes from the create request — payments sends `reference` on payment executions from Stage 2). Unattributable reviews are not delivered.
15. **Refund receipts.** notify resolves the recipient via `GET /internal/refunds/{id}/receipt-context` (customer email of the intent or its session; never in events).
16. **Kill switch blocks live refunds (lead decision, integration).** While the kill switch is engaged, payments refuses to prepare new **live** refund legs (`create`, `prepare` → `503 rail_unavailable`) and agentpay refuses live refund relays (`kind: refund`, agentpay §8), fail closed: the deposit-instruction integrity concern that triggers the switch applies to refund route legs too. Legs already funded by the merchant keep being tracked and verified (truth wins, INV-3). Test mode is unaffected; refunds resume on release.

## 18. Step 2.2 implementation decisions (nearpay-payments `stage-2/2.2-payments`, spec 0.2.0, kit 0.2.0)

1. **Refund budgets; Σ refunded ≤ settled always.** A refund reserves its `amount` in a budget: the intent's
   (`amount_settled/refunded/refund_pending`) when it refunds the attempt that paid a `succeeded` intent, otherwise the
   attempt's own (`payment_attempts.amount_refunded/amount_refund_pending`, check `refunded + pending ≤ settled_amount`)
   — overpayments and rejected short settlements. Duplicate legs never touch `amount_refunded` (supersedes the §5 row
   "`amount_refunded` += …", which would violate the intent's check constraint and the invariant); they add to
   `extra_amount_sent`. A late success of a `canceled`/`failed` refund (INV-3) counts only what still fits the budget
   (`refunds.amount_counted`); the rest is `extra_amount_sent`. Property-tested (`test/property/refunds.property.test.ts`):
   every unit that left the merchant through a settled leg = counted + extra.
2. **Leg states.** A `succeeded` execution without evidence makes the leg `held` (INV-2). A merchant-reported tx hash
   marks the leg `submitted` but the refund stays `requires_action` until rails detects the deposit; a relayed signed
   payload moves the refund to `pending`. A `pending` refund whose last moving leg returns, fails or expires becomes
   `failed` (prepare again). `prepare` on a never-prepared leg re-asks rails with the same key (no second leg); a new
   leg needs the current one expired/failed/past its deadline. Legs in `preparing` are not listed in `Refund.legs`
   (the spec requires a deadline). Consumer events for a leg still `preparing` are retried until the recovery job links
   it (the next action must exist first).
3. **`sign_intent` nonce.** Payments needs the Verifier salt: live mode reads `current_salt` over NEAR RPC
   (`NP_NEAR_RPC_URLS`, 60 s cache — a new egress for payments, deploy must allow it); test mode uses the sandbox salt
   `5a0dbe11`. A salt read failure is transient (leg stays `preparing`). The payload deadline is exactly
   `RefundTransfer.deadline`. Wallets must sign the exact strings (`nep413.message` / `payload`); `next_action.message`
   is display-only (stored as jsonb, key order not preserved). `submit_tx` verifies the signature locally
   (`intent_invalid`) and byte-compares with the issued payload (`signed_intent_mismatch`) before relaying.
4. **Relay.** agentpay `POST /internal/relay` timeout 8 s with one idempotent retry; the relay happens outside any row
   lock, after the leg was marked `submitting` under it (§19.4); `getInternalRefund` is a plain read.
5. **Kill switch.** Checked before any refund row is written (create/prepare/recipient/relay → `503 rail_unavailable`,
   live only). The recovery job skips live legs while engaged (they fail with `rail_timeout` after 2 min, releasing the
   reservation). A review `reject` while engaged records the decision without a draft refund.
6. **Reconciliation.** A statement line's evidence day is its execution's `terminal_at` (new columns
   `balance_transactions.execution_id`, `terminal_at`; Stage 1 lines fall back to `created_at` and are linked through
   their attempt), so consistent data reconciles to zero diffs. Outflow compares the refund line's `settled` with
   `Execution.refund.amount` and `received` with `settled_amount`. A scheduled run starts only when rails' summary is
   `complete` (otherwise the job retries); an ops run of an incomplete day ends `error`. Runs where the evidence pages
   disagree with the summary count end `error`. Transient failures retry twice before `error`. Fees: rails reports no
   observed fee on executions, so `fees.observed` is null and `fee_mismatch` cannot fire yet (change request). Completed
   runs are immutable by trigger, except setting `resolution` and `case_id` once.
7. **Ops actions** go out with envelope `aggregate.id = aud_<action uuid>` (the outbox needs a prefixed id; same choice
   as compliance); identity dedupes on `data.object.id`.
8. **FX pricing.** `NP_PRICING_CURRENCIES` defaults to `usd,eur,gbp,chf,jpy,cad,aud`; non-USD currencies need rails
   `GET /internal/fx-rates` (absent → `currency_unsupported`; unavailable/stale or rails down → `fx_rate_unavailable`) and
   are valued with `GET /internal/settlement-estimate` for minimums, the unverified cap (fails closed) and payment options.
9. **Payment links.** A link without `success_url` sends payers to `<pay origin>/l/<plink>/complete`; `customer_email`
   is kept only when the link collects it; rate limits are shared through Valkey (superseded by §19.1).
10. **Customers.** Email lookup hash key = HMAC(KEK, "nearpay:payments:customer-email:v1"); DELETE also clears
    description, wallets and metadata; a deleted customer reads as 404. Receipt email = the session's, else the
    customer's (unless deleted), for payment and refund receipts.
11. **Execution reference.** Payment executions now send `reference {payment_intent, payment_attempt}` (§17 item 14).

## 19. Stage 2 audit fixes (nearpay-payments `fix/stage-2-audit`, migration `0003_audit_fixes.sql`)

1. **Payer client IP and anonymous link limits (audit 3).** The payer IP follows identity's rule (L-ID-07):
   `CF-Connecting-IP` on `NP_ENV=server`, the first `X-Forwarded-For` hop elsewhere, `unknown` when absent or malformed
   (used for link limits, Turnstile `remoteip` and the outage limit). `POST /v1/payment_links/{id}/public/sessions` first
   loads the link and checks it accepts payers (unknown ids → 404, inactive → `link_inactive`, nothing counted); the
   session opens in the link's own account and mode (anonymous principals carry no mode). Then three limits per minute,
   all-or-nothing (a refused request counts nowhere): per client network (`/24` IPv4 incl. IPv4-mapped, `/64` IPv6;
   `NP_LINK_SESSIONS_PER_IP`, 10), per link + client network (`NP_LINK_SESSIONS_PER_LINK_CLIENT`, 5) — one client can no
   longer 429 a link for everyone — and a generous per-link ceiling (`NP_LINK_SESSIONS_PER_LINK`, 600) that throttles
   session creation only (`GET …/public` is never limited by it). Windows are sliding, estimated from two fixed windows
   (O(1) state per key); counters live in Valkey (`np:payments:rl:*`, one Lua script per request: check all, then INCR +
   PEXPIRE 2 W; single-node Valkey per ADR-0015 — a cluster would need hash tags) with an in-process LRU fallback (50k
   keys) when Valkey is unset, down or errors.
2. **`max_uses` under concurrency (audit 12).** Opening a session from a link locks the link row (`FOR UPDATE`, inside the
   session transaction) and requires `uses + reserved < max_uses`, where `reserved` = the link's intents in
   `requires_payment | processing | requires_review` (index `pi_payment_link_open`). A reservation ends with its intent:
   succeeded → a use (`uses += 1` in the same transaction), canceled/expired → released; nothing is given back
   explicitly. A full link answers `link_inactive` ("held by a checkout in progress"). Trade-off: abandoned sessions hold
   a use until they expire (`session_expires_in`).
3. **Kill switch fails closed (audit 8).** No stored sentinel state (fresh database, or the last event aged out of the
   7-day stream before the durable consumer existed) = `unknown` = engaged for live refunds (`503 rail_unavailable`,
   reason `kill_switch_unknown`); the worker logs it at boot. Sentinel 0.2.0 has no service-readable state route
   (`getOpsKillSwitch` is ops-only), so the state becomes known with the next `evt.sentinel.kill_switch.*`; change
   request: `GET /internal/kill-switch` (`KillSwitchState`, service auth for payments/rails), read at boot.
4. **`submitting` relay state (audit 5).** A `sign_intent` leg moves `awaiting_signature → submitting` under the row lock
   before agentpay is called (transition row; no refund event). `submitting` counts as money moving: cancel, prepare
   and recipient changes refuse. Accepted → `submitted` (refund `pending`); refused before publishing (`intent_invalid`,
   `insufficient_balance`, `signed_intent_mismatch`) → back to `awaiting_signature`; unknown outcome (timeout, 5xx) →
   stays `submitting` (the payload may be on chain) — the merchant may submit the same payload again (its nonce makes a
   second relay harmless), rails' execution events move it on, or the deadline expires it. Shown as `submitted` in the
   API (the public enum has no `submitting`).
5. **Late execution states (audit 6).** A leg that is over (`failed` — preparation failed, canceled, superseded —
   `succeeded`, `returned`) follows only states where money moved (`processing`, `succeeded`, `returned`, `held`); a
   pre-deposit state (`quoting`, `awaiting_deposit`) never takes back a leg past `awaiting_signature`. Ignored states
   still link the execution to the leg (later money is found, INV-3) and write a `failed → failed` transition note;
   rails has no route to abandon an execution (change request), so nothing is shown to the merchant.
6. **Execution/leg consistency (audit 7).** Before a next action is issued: `EXACT_INPUT` (and `intents_transfer`) →
   `deposit.amount_in` / `transfer.amount` = leg amount; a transfer must be signed by the destination and move the
   settlement asset; an `intents_transfer` leg must pay the refund recipient. A mismatch fails the leg
   (`execution_mismatch`, execution still linked), releases the reservation (refund `failed`), alerts
   (`ALERT refund execution mismatch`) and answers `503 rail_unavailable`; the instructions are never shown.
7. **Turnstile test secrets (audit 2).** Answered locally only on `NP_ENV ∈ {local, test}` (allow-list); every other
   stage asks siteverify.

## Spec 0.2.1 contract changes (step 2.2/2.3 change requests, `nearpay-spec/CHANGELOG.md`)

- **Kill switch at boot:** read sentinel `getKillSwitch` (`service:sentinel.read`) when the stored state is unknown (§19 item 3); store `version` and apply only events with a higher `aggregate.version`. Still fail closed while unknown.
- **Orphaned legs:** call rails `abandonExecution` for a leg that is over before any deposit (preparation failed after rails created it, `execution_mismatch`, superseded, refund canceled) — replaces "rails has no route to abandon an execution" (§19 item 5). A late deposit still arrives as an execution event and counts as `extra_amount_sent`.
- **Fees:** `fees.observed` = Σ `Execution.observed_fee` (null fees → keep null); `fee_mismatch` can fire.
- **`sign_intent` nonce:** use `RefundTransfer.nonce` when present; keep the salt read only as fallback for rails 0.2.0 (§18 item 3).
- New response fields to fill: `DepositInstructions.fee_amount` (= `ExecutionDeposit.fee_amount_in`), `PayerAttempt.amount_received` and `PaymentAttemptSummary.origin_amount` (= `Execution.deposited_amount`), `PaymentAttemptSummary.refund_to`, `pricing_asset {asset_id, symbol, decimals}` on `PublicCheckoutSession` / `PublicPaymentLink` (rails `getInternalAsset`), `Customer.total_paid` (per pricing currency/asset, succeeded intents, refunds not subtracted) and `last_payment_at`, `PaymentLink.name` (create/update), `ReconciliationTotals.decimals`, `ReconciliationRun.account_display_name`.
- `relaySignedIntent` refusals are all pre-publish (spec doc): `intent_invalid`, `insufficient_balance`, `signed_intent_mismatch`, `rail_unavailable` → back to `awaiting_signature`; timeout/5xx → `submitting` (§19 item 4 confirmed).

### Implementation decisions (v1.9, payments `stage-2/spec-0.2.1-impl`)

1. **Kill-switch read.** `NP_SENTINEL_URL` is optional (compose already sets it); without it the state stays unknown until an event arrives. Both `api` and `worker` read `getKillSwitch` at boot when no state is stored, and while it stays unknown every live-refund guard re-reads it at most every 10 s per process. A response without `version` (sentinel 0.2.0) is stored as version 0, so any event replaces it. The store is a conditional upsert (`… on conflict do update … where kill_switch.version < excluded`): a concurrent boot read and event can never regress the state. A stored state is never replaced by a read.
2. **Orphaned legs → `abandonExecution`.** Decided by the pure rule `abandonReasonOf` (only for executions before any deposit: state `quoting`/`awaiting_deposit`, or a leg still `awaiting_signature`/failed with no newer state known): refund canceled → `refund_canceled`; leg failed `execution_mismatch` → `execution_mismatch`; `superseded` failure, or a non-current leg still awaiting its signature (recipient override, `prepare` replacing an expired current leg) → `superseded`; other failed/expired legs → `leg_failed`. Sites: the late-create path (finding 6), `failPreparation` with an execution (finding 7), cancel, recipient override, `prepare` (new leg). The call runs in the durable job `refund.abandon_leg` enqueued in the same transaction as the leg change (dedupe `refund.abandon_leg:<ex_id>`, 20 attempts, `Idempotency-Key: abandon:<ex_id>`); success sets `refund_legs.abandoned_at`/`abandon_reason` (migration 0004) and a leg transition note; `409 state_conflict` (money moved) records a note and stops — the execution keeps being tracked and a late deposit counts as `extra_amount_sent`; `404` raises an ALERT log. Rails' `expired_unpaid` event then closes an `awaiting_signature` leg as `expired` (no refund change). Abandoning moves no money, so it runs while the kill switch is engaged.
3. **`sign_intent` nonce.** `RefundTransfer.nonce` is copied unchanged into `next_action` and `signed_nonce`; the Verifier-salt read (`NP_NEAR_RPC_URLS`) remains only for a transfer without it.
4. **Fees.** `observed_fee` of payment executions only (refund executions carry no NearPay fee estimate on the statement). `fees.observed` per asset is filled only when every payment execution of the run's day reports one; otherwise it stays `null` ("incomplete") and no `fee_mismatch` is raised.
5. **`ReconciliationTotals.decimals`** from rails `getInternalAsset` (5-min cache), falling back to the statement's recorded `settlement_decimals`; omitted when neither is known. Runs completed before 0.2.1 keep their stored totals (no decimals).
6. **`ReconciliationRun.account_display_name`** from the settlement-context read model (`context.account.display_name`, refreshed on `evt.identity.account.updated`, which payments already consumes); null when payments holds no context for the account. No new consumer (`evt.identity.account.created` does not list payments as a consumer, and an account without a context has had no payments to reconcile).
7. **`Customer.total_paid` / `last_payment_at`.** Table `customer_totals (customer_id, currency, asset_id '' for fiat) → amount numeric(78,0), count` plus `customers.last_payment_at`, backfilled by migration 0004 and maintained in `ledger.updateIntent` — the one writer of `payment_intents` — in the same transaction as every intent change: the old contribution leaves, the new one enters (pure `totalsDelta`; signed upsert, so concurrent payments of one customer serialise on the row). Per the contract, refunds are **not** subtracted (refunds are settlement-asset amounts, not pricing-currency amounts). `last_payment_at = greatest(stored, succeeded_at)`. Property test: delta-maintained totals equal the recomputed Σ of succeeded intents under random cross-intent interleavings with refunds.
8. **Display fields.** `DepositInstructions.fee_amount` is stored in the attempt's `deposit` JSON (`null` for deposits stored before 0.2.1); `payment_attempts.deposited_amount` (numeric) feeds `PayerAttempt.amount_received` and `PaymentAttemptSummary.origin_amount` (an execution without the field keeps the stored value); `refund_to` is the attempt's recorded refund address. `pricing_asset` comes from rails `getInternalAsset` (cached): `null` for fiat pricing, **omitted** when rails cannot describe the asset (the payer page falls back). `PaymentLink.name` is a nullable column, never on the public link.

## 20. Stage 3 contracts (step 3.1, `nearpay-spec` 0.3.0)

Billing owns invoices; payments owns the checkout session and intent that pay one (`lld/billing.md` §11 d3–d4). Contract decisions:

1. `createInvoiceCheckoutSession` `POST /internal/checkout_sessions` body `{account_id, livemode, invoice, amount, currency, asset_id?, customer, description, settlement_destination?, success_url, expires_at, locale?, metadata?}` (pricing refined like public sessions) → `201 CheckoutSession` **including** `client_secret` and `url`. Same guards and errors as `createCheckoutSession` plus `customer_deleted`; `409 state_conflict` when the invoice already has an `open` session (enforce with a partial unique index on `payment_intents(invoice_id)` joined to open sessions, or `checkout_sessions.invoice_id` + `where status = 'open'`). Honours `Idempotency-Key`. `ui_mode` is `hosted`; `late_payments` follows the account setting.
2. `PaymentIntent.invoice` and `CheckoutSession.invoice` (optional, nullable; servers ≥ 0.3.0 send them, null unless `source = invoice`); `listPaymentIntents?invoice=`. They ride on every existing payment-intent and checkout-session event, so billing matches `evt.payments.payment_intent.succeeded|payment_failed|expired` and `evt.payments.checkout_session.expired` by `invoice` (billing is now a consumer of those subjects).
3. `getInternalCheckoutSession` `GET /internal/checkout_sessions/{id}` (`payments.read`, never the secret) and `expireInvoiceCheckoutSession` `POST /internal/checkout_sessions/{id}/expire` `{reason: invoice_voided|invoice_paid|superseded}` (`payments.sessions.create`; only sessions with `invoice`; `attempt_in_flight` / `state_conflict` like the public expire; an `expired` session is returned unchanged).
4. `getInternalCustomer` `GET /internal/customers/{id}` (`payments.read`) → `InternalCustomer` = `Customer` + `account_id` + `deleted` (deleted customers have `email`/`name` null).
5. notify does not send `payment.receipt` for `source = invoice` intents: the customer gets `invoice.paid` from billing's events instead (notify §4).

## 21. Step 3.2 implementation decisions (nearpay-payments `stage-3/3.2-payments`, spec 0.3.0, kit 0.2.1, migration `0005_stage3_invoices.sql`)

1. **One open session per invoice.** `checkout_sessions.invoice_id` (+ the existing `payment_intents.invoice_id`) and a partial unique index `(account_id, livemode, invoice_id) where status = 'open'`; the unique violation (raised inside the creating transaction, which rolls back the intent, transitions and outbox rows) maps to `409 state_conflict`. An `expiring` session does not block a new one (its attempts are hidden; a late settlement is still honoured, INV-3 — billing sees a second succeeded intent for the invoice in that case and must treat the extra as an overpayment to refund). Check constraints: `source = 'invoice'` ⇔ `invoice_id` set; invoice sessions are `hosted`.
2. **Idempotency for billing.** The service kit stores `Idempotency-Key` only for account principals; billing's service token has none, so `createInvoiceCheckoutSession` runs the kit's `idempotencyStore` itself, scoped to the body's `account_id`/`livemode` and the key namespaced `billing:<key>` (SHA-256 when longer than 255) so it never meets a merchant key. The stored response (incl. `client_secret`) is envelope-encrypted like every idempotent response; a replay returns the same session and secret. Errors release the key. `expireInvoiceCheckoutSession` is naturally idempotent (an `expired` session is returned unchanged) and needs no stored key.
3. **Guards.** `createInvoiceCheckoutSession` runs §8.1 unchanged (`openSession`): `account_not_live`, `destination_not_active`, `currency_unsupported`, `fx_rate_unavailable`, `amount_below_destination_minimum` and `amount_above_unverified_cap` are all enforced at creation (billing cannot check them at finalize). As in §8.1 the minimum check is skipped when rails cannot answer (economic guard; rails re-checks on quote, so such a session cannot be paid below the minimum); the unverified cap fails closed. `expires_at` outside 5 min–30 d → `400 invalid_request` (`param: expires_at`). The customer must exist in the same account and mode (`resource_missing`) and not be deleted (`customer_deleted`). `late_payments` = the account setting.
4. **Events.** Invoice sessions emit the ordinary events; `checkout_session.expired` is emitted on every path that ends a session unpaid (billing expire, merchant expire, intent cancel, `expires_at` with nothing in flight, `expiring → expired` when the last attempt ends, review reject). A session whose intent is in review stays `expiring` and emits nothing until the review is decided (§5).
5. **Test helper.** `payCheckoutSessionTestHelper` loads the session by id, account and test mode (another account's → 404), requires `open` with a payable intent (`session_not_open`), then quotes like §8.2 without Turnstile and geo. A requested `origin_asset_id` is resolved through rails `getInternalAsset` and quoted (rails accepts or refuses it); otherwise the first available `route_quote` option of `sessionOptions` (no option → `asset_unsupported`). Refund addresses of the simulated payer (sandbox §4 names none): EVM `0x000000000000000000000000000000000000dEaD`, NEAR `sandbox-payer.near`, Solana `11111111111111111111111111111111`, Bitcoin `bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq`. Then sandbox `simulateDeposit` (`amount_in` passed through) and `advance` to `success`/`fail`/`refund` (`none`: neither). Kit idempotency applies (merchant principal). Nothing is marked paid by the helper: the outcome arrives as rails events with evidence (INV-2).
6. **Restricted keys.** The new auth alternatives on `advancePaymentAttempt` / `simulateRefundDeposit` come from the spec route table and are enforced by the kit; no code change beyond the dependency bump.
7. **Executions of other owners.** The consumer acks an `evt.rails.execution.*` whose `owner` is set and ≠ `payments` before the inbox (no inbox row, never retried or dead-lettered); no owner (rails < 0.3.0) = payments. If such an event names an execution id linked to one of our attempts or refund legs it is still not applied and logs `ALERT` (rails mislabelled it; reconciliation reports any missing line). `applyExecutionEvent` repeats the check (defence in depth).
8. **Reconciliation.** `listEvidence` and (spec 0.3.1) `getEvidenceSummary` are both called with `owner=payments`, so agentpay's executions are in neither. The owned listing must match the summary count exactly; a row of another owner in the `owner=payments` listing (a broken rails filter) fails the run closed (`evidenceCountProblem`, property-tested); otherwise the run is `error`. The 0.3.0 fallback (relisting the day without `owner`) is removed (v1.12). Rails must serve spec 0.3.1 first: an older rails rejects `owner` on the summary (strict query schema), so runs end `error` (fail closed) until it does. The simulated payer's refund addresses of §21.5 are now the sandbox's documented test addresses (sandbox §4.1).

## 22. Changes in version 1.13 (backend resilience audit 2026-11-12, findings PAYMENTS-1/2)

- **PAYMENTS-1 periodic kill-switch read** (§7): the rails pattern (`lld/rails.md` §7 v1.18) applied to payments' copy; gauges `nearpay_payments_kill_switch_{engaged,version,source_engaged,source_version,diverged,last_read_timestamp_seconds}`.
- **PAYMENTS-2 interim** (§10): the rails write client runs without a breaker; each write stays bounded (8 s, one attempt) and is retried by its recovery job, so a rails outage costs one timed-out call per payer action instead of a permanent lockout. Reads keep their breaker until the kit KIT-2 fix is released and bumped.

