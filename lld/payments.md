# LLD — `nearpay-payments`

> Version 1.1 (audited, `audits/payments.md`) · HLD §5.1, §7 · Domain model §2–§6 · Flows §2–§4 · Foundations `00-foundations.md`

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
  context_version timestamptz not null, refreshed_at timestamptz not null, primary key (account_id, livemode));

create table customers (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  email_hash bytea, email_enc bytea, name_enc bytea, wallets jsonb not null default '[]',
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
  settlement_asset_id text not null,
  amount_settled numeric(78,0) not null default 0, amount_refunded numeric(78,0) not null default 0,
  amount_refund_pending numeric(78,0) not null default 0,
  succeeded_attempt_id uuid, succeeded_at timestamptz, late boolean not null default false,
  canceled_at timestamptz, cancellation_reason text, review_reason text,
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
  execution_id text,                                   -- rails ex_…
  execution_version integer not null default 0,
  applied_fee_bps integer, route_class text,
  settlement_amount numeric(78,0),                     -- expected output incl. amount tag
  amount_tag numeric(78,0), price_snapshot jsonb,
  deposit jsonb,                                       -- {address, memo, amount_in, min_amount_in, deadline, chain}
  deadline timestamptz, settled_amount numeric(78,0), refunded_amount numeric(78,0),
  evidence jsonb, failure_code text, is_overpayment boolean not null default false,
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create index pa_intent on payment_attempts (payment_intent_id);
create index pa_quoting on payment_attempts (created_at) where status = 'quoting';
create unique index pa_execution on payment_attempts (execution_id) where execution_id is not null;

create table refunds (
  id uuid primary key, payment_intent_id uuid not null references payment_intents(id),
  account_id uuid not null, livemode boolean not null,
  amount numeric(78,0) not null,                      -- settlement-asset base units
  basis text not null default 'settled_amount', fee_mode text not null check (fee_mode in ('payer_absorbs','merchant_absorbs')),
  reason text check (reason in ('requested_by_customer','duplicate','fraudulent','overpayment','other')),
  status text not null check (status in ('requires_action','pending','succeeded','failed','canceled')),
  recipient text not null, recipient_asset_id text not null, recipient_chain text not null,
  current_leg_id uuid, created_by text not null, failure_code text, succeeded_at timestamptz,
  metadata jsonb not null default '{}',
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);

create table refund_legs (
  id uuid primary key, refund_id uuid not null references refunds(id),
  execution_id text, status text not null, next_action jsonb not null, deadline timestamptz not null,
  submitted_tx text, created_at timestamptz not null default now(), updated_at timestamptz not null default now());

create table payment_links (
  id uuid primary key, account_id uuid not null, livemode boolean not null, active boolean not null default true,
  amount bigint, currency text not null, pricing_asset_id text, pricing_asset_amount numeric(78,0),
  description text not null, settlement_destination_id uuid, success_url text, max_uses integer, uses integer not null default 0,
  metadata jsonb not null default '{}', created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0);

create table balance_transactions (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  type text not null check (type in ('payment','overpayment','refund','fee_adjustment')),
  payment_intent_id uuid, refund_id uuid, payment_attempt_id uuid,
  gross_amount bigint, gross_currency text,
  settled_amount numeric(78,0) not null, settlement_asset_id text not null,
  fee_bps integer, fee_amount_est numeric(78,0), fee_asset_id text,
  evidence jsonb not null, created_at timestamptz not null default now());
create index bt_account_created on balance_transactions (account_id, livemode, created_at desc);
create unique index bt_once on balance_transactions (type, coalesce(payment_attempt_id, refund_id));

create table reconciliation_runs (
  id uuid primary key, account_id uuid not null, livemode boolean not null, day date not null,
  status text not null check (status in ('ok','diff','error')), diffs jsonb not null default '[]',
  created_at timestamptz not null default now(), unique (account_id, livemode, day));

-- transitions: checkout_session_transitions, payment_intent_transitions, payment_attempt_transitions, refund_transitions (foundations §7)
```

Client secrets: `cs_…_secret_<32 base62>`; stored as SHA-256; compared constant-time.

## 5. State machines (guards and side effects)

**PaymentAttempt** mirrors the execution state from rails (`domain-model.md` §4.3). Payments never infers provider state; it only applies `rails.execution.*` events whose `aggregate.version > execution_version`.

**PaymentIntent**

| From | Trigger | Guard | To | Side effects (same tx) |
|---|---|---|---|---|
| `requires_payment` | attempt → `deposit_detected`/`settling`/`incomplete_deposit` | — | `processing` | event `processing` |
| `processing` | attempt → `refunded`/`failed`/`expired_unpaid` | no other attempt in `deposit_detected…verifying` | `requires_payment` (session `open`) or `expired` (session `expiring`) | event `payment_failed` (reason from attempt) |
| `requires_payment`/`processing` | attempt → `succeeded` with `settled ≥ expected × (1 − tolerance)` | — | `succeeded` | `amount_settled`, `succeeded_attempt_id`, `late` if session not `open`; session → `complete`; statement `payment`; events `succeeded`, `checkout.session.completed` |
| `requires_payment`/`processing` | attempt → `succeeded` with short amount | — | `requires_review` (`review_reason='short_settlement'`) | statement `payment` (actual amount); event `requires_review` |
| any non-terminal | attempt → `held` | — | `requires_review` (`review_reason='verification_held'`) | event `requires_review` |
| `requires_review` | merchant `accept` | — | `succeeded` | session `complete`; events |
| `requires_review` | merchant `reject` | — | `succeeded` stays false → `requires_payment` if session open else `expired` | refund recommended (Stage 2: auto-create draft refund) |
| `succeeded`/`canceled`/`expired` | another attempt → `succeeded` | — | unchanged | attempt `is_overpayment`; statement `overpayment`; event `overpaid` (reason `duplicate`, `after_cancel`) |
| `requires_payment` | merchant `cancel` | no attempt in `deposit_detected…verifying` | `canceled` | hide attempts; session → `expired`; event `canceled` |
| `requires_payment` | session expiry finalisation | all attempts terminal | `expired` | event (internal) |

**CheckoutSession:** `open → expiring` at `expires_at` (job); `expiring → expired` when every attempt is terminal and the intent is not succeeded; `open|expiring → complete` with the intent. Attempt creation requires `open`.

**Refund:** per `domain-model.md` §4.4; `refundable = amount_settled − amount_refunded − amount_refund_pending` under `select … for update` on the intent; `amount_refund_pending` increments on create, moves to `amount_refunded` on success, decrements on fail/cancel.

## 6. API

### 6.1 Merchant API (secret / restricted keys)

| Method & path | Scope | Request → Response | Errors |
|---|---|---|---|
| `POST /v1/checkout/sessions` | `checkout_sessions.write` | `{amount, currency, asset_id?, settlement_destination?, success_url, cancel_url?, expires_in? (300–86400, default 3600), ui_mode?, customer?, customer_email?, description?, allowed_origins?, late_payments?, metadata?}` → session `{id, object, url, client_secret, status, expires_at, payment_intent, amount, currency, livemode}` | `account_not_live`, `destination_not_active`, `amount_below_destination_minimum`, `amount_above_unverified_cap`, `currency_unsupported` |
| `GET /v1/checkout/sessions/{id}` · list | `.read` | session (`expand[]=payment_intent`) | |
| `POST /v1/checkout/sessions/{id}/expire` | `.write` | session | `state_conflict` |
| `POST /v1/payment_intents` | `payment_intents.write` | same pricing fields, no UI → intent (Stage 2; for custom UIs using `@nearpay/js` attempts) | |
| `GET /v1/payment_intents/{id}` · list · `/search` | `.read` | intent with `attempts[]`, `overpayments[]` | |
| `POST /v1/payment_intents/{id}/cancel` | `.write` | intent | `state_conflict`, `attempt_in_flight` |
| `POST /v1/payment_intents/{id}/review` | `.write` | `{decision: accept\|reject}` | `state_conflict` |
| `POST /v1/refunds` | `refunds.write` | `{payment_intent, amount?, reason?, fee_mode?}` → refund with `next_action` | `amount_exceeds_refundable`, `intent_not_succeeded` |
| `GET /v1/refunds/{id}` · list | `refunds.read` | | |
| `POST /v1/refunds/{id}/prepare` | `refunds.write` | re-prepare an expired leg → refund | `state_conflict` |
| `POST /v1/refunds/{id}/submit_tx` | `refunds.write` | `{tx_hash}` or `{signed_intent}` → refund | `leg_expired` |
| `POST /v1/refunds/{id}/cancel` | `refunds.write` | only `requires_action` | |
| `POST/GET /v1/payment_links…` | `payment_links.*` | CRUD + `active` toggle | |
| `POST/GET/DELETE /v1/customers…` | `customers.*` | CRUD | |
| `GET /v1/balance_transactions` | `balance_transactions.read` | statement lines | |

All `POST` honour `Idempotency-Key` (foundations §7).

### 6.2 Payer API (anonymous route + `X-NearPay-Client-Secret`)

| Method & path | Request → Response | Errors |
|---|---|---|
| `GET /v1/checkout/sessions/{id}/public` | → `{id, status, amount, currency, display:{merchant_name_unverified, description}, expires_at, settlement:{asset_id, chain, type}, attempt?}` | `client_secret_invalid` |
| `GET /v1/checkout/sessions/{id}/public/payment_options?chain=&wallet=` | → `{options:[{origin_asset_id, chain, symbol, estimated_amount_in, route_class, min_deadline_s, available}]}` (indicative, from rails) | |
| `POST /v1/checkout/sessions/{id}/public/attempts` | `{origin_asset_id, refund_to?, payer_wallet?, method: route_quote\|intents_transfer, turnstile_token}` — `route_quote` requires `refund_to` (wallet path: the connected address); `intents_transfer` requires `payer_wallet` (the payer's Intents account; refunds go there) → attempt `{id, status, method, deposit?:{address, memo, amount_in, min_amount_in, deadline, chain}, intents_requirement?:{pay_to, token, amount, fee_to, fee_amount, binding, deadline}, attestation_ticket}` | `session_not_open`, `attempt_in_flight`, `too_many_attempts`, `route_too_slow`, `refund_address_invalid`, `screening_blocked`, `geo_blocked`, `rail_unavailable`, `turnstile_failed` |
| `POST …/attempts/{pa}/deposit_tx` | `{tx_hash}` → `202` | |
| `POST …/attempts/{pa}/sent` | → `202` (deposit signal) | |
| `POST …/attempts/{pa}/signed_intent` | `{signed: MultiPayload}` → `202` (relayed by agentpay) | `intent_invalid`, `insufficient_balance` |
| `GET …/public/stream` | SSE (`text/event-stream`, client secret in header — the SPA uses a fetch-based SSE client because `EventSource` cannot send headers): `event: session` / `attempt` / `intent` snapshots; heartbeat 15 s; `retry: 3000` | |
| `POST …/attempts/{pa}/simulate_deposit` | **test mode only** → sandbox `POST /sim/deposits` → `202` | `livemode_forbidden` |
| `POST /v1/test_helpers/payment_attempts/{pa}/advance` (secret test key) | `{to: deposit\|processing\|success\|refund\|fail}` → sandbox `POST /sim/advance` → `202` | `livemode_forbidden` |
| `GET /v1/payment_links/{id}/public` · `POST …/public/sessions` | → link display / new session `{url}` | `link_inactive` |

The payer API never returns the merchant's raw destination address; it returns the asset/chain only. The co-signed merchant name and destination fingerprint come from sentinel (ADR-0010).

### 6.3 Internal

| Method & path | Caller (scope) | Purpose |
|---|---|---|
| `POST /internal/checkout_sessions` | billing (`payments.sessions.create`) | hosted invoice sessions (`source='invoice'`, `invoice_id`, `expires_at` = invoice void date or +30 d) |
| `GET /internal/payment_intents/{id}` | billing, agentpay, webhooks (`payments.read`) | full object |
| `POST /internal/checkout_sessions/{id}/agent_attempts` | agentpay (`payments.attempts.create`) | `{payer_wallet}` → creates (or reuses the caller's still-valid, non-expired) `intents_transfer` attempt and returns `{attempt_id, execution_id, account_id, livemode, intents_requirement}`; same guards as §8.2 (limits apply per session) |
| `GET /internal/payment_intents/{id}/receipt-context` | notify (`payments.receipts.read`) | `{customer_email, merchant_display_name, amount, currency, settled, asset, evidence_links}` (PII; never in events) |
| `GET /internal/customers/{id}` | billing (`payments.read`) | customer incl. decrypted email for invoicing |
| `GET /internal/reconciliation/{account}/{day}` | ops console | run result |

## 7. Events

| Produced | Payload object |
|---|---|
| `evt.payments.checkout_session.created` · `.completed` · `.expired` | session |
| `evt.payments.payment_intent.created` · `.processing` · `.succeeded` · `.payment_failed` · `.requires_review` · `.overpaid` · `.canceled` · `.expired` | intent (+ attempts) |
| `evt.payments.payment_attempt.updated` | attempt (internal; consumed by compliance for post-screen, by sentinel for audit sampling) |
| `evt.payments.refund.created` · `.requires_action` · `.pending` · `.succeeded` · `.failed` · `.canceled` | refund |
| `evt.payments.balance_transaction.created` | statement line |
| `evt.payments.payment_link.*`, `evt.payments.customer.*` | object |

| Consumed | Handler |
|---|---|
| `evt.rails.execution.updated` / `.settled` / `.refunded` / `.failed` / `.held` | `applyExecutionUpdate` (version-checked) |
| `evt.identity.account.updated`, `evt.identity.settlement_destination.*` | refresh `settlement_contexts` |
| `evt.compliance.review.opened` (subject = payment) | set intent `metadata.review` flag + event for dashboard (no state change; payment stands) |

## 8. Algorithms

### 8.1 Create checkout session
```
ctx = settlementContext(acct, lm)          // read model; if missing or older than 5 min → sync GET identity (1 s), upsert
if lm and not ctx.account.live_enabled → 403 account_not_live
dest = param ?? ctx.defaults[pricing_asset] ?? ctx.defaults['*'];  dest.status == active else 409 destination_not_active
min = rails.destinationMinimum(dest) (cached 1 h) → convert to pricing currency with indicative price; amount < min → 422
if dest.verification == 'unverified' and usd(amount) > cap → 422 amount_above_unverified_cap
if ui_mode == 'embedded' and allowed_origins empty → 400 allowed_origins_required (https origins; http://localhost:* only in test mode)
tx: insert intent(requires_payment, snapshots: destination, fee(schedule+bearer), settings) + session(open, secret hash)
    + transitions + outbox(created ×2)
return session + url = PAY_ORIGIN + '/c/' + cs_id + '#' + client_secret   (fragment never reaches servers)
```

### 8.2 Create attempt (payer)
```
verify client secret; verify Turnstile (skip if Turnstile outage flag → stricter IP limit)
geo: country = CF-IPCountry (+ cf-region-code) forwarded by the gateway; blocked per cached compliance geo policy → 403 geo_blocked (local: no header → allowed)
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
  on 4xx → attempt failed(failure_code) + return error;  on timeout → leave quoting (recovery job) + 503 rail_unavailable
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
tx: intent FOR UPDATE; status == succeeded (or requires_review accepted); refundable >= amount
    recipient = succeeded attempt refund_to / origin asset+chain (or payer Intents account for intents_transfer)
    insert refund(requires_action), amount_refund_pending += amount
leg = rails.createExecution(idem='refund:'+id+':leg:'+n, {kind:'refund', origin: destination_snapshot (merchant asset/chain),
        refund_to: merchant destination address, destination:{asset: origin asset, chain, recipient},
        swap: fee_mode==payer_absorbs ? EXACT_INPUT(amount) : EXACT_OUTPUT(equivalent), fee: none})
next_action = {type: 'sign_and_send', chain, to: deposit.address, asset_id, amount: amount_in, memo, deadline}
            | {type: 'sign_intent', payload: unsigned Verifier transfer (merchant Intents → payer Intents)}  // intents_transfer refunds
insert refund_leg; event refund.requires_action
submit_tx → rails.submitDepositTx(leg.execution, tx) ;  signed_intent → agentpay.relay(signed)
execution settled → refund succeeded (amount_refunded += amount, pending −= amount, statement 'refund'); failed → refund failed (pending −= amount)
```

## 9. Jobs

| Kind | Trigger | Behaviour |
|---|---|---|
| `session.expire` | at `expires_at` (dedupe `cs:<id>:expire`) | `open → expiring`; hide visible attempts; if no non-terminal attempt → `expired` |
| `attempt.quoting_recovery` | every 15 s scan `status='quoting' and created_at < now()-20 s` | re-call rails with same idempotency key; after 2 min → `failed(rail_timeout)` |
| `context.refresh` | every 5 min for active accounts | sync settlement contexts (backstop to events) |
| `reconcile.daily` | 02:00 UTC per account with activity | compare statement vs rails `GET /internal/evidence?account&day`; write run; diff → ops case (RB-05) |
| `purge.*` | hourly | idempotency, inbox, published outbox |

## 10. Integrations

| Target | Timeout | Retries | Breaker | Failure |
|---|---|---|---|---|
| rails `POST /internal/executions` | 8 s | 0 inline (recovery job retries) | per account? no — global | `503 rail_unavailable` |
| rails indicative options | 2 s | 1 | yes | options marked `available:false` |
| identity settlement context | 1 s | 1 | yes | use cached context ≤ 1 h old; else `503` |
| agentpay `POST /internal/relay` | 5 s | 0 (idempotent on payload hash; client may resubmit) | yes | `503` |
| Turnstile siteverify | 2 s | 1 | yes | outage flag → stricter per-IP limit (F-12) |

## 11. Configuration

`NP_PAY_ORIGIN` (`https://pay.nearpay.localhost`), `NP_TICKET_SIGNING_KEY_FILE` (Ed25519; public key configured in sentinel), `NP_SANDBOX_BASE_URL`, `NP_COMPLIANCE_URL`, `NP_MAX_LIVE_ATTEMPTS` (5), `NP_SESSION_TTL_DEFAULT` (3600), `NP_TURNSTILE_SECRET_FILE`, `NP_UNDERPAY_TOLERANCE_MAX_BPS` (100), `NP_PII_KEK_FILE`, `NP_RAILS_URL`, `NP_AGENTPAY_URL`.

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
| 1 | sessions (hosted + embedded), intents, attempts (`route_quote`, `intents_transfer`), application of rails events incl. overpayment/late/review(read-only), statement lines, SSE, test mode, agent requirements/attempts |
| 2 | refunds (+ legs, dashboard signing), payment links, customers, `POST /v1/payment_intents`, review accept/reject, reconciliation |
| 3 | internal invoice sessions for billing |
