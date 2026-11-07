# LLD — `nearpay-payments`

> Version 1.6 (step 2.1, Stage 2 contracts in `nearpay-spec` *Unreleased*: data §4, refund rules §5, API §6, events §7, refund algorithm §8.4, jobs §9, decisions §17) · Version 1.5 (step 1.4, spec 0.1.5: `PublicCheckoutSession.payment_intent` §6.2, `CheckoutSession.sequence` §6.1, Turnstile test secrets §10) · Version 1.4 (step 1.2 implementation findings folded: data §4, intent and session rules §5, payer API §6.2, consumers §7, session/attempt algorithms §8, jobs §9, integrations §10, configuration §11) · Version 1.3 list filters instead of `/search` (ADR-0018) · Version 1.1 audited (`audits/payments.md`) · HLD §5.1, §7 · Domain model §2–§6 · Flows §2–§4 · Foundations `00-foundations.md`

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
| `succeeded` | another leg settles | unchanged | leg `duplicate`, `extra_amount_sent += leg.amount`, statement `refund` line for the leg, `amount_refunded` += (may exceed the original refund) |
| `canceled`/`failed` | an older leg still settles (merchant had sent) | `succeeded` | INV-3 (truth wins): recorded like a success; needs domain-model §4.4 edge `canceled/failed → succeeded` (requested from brain) |

## 6. API

### 6.1 Merchant API (secret / restricted keys)

| Method & path | Scope | Request → Response | Errors |
|---|---|---|---|
| `POST /v1/checkout/sessions` | `checkout_sessions.write` | `{amount, currency, asset_id?, settlement_destination?, success_url, cancel_url?, expires_in? (300–86400, default 3600), ui_mode?, customer?, customer_email?, description?, allowed_origins?, late_payments?, metadata?}` → session `{id, object, url, client_secret, status, expires_at, payment_intent, amount, currency, livemode}` | `account_not_live`, `destination_not_active`, `amount_below_destination_minimum`, `amount_above_unverified_cap`, `currency_unsupported` |
| `GET /v1/checkout/sessions/{id}` · list | `.read` | session (`expand[]=payment_intent`); `sequence` (spec 0.1.5, also on the bus `CheckoutSessionEvent`) = `checkout_sessions.version` — 1 on create, bumped by every session write (`ledger.updateSession`), equal to the session events' `aggregate.version`; independent of the intent's own `sequence` | |
| `POST /v1/checkout/sessions/{id}/expire` | `.write` | session | `state_conflict` |
| `POST /v1/payment_intents` | `payment_intents.write` | same pricing fields + `return_url` (required), `expires_in?`, `customer?`, no UI → `CreatedPaymentIntent` = intent + `checkout_session` + `client_secret` of a **headless session** (§17) for custom UIs on the payer API (`@nearpay/js`) (Stage 2) | as session create + `customer_deleted` |
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
| `GET /v1/payment_links/{id}/public` · `POST …/public/sessions` (anonymous, no client secret) | → `PublicPaymentLink` (`active`, pricing, `display`, `settlement` asset/chain/type, `collect_email`) / `{customer_email?, locale?}` → `PaymentLinkSession` `{checkout_session, client_secret, url, expires_at}`; per-IP and per-link rate limits; the session is `source: payment_link` with the link's metadata | `link_inactive` (inactive or `uses ≥ max_uses`), `account_not_live`, `destination_not_active`, `currency_unsupported` |

The payer API never returns the merchant's raw destination address; it returns the asset/chain only. The co-signed merchant name and destination fingerprint come from sentinel (ADR-0010).

### 6.3 Internal

| Method & path | Caller (scope) | Purpose |
|---|---|---|
| `POST /internal/checkout_sessions` | billing (`payments.sessions.create`) | hosted invoice sessions (`source='invoice'`, `invoice_id`, `expires_at` = invoice void date or +30 d) |
| `GET /internal/payment_intents/{id}` | billing, agentpay (`payments.read`) | full object |
| `POST /internal/checkout_sessions/{id}/agent_attempts` | agentpay (`payments.attempts.create`) | `{payer_wallet}` → creates (or reuses the caller's still-valid, non-expired) `intents_transfer` attempt and returns `{attempt_id, execution_id, account_id, livemode, intents_requirement}`; same guards as §8.2 (limits apply per session) |
| `GET /internal/payment_intents/{id}/receipt-context` | notify (`payments.receipts.read`) | `{customer_email, merchant_display_name, amount, currency, settled, asset, evidence_links}` (PII; never in events) |
| `GET /internal/customers/{id}` | billing (`payments.read`) | customer incl. decrypted email for invoicing |
| `GET /internal/refunds/{id}` | services, ops (`payments.read`) | full refund (Stage 2) |
| `GET /internal/refunds/{id}/receipt-context` | notify (`payments.receipts.read`) | `RefundReceiptContext` `{customer_email, merchant_display_name, livemode, refund, payment_intent, original_amount/currency, refunded, received, recipient_display, evidence_links, email_receipts}` (Stage 2) |
| `GET /internal/reconciliation_runs` · `/{id}` | ops console (`ops:payments.read`) | runs newest first; filters `account_id`, `livemode`, `status`, `day`, `day[gte\|lte]`, `unresolved` (replaces the planned `GET /internal/reconciliation/{account}/{day}`: use `account_id` + `day`) |
| `POST /internal/reconciliation_runs` | ops console (`ops:payments.reconcile`) | `{account_id, livemode, day}` → `202` run (`running`); completed UTC days only (`reconciliation_day_open`); one running run per day (`state_conflict`) |
| `POST /internal/reconciliation_runs/{id}/resolution` | ops console (`ops:payments.reconcile`) | `{classification, note}` on a `diff`/`error` run (RB-05) |

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
                            agentpay POST /internal/relay {execution_id, binding, signed, kind:'refund', refund}; leg submitted
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
| `reconcile.daily` | 02:00 UTC per account with activity | compare statement vs rails `GET /internal/evidence?account&day` (payment and refund executions) per asset: inflow (`payment` + `overpayment` lines vs settled payment executions), outflow (`refund` lines vs settled refund executions), fees (statement estimates vs observed fees when rails reports them); write an append-only run (`rcn_`); a `diff`/`error` run is the ops case (console "Reconciliation runs", RB-05; resolution via `…/resolution`); metric + alert on any diff |
| `refund.prepare_recovery` | every 15 s: legs `preparing` older than 20 s | re-call rails with the same idempotency key; after 2 min → refund `failed(rail_timeout)` |
| `kit.purge` | hourly | idempotency, inbox, published outbox |

## 10. Integrations

| Target | Timeout | Retries | Breaker | Failure |
|---|---|---|---|---|
| rails `POST /internal/executions` | 8 s | 0 inline (recovery job retries) | per account? no — global | `503 rail_unavailable` |
| rails indicative options | 2 s | 1 | yes | empty options (manual path) |
| rails deposit tx / signals | 2 s | 1 (idempotent) | yes | `503` |
| sandbox (test mode) | 2 s | 1 | no | `503` |
| rails reads (minimums, chain profiles, assets) | 2 s | 1 | yes | minimum check skipped (§8.1) |
| compliance geo policy | 1 s | 1 | yes | cached policy; never loaded → `503 screening_unavailable` |
| identity settlement context | 1 s | 1 | yes | use cached context ≤ 1 h old; else `503` |
| agentpay `POST /internal/relay` | 5 s | 0 (idempotent on payload hash; client may resubmit) | yes | `503` |
| Turnstile siteverify | 2 s | 1 | yes | outage flag → stricter per-IP limit (F-12). Cloudflare's test secrets (`1x…AA` pass, `2x…AA` fail, `3x…AA` spent token) are answered locally when `NP_ENV` ≠ `server`, as identity does (identity §11), so an offline stack never falls into the outage limit; on the server and for real secrets siteverify is always asked |

## 11. Configuration

`NP_PAY_ORIGIN` (`https://pay.nearpay.localhost`), `NP_TICKET_SIGNING_KEY_FILE` (Ed25519; public key configured in sentinel), `NP_SANDBOX_BASE_URL`, `NP_COMPLIANCE_URL`, `NP_MAX_LIVE_ATTEMPTS` (5), `NP_SESSION_TTL_DEFAULT` (3600), `NP_TURNSTILE_SECRET_FILE` (unset → Turnstile disabled), `NP_UNDERPAY_TOLERANCE_MAX_BPS` (100), `NP_PRICING_CURRENCIES` (Stage 1: `usd`; `currency: asset` is always accepted), `NP_PII_KEK_FILE`, `NP_RAILS_URL`, `NP_AGENTPAY_URL`, `NP_IDENTITY_URL`, `NP_VALKEY_URL` (SSE fan-out `cs:<uuid>`; in-process without it).

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

## 16. Stage mapping

| Stage | Scope |
|---|---|
| 1 | sessions (hosted + embedded), intents, attempts (`route_quote`, `intents_transfer`), application of rails events incl. overpayment/late/review(read-only), statement lines and `GET /v1/balance_transactions`, SSE, test mode, agent requirements/attempts |
| 2 | refunds (+ legs, dashboard signing), payment links, customers, `POST /v1/payment_intents`, richer list filters on payment intents (ADR-0018), review accept/reject, reconciliation — contracts in §17 |
| 3 | internal invoice sessions for billing |

## 17. Stage 2 contracts (step 2.1, `nearpay-spec` *Unreleased*)

Spec modules: public `refunds.ts`, `payment-links.ts`, `customers.ts`, `reviews.ts`, additions in `payments.ts`; internal `payments.ts`, refund legs in `rails.ts`, relay in `services.ts`, `sandbox.ts`; events `evt.payments.refund.*`. Decisions made by the contract (not in earlier versions of this LLD):

1. **Refund legs are their own rails route.** `POST /internal/refund_executions` (`CreateRefundExecution`) instead of `kind: refund` on `POST /internal/executions`: the payment request requires pricing and a fee schedule that refunds do not have, and changing that body to a union would not be additive. Reads, deposit tx and events reuse the execution routes. For `kind: refund`, `Execution.origin` is the merchant side (`refund_to` = destination address) and `Execution.settlement` the payer side (tag 0); `Execution.refund` carries `swap_type`, amounts, the source `token` and, for Intents sources, the one `RefundTransfer` to sign (with a binding). `EXACT_OUTPUT` converts `amount` into the recipient asset with rails pricing.
2. **Intents-source route legs are signed intents too.** A merchant whose destination is an Intents account refunds a cross-chain payment through a 1Click `INTENTS` deposit: `next_action` is `sign_intent` with a transfer to the deposit address (memo = binding), relayed by agentpay like an `intents_transfer` leg.
3. **`sign_intent` payload.** Payments builds the Verifier message (fresh nonce per leg) and returns the exact NEP-413 `signMessage` arguments, the exact ERC-191 / raw-ed25519 string and the parsed message; `submit_tx` accepts only that payload (`signed_intent_mismatch`). agentpay validates against `Execution.refund.transfer` (one transfer; signer = merchant) — `relaySignedIntent` gains optional `kind: refund` and `refund`.
4. **Overpayment refunds.** `payment_attempt` on create refunds a specific settled attempt (recipient = that attempt's refund address; refundable per attempt). Needed for scenario `06`.
5. **Duplicate legs.** If two legs of one refund both settle (merchant sent to an expired-then-re-prepared leg and to the new one), the second is `duplicate`, its amount is added to `extra_amount_sent` and gets its own statement line (`refund_leg`). Sandbox scenario `06` on a refund leg reproduces it (sandbox §4).
6. **Create semantics.** Definitive preparation errors are returned and the refund is recorded `failed` (retry with `prepare`); transient rail failures return `201` with `next_action: null` and a recovery job (§9). Refund creation is replay-safe through `refunds.idempotency_key`.
7. **Cancel vs truth.** `cancel` is allowed only before anything was submitted for the current leg; a deposit the merchant still makes to an open leg either returns to the destination (deadline) or, if it settles, moves the refund to `succeeded` (INV-3). This adds edges `canceled → succeeded` and `failed → succeeded` (older leg) to domain-model §4.4 — requested from the brain owner.
8. **Recipient override (INV-7).** `POST /v1/refunds/{id}/recipient`: session, owner, step-up; same asset/chain/type; rails screens again (`screen: true`) when preparing the new leg.
9. **Review decisions.** `accept` is allowed for `short_settlement` only: a `verification_held` intent has no evidence and accepting it would mark it succeeded without evidence (INV-2) → `review_requires_evidence`. `reject` returns the intent to `requires_payment` (session open) or `expired`, emits `payment_intent.payment_failed` when it returns to `requires_payment`, and by default (`create_refund`) creates a `review_rejection` refund of the settled amount for the merchant to sign. The decision is on the intent as `review_decision`.
10. **`POST /v1/payment_intents` uses a headless session.** The payer API is session-scoped, so an API intent is created with a checkout session (`ui_mode: hosted`, `success_url` = `return_url`) whose `client_secret` is returned once in `CreatedPaymentIntent` (never on `PaymentIntent` or in events).
11. **Payment links** are deactivated, not deleted; `uses` counts succeeded payments; a link with `uses ≥ max_uses` answers `link_inactive`. Payer routes are anonymous (identity §11 allow-list `GET|POST /v1/payment_links/{id}/public/**`) with rate limits; attempts on the resulting session stay Turnstile-protected.
12. **Metadata filters** (`metadata[key]=value`) arrive as flat query keys; spec helper `withMetadataFilter` accepts at most 10 such keys and rejects any other unknown key; OpenAPI documents them as one `deepObject` parameter.
13. **Reconciliation runs** get the `rcn_` prefix (ops object, not public), are append-only (a re-run records `rerun_of`), and a `diff`/`error` run is the RB-05 ops case. "Zero diffs" is demonstrable as a run in status `ok` with `diff_count: 0` (console page, or `GET /internal/reconciliation_runs?account_id=…&day=…`).
14. **Reviews as webhooks.** Compliance reviews are delivered as `review.opened` / `review.closed` with `PublicReview`; a review of an execution is shown as a review of its payment intent (`Review.payment_intent`, filled by compliance from `Execution.reference`, which rails echoes from the create request — payments sends `reference` on payment executions from Stage 2). Unattributable reviews are not delivered.
15. **Refund receipts.** notify resolves the recipient via `GET /internal/refunds/{id}/receipt-context` (customer email of the intent or its session; never in events).
