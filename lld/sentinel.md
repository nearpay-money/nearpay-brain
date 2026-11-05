# LLD — `nearpay-sentinel`

> Version 1.1 (audited, `audits/sentinel.md`) · HLD §6 (two-party release), ADR-0010, ADR-0012, security T-03 · Foundations `00-foundations.md`

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
  account_id uuid, result text not null check (result in ('signed','rejected','unavailable')),
  reasons text[] not null default '{}', response jsonb, expires_at timestamptz, created_at timestamptz not null default now());
create index att_key on attestations (deposit_key, created_at desc);

create table audit_findings (
  id uuid primary key, execution_id text not null, account_id uuid not null, livemode boolean not null,
  check_kind text not null, result text not null check (result in ('pass','mismatch','unverifiable')),
  details jsonb not null, created_at timestamptz not null default now(), acknowledged_at timestamptz, acknowledged_by text);

create table kill_switch_log (id uuid primary key, engaged boolean not null, reason text not null, actor text not null,
  created_at timestamptz not null default now());
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
| R4 | `quote.depositAddress == deposit.address`, memo equal, `quoteRequest.originAsset == deposit.origin_asset_id`, `quote.amountIn == deposit.amount_in`, `quote.deadline` and `deposit.deadline` equal as instants | `security` |
| R5 | registry rows with `address_norm == norm(quoteRequest.recipient)`, `asset_id == quoteRequest.destinationAsset`, matching `type`/`livemode`, `status = active`: exactly one → pass; several → prefer the single `signature`-verified row, else `ambiguous`; none → `security` | `security` / `ambiguous` |
| R6 | `appFees` has exactly one entry, `recipient == TREASURY`, `fee ≤ 500` | `security` |
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
| `unavailable` | `503 attestation_unavailable` | never |

No attacker-controllable input can engage the global kill switch; a compromised rails can at most *suppress* the switch, while the address is still refused.

## 6. API

| Method & path | Auth | Response | Errors |
|---|---|---|---|
| `POST /v1/attestations/deposit_instructions` | anonymous (gateway per-IP limit) | `200` signed attestation (cached per `deposit_key` until `expires_at`) | `400 invalid_request`, `409 attestation_refused`, `409 attestation_stale`, `503 attestation_unavailable` |
| `GET /v1/attestations/keys` | anonymous | `{keys:[{kid, public_key, not_after}]}` (informational; checkout uses build-pinned keys) | |
| `POST /internal/kill-switch` | ops (`sentinel.killswitch`) | `{engaged, reason}` → state | |
| `GET /internal/findings` | ops | findings list | |

## 7. Events

| Produced | Consumers |
|---|---|
| `evt.sentinel.kill_switch.engaged` · `.released` | rails (stores flag `quoting_enabled`), notify (page/email security contacts) |
| `evt.sentinel.audit_finding.created` (result ≠ pass) | compliance (opens case), notify |

| Consumed | Handler |
|---|---|
| `evt.rails.execution.settled` | sampling: 100% in Stage 1–2 and for amounts ≥ USD 10k; 5% otherwise later; independent verification via own RPC (same rules as rails ADR-0012) |
| `evt.identity.settlement_destination.disabled` | evict cached attestations for that destination |

## 8. Algorithms

- **Canonicalisation:** RFC 8785 JCS over the `attestation` object; Ed25519 over the UTF-8 bytes; `base64url` without padding.
- **Cache:** Valkey `att:<deposit_key>` → response until `expires_at`; a cached `signed` response is re-served without re-querying (idempotent for the SPA's retries); rejected/unavailable not cached.
- **Kill switch:** `engage(reason)` → insert log → outbox `evt.sentinel.kill_switch.engaged` → rails sets `control_flags.quoting_enabled=false` (consumed within seconds) → `rails` returns `rail_unavailable` for new live quotes. Release only by ops (two-person: request + approve in console) → `released` event.

## 9. Integrations

| Target | Timeout | Notes |
|---|---|---|
| 1Click `/v0/status` (own API key) | 2 s, 1 retry | separate credential from rails; budget 5 rps |
| sandbox `/v0/status` (test) | 1 s | |
| identity registry view (read-only replica/role) | 300 ms | |
| NEAR/EVM RPC (own provider accounts) | 3 s | auditor only |

## 10. Configuration

`NP_SENTINEL_SIGNING_KEY_FILE` (Ed25519, kid in filename), `NP_PAYMENTS_TICKET_PUBKEY`, `NP_RAILS_URL` (+ sentinel service credentials, scope `rails.read`), `NP_TREASURY_ACCOUNT`, `NP_ONECLICK_BASE_URL`, `NP_ONECLICK_API_KEY_FILE`, `NP_SANDBOX_BASE_URL`, `NP_SANDBOX_SIGNING_PUBKEY`, `NP_REGISTRY_DB_URL_FILE` (sentinel_ro), `NP_AUDIT_SAMPLE_RATE` (1.0 in Stages 1–2).

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
