# LLD — `nearpay-rails`

> Version 1.12 (step 4.1, spec 0.4.0: Solana and Bitcoin as Tier A chains — profiles, assets, Solana watcher, Bitcoin confirmation tracking, verifiers, `Evidence.solana`/`Evidence.bitcoin`, test mode — §21) · Version 1.11 (step 4.1 auto-debit contracts, spec 0.4.0: `evt.rails.vault.event` payload `VaultEventRecord`, envelope rules and the sandbox stand-in feed fields, §8.6) · Version 1.10 (step 4.1: vault event forwarding rules, §8.6 — `contracts-vault.md` v1.4 §4.1, AI audit A2-M-4) · Version 1.9 (step 3.2b, spec 0.3.1: `getEvidenceSummary?owner=`, sandbox verifier sender from `SimTx.transfers[].from` — §20.1) · Version 1.8 (step 3.2 implementation: ownership enforced by caller identity, agentpay read rule, `Evidence.sender` per detector, migration `0004`, evidence-summary gap — §20.1) · Version 1.7 (step 3.1, spec 0.3.0: executions owned by agentpay for cross-chain x402 — `owner`, `x402_binding`, `Evidence.sender`, `listEvidence?owner` — §20) · Version 1.6 (spec 0.2.1 implementation: observed-fee and deposited-amount sources, abandon rules, `control_flags.known` and the boot read, nonce issuance, migration `0003` — §"Spec 0.2.1 contract changes") · Spec 0.2.1 update (spec 0.2.1: abandon refund legs, observed fee, deposited amount, decimals, fee share of `amount_in`, refund-leg nonce, fresh screening, FX codes on refund legs — §"Spec 0.2.1 contract changes") · Version 1.5 (stage 2 audit finding 4: refund quote bounds against §8.2 pricing — payer floor within the route slippage, price band, merchant cost cap; every route leg is priced, pricing unavailable refuses — §8.1 step 10, §6.1) · Version 1.4 (step 2.2 implementation findings: refund-leg mapping and verification floor §6.1, migration `0002`, test-mode FX from config, asset-priced par, completeness and archive details §8.10–§8.11, Explorer test cursor, contract gaps; changes in §19) · Version 1.3 (step 2.1 contracts, spec 0.2.0 — console executions routes, reverify and `ExecutionRefund.token` = `TokenContract` settled at integration §6: pricing/FX with ECB + CoinGecko and staleness bounds §8.2, Explorer feed with the owner token §8.5, reconciliation evidence §8.10, observation archive §8.11, live 1Click shape §3; changes in §18) · Version 1.2 (step 1.2 implementation findings folded: data and chain profiles §4, state rules §5, appFees split and SP-01 §8.1, tags §8.3, tracking §8.4–§8.8, minimums and options §8.9, configuration §11, ops CLI §17; SP-01 and SP-14 resolved) · Version 1.1 audited (`audits/rails.md`) · HLD §10 · ADR-0003, ADR-0006, ADR-0012 · Foundations `00-foundations.md` · Provider facts `reference/landscape.md` §2

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
src/providers/oneclick/      1Click client (JWT partner key → Authorization: Bearer, else X-API-Key), request/response mapping, signature check (SDK); since spec 0.2.0 the `OneClick*` schemas accept the live shape (nullish optionals, status-nested quote without `correlationId`), so the Stage 1 body normalisation can go
src/providers/sandbox/       same client, sandbox base URL + sandbox signature key
src/providers/direct-intents/ requirement builder for signed Verifier transfers (no swap)
src/providers/prices/        PriceProvider port: oneclick (primary), coingecko (secondary), sandbox (test)
src/providers/fx/            FxProvider port: ecb (live), fixed (test table)
src/providers/explorer/      Explorer client (owner JWT) and sandbox mirror (same schemas)
src/archive/                 observation archive writer (Parquet) and archive store (file | s3)
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
-- Stage 2 (migration 0003): prices are kept as exact rationals besides the display numerics
alter table tokens add column peg_currency text,                      -- 'usd' for is_usd_stable; 'eur' etc. from NP_STABLE_PEGS
  add column coingecko_id text,                                          -- from /v0/tokens coingeckoId (NP_PRICE_SECONDARY_IDS overrides)
  add column price_num numeric(78,0), add column price_den numeric(78,0),
  add column secondary_num numeric(78,0), add column secondary_den numeric(78,0);
create table fx_rates (                                   -- one row per (mode, currency): the latest rate to USD
  livemode boolean not null, currency text not null, num numeric(78,0) not null, den numeric(78,0) not null,
  source text not null check (source in ('ecb','fixed')), reference_date date, as_of timestamptz not null,
  fetched_at timestamptz not null, valid_until timestamptz, primary key (livemode, currency));
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
  x402_binding text,                                     -- Stage 3 (migration 0004): owner agentpay only, unique; check (owner = 'agentpay') = (x402_binding is not null)
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
-- Stage 2 (migration 0003): payload becomes nullable; archived rows keep the hash and point into the archive (§8.11)
alter table provider_observations alter column payload drop not null,
  add column archive_key text, add column archive_row integer, add column archived_at timestamptz,
  add constraint obs_archived check ((payload is null) = (archive_key is not null));
alter table executions add column explorer_status text, add column explorer_seen_at timestamptz;   -- last Explorer row status (§8.5, Tier-B check)
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
| Optimism, Polygon, BSC, Avalanche, Gnosis | B | 300 | 1800 | 1800 | — |
| `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` Solana (Stage 4: Tier A, §21) | A | 300 | 1800 | 1800 | every 4 s (§21.2) |
| `bip122:000000000019d6689c085ae165831e93` Bitcoin (1 conf < USD 1k, else 2; Stage 4: Tier A, §21) | A | 3600 | 7200 | 7200 | every 60 s, signal only (§21.2) |
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
| `POST /internal/executions` | payments (`rails.executions.write`) | see §8.1; non-USD fiat pricing since Stage 2 (`currency_unsupported`, `fx_rate_unavailable`) → execution view `{id, version, state, method, provider, route_class, applied_fee_bps, settlement:{amount_base, tag, expected, asset_id}, price_snapshot, deposit?:{address, memo, amount_in, min_amount_in, chain, deadline}, intents_requirement?:{pay_to, token, amount, fee_to, fee_amount, binding, deadline}}` | `asset_unsupported`, `route_unavailable`, `pricing_unavailable`, `screening_blocked`, `deadline_too_short`, `quote_rejected`, `quote_integrity_failed`, `rail_unavailable` |
| `POST /internal/refund_executions` (Stage 2) | payments (`rails.executions.write`) | `CreateRefundExecution` (payments §8.4, §17): §8.1 with `kind: refund` — screen the recipient, no app fee, no tag, `swapType` from the request (`EXACT_OUTPUT` amount converted by §8.2 pricing), `refundTo` = the merchant destination, deposit type `INTENTS` for Intents sources; `intents_transfer` legs get a binding and provider `direct_intents`; view with `refund` and `reference` set | as `POST /internal/executions` |
| `GET /internal/executions/{id}` | payments, agentpay (`rails.read`) | execution view + latest evidence | |
| `GET /internal/executions?deposit_address=&memo=` | sentinel (`rails.read`) | `{exists, execution_id, account_id, livemode}` — used by sentinel **only** to decide whether a mismatch is a genuine integrity breach (kill switch) or a spoofed request | |
| `GET /internal/assets/{asset_id}?livemode=` | agentpay, payments, sentinel (`rails.read`) | `{asset_id, chain, decimals, is_usd_stable, price_usd, price_ok, as_of}` from that mode's registry (`livemode` optional since spec 0.1.3: omitted → live, falling back to test) | `asset_unsupported` |
| `POST /internal/executions/{id}/deposit_tx` | payments | `{tx_hash, sender?}` → `202` | `state_conflict` |
| `POST /internal/executions/{id}/signals` | payments | `{type:'payer_sent'}` → `202` | |
| `GET /internal/options?destination_asset=&destination_type=&amount_out=&chain=&wallet=&livemode=` | payments | fees are **indicative from configuration** (rails has no merchant fee schedule on this call; the attempt quotes the real fee) → `{options:[{method: route_quote\|intents_transfer, origin_asset_id, chain, symbol, decimals, estimated_amount_in, route_class, applied_fee_bps, min_deadline_s, available, reason?}]}` — an `intents_transfer` option is listed when the destination is an Intents account (paying asset = destination asset) | |
| `GET /internal/destination-minimums?keys=` | identity, payments | `{<destination_key>: {min_amount_base, min_usd_micros, computed_at}}` | |
| `GET /internal/chain-profiles` | payments, checkout via payments | profiles (public subset) | |
| `GET /internal/evidence?account_id=&livemode=&day=&states=&kind=` | payments (reconciliation), sentinel, ops | executions that reached a terminal state on the UTC day (by `terminal_at`) with evidence; default `states=succeeded`, both kinds (Stage 2 filters, spec 0.2.0) | |
| `GET /v1/assets` | anonymous | enabled tokens `{asset_id, chain, symbol, decimals, family, is_usd_stable}` | |
| `GET /internal/fx-rates?livemode=&currencies=` (Stage 2) | payments, ops (`rails.read`) | `FxRates {base:'usd', livemode, currencies:[FxRate{currency, exponent, available, stale, reference_date, rate: PriceRate}], as_of}` — the accepted pricing currencies (`NP_PRICING_CURRENCIES`, `usd` first) with source, `as_of` and `valid_until` of each rate; a stale rate is shown with `available:false` and never used | |
| `GET /internal/settlement-estimate?amount=&currency=&asset_id=&settlement_asset=&livemode=` (Stage 2) | payments, ops (`rails.read`) | `SettlementEstimate {pricing, usd_micros, settlement?:{asset_id, amount_base, par}, price_snapshot}` — the §8.2 rules without quoting: payments values non-USD prices in USD micros (minimum, unverified cap) and gets the options `amount_out` | `currency_unsupported`, `asset_unsupported`, `pricing_unavailable`, `fx_rate_unavailable` |
| `GET /internal/executions/{id}/observations?starting_after=&limit=` (Stage 2) | payments, ops (`rails.read`) | `{data:[ProviderObservation{id obs_, source, provider_status, provider_updated_at, applied, observed_at, payload_sha256, payload \| null, archive \| null}], has_more}` oldest first | |
| console `/ops/executions…` (Stage 2, routes in the identity-console `/ops/…` table, served by rails; final shape at step 2.1 integration, spec 0.2.0) | console (`ops:rails.read`; reverify `ops:rails.verify`) | `listOpsExecutions` `GET /ops/executions` (`livemode` **required** — the explorer never mixes modes, INV-8; `state` typed `AttemptStatus`, `held` = Verifier holds; `held_reason`, `kind`, `provider`, `account`, `deposit_address`, created, paging) · `getOpsExecution` `GET /ops/executions/{id}` → `Execution` · `listOpsExecutionObservations` `GET /ops/executions/{id}/observations` → `ProviderObservationPage` · `reverifyOpsExecution` `POST /ops/executions/{id}/reverify` (`{note}`, ops + ops_admin, step-up, `202 Execution`): sets `verify_at = now()` for a `held`/`verifying` execution and records an `operator` observation; never forces a state (INV-2); other states `409 state_conflict`; audited with the operator id | `state_conflict` |
| `GET /internal/evidence/summary?account_id=&livemode=&day=` (Stage 2) | payments (reconciliation), sentinel, ops (`rails.read`) | `EvidenceSummary` (§8.10) | |

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
    kind=='refund' (stage 2 audit finding 4), on top of the above: priced = settle({asset: origin, amount}, d) (§8.2)
    for every route leg (pricing failure → pricing_unavailable / fx_rate_unavailable before quoting, fail closed);
    EXACT_INPUT → amountIn == amount, 0 < minAmountOut ≤ amountOut,
                  minAmountOut ≥ floor(amountOut × (10000 − SLIPPAGE[route_class]) / 10000),
                  |amountOut − priced| ≤ priced × BAND / 10000;
    EXACT_OUTPUT → amountOut == priced, amountIn ≤ amount × (10000 + BAND + SLIPPAGE[route_class]) / 10000;
    BAND = 150 bps (the §8.2 dual-source agreement band). The payer of an EXACT_INPUT leg is thus guaranteed
    ≈ (1 − BAND)(1 − SLIPPAGE) of the priced value; the merchant of an EXACT_OUTPUT leg pays at most BAND + SLIPPAGE
    over it. A failure is quote_integrity_failed + alert like the checks above (`checkRefundBounds`, pure)
11. tx: insert execution(awaiting_deposit, …, version 1) + observation(source 'quote') + transition + outbox updated
```
`SLIPPAGE`: same_asset 30 bps, stable_pair 50 bps, volatile 100 bps (config).

**appFees split (recorded dry quote 2026-11-05):** 1Click echoes `appFees: [{treasury, fee}]` split 50/50 with its own fee account, e.g. `[{treasury, 50}]` → `[{treasury, 25}, {<1Click implicit account>, 25}]`. The signed request sets `appFees: undefined`, so **appFees are not covered by the quote signature** (SP-01) and must be checked here: at most 2 entries, their total equals the requested fee, exactly one entry is the treasury and it keeps at least `fee − max(ceil(fee × share_bps / 10 000), floor)` (`NP_PROVIDER_SPLIT`: share 5000, floor 20 bps, 1 bp on stable/same-asset routes). A fee-free quote has no entries. Sentinel R6 applies the same rule. **SP-01 resolved:** `verifyQuoteSignature` of the SDK verifies 1Click quotes with the key pinned in SDK 0.1.26 (`ed25519:reYaWhvwu8Jzo3WUM3zhn6VrhuMEF4eADL17qtRVifc`); the sandbox signs with the same construction and its own key.

**`intents_transfer` method:** steps 1–6 without a provider; destination must be `intents_account` and `origin.asset_id == destination.asset_id`; `fee_amount = max(ceil(base × bps / 10000), rule.min_fee)`; `binding = 'np1:' + base58(sha256(execution_id ‖ pay_to ‖ token ‖ amount ‖ fee ‖ deadline))`; no tag (binding attributes); provider `direct_intents`; state `awaiting_deposit` until the indexer sees the transfer.

### 8.2 Pricing (`domain/pricing`)
```
settle(pricing, d):                                                   // exact rationals, kit money (no floats, INV-5)
  if pricing.currency == 'asset':
     if pricing.asset == d → return amount
     usd = amount / 10^dec(pricing.asset) × price(pricing.asset)        // price() = primary, dual-source checked
  else:
     c = pricing.currency; c ∉ NP_PRICING_CURRENCIES → currency_unsupported
     if d.peg_currency == c and parOk(d): return amount × 10^(d.decimals − exp(c))       // same-currency par (USDC for usd, EURC for eur)
     usd = amount / 10^exp(c) × fx(c)                                    // fx(usd) = 1 (source fixed)
  if d.peg_currency == 'usd' and parOk(d): return floor(usd × 10^d.decimals)              // USD par
  return floor(usd / price(d) × 10^d.decimals)                          // non-par: needs both price sources
price(t): primary = 1Click /v0/tokens price, valid ≤ NP_PRICE_PRIMARY_MAX_AGE_S (120) after priceUpdatedAt;
          secondary = CoinGecko price, valid ≤ NP_PRICE_SECONDARY_MAX_AGE_S (300) after last_updated_at;
          both valid and |primary − secondary| / secondary ≤ 1.5% → primary; else pricing_unavailable(t)
parOk(d): d.peg_currency set and |price_primary(d) / fx(peg) − 1| ≤ 2% with a primary price ≤ 5 min old; when a valid
          secondary exists it must pass the same band (depeg guard); false → par disabled for d (alert), non-par rules apply
fx(c):    the stored rate of c with now < valid_until, else fx_rate_unavailable(c) — a stale rate is never used
```
All arithmetic uses bigint rationals (kit `@nearpay-internal/money`: `parseDecimal`, `mul`/`div`, `floor`); conversions to the settlement amount round **down** (foundations §6). The snapshot stored on the execution and returned as `price_snapshot` is `{primary_usd, secondary_usd, fx, par, as_of}` (decimal strings, kept for 0.1.5 readers) plus, since spec 0.2.0, `currency`, `par_currency` and `rates[]` — every `PriceRate` used, exact (`value: {num, den}`) with `source`, `as_of`, `fetched_at` and `valid_until`, so payments, the dashboard and the console can show "EUR→USD 1.0823, ECB reference rate of 2026-11-06".

**Sources (ADR-0020 decision 5; facts in `reference/landscape.md` §8).** Each sits behind a port so a vendor is a new adapter, not a contract change.

| Port | Live adapter | Test-mode adapter | Refresh |
|---|---|---|---|
| `PriceProvider` primary | 1Click `GET /v0/tokens` `price` + `priceUpdatedAt` (the token refresh, §9) | sandbox `/v0/tokens` fixed prices | 30 s |
| `PriceProvider` secondary | **CoinGecko keyless** `GET https://api.coingecko.com/api/v3/simple/price?ids=<ids>&vs_currencies=usd&include_last_updated_at=true&precision=full`, one batched call (≤ 515 ids) for enabled tokens that are non-par settlement assets or stablecoins (depeg guard); ids = `/v0/tokens` `coingeckoId`, overridden per asset by `NP_PRICE_SECONDARY_IDS` | the sandbox price again (role `secondary`, source `sandbox`) | 60 s; 429 → back off ×2 up to 10 min |
| `FxProvider` | **ECB** daily XML `https://www.ecb.europa.eu/stats/eurofxref/eurofxref-daily.xml` (29 currencies vs EUR) | fixed table `NP_TEST_FX_RATES` (source `fixed`, `valid_until` null) | every 30 min (conditional GET) |

Feed numbers are read without floats: JSON prices go through kit `parseJsonDecimals` (number tokens kept as their source text) and then `parseDecimal`; the ECB `rate="1.0823"` attributes are decimal strings already. **Cross rates:** ECB quotes `C per EUR`; `fx(eur) = USD_per_EUR`, `fx(c) = USD_per_EUR / C_per_EUR` (an exact rational, generally not a finite decimal, hence `Ratio`). `usd` needs no source.

**ECB staleness bound (publication calendar).** The ECB publishes once per TARGET working day at about 16:00 CET; TARGET closing days are Saturdays, Sundays, 1 January, Good Friday, Easter Monday, 1 May, 25 and 26 December (`domain/pricing/target-calendar.ts`, Easter by the Gregorian computus). For the rates of reference day `D`: `as_of = D 16:00 Europe/Berlin`; `valid_until = min(next TARGET working day after D at 16:00 Europe/Berlin + NP_FX_GRACE_S (7200 s), as_of + NP_FX_MAX_AGE_S (518 400 s = 6 days))`. A Friday rate therefore serves until Monday 18:00 CET (~74 h); the Thursday before Easter until Tuesday 18:00 CET (~5 days 2 h); the hard cap catches a missed publication or an unknown closing day. A late or missing publication makes the currency **unavailable** at `valid_until` (`FxRate.stale = true`, `available = false`, alert `fx_stale{currency}`), never silently older: new sessions/intents and executions priced in it fail with `fx_rate_unavailable`; executions already quoted keep their snapshot. USD pricing settled at par in USD stablecoins never needs FX or the secondary feed (a non-USD peg such as EURC needs the FX rate for its depeg guard).

**Accepted pricing currencies:** `NP_PRICING_CURRENCIES` (Stage 2 default `usd,eur,gbp,chf,jpy,cad,aud`), each must be an ECB currency (or `usd`/`eur`); `GET /internal/fx-rates` is the source of truth payments reads (payments §8.1 `currency_unsupported` / `fx_rate_unavailable`). Disclosure (ADR-0020): ECB reference rates are for information purposes and move once a day; merchants pricing in non-USD fiat carry the intraday drift, shown with the rate's source and reference date.

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

**Stage 2 (spec 0.2.0 `Explorer*` schemas).** The feed runs when the owner's Partner Dashboard JWT is present (`NP_EXPLORER_TOKEN_FILE`; `NP_EXPLORER_ENABLED` defaults to true then). Live rows are validated with `ExplorerTransaction` (nullish optionals, `status` a plain string). Window (a) and (b) set `statuses` explicitly (the API default omits `PENDING_DEPOSIT`/`INCOMPLETE_DEPOSIT`); `numberOfTransactions=1000`; a window that returns 1000 rows continues on its next turn with `lastDepositAddress`/`lastDepositMemo` (`direction=next`) until a page is short. Only rows whose status differs from `executions.explorer_status` are recorded (observation `explorer`, payload hash) and nudge the poller; an unknown status is recorded and nudges the poller too (its status mapping decides `held`). `explorer_status`/`explorer_seen_at` feed the Tier-B cross-check of §8.10. Rows of unknown deposit addresses are ignored (another integrator's `nearpay` referral would be a misconfiguration; counted as `explorer_unmatched_total`). **Test mode** reads the sandbox mirror `GET /api/v0/transactions` with the same client and schemas (service token instead of the JWT, 1 s cadence, no back-off), so the feed is covered by E2E. With the feed running, awaiting executions without a signal keep only the single deadline sweep (§8.4).

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
  nearpay_vault events → outbox evt.rails.vault.event   (only per the vault rules below)
gap > 50 blocks behind final → alert; restart resumes from cursor (at-least-once; matching is idempotent)
```
**Vault event forwarding (Stage 4, v1.10).** NEAR keeps the logs of failed receipts, and a payer can tune gas so a vault receipt logs `mandate_created`/`topped_up` and then fails (phantom events with a reusable `mandate_id`). Rails therefore forwards a `nearpay_vault` event only if **all** hold: (1) the receipt's `executor_id` is exactly `NP_VAULT_CONTRACT` (the event's `standard` alone proves nothing — any contract can log it); (2) the receipt outcome status is `SuccessValue` or `SuccessReceiptId`; (3) the block is final; (4) the envelope parses with `version = "1.0.0"` and the event's spec schema (`contracts-vault.md` §4.1). Anything else is counted (`vault_event_rejected_total{reason}`) and dropped, never forwarded. The outbox payload carries `{receipt_id, block_height, block_hash, log_index, event}`; the dedupe key is `receipt_id:log_index` (at-least-once, INV-4). In test mode the same rules apply to the sandbox stand-in's receipts (OQ-T-03).

**Spec 0.4.0 contract (v1.11).** Payload = `VaultEventRecord {vault_contract, tx_hash, receipt_id, log_index, block_height, block_hash, block_timestamp (ns string), event}` — `vault_contract` (the executor, = `NP_VAULT_CONTRACT`), `tx_hash` (explorer links on the mandate and invoice) and `block_timestamp` (billing's `created`/`collected_at`) are added to the v1.10 list; `event` is the parsed `VaultEvent` (spec `parseVaultEventLog`, which also enforces rule (4)). Envelope: `account_id: null` (rails cannot know the merchant; billing resolves it from `ext_ref` / its mirror), `livemode: true` for the mainnet vault and `false` for the sandbox stand-in, aggregate `{type: "vault_mandate", id: "<vault_contract>#<mandate_id>", version: block_height}`; consumer billing only. Rejections are counted per `VAULT_EVENT_REJECT_REASONS` (`wrong_executor`, `failed_receipt`, `not_final`, `bad_version`, `bad_schema`). **Sandbox adapter:** the stand-in's lines arrive on `GET /sim/intents/events` with `receipt_id`, `log_index`, `executor_id`, `block_hash`, `block_timestamp`, `receipt_status` (sandbox §3.1); an item without `executor_id` is a Verifier event as before, and the adapter applies rules (1)–(4) with `NP_VAULT_CONTRACT` = the stand-in account (sandbox blocks count as final). Byte-level parity of the lines with the vault: `test-vectors/vault.json`.

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
- Solana / Bitcoin (Stage 4, §21.3): finalized tx token-balance delta / outputs to the address with the required confirmations (`confirmation_tiers`).
Compare `amount ≥ amount_out_expected` → `pass`; `0 < amount < expected` → `short`; otherwise `mismatch`. Not yet final → retry every 10 s (`verify_at`) until `verification_timeout_s` → `held`; only final outcomes are written to `verifications`.

### 8.9 Destination minimums
For `(chain, asset)` hourly: dry quote `EXACT_OUTPUT` of USD 100 equivalent paid from a NEAR USD stablecoin in an Intents balance (`depositType INTENTS`; `refundTo` = the treasury, or `intents.near` when no live treasury is set; test mode: the test treasury) → `withdrawFee` (USD); `min_usd = max(1.00, 20 × withdrawFee_usd)`; Intents-account destinations: `0.01`. Stored in `destination_minimums`; unknown keys and unavailable quotes → `null`.

### 8.10 Reconciliation evidence (Stage 2)
Payments' daily reconciliation (payments §9 `reconcile.daily`, 02:00 UTC) compares its statement with rails per account, mode and UTC day:
```
day membership: executions with terminal_at in [day 00:00Z, day+1 00:00Z) (succeeded, refunded, failed); refund legs are kind 'refund'
GET /internal/evidence/summary → totals per (kind, terminal state, asset): count, Σ settled_amount, Σ refunded_amount;
    in_flight = non-terminal executions of the account created before day+1; held_open = those in 'held'
    complete = now ≥ day+1 + 1 h and every tracker cursor (near-indexer, evm:<chain>, explorer) is past day+1
               and in_flight − held_open = 0 for executions whose deadline < day+1
    tier_b = Tier-B settlements of the day; explorer_confirmed = those whose explorer_status = 'SUCCESS'
             (unconfirmed listed by id; empty with the Explorer feed off)
    digest = sha256 over "<ex_id>:<version>\n" of the day's executions in id order
GET /internal/evidence?states=succeeded,refunded,failed&kind=… → the executions themselves (paged) for line-by-line diffs
```
Payments reconciles only `complete` days (retrying later otherwise) and stores the digest with its run; a later different digest for a closed day means late evidence and re-runs the day. Holds are not diffs: they are open cases (console Verifier holds).

### 8.11 Observation archive (Stage 2)
Daily job `observations.archive` (feeds, 03:00 UTC, after reconciliation): observations of executions with `terminal_at` older than `NP_OBS_ARCHIVE_AFTER_DAYS` (90) are written per UTC day of `observed_at` to Parquet (`id, execution_id, source, provider_status, provider_updated_at, applied, observed_at, payload_json, payload_sha256`) under `<prefix>/rails/observations/<live|test>/<yyyy>/<mm>/<dd>/<batch>.parquet`; the file is read back and every row's payload hash re-checked before, in one transaction per batch, `payload` is set to null and `archive_key`/`archive_row`/`archived_at` are set (the hash stays forever). Store: `file` (a Compose volume, local stage) or `s3` (EU object storage, server stage); `NP_OBS_ARCHIVE_STORE` unset → the job does nothing. `GET /internal/executions/{id}/observations` returns archived rows with `payload: null` and `archive {object_key, row, archived_at}`; ops fetch the file from the store.

## 9. Jobs and loops

| Loop/job | Where | Cadence | Failure |
|---|---|---|---|
| token & price refresh | feeds (leader) | `/v0/tokens` every 30 s (1Click refreshes prices ≈ 1/min; the par policy accepts a primary price ≤ 5 min old, the dual-source rule ≤ 120 s primary / ≤ 300 s secondary, §8.2) | stale → pricing_unavailable for affected non-par assets |
| secondary prices (Stage 2) | feeds (leader) | CoinGecko batch every 60 s; 429 → back off ×2 up to 10 min | stale → non-par assets unavailable; par unaffected |
| FX refresh (Stage 2) | feeds (leader) | ECB daily XML every 30 min (conditional GET); test mode loads `NP_TEST_FX_RATES` at boot | rate past `valid_until` → currency unavailable + alert `fx_stale` (§8.2) |
| `tags.release_orphans` | feeds (job) | every 5 min | releases tags of creates that crashed before the execution row existed |
| `kit.purge` | feeds (job) | hourly | published outbox, old inbox, expired idempotency keys |
| hot poller | tracker | 500 ms tick | breaker on provider; executions keep `next_poll_at` |
| Explorer feed | feeds (leader) | 5 s | back off on 429 |
| NEAR indexer | feeds (leader) | continuous | switch source on gap/stall > 30 s |
| sandbox feed (test mode) | feeds (leader) | 1 s poll of sandbox `/sim/intents/events` and sandbox status for test executions | same matching code as the NEAR indexer |
| EVM watchers | feeds (leader per chain) | every 2 s (L2) / 12 s (L1) | RPC pool failover |
| verifier | tracker | event-driven + 10 s retries | → held after timeout |
| minimums | feeds | hourly | keep last value |
| `observations.archive` (Stage 2) | feeds (job) | daily 03:00 UTC: observations of executions terminal > 90 days → Parquet in the archive store (file locally, S3 on the server), keep hash (§8.11) | retry; a batch is nulled only after read-back verification |

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
| CoinGecko keyless `/api/v3/simple/price` (secondary prices, Stage 2) | 3 s | 1 call / 60 s (keyless limits undocumented, ⚠ landscape §8) | breaker; stale → non-par unavailable |
| ECB `eurofxref-daily.xml` (FX, Stage 2) | 5 s | 1 call / 30 min | keep last rate until `valid_until` |
| Explorer `/api/v0/transactions` in test mode → sandbox `/api/v0/transactions` | 2 s | — | same client |
| compliance screening | 1.5 s | — | policy above |

## 11. Configuration

`NP_ONECLICK_BASE_URL`, `NP_ONECLICK_API_KEY_FILE` (live only; absent in Stage 1 local → unauthenticated quotes with +25 bps, appFees still sent), `NP_TEST_TREASURY_ACCOUNT` (test-mode fee recipient, sandbox only), `NP_PROVIDER_SPLIT` (1Click appFees split, §8.1), `NP_AWAITING_SWEEP_S` (30), `NP_SCREENING_FAIL_CLOSED_USD` (1000), `NP_AGREEMENT_USD` (EVM pool agreement threshold), `NP_IDENTITY_URL` + `NP_SERVICE_CLIENT_SECRET_FILE` (service token: `compliance.screen`, `sandbox.use`), `NP_COMPLIANCE_URL`, `NP_VALKEY_URL` (shared 1Click budgets; down → half per replica), `NP_SANDBOX_BASE_URL`, `NP_SANDBOX_SIGNING_PUBKEY`, `NP_EXPLORER_TOKEN_FILE`, `NP_TREASURY_ACCOUNT` (e.g. `treasury.nearpay.near`, provided by the owner; unset → live quoting refused), `NP_NEAR_RPC_URLS`, `NP_NEARDATA_URL`, `NP_EVM_RPC_URLS_<CHAINID>`, `NP_PRICE_SECONDARY_URL` (default `https://api.coingecko.com/api/v3`; keyless — `NP_PRICE_SECONDARY_KEY_FILE` is not used in v1), `NP_PRICE_SECONDARY_IDS` (json `{asset_id: coingecko id}` overrides), `NP_PRICE_PRIMARY_MAX_AGE_S` (120), `NP_PRICE_SECONDARY_MAX_AGE_S` (300), `NP_FX_URL` (default the ECB daily XML), `NP_FX_GRACE_S` (7200), `NP_FX_MAX_AGE_S` (518400), `NP_TEST_FX_RATES` (json `{currency: "<USD per unit, decimal string>"}`), `NP_STABLE_PEGS` (json `{asset_id: currency}` for non-USD stablecoins, e.g. EURC → `eur`), `NP_OBS_ARCHIVE_STORE` (`file:<dir>` or `s3://<bucket>/<prefix>`; unset → no archive), `NP_OBS_ARCHIVE_AFTER_DAYS` (90), `NP_SLIPPAGE_BPS` (json), `NP_CHAINS_FILE`, `NP_ONECLICK_QUOTE_RPS`, `NP_ONECLICK_STATUS_RPS`, `NP_EXPLORER_ENABLED` (default: true when `NP_EXPLORER_TOKEN_FILE` exists), `NP_PRICING_CURRENCIES` (Stage 1: `usd`; Stage 2 default `usd,eur,gbp,chf,jpy,cad,aud`), `NP_TIER_B_LIMITS` (`{per_payment_usd: 2000, per_day_usd: 20000}`).

**Stage 1 pricing posture:** without a secondary price source configured, non-par settlement assets are disabled (`pricing_unavailable`) — Stage 1 demos settle in USD stablecoins; fiat pricing currency is `usd` only (FX lands in Stage 2).

**Stage 2 pricing posture:** non-par settlement (wNEAR, ETH, BTC, …) is enabled when both price sources are valid and agree (§8.2); non-USD fiat pricing for the currencies in `NP_PRICING_CURRENCIES` with ECB rates inside their staleness bound. Either source failing disables only what needs it (HLD §10).

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

- Unit: status mapping table; pricing (par incl. non-USD pegs, depeg, dual-source agreement and freshness, ECB cross rates, TARGET-calendar `valid_until` incl. weekends, Easter and the 6-day cap, stale rate refused, rounding down); route classification; tag allocation; deadline cap; binding derivation.
- Property: for any sequence of observations (duplicates, reordering, conflicting sources) the execution never leaves a terminal state (`succeeded`, `refunded`, `failed`); `expired_unpaid` only moves to `refunded`, `succeeded` (with evidence) or `held`, and `held` only to `succeeded` (with evidence) or `refunded`; settled ⇒ evidence present; `base + tag` unique among in-flight executions per destination.
- Contract: 1Click adapter against recorded fixtures (`test/fixtures/oneclick`), including `KNOWN_DEPOSIT_TX`, 404, 400, signature failure.
- Integration: indexer parsing on recorded mainnet blocks (fixtures from SP-14); EVM watcher against an Anvil fork with Transfer logs; verifier tiers.
- Load: hot poller at 1,000 in-flight executions within budget.

**Unified console audit trail (T-12, step 2.1 integration).** `reverifyOpsExecution` publishes `evt.rails.ops_action.recorded` (`OpsActionEvent`, ids only, via the outbox in the same transaction); identity records it in `audit_events`.

## 16. Stage mapping

| Stage | Scope |
|---|---|
| 1 | token registry + pricing (par + primary/secondary), `oneclick` + `sandbox` + `direct_intents` providers, executions, hot poller, Explorer feed, NEAR indexer (Intents + NEP-141), EVM watchers for Base/Arbitrum/Ethereum, verifiers NEAR + EVM, minimums, options, evidence API |
| 2 | refunds (`EXACT_INPUT`/`EXACT_OUTPUT` legs; route shapes owned by payments), secondary prices + ECB FX (non-par settlement, multi-currency pricing, §8.2), Explorer feed with the owner token (§8.5), reconciliation evidence (§8.10), observation archive (§8.11), console reads (§6) |
| 3 | executions owned by agentpay (cross-chain x402: `owner`, `x402_binding`, ownership by caller identity), `Evidence.sender`, `listEvidence?owner` (§20). No invoice/billing work in rails: invoices collect through payments' checkout sessions; `evt.rails.vault.event` stays Stage 4 |
| 4 | Solana + Bitcoin watchers/verifiers (§21), vault event forwarding hardening |

## 17. Ops CLI (Stage 1)

`docker compose -p nearpay exec rails-api /nodejs/bin/node dist/ops.js <command>`: `execution-show <ex_… | deposit address>` · `flags` · `flag-set quoting_enabled|cross_chain_enabled true|false <reason>` (RB-01) · `tokens-refresh live|test` · `minimums-refresh`.

## 18. Changes in version 1.3 (step 2.1 contracts, spec 0.2.0)

- **Pricing/FX (§8.2, ADR-0020):** ECB reference rates (fiat FX, cross rates via EUR as exact rationals) and CoinGecko keyless `simple/price` (secondary token price) behind `FxProvider`/`PriceProvider` ports. The 5-minute real-time FX with a 36 h ECB fallback is replaced by a **publication-calendar bound**: valid until the next TARGET working day's 16:00 CET publication + 2 h, capped at 6 days; past it the currency is unavailable (`fx_rate_unavailable`), never silently stale. Dual-source freshness: primary ≤ 120 s (was 60 s, too tight for 1Click's ≈ 1/min refresh), secondary ≤ 300 s. Multi-currency par via `peg_currency` (`NP_STABLE_PEGS`). Contract: `PriceRate`/`FxRate`/`Ratio`, `PriceSnapshot.rates[]`, `GET /internal/fx-rates`, `GET /internal/settlement-estimate`.
- **Explorer (§8.5):** owner token enables the feed; explicit `statuses`, cursor paging, change detection against `explorer_status`, sandbox mirror for test mode.
- **Reconciliation evidence (§8.10)** and **observation archive (§8.11)** specified; `GET /internal/evidence/summary`, `listEvidence` `states`/`kind`, `GET /internal/executions/{id}/observations`.
- **Ops (§6):** the console's executions explorer and Verifier holds are `/ops/executions…` routes on the identity-console `/ops/…` table, served by rails over `Execution`/`ProviderObservation`; a reverify action is requested there.
- **1Click (§3):** the spec accepts the live body (nullish optionals, status-nested quote without `correlationId`); the adapter's normalisation is no longer needed.
- **CR-15-06:** rails fills `token` (`TokenContract`) on options and assets from the registry's `contract_address` (`erc20` on EVM, `nep141` on NEAR, `native` without a contract).


## 19. Changes in version 1.4 (step 2.2 implementation)

### 19.1 Refund legs (§6 `POST /internal/refund_executions`)
- **Row mapping.** `origin_*` is the merchant side (`origin_asset_id` = source asset, `refund_to` = merchant destination, `refund_type` = `intents` for an Intents-account source), `destination_*` the payer side (`recipient` = payer refund address, Intents accounts lower-cased, `destination_key` = `chain|address_norm|asset`), `amount_tag = 0`, `applied_fee_bps = 0`, `fee_recipient` null. The verifier, the indexer's binding match and the Tier-B rules therefore apply unchanged to the payer-side transfer (INV-2). `reference` (jsonb) and `refund` (jsonb, the immutable `ExecutionRefund` snapshot, INV-6) are new columns (migration `0002_stage2.sql`; the LLD's "migration 0003" is `0002` in this repo, kit tables have no numbered file here).
- **Verification floor.** `EXACT_OUTPUT`: `amount_out_expected` = the converted amount (1Click must deliver it exactly — integrity). `EXACT_INPUT`: `amount_out_expected` = the quote's `minAmountOut` (the payer's guaranteed floor; `refund.amount_out_expected` shows the quote's `amountOut`); the floor is bounded by §8.1 step 10 (refund bounds), so it cannot sit more than the route slippage below a quote that is itself within the price band. Evidence below the floor is `short` (payments decides).
- **Integrity for `EXACT_INPUT`** (added to `checkQuote`): `amountIn` equals the requested amount, `minAmountOut` positive and ≤ `amountOut`. All Stage 1 checks still apply (echoed fields, `dry: false`, deposit address valid for the source, deadline, signature); appFees must be empty.
- **Intents-source route legs** carry a binding (`np1:` over execution id, deposit address, source asset, `amountIn`, no fee, deadline); the DB check on `binding` now allows `route_quote` + `kind = refund` + `refund_type = intents`. `refund.transfer` = `{signer_id: merchant account, receiver_id: deposit address, token, amount: amountIn, memo: binding, deadline}`; agentpay relays it. `intents_transfer` legs: `receiver_id` = payer account, `amount` = request amount, provider `direct_intents`; `intents_requirement` stays null for refunds (payments-only shape).
- **Screening** uses context `refund`. `screen: true` (owner override) makes an unavailable compliance fail closed at any amount; the cache cannot be bypassed yet (gap, below).
- **Kill switch / config:** live legs need `quoting_enabled` and `NP_TREASURY_ACCOUNT` like payments (fail closed); `cross_chain_enabled` gates route legs. An idempotency key owned by the other kind is `idempotency_key_reused`.
- **Sandbox:** quotes send `scenario_amount` (payments: pricing amount; refunds: the refund amount), `scenario_group` (`pi_…` / `re_…`) and `scenario_kind`.

### 19.2 Pricing (§8.2)
- **Test-mode FX** comes straight from `NP_TEST_FX_RATES` (no DB rows), so `rails-api` prices non-USD currencies in test mode without the feeds process; `fx_rates` holds live (ECB) rows only. An ECB answer older than the stored `as_of` never replaces it.
- **Test-mode secondary price** = the sandbox registry price (`secondary_source = sandbox`), written by the token refresh, so non-par settlement works offline.
- **Asset pricing at par:** an asset-priced payment whose pricing asset is a stablecoin passing the depeg guard is valued at `fx(peg)` (USD 1 for USDC), not through the dual-source rule (keeps Stage 1 behaviour for USDC-priced payments into USDT). A non-USD-peg settlement asset (EURC) priced in USD uses the dual-source rule, as specified.
- 1Click `/v0/tokens` is read as text so `price` keeps its source digits (`parseJsonDecimals`); exact ratios in `price_num/den`, display numerics in `price_usd`.
- `currency_unsupported` is checked before par (a currency outside `NP_PRICING_CURRENCIES` is refused even for a same-currency stablecoin).

### 19.3 Reconciliation evidence (§8.10)
- `in_flight` excludes `expired_unpaid` executions whose 24 h tracking window has closed (they never become terminal and would block `complete` forever).
- Cursor rule implemented as: every existing tracker cursor (`near-indexer`, `evm:*`, and `explorer` when the feed runs) has `updated_at` ≥ the day's end; an absent cursor means an idle demand-driven tracker (nothing in flight on its chain). Test-mode summaries ignore cursors (the sandbox feed is not a chain tracker).
- `totals.asset_id` is `destination_asset_id` for both kinds (the payer-side asset for refund legs).
- `tier_b.explorer_confirmed` / `unconfirmed` are `0` / `[]` while the Explorer feed is off; `count` is always reported.

### 19.4 Observation archive (§8.11)
- Batches of ≤ 2,000 rows per (mode, UTC day); `payload_json` is the stored `jsonb` text. `payload_sha256` stays the hash of the payload **as received** (`JSON.stringify` before `jsonb` normalisation), so the read-back check compares the file's `payload_json` bytes with the database text and the stored hash with the file's hash (it does not re-hash `payload_json` against `payload_sha256`).
- `s3://` stores sign path-style requests with SigV4 (`NP_OBS_ARCHIVE_S3_ENDPOINT`, `_REGION`, `_ACCESS_KEY_ID_FILE`, `_SECRET_ACCESS_KEY_FILE`); no SDK.

### 19.5 Explorer feed (§8.5)
- Window (a) also asks for `PENDING_DEPOSIT` (`PENDING_DEPOSIT,INCOMPLETE_DEPOSIT,PROCESSING`). Terminal executions receive `explorer_status` (Tier-B cross-check) without a nudge. Test mode keeps its own cursor `explorer:test` (not a reconciliation cursor) and idles 60 s if the sandbox build has no mirror (404). `NP_EXPLORER_ENABLED` defaults to on when the token file exists.

### 19.6 Console
- `reverify` records an `operator` observation (`{action, action_id, operator, ops_role, note}`), sets `verify_at` (scheduling only: no version bump, no execution event) and emits `evt.rails.ops_action.recorded` with action `execution.reverify_requested`, target `execution`; the envelope aggregate id is `aud_<action uuid>` (as sentinel; the outbox keys aggregates by prefixed id).
- `listOpsExecutions` pages by id (UUIDv7 = creation order), newest first; `ending_before` supported.

### 19.7 Contract gaps (change requests, not applied)
- **CR-22-RL-01** `createScreening` body: optional `fresh: boolean` ("bypass the screening cache") so rails can honour `CreateRefundExecution.screen` exactly (today an owner override re-screens through the cache).
- **CR-22-RL-02** `createRefundExecution.errors` lacks `fx_rate_unavailable` and `currency_unsupported`; an `EXACT_OUTPUT` leg converting through a non-USD-peg stablecoin can hit `fx_rate_unavailable`, which rails returns as is (the kit does not restrict codes to the declared list) — declare both codes for parity with `createExecution`.
- **CR-22-RL-03** `ExecutionRefund` has no field for the binding of an `intents_transfer` leg other than `transfer.memo`; fine for agentpay, noted only.

## Spec 0.2.1 contract changes (step 2.2/2.3 change requests, `nearpay-spec/CHANGELOG.md`)

- **`abandonExecution` `POST /internal/executions/{id}/abandon`** (`service:rails.executions.write`; caller payments, body `{reason: leg_failed|execution_mismatch|superseded|refund_canceled, note?}` → `Execution`). Only `kind: refund` in `quoting`/`awaiting_deposit`/`expired_unpaid`; sets `abandoned_at`/`abandon_reason`, moves a pre-deposit leg to `expired_unpaid` at once (deadline = now, `tracking_until` = now + 24 h — late money still settles, INV-3), outbox `evt.rails.execution.updated`. Repeats return the execution unchanged; money moved / payment executions / terminal → `409 state_conflict`.
- **`Execution.observed_fee {amount, asset_id}`** (nullable): route legs — the app fee 1Click/Explorer reports for the execution; `intents_transfer` — the executed payload's fee transfer; refund legs null. Payments' reconciliation compares it with statement fees (`fee_mismatch`).
- **`Execution.deposited_amount`**: provider `swapDetails.depositedAmount` (route) or the executed Intents transfer incl. fee; null until a deposit is detected.
- **`decimals`** on `Execution.origin` / `.settlement` (registry of the execution's mode).
- **`ExecutionDeposit.fee_amount_in`**: NearPay's share of `amount_in` in the origin asset when the payer bears the fee (`0` when the merchant does; null when it cannot be apportioned) — display only.
- **`RefundTransfer.nonce`**: rails issues the Verifier nonce (current salt ‖ 28 random bytes) when it prepares an Intents-source leg, so payments drops its NEAR RPC salt read.
- **`createScreening.fresh`** (CR-22-RL-01): send `fresh: true` for an owner-overridden refund recipient (`CreateRefundExecution.screen`).
- `createRefundExecution.errors` now declare `fx_rate_unavailable`, `currency_unsupported` (CR-22-RL-02; behaviour unchanged).
- Reads `getKillSwitch` at boot (`sentinel.read`) to seed `control_flags` with the switch version when no event was applied yet.
- **Implementation (v1.6, migration `0003`).**
  - *Observed fee* (payments only): route legs — on provider `SUCCESS`, `bpsOf(swapDetails.amountIn, Σ appFees bps)` from the status' echoed `quoteResponse.quoteRequest.appFees` (the treasury entry + 1Click's split = the applied fee), origin asset; the Explorer feed fills it the same way from a `SUCCESS` row's `appFees`/`amountIn` when still unknown (through the ledger: version + `execution.updated`). Same rule as payments' statement estimate (`bpsOf(amount_in, applied_fee_bps)`), so `fee_mismatch` compares like with like. `intents_transfer` — Σ of the DIP-4 transfers in the settling tx with the same binding to `fee_recipient` (settlement asset; `0` when none). Unknown stays null (never guessed).
  - *Deposited amount*: route — `swapDetails.depositedAmount` when > 0 on any status; `intents_transfer` — merchant transfer + fee transfer of the settling tx.
  - *`fee_amount_in`*: `bpsOf(amount_in, applied_fee_bps)` when the payer bears the fee, `0` when the merchant does and on refund legs; stored at quote time.
  - *Decimals*: `origin_decimals`/`destination_decimals` stored at creation from the mode's registry (existing rows backfilled from `tokens`); omitted when unknown.
  - *Abandon*: allowed only for `kind: refund` in `quoting`/`awaiting_deposit`/`expired_unpaid` with no deposit tx, no deposited amount and no provider status past `PENDING_DEPOSIT`; `quoting → expired_unpaid` added to the graph for it. Sets `deadline = min(deadline, now)`, `tracking_until = now + 24 h`, polling every 15 min for route legs (direct legs rely on the indexer); the note stays in rails (`abandon_note`). A leg already abandoned replays unchanged whatever its state now.
  - *Kill switch*: `control_flags.known` (false on a fresh database) — an unknown `quoting_enabled` refuses live payment quotes, live refund legs and live options (`rail_unavailable`, "kill switch state unknown"). `NP_SENTINEL_URL` (unset → waits for an event). The api and feeds processes read at boot and retry every 10 s while unknown; a quote path also reads (throttled 10 s). A read applies only while unknown or when newer; events apply when newer or while unknown.
  - *Nonce*: issued before the insert transaction for Intents-source legs (live: `current_salt` over `NP_NEAR_RPC_URLS`, cached 60 s; test: the sandbox Verifier's fixed salt `5a0dbe11`); an unreadable live salt gives `nonce: null` (payments falls back to its own read) rather than refusing the leg.
  - *Fresh screening*: `fresh: true` only when `screen: true` (owner override).

## 20. Stage 3 contracts (step 3.1, spec 0.3.0): executions owned by agentpay

Cross-chain x402 (`exact` + `near-intents`, agentpay §3.1) quotes through rails like a checkout attempt, but no payments object exists.

- `CreateExecution.owner = agentpay` (default `payments`) with `x402_binding`, `idempotency_key = x402:<binding>`, `kind: payment`, `method: route_quote`, destination `intents_account` (the merchant's Intents account), `pricing: {currency: asset, asset_id, amount}`, fee bearer `payer`, `deadline_cap` = now + the requirement's `maxTimeoutSeconds`, `slippage_bps` as clamped by agentpay; no `reference`. Caller scope `rails.executions.write` (granted to agentpay in spec 0.3.0). §8.1 applies unchanged: kill switch, `refund_to` screening (context `payment`), appFees integrity, quote signature, tag allocation (the merchant may receive `amount + tag`; agentpay reports the settled amount).
- `executions.owner` already exists (§4); the view now carries `owner` and `x402_binding` (`Execution.owner`, `Execution.x402_binding`, optional on the wire). Events are unchanged in subject; **payments acks and ignores executions whose `owner` is not `payments`** (Stage 3 payments change), agentpay consumes `settled`/`refunded`/`failed` of its own.
- `Evidence.sender` (optional, nullable): the account that sent the settlement transfer as the detector saw it. Rails MUST set it for Intents-recipient evidence (NEAR indexer: the NEP-245 `mt_transfer` `old_owner_id`; verifier: the transfer it checked) — agentpay records it as the agent payment's `payer`. Other detectors may leave it null.
- `listEvidence` query `owner` (default: all). Payments' reconciliation sends `owner=payments` so agentpay-owned executions never show up as unmatched payments.
- `POST /internal/executions/{id}/deposit_tx` is also called by agentpay (deposit `txHash` from the x402 `deposit` payload; best effort, idempotent).

### 20.1 Implementation (step 3.2, v1.8)

- **Ownership by caller identity** (`domain/execution/owner.ts`, pure; enforced in `api/routes.ts` + `app/access.ts`). The caller is the verified internal token: `ptype=service`, `sub` = service name. Every rule fails closed:
  - *Create* (`createExecution`): the effective owner (`owner ?? payments`) must equal the calling service, else `403 permission_denied` (agentpay omitting `owner`, payments sending `owner: agentpay`, any other service). The owner is decided before the idempotency replay; a replay whose stored owner differs is `idempotency_key_reused`. `owner: agentpay` must have exactly the §20 shape, else `400 invalid_request` naming the field: `x402_binding` set, `idempotency_key = x402:<x402_binding>`, no `reference`, `method: route_quote`, `destination.type: intents_account`, `pricing = {currency: asset, asset_id = destination.asset_id}`, `fee.bearer: payer`. `owner: payments` may not send `x402_binding` nor use the `x402:` key namespace. Sandbox scenario group of an agentpay quote = its binding.
  - *Refund legs*: `createRefundExecution` and `abandonExecution` are payments' alone (`403` for agentpay, which also holds `rails.executions.write` since 0.3.0).
  - *Writes* `deposit_tx` / `signals`: only the owning service; anyone else gets `404 resource_missing` (no existence leak).
  - *Reads* `getExecution`, `listExecutionObservations`: agentpay reads its own executions **and the payments executions it relays signed intents for** — `intents_transfer` payments and refund legs carrying `refund.transfer` (agentpay §5 `/internal/relay` validation reads them; "own only" would break Stage 1/2 relays). Any other payments execution is `404` to agentpay. Other services and console operators read every execution (unchanged). `lookupExecutionByDeposit` from agentpay only finds agentpay deposit addresses.
  - *Lists*: `listEvidence` from agentpay is narrowed to `owner=agentpay` (`owner=payments` → `403`); other callers get the filter they ask for (default all). `getEvidenceSummary` (spec 0.3.1 `?owner=`) follows the same rule: agentpay is narrowed to `owner=agentpay` (`owner=payments` → `403`); the owner filters `totals`, `tier_b`, `digest`, `in_flight` and `held_open`, while `complete` stays account-wide (its open-before-end count and cursors cover every owner). Payments' reconciliation sends `owner=payments`.
- **Events**: the outbox payload is the execution view, so every `evt.rails.execution.*` carries `owner` (always set) and `x402_binding` (null unless agentpay).
- **`Evidence.sender`** (stored in the `evidence` jsonb, no migration): NEAR indexer and sandbox feed — DIP-4 `transfer.account_id` for binding credits, NEP-245 `mt_transfer.old_owner_id` and NEP-141 `ft_transfer.old_owner_id` for amount credits; EVM watcher — the ERC-20 `Transfer` `from`; verifier — EVM ERC-20 `from` / native `tx.from`, NEAR `old_owner_id`, each the sender of the **largest counted transfer** to the recipient (first on a tie; the amount rule is unchanged). Tier B (provider-attested) evidence: `null` (no chain view). Sandbox verifier: `SimTx.transfers[].from` (spec 0.3.1; sandbox §4.1 — the simulated liquidity provider for payouts), `null` when absent. A sender the event does not name is `null`, never guessed. On mainnet a 1Click delivery's sender is the delivering solver/1Click account, not the agent's origin wallet — agentpay's `payer` is therefore "the Intents account that delivered" (as §20 says).
- **Migration `0004_stage3_owner.sql`** (additive): `x402_binding text`, check `(owner = 'agentpay') = (x402_binding is not null)`, unique partial index on `x402_binding`, index `(account_id, livemode, owner, terminal_at)` for `listEvidence?owner`. `owner` exists since `0001` (not null, default `payments`): the backfill is a no-op.
- **Gap — evidence summary has no owner — resolved in spec 0.3.1 (v1.9).** `getEvidenceSummary?owner=` implemented as above; payments sends `owner=payments` and dropped its unfiltered relisting (payments §21.8).

## 21. Stage 4 — Solana and Bitcoin as Tier A chains (step 4.1, spec 0.4.0)

Contracts: `nearpay-spec` `src/common/chains.ts`, `Evidence.solana` / `Evidence.bitcoin`, `ChainProfile.explorer_address_url` / `confirmation_tiers`, sandbox `SimTx` fields; vectors `test-vectors/chains-solana-bitcoin.json` (profiles, assets, address rules, omft ids, evidence parsing — rails and sentinel test their adapters against the same file).

### 21.1 Chain profiles and assets

| | Solana | Bitcoin |
|---|---|---|
| `chain` (CAIP-2) | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | `bip122:000000000019d6689c085ae165831e93` |
| `blockchain` (1Click) / `family` | `sol` / `solana` | `btc` / `bitcoin` |
| `tier` | A | A |
| `min_deadline_s` / `default_deadline_s` / `verification_timeout_s` | 300 / 1800 / 1800 | 3600 / 7200 / 7200 |
| finality | `finalized` commitment (≈ 32 slots, ~13 s) | `confirmation_tiers` = `BITCOIN_CONFIRMATION_TIERS`: 1 confirmation below USD 1,000, else 2 (USD value from the execution's `price_snapshot`) |
| `memo_mode` | `none` | `none` |
| explorer | `https://solscan.io/tx/{tx}`, `https://solscan.io/account/{address}` | `https://mempool.space/tx/{tx}`, `https://mempool.space/address/{address}` |
| destination types | `chain_address` only | `chain_address` only (P2WPKH / P2TR, identity §8.2) |
| `Evidence.finality` | `finalized` | `confirmed` |

Settlement assets enabled with the profiles (asset ids from the live `/v0/tokens`, recorded 2026-11-06; all `nep141` omft ids — none of these assets has a `nep245` form):

| Asset id | Symbol | Decimals | Token (`TokenContract`) |
|---|---|---|---|
| `nep141:sol.omft.near` | SOL | 9 | `native` |
| `nep141:sol-5ce3bf3a31af18be40ba30f721101b4341690186.omft.near` | USDC | 6 | `spl` `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` (Token program) |
| `nep141:sol-c800a4bd850783ccb82c2b2c7e84175443606352.omft.near` | USDT | 6 | `spl` `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` (Token program) |
| `nep141:btc.omft.near` | BTC | 8 | `native` |

- **SPL asset-id rule (verified against all 12 `sol-` omft entries of the live registry):** `nep141:sol-<hex(keccak256(mint bytes)[12..32])>.omft.near`. The token registry rejects a `sol-` entry whose `contractAddress` does not reproduce its id (registry entry dropped + alert), so `TokenContract.address` for SPL is bound to the asset id like EVM contracts are (`parseOmftAssetId`, kit `chains` `splAssetId`).
- **`1cs_v1:` ids.** The live registry now also lists `1cs_v1:<chain>:<kind>:<id>` asset ids (27 on 2026-11-06, e.g. `1cs_v1:btc:native:coin`, `1cs_v1:sol:spl:<mint>`). They are not NEAR Intents asset ids (`AssetId` rejects them); the registry ignores them (counted, not stored) until a spec version adopts them.
- **Amount tags (§8.3).** USDC/USDT (6 decimals) and SOL (9 decimals at ≈ USD 150) are tagged as usual. BTC is **never tagged**: 1 sat ≈ USD 0.0008 gives `T = floor(0.01 / unit_usd) < 100`, so §8.3 sets tag 0; Bitcoin evidence therefore comes from the verifier on the provider's destination txid (§21.3), never from amount matching.
- Minimums (§8.9) apply unchanged; Bitcoin dust makes the `20 × withdrawFee` floor dominate.

### 21.2 Watchers (`rails-feeds`, demand-driven like §8.6/§8.7)

```
solana (every 4 s, only while live executions with a Solana destination are in flight):
  for each watched destination owner with tagged in-flight executions:
    accounts = mint ≠ null ? [ATA(owner, mint)] : [owner]           // ATA = findProgramAddress([owner, Token, mint], ATA program)
    sigs = getSignaturesForAddress(account, {until: cursor[account], commitment: 'finalized', limit: 100})
    for each sig (oldest first): tx = getTransaction(sig, {commitment:'finalized', maxSupportedTransactionVersion:0, encoding:'jsonParsed'})
      skip if tx.meta.err ≠ null
      delta = Σ postTokenBalances − Σ preTokenBalances where owner = destination and mint = asset mint  (missing pre entry = 0)
              (SOL: postBalances[i] − preBalances[i] at the destination's index in static keys + loadedAddresses)
      exec = tagged in-flight execution with destination_key = chain|owner|asset and amount_out_expected == delta
        → exact single match → evidence(pass, source 'watcher', Evidence.solana filled) → succeeded
        → none → 'unattributed_transfer'
    cursor[account] = newest processed sig (per address; forgotten when nothing is in flight, like §8.6)
bitcoin (every 60 s, signal only): for each destination with in-flight executions: Esplora GET /address/{addr}/txs
    a new tx paying the address → has_signal = true, next_poll_at = now() (the provider status then carries the
    destination txid for the verifier). Never evidence by itself: BTC is untagged (§21.1).
```
Tracker cursors `solana` and `bitcoin` join the §8.10 completeness rule (each is "past the day" when its last cycle finished after `day+1` with no error).

### 21.3 Verifiers (§8.8 adapters) and evidence shapes

- **Solana:** for each `destinationChainTxHashes` signature: `getTransaction` as above; not found or not yet `finalized` → retry every 10 s until `verification_timeout_s`; `meta.err ≠ null` → that signature proves nothing (another hash or timeout → `held(evidence_timeout)`). Amount = the owner's balance delta for the asset (token-balance delta for SPL, lamports for SOL), so a payout that also creates the ATA counts in full. `Evidence`: `tx_hash` = signature, `block_ref` = slot, `recipient` = owner, `finality: finalized`, `solana {signature, slot, block_time, mint, token_program, owner, token_accounts, pre_amount, post_amount}`. Only the classic Token program is enabled in v1 (a Token-2022 mint is not in the asset table).
- **Bitcoin:** Esplora `GET /tx/{txid}` + `GET /blocks/tip/height`: amount = Σ outputs whose scriptPubKey encodes the destination (compare re-derived lower-case address); `confirmations = tip − block_height + 1`; below `requiredConfirmations(confirmation_tiers, usd_value)` → not final, retry every 10 s until `verification_timeout_s`. `Evidence`: `tx_hash` = txid, `block_ref` = height, `finality: confirmed`, `bitcoin {txid, outputs[{vout, value, script_pubkey}], block_height, block_hash, confirmations, required_confirmations}`.
- Comparison as §8.8 (`pass` / `short` / `mismatch`). Agreement: at or above `NP_AGREEMENT_USD` two pool members must return the same amount and block (Solana slot / Bitcoin block hash), as for EVM.
- **Sender:** `Evidence.sender` = the solver's address when visible (Solana: the token-balance owner whose balance fell; Bitcoin: null — inputs are not attributed).

### 21.4 Integrations and configuration

| Target | Timeout | Notes |
|---|---|---|
| Solana RPC pool `NP_SOLANA_RPC_URLS` (≥ 2; default `https://api.mainnet-beta.solana.com` + an owner-provided provider URL) | 3 s | hedged reads; `getSignaturesForAddress`, `getTransaction`; 429 → back off ×2 |
| Bitcoin Esplora pool `NP_BITCOIN_ESPLORA_URLS` (default `https://mempool.space/api,https://blockstream.info/api`; keyless) | 5 s | `/tx/{txid}`, `/blocks/tip/height`, `/address/{addr}/txs` |

Both chains stay **Tier B** (§4 limits, provider-attested) when their pool is unset or unhealthy for > 5 min: the profile's `tier` is computed at boot from configuration and an alert `tier_a_degraded{chain}` fires; executions already created keep the tier they were created with.

### 21.5 Test mode

Test executions on these chains settle through the sandbox exactly like EVM: the sandbox verifier adapter reads `GET /sim/chains/{caip2}/tx/{hash}` and fills `Evidence.solana` / `Evidence.bitcoin` from `SimTx` (`block_ref` → slot/height, `transfers[].token_account`, `transfers[].vout`, `confirmations`; `pre_amount`/`post_amount` null; `block_hash` = sha256 hex of `<chain>|<block_ref>`), applying the same confirmation tiers and the `failed` flag (sandbox §10). Watchers do not run in test mode.

