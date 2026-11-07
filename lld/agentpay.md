# LLD — `nearpay-agentpay`

> Version 1.6 (stage 2 audit finding 11: the public payer-route fallback is removed — the payer session comes only from payments' internal route, a `404` is `503 rail_unavailable` §5/§10) · Version 1.5 (step 2.2 implementation: refund relay checks the signed payload against payments' issued `next_action` **and** rails' `refund.transfer` (`signed_intent_mismatch`, CR-22-AP-01) §5; `agent_payments.refund_id`, nullable `fee_to` (migration 0003) §4; payer-route fallback rule §5; `sendSignedTx` rejection rule §7; binding-index replays §7) · Version 1.4 (step 2.1, spec 0.2.0: `/internal/relay` `kind: refund` for Intents-account refund legs §5, payments §17; kill switch blocks live refund relays §8; payer session read through payments' internal route `getPayerCheckoutSession`, CR-15-15, §5/§10) · Version 1.3 (step 1.4, spec 0.1.5: `extra.deadline` §3/§6, settle `replayed` and `result_reported_at` §4/§7, `AgentPayment.sequence` §5/§7) · Version 1.2 (step 1.2 implementation findings folded: runtime §2, data §4, checkout routes and relay §5, verify §6, settle and batcher §7, kill switch §8, relayer §9, configuration §11, ops CLI §16) · Version 1.1 audited (`audits/agentpay.md`) · HLD §11 · ADR-0009 · Flows §3, §7 · x402 v2 wire formats `reference/landscape.md` §4 · Foundations `00-foundations.md`

## 1. Scope

**Owns:** x402 v2 facilitator (`/x402/supported`, `/x402/requirements`, `/x402/verify`, `/x402/settle`), issued requirements, agent payments (`ap_`), the batching relayer for payer-signed Verifier payloads (x402, checkout `intents_transfer` attempts, Intents-account refunds), relayer key pool.
**Does not own:** checkout sessions/attempts (payments), executions/evidence for checkout (rails), destinations/fees (identity).

## 2. Runtime

| Compose service | Command | Replicas | Notes |
|---|---|---|---|
| `agentpay-api` | `api` | 2 | public x402 + `/v1/agent_payments`, internal `/internal/relay` |
| `agentpay-batcher` | `batcher` | 1 (leader) | builds, simulates, submits batches; reads outcomes; also runs the kill-switch consumer, jobs and the outbox relay. The relay loop is leader-elected (advisory lock `agentpay:batcher`) |

Processes: `node dist/main.js api|batcher|migrate|healthcheck`.

## 3. NearPay x402 scheme profile (`exact` + `intents-transfer`)

**PaymentRequirements** (an `accepts[]` entry, issued by `/x402/requirements`):
```json
{"scheme":"exact","network":"near:mainnet","amount":"10000","asset":"nep141:17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1",
 "payTo":"merchant.near","maxTimeoutSeconds":120,
 "extra":{"assetTransferMethod":"intents-transfer","paymentFlow":"authorization",
          "fee":{"to":"treasury.nearpay.near","amount":"25"},"binding":"np1:…","verifyingContract":"intents.near",
          "deadline":"2026-11-07T12:02:00.000Z"}}
```
`extra.deadline` (spec 0.1.5, optional on the wire) is the binding's `expires_at` in RFC 3339 — the absolute expiry of the issued requirement.
**PaymentPayload.payload:** `{"signed": <MultiPayload>}` (foundations §14) whose message contains exactly two intents, in order: `transfer(receiver_id=payTo, tokens={asset: amount}, memo=binding)` and `transfer(receiver_id=fee.to, tokens={asset: fee.amount}, memo=binding)`; `deadline ≤ issued_at + maxTimeoutSeconds`; `verifying_contract = intents.near`.
**`binding`** = `np1:` + base58(sha256(JCS({account_id, resource_url, payTo, asset, amount, fee, nonce16, expires_at}))); stored server-side, single use. Requirements are issued **per 402 response** (never cached and shared between clients); issuance is rate-limited per merchant (default 200/s) and expires after `maxTimeoutSeconds`.
**Settlement response:** `{"success":true,"transaction":"<NEAR tx hash>","network":"near:mainnet","payer":"<signer_id>","amount":"10000"}`, plus `"replayed":true` when an earlier settle response already carried this final outcome (§7).

`exact` + `near-intents` (cross-chain upfront, PR #3370 shape) is Stage 3: `/x402/verify` with `payload.type='quote'` creates a rails execution (owner `agentpay`), `/x402/settle` with `payload.type='deposit'` + `txHash` waits on rails evidence.

## 4. Data

```sql
create table issued_requirements (
  binding text primary key, account_id uuid not null, livemode boolean not null, resource_url text not null,
  pay_to text not null, asset_id text not null, amount numeric(78,0) not null, fee_to text not null, fee_amount numeric(78,0) not null,
  max_timeout_s integer not null, expires_at timestamptz not null, consumed_at timestamptz, created_at timestamptz not null default now(),
  resource jsonb not null,                -- {url, description?, mimeType?} echoed in PAYMENT-REQUIRED
  nonce16 text not null, consumed_by uuid); -- consumed_by: the agent payment that consumed it
create index ir_expiry on issued_requirements (expires_at) where consumed_at is null;

create table checkout_requirements (     -- what `agent_pay/requirements` handed out, so `agent_pay {signed}` can be matched
  binding text primary key, account_id uuid not null, livemode boolean not null, checkout_session_id uuid not null,
  attempt_id uuid not null, execution_id text not null, payer text not null,
  requirement jsonb not null,             -- IntentsRequirement as payments returned it
  attestation_ticket text not null, created_at timestamptz not null default now());
create index cr_session on checkout_requirements (checkout_session_id);

create table agent_payments (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  kind text not null check (kind in ('x402','checkout','refund')),
  refund_id uuid,                          -- migration 0003: payments refund (re_) of a refund leg
  binding text not null, payer text not null,       -- refund kind: payer = merchant signer, pay_to = receiver,
                                                    -- fee_to null (0003 drops not null), fee_amount 0 payload_hash bytea not null unique, payload jsonb not null,
  intent_hashes text[], nonces text[],
  status text not null check (status in ('verified','queued','submitted','settled','failed')),
  amount numeric(78,0) not null, asset_id text not null, fee_amount numeric(78,0) not null,
  batch_id uuid, tx_hash text, block_height bigint, error_reason text,
  execution_id text,                       -- rails ex_ for checkout/refund kinds
  pay_to text, fee_to text, standard text, deadline timestamptz, resource_url text, checkout_session_id uuid,
  solo boolean not null default false,     -- requeued for a size-1 batch after a split
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), settled_at timestamptz,
  version integer not null default 0,
  result_reported_at timestamptz);          -- migration 0002: first settle response with the final outcome (§7); not on the object, no version bump
create index ap_queue on agent_payments (livemode, created_at) where status = 'queued';
create index ap_account on agent_payments (account_id, livemode, created_at desc);
create unique index ap_binding_live on agent_payments (binding) where status in ('verified','queued','submitted','settled');
-- one live payload per binding: a second signed payload for the same requirement is refused (intent_invalid)
-- while the first is live; a failed one frees the binding
create table agent_payment_transitions (… per foundations §7, append-only …);

create table relay_batches (
  id uuid primary key, livemode boolean not null, relayer_key text not null, size smallint not null, gas_attached bigint not null,
  tx_hash text, tx_nonce bigint, status text not null check (status in ('building','submitted','final_ok','final_failed','split')),
  failure text, gas_burnt bigint, created_at timestamptz not null default now(), submitted_at timestamptz, finalized_at timestamptz);

create table relayer_keys (
  public_key text primary key, account_id text not null, status text not null check (status in ('active','draining','retired')),
  in_flight boolean not null default false, last_nonce bigint, allowance_yocto numeric(40,0), allowance_checked_at timestamptz,
  last_used_at timestamptz, created_at timestamptz not null default now());

create table quotas (account_id uuid primary key, rps integer not null default 50, daily_count_limit integer);

create table kill_switch (id smallint primary key check (id = 1), engaged boolean not null, reason text not null, actor text not null,
  event_version integer not null, updated_at timestamptz not null default now());
-- single row from evt.sentinel.kill_switch.*; events with a version ≤ event_version are ignored
```
Purges: unconsumed x402 requirements expired more than 24 h; checkout requirements older than 7 days.

## 5. API

| Method & path | Auth | Request → Response | Errors (`invalidReason` / `errorReason`) |
|---|---|---|---|
| `GET /x402/supported` | anonymous | `{kinds:[{x402Version:2,scheme:'exact',network:'near:mainnet',extra:{assetTransferMethod:'intents-transfer'}}], extensions:[], signers:{'near:*':[relayer account]}}` | |
| `POST /x402/requirements` | secret/restricted key `agentpay.facilitate` | `{resource:{url, description?, mimeType?}, amount, asset_id?, pay_to?, max_timeout_s? (30–300, default 120)}` → `{x402Version:2, accepts:[PaymentRequirements], resource}` (ready to base64 into `PAYMENT-REQUIRED`) | `destination_not_intents_account`, `asset_unsupported`, `amount_below_minimum` |
| `POST /x402/verify` | same | `{x402Version, paymentPayload, paymentRequirements}` → `{isValid, payer?, invalidReason?, invalidMessage?}` | `invalid_x402_version`, `invalid_scheme`, `invalid_network`, `invalid_payment_requirements`, `requirement_unknown`, `requirement_expired`, `requirement_consumed`, `invalid_payload`, `invalid_signature`, `recipient_mismatch`, `amount_mismatch`, `deadline_invalid`, `insufficient_funds`, `nonce_used`, `payer_blocked`, `quota_exceeded` |
| `POST /x402/settle` | same | same body → `{success, transaction, network, payer, amount?, errorReason?, replayed?}`; idempotent on payload hash (`replayed`, spec 0.1.5: §7) | `settlement_pending` (non-terminal), any verify reason, `unexpected_settle_error` |
| `GET /v1/agent_payments` · `/{id}` | `agentpay.facilitate` or session | list/get; `sequence` (spec 0.1.5) = the row's `version`, the same value the events carry as `aggregate.version` | |
| `POST /v1/checkout/sessions/{id}/agent_pay` | anonymous + `X-NearPay-Client-Secret` | `{signed}` → `{status:'submitted', attempt}`; agent obtains requirements from `POST …/agent_pay/requirements`; the payload is matched to the handed-out requirement through `checkout_requirements` | as verify + `session_not_open` |
| `POST /v1/checkout/sessions/{id}/agent_pay/requirements` | anonymous + client secret | `{payer: <agent Intents account>}` → PaymentRequirements-shaped object (pay_to, asset, amount, fee, binding, deadline) + `attempt_id` + `attestation_ticket` from payments `POST /internal/checkout_sessions/{id}/agent_attempts` (creates or reuses the attempt); `@nearpay/agent-signer` verifies a sentinel co-signature before signing (same two-party release as the human checkout) | `session_not_open`, `too_many_attempts`, `destination_not_intents_account` |
| `POST /internal/relay` | payments (`agentpay.relay`) | `{execution_id, binding, signed, kind?: checkout\|refund, refund?}` → `202 {agent_payment_id}` (Stage 2 `kind: refund`: payments §17 — the execution is `kind: refund` with `refund.transfer` set; exactly that one transfer, signed by `refund.transfer.signer_id` = the merchant, deadline ≤ `refund.transfer.deadline`; recorded as `agent_payments.kind = 'refund'`) | `intent_invalid`, `insufficient_balance` |

**Client secret on the checkout routes:** agentpay cannot check payments' client secrets. It calls payments' public `GET /v1/checkout/sessions/{id}/public` with the caller's `X-NearPay-Client-Secret` (no service token); that read also gives the session status and the `PayerAttempt` returned by `agent_pay`. This is the Stage 1 path. **Stage 2 (spec 0.2.0, CR-15-15):** agentpay calls payments `GET /internal/checkout_sessions/{id}/payer` with its service token (`payments.read`) and forwards the caller's `X-NearPay-Client-Secret`; the answer `PayerSessionContext {account_id, livemode, session: PublicCheckoutSession}` also gives account and mode without trusting the public route; a wrong secret is `client_secret_invalid` as before. Payments serves the route (nearpay-payments PR #4); the public-route fallback is removed (stage 2 audit finding 11).

**`/internal/relay` `kind: refund` validation (Stage 2, step 2.2).** In order: replay by payload hash (same execution and kind → the existing `ap_`; else `intent_invalid`) → rails execution: `kind = refund`, `refund.transfer` set, `transfer.memo = binding`, `reference.refund` set and equal to the body's `refund` (if given), `state = awaiting_deposit` → live + kill switch → `503 rail_unavailable` → signature → payments `GET /internal/refunds/{id}` (`payments.read`, 1 s): same mode, `status = requires_action`, `next_action.type = sign_intent` → the issued payload must agree with rails (signer, receiver, token, amount, memo, `verifying_contract`, one transfer, NEP-413 nonce = message nonce, issued deadline ≤ `transfer.deadline`; otherwise `intent_invalid` "prepare it again" and an error log — payments and rails disagree) → the signed payload must be **exactly** the issued one: standard ∈ `standards`; signer, contract, nonce, deadline and the one transfer field by field; for NEP-413 the envelope's recipient, nonce, no `callbackUrl` and `message` byte-equal to `nep413.message`; for ERC-191 / raw ed25519 `payload` byte-equal to the issued `payload` → else **`422 signed_intent_mismatch`** (not declared on `relaySignedIntent` in 0.2.0: CR-22-AP-01) → deadline > now + 5 s (`intent_invalid`) → `simulate_intents` in the leg's mode (`insufficient_balance` / `intent_invalid`) → insert `agent_payments(kind refund, queued)` + transition (actor `payments`) in one transaction. No events, no screening (rails screens the recipient when preparing the leg). **payments must call the relay while the leg still shows its `next_action`** (before marking it `submitted`, as payments §8 orders it) and its `getInternalRefund` read must not block on the row lock it holds; the relay's peer reads (rails 1 s, payments 1 s, simulate 3 s) fit payments' 5 s timeout and payments retries a timeout (idempotent). A `refund` field without `kind: refund` is `400 invalid_request`; a checkout relay refuses an execution whose `kind` is not `payment`.

**Payer session (stage 2 audit finding 11).** agentpay reads the session only through `getPayerCheckoutSession`; there is no fallback to the public route (it carries no account or mode, so the scope checks were skipped — `scope: null`). A `401`/`client_secret_invalid` is `client_secret_invalid` (payments answers it for an unknown session too); anything else, `404 resource_missing` included, is `503 rail_unavailable` (fail closed, error log). The agent attempt's and the stored requirement's account and mode must always equal the session's.

**`/internal/relay` validation** uses rails `GET /internal/executions/{id}`: `method = intents_transfer`, `intents_requirement.binding` equal, `state = awaiting_deposit`; then the same exact-two-transfers check, signature check and `simulate_intents` as verify. Account and mode come from the execution. Checkout relays are not screened again (rails screened at execution creation).

## 6. Verify algorithm
```
1. x402Version == 2; accepted == paymentRequirements (deep-equal, JCS); scheme 'exact'; network 'near:mainnet'
2. req = issued_requirements[binding] (or rails execution for checkout/refund kinds): exists, account == caller account,
   livemode == caller, not expired, not consumed; the stored terms (payTo, asset, amount, fee, maxTimeout) equal
   paymentRequirements, and an echoed extra.deadline equals the issued expires_at (an absent one is accepted:
   0.1.4 agents strip it); if paymentPayload.resource is present its url must equal req.resource_url
   → else invalid_payment_requirements (resource is optional in the spec, so a payload without it still verifies)
3. mp = paymentPayload.payload.signed; standard ∈ {nep413, erc191, raw_ed25519}; verify signature locally (@nearpay-internal/intents)
   msg = parse message: verifying_contract == intents.near; deadline in (now+5 s, req.expires_at]; signer_id = payer
   intents == [transfer(payTo, {asset: amount}, memo=binding), transfer(fee_to, {asset: fee_amount}, memo=binding)] exactly
   named NEAR payer with nep413 → public_key must be registered for payer in intents.near (simulate covers; fails → invalid_signature)
4. sim = simulate_intents({signed:[mp]}) (3 s) → error text maps: insufficient balance → insufficient_funds; nonce → nonce_used; else invalid_payload
5. compliance.screen({type:'near_account', address: payer}) cached 24 h → block → payer_blocked; review → allowed (payment stands);
   compliance unreachable → 503 screening_unavailable (fail closed)
6. quota: one-second fixed windows (Valkey INCR + EXPIRE; in-process without Valkey) per account (quotas.rps, default 50)
   and per payer (NP_PAYER_RATE_PER_S, 10) → quota_exceeded; requirement issuance per merchant: NP_REQUIREMENT_RATE_PER_S (200)
7. cache verify result by payload_hash for 60 s; return {isValid:true, payer}
```
**Deadlines:** since spec 0.1.5 requirements carry their absolute expiry as `extra.deadline` (= `expires_at`), and `@nearpay/agent-signer` signs exactly that instant as the Verifier deadline (independent of the agent's clock). Requirements without it (facilitators older than 0.1.5) keep the fallback `deadline = now + floor(maxTimeoutSeconds / 2)`, which stays inside `(now + 5 s, issued.expires_at]` despite latency and clock skew. The payload-deadline check of step 3 is unchanged.

**Fee:** `fee_amount = max(ceil(amount × bps / 10 000), min_fee_usd_micros at the asset price (par for USD stablecoins without a price), 1 base unit)` — the floor of 1 keeps the fee transfer non-empty; `amount_below_minimum` when `amount < max(1, minimum fee units)`.

## 7. Settle and batcher
```
settle: verify (cached ok) → insert agent_payments(queued) on conflict(payload_hash) → return existing outcome if final
        mark requirement consumed (same tx) → wait on Valkey channel 'ap:<id>' up to 8 s (in-process without Valkey)
        a replayed PAYMENT-SIGNATURE gets the same response and never a second agent payment or relay (idempotent on the
        payload hash)
replayed (spec 0.1.5): the first response that carries the final outcome (settled or failed) claims it —
        update agent_payments set result_reported_at = now where id = … and result_reported_at is null (two concurrent
        calls cannot both win); every later response repeats the outcome with replayed: true, so the merchant delivers the
        resource at most once per settlement. settlement_pending is not a result: it never says replayed and never claims,
        so "pending, then call again" still delivers once
replays: a concurrent submission of the same payload may hit ap_binding_live before the payload_hash arbiter; the relay then
        re-reads by payload hash and answers the winner (same execution and kind), so identical submissions never get intent_invalid
statuses: payloads are inserted `queued` (verify and queue in one step; `verified` is unused); a claimed payload moves
        `queued → submitted` when its batch is built (before simulation and sending); `submitted → queued` on abandon/split
        final → {success:true, transaction}; not final → {success:false, errorReason:'settlement_pending', transaction?: tx if submitted}
sequence: every change bumps agent_payments.version (ledger.updatePayment), including the batcher's bulk tx_hash write
        after submission (tx_hash is on the object); later writers re-read the row FOR UPDATE

batcher (leader, tick 150 ms):
  batch = select queued order by created_at limit MAX_BATCH (=floor(240 / NP_GAS_PER_PAYLOAD_TGAS)) for update skip locked
  key = free relayer key (in_flight=false); none free → wait next tick
  sim = simulate_intents({signed: batch}) → failure → re-simulate each, mark failing ones failed(reason), rebuild batch
  tx = FunctionCall(intents.near, 'execute_intents', {signed: batch}, gas = n × GAS_PER_PAYLOAD + 20 Tgas, deposit 0)
       signed by key (function-call access key, receiver intents.near, method execute_intents)
  live: sign with kit buildExecuteTx, store the tx hash, then kit sendSignedTx({bytes, txHash}, wait FINAL, 10 s) (Stage 2,
       CR-15-16); rejected (requeue, forget the nonce) only when every RPC provider refused it as an invalid transaction;
       a timeout, transport error or any other answer keeps it 'submitted' and polls; tx status stays agentpay's own
       EXPERIMENTAL_tx_status call (unknown → not final). A key carries one tx at a time (in_flight), next nonce = last_nonce + 1
  outcome:
     success → for each payload: match intents_executed (account_id, nonce) → settled(tx_hash, block); emit events; publish 'ap:<id>'
     failure → mark batch 'split' → requeue payloads individually (size-1 batches, `solo`) → failing → failed(reason)
     InvalidTxError / InvalidNonce / Expired… → nothing executed: payloads requeued, the key's nonce forgotten
  any other error or timeout → keep 'submitted'; poll tx status every 2 s (every 30 s after 60 s), alert after 60 s,
     abandon only after 25 h (the block hash has expired, so the tx can no longer land)
```
**Test-mode limitation:** the sandbox has no tx-status route, so a sandbox execute that fails in transport (or a test batch found `submitted` at restart) is requeued, and an already-executed payload then fails with `nonce_used`. Test mode only; a sandbox tx-status route is requested (`roadmap/open-questions.md`).
Checkout/refund kinds do not emit evidence themselves: rails' NEAR indexer matches the `binding` memo and settles the execution (single evidence path); agentpay marks its record `settled` from the tx outcome for idempotency only.

## 8. Events

| Produced | Consumers |
|---|---|
| `evt.agentpay.agent_payment.settled` · `.failed` (kind x402) | webhooks (`agent_payment.settled/failed`), notify (none), compliance (post-screen payer) |

Checkout payloads emit nothing (rails' indexer is the single evidence path).

Consumed: `evt.sentinel.kill_switch.engaged` → stop accepting new **live** x402 settles **and** new live checkout relays / `agent_pay` (`503 rail_unavailable`) until released; replays of already-queued payloads and the batcher continue; test mode is unaffected. Blocking checkout relays is the conservative reading of this section (which named only x402 settles), to be revisited. **Stage 2 (lead decision at step 2.1 integration):** live refund relays (`/internal/relay` `kind: refund`) are refused the same way (`503 rail_unavailable`, fail closed) — the deposit-instruction integrity concern applies to refund route legs too; payments also stops preparing live refund legs (payments §17 item 16). Refunds resume on release.

## 9. Relayer accounts and keys

- Owner-provided account (e.g. `relayer.nearpay.near`); N function-call keys (Stage 1: N = 4) restricted to `intents.near` / `execute_intents`, allowance 5 NEAR each; private keys as files `relayer_key_<n>`; adding keys is a human-signed transaction (script prepares, owner signs).
- Allowance monitor: job `relayer.allowance` every 5 min (`view_access_key`), gauge `relayer_allowance_near{key}`, alert below 1 NEAR (20% of 5; RB-09); rotation = add new key, mark old `draining`, delete after in-flight drains. Without owner-provided key files live relaying is off.

## 10. Integrations

NEAR RPC pool (simulate 3 s, send_tx 10 s, tx_status 3 s; tried in order), compliance screening (1 s, cached), payments `GET /internal/checkout_sessions/{id}/payer` (Stage 2; client secret forwarded, service token; no public-route fallback), payments internal (`agent_attempts` 8 s — it creates a rails execution), rails `GET /internal/executions/{id}` (1 s, relay validation) and `GET /internal/assets/{id}` (1 s, decimals + price for the minimum fee), identity `GET /internal/accounts/{id}/settlement-context` (1 s, cached 60 s: default Intents destination for `payTo`, fee schedule), Valkey (quotas, waiters).

## 11. Configuration

`NP_NEAR_RPC_URLS` (empty → live payloads refused), `NP_INTENTS_CONTRACT` (`intents.near`), `NP_TREASURY_ACCOUNT` (unset → live requirements refused), `NP_TEST_TREASURY_ACCOUNT` (`treasury.nearpay.near`, sandbox only), `NP_RELAYER_ACCOUNT`, `NP_RELAYER_KEY_FILES` (batcher; `*` in the file name, e.g. `/run/secrets/relayer_key_*`), `NP_GAS_PER_PAYLOAD_TGAS` (15; SP-07), `NP_SETTLE_WAIT_MS` (8000), `NP_REQUIREMENT_TTL_MAX_S` (300), `NP_REQUIREMENT_RATE_PER_S` (200), `NP_PAYER_RATE_PER_S` (10), `NP_BATCH_TICK_MS` (150), `NP_SANDBOX_BASE_URL` (test mode submits to sandbox `/sim/intents/*`; unset → test payloads refused), `NP_IDENTITY_URL`, `NP_PAYMENTS_URL`, `NP_RAILS_URL`, `NP_COMPLIANCE_URL`, `NP_SERVICE_CLIENT_SECRET_FILE`, `NP_PII_KEK_FILE` (idempotency store), `NP_VALKEY_URL` (settle waiters and quotas across replicas), `NP_NATS_URL` (batcher).

## 12. Security

- Relayer keys can only call `execute_intents` with zero deposit: they cannot move funds (INV-1); a payload is executed exactly as signed.
- Binding prevents reuse across resources/merchants (T-07); requirements single-use.
- Minimum absolute fee from the fee schedule (`intents_transfer` rule) covers gas (F-15).

## 13. Observability

`…_verify_total{result}`, `…_settle_latency_seconds`, `…_batch_size`, `…_batch_failures_total`, `…_relayer_allowance_near{key}`, `…_queue_depth`. Alerts: settle p95 > 3 s; allowance < 20%; batch failure rate > 5%.

## 14. Tests

Stage 2: property tests — every single-field mutation of an issued refund payload, validly re-signed, is `signed_intent_mismatch`; any interleaving of replays / concurrent / non-issued submissions gives one agent payment and one relay per leg (Testcontainers). Signature vectors (NEP-413 named + implicit, ERC-191, raw ed25519); every invalidReason; batch split on one bad payload; idempotent settle; settlement_pending path; sandbox E2E with `@nearpay/x402` middleware + agent client.

## 15. Stage mapping

| Stage | Scope |
|---|---|
| 1 | x402 `intents-transfer` (supported/requirements/verify/settle), batcher, relayer pool, checkout `agent_pay`, `/internal/relay` for checkout `intents_transfer` |
| 2 | relay for Intents-account refunds; agent payments in dashboard; payer session via payments `getPayerCheckoutSession` (CR-15-15); relayer sends with kit `sendSignedTx` (CR-15-16) instead of its own JSON-RPC call — implemented in step 2.2 (the dashboard uses the Stage 1 `GET /v1/agent_payments` routes with `kind=refund`; CR-22-AP-02 asks for `refund` on `AgentPayment`) |
| 3 | `near-intents` cross-chain method; MPP adapter evaluation |

## 16. Ops CLI (Stage 1)

`docker compose exec agentpay-api /nodejs/bin/node dist/ops.js <command>`: `payment` · `queue` · `keys` · `key-status` (drain / retire / activate) · `kill-switch`.
