# LLD — `nearpay-rails`

> Version 1.2 (step 1.2 implementation findings folded: data and chain profiles §4, state rules §5, appFees split and SP-01 §8.1, tags §8.3, tracking §8.4–§8.8, minimums and options §8.9, configuration §11, ops CLI §17; SP-01 and SP-14 resolved) · Version 1.1 audited (`audits/rails.md`) · HLD §10 · ADR-0003, ADR-0006, ADR-0012 · Foundations `00-foundations.md` · Provider facts `reference/landscape.md` §2

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
src/providers/oneclick/      1Click client (JWT partner key → Authorization: Bearer, else X-API-Key), live-body normalisation (nulls dropped, nested correlationId), request/response mapping, signature check (SDK)
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
create table tokens (                                    -- one registry per mode: live = 1Click /v0/tokens, test = the sandbox's
  livemode boolean not null, asset_id text not null, blockchain text not null, chain text not null, symbol text not null, decimals smallint not null,
  contract_address text, family text not null,          -- 'USDC','USDT','ETH','BTC',… (same-asset routing)
  is_usd_stable boolean not null default false, enabled boolean not null default true,
  price_usd numeric(38,18), price_updated_at timestamptz, secondary_price_usd numeric(38,18), secondary_updated_at timestamptz,
  memo_required boolean not null default false, refreshed_at timestamptz not null, primary key (livemode, asset_id));
create index tokens_contract on tokens (livemode, chain, contract_address) where enabled;
-- 1Click /v0/tokens also lists `1cs_v1:…` ids (27 on 2026-11-06), which are not NEAR Intents asset ids: dropped at refresh
-- (the spec AssetId rejects them). Test mode never offers an asset the sandbox cannot quote and works offline.

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
  next_poll_at timestamptz, poll_count integer not null default 0, not_found_count integer not null default 0,
  priority smallint not null default 3,
  verify_at timestamptz, verifying_since timestamptz,      -- verifier schedule (verifying, and held with provider SUCCESS), own columns
  fee_amount numeric(78,0),                              -- intents_transfer fee transfer (route quotes carry it in appFees)
  settled_amount numeric(78,0), refunded_amount numeric(78,0), refund_reason text,
  provider_amount_out numeric(78,0),                     -- the provider's claim; evidence decides settled_amount
  destination_tx_hashes text[] not null default '{}',
  evidence jsonb, failure_code text,
  held_reason text,                                      -- evidence_mismatch | evidence_timeout | failed_after_deposit | unknown_status
  terminal_at timestamptz,
  tracking_until timestamptz,                            -- expired executions: late-settlement window (deadline + 24 h)
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create unique index ex_deposit on executions (provider, deposit_address, coalesce(deposit_memo,'')) where deposit_address is not null;
create unique index ex_binding on executions (binding) where binding is not null;
create index ex_due on executions (next_poll_at) where terminal_at is null and next_poll_at is not null;
create index ex_verify on executions (verify_at) where verify_at is not null;
create index ex_inflight_dest on executions (destination_key, amount_out_expected) where terminal_at is null;
create index ex_account_day on executions (account_id, livemode, terminal_at);

create table amount_tags (
  destination_key text not null, tag numeric(78,0) not null check (tag > 0),
  expected numeric(78,0) not null,                       -- base + tag: the amount a watcher matches
  execution_id uuid not null, allocated_at timestamptz not null default now(), released_at timestamptz);
create unique index tags_live on amount_tags (destination_key, expected) where released_at is null;   -- not (destination_key, tag):
-- two executions with base1 + t1 = base2 + t2 would collide although their tags differ
create unique index tags_execution on amount_tags (execution_id) where released_at is null;

create table provider_observations (
  id uuid primary key, execution_id uuid not null, source text not null
    check (source in ('quote','submit','hot_poll','explorer','indexer','watcher','verifier','operator')),
  provider_status text, payload jsonb not null, payload_sha256 bytea not null,
  provider_updated_at timestamptz, applied boolean not null default false, observed_at timestamptz not null default now());
create index obs_execution on provider_observations (execution_id, observed_at);

create table verifications (                             -- final outcomes only (pending retries are not recorded)
  id uuid primary key, execution_id uuid not null, tier char(1) not null, chain text not null, source text not null,
  tx_hash text, block_ref text, recipient text, asset_id text, amount numeric(78,0), finality text,
  result text not null check (result in ('pass','short','mismatch','not_found','pending')),
  details jsonb not null default '{}', checked_at timestamptz not null default now());

create table unattributed_transfers (                   -- transfers to watched destinations that matched no execution
  id uuid primary key, chain text not null, tx_hash text not null, log_ref text not null, recipient text not null,
  asset_id text not null, amount numeric(78,0) not null, reason text not null, observed_at timestamptz not null default now(),
  unique (chain, tx_hash, log_ref));

create table cursors (name text primary key, value jsonb not null, updated_at timestamptz not null default now());
-- names: 'explorer', 'near-indexer', 'evm:<caip2>'
create table control_flags (name text primary key, value boolean not null, reason text,
  source_version integer not null default 0,             -- kill-switch event version: stale engaged/released events are ignored
  updated_at timestamptz not null default now());
-- 'quoting_enabled' (sentinel kill switch), 'cross_chain_enabled' (RB-01); seeded true

create table destination_minimums (                      -- §8.9, recomputed hourly; the API answers from here
  livemode boolean not null, chain text not null, asset_id text not null, kind text not null check (kind in ('intents','chain')),
  min_amount_base numeric(78,0), min_usd_micros bigint, computed_at timestamptz, primary key (livemode, chain, asset_id, kind));

create table execution_transitions (… foundations §7 …);
```

**Chain profiles** (config file `config/chains.yaml`, versioned with the code): `chain`, `blockchain` (1Click code), `family` (evm|near|solana|bitcoin|other), `tier` (A|B), `min_deadline_s`, `default_deadline_s`, `finality` (`evm: finalized tag` · `near: final` · `solana: finalized` · `bitcoin: 1 conf < USD 1k else 2`), `verification_timeout_s`, `watcher` (bool), `memo_mode` (none|required), `explorer_tx_url`. Stage 1 enables Tier A for `near:mainnet` (incl. Intents) and EVM `eip155:8453` (Base), `eip155:42161` (Arbitrum), `eip155:1` (Ethereum); other chains Tier B.

Stage 1 values (`config/chains.yaml`; L2 `finalized` follows L1 finality, ≈ 15–20 min, so the L2 verification timeouts allow for it):

| Chain | Tier | `min_deadline_s` | `default_deadline_s` | `verification_timeout_s` | Watcher |
|---|---|---|---|---|---|
| `near:mainnet` | A | 120 | 1800 | 600 | — (NEAR indexer) |
| `eip155:1` Ethereum | A | 600 | 1800 | 2700 | every 12 s |
| `eip155:8453` Base, `eip155:42161` Arbitrum | A | 300 | 1800 | 3600 | every 2 s |
| Optimism, Polygon, BSC, Avalanche, Gnosis, Solana | B | 300 | 1800 | 1800 | — |
| Bitcoin (1 conf < USD 1k, else 2) | B | 3600 | 7200 | 7200 | — |
| Litecoin, Dogecoin (6 confirmations) | B | 1800 | 3600 | 5400 | — |

## 5. Execution state machine

States and 1Click mapping: `domain-model.md` §4.3. Additional rules:

| Rule | Detail |
|---|---|
| Version | every applied observation that changes `state`, `provider_status` or amounts increments `version` and writes outbox `rails.execution.updated` (or the terminal subject) |
| Monotonic | observations with `provider_updated_at` older than stored are recorded but not applied |
| Changed status only | only a provider status that **differs** from the stored one moves the state; a repeated identical status carries no information (otherwise a contradiction hold would resolve itself on the next identical poll) |
| Success | `verifying → succeeded` only with a `verifications.result='pass'` (Tier A) or provider SUCCESS + Tier B limits respected; a watcher/indexer match on a tagged/bound execution is itself a `pass` and may move any non-terminal state directly to `succeeded` |
| Short | `result='short'` → `succeeded` with `settled_amount` < expected (payments decides review) |
| Held | verification `mismatch`/`not_found` after `verification_timeout_s`, provider `FAILED` after a deposit, or unknown provider status → `held` (+ alert); `held_reason` ∈ `evidence_mismatch`, `evidence_timeout`, `failed_after_deposit`, `unknown_status`. A Tier B settlement over `NP_TIER_B_LIMITS` → `held(evidence_timeout)` (the enum has no Tier-B value) |
| Expiry | no deposit signal and `now > deadline + 10 min` with provider `PENDING_DEPOSIT` → `expired_unpaid`; tracked for 24 h (`tracking_until`) for late refunds and late settlements |
| Late settlement | `expired_unpaid → succeeded` on evidence (evidence wins: the merchant holds the funds, see Success); `expired_unpaid → held` on provider progress without evidence; `expired_unpaid → refunded` on REFUNDED. Terminal states are `succeeded`, `refunded`, `failed` only |
| Tag release | at terminal state (`terminal_at`), or when an expired execution's 24 h tracking window ends → `amount_tags.released_at = now()`; job `tags.release_orphans` (5 min) releases tags reserved by a create that crashed before its execution existed |

## 6. API

| Method & path | Caller (scope) | Request → Response | Errors |
|---|---|---|---|
| `POST /internal/executions` | payments (`rails.executions.write`) | see §8.1 → execution view `{id, version, state, method, provider, route_class, applied_fee_bps, settlement:{amount_base, tag, expected, asset_id}, price_snapshot, deposit?:{address, memo, amount_in, min_amount_in, chain, deadline}, intents_requirement?:{pay_to, token, amount, fee_to, fee_amount, binding, deadline}}` | `asset_unsupported`, `route_unavailable`, `pricing_unavailable`, `screening_blocked`, `deadline_too_short`, `quote_rejected`, `quote_integrity_failed`, `rail_unavailable` |
| `GET /internal/executions/{id}` | payments, agentpay (`rails.read`) | execution view + latest evidence | |
| `GET /internal/executions?deposit_address=&memo=` | sentinel (`rails.read`) | `{exists, execution_id, account_id, livemode}` — used by sentinel **only** to decide whether a mismatch is a genuine integrity breach (kill switch) or a spoofed request | |
| `GET /internal/assets/{asset_id}?livemode=` | agentpay, payments, sentinel (`rails.read`) | `{asset_id, chain, decimals, is_usd_stable, price_usd, price_ok, as_of}` from that mode's registry (`livemode` optional since spec 0.1.3: omitted → live, falling back to test) | `asset_unsupported` |
| `POST /internal/executions/{id}/deposit_tx` | payments | `{tx_hash, sender?}` → `202` | `state_conflict` |
| `POST /internal/executions/{id}/signals` | payments | `{type:'payer_sent'}` → `202` | |
| `GET /internal/options?destination_asset=&destination_type=&amount_out=&chain=&wallet=&livemode=` | payments | fees are **indicative from configuration** (rails has no merchant fee schedule on this call; the attempt quotes the real fee) → `{options:[{method: route_quote\|intents_transfer, origin_asset_id, chain, symbol, decimals, estimated_amount_in, route_class, applied_fee_bps, min_deadline_s, available, reason?}]}` — an `intents_transfer` option is listed when the destination is an Intents account (paying asset = destination asset) | |
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
   livemode and NP_TREASURY_ACCOUNT unset → rail_unavailable (live quoting refused); test mode uses NP_TEST_TREASURY_ACCOUNT
1. existing = by idempotency_key → return view (idempotent)
2. o = tokens[livemode][origin], d = tokens[livemode][destination]; both enabled else asset_unsupported
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
10. integrity: res.quoteRequest ⊇ quoteReq on {recipient, destinationAsset, amount, refundTo, refundType, recipientType, originAsset};
    appFees (split-aware, see below); EXACT_OUTPUT → res.quote.amountOut == amount; depositAddress valid for origin chain;
    quote.deadline ≥ now + min; verifyQuoteSignature(res) (SDK) → any failure → quote_integrity_failed + alert, tag released
11. tx: insert execution(awaiting_deposit, …, version 1) + observation(source 'quote') + transition + outbox updated
```
`SLIPPAGE`: same_asset 30 bps, stable_pair 50 bps, volatile 100 bps (config).

**appFees split (recorded dry quote 2026-11-05):** 1Click echoes `appFees: [{treasury, fee}]` split 50/50 with its own fee account, e.g. `[{treasury, 50}]` → `[{treasury, 25}, {<1Click implicit account>, 25}]`. The signed request sets `appFees: undefined`, so **appFees are not covered by the quote signature** (SP-01) and must be checked here: at most 2 entries, their total equals the requested fee, exactly one entry is the treasury and it keeps at least `fee − max(ceil(fee × share_bps / 10 000), floor)` (`NP_PROVIDER_SPLIT`: share 5000, floor 20 bps, 1 bp on stable/same-asset routes). A fee-free quote has no entries. Sentinel R6 applies the same rule. **SP-01 resolved:** `verifyQuoteSignature` of the SDK verifies 1Click quotes with the key pinned in SDK 0.1.26 (`ed25519:reYaWhvwu8Jzo3WUM3zhn6VrhuMEF4eADL17qtRVifc`); the sandbox signs with the same construction and its own key.

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
loop ≤ 5: t = random(1..T); insert amount_tags(destination_key, t, expected = base + t) → unique violation on (destination_key, expected) → retry
all 5 collide → choose smallest free via scan; none free → tag = 0 and mark execution 'untagged' (attribution via provider hash)
```
Amount matching by watchers and the indexer considers **only tagged executions** (untagged route executions settle through the verifier on the provider's tx hash; `intents_transfer` executions match by binding); more than one candidate is ambiguous and attributes nothing. Expired executions are candidates only during their 24 h tracking window.

### 8.4 Hot poller (`rails-tracker`, N replicas)
```
every 500 ms: batch = select … from executions where terminal_at is null and next_poll_at <= now()
                order by priority, next_poll_at limit 50 for update skip locked
for each (token bucket NP_ONECLICK_STATUS_RPS, priority: has tx hash 1, signalled 2, sweep 3):
  r = provider.status(deposit_address, memo) (4 s)
  observation(hot_poll); applyObservation(r)
  next_poll_at = schedule(execution): signalled/deposit_detected/settling: [2,4,8,15,15,…] s by poll_count, capped 15 s, stop after 2 h → 5 min
                 awaiting without signal: deadline + 10 min (single sweep) when the Explorer feed runs; without it (Stage 1: no
                   Explorer token) every NP_AWAITING_SWEEP_S (30 s), since nothing else notices an unsignalled deposit
                 expired_unpaid: every 15 min until deadline + 24 h (late refunds / settlements)
                 verifying: verifier owns it (null)
```

### 8.5 Explorer feed (`rails-feeds`, leader)
Every 5 s one request (rotating three windows): (a) `statuses=PROCESSING,INCOMPLETE_DEPOSIT` newest 1,000; (b) `statuses=SUCCESS,REFUNDED,FAILED` with `startTimestamp = cursor − 30 min`; (c) newest 1,000 of all statuses since cursor. Rows are matched by `(depositAddress, depositMemo)` → if row status ≠ execution.provider_status: `has_signal = true, next_poll_at = now()`, record observation(explorer). Filter `referral=nearpay` (SP-10 confirms semantics). 429 → back off to 10 s. Contract as read from the Explorer `openapi.yaml` (2026-11-06): Bearer JWT; the response is a bare array; `numberOfTransactions` ≤ 1000; `statuses` is comma-separated and its default omits `PENDING`/`INCOMPLETE`; `startTimestamp` is ISO-8601 and exclusive; rows carry no `updatedAt`, and whether the time filter applies to creation or update is undocumented (SP-10 stays open); 1 request per 5 s.

### 8.6 NEAR indexer (`rails-feeds`, leader)
```
source A: neardata (FastNEAR) finalized block stream; source B: NEAR Lake; dedupe by block hash; cursor 'near-indexer'
for each receipt outcome with executor in {intents.near, v1.vault.nearpay.near, NEP-141 contracts of enabled destination tokens}:
  parse logs EVENT_JSON → dip4 (intents_executed, token_diff, transfer), nep245 mt_transfer, nep141 ft_transfer, nearpay_vault
  intents.near DIP-4 transfer to watched intents account:
     match execution by binding (the DIP-4 transfer memo 'np1:…') or by (destination_key, amount == amount_out_expected) among
     tagged in-flight executions → evidence(pass)
     (SP-14: the DIP-4 `transfer` event carries the memo; the NEP-245 `mt_transfer` of a transfer intent carries none, so
      binding attribution never reads mt_transfer memos — foundations §14)
  nep141 ft_transfer to watched NEAR address → same matching (chain_address destinations on NEAR)
  nearpay_vault events → outbox evt.rails.vault.event
gap > 50 blocks behind final → alert; restart resumes from cursor (at-least-once; matching is idempotent)
```
**Demand-driven trackers:** the NEAR indexer and the EVM watchers make no external call while nothing live is in flight on their chain, and then forget their cursor (they restart at the head when the next execution appears).

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
- NEAR: `EXPERIMENTAL_tx_status(final)` → NEP-141 `ft_transfer` event to recipient, or `intents.near` transfer to the Intents account. All-shard RPC nodes (FastNEAR) accept any `sender_account_id`, so the verifier needs only the tx hash; `wait_until: NONE` answers at once (`UNKNOWN_TRANSACTION` for an unknown hash).
- Solana / Bitcoin (Stage 4): finalized tx token-balance delta / output to address with required confirmations.
Compare `amount ≥ amount_out_expected` → `pass`; `0 < amount < expected` → `short`; otherwise `mismatch`. Not yet final → retry every 10 s (`verify_at`) until `verification_timeout_s` → `held`; only final outcomes are written to `verifications`.

### 8.9 Destination minimums
For `(chain, asset)` hourly: dry quote `EXACT_OUTPUT` of USD 100 equivalent paid from a NEAR USD stablecoin in an Intents balance (`depositType INTENTS`; `refundTo` = the treasury, or `intents.near` when no live treasury is set; test mode: the test treasury) → `withdrawFee` (USD); `min_usd = max(1.00, 20 × withdrawFee_usd)`; Intents-account destinations: `0.01`. Stored in `destination_minimums`; unknown keys and unavailable quotes → `null`.

## 9. Jobs and loops

| Loop/job | Where | Cadence | Failure |
|---|---|---|---|
| token & price refresh | feeds (leader) | `/v0/tokens` every 30 s (1Click refreshes prices ≈ 1/min, so the par policy accepts a price ≤ 5 min old; the dual-source rule keeps 60 s); secondary prices every 30 s; FX every 5 min (no secondary source or FX in Stage 1) | stale → pricing_unavailable for affected assets |
| `tags.release_orphans` | feeds (job) | every 5 min | releases tags of creates that crashed before the execution row existed |
| `kit.purge` | feeds (job) | hourly | published outbox, old inbox, expired idempotency keys |
| hot poller | tracker | 500 ms tick | breaker on provider; executions keep `next_poll_at` |
| Explorer feed | feeds (leader) | 5 s | back off on 429 |
| NEAR indexer | feeds (leader) | continuous | switch source on gap/stall > 30 s |
| sandbox feed (test mode) | feeds (leader) | 1 s poll of sandbox `/sim/intents/events` and sandbox status for test executions | same matching code as the NEAR indexer |
| EVM watchers | feeds (leader per chain) | every 2 s (L2) / 12 s (L1) | RPC pool failover |
| verifier | tracker | event-driven + 10 s retries | → held after timeout |
| minimums | feeds | hourly | keep last value |
| observation archive | feeds | daily: > 90 days → Parquet to S3 (server stage), keep hash — Stage 2 | retry |

## 10. Integrations

| Target | Timeout | Budget | Notes |
|---|---|---|---|
| 1Click `/v0/quote` | 6 s | `NP_ONECLICK_QUOTE_RPS` (5) | partner key (JWT → `Authorization: Bearer`); breaker |
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

`NP_ONECLICK_BASE_URL`, `NP_ONECLICK_API_KEY_FILE` (live only; absent in Stage 1 local → unauthenticated quotes with +25 bps, appFees still sent), `NP_TEST_TREASURY_ACCOUNT` (test-mode fee recipient, sandbox only), `NP_PROVIDER_SPLIT` (1Click appFees split, §8.1), `NP_AWAITING_SWEEP_S` (30), `NP_SCREENING_FAIL_CLOSED_USD` (1000), `NP_AGREEMENT_USD` (EVM pool agreement threshold), `NP_IDENTITY_URL` + `NP_SERVICE_CLIENT_SECRET_FILE` (service token: `compliance.screen`, `sandbox.use`), `NP_COMPLIANCE_URL`, `NP_VALKEY_URL` (shared 1Click budgets; down → half per replica), `NP_SANDBOX_BASE_URL`, `NP_SANDBOX_SIGNING_PUBKEY`, `NP_EXPLORER_TOKEN_FILE`, `NP_TREASURY_ACCOUNT` (e.g. `treasury.nearpay.near`, provided by the owner; unset → live quoting refused), `NP_NEAR_RPC_URLS`, `NP_NEARDATA_URL`, `NP_EVM_RPC_URLS_<CHAINID>`, `NP_PRICE_SECONDARY_URL`, `NP_PRICE_SECONDARY_KEY_FILE`, `NP_FX_URL`, `NP_SLIPPAGE_BPS` (json), `NP_CHAINS_FILE`, `NP_ONECLICK_QUOTE_RPS`, `NP_ONECLICK_STATUS_RPS`, `NP_EXPLORER_ENABLED` (false until the owner provides an Explorer token), `NP_PRICING_CURRENCIES` (Stage 1: `usd`), `NP_TIER_B_LIMITS` (`{per_payment_usd: 2000, per_day_usd: 20000}`).

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
- Property: for any sequence of observations (duplicates, reordering, conflicting sources) the execution never leaves a terminal state (`succeeded`, `refunded`, `failed`); `expired_unpaid` only moves to `refunded`, `succeeded` (with evidence) or `held`, and `held` only to `succeeded` (with evidence) or `refunded`; settled ⇒ evidence present; `base + tag` unique among in-flight executions per destination.
- Contract: 1Click adapter against recorded fixtures (`test/fixtures/oneclick`), including `KNOWN_DEPOSIT_TX`, 404, 400, signature failure.
- Integration: indexer parsing on recorded mainnet blocks (fixtures from SP-14); EVM watcher against an Anvil fork with Transfer logs; verifier tiers.
- Load: hot poller at 1,000 in-flight executions within budget.

## 16. Stage mapping

| Stage | Scope |
|---|---|
| 1 | token registry + pricing (par + primary/secondary), `oneclick` + `sandbox` + `direct_intents` providers, executions, hot poller, Explorer feed, NEAR indexer (Intents + NEP-141), EVM watchers for Base/Arbitrum/Ethereum, verifiers NEAR + EVM, minimums, options, evidence API |
| 2 | refunds (`EXACT_INPUT`/`EXACT_OUTPUT` legs), reconciliation evidence, observation archive |
| 4 | Solana + Bitcoin watchers/verifiers, vault event forwarding hardening |

## 17. Ops CLI (Stage 1)

`docker compose -p nearpay exec rails-api /nodejs/bin/node dist/ops.js <command>`: `execution-show <ex_… | deposit address>` · `flags` · `flag-set quoting_enabled|cross_chain_enabled true|false <reason>` (RB-01) · `tokens-refresh live|test` · `minimums-refresh`.
