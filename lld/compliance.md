# LLD — `nearpay-compliance`

> Version 1.3 (step 1.2 implementation findings folded: data §3, screening §4, KYB §5, list refresh §7, configuration §8, ops CLI §13) · Version 1.1 audited (`audits/compliance.md`) · HLD §13 · FR-CMP-01…04 · Foundations `00-foundations.md`

## 1. Scope

**Owns:** address/account screening (pre and post), screening cache, blocklist, geo policy, KYB cases and vendor integration, compliance cases, destination re-screening, account suspension decisions.
**Policy is data:** region blocking needs Cloudflare's visitor-location headers (`cf-ipcountry`, `cf-region-code` managed transform) on the server stage; jurisdiction-specific thresholds live in `policies` rows, not code (jurisdiction undecided).

## 2. Runtime

| Compose service | Command | Replicas |
|---|---|---|
| `compliance-api` | `api` | 2 (internal only + vendor webhook route + KYB start for dashboard) |
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
  created_at timestamptz not null default now(), primary key (subject_type, subject));

create table screenings (
  id uuid primary key, subject_type text not null check (subject_type in ('address','near_account','destination')),
  chain text, subject_norm text not null, context text not null, account_id uuid, livemode boolean not null,
  decision text not null check (decision in ('allow','block','review')), risk text, reasons text[] not null default '{}',
  provider text not null, cached boolean not null, expires_at timestamptz not null, created_at timestamptz not null default now());
create index screenings_cache on screenings (subject_type, chain, subject_norm, expires_at desc);

create table kyb_cases (
  id uuid primary key, account_id uuid not null unique, status text not null
    check (status in ('not_started','pending','needs_info','approved','rejected')),
  vendor text, vendor_applicant_id text, level text, decision_reason text, decided_by text,
  created_at timestamptz not null default now(), decided_at timestamptz, version integer not null default 0);
create table kyb_case_transitions (… per foundations §7 …);

create table account_flags (account_id uuid primary key, suspended boolean not null, reason text not null, actor text not null,
  version integer not null default 1, updated_at timestamptz not null default now());   -- suspension decisions

create table cases (
  id uuid primary key, account_id uuid, livemode boolean not null,
  kind text not null check (kind in ('screening_hit','post_screen_hit','held_execution','audit_mismatch','merchant_report')),
  subject jsonb not null, severity text not null check (severity in ('low','medium','high','critical')),
  status text not null check (status in ('open','investigating','closed')), resolution text,
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
  no sanctions list imported yet (sanctions_lists empty) → degraded, exactly like "vendor unavailable" below (fail-safe)
  vendor (Stage 2+): risk API (1 s) → map: severe/high → block, medium → review (treated as allow + case), low → allow
  vendor unavailable: amount_usd_micros >= policy.fail_closed_threshold → 503 screening_unavailable else allow (reason 'vendor_unavailable', post_screen_required)
  persist screening (expires 24 h allow / 7 d block); degraded allows are never cached
```

## 5. API

| Method & path | Caller (scope) | Request → Response |
|---|---|---|
| `POST /internal/screenings` | rails, identity, agentpay (`compliance.screen`) | above → `{screening_id, decision, risk, reasons, cached, post_screen_required}` |
| `GET /internal/geo-policy` | payments (`compliance.read`) | `{blocked_countries, blocked_regions, version}` (payments caches; refreshes on `evt.compliance.policy.updated`) |
| `POST /v1/account/kyb` | session owner (dashboard) | starts/returns KYB case → `{status, vendor_flow_url?}` (Stage 2: vendor hosted flow) |
| `POST /webhooks/kyb/{vendor}` | anonymous route; vendor HMAC signature verified | vendor decision → case update |
| `POST /internal/kyb_cases/{account_id}/decision` | ops (`compliance.kyb.decide`) | `{decision: approved\|rejected\|needs_info, reason}` (Stage 1 manual path). The case must exist (created by `account.created` or the owner's start) → else `404 resource_missing`, so a mistyped account id never mints a decision. Repeating the current decision is a no-op without an event; a different decision (including reversing an approval) transitions and emits `kyb.<decision>`; identity recomputes live mode from the event |
| `GET/POST /internal/cases…`, `POST /internal/blocklist` | ops | case management, blocklist edits (audited). HTTP shapes are specified with the Stage 2 console, their first network consumer (OQ-T-02); in Stage 1 the same use cases run in-process through the ops CLI (`docker compose exec compliance-api node dist/ops.js …`, platform §3) |

## 6. Events

| Produced | Consumers |
|---|---|
| `evt.compliance.kyb.approved` · `.rejected` · `.needs_info` | identity, notify |
| `evt.compliance.destination.blocked` | identity, notify |
| `evt.compliance.account.suspended` · `.reinstated` | identity, notify |
| `evt.compliance.review.opened` · `.closed` | payments (flag), webhooks (`review.opened/closed`), notify |
| `evt.compliance.policy.updated` | payments |

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

## 8. Configuration

`NP_SANCTIONS_SOURCE_URL` (default `https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/SDN_ADVANCED.XML`, SP-15; `file:` for air-gapped machines), `NP_SANCTIONS_REFRESH` (default `true`; offline stacks set `false` and import with the ops CLI), `NP_SANCTIONS_TIMEOUT_MS` (300000), `NP_OPS_ACTOR` (default actor recorded by the ops CLI; `--actor` overrides, else `cli@<hostname>`), `NP_SCREENING_VENDOR` (`none` Stage 1 · `chainalysis` Stage 2), `NP_SCREENING_VENDOR_KEY_FILE`, `NP_KYB_VENDOR` (`manual` Stage 1 · `sumsub` Stage 2), `NP_KYB_VENDOR_SECRET_FILE`, `NP_KYB_WEBHOOK_SECRET_FILE`.

## 9. Security and privacy

Case notes and KYB metadata are PII → envelope-encrypted; ops access via console only (audited); vendor documents never stored by NearPay (vendor-hosted). Retention per `policies.retention_years`.

## 10. Observability

`…_screenings_total{context,decision,provider,cached}`, `…_screening_duration_seconds`, `…_vendor_unavailable_total`, `…_cases_open{kind,severity}`, `…_sanctions_list_age_hours`. Alerts: list age > 48 h; any `critical` case; vendor unavailable > 5 min.

## 11. Tests

Normalisation per chain family; OFAC XML parser on a fixture; fail-closed threshold both sides; sandbox magic addresses; post-screen flow; KYB manual decision → identity live mode in testbed.

## 12. Stage mapping

| Stage | Scope |
|---|---|
| 1 | local OFAC list + blocklist screening, geo policy (country only locally — no geo headers on localhost), manual KYB decision endpoint (ops script), post-screen consumer (Stage 1 has no Explorer `senders`, so post-screening re-checks `refund_to`/payer accounts only), cases (ops CLI), daily destination re-screen |
| 2 | vendor screening + KYB vendor (hosted flow, webhooks), console case management |

## 13. Ops CLI (Stage 1)

No network path exists for these actions in Stage 1 (Stage 2 moves them to the console). Inside the api container (`docker compose exec compliance-api /nodejs/bin/node dist/ops.js …`): `kyb-show` · `kyb-decide <acct> <decision> --reason [--actor]` · `cases [--all]` · `case-investigate` · `case-close --resolution` · `case-note --note` · `blocklist-add` · `blocklist-remove` · `account-suspend` · `account-reinstate` · `geo-policy` · `geo-policy-set --countries [--regions]` (emits `policy.updated`) · `sanctions-status` · `sanctions-refresh` · `sanctions-import <file | ->` · `screen <type> <subject> [--chain] [--live]`. Output is JSON; errors are JSON on stderr with exit 1 (2 for usage).
