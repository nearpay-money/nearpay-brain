# LLD — `nearpay-webhooks`

> Version 1.5 (step 2.1 contracts: secret rotation and resend §5, `account.updated` and `review.*` types §4/§7, stage map §12) · Version 1.4 (step 1.4, spec 0.1.5: delivery log §5, open `enabled_events` items §5, the producer's `sequence` §4/§6, UUIDv7 delivery ids §3) · Version 1.3 (step 1.2 implementation findings folded: data §3, ingest and dispatch §6, URL policy §5/§8, configuration §9) · Version 1.1 audited (`audits/webhooks.md`) · API conventions §9 · Flows §8 · FR-WH-01…04 · Foundations `00-foundations.md`

## 1. Scope

**Owns:** the public Event log (`evt_`), webhook endpoints (`we_`), deliveries and attempts, signing secrets and the platform Ed25519 webhook key, domain-event → public-event mapping, events SSE stream for the CLI.

## 2. Runtime

| Compose service | Command | Replicas |
|---|---|---|
| `webhooks-api` | `api` | 2 |
| `webhooks-ingest` | `ingest` | 1 (consumers of all domain streams, outbox relay) |
| `webhooks-dispatcher` | `dispatcher` | 2 (deliveries via egress proxy) |

## 3. Data

```sql
create table webhook_endpoints (
  id uuid primary key, account_id uuid not null, livemode boolean not null, url text not null,
  enabled_events text[] not null,              -- ['*'] or explicit public types
  api_version text not null, description text,
  secret_enc bytea not null, secondary_secret_enc bytea, secondary_expires_at timestamptz,
  status text not null check (status in ('enabled','disabled')),
  disabled_reason text check (disabled_reason in ('merchant','failing')),   -- tied to status by a check constraint
  last_success_at timestamptz, failing_since timestamptz,
  deleted_at timestamptz,                      -- soft delete (deliveries keep their foreign key)
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create index we_account on webhook_endpoints (account_id, livemode) where status = 'enabled';

create table webhook_endpoint_transitions (… per foundations §7, append-only …);
-- causes: created, merchant_disabled, merchant_enabled, deleted, failing_72h (actor webhooks-dispatcher), ops_enabled

create table events (
  id uuid primary key, account_id uuid not null, livemode boolean not null, type text not null,
  source_event_id uuid not null unique, object_type text not null, object_id text not null, object_sequence integer not null,
  data jsonb not null,              -- canonical object snapshot (latest API version shape)
  previous jsonb, request jsonb, created_at timestamptz not null);
create index events_account on events (account_id, livemode, created_at desc);
create index events_type on events (account_id, livemode, type, created_at desc);
create index events_created on events (created_at);   -- purge
-- created_at = the domain event's occurred_at; lists are ordered by event id (UUIDv7 at ingest), created[…] filters use created_at

create table deliveries (
  id uuid primary key,              -- UUIDv7 generated at ingest since step 1.4 (earlier rows: random UUIDs); API id whd_…
  event_id uuid not null references events(id), endpoint_id uuid not null references webhook_endpoints(id),
  status text not null check (status in ('pending','succeeded','failed','abandoned')),
  attempts smallint not null default 0, next_attempt_at timestamptz, last_status_code smallint, last_error text,
  created_at timestamptz not null default now(), succeeded_at timestamptz, unique (event_id, endpoint_id));
create index deliveries_due on deliveries (next_attempt_at) where status in ('pending','failed');
create index deliveries_endpoint on deliveries (endpoint_id, created_at desc);

create table delivery_attempts (
  id uuid primary key, delivery_id uuid not null references deliveries(id), attempt_no smallint not null,
  started_at timestamptz not null, duration_ms integer, status_code smallint, error text, response_excerpt text,
  unique (delivery_id, attempt_no));
```
**Delete** is a soft delete: `status = disabled`, `deleted_at` set, a `deleted` transition, and the endpoint's undelivered deliveries become `abandoned` in the same transaction; reads exclude deleted endpoints (`404`). Every status change writes a `webhook_endpoint_transitions` row.
Retention: events, deliveries and attempts 30 days (daily purge).

## 4. Domain → public mapping

| Domain subject | Public type | Object |
|---|---|---|
| `evt.payments.checkout_session.completed` / `.expired` | `checkout.session.completed` / `.expired` | checkout_session |
| `evt.payments.payment_intent.{created,processing,succeeded,payment_failed,requires_review,overpaid,canceled}` | `payment_intent.<same>` | payment_intent |
| `evt.payments.refund.{created,requires_action,succeeded,failed,canceled}` | `refund.<same>` | refund |
| `evt.billing.invoice.{created,finalized,paid,payment_failed,overdue,voided,marked_uncollectible}` | `invoice.<same>` | invoice |
| `evt.billing.subscription.{created,updated,deleted,renewal_upcoming}` | `customer.subscription.<same>` | subscription |
| `evt.billing.mandate.{activated,collected,insufficient_funds,canceled,exhausted}` | `mandate.<same>` | mandate |
| `evt.agentpay.agent_payment.{settled,failed}` | `agent_payment.<same>` | agent_payment |
| `evt.identity.account.updated` | `account.updated` (step 2.1: the catalogue routes it to webhooks) | account (public `Account`) |
| `evt.identity.settlement_destination.{cooling_off,activated,disabled}` | `settlement_destination.{pending,activated,disabled}` | settlement_destination |
| `evt.compliance.review.{opened,closed}` | `review.{opened,closed}` (step 2.1: routed to webhooks; `Review` now lives in `@nearpay/spec`) | review |

Everything else (attempt, execution, internal events) is never exposed. Rendering per endpoint `api_version` is done by version transformers in `@nearpay/spec` (`webhooks/versions/*.ts`). Since spec 0.1.5 CheckoutSession and AgentPayment carry their own `sequence` too (payments and agentpay send their aggregate version). The producer's `sequence` is preferred: ingest stores it as `object_sequence` and rendering keeps it; only an object without a valid one (a producer older than 0.1.5) gets `aggregate.version` (api-conventions §9).

## 5. API

| Method & path | Auth | Notes |
|---|---|---|
| `POST /v1/webhook_endpoints` | secret key `webhook_endpoints.write` or session developer+ | `{url (URL policy below), enabled_events, api_version?, description?}` → endpoint + `secret` (`whsec_…`, once); `*` anywhere in `enabled_events` normalises to `["*"]`, duplicates are dropped. The idempotent response (it carries the secret) is stored encrypted with the endpoint-secret KEK |
| `GET /v1/webhook_endpoints` · `/{id}` · `PATCH` · `DELETE` | same (read/write) | URL changes re-validate SSRF rules |
| `POST /v1/webhook_endpoints/{id}/rotate_secret` (`rotateWebhookEndpointSecret`, step 2.1) | write | `{expire_previous_in_seconds 0–86400, default 86400}` → endpoint with the new `secret` (once) and `previous_secret_expires_at`; during the overlap the current secret moves to `secondary_secret_enc` and every delivery carries both `v1=` signatures; rotating again during an overlap replaces the secondary with the then-current secret; `0` drops the previous secret at once; the idempotent response is stored encrypted like create's |
| `POST /v1/webhook_endpoints/{id}/test` | write | stores a `ping` event (listed by `GET /v1/events`, `object_sequence 0`, no `sequence` injected, `request` = the API request) and delivers it to that endpoint regardless of its subscriptions; disabled endpoint → `409 state_conflict` |
| `GET /v1/webhook_endpoints/{id}/deliveries` | read | delivery log (spec 0.1.5, Stage 1): cursor list of `WebhookDelivery` summaries (`whd_…`: `webhook_endpoint`, `event`, `event_type`, `status`, `attempts`, `last_status_code`, `last_error`, `next_attempt_at` only while an attempt is due, `succeeded_at`), newest first, last 30 days (hidden before the purge, like events). Keyset paging on `(created_at, id)` because pre-1.4 rows have random UUIDs; a cursor must be a delivery of the same endpoint (`400` otherwise). The endpoint must be the caller's, in its mode, not deleted (`404`) |
| `GET /v1/events` · `/{id}` | `events.read` | filters `type`, `created[gte/lte]`; rendered in the latest API version (`2026-11-01`: webhooks has no outbound edge to read the account's pinned version); events older than 30 days are hidden even before the purge runs |
| `POST /v1/events/{id}/resend` (`resendEvent`, step 2.1) | write | `{endpoint?}` → `202 EventResend {event, deliveries[]}`: one new delivery (UUIDv7, fresh schedule) per endpoint — the given one (must be the caller's, same mode, enabled, else `409 state_conflict`; subscriptions ignored, like `test`) or every enabled endpoint subscribed to the type; `deliveries.unique(event_id, endpoint_id)` becomes `unique(event_id, endpoint_id, resend_no)` with `resend_no smallint not null default 0`; events older than 30 days are `404` |
| `GET /v1/events/stream` | `events.read` (test mode or session) | SSE of new events (CLI `listen`, Stage 3) |
| `GET /v1/webhook_signing_keys` | anonymous | JWKS of Ed25519 keys |

Endpoint create/update errors: `url_invalid` (fails the URL policy) and `event_type_unknown` (an `enabled_events` entry that is not a webhook type); both in the spec error catalogue. Since spec 0.1.5 request items are any string of 1–100 chars, so the service check is the only one: the first entry that is neither `*` nor a webhook type is `400 event_type_unknown` with `param enabled_events[i]`, even when `*` is also listed (types are case-sensitive).

**URL policy** (checked at the API; DNS is not resolved there — the proxy resolves at connect time, §8): live mode requires `https`; test mode also allows `http` for any public host; ports 80/443 only (local hosts: any); credentials in the URL refused; IP literals refused in RFC 1918, loopback, link-local/metadata, CGNAT, `0/8`, IETF and TEST-NETs, 6to4 relay, benchmarking, multicast, `240/4`, IPv6 `::`/`::1`, discard, documentation, ULA, link/site-local and multicast ranges, including IPv4 embedded in IPv6 (mapped, compatible, NAT64); single-label hosts (Compose service names) and `.local`, `.internal`, `.localdomain`, `.home.arpa`, `.intranet`, `.corp`, `.lan` refused. The local development hosts — `localhost`, `*.localhost`, loopback literals and the Compose host `merchant-demo` — are allowed (any port, `http`) only when `NP_ENV` is `local` or `test`, never on the server stage.

## 6. Algorithms

**Ingest** (`webhooks-ingest`, consumers `webhooks--EVT_*` with filters from §4):
```
tx: inbox.consume(evt.id); m = mapping(evt.subject) or ack-skip
    envelope without account/livemode, or data.object invalid against the type's public schema → dead letter ('consumer:webhooks') + ack
    insert events(source_event_id=evt.id, type, data=evt.data.object, previous,
                  object_sequence=evt.data.object.sequence (integer ≥ 0) ?? evt.aggregate.version,
                  created_at=evt.occurred_at) on conflict do nothing
    for ep in endpoints(account, livemode) enabled at ingest time where '*' ∈ enabled_events or type ∈ enabled_events:
        insert deliveries(id=uuidv7(), pending, next_attempt_at=now())   // disabled endpoints get none; no backfill on re-enable
```
**Dispatch** (`webhooks-dispatcher`):
```
loop: claim = update deliveries set next_attempt_at = now + lease from (select … where status in (pending, failed)
        and next_attempt_at <= now order by next_attempt_at limit n for update skip locked)   // lease = delivery timeout + 50 s
      (a lease instead of row locks held across the HTTP call: a crashed dispatcher's claims return by themselves and
       replicas never send the same attempt twice; schedule and claims use the process clock, injectable for tests)
  at most NP_DISPATCH_CONCURRENCY (50) deliveries in flight per replica
  per-endpoint concurrency ≤ 5 (Valkey sorted set 'wh:<ep>' whose members expire after 30 s, acquired by a Lua script;
       Valkey down → 1 in flight per endpoint per dispatcher replica); endpoint at its limit → postpone the claimed
       delivery 0.5–2 s without counting an attempt
  body = JSON(render(event, endpoint.api_version)); t = now_unix
  sig = 't='+t+',v1='+hex(hmac(secret, t+'.'+body))[+',v1='+secondary]+',v1a='+b64url(ed25519(t+'.'+body))+',kid='+kid
  POST url via egress proxy (timeout 10 s, no redirects) headers: Content-Type 'application/json; charset=utf-8',
       User-Agent 'NearPay-Webhooks/1.0', NearPay-Signature, NearPay-Event-Id, NearPay-Event-Type
  2xx → succeeded; endpoint.last_success_at=now, failing_since=null
  else → attempts++, next_attempt_at = now + SCHEDULE[attempts] (1m,5m,30m,2h,5h,10h, then 12h) ± 10%;
         created_at + 72 h passed at a failed attempt → abandoned
         failing_since null → set; now − failing_since > 72 h on an enabled endpoint → disabled (reason 'failing'),
         transition 'failing_72h', outbox evt.webhooks.endpoint.disabled with previous {status: 'enabled'} (notify)
  proxy refusals (non-200 CONNECT, or X-Smokescreen-Error on an absolute-form response) and timeouts are recorded as
  attempts with status_code null and an error — never as the merchant's status
```
A merchant or ops re-enable clears `failing_since` (a new streak). Job `deliveries.abandon` (every 10 min) abandons deliveries past `created_at + 72 h` that the dispatcher never reaches (disabled endpoints). Default signing `kid` = `wh-` + 12 hex of SHA-256(raw public key); `NP_WEBHOOK_SIGNING_KID` overrides (deploy's `keys.env` derives `local-…` for the same key).
Event payload envelope: `api-conventions.md` §9 (object includes `sequence`).

## 7. Events

Produced: `evt.webhooks.endpoint.disabled` (notify). Consumed: see §4 (`evt.identity.account.updated` from Stage 2).

## 8. Egress and SSRF

All deliveries use the `egress-proxy` container (Smokescreen): deny RFC 1918, loopback, link-local, CGNAT, multicast, IPv6 ULA/link-local and cloud metadata ranges; ports 80/443 only; DNS resolved by the proxy at connect time (no rebinding); `localhost`/`*.localhost` and the Compose host `merchant-demo` are allowed only when `NP_ENV=local` (lets the demo merchant receive webhooks locally); never on the server stage.

## 9. Configuration

`NP_WEBHOOK_SIGNING_KEY_FILE` (Ed25519), `NP_WEBHOOK_SIGNING_KID` (optional, §6), `NP_ENDPOINT_SECRET_KEK_FILE`, `NP_EGRESS_PROXY` (`http://egress-proxy:4750`; **required** when `NP_ENV=server` — the dispatcher refuses to start without it; unset locally/in tests means direct delivery), `NP_DELIVERY_TIMEOUT_MS` (10000), `NP_MAX_CONCURRENCY_PER_ENDPOINT` (5), `NP_DISPATCH_CONCURRENCY` (50).

## 10. Observability

`…_events_ingested_total{type}`, `…_delivery_latency_seconds` (event created → first attempt), `…_deliveries_total{outcome,code_class}`, `…_endpoints_disabled_total`, `…_due_backlog`. Alert: event freshness SLO (`operations/slos.md`), backlog > 10k.

## 11. Tests

Signature vectors shared with `@nearpay/node` verify helper; retry schedule; abandonment at 72 h (fake clock); endpoint disable; SSRF table (each blocked range); version rendering; ingest idempotency on duplicate domain events.

## 12. Stage mapping

| Stage | Scope |
|---|---|
| 1 | endpoints CRUD, ingest for payments + agentpay events, dispatch with HMAC + Ed25519, retries, events API, signing keys, egress proxy, delivery log (`GET …/deliveries`, step 1.4, spec 0.1.5) |
| 2 | refunds (payments' types) / account / review events, secret rotation, resend (step 2.1 contracts) |
| 3 | billing/mandate events, `GET /v1/events/stream` for CLI |
