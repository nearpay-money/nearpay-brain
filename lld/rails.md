# LLD — `nearpay-rails`

> Version 1.1 (audited, `audits/rails.md`) · HLD §10 · ADR-0003, ADR-0006, ADR-0012 · Foundations `00-foundations.md` · Provider facts `reference/landscape.md` §2

## 1. Scope

**Owns:** token registry, chain profiles, pricing/FX, route classification and fee selection, amount tags, executions (provider runs for payments, refunds and Intents transfers), provider observations, tracking (hot poller, Explorer feed, NEAR indexer, destination watchers), Tier-A verification and evidence, destination minimums, indicative payment options.
**Does not own:** intents/sessions/refunds as merchant objects (payments), relaying signed intents (agentpay), destinations (identity; rails keeps a watch-list read model).

## 2. Runtime

| Compose service | Command | Replicas | Role |
|---|---|---|---|
| `rails-api` | `api` | 2 | internal executions/options/minimums/evidence; public `GET /v1/assets` |
| `rails-tracker` | `tracker` | 2 | hot poller (shared via `SKIP LOCKED`), verifier, tag release |
| `rails-feeds` | `feeds` | 1 (leader-elected loops) | Explorer feed, NEAR indexer, EVM destination watchers, token/price refresh, outbox relay, consumers |

## 3. Code layout

```
src/domain/execution/        state machine, provider-status mapping, transition guards
src/domain/pricing/          settlement amount, FX, par policy, depeg guard, rounding
src/domain/token-registry/   token policy (enabled, stable, family), route classes
src/domain/verification/     tiers, evidence model, comparison rules
src/providers/port.ts        RouteProvider interface
src/providers/oneclick/      1Click client (X-API-Key), request/response mapping, signature check (SDK)
src/providers/sandbox/       same client, sandbox base URL + sandbox signature key
src/providers/direct-intents/ requirement builder for signed Verifier transfers (no swap)
src/trackers/hot-poller/     scheduler, token buckets, status application
src/trackers/explorer-feed/  cursor windows, change detection
src/trackers/near-indexer/   finalized block stream, DIP-4/NEP-245/NEP-141 parsing, vault events
src/trackers/destination-watchers/  EVM ERC-20 log scans, tag attribution
src/verifiers/near|evm|solana|bitcoin|sandbox/
src/app/                     createExecution, submitDepositTx, signal, applyObservation, verify, options, minimums, evidence
```

## 4. Data

```sql
create table tokens (
  asset_id text primary key, blockchain text not null, chain text not null, symbol text not null, decimals smallint not null,
  contract_address text, family text not null,          -- 'USDC','USDT','ETH','BTC',… (same-asset routing)
  is_usd_stable boolean not null default false, enabled boolean not null default true,
  price_usd numeric(38,18), price_updated_at timestamptz, secondary_price_usd numeric(38,18), secondary_updated_at timestamptz,
  memo_required boolean not null default false, refreshed_at timestamptz not null);

create table watched_destinations (                       -- read model from identity events
  destination_id uuid primary key, account_id uuid not null, livemode boolean not null,
  chain text not null, address_norm text not null, asset_id text not null, type text not null, active boolean not null,
  updated_at timestamptz not null);
create index wd_chain on watched_destinations (chain, address_norm) where active;

create table executions (
  id uuid primary key, account_id uuid not null, livemode boolean not null,
  owner text not null default 'payments' check (owner in ('payments','agentpay')),   -- which service consumes this execution's events
  kind text not null check (kind in ('payment','refund')), method text not null check (method in ('route_quote','intents_transfer')),
  provider text not null check (provider in ('oneclick','sandbox','direct_intents')),
  idempotency_key text not null unique,
  state text not null,                                   -- domain-model §4.3 states
  provider_status text, provider_updated_at timestamptz,
  origin_asset_id text not null, origin_chain text not null, refund_to text not null, refund_type text not null,
  destination_asset_id text not null, recipient text not null, recipient_type text not null check (recipient_type in ('destination_chain','intents')),
  destination_chain text not null, destination_key text not null,   -- chain|address_norm|asset_id
  route_class text not null check (route_class in ('same_asset','stable_pair','volatile','intents_transfer')),
  applied_fee_bps integer not null, fee_recipient text,
  amount_out_base numeric(78,0) not null, amount_tag numeric(78,0) not null default 0,
  amount_out_expected numeric(78,0) not null,            -- base + tag
  price_snapshot jsonb,
  quote_request jsonb, quote_response jsonb, correlation_id text,
  deposit_address text, deposit_memo text, amount_in numeric(78,0), min_amount_in numeric(78,0),
  binding text,                                          -- intents_transfer memo binding
  deadline timestamptz not null, time_when_inactive timestamptz,
  deposit_tx_hashes text[] not null default '{}', has_signal boolean not null default false,
  next_poll_at timestamptz, poll_count integer not null default 0, priority smallint not null default 3,
  settled_amount numeric(78,0), refunded_amount numeric(78,0), refund_reason text,
  evidence jsonb, failure_code text, held_reason text, terminal_at timestamptz,
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create unique index ex_deposit on executions (provider, deposit_address, coalesce(deposit_memo,'')) where deposit_address is not null;
create unique index ex_binding on executions (binding) where binding is not null;
create index ex_due on executions (next_poll_at) where terminal_at is null;
create index ex_inflight_dest on executions (destination_key, amount_out_expected) where terminal_at is null;
create index ex_account_day on executions (account_id, livemode, terminal_at);

create table amount_tags (
  destination_key text not null, tag numeric(78,0) not null, execution_id uuid not null,
  allocated_at timestamptz not null default now(), released_at timestamptz);
create unique index tags_live on amount_tags (destination_key, tag) where released_at is null;

create table provider_observations (
  id uuid primary key, execution_id uuid not null, source text not null
    check (source in ('quote','submit','hot_poll','explorer','indexer','watcher','verifier','operator')),
  provider_status text, payload jsonb not null, payload_sha256 bytea not null,
  provider_updated_at timestamptz, observed_at timestamptz not null default now());
create index obs_execution on provider_observations (execution_id, observed_at);

create table verifications (
  id uuid primary key, execution_id uuid not null, tier char(1) not null, chain text not null,
  tx_hash text, block_ref text, recipient text, asset_id text, amount numeric(78,0), finality text,
  result text not null check (result in ('pass','short','mismatch','not_found','pending')),
  details jsonb not null default '{}', checked_at timestamptz not null default now());

create table cursors (name text primary key, value jsonb not null, updated_at timestamptz not null default now());
create table control_flags (name text primary key, value boolean not null, reason text, updated_at timestamptz not null default now());
-- 'quoting_enabled' (sentinel kill switch), 'explorer_enabled', 'cross_chain_enabled' (RB-01)
-- names: 'explorer', 'near-indexer', 'evm:<caip2>'

create table execution_transitions (… foundations §7 …);
```

**Chain profiles** (config file `config/chains.yaml`, versioned with the code): `chain`, `blockchain` (1Click code), `family` (evm|near|solana|bitcoin|other), `tier` (A|B), `min_deadline_s`, `default_deadline_s`, `finality` (`evm: finalized tag` · `near: final` · `solana: finalized` · `bitcoin: 1 conf < USD 1k else 2`), `verification_timeout_s`, `watcher` (bool), `memo_mode` (none|required), `explorer_tx_url`. Stage 1 enables Tier A for `near:mainnet` (incl. Intents) and EVM `eip155:8453` (Base), `eip155:42161` (Arbitrum), `eip155:1` (Ethereum); other chains Tier B.

## 5. Execution state machine

States and 1Click mapping: `domain-model.md` §4.3. Additional rules:

| Rule | Detail |
|---|---|
| Version | every applied observation that changes `state`, `provider_status` or amounts increments `version` and writes outbox `rails.execution.updated` (or the terminal subject) |
| Monotonic | observations with `provider_updated_at` older than stored are recorded but not applied |
| Success | `verifying → succeeded` only with a `verifications.result='pass'` (Tier A) or provider SUCCESS + Tier B limits respected; a watcher/indexer match on a tagged/bound execution is itself a `pass` and may move any non-terminal state directly to `succeeded` |
| Short | `result='short'` → `succeeded` with `settled_amount` < expected (payments decides review) |
| Held | verification `mismatch`/`not_found` after `verification_timeout_s`, provider `FAILED` after a deposit, or unknown provider status → `held` (+ alert) |
| Expiry | no deposit signal and `now > deadline + 10 min` with provider `PENDING_DEPOSIT` → `expired_unpaid` (stays tracked by Explorer for late refunds) |
| Tag release | at terminal state (`terminal_at`) → `amount_tags.released_at = now()` |

## 6. API

| Method & path | Caller (scope) | Request → Response | Errors |
|---|---|---|---|
| `POST /internal/executions` | payments (`rails.executions.write`) | see §8.1 → execution view `{id, version, state, method, provider, route_class, applied_fee_bps, settlement:{amount_base, tag, expected, asset_id}, price_snapshot, deposit?:{address, memo, amount_in, min_amount_in, chain, deadline}, intents_requirement?:{pay_to, token, amount, fee_to, fee_amount, binding, deadline}}` | `asset_unsupported`, `route_unavailable`, `pricing_unavailable`, `screening_blocked`, `deadline_too_short`, `quote_rejected`, `quote_integrity_failed`, `rail_unavailable` |
| `GET /internal/executions/{id}` | payments, agentpay (`rails.read`) | execution view + latest evidence | |
| `GET /internal/executions?deposit_address=&memo=` | sentinel (`rails.read`) | `{exists, execution_id, account_id, livemode}` — used by sentinel **only** to decide whether a mismatch is a genuine integrity breach (kill switch) or a spoofed request | |
| `GET /internal/assets/{asset_id}` | agentpay, payments (`rails.read`) | `{asset_id, chain, decimals, is_usd_stable, price_usd, price_ok, as_of}` | `asset_unsupported` |
| `POST /internal/executions/{id}/deposit_tx` | payments | `{tx_hash, sender?}` → `202` | `state_conflict` |
| `POST /internal/executions/{id}/signals` | payments | `{type:'payer_sent'}` → `202` | |
| `GET /internal/options?destination_asset=&destination_type=&amount_out=&chain=&wallet=` | payments | `{options:[{method: route_quote\|intents_transfer, origin_asset_id, chain, symbol, decimals, estimated_amount_in, route_class, applied_fee_bps, min_deadline_s, available, reason?}]}` — an `intents_transfer` option is listed when the destination is an Intents account (paying asset = destination asset) | |
| `GET /internal/destination-minimums?keys=` | identity, payments | `{<destination_key>: {min_amount_base, min_usd_micros, computed_at}}` | |
| `GET /internal/chain-profiles` | payments, checkout via payments | profiles (public subset) | |
| `GET /internal/evidence?account_id=&livemode=&day=` | payments (reconciliation), sentinel | settled executions with evidence | |
| `GET /v1/assets` | anonymous | enabled tokens `{asset_id, chain, symbol, decimals, family, is_usd_stable}` | |

## 7. Events

| Produced | When | Consumers |
|---|---|---|
| `evt.rails.execution.updated` | any non-terminal change | payments |
| `evt.rails.execution.settled` | → `succeeded` (payload: evidence, settled amount, provider swap details) | payments, compliance (post-screen of senders), sentinel (sampling) |
| `evt.rails.execution.refunded` · `.failed` · `.held` | terminal / held | payments, compliance (held → case) |
| `evt.rails.vault.event` | indexer parsed a `nearpay_vault` NEP-297 event | billing |

| Consumed | Handler |
|---|---|
| `evt.identity.settlement_destination.activated` · `.disabled` | upsert `watched_destinations` |
| `evt.sentinel.kill_switch.engaged` · `.released` | set `control_flags.quoting_enabled` |

## 8. Algorithms

### 8.1 Create execution (`route_quote`)
```
req: {idempotency_key, kind, method, account_id, livemode, origin:{asset_id, chain, refund_to, refund_type},
      destination:{asset_id, type, chain, address}, pricing:{amount, currency | asset_id+amount},
      fee:{schedule, bearer}, deadline_cap?, refund_swap?:{type: EXACT_INPUT|EXACT_OUTPUT, amount}}
0. livemode and not control_flags.quoting_enabled → rail_unavailable(kill_switch)   // sentinel kill switch (consumed event)
1. existing = by idempotency_key → return view (idempotent)
2. o = tokens[origin], d = tokens[destination]; both enabled else asset_unsupported
3. compliance.screen({address: refund_to, chain: origin.chain, context: kind, amount_usd}) (1.5 s) → block → screening_blocked
   unavailable → policy: amount_usd ≥ 1,000 → rail_unavailable(screening) else proceed + mark post_screen_required
4. route_class = o.asset_id == d.asset_id or o.family == d.family ? 'same_asset'
               : (o.is_usd_stable and d.is_usd_stable) ? 'stable_pair' : 'volatile'
   rule = fee.schedule.rules[route_class]; bps = rule.total_bps (refund kind → 0 app fee)
5. settlement base (payments) = pricing.settle(pricing, d)   // §8.2; bearer merchant → floor(base × (10000 − bps) / 10000)
6. tag = allocateTag(destination_key, d)   // §8.3 (payments only; refunds use 0)
7. deadline = min(deadline_cap ?? +∞, now + profile(origin).default_deadline_s); deadline − now < profile.min_deadline_s → deadline_too_short
8. quoteReq = {dry:false, swapType: kind=='refund' ? refund_swap.type : 'EXACT_OUTPUT', slippageTolerance: SLIPPAGE[route_class],
     originAsset:o, depositType: refund_type=='intents' ? 'INTENTS' : 'ORIGIN_CHAIN', destinationAsset:d,
     amount: kind=='refund' ? refund_swap.amount : base+tag, refundTo, refundType, recipient: destination.address,
     recipientType: destination.type=='intents_account' ? 'INTENTS' : 'DESTINATION_CHAIN', deadline,
     referral:'nearpay', appFees: bps>0 ? [{recipient: TREASURY, fee: bps}] : [], quoteWaitingTimeMs: 3000,
     depositMode: profile(origin).memo_mode=='required' ? 'MEMO' : 'SIMPLE'}
9. res = provider.quote(quoteReq) (6 s); 400 → quote_rejected(res.message); 401/5xx/timeout → rail_unavailable (breaker)
    any failure from step 7 onwards releases the allocated tag (same transaction as the failure record)
10. integrity: res.quoteRequest ⊇ quoteReq on {recipient, destinationAsset, amount, refundTo, appFees, originAsset};
    EXACT_OUTPUT → res.quote.amountOut == amount; depositAddress valid for origin chain; quote.deadline ≥ now + min;
    verifyQuoteSignature(res) (SDK) → any failure → quote_integrity_failed + alert, tag released
11. tx: insert execution(awaiting_deposit, …, version 1) + observation(source 'quote') + transition + outbox updated
```
`SLIPPAGE`: same_asset 30 bps, stable_pair 50 bps, volatile 100 bps (config).

**`intents_transfer` method:** steps 1–6 without a provider; destination must be `intents_account` and `origin.asset_id == destination.asset_id`; `fee_amount = max(ceil(base × bps / 10000), rule.min_fee)`; `binding = 'np1:' + base58(sha256(execution_id ‖ pay_to ‖ token ‖ amount ‖ fee ‖ deadline))`; no tag (binding attributes); provider `direct_intents`; state `awaiting_deposit` until the indexer sees the transfer.

### 8.2 Pricing (`domain/pricing`)
```
settle(pricing, d):
  if pricing.currency == 'asset': if pricing.asset == d → amount; else usd = amount × price(pricing.asset); → toUnits(usd, d)
  usd = pricing.currency == 'usd' ? amount/100 : amount/10^exp × fx(currency→usd)          // exact rational math
  if d.is_usd_stable and parOk(d): return floor(usd × 10^d.decimals)                       // par policy
  return floor(usd / price(d) × 10^d.decimals)
price(t): requires |primary − secondary| / secondary ≤ 1.5% and both fresher than 60 s → else pricing_unavailable
parOk(d): |price(d) − 1| ≤ 2% (depeg guard) — if false, par disabled for d (alert)
fx(c): real-time FX ≤ 5 min old, ECB daily fallback ≤ 36 h; else pricing_unavailable
```
All arithmetic uses bigint rationals (`mulDivFloor`); snapshot `{primary, secondary, fx, par:boolean, as_of}` stored.

### 8.3 Amount tags
```
unit_usd = price(d) / 10^d.decimals;  T = min(9999, floor(0.01 / unit_usd)); if T < 100 → tag = 0 (untagged)
loop ≤ 5: t = random(1..T); insert amount_tags(destination_key, t) → unique violation → retry
all 5 collide → choose smallest free via scan; none free → tag = 0 and mark execution 'untagged' (attribution via provider hash)
```

### 8.4 Hot poller (`rails-tracker`, N replicas)
```
every 500 ms: batch = select … from executions where terminal_at is null and next_poll_at <= now()
                order by priority, next_poll_at limit 50 for update skip locked
for each (token bucket NP_ONECLICK_STATUS_RPS, priority: has tx hash 1, signalled 2, sweep 3):
  r = provider.status(deposit_address, memo) (4 s)
  observation(hot_poll); applyObservation(r)
  next_poll_at = schedule(execution): signalled/deposit_detected/settling: [2,4,8,15,15,…] s by poll_count, capped 15 s, stop after 2 h → 5 min
                 awaiting without signal: deadline + 10 min (single sweep)
                 verifying: verifier owns it (null)
```

### 8.5 Explorer feed (`rails-feeds`, leader)
Every 5 s one request (rotating three windows): (a) `statuses=PROCESSING,INCOMPLETE_DEPOSIT` newest 1,000; (b) `statuses=SUCCESS,REFUNDED,FAILED` with `startTimestamp = cursor − 30 min`; (c) newest 1,000 of all statuses since cursor. Rows are matched by `(depositAddress, depositMemo)` → if row status ≠ execution.provider_status: `has_signal = true, next_poll_at = now()`, record observation(explorer). Filter `referral=nearpay` (SP-10 confirms semantics). 429 → back off to 10 s.

### 8.6 NEAR indexer (`rails-feeds`, leader)
```
source A: neardata (FastNEAR) finalized block stream; source B: NEAR Lake; dedupe by block hash; cursor 'near-indexer'
for each receipt outcome with executor in {intents.near, v1.vault.nearpay.near, NEP-141 contracts of enabled destination tokens}:
  parse logs EVENT_JSON → dip4 (intents_executed, token_diff, transfer), nep245 mt_transfer, nep141 ft_transfer, nearpay_vault
  intents.near mt_transfer/transfer to watched intents account:
     match execution by binding (memo 'np1:…') or by (destination_key, amount == amount_out_expected) among in-flight → evidence(pass)
  nep141 ft_transfer to watched NEAR address → same matching (chain_address destinations on NEAR)
  nearpay_vault events → outbox evt.rails.vault.event
gap > 50 blocks behind final → alert; restart resumes from cursor (at-least-once; matching is idempotent)
```

### 8.7 EVM destination watchers (`rails-feeds`, one loop per enabled EVM chain)
```
from = cursor+1; to = min(from+499, finalized block)
addresses = watched_destinations(chain, active, livemode) chunked by 500
logs = eth_getLogs({fromBlock, toBlock, address: enabled token contracts on chain, topics:[TRANSFER, null, padded(addresses)]})
for each log: asset = token by contract; key = chain|to|asset; exec = in-flight execution with destination_key=key and amount_out_expected == value
   → exact match → evidence(pass, tx, block, finality 'finalized') → succeeded
   → no match → record 'unattributed_transfer' (merchant received unrelated funds; ignored)
cursor = to
```
Native-asset destinations (ETH, etc.) are not log-visible → verified via provider tx hash (§8.8).

### 8.8 Verifier
On provider `SUCCESS` without watcher/indexer evidence: for each `destinationChainTxHashes` → chain adapter:
- EVM: `eth_getTransactionReceipt` (status 1, block ≤ finalized) → ERC-20 `Transfer(to=recipient, value)` on token contract, or native `value` via `eth_getTransactionByHash` + trace-free check (`to == recipient`) — if not provable (internal transfer) → Tier B rules apply for that asset.
- NEAR: `EXPERIMENTAL_tx_status(final)` → NEP-141 `ft_transfer` event to recipient, or `intents.near` transfer to the Intents account.
- Solana / Bitcoin (Stage 4): finalized tx token-balance delta / output to address with required confirmations.
Compare `amount ≥ amount_out_expected` → `pass`; `0 < amount < expected` → `short`; otherwise `mismatch`. Not yet final → `pending`, retry every 10 s until `verification_timeout_s` → `held`.

### 8.9 Destination minimums
For `(chain, asset)` hourly: dry quote `EXACT_OUTPUT` of USD 100 equivalent from a liquid origin → `withdrawFee` (USD); `min_usd = max(1.00, 20 × withdrawFee_usd)`; Intents-account destinations: `0.01`. Cached in Valkey + `cursors`-like table; `null` if unavailable.

## 9. Jobs and loops

| Loop/job | Where | Cadence | Failure |
|---|---|---|---|
| token & price refresh | feeds (leader) | `/v0/tokens` every 30 s; secondary prices every 30 s; FX every 5 min | stale → pricing_unavailable for affected assets |
| hot poller | tracker | 500 ms tick | breaker on provider; executions keep `next_poll_at` |
| Explorer feed | feeds (leader) | 5 s | back off on 429 |
| NEAR indexer | feeds (leader) | continuous | switch source on gap/stall > 30 s |
| sandbox feed (test mode) | feeds (leader) | 1 s poll of sandbox `/sim/intents/events` and sandbox status for test executions | same matching code as the NEAR indexer |
| EVM watchers | feeds (leader per chain) | every 2 s (L2) / 12 s (L1) | RPC pool failover |
| verifier | tracker | event-driven + 10 s retries | → held after timeout |
| minimums | feeds | hourly | keep last value |
| observation archive | feeds | daily: > 90 days → Parquet to S3 (server stage), keep hash | retry |

## 10. Integrations

| Target | Timeout | Budget | Notes |
|---|---|---|---|
| 1Click `/v0/quote` | 6 s | `NP_ONECLICK_QUOTE_RPS` (5) | `X-API-Key`; breaker |
| 1Click `/v0/status` | 4 s | `NP_ONECLICK_STATUS_RPS` (10) | 404 for unknown address → `held` after 3 tries |
| 1Click `/v0/deposit/submit` | 4 s | shared with status | best effort |
| 1Click `/v0/tokens` | 5 s | 2/min | no auth |
| Explorer `/api/v0/transactions` | 5 s | 1 per 5 s | JWT/API key per Partner Dashboard |
| NEAR RPC pool (≥ 3 providers) | 3 s | — | hedged reads |
| neardata / NEAR Lake | 10 s | — | two sources |
| EVM RPC pool (≥ 2 per chain) | 3 s | — | agreement required ≥ USD 10k |
| Secondary price API, FX API | 3 s | — | cached |
| compliance screening | 1.5 s | — | policy above |

## 11. Configuration

`NP_ONECLICK_BASE_URL`, `NP_ONECLICK_API_KEY_FILE` (live only; absent in Stage 1 local → unauthenticated quotes with +fee, appFees still sent), `NP_SANDBOX_BASE_URL`, `NP_SANDBOX_SIGNING_PUBKEY`, `NP_EXPLORER_TOKEN_FILE`, `NP_TREASURY_ACCOUNT` (e.g. `treasury.nearpay.near`, provided by the owner), `NP_NEAR_RPC_URLS`, `NP_NEARDATA_URL`, `NP_EVM_RPC_URLS_<CHAINID>`, `NP_PRICE_SECONDARY_URL`, `NP_PRICE_SECONDARY_KEY_FILE`, `NP_FX_URL`, `NP_SLIPPAGE_BPS` (json), `NP_CHAINS_FILE`, `NP_ONECLICK_QUOTE_RPS`, `NP_ONECLICK_STATUS_RPS`, `NP_EXPLORER_ENABLED` (false until the owner provides an Explorer token), `NP_PRICING_CURRENCIES` (Stage 1: `usd`), `NP_TIER_B_LIMITS` (`{per_payment_usd: 2000, per_day_usd: 20000}`).

**Stage 1 pricing posture:** without a secondary price source configured, non-par settlement assets are disabled (`pricing_unavailable`) — Stage 1 demos settle in USD stablecoins; fiat pricing currency is `usd` only (FX lands in Stage 2).

## 12. Security

- Livemode routing assertion in three places (INV-8): provider selection, `oneclick` adapter rejects `livemode=false`, `sandbox` adapter rejects `livemode=true`.
- Recipient/amount integrity check + signature verification on every live quote (T-03 layer 1).
- Rails never signs anything and holds no keys except API credentials.

## 13. Observability

`…_quote_duration_seconds{provider,outcome}`, `…_status_calls_total{source,outcome}`, `…_status_budget_utilisation`, `…_executions_inflight{state}`, `…_time_to_evidence_seconds{route_class,chain}`, `…_verification_results_total{tier,result}`, `…_indexer_lag_blocks`, `…_watcher_lag_blocks{chain}`, `…_unattributed_transfers_total{chain}`, `…_pricing_unavailable_total{asset}`, `…_tags_exhausted_total`. Alerts: held rate > 1% / 15 min (RB-04); indexer/watcher lag > 50 blocks; status budget > 80%; quote error rate > 5% (RB-01); depeg guard tripped.

## 14. Failure modes

| Failure | Outcome |
|---|---|
| 1Click quote/status down | breaker open → `rail_unavailable`; in-flight Tier-A executions still settle via watchers/indexer |
| Quote integrity failure | execution not created; alert (possible provider/API compromise) |
| Indexer gap | resume from cursor; matching idempotent |
| Tag space exhausted | untagged fallback (provider hash attribution) |
| Price sources disagree | non-par settlement blocked; stablecoin par unaffected |

## 15. Tests

- Unit: status mapping table; pricing (par, depeg, FX fallback, rounding down); route classification; tag allocation; deadline cap; binding derivation.
- Property: for any sequence of observations (duplicates, reordering, conflicting sources) the execution never leaves a terminal state except `expired_unpaid → refunded` and `held → succeeded|refunded`; settled ⇒ evidence present; tags unique among in-flight per destination.
- Contract: 1Click adapter against recorded fixtures (`test/fixtures/oneclick`), including `KNOWN_DEPOSIT_TX`, 404, 400, signature failure.
- Integration: indexer parsing on recorded mainnet blocks (fixtures from SP-14); EVM watcher against an Anvil fork with Transfer logs; verifier tiers.
- Load: hot poller at 1,000 in-flight executions within budget.

## 16. Stage mapping

| Stage | Scope |
|---|---|
| 1 | token registry + pricing (par + primary/secondary), `oneclick` + `sandbox` + `direct_intents` providers, executions, hot poller, Explorer feed, NEAR indexer (Intents + NEP-141), EVM watchers for Base/Arbitrum/Ethereum, verifiers NEAR + EVM, minimums, options, evidence API |
| 2 | refunds (`EXACT_INPUT`/`EXACT_OUTPUT` legs), reconciliation evidence, observation archive |
| 4 | Solana + Bitcoin watchers/verifiers, vault event forwarding hardening |
