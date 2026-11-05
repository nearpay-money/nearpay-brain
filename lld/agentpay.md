# LLD — `nearpay-agentpay`

> Version 1.0 · HLD §11 · ADR-0009 · Flows §3, §7 · x402 v2 wire formats `reference/landscape.md` §4 · Foundations `00-foundations.md`

## 1. Scope

**Owns:** x402 v2 facilitator (`/x402/supported`, `/x402/requirements`, `/x402/verify`, `/x402/settle`), issued requirements, agent payments (`ap_`), the batching relayer for payer-signed Verifier payloads (x402, checkout `intents_transfer` attempts, Intents-account refunds), relayer key pool.
**Does not own:** checkout sessions/attempts (payments), executions/evidence for checkout (rails), destinations/fees (identity).

## 2. Runtime

| Compose service | Command | Replicas | Notes |
|---|---|---|---|
| `agentpay-api` | `api` | 2 | public x402 + `/v1/agent_payments`, internal `/internal/relay` |
| `agentpay-batcher` | `batcher` | 1 (leader) | builds, simulates, submits batches; reads outcomes; outbox relay |

## 3. NearPay x402 scheme profile (`exact` + `intents-transfer`)

**PaymentRequirements** (an `accepts[]` entry, issued by `/x402/requirements`):
```json
{"scheme":"exact","network":"near:mainnet","amount":"10000","asset":"nep141:17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1",
 "payTo":"merchant.near","maxTimeoutSeconds":120,
 "extra":{"assetTransferMethod":"intents-transfer","paymentFlow":"authorization",
          "fee":{"to":"treasury.nearpay.near","amount":"25"},"binding":"np1:…","verifyingContract":"intents.near"}}
```
**PaymentPayload.payload:** `{"signed": <MultiPayload>}` (foundations §14) whose message contains exactly two intents, in order: `transfer(receiver_id=payTo, tokens={asset: amount}, memo=binding)` and `transfer(receiver_id=fee.to, tokens={asset: fee.amount}, memo=binding)`; `deadline ≤ issued_at + maxTimeoutSeconds`; `verifying_contract = intents.near`.
**`binding`** = `np1:` + base58(sha256(JCS({account_id, resource_url, payTo, asset, amount, fee, nonce16, expires_at}))); stored server-side, single use.
**Settlement response:** `{"success":true,"transaction":"<NEAR tx hash>","network":"near:mainnet","payer":"<signer_id>","amount":"10000"}`.

`exact` + `near-intents` (cross-chain upfront, PR #3370 shape) is Stage 3: `/x402/verify` with `payload.type='quote'` creates a rails execution (owner `agentpay`), `/x402/settle` with `payload.type='deposit'` + `txHash` waits on rails evidence.

## 4. Data

```sql
create table issued_requirements (
  binding text primary key, account_id uuid not null, livemode boolean not null, resource_url text not null,
  pay_to text not null, asset_id text not null, amount numeric(78,0) not null, fee_to text not null, fee_amount numeric(78,0) not null,
  max_timeout_s integer not null, expires_at timestamptz not null, consumed_at timestamptz, created_at timestamptz not null default now());
create index ir_expiry on issued_requirements (expires_at) where consumed_at is null;

create table agent_payments (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  kind text not null check (kind in ('x402','checkout','refund')),
  binding text not null, payer text not null, payload_hash bytea not null unique, payload jsonb not null,
  intent_hashes text[], nonces text[],
  status text not null check (status in ('verified','queued','submitted','settled','failed')),
  amount numeric(78,0) not null, asset_id text not null, fee_amount numeric(78,0) not null,
  batch_id uuid, tx_hash text, block_height bigint, error_reason text,
  execution_id text,                       -- rails ex_ for checkout/refund kinds
  created_at timestamptz not null default now(), settled_at timestamptz, version integer not null default 0);
create index ap_queue on agent_payments (created_at) where status = 'queued';
create index ap_account on agent_payments (account_id, livemode, created_at desc);

create table relay_batches (
  id uuid primary key, relayer_key text not null, size smallint not null, gas_attached bigint not null,
  tx_hash text, status text not null check (status in ('building','submitted','final_ok','final_failed','split')),
  gas_burnt bigint, created_at timestamptz not null default now(), finalized_at timestamptz);

create table relayer_keys (
  public_key text primary key, account_id text not null, status text not null check (status in ('active','draining','retired')),
  in_flight boolean not null default false, last_nonce bigint, allowance_yocto numeric(40,0), last_used_at timestamptz);

create table quotas (account_id uuid primary key, rps integer not null default 50, daily_count_limit integer);
```

## 5. API

| Method & path | Auth | Request → Response | Errors (`invalidReason` / `errorReason`) |
|---|---|---|---|
| `GET /x402/supported` | anonymous | `{kinds:[{x402Version:2,scheme:'exact',network:'near:mainnet',extra:{assetTransferMethod:'intents-transfer'}}], extensions:[], signers:{'near:*':[relayer account]}}` | |
| `POST /x402/requirements` | secret/restricted key `agentpay.facilitate` | `{resource:{url, description?, mimeType?}, amount, asset_id?, pay_to?, max_timeout_s? (30–300, default 120)}` → `{x402Version:2, accepts:[PaymentRequirements], resource}` (ready to base64 into `PAYMENT-REQUIRED`) | `destination_not_intents_account`, `asset_unsupported`, `amount_below_minimum` |
| `POST /x402/verify` | same | `{x402Version, paymentPayload, paymentRequirements}` → `{isValid, payer?, invalidReason?, invalidMessage?}` | `invalid_x402_version`, `invalid_scheme`, `invalid_network`, `invalid_payment_requirements`, `requirement_unknown`, `requirement_expired`, `requirement_consumed`, `invalid_payload`, `invalid_signature`, `recipient_mismatch`, `amount_mismatch`, `deadline_invalid`, `insufficient_funds`, `nonce_used`, `payer_blocked`, `quota_exceeded` |
| `POST /x402/settle` | same | same body → `{success, transaction, network, payer, amount?, errorReason?}`; idempotent on payload hash | `settlement_pending` (non-terminal), any verify reason, `unexpected_settle_error` |
| `GET /v1/agent_payments` · `/{id}` | `agentpay.facilitate` or session | list/get | |
| `POST /v1/checkout/sessions/{id}/agent_pay` | anonymous + `X-NearPay-Client-Secret` | `{signed}` → `{status:'submitted', attempt}`; agent obtains requirements from `GET …/agent_pay/requirements` | as verify + `session_not_open` |
| `GET /v1/checkout/sessions/{id}/agent_pay/requirements` | anonymous + client secret | PaymentRequirements-shaped object (pay_to, asset, amount, fee, binding, deadline) from payments `GET /internal/checkout_sessions/{id}/agent_requirements` | |
| `POST /internal/relay` | payments (`agentpay.relay`) | `{execution_id, binding, signed}` → `202 {agent_payment_id}` | `intent_invalid`, `insufficient_balance` |

## 6. Verify algorithm
```
1. x402Version == 2; accepted == paymentRequirements (deep-equal, JCS); scheme 'exact'; network 'near:mainnet'
2. req = issued_requirements[binding] (or rails execution for checkout/refund kinds): exists, account == caller account,
   livemode == caller, not expired, not consumed
3. mp = paymentPayload.payload.signed; standard ∈ {nep413, erc191, raw_ed25519}; verify signature locally (@nearpay-internal/intents)
   msg = parse message: verifying_contract == intents.near; deadline in (now+5 s, req.expires_at]; signer_id = payer
   intents == [transfer(payTo, {asset: amount}, memo=binding), transfer(fee_to, {asset: fee_amount}, memo=binding)] exactly
   named NEAR payer with nep413 → public_key must be registered for payer in intents.near (simulate covers; fails → invalid_signature)
4. sim = simulate_intents({signed:[mp]}) (3 s) → error text maps: insufficient balance → insufficient_funds; nonce → nonce_used; else invalid_payload
5. compliance.screen({type:'near_account', address: payer}) cached 24 h → block → payer_blocked
6. quota: Valkey token bucket per account and per payer → quota_exceeded
7. cache verify result by payload_hash for 60 s; return {isValid:true, payer}
```

## 7. Settle and batcher
```
settle: verify (cached ok) → insert agent_payments(queued) on conflict(payload_hash) → return existing outcome if final
        mark requirement consumed (same tx) → wait on Valkey channel 'ap:<id>' up to 8 s
        final → {success:true, transaction}; not final → {success:false, errorReason:'settlement_pending', transaction?: tx if submitted}

batcher (leader, tick 150 ms):
  batch = select queued order by created_at limit MAX_BATCH (=floor(240 / NP_GAS_PER_PAYLOAD_TGAS)) for update skip locked
  key = free relayer key (in_flight=false); none free → wait next tick
  sim = simulate_intents({signed: batch}) → failure → re-simulate each, mark failing ones failed(reason), rebuild batch
  tx = FunctionCall(intents.near, 'execute_intents', {signed: batch}, gas = n × GAS_PER_PAYLOAD + 20 Tgas, deposit 0)
       signed by key (function-call access key, receiver intents.near, method execute_intents)
  send_tx(wait_until = FINAL) (10 s) → outcome:
     success → for each payload: match intents_executed (account_id, nonce) → settled(tx_hash, block); emit events; publish 'ap:<id>'
     failure → mark batch 'split' → requeue payloads individually (size-1 batches) → failing → failed(reason)
  timeout → keep 'submitted'; poll tx status every 2 s up to 60 s (EXPERIMENTAL_tx_status final)
```
Checkout/refund kinds do not emit evidence themselves: rails' NEAR indexer matches the `binding` memo and settles the execution (single evidence path); agentpay marks its record `settled` from the tx outcome for idempotency only.

## 8. Events

| Produced | Consumers |
|---|---|
| `evt.agentpay.agent_payment.settled` · `.failed` (kind x402) | webhooks (`agent_payment.settled/failed`), notify (none), compliance (post-screen payer) |

Consumed: `evt.sentinel.kill_switch.engaged` → stop accepting new x402 settles in live mode (`503`) until released.

## 9. Relayer accounts and keys

- Owner-provided account (e.g. `relayer.nearpay.near`); N function-call keys (Stage 1: N = 4) restricted to `intents.near` / `execute_intents`, allowance 5 NEAR each; private keys as files `relayer_key_<n>`; adding keys is a human-signed transaction (script prepares, owner signs).
- Allowance monitor: below 20% → alert (RB-09); rotation = add new key, mark old `draining`, delete after in-flight drains.

## 10. Integrations

NEAR RPC pool (simulate 3 s, send_tx 10 s, tx_status 3 s), compliance screening (1 s, cached), payments internal (requirements 1 s, agent_attempts 2 s), rails `GET /internal/executions/{id}` (1 s, for relay validation), Valkey (quotas, waiters).

## 11. Configuration

`NP_NEAR_RPC_URLS`, `NP_INTENTS_CONTRACT` (`intents.near`), `NP_TREASURY_ACCOUNT`, `NP_RELAYER_ACCOUNT`, `NP_RELAYER_KEY_FILES` (glob), `NP_GAS_PER_PAYLOAD_TGAS` (15; SP-07), `NP_SETTLE_WAIT_MS` (8000), `NP_REQUIREMENT_TTL_MAX_S` (300), `NP_SANDBOX_BASE_URL` (test mode submits to sandbox `/sim/intents/*`).

## 12. Security

- Relayer keys can only call `execute_intents` with zero deposit: they cannot move funds (INV-1); a payload is executed exactly as signed.
- Binding prevents reuse across resources/merchants (T-07); requirements single-use.
- Minimum absolute fee from the fee schedule (`intents_transfer` rule) covers gas (F-15).

## 13. Observability

`…_verify_total{result}`, `…_settle_latency_seconds`, `…_batch_size`, `…_batch_failures_total`, `…_relayer_allowance_near{key}`, `…_queue_depth`. Alerts: settle p95 > 3 s; allowance < 20%; batch failure rate > 5%.

## 14. Tests

Signature vectors (NEP-413 named + implicit, ERC-191, raw ed25519); every invalidReason; batch split on one bad payload; idempotent settle; settlement_pending path; sandbox E2E with `@nearpay/x402` middleware + agent client.

## 15. Stage mapping

| Stage | Scope |
|---|---|
| 1 | x402 `intents-transfer` (supported/requirements/verify/settle), batcher, relayer pool, checkout `agent_pay`, `/internal/relay` for checkout `intents_transfer` |
| 2 | relay for Intents-account refunds; agent payments in dashboard |
| 3 | `near-intents` cross-chain method; MPP adapter evaluation |
