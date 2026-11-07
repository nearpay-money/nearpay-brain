# LLD — `nearpay-compliance`

> Spec 0.2.1 update (spec 0.2.1: account display names, operator check on assignment, `fresh` screenings, `subject` case filter, CLI actor on ops actions — §"Spec 0.2.1 contract changes") · Version 1.7 (stage 2 audit fixes: single-person suspension only on `NP_ENV` local/test §5.1; an unknown amount fails closed when the provider is unavailable §4; blocklist soft delete — migration `0003`, `ops_actions.details` §3/§5.1; two-person blocklist removal proposed §5.1) · Version 1.6 (step 2.2 implementation: data §3 — migration `0002`, `ops_approvals` checks, `ops_actions`, `cases.payment_intent`, `blocklist` chain; screening provider port §4; `startKyb` no longer moves to `pending`, console approval needs a complete declaration §5; principal-scoped idempotency and operator guard §5.1; jobs §7; config §8) · Version 1.5 (step 2.1 integration, spec 0.2.0: `createInternalCase` for payments' `reconciliation_diff` cases §5, operator-action events for the unified audit trail §5.1/§6, KYB field semantics §5) · Version 1.4 (step 2.1 contracts: self-declared KYB without a vendor per ADR-0020 §3–4 — data §3, screening interface §4, API §5 and §5.1 console routes, events §6, configuration §8, privacy §9, stage map §12) · Version 1.3 (step 1.2 implementation findings folded: data §3, screening §4, KYB §5, list refresh §7, configuration §8, ops CLI §13) · Version 1.1 audited (`audits/compliance.md`) · HLD §13 · FR-CMP-01…04 · Foundations `00-foundations.md`

## 1. Scope

**Owns:** address/account screening (pre and post), screening cache, blocklist, geo policy, KYB cases (self-declared minimum business details + ops decision; no KYB vendor, ADR-0020), compliance cases, destination re-screening, account suspension decisions (two-person rule).
**Policy is data:** region blocking needs Cloudflare's visitor-location headers (`cf-ipcountry`, `cf-region-code` managed transform) on the server stage; jurisdiction-specific thresholds live in `policies` rows, not code (jurisdiction undecided).

## 2. Runtime

| Compose service | Command | Replicas |
|---|---|---|
| `compliance-api` | `api` | 2 (internal routes + dashboard KYB routes `/v1/account/kyb…` + console routes `/ops/…`; no vendor webhook route in v1) |
| `compliance-worker` | `worker` | 1 (list refresh, re-screening, consumers, outbox relay) |

## 3. Data

```sql
create table policies (
  id text primary key,                    -- 'default', later per jurisdiction
  fail_closed_threshold_usd_micros bigint not null default 1000000000,   -- USD 1,000
  blocked_countries char(2)[] not null default '{CU,IR,KP,SY}',
  blocked_regions text[] not null default '{UA-43,UA-14,UA-09}',          -- Crimea, Donetsk, Luhansk (ISO 3166-2)
  kyb_required_for_live boolean not null default true, retention_years smallint not null default 5,
  updated_at timestamptz not null default now());

create table sanctions_entries (          -- local list (OFAC SDN digital currency addresses + vendor deltas)
  chain_family text not null,             -- evm | bitcoin | solana | tron | other
  address_norm text not null, source text not null,
  currency text not null,                 -- OFAC feature code (XBT, ETH, USDT, …)
  listed_at date, primary key (chain_family, address_norm));
create index sanctions_entries_address on sanctions_entries (address_norm);   -- subjects on chains the registry does not know

create table sanctions_lists (source text primary key, fetched_at timestamptz not null, entries integer not null, sha256 bytea not null);

create table blocklist (subject_type text not null, subject text not null, reason text not null, created_by text not null,
  created_at timestamptz not null default now(), primary key (subject_type, subject),
  id uuid not null unique default gen_random_uuid(), chain text);   -- step 2.1: API id bl_…, optional chain (console routes)
-- stage 2 audit (migration 0003, finding 10): removal is a soft delete so the trail keeps the subject (INV-9).
alter table blocklist add removed_at timestamptz, add removed_by text, add removal_note text,
  check ((removed_at is null) = (removed_by is null));
-- primary key moves to id; (subject_type, subject) is unique among active rows only (a removed subject can be
-- listed again as a new bl_); a trigger refuses DELETE and any change to a removed row. Screening, the console
-- list and the CLI read/modify active rows (removed_at is null) only.

create table screenings (
  id uuid primary key, subject_type text not null check (subject_type in ('address','near_account','destination')),
  chain text, subject_norm text not null, context text not null, account_id uuid, livemode boolean not null,
  decision text not null check (decision in ('allow','block','review')), risk text, reasons text[] not null default '{}',
  provider text not null, cached boolean not null, expires_at timestamptz not null, created_at timestamptz not null default now());
create index screenings_cache on screenings (subject_type, chain, subject_norm, expires_at desc);

create table kyb_cases (
  id uuid primary key, account_id uuid not null unique, status text not null
    check (status in ('not_started','pending','needs_info','approved','rejected')),
  decision_reason text, decided_by text,
  -- step 2.1 (ADR-0020): self-declared profile instead of vendor columns. The Stage 1 columns vendor,
  -- vendor_applicant_id and level stay in the table (always null) until a later migration drops them.
  profile_enc bytea,                        -- envelope-encrypted JSON of `KybProfile` (legal name, country of incorporation,
                                            -- registration number, business type, website, address, representative, volume band)
  declared_by uuid,                         -- owner (user id) who submitted the declaration
  requested_info text,                      -- shown to the merchant while needs_info (never the internal decision_reason)
  submitted_at timestamptz,
  created_at timestamptz not null default now(), decided_at timestamptz, version integer not null default 0);
create table kyb_case_transitions (… per foundations §7 …);

create table ops_approvals (               -- step 2.1: two-person rule for account suspend/reinstate (`OpsApproval`, opa_)
  id uuid primary key, action text not null check (action in ('account.suspend','account.reinstate')),
  account_id uuid not null, reason text not null,
  status text not null check (status in ('pending','approved','rejected','expired')),
  requested_by uuid not null, requested_role text not null, requested_at timestamptz not null default now(),
  decided_by uuid, decided_role text, decided_at timestamptz, decision_note text, expires_at timestamptz not null,
  -- step 2.2: the database repeats the server-side rule for approvals only — a requester may still withdraw
  -- (reject) their own request, which the v1.4 `decided_by <> requested_by` check would have forbidden
  check (status <> 'approved' or (decided_by is not null and decided_by <> requested_by and decided_role = 'ops_admin')));
create unique index ops_approvals_one_pending on ops_approvals (account_id) where status = 'pending';

create table ops_actions (                 -- step 2.2: append-only log of every console write (INV-9); the same id is
  id uuid primary key, actor_sub uuid not null, actor_role text not null,   -- published as ops_action.recorded
  action text not null, target_type text not null, target_id text not null, account_id uuid, approval_id uuid,
  note_enc bytea,                          -- reason/note, envelope-encrypted; never published
  created_at timestamptz not null default now(),
  details jsonb);                          -- 0003: internal structured details (blocklist add/remove: subject_type,
                                           -- subject, chain); never published

create table account_flags (account_id uuid primary key, suspended boolean not null, reason text not null, actor text not null,
  version integer not null default 1, updated_at timestamptz not null default now());   -- suspension decisions

create table cases (
  id uuid primary key, account_id uuid, livemode boolean not null,
  kind text not null check (kind in ('screening_hit','post_screen_hit','held_execution','audit_mismatch','merchant_report','reconciliation_diff')),
  subject jsonb not null, severity text not null check (severity in ('low','medium','high','critical')),
  status text not null check (status in ('open','investigating','closed')), resolution text,
  assignee uuid,                          -- step 2.1: operator working the case (console assign)
  payment_intent text,                    -- step 2.2: `Execution.reference.payment_intent` of execution reviews (Review.payment_intent)
  created_at timestamptz not null default now(), closed_at timestamptz, version integer not null default 0);
create table case_events (id uuid primary key, case_id uuid not null references cases(id), actor text not null,
  action text not null, note text, created_at timestamptz not null default now());

create table monitored_destinations (destination_id uuid primary key, account_id uuid not null, chain text not null,
  address_norm text not null, active boolean not null, last_screened_at timestamptz);
```

## 4. Screening decision
```
screen({subject_type, chain, subject, context, account_id, livemode, amount_usd_micros?}):
  norm = normalize(chain, subject)     // EVM and bech32 lower-case; NEAR accounts lower-case and eth-implicit NEAR accounts
                                       // also checked against EVM listings; a chain the registry does not know is matched in
                                       // every family (list key family '*')
  if livemode=false: decision = (subject contains 'sanctioned', case-insensitive) ? block : allow   // sandbox magic (api-conventions §8)
  hit = blocklist(norm) or sanctions_entries(family(chain), norm) → block (reason 'sanctions' | 'blocklist')
  cached = latest screening for (type, chain, norm) with expires_at > now → return (cached=true)
  no sanctions list imported yet (sanctions_lists empty) → degraded, exactly like "provider unavailable" below (fail-safe)
  provider (vendor-neutral interface, ADR-0020 §4; step 2.2: port `ScreeningProvider.check(db, subject) → {name, hits,
    unavailable, risk}`, the fail policy stays in the pure decision): v1 has one live provider, `local` (the list + blocklist checks above; risk null);
    a later vendor is a second provider behind the same interface (risk API → severe/high block, medium review = allow + case,
    low allow) and needs no contract change. Screening responses name it in `provider` (`local` | `sandbox`)
  provider unavailable: amount_usd_micros absent OR >= policy.fail_closed_threshold → 503 screening_unavailable
    else allow (reason 'vendor_unavailable', post_screen_required). An unknown amount fails closed (stage 2 audit,
    finding 9): callers without an amount — identity destinations (create and activation), agentpay x402 payers,
    rails legs it cannot price — get 503 while the provider is degraded (for `local`: before the first list import)
  persist screening (expires 24 h allow / 7 d block); degraded allows are never cached
```

## 5. API

| Method & path | Caller (scope) | Request → Response |
|---|---|---|
| `POST /internal/screenings` | rails, identity, agentpay (`compliance.screen`) | above → `{screening_id, decision, risk, reasons, cached, post_screen_required}` |
| `GET /internal/geo-policy` | payments (`compliance.read`) | `{blocked_countries, blocked_regions, version}` (payments caches; refreshes on `evt.compliance.policy.updated`) |
| `POST /v1/account/kyb` | session owner (dashboard) | creates/returns the KYB case → `KybCase` (`vendor_flow_url` is always null: no vendor flow, ADR-0020). Step 2.2: it no longer moves the case to `pending` (Stage 1 did) — `pending` means submitted, which needs `submitKyb` |
| `GET /v1/account/kyb` (`getKyb`, step 2.1) | session owner/admin | → `KybCase` with `profile`, `editable`, `requested_info`, `submitted_at`, `decided_at` |
| `PATCH /v1/account/kyb` (`updateKybProfile`) | session owner | partial `KybProfileUpdate` (null clears) → `KybCase`; only in `not_started`/`needs_info` (else `409 state_conflict`); creates the case if missing |
| `POST /v1/account/kyb/submit` (`submitKyb`) | session owner, step-up | `{declaration: true}` → `pending`; every profile field but `website` required (`400 kyb_profile_incomplete`, `params.fields`); only from `not_started`/`needs_info`; emits `kyb.submitted` |
| `GET /internal/kyb_cases/{account_id}` (`getInternalKybCase`, step 2.1) | identity (`compliance.read`) | → `{account_id, status, profile, decided_at}`; identity reads it on `kyb.approved` (legal name stays off the bus) |
| ~~`POST /webhooks/kyb/{vendor}`~~ | — | not in v1 (no KYB vendor, ADR-0020); the route is not implemented |
| `POST /internal/kyb_cases/{account_id}/decision` | ops (`compliance.kyb.decide`) | `{decision: approved\|rejected\|needs_info, reason}` (Stage 1 manual path). The case must exist (created by `account.created` or the owner's start) → else `404 resource_missing`, so a mistyped account id never mints a decision. Repeating the current decision is a no-op without an event; a different decision (including reversing an approval) transitions and emits `kyb.<decision>`; identity recomputes live mode from the event |
| `POST /internal/cases` (`createInternalCase`, step 2.1 integration) | payments (`compliance.cases.write`) | `{kind: reconciliation_diff, account_id, livemode, subject: {type: reconciliation_run, id: rcn_…}, severity?, note}` → `201 ComplianceCase`; idempotent on `Idempotency-Key: rcn:<run id>`; default severity `high`; never a merchant review (`review: false`, no `review.opened`); one open case per kind and subject applies. The operator closes the case (`closeOpsCase`) after resolving the run on payments' console routes (payments §6.4) |
| console routes `/ops/…` | ops principals | KYB queue and decisions, cases, blocklist, screening, re-screen, suspension with two-person approval — §5.1 (step 2.1, closes OQ-T-02 for compliance). The ops CLI keeps the same use cases in-process |

**KYB state machine (self-declared, ADR-0020 §3).** Status values are unchanged (`pending` means "submitted for review").

| From | Command | Guard | To | Side effects |
|---|---|---|---|---|
| — | `account.created` consumer, `startKyb`, or the first `updateKybProfile` | — | `not_started` | — |
| `not_started` · `needs_info` | `updateKybProfile` (owner) | — | same | profile stored (encrypted), audited |
| `not_started` · `needs_info` | `submitKyb` (owner, step-up) | profile complete; declaration `true` | `pending` | `submitted_at`, `declared_by`; event `kyb.submitted` (identity sets `kyb_status`, notify confirms to owners) |
| `pending` | ops `approved` | — | `approved` | event `kyb.approved` → identity copies the declared legal name and country (`getInternalKybCase`), recomputes live mode |
| `pending` · `approved` | ops `rejected` | — | `rejected` | event `kyb.rejected` (reverses live mode) |
| `pending` · `approved` · `rejected` | ops `needs_info` (with `requested_info` in the console) | — | `needs_info` | event `kyb.needs_info` with `requested_info`; the merchant can edit again |

The console's `decideOpsKybCase` approves only from `pending` and only with a complete declaration (step 2.2: a Stage 1 case started without details can sit in `pending` with an empty profile → `409 state_conflict`; ask for information instead); the Stage 1 manual endpoint above keeps its rules (any decision on an existing case, repeats are no-ops), so the CLI and testbed T1 keep working. `approved` means "ops reviewed the self-declared details", not third-party verification (identity §4).

**KYB field semantics (lead decision at step 2.1 integration, ADR-0020 decision 3).** The public field names `kyb_verified` (sentinel attestation, identity settlement context) and `verified_legal_name` (identity) stay as they are — renaming them would not be additive. Their meaning is: the legal name an owner self-declared and NearPay ops approved. No client may present them as identity verification; the UI copy is "Business details reviewed by NearPay" and never "verified identity" (checkout, dashboard, console, docs).

### 5.1 Console routes (step 2.1, spec `OPS_ROUTES`, service `compliance`)

All `ops` visibility under `/ops/…`, idempotent writes, audited with the operator id; `opsRoles` `ops`+`ops_admin` unless marked.

| operationId | Method & path | Scope · notes |
|---|---|---|
| `listOpsKybCases` · `getOpsKybCase` | `GET /ops/kyb_cases` (default `pending`, oldest first) · `GET /ops/kyb_cases/{acct}` | `compliance.read` → `OpsKybCase` (profile, history, internal reason) |
| `decideOpsKybCase` | `POST /ops/kyb_cases/{acct}/decision` | `compliance.kyb.decide`, step-up; `{decision, reason, requested_info?}` |
| `listOpsCases` · `getOpsCase` | `GET /ops/cases` · `GET /ops/cases/{case}` | `compliance.read` → `ComplianceCase` (timeline on retrieve) |
| `assignOpsCase` · `commentOpsCase` · `closeOpsCase` | `POST /ops/cases/{case}/assign` · `/comments` · `/close` | `compliance.cases.write`; assign moves `open → investigating`; close `{resolution, note}` emits `review.closed` for merchant-visible reviews |
| `listOpsBlocklist` · `addOpsBlocklistEntry` · `removeOpsBlocklistEntry` | `GET /ops/blocklist` · `POST /ops/blocklist` (step-up) · `POST /ops/blocklist/{bl}/remove` (**ops_admin**, step-up) | `compliance.read` / `compliance.blocklist.write`; adding invalidates cached allows of the subject and queues a re-screen of matching destinations |
| `createOpsScreening` · `createOpsRescreen` | `POST /ops/screenings` · `POST /ops/rescreens` | `compliance.screen`; uncached lookup; re-screen job (`job_`) for one account or all |
| `getOpsMerchantSuspension` | `GET /ops/merchants/{acct}/suspension` | `compliance.read` → `MerchantSuspension` with `pending_request` |
| `createOpsSuspensionRequest` | `POST /ops/merchants/{acct}/suspension_requests` | `compliance.accounts.write`, step-up; `{action: suspend\|reinstate, reason}` → pending `OpsApproval` (24 h); emits `account.action_requested` |
| `approveOpsSuspensionRequest` · `rejectOpsSuspensionRequest` | `POST …/suspension_requests/{opa}/approve` · `/reject` | **ops_admin**; approve needs step-up and executes: `account_flags` + `account.suspended`/`reinstated` in the same transaction |

**Principals without an account (step 2.2).** The service bootstrap's automatic idempotency is scoped to `principal.acct`; console writes and `createInternalCase` have none, so compliance scopes their `Idempotency-Key`s to the principal (`ops:<sub>` / `service:<sub>`, a deterministic pseudo-account uuid in `idempotency_keys`, same replay/conflict rules and retention). Every console handler also re-checks the operator from the route (`ptype=ops`, `usr_` sub, `ops_role` ∈ `opsRoles`, fresh step-up on step-up routes) on top of the kit. The ops-action envelope `aggregate.id` is `aud_<action uuid>` (the outbox keys aggregates by prefixed id; `ops_action` has no prefix — same as sentinel and rails); `object.id` stays the uuid.

**Two-person rule (server-side, never trusted from the client):** the approver's operator id (`sub` of the verified ops token) must differ from `requested_by`, and the approver must hold `ops_admin` (`403 approver_conflict`); both need a fresh step-up; one pending request per account; an action that no longer matches the account state at approval expires the request (`409 state_conflict`). The `ops_approvals` check constraint repeats the different-approver rule. The ops CLI's `account-suspend`/`account-reinstate` remain for local and test stacks only: they run only when `NP_ENV` is exactly `local` or `test` (allow-list, stage 2 audit finding 2); any other value — `server`, an unknown stage, empty — refuses with `two_person_required`. The use case takes the environment without a default.

**Blocklist removal (stage 2 audit, finding 10).** Removal (console and CLI) is a soft delete: the entry keeps subject, chain and reason and records `removed_at`, `removed_by` (`ops:<sub>` or the CLI actor) and `removal_note`; the operator action (`blocklist_entry.added` / `.removed`) stores the subject and chain in `ops_actions.details` (internal; the event stays ids-only, and its `bl_` target id now always resolves to the entry). A removed entry is `404` to a second removal. *Proposal (not implemented):* make blocklist removal two-person like suspension (an `ops_approvals` action `blocklist.remove` requested by one operator and approved by a different `ops_admin`), since un-blocking a sanctioned/fraud address is as sensitive as reinstating a merchant; needs a spec change (`OpsApproval.action` enum and the request/approve routes).

**Unified console audit trail (T-12, lead decision at step 2.1 integration).** Every console write that compliance executes (KYB decision, case assign/comment/close, blocklist add/remove, screening, re-screen, suspension request/approve/reject) also publishes `evt.compliance.ops_action.recorded` (`OpsActionEvent`: action id, `actor {sub, ops_role}`, `action` such as `case.closed`, `target {type, id}`, `account_id`, `approval` (`opa_…`), `occurred_at`; ids only — no notes, emails or addresses) from the same transaction (outbox). identity consumes it into `audit_events`, so the console audit log (`listOpsAuditEvents`) is one trail; compliance's own append-only logs (case events, `ops_approvals`) stay the source for the detail pages.

## 6. Events

| Produced | Consumers |
|---|---|
| `evt.compliance.kyb.approved` · `.rejected` · `.needs_info` (step 2.1: `requested_info` for `needs_info`) | identity, notify |
| `evt.compliance.destination.blocked` | identity, notify |
| `evt.compliance.account.suspended` · `.reinstated` | identity, notify |
| `evt.compliance.review.opened` · `.closed` | payments (flag), webhooks (`review.opened/closed`, consumed from step 2.1), notify |
| `evt.compliance.policy.updated` | payments |
| `evt.compliance.kyb.submitted` (step 2.1) | identity (`kyb_status: pending`), notify (`kyb.submitted` to owners) |
| `evt.compliance.account.action_requested` (step 2.1, `OpsApproval`) | notify (security contacts: a second operator is needed) |
| `evt.compliance.ops_action.recorded` (step 2.1 integration, `OpsActionEvent`) | identity (`audit_events`: one console audit trail, §5.1) |

| Consumed | Handler |
|---|---|
| `evt.identity.account.created` | create `kyb_cases(not_started)` |
| `evt.identity.settlement_destination.activated` · `.disabled` | maintain `monitored_destinations` |
| `evt.rails.execution.settled` | post-screen without the cache: the refund address (an Intents refund is a NEAR account), and sender addresses when known (Explorer `senders`, not in Stage 1); hit → case `post_screen_hit` + `review.opened` (subject type `execution`; payments maps it to the intent). The payment stands (INV-1) |
| `evt.rails.execution.held` | case `held_execution` (severity by amount) + `review.opened` |
| `evt.sentinel.audit_finding.created` | case `audit_mismatch` (critical in live mode) + `review.opened` |
| `evt.agentpay.agent_payment.settled` | post-screen the payer without the cache |

One open case per kind and subject; destination blocks open `screening_hit` cases.

## 7. Jobs

| Job | Cadence | Behaviour |
|---|---|---|
| `sanctions.refresh` | on first start and daily 03:00 UTC | download the OFAC SDN advanced XML (follows the 302 to a signed S3 URL; ~128 MB, streamed) → extract every "Digital Currency Address - <CODE>" feature (feature type ids are read from the document, not hard-coded; SP-15) → replace the list in one transaction (diff logged, `sanctions_lists` row); an empty result is refused; a failed download keeps the previous list; alert after 48 h stale. Offline: `ops.js sanctions-import -` reads the file from stdin |
| `destinations.rescreen` | daily | screen every `monitored_destinations.active`; block → `destination.blocked` + case |
| `purge.screenings` | daily | delete expired cache rows older than retention of the decision |
| `ops_approvals.expire` (step 2.2) | every 15 min | pending two-person requests past `expires_at` → `expired` (requests and approvals also expire due rows of their account inline) |
| `destinations.rescreen` (console) | on `createOpsRescreen` | same job with payload `{account_id}`; dedupe key `destinations.rescreen:ops:<uuid>` |

## 8. Configuration

`NP_SANCTIONS_SOURCE_URL` (default `https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/SDN_ADVANCED.XML`, SP-15; `file:` for air-gapped machines), `NP_SANCTIONS_REFRESH` (default `true`; offline stacks set `false` and import with the ops CLI), `NP_SANCTIONS_TIMEOUT_MS` (300000), `NP_OPS_ACTOR` (default actor recorded by the ops CLI; `--actor` overrides, else `cli@<hostname>`), `NP_SCREENING_PROVIDER` (`local`, the only v1 value; ADR-0020 §4 — replaces `NP_SCREENING_VENDOR`; no vendor key file; step 2.2: `NP_SCREENING_VENDOR=none` and `NP_KYB_VENDOR=manual` are still accepted so older Compose files boot), `NP_OPS_APPROVAL_TTL_SECONDS` (86400). No KYB vendor settings in v1 (`NP_KYB_VENDOR*`, `NP_KYB_WEBHOOK_SECRET_FILE` removed).

## 9. Security and privacy

Case notes and the KYB profile (a sole proprietor's legal name and the representative are personal data) are envelope-encrypted; they never travel in events; ops access via console only (audited). No identity documents are collected at all (self-declared KYB, ADR-0020). Retention per `policies.retention_years`.

## 10. Observability

`…_screenings_total{context,decision,provider,cached}`, `…_screening_duration_seconds`, `…_vendor_unavailable_total`, `…_cases_open{kind,severity}`, `…_sanctions_list_age_hours`. Alerts: list age > 48 h; any `critical` case; vendor unavailable > 5 min.

## 11. Tests

Normalisation per chain family; OFAC XML parser on a fixture; fail-closed threshold both sides; sandbox magic addresses; post-screen flow; KYB manual decision → identity live mode in testbed.

## 12. Stage mapping

| Stage | Scope |
|---|---|
| 1 | local OFAC list + blocklist screening, geo policy (country only locally — no geo headers on localhost), manual KYB decision endpoint (ops script), post-screen consumer (Stage 1 has no Explorer `senders`, so post-screening re-checks `refund_to`/payer accounts only), cases (ops CLI), daily destination re-screen |
| 2 | self-declared KYB profile + submit, ops KYB decisions in the console (no KYB vendor, ADR-0020); screening stays `local` behind the provider interface; console case management, blocklist, screening/re-screen, two-person suspension (§5.1) |

## 13. Ops CLI (Stage 1)

No network path exists for these actions in Stage 1 (Stage 2 moves them to the console). Inside the api container (`docker compose exec compliance-api /nodejs/bin/node dist/ops.js …`): `kyb-show` · `kyb-decide <acct> <decision> --reason [--actor]` · `cases [--all]` · `case-investigate` · `case-close --resolution` · `case-note --note` · `blocklist-add` · `blocklist-remove [--reason]` (soft delete) · `account-suspend` · `account-reinstate` · `geo-policy` · `geo-policy-set --countries [--regions]` (emits `policy.updated`) · `sanctions-status` · `sanctions-refresh` · `sanctions-import <file | ->` · `screen <type> <subject> [--chain] [--live]`. Output is JSON; errors are JSON on stderr with exit 1 (2 for usage).

No network path exists for these actions in Stage 1 (Stage 2 moves them to the console). Inside the api container (`docker compose exec compliance-api /nodejs/bin/node dist/ops.js …`): `kyb-show` · `kyb-decide <acct> <decision> --reason [--actor]` · `cases [--all]` · `case-investigate` · `case-close --resolution` · `case-note --note` · `blocklist-add` · `blocklist-remove` · `account-suspend` · `account-reinstate` · `geo-policy` · `geo-policy-set --countries [--regions]` (emits `policy.updated`) · `sanctions-status` · `sanctions-refresh` · `sanctions-import <file | ->` · `screen <type> <subject> [--chain] [--live]`. Output is JSON; errors are JSON on stderr with exit 1 (2 for usage).

## Spec 0.2.1 contract changes (step 2.2/2.3 change requests, `nearpay-spec/CHANGELOG.md`)

- Consumes **`evt.identity.account.updated`** (besides `.created`) to keep account display names; `ComplianceCase.account_display_name` (null for platform cases/unknown).
- **`assignOpsCase`** checks the assignee with identity `getInternalOpsOperator` (`identity.operators.read`): unknown or `active: false` → `400 invalid_request` (`assignee`); identity unavailable → fail closed.
- `createScreening.fresh: true` bypasses the screening cache.
- `listOpsCases` `subject` filter: exact `subject.id`.
- Ops-action events from the CLI may use `actor.sub = cli@<host>[:<name>]` without `ops_role`; envelope `aggregate.id` = `aud_<TypeID>` (now in the spec).
