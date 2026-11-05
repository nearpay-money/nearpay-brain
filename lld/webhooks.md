# LLD — `nearpay-webhooks`

> Version 1.2 (error codes named in §5) · Version 1.1 audited (`audits/webhooks.md`) · API conventions §9 · Flows §8 · FR-WH-01…04 · Foundations `00-foundations.md`

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
  status text not null check (status in ('enabled','disabled')), disabled_reason text,
  last_success_at timestamptz, failing_since timestamptz,
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);
create index we_account on webhook_endpoints (account_id, livemode) where status = 'enabled';

create table events (
  id uuid primary key, account_id uuid not null, livemode boolean not null, type text not null,
  source_event_id uuid not null unique, object_type text not null, object_id text not null, object_sequence integer not null,
  data jsonb not null,              -- canonical object snapshot (latest API version shape)
  previous jsonb, request jsonb, created_at timestamptz not null);
create index events_account on events (account_id, livemode, created_at desc);
create index events_type on events (account_id, livemode, type, created_at desc);

create table deliveries (
  id uuid primary key, event_id uuid not null references events(id), endpoint_id uuid not null references webhook_endpoints(id),
  status text not null check (status in ('pending','succeeded','failed','abandoned')),
  attempts smallint not null default 0, next_attempt_at timestamptz, last_status_code smallint, last_error text,
  created_at timestamptz not null default now(), succeeded_at timestamptz, unique (event_id, endpoint_id));
create index deliveries_due on deliveries (next_attempt_at) where status in ('pending','failed');

create table delivery_attempts (
  id uuid primary key, delivery_id uuid not null references deliveries(id), attempt_no smallint not null,
  started_at timestamptz not null, duration_ms integer, status_code smallint, error text, response_excerpt text);
```
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
| `evt.identity.account.updated` | `account.updated` | account (public fields) |
| `evt.identity.settlement_destination.{cooling_off,activated,disabled}` | `settlement_destination.{pending,activated,disabled}` | settlement_destination |
| `evt.compliance.review.{opened,closed}` | `review.{opened,closed}` | review |

Everything else (attempt, execution, internal events) is never exposed. Rendering per endpoint `api_version` is done by version transformers in `@nearpay/spec` (`webhooks/versions/*.ts`).

## 5. API

| Method & path | Auth | Notes |
|---|---|---|
| `POST /v1/webhook_endpoints` | secret key `webhook_endpoints.write` or session developer+ | `{url (https; http allowed only for test mode and localhost), enabled_events, api_version?, description?}` → endpoint + `secret` (`whsec_…`, once) |
| `GET /v1/webhook_endpoints` · `/{id}` · `PATCH` · `DELETE` | same (read/write) | URL changes re-validate SSRF rules |
| `POST /v1/webhook_endpoints/{id}/rotate_secret` | write | `{expire_previous_in_seconds ≤ 86400}` → new secret; both signatures sent during overlap |
| `POST /v1/webhook_endpoints/{id}/test` | write | sends `ping` event |
| `GET /v1/webhook_endpoints/{id}/deliveries` | read | delivery log with attempts |
| `GET /v1/events` · `/{id}` | `events.read` | filters `type`, `created[gte/lte]` |
| `POST /v1/events/{id}/resend` | write | `{endpoint?}` → new delivery |
| `GET /v1/events/stream` | `events.read` (test mode or session) | SSE of new events (CLI `listen`, Stage 3) |
| `GET /v1/webhook_signing_keys` | anonymous | JWKS of Ed25519 keys |

Endpoint create/update errors: `url_invalid` (not https outside test mode/localhost, or fails the SSRF rules) and `event_type_unknown` (an `enabled_events` entry that is not a webhook type); both in the spec error catalogue.

## 6. Algorithms

**Ingest** (`webhooks-ingest`, consumers `webhooks--EVT_*` with filters from §4):
```
tx: inbox.consume(evt.id); m = mapping(evt.subject) or ack-skip
    insert events(source_event_id=evt.id, type, data=evt.data.object, previous, object_sequence=evt.aggregate.version) on conflict do nothing
    for ep in enabled endpoints(account, livemode) where '*' ∈ enabled_events or type ∈ enabled_events:
        insert deliveries(pending, next_attempt_at=now())
```
**Dispatch** (`webhooks-dispatcher`):
```
loop: due = select … where status in (pending, failed) and next_attempt_at <= now() order by next_attempt_at limit 100 for update skip locked
  per-endpoint concurrency ≤ 5 (Valkey semaphore 'wh:<ep>' with 30 s TTL; Valkey down → 1 in flight per endpoint per dispatcher replica)
  body = JSON(render(event, endpoint.api_version)); t = now_unix
  sig = 't='+t+',v1='+hex(hmac(secret, t+'.'+body))[+',v1='+secondary]+',v1a='+b64url(ed25519(t+'.'+body))+',kid='+kid
  POST url via egress proxy (timeout 10 s, no redirects) headers: Content-Type, User-Agent 'NearPay-Webhooks/1.0',
       NearPay-Signature, NearPay-Event-Id, NearPay-Event-Type
  2xx → succeeded; endpoint.last_success_at=now, failing_since=null
  else → attempts++, next_attempt_at = now + SCHEDULE[attempts] (1m,5m,30m,2h,5h,10h, then 12h) ± 10%;
         created_at + 72 h passed → abandoned
         failing_since null → set; now − failing_since > 72 h and no success → endpoint disabled + notify owners
```
Event payload envelope: `api-conventions.md` §9 (object includes `sequence`).

## 7. Events

Produced: `evt.webhooks.endpoint.disabled` (notify). Consumed: see §4 (+ `evt.identity.account.updated` for status).

## 8. Egress and SSRF

All deliveries use the `egress-proxy` container (Smokescreen): deny RFC 1918, loopback, link-local, CGNAT, multicast, IPv6 ULA/link-local and cloud metadata ranges; ports 80/443 only; DNS resolved by the proxy at connect time (no rebinding); `localhost`/`*.localhost` and the Compose host `merchant-demo` are allowed only when `NP_ENV=local` (lets the demo merchant receive webhooks locally); never on the server stage.

## 9. Configuration

`NP_WEBHOOK_SIGNING_KEY_FILE` (Ed25519), `NP_ENDPOINT_SECRET_KEK_FILE`, `NP_EGRESS_PROXY` (`http://egress-proxy:4750`), `NP_DELIVERY_TIMEOUT_MS` (10000), `NP_MAX_CONCURRENCY_PER_ENDPOINT` (5).

## 10. Observability

`…_events_ingested_total{type}`, `…_delivery_latency_seconds` (event created → first attempt), `…_deliveries_total{outcome,code_class}`, `…_endpoints_disabled_total`, `…_due_backlog`. Alert: event freshness SLO (`operations/slos.md`), backlog > 10k.

## 11. Tests

Signature vectors shared with `@nearpay/node` verify helper; retry schedule; abandonment at 72 h (fake clock); endpoint disable; SSRF table (each blocked range); version rendering; ingest idempotency on duplicate domain events.

## 12. Stage mapping

| Stage | Scope |
|---|---|
| 1 | endpoints CRUD, ingest for payments + agentpay events, dispatch with HMAC + Ed25519, retries, events API, signing keys, egress proxy |
| 2 | refunds/account/review events, secret rotation, resend, deliveries API in dashboard |
| 3 | billing/mandate events, `GET /v1/events/stream` for CLI |
