# LLD — `nearpay-sentinel`

> Version 1.8 (stage 2 audit finding 2: the single-person release is allowed only when `NP_ENV` is `local` or `test` — allow-list §6/§15) · Version 1.7 (step 2.2 implementation: attested origin asset CR-15-07 §5; R1 without normalisation §5; release_requests constraints §4; idempotent console retries, ops-action aggregate id §6; refund legs in the audit §8.1; tests §13) · Version 1.6 (step 2.1 integration, spec 0.2.0: operator-action events for the unified audit trail §6/§7; payments consumes the kill switch, which blocks live refund legs and relays §7) · Version 1.5 (step 2.1 contracts: two-person kill-switch release and console routes §6, data §4, stage map §14) · Version 1.4 (mainnet findings 2026-11-07: R1 normalisation, R2 waiting-time echo, R4 deadline semantics; sentinel #2/#3) · Version 1.3 (step 1.2 implementation findings folded: data §4, R4/R5/R6 and classes §5, audit §8.1, cache §8, configuration §10, ops CLI §15) · Version 1.1 audited (`audits/sentinel.md`) · HLD §6 (two-party release), ADR-0010, ADR-0012, security T-03 · Foundations `00-foundations.md`

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

create table release_requests (              -- step 2.1: two-person release (`OpsApproval`, opa_)
  id uuid primary key, reason text not null, status text not null check (status in ('pending','approved','rejected','expired')),
  requested_by uuid not null, requested_role text not null, requested_at timestamptz not null default now(),
  decided_by uuid, decided_role text, decided_at timestamptz, decision_note text, expires_at timestamptz not null,
  switch_version integer not null,           -- version of the engaged switch the request is for
  check (status <> 'approved' or (decided_by is not null and decided_by <> requested_by)),   -- step 2.2: binds approvals only (the requester may withdraw = reject)
  check (status not in ('approved','rejected') or (decided_role = 'ops_admin' and decided_at is not null)));
create unique index release_one_pending on release_requests ((true)) where status = 'pending';
-- trigger: only pending → approved | rejected | expired; the request half never changes; no deletes (migration 0002)
-- audit_findings gains acknowledgement_note text (console acknowledge; acknowledged_by = usr_… or the CLI actor)
```

## 5. Co-signing rules

Request (from checkout): `{livemode, kind, attempt_id, attestation_ticket, deposit?: {address, memo, chain, origin_asset_id, amount_in, deadline}, requirement?: {pay_to, token, amount, fee_to, fee_amount, binding, deadline}}`.

**Gate (before any provider call):** `attestation_ticket` must verify with the payments ticket public key, be unexpired, and match `attempt_id`, `livemode` and the deposit key/binding; otherwise `401 ticket_invalid` (no provider call, no kill switch). The ticket only rate-gates access; it is never evidence of correctness.

**`route_quote` checks (all must pass):**

| # | Check | Failure class |
|---|---|---|
| R1 | `GET /v0/status?depositAddress&depositMemo` (live: 1Click with the partner key, a JWT sent as `Authorization: Bearer`; test: sandbox) returns 200 and a body that validates **as received** against spec 0.2.0's live shape (optional fields nullish, nested `quoteResponse.correlationId` optional; the Stage 1 normalisation is gone, step 2.2). A `null` signed optional field is "unset" like an absent one (the SDK hashes neither) | `unavailable` (503) |
| R2 | `verifyQuoteSignature(status.quoteResponse)` (live: SDK; test: sandbox key). The status echo omits the signed `quoteRequest.quoteWaitingTimeMs`; when the echo has none and the signature does not verify as echoed, it is re-checked with rails' value (3000) put back and nothing else | `security` |
| R3 | `quoteRequest.referral == 'nearpay'` | `security` |
| R4 | `quote.depositAddress == deposit.address`, memo equal, `quoteRequest.originAsset == deposit.origin_asset_id`, `quote.amountIn == deposit.amount_in`, `quoteRequest.deadline` (when refunds begin; not covered by the 1Click signature, whose hash spreads the quote over the request, but read from 1Click itself over TLS in R1) and `deposit.deadline` equal as instants, `quote.deadline` (signed; when the address goes inactive, 3 days later on live 1Click) not before it, and `deposit.chain` is the provider's deposit chain: `INTENTS` deposit → `near:mainnet`; otherwise the origin asset's chain when derivable (omft bridge prefix, or NEP-141 → NEAR). An EVM address is valid on every EVM chain, so the chain must be checked, not only the address | `security` |
| R5 | registry rows (identity view v2) with `address_norm == norm(quoteRequest.recipient)`, `asset_id == quoteRequest.destinationAsset`, matching `type`/`livemode`, `status = active`: exactly one → pass; several → prefer the single `signature`-verified row, else `ambiguous`; none active but a once-active **disabled** destination matches → `stale` (a merchant disabling a destination mid-checkout must not engage the global kill switch); none at all → `security` | `security` / `ambiguous` / `stale` |
| R6 | `appFees` (1Click echoes them split 50/50 and does not sign them, rails §8.1): at most 2 entries, total ≤ 500 bps, exactly one entry with `recipient == TREASURY` keeping `≥ total − max(ceil(total × share_bps / 10 000), max(floor_bps, stable_floor_bps))`; a fee-free quote (no entries) passes | `security` |
| R7 | `status ∈ {PENDING_DEPOSIT, KNOWN_DEPOSIT_TX, INCOMPLETE_DEPOSIT, PROCESSING}` and `quoteRequest.deadline > now + 60 s` | `stale` (409) |

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

**Attested asset (CR-15-07, step 2.2).** Every signed attestation carries `asset: {asset_id, symbol, decimals}` of the asset the payer sends (`deposit.origin_asset_id` / `requirement.token`), so checkout renders `amount_in` with signed decimals. Source: **sentinel's own read of the provider's token registry for the mode** — live 1Click `GET /v0/tokens` (sentinel's key, 5 s), test the sandbox's `GET /v0/tokens` (service token); never rails (rails stays sampling-only), checkout or the payer. Per-process cache: fresh 10 min; when a refresh fails a list younger than 6 h is used (an asset id's decimals do not change); an asset missing from the list refreshes at most once per 30 s. For `route_quote` the registry's decimals must also explain the provider-**signed** `quote.amountInFormatted` (= `amountIn / 10^decimals` truncated, or rounded at the half, to its own fraction digits), which binds the attested decimals to the signature. No registry, an unlisted asset, decimals outside 0–255, a symbol outside 1–32 chars, or a contradiction with the signed quote → `503 attestation_unavailable` (class `unavailable`, reason `asset: …`; never the kill switch). The attestation is assembled field by field (deposit/requirement copied key by key; no request field reaches `asset`).

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
| `GET /internal/findings` | ops | superseded by the console routes below (step 2.1); Stage 1 reads findings through the ops CLI | |

**Console routes (step 2.1, spec `OPS_ROUTES`, service `sentinel`, visibility `ops`, `/ops/…`):**

| operationId | Method & path | Scope · roles | Notes |
|---|---|---|---|
| `getOpsKillSwitch` | `GET /ops/kill_switch` | `sentinel.read` · ops, ops_admin | `OpsKillSwitch` = state + `version` + `pending_release` |
| `engageOpsKillSwitch` | `POST /ops/kill_switch/engage` | `sentinel.killswitch` · ops, ops_admin | one operator, no step-up (an emergency is never delayed); engaging an engaged switch is a no-op; a pending release request expires |
| `requestOpsKillSwitchRelease` | `POST /ops/kill_switch/release_requests` | `sentinel.killswitch` · ops, ops_admin, step-up | only while engaged, one pending at a time (`409 state_conflict`), expires after 1 h; emits `evt.sentinel.kill_switch.release_requested` (notify pages security contacts) |
| `approveOpsKillSwitchRelease` | `POST /ops/kill_switch/release_requests/{opa}/approve` | `sentinel.killswitch` · **ops_admin**, step-up | the second operator; releases in the same transaction (switch row version + 1, log row with actor `<requester>+<approver>`, outbox `released`) |
| `rejectOpsKillSwitchRelease` | `POST /ops/kill_switch/release_requests/{opa}/reject` | `sentinel.killswitch` · **ops_admin** | rejects or (by the requester) withdraws; executes nothing |
| `listOpsKillSwitchLog` | `GET /ops/kill_switch/log` | `sentinel.read` | newest first, `before_version` cursor |
| `listOpsFindings` · `getOpsFinding` | `GET /ops/findings` · `/{fnd}` | `sentinel.read` | default: non-pass, unacknowledged; filters result, acknowledged, account, execution, livemode, created |
| `acknowledgeOpsFinding` | `POST /ops/findings/{fnd}/acknowledge` | `sentinel.findings.write` | `{note}`; the compliance case is worked separately |

**Step 2.2 implementation notes.** The handlers re-check the ops principal (`ptype=ops`, `sub` = `usr_…`), `opsRoles` and step-up although kit 0.2.0 enforces them (fail closed). Every decision runs with the `kill_switch` row locked (`FOR UPDATE`), so concurrent requests/approvals serialise (one request created, one release). Console POSTs are idempotent by state, not by `Idempotency-Key` (ops principals have no account scope for the kit's key store): the same operator repeating the same request (still pending, same reason) gets it back with 201; the approver repeating an approval gets the current state; the rejecter repeating a rejection gets the approval; anything else on a decided request is `409`. Engage on an engaged switch changes nothing but still expires a pending request (pressing engage objects to it); a stale pending request (expired, or the switch version moved) is recorded `expired` when touched. Ops actions: `kill_switch.engaged`, `kill_switch.release_requested`, `kill_switch.release_approved`, `kill_switch.release_rejected`, `kill_switch.release_withdrawn` (requester rejects), `finding.acknowledged` (target `audit_finding`, with the account and mode). Envelope aggregate id of an ops action is `aud_<action uuid>` (the outbox needs a prefixed id; the object `id` stays the bare uuid). The ops CLI publishes **no** ops-action event: `OpsActionEvent.actor.sub` must be a `usr_…` id and the CLI has none (change request in the step 2.2 report).

**Two-person release contract (server-side):** the approver is an `ops_admin` whose operator id (`sub` of the verified ops token, never a client field) differs from the requester's, else `403 approver_conflict`; requester and approver both hold a fresh step-up; the request must be `pending`, unexpired and for the current engaged `version` (an engage or release in between expires it, `409 state_conflict`); the table's check constraint repeats the different-operator rule. `POST /internal/kill-switch` (ops CLI) keeps engage; its single-person release is allowed only when `NP_ENV` is exactly `local` or `test` (testbed T7) and answers `409 two_person_required` on any other value — `server`, an unknown stage, empty (allow-list, fail closed; stage 2 audit finding 2).

**Unified console audit trail (T-12, lead decision at step 2.1 integration).** Every console write sentinel executes (engage, release request/approve/reject, finding acknowledge) also publishes `evt.sentinel.ops_action.recorded` (`OpsActionEvent`: action id, `actor {sub, ops_role}`, `action` such as `kill_switch.engaged` / `kill_switch.release_approved` / `finding.acknowledged`, `target {type, id}`, `account_id`, `approval`, `occurred_at`; ids only) through the outbox in the same transaction. identity consumes it into `audit_events` so the console audit log is one trail; `kill_switch_log` and `ops_approvals` stay the detail source. The ops CLI's engage/release publish the same event with the CLI actor.

## 7. Events

| Produced | Consumers |
|---|---|
| `evt.sentinel.kill_switch.engaged` · `.released` | rails (stores flag `quoting_enabled`), agentpay (refuses live settles, checkout relays and, from Stage 2, refund relays), payments (Stage 2: no new live refund legs, payments §17 item 16), notify (page/email security contacts) |
| `evt.sentinel.ops_action.recorded` (step 2.1 integration, `OpsActionEvent`) | identity (`audit_events`: one console audit trail, §6) |
| `evt.sentinel.kill_switch.release_requested` (step 2.1, `OpsApproval`) | notify (security contacts: a second operator is needed) |
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
- **Refund legs (`kind: refund`, step 2.2).** Rails emits `evt.rails.execution.settled` for refund legs too (`origin` = merchant side, `settlement` = payer side). The `destination` finding then checks the **refund source**: `origin.refund_to` (type from `origin.refund_type`, asset `origin.asset_id`, chain `origin.chain`) was a destination of the account active at creation (details `side: refund_source`); the payer's refund address is never a registry entry, so checking it would raise false mismatches. The `settlement` finding is unchanged: sentinel's own read shows `settled_amount` reached the payer's address in the settlement asset (double-use rule included). A merchant-signed Intents transfer (`refund.transfer`) must be signed by the source, in the source asset, from an Intents source, else `mismatch`.
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
- Property (step 2.2): two-person release over any interleaving of engage/request/approve/reject by any operators and times — never released by one principal, approver an `ops_admin` ≠ requester, both step-ups fresh, inside the hour, for the request's switch version; at most one pending; concurrent requests create one. Attestation field equality (deposit/requirement key by key, `asset` = origin asset with the registry's display data) and a full-precision `amountInFormatted` pinning the decimals.
- Integration (step 2.2): every console route over HTTP with kit ops tokens, the DB constraint and trigger, expiry with a shifted clock, concurrent requests/approvals, `two_person_required` on an `NP_ENV=server` api and on the CLI for `server`, unknown and empty `NP_ENV`, outbox rows per change and the relay publishing `release_requested` / `ops_action.recorded`; refund-leg audits on NEAR.

## 14. Stage mapping

Stage 1: everything above except two-person kill-switch release (Stage 2 console) — Stage 1 release via ops CLI with audit log. Stage 2: console routes and the two-person release (§6); the single-person CLI release is refused on the server stage; attested origin asset (CR-15-07, §5); refund legs in the audit (§8.1). Implemented in step 2.2.

## 15. Ops CLI (Stage 1)

`docker compose -p nearpay exec sentinel-api /nodejs/bin/node dist/ops.js <command>`: `kill-switch` · `kill-switch-engage <reason>` · `kill-switch-release <reason>` (one operator, logged; only when `NP_ENV` is `local`/`test`, otherwise refused with `two_person_required`; the CLI has no `--actor`, so the testbed wrapper prefixes the operator's name to the reason) · `findings [--all] [--limit N]` · `finding-ack <fnd_…> <note>` · `attestations <pa_… | deposit key | deposit address>`.
