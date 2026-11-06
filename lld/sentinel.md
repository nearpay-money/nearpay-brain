# LLD — `nearpay-sentinel`

> Version 1.3 (step 1.2 implementation findings folded: data §4, R4/R5/R6 and classes §5, audit §8.1, cache §8, configuration §10, ops CLI §15) · Version 1.1 audited (`audits/sentinel.md`) · HLD §6 (two-party release), ADR-0010, ADR-0012, security T-03 · Foundations `00-foundations.md`

## 1. Scope

**Owns:** co-signing of deposit instructions and Intents-transfer requirements shown by checkout; independent re-verification (audit) of settled payments; the global quoting kill switch.
**Trusts only:** 1Click (or sandbox) status responses verified with the provider signature, identity's destination registry (read-only view), its own chain RPC credentials. It does **not** call payments or rails to decide anything.

## 2. Runtime

| Compose service | Command | Replicas | Notes |
|---|---|---|---|
| `sentinel-api` | `api` | 2 | `POST /v1/attestations/deposit_instructions` (anonymous route via gateway, per-IP limit 10 rps) |
| `sentinel-auditor` | `worker` | 1 | consumes `evt.rails.execution.settled`; kill-switch publisher; outbox relay |

Separate image pipeline, separate secret files (`sentinel_signing_key`, `sentinel_oneclick_api_key`, `sentinel_rpc_*`), separate DB roles: `sentinel_app` (own DB) and `sentinel_ro` (SELECT on `identity.sentinel_destination_registry` only).

## 3. Code layout

```
src/co-signer/        request validation, provider lookup, registry match, canonical JSON (RFC 8785), Ed25519 signing
src/address-integrity/ checks shared by co-signer and auditor
src/independent-reconciliation/ auditor: chain adapters (own RPC clients), sampling, findings
src/kill-switch/      engage/release, outbox events
```

## 4. Data

```sql
create table attestations (
  id uuid primary key, livemode boolean not null, kind text not null check (kind in ('route_quote','intents_transfer')),
  deposit_key text not null,                 -- provider|address|memo  or  binding
  attempt_id text not null, account_id uuid, destination_id uuid,
  result text not null check (result in ('signed','rejected','unavailable')),
  class text,                                -- security|ambiguous|stale|unavailable (rejections)
  reasons text[] not null default '{}', response jsonb, client_ip text,
  expires_at timestamptz, created_at timestamptz not null default now());
create index att_key on attestations (deposit_key, created_at desc);
create index att_destination on attestations (destination_id) where result = 'signed';

create table audit_findings (                -- one row per (execution, check)
  id uuid primary key, execution_id text not null, account_id uuid not null, livemode boolean not null,
  check_kind text not null check (check_kind in ('destination','settlement')),
  result text not null check (result in ('pass','mismatch','unverifiable')), details jsonb not null,
  tx_chain text, tx_hash text, recipient text, asset_id text, amount numeric(78,0),   -- double-use attribution (§8.1)
  created_at timestamptz not null default now(), acknowledged_at timestamptz, acknowledged_by text,
  unique (execution_id, check_kind));
create index findings_open on audit_findings (created_at) where result <> 'pass' and acknowledged_at is null;
create index findings_tx on audit_findings (tx_chain, tx_hash) where result = 'pass' and tx_hash is not null;

create table kill_switch (id smallint primary key check (id = 1), engaged boolean not null, reason text not null,
  actor text not null, version integer not null, updated_at timestamptz not null default now());   -- singleton (`ks_` id, spec 0.1.4); version orders events
create table kill_switch_log (id uuid primary key, engaged boolean not null, reason text not null, actor text not null,
  version integer not null, created_at timestamptz not null default now());   -- append-only (trigger)
```

## 5. Co-signing rules

Request (from checkout): `{livemode, kind, attempt_id, attestation_ticket, deposit?: {address, memo, chain, origin_asset_id, amount_in, deadline}, requirement?: {pay_to, token, amount, fee_to, fee_amount, binding, deadline}}`.

**Gate (before any provider call):** `attestation_ticket` must verify with the payments ticket public key, be unexpired, and match `attempt_id`, `livemode` and the deposit key/binding; otherwise `401 ticket_invalid` (no provider call, no kill switch). The ticket only rate-gates access; it is never evidence of correctness.

**`route_quote` checks (all must pass):**

| # | Check | Failure class |
|---|---|---|
| R1 | `GET /v0/status?depositAddress&depositMemo` (live: 1Click with sentinel's own key; test: sandbox) returns 200 | `unavailable` (503) |
| R2 | `verifyQuoteSignature(status.quoteResponse)` (live: SDK; test: sandbox key) | `security` |
| R3 | `quoteRequest.referral == 'nearpay'` | `security` |
| R4 | `quote.depositAddress == deposit.address`, memo equal, `quoteRequest.originAsset == deposit.origin_asset_id`, `quote.amountIn == deposit.amount_in`, `quote.deadline` and `deposit.deadline` equal as instants, and `deposit.chain` is the provider's deposit chain: `INTENTS` deposit → `near:mainnet`; otherwise the origin asset's chain when derivable (omft bridge prefix, or NEP-141 → NEAR). An EVM address is valid on every EVM chain, so the chain must be checked, not only the address | `security` |
| R5 | registry rows (identity view v2) with `address_norm == norm(quoteRequest.recipient)`, `asset_id == quoteRequest.destinationAsset`, matching `type`/`livemode`, `status = active`: exactly one → pass; several → prefer the single `signature`-verified row, else `ambiguous`; none active but a once-active **disabled** destination matches → `stale` (a merchant disabling a destination mid-checkout must not engage the global kill switch); none at all → `security` | `security` / `ambiguous` / `stale` |
| R6 | `appFees` (1Click echoes them split 50/50 and does not sign them, rails §8.1): at most 2 entries, total ≤ 500 bps, exactly one entry with `recipient == TREASURY` keeping `≥ total − max(ceil(total × share_bps / 10 000), max(floor_bps, stable_floor_bps))`; a fee-free quote (no entries) passes | `security` |
| R7 | `status ∈ {PENDING_DEPOSIT, KNOWN_DEPOSIT_TX, INCOMPLETE_DEPOSIT, PROCESSING}` and `deadline > now + 60 s` | `stale` (409) |

**`intents_transfer` checks:** registry row with `type='intents_account'`, `address_norm == pay_to`, `asset_id == token`, active, livemode; `fee_to == TREASURY`; `fee_amount ≤ ceil(amount × 500 / 10 000)`; `binding` matches `^np1:[1-9A-HJ-NP-Za-km-z]{43,44}$`; `deadline > now + 30 s`.

**Response (signed):**
```json
{"attestation":{"v":1,"kind":"route_quote","attempt_id":"pa_…","livemode":true,
  "deposit":{"address":"0x…","memo":null,"chain":"eip155:8453","origin_asset_id":"nep141:base-0x…omft.near","amount_in":"10012345","deadline":"…"},
  "merchant":{"name":"Acme GmbH","kyb_verified":true},
  "destination":{"asset_id":"nep141:…","chain":"near:mainnet","fingerprint":"7KQ2-M9XA-31"},
  "issued_at":"…","expires_at":"<= deposit.deadline"},
 "kid":"sent-2026-11","signature":"<base64url Ed25519 over RFC 8785 canonical JSON of attestation>"}
```
`fingerprint` = first 10 chars of Crockford base32 of `sha256(chain|address_norm|asset_id)`, grouped 4-4-2.

**Classes and reactions:**

| Class | Response | Kill switch |
|---|---|---|
| `security` | `409 attestation_refused` | **Only if** rails confirms NearPay issued this deposit address (`GET /internal/executions?deposit_address=` → `exists`) — a genuine integrity breach. If rails does not know it, the request is a spoof: refuse, count, rate-limit the IP, low-severity alert. If rails is unreachable: refuse + page (no automatic switch). |
| `ambiguous` | `409 attestation_refused` | never (attacker-inducible); case for ops |
| `stale` | `409 attestation_stale` | never |
| `unavailable` (provider or identity registry unreachable) | `503 attestation_unavailable` | never |

Spoofs (a `security` refusal for an address rails does not know) are counted per client IP (first `X-Forwarded-For` hop; per replica): 5 in 10 min → `429 rate_limited`. Without a live `NP_TREASURY_ACCOUNT` every live request is refused.

No attacker-controllable input can engage the global kill switch; a compromised rails can at most *suppress* the switch, while the address is still refused.

## 6. API

| Method & path | Auth | Response | Errors |
|---|---|---|---|
| `POST /v1/attestations/deposit_instructions` | anonymous (gateway per-IP limit) | `200` signed attestation (cached per `deposit_key` until `expires_at`) | `400 invalid_request`, `401 ticket_invalid`, `409 attestation_refused`, `409 attestation_stale`, `429 rate_limited`, `503 attestation_unavailable` |
| `GET /v1/attestations/keys` | anonymous | `{keys:[{kid, public_key, not_after}]}` (current key + `NP_SENTINEL_ANNOUNCED_KEYS`; informational; checkout and agent-signer use pinned keys and never fetch this) | |
| `POST /internal/kill-switch` | ops (`sentinel.killswitch`) | `{engaged, reason}` → state | |
| `GET /internal/findings` | ops | findings list (HTTP shape specified with the Stage 2 console, OQ-T-02; Stage 1 reads findings through the ops CLI) | |

## 7. Events

| Produced | Consumers |
|---|---|
| `evt.sentinel.kill_switch.engaged` · `.released` | rails (stores flag `quoting_enabled`), notify (page/email security contacts) |
| `evt.sentinel.audit_finding.created` (result ≠ pass) | compliance (opens case), notify |

| Consumed | Handler |
|---|---|
| `evt.rails.execution.settled` | sampling: 100% in Stage 1–2 and for amounts ≥ `NP_AUDIT_ALWAYS_USD` (10,000); 5% otherwise later → job `audit.execution` (§8.1). Below rate 1 the USD value comes from rails' price snapshot in the event and rails asset decimals (`GET /internal/assets`); used for sampling only, an unknown value is audited |
| `evt.identity.settlement_destination.disabled` | evict cached attestations for that destination |

## 8. Algorithms

- **Canonicalisation:** RFC 8785 JCS over the `attestation` object; Ed25519 over the UTF-8 bytes; `base64url` without padding.
- **Cache:** Valkey `att:<deposit_key>` (down → in-process per replica) → response + `destination_id` until `expires_at`; a cached `signed` response is re-served without re-querying the provider (idempotent for the SPA's retries), but every hit re-checks in the registry that the destination is still active (else `stale`); rejected/unavailable not cached.
- **Kill switch:** `engage(reason)` → insert log → outbox `evt.sentinel.kill_switch.engaged` → rails sets `control_flags.quoting_enabled=false` (consumed within seconds) → `rails` returns `rail_unavailable` for new live quotes. Release only by ops (two-person: request + approve in console) → `released` event. Every change updates the versioned `kill_switch` row, appends `kill_switch_log` and writes the outbox event in one transaction; consumers ignore events older than the version they applied.

### 8.1 Audit (`audit.execution` job)

Ten attempts (≈ 85 min); not final / not found / RPC unreachable retries; the last attempt records `unverifiable`. Two findings per execution:
- **`destination`:** the recipient was the account's destination **active when the execution was created** (±5 min clock skew), read from the registry view (which lists once-active disabled destinations with `disabled_at`).
- **`settlement`:** rails' settlement is self-consistent, then sentinel's own read — EVM: receipt at or below `finalized` with the ERC-20 `Transfer` on the contract derived from the asset id (or the native value); NEAR: FINAL tx with `mt_transfer` emitted by the Intents contract or NEP-141 `ft_transfer`; test mode: the sandbox's simulated chain tx; Tier B: the 1Click status signed with sentinel's own key — `SUCCESS`, `amountOut ≥ settled`, and the evidence tx among the provider's destination txs.
- **Double use:** a transaction is never attributed beyond what it moved: under an advisory lock per tx, the passing settlement rows of that `(tx_chain, tx_hash)` may not exceed the transferred amount (batches paying several executions are allowed).

Non-pass findings emit `evt.sentinel.audit_finding.created`.

## 9. Integrations

| Target | Timeout | Notes |
|---|---|---|
| 1Click `/v0/status` (own API key) | 2 s, 1 retry | separate credential from rails; budget 5 rps |
| sandbox `/v0/status` (test) | 1 s | |
| identity registry view (read-only replica/role) | 300 ms | |
| NEAR/EVM RPC (own provider accounts) | 3 s | auditor only |

## 10. Configuration

`NP_SENTINEL_SIGNING_KEY_FILE` (Ed25519 PKCS#8 PEM; the kid is the file name `sentinel-<kid>.pem`, else `local-` + 12 hex of the key's SHA-256), `NP_SENTINEL_KEY_NOT_AFTER` (default none → `9999-12-31`), `NP_SENTINEL_ANNOUNCED_KEYS` (rotation announcements for `GET /v1/attestations/keys`), `NP_PAYMENTS_TICKET_PUBKEY`, `NP_IDENTITY_URL` + `NP_SERVICE_CLIENT_SECRET_FILE` + `NP_RAILS_URL` (service token: `rails.read`, `sandbox.use`), `NP_TREASURY_ACCOUNT` (unset → every live request refused), `NP_TEST_TREASURY_ACCOUNT`, `NP_PROVIDER_SPLIT`, `NP_REFERRAL`, `NP_INTENTS_CONTRACT`, `NP_ONECLICK_BASE_URL`, `NP_ONECLICK_API_KEY_FILE`, `NP_SANDBOX_BASE_URL`, `NP_SANDBOX_SIGNING_PUBKEY`, `NP_REGISTRY_DB_URL_FILE` (sentinel_ro), `NP_VALKEY_URL`, `NP_NATS_URL`, `NP_AUDIT_SAMPLE_RATE` (1.0 in Stages 1–2), `NP_AUDIT_ALWAYS_USD` (10000), `NP_SENTINEL_NEAR_RPC_URLS_FILE` / `NP_SENTINEL_EVM_RPC_URLS_<chain id>_FILE` (own provider accounts; take precedence over the shared `NP_NEAR_RPC_URLS` / `NP_EVM_RPC_URLS_<chain id>` pools).

## 11. Security

- Fail-closed: no attestation → checkout renders no address (FR-PAY-10). Latency budget p99 1.5 s (dominated by one 1Click status call; cached responses ≤ 50 ms); 2 replicas on different CPU sets.
- Sentinel never reads payments/rails databases and never accepts their claims as truth.
- Key rotation: deploy new key with new `kid` → release checkout pinning both → switch signing → remove old after all open attestations expire.

## 12. Observability

`…_attestations_total{kind,result,reason}`, `…_attestation_duration_seconds`, `…_kill_switch_engaged` (gauge), `…_audit_findings_total{result}`. Alerts: any `security` rejection (page); unavailable > 1% / 5 min; audit mismatch (page).

## 13. Tests

- Each check R1–R7 and intents checks with a failing fixture → correct class; signature over JCS matches a vector shared with checkout tests (same fixture file in both repos via `@nearpay/spec` test vectors).
- Kill-switch flow end-to-end with rails in testbed.
- Property: any mutation of a signed attestation fails checkout verification.

## 14. Stage mapping

Stage 1: everything above except two-person kill-switch release (Stage 2 console) — Stage 1 release via ops CLI with audit log.

## 15. Ops CLI (Stage 1)

`docker compose -p nearpay exec sentinel-api /nodejs/bin/node dist/ops.js <command>`: `kill-switch` · `kill-switch-engage <reason>` · `kill-switch-release <reason>` (one operator, logged; the CLI has no `--actor`, so the testbed wrapper prefixes the operator's name to the reason) · `findings [--all] [--limit N]` · `finding-ack <fnd_…> <note>` · `attestations <pa_… | deposit key | deposit address>`.
