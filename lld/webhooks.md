# LLD — `nearpay-webhooks`

> Version 1.11 (step 4.2 implementation, spec 0.4.0: mandate ingest — §"Step 4.2 implementation") · Version 1.10 (step 4.1, spec 0.4.0: `mandate.*` types §4) · Version 1.9 (step 3.2 implementation: stream tail dedupe window, slots as a Valkey sorted set, close reasons, `drain()` semantics, NOTIFY from `…/test` too §5.1; billing ingest guards §4 — §"Step 3.2 implementation") · Version 1.8 (step 3.1, spec 0.3.0: events SSE stream `streamEvents` — §5.1, stage map §12) · Version 1.7 (spec 0.2.1 implementation: secret_rotated actor and version — §"Spec 0.2.1 contract changes") · Spec 0.2.1 update (spec 0.2.1: `evt.webhooks.endpoint.secret_rotated` — §"Spec 0.2.1 contract changes") · Version 1.6 (step 2.2 implementation: mode-less `account.updated` → one event per mode, review skip, INV-8 mode check §4/§6; `resend_no` and events unique per mode §3; rotation transition, `secrets.expire` job, resend targets/limits §5–§6) · Version 1.5 (step 2.1 contracts, spec 0.2.0: secret rotation and resend §5, `account.updated`, `review.*` (rendered as `PublicReview`) and `refund.pending` types §4/§7, stage map §12) · Version 1.4 (step 1.4, spec 0.1.5: delivery log §5, open `enabled_events` items §5, the producer's `sequence` §4/§6, UUIDv7 delivery ids §3) · Version 1.3 (step 1.2 implementation findings folded: data §3, ingest and dispatch §6, URL policy §5/§8, configuration §9) · Version 1.1 audited (`audits/webhooks.md`) · API conventions §9 · Flows §8 · FR-WH-01…04 · Foundations `00-foundations.md`

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
-- causes: created, merchant_disabled, merchant_enabled, deleted, failing_72h (actor webhooks-dispatcher), ops_enabled,
--         secret_rotated (step 2.2: from_state = to_state, evidence {previous_secret_expires_at})

create table events (
  id uuid primary key, account_id uuid not null, livemode boolean not null, type text not null,
  source_event_id uuid not null, object_type text not null, object_id text not null, object_sequence integer not null,
  data jsonb not null,              -- canonical object snapshot (latest API version shape)
  previous jsonb, request jsonb, created_at timestamptz not null,
  unique (source_event_id, livemode));   -- step 2.2 (migration 0002): a mode-less domain event yields one event per mode
create index events_account on events (account_id, livemode, created_at desc);
create index events_type on events (account_id, livemode, type, created_at desc);
create index events_created on events (created_at);   -- purge
-- created_at = the domain event's occurred_at; lists are ordered by event id (UUIDv7 at ingest), created[…] filters use created_at

create table deliveries (
  id uuid primary key,              -- UUIDv7 generated at ingest since step 1.4 (earlier rows: random UUIDs); API id whd_…
  event_id uuid not null references events(id), endpoint_id uuid not null references webhook_endpoints(id),
  resend_no smallint not null default 0,   -- step 2.2: 0 = ingest/test delivery, 1, 2, … = resends to this endpoint
  status text not null check (status in ('pending','succeeded','failed','abandoned')),
  attempts smallint not null default 0, next_attempt_at timestamptz, last_status_code smallint, last_error text,
  created_at timestamptz not null default now(), succeeded_at timestamptz, unique (event_id, endpoint_id, resend_no));
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
| `evt.payments.refund.{created,requires_action,pending,succeeded,failed,canceled}` | `refund.<same>` (`pending`: merchant deposit detected for the current leg, step 2.1) | refund |
| `evt.billing.invoice.{created,finalized,paid,payment_failed,overdue,voided,marked_uncollectible}` | `invoice.<same>` | invoice |
| `evt.billing.subscription.{created,updated,deleted,renewal_upcoming}` | `customer.subscription.<same>` | subscription |
| `evt.billing.mandate.{activated,collected,insufficient_funds,canceled,exhausted,updated,unlinked,released}` | `mandate.<same>` (Stage 4, spec 0.4.0; billing §14 d10) | mandate (public `Mandate`, `sequence` from billing) |
| `evt.agentpay.agent_payment.{settled,failed}` | `agent_payment.<same>` | agent_payment |
| `evt.identity.account.updated` | `account.updated` (step 2.1: the catalogue routes it to webhooks) | account (public `Account`). Identity emits it with `livemode: null` (an account is neither test nor live): step 2.2 stores **one public event per mode** (distinct `evt_` ids, same `source_event_id`), each fanned out to that mode's endpoints and listed by that mode's events API |
| `evt.identity.settlement_destination.{cooling_off,activated,disabled}` | `settlement_destination.{pending,activated,disabled}` | settlement_destination |
| `evt.compliance.review.{opened,closed}` | `review.{opened,closed}` (step 2.1: routed to webhooks) | review — the bus `Review` is rendered with `toPublicReview()` into `PublicReview` at ingest (unix times; a review of a rails execution is shown as a review of its payment intent via `Review.payment_intent`; no `payment_intent` → no public event at all, acked without a dead letter; a payload that is not a bus `Review` → dead letter) |

Everything else (attempt, execution, internal events) is never exposed. Rendering per endpoint `api_version` is done by version transformers in `@nearpay/spec` (`webhooks/versions/*.ts`). Since spec 0.1.5 CheckoutSession and AgentPayment carry their own `sequence` too (payments and agentpay send their aggregate version). The producer's `sequence` is preferred: ingest stores it as `object_sequence` and rendering keeps it; only an object without a valid one (a producer older than 0.1.5) gets `aggregate.version` (api-conventions §9).

## 5. API

| Method & path | Auth | Notes |
|---|---|---|
| `POST /v1/webhook_endpoints` | secret key `webhook_endpoints.write` or session developer+ | `{url (URL policy below), enabled_events, api_version?, description?}` → endpoint + `secret` (`whsec_…`, once); `*` anywhere in `enabled_events` normalises to `["*"]`, duplicates are dropped. The idempotent response (it carries the secret) is stored encrypted with the endpoint-secret KEK |
| `GET /v1/webhook_endpoints` · `/{id}` · `PATCH` · `DELETE` | same (read/write) | URL changes re-validate SSRF rules |
| `POST /v1/webhook_endpoints/{id}/rotate_secret` (`rotateWebhookEndpointSecret`, step 2.1) | write | `{expire_previous_in_seconds 0–86400, default 86400}` → endpoint with the new `secret` (once) and `previous_secret_expires_at`; during the overlap the current secret moves to `secondary_secret_enc` and every delivery carries both `v1=` signatures; rotating again during an overlap replaces the secondary with the then-current secret; `0` drops the previous secret at once; the idempotent response is stored encrypted like create's. Step 2.2: allowed on a disabled endpoint (a leaked secret must be replaceable; deleted → `404`); writes a `secret_rotated` transition; the previous secret signs while `now < secondary_expires_at` (the expiry instant is outside); reads show `previous_secret_expires_at` only inside the overlap (null otherwise); job `secrets.expire` (10 min) erases expired secondaries |
| `POST /v1/webhook_endpoints/{id}/test` | write | stores a `ping` event (listed by `GET /v1/events`, `object_sequence 0`, no `sequence` injected, `request` = the API request) and delivers it to that endpoint regardless of its subscriptions; disabled endpoint → `409 state_conflict` |
| `GET /v1/webhook_endpoints/{id}/deliveries` | read | delivery log (spec 0.1.5, Stage 1): cursor list of `WebhookDelivery` summaries (`whd_…`: `webhook_endpoint`, `event`, `event_type`, `status`, `attempts`, `last_status_code`, `last_error`, `next_attempt_at` only while an attempt is due, `succeeded_at`), newest first, last 30 days (hidden before the purge, like events). Keyset paging on `(created_at, id)` because pre-1.4 rows have random UUIDs; a cursor must be a delivery of the same endpoint (`400` otherwise). The endpoint must be the caller's, in its mode, not deleted (`404`) |
| `GET /v1/events` · `/{id}` | `events.read` | filters `type`, `created[gte/lte]`; rendered in the latest API version (`2026-11-01`: webhooks has no outbound edge to read the account's pinned version); events older than 30 days are hidden even before the purge runs |
| `POST /v1/events/{id}/resend` (`resendEvent`, step 2.1) | write | `{endpoint?}` → `202 EventResend {event, deliveries[]}`: one new delivery (UUIDv7, fresh schedule, `created_at` now so the 72 h window restarts) per endpoint — the given one (not the caller's or other mode → `404 resource_missing` `param endpoint`; disabled → `409 state_conflict`; subscriptions ignored, like `test`) or every enabled endpoint subscribed to the type (step 2.2: a `ping` only to the endpoint it tested, never to `*` subscribers; none → `202` with `deliveries: []`); `deliveries.unique(event_id, endpoint_id)` becomes `unique(event_id, endpoint_id, resend_no)` with `resend_no smallint not null default 0`; step 2.2: a resend takes `max(resend_no of the pair, 0) + 1` while holding `FOR UPDATE` on the event row (concurrent resends serialise); a pair at 32 767 → `409 state_conflict`; events older than 30 days are `404` |
| `GET /v1/events/stream` (`streamEvents`, Stage 3) | secret key, restricted `events.read`, or session — API keys test mode only | SSE of public events (CLI `listen`, dashboard live tail) — §5.1 |
| `GET /v1/webhook_signing_keys` | anonymous | JWKS of Ed25519 keys |

### 5.1 Events stream (Stage 3, spec 0.3.0)

`GET /v1/events/stream?types=&starting_after=` (`streamEvents`), served by `webhooks-api` with kit `ctx.sse(limits)`.

- **Auth and mode (INV-8).** `secret_key`, `restricted_key:events.read` or `session`. The mode is the credential's: an API key streams **test mode only** (a live key → `403 livemode_forbidden`, as the LLD's "test mode or session" rule); a dashboard session streams the mode of its `NearPay-Mode` header (live tail in the dashboard). There is no `livemode` query parameter: the credential decides, so a stream can never mix modes.
- **Filters.** `types` = comma-separated webhook types (`WEBHOOK_TYPES`, so Stage 3 billing types `invoice.*` and `customer.subscription.*` stream as soon as they are in the catalogue) or `*` (default). The first unknown entry → `400 event_type_unknown`, `param types[i]` (same rule as endpoint `enabled_events`, case-sensitive). `ping` events (endpoint tests) are streamed too, so `nearpay listen` shows them.
- **Frames.** `retry: 3000`; `event: ready` (`EventStreamReady {livemode, account, types, resumed_after, expires_at}`) once; then `id: <evt_…>` + `event: event` + `data: <Event>` per event — the stored canonical object rendered in the latest API version (`2026-11-01`), byte-for-byte what `GET /v1/events/{id}` returns; a comment heartbeat every 15 s. Streamed events are not deliveries: not signed, not retried, no effect on endpoints (`nearpay listen` re-signs locally before forwarding).
- **Resume.** Cursor = `Last-Event-ID` header (sent by `EventSource` on reconnect) or `starting_after` (header wins; kit `resumeCursor`). With a cursor: replay every later event of the account and mode in id order (`id > cursor`, events of the last 30 days, pages of 100, awaiting `drain()` between pages), then switch to live. The cursor must be an event of this account and mode within retention; otherwise the stream sends `event: reset` (`EventStreamReset {reason: cursor_unknown | cursor_expired, cursor}`) and continues live only (a 4xx would make `EventSource` give up). A malformed id is `400 invalid_request`. Without a cursor: live only.
- **Live tail.** Ingest gives events UUIDv7 ids generated when the row is built, so ids follow generation order, not commit order. Each stream polls `select … from events where account_id = $1 and livemode = $2 and id > $cursor [and type = any($types)] order by id limit 100` every 1 s, and also wakes on Postgres `NOTIFY events_<mode>` that ingest sends after commit (payload = account id; the poll is the source of truth). To not skip an event whose id is lower than one already sent but committed later, the live query only returns rows with `inserted_at < now() − 2 s` (a **2 s settle window**; ingest transactions are short) and re-reads the last 10 s behind its cursor, skipping ids it already sent (kept in memory for 10 s); at-least-once, in id order per page. Clients dedupe by `id`.
- **Limits and backpressure.** ≤ 5 concurrent streams per (account, mode) (Valkey counter with a 30 s TTL refreshed on each heartbeat; without Valkey, per replica) → `429 rate_limited` with `Retry-After: 5`. Each stream ends after 1 h (`maxDurationMs`; `ready.expires_at` says when) — the client reconnects with `Last-Event-ID`. A client that keeps > 1 MiB unsent for 30 s is closed (`maxBufferedBytes`, `slowClientMs`). Shutdown ends every stream (kit). Metrics: `…_event_streams{mode}` (gauge), `…_event_stream_events_total`, `…_event_stream_closed_total{reason}`.
- **Data.** Migration `0004_event_stream`: `alter table events add column inserted_at timestamptz not null default clock_timestamp()` + index `events_stream (account_id, livemode, id)`; ingest issues `pg_notify('events_' || livemode, account_id::text)` in the same transaction (delivered at commit).

Endpoint create/update errors: `url_invalid` (fails the URL policy) and `event_type_unknown` (an `enabled_events` entry that is not a webhook type); both in the spec error catalogue. Since spec 0.1.5 request items are any string of 1–100 chars, so the service check is the only one: the first entry that is neither `*` nor a webhook type is `400 event_type_unknown` with `param enabled_events[i]`, even when `*` is also listed (types are case-sensitive).

**URL policy** (checked at the API; DNS is not resolved there — the proxy resolves at connect time, §8): live mode requires `https`; test mode also allows `http` for any public host; ports 80/443 only (local hosts: any); credentials in the URL refused; IP literals refused in RFC 1918, loopback, link-local/metadata, CGNAT, `0/8`, IETF and TEST-NETs, 6to4 relay, benchmarking, multicast, `240/4`, IPv6 `::`/`::1`, discard, documentation, ULA, link/site-local and multicast ranges, including IPv4 embedded in IPv6 (mapped, compatible, NAT64); single-label hosts (Compose service names) and `.local`, `.internal`, `.localdomain`, `.home.arpa`, `.intranet`, `.corp`, `.lan` refused. The local development hosts — `localhost`, `*.localhost`, loopback literals and the Compose host `merchant-demo` — are allowed (any port, `http`) only when `NP_ENV` is `local` or `test`, never on the server stage.

## 6. Algorithms

**Ingest** (`webhooks-ingest`, consumers `webhooks--EVT_*` with filters from §4):
```
tx: inbox.consume(evt.id); m = mapping(evt.subject) or ack-skip
    envelope without account → dead letter ('consumer:webhooks') + ack
    modes = [evt.livemode], or [false, true] when livemode is null and the type is mode-less (account.updated); else dead letter
    per mode: object = public object (reviews: toPublicReview(Review, mode); null → ack, no event);
        invalid against the type's public schema, or object.livemode ≠ mode (INV-8) → dead letter + ack
        (all modes are rendered and checked before anything is inserted)
    per mode: insert events(source_event_id=evt.id, livemode=mode, type, data=object, previous,
                  object_sequence=object.sequence (integer ≥ 0) ?? evt.aggregate.version,
                  created_at=evt.occurred_at) on conflict (source_event_id, livemode) do nothing
        for ep in endpoints(account, mode) enabled at ingest time where '*' ∈ enabled_events or type ∈ enabled_events:
            insert deliveries(id=uuidv7(), resend_no=0, pending, next_attempt_at=now())   // disabled endpoints get none; no backfill on re-enable
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
  t = now_unix; secrets = [current] + [secondary if now < secondary_expires_at]   // one instant for both (step 2.2)
  body = JSON(render(event, endpoint.api_version))
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
A merchant or ops re-enable clears `failing_since` (a new streak). Job `secrets.expire` (every 10 min, step 2.2) nulls `secondary_secret_enc`/`secondary_expires_at` once expired. Job `deliveries.abandon` (every 10 min) abandons deliveries past `created_at + 72 h` that the dispatcher never reaches (disabled endpoints). Default signing `kid` = `wh-` + 12 hex of SHA-256(raw public key); `NP_WEBHOOK_SIGNING_KID` overrides (deploy's `keys.env` derives `local-…` for the same key).
Event payload envelope: `api-conventions.md` §9 (object includes `sequence`).

## 7. Events

Produced: `evt.webhooks.endpoint.disabled` (notify). Consumed: see §4 (`evt.identity.account.updated` from Stage 2).

## 8. Egress and SSRF

All deliveries use the `egress-proxy` container (Smokescreen): deny RFC 1918, loopback, link-local, CGNAT, multicast, IPv6 ULA/link-local and cloud metadata ranges; ports 80/443 only; DNS resolved by the proxy at connect time (no rebinding); `localhost`/`*.localhost` and the Compose host `merchant-demo` are allowed only when `NP_ENV=local` (lets the demo merchant receive webhooks locally); never on the server stage.

## 9. Configuration

`NP_WEBHOOK_SIGNING_KEY_FILE` (Ed25519), `NP_WEBHOOK_SIGNING_KID` (optional, §6), `NP_VALKEY_URL` (dispatcher: per-endpoint semaphore; api, step 3.2: events-stream slots; optional), `NP_ENDPOINT_SECRET_KEK_FILE`, `NP_EGRESS_PROXY` (`http://egress-proxy:4750`; **required** when `NP_ENV=server` — the dispatcher refuses to start without it; unset locally/in tests means direct delivery), `NP_DELIVERY_TIMEOUT_MS` (10000), `NP_MAX_CONCURRENCY_PER_ENDPOINT` (5), `NP_DISPATCH_CONCURRENCY` (50).

## 10. Observability

`…_events_ingested_total{type}`, `…_delivery_latency_seconds` (event created → first attempt), `…_deliveries_total{outcome,code_class}`, `…_endpoints_disabled_total`, `…_due_backlog`. Alert: event freshness SLO (`operations/slos.md`), backlog > 10k.

## 11. Tests

Signature vectors shared with `@nearpay/node` verify helper; property tests (step 2.2): rotation window (a delivery verifies under exactly the current secret plus the previous one strictly before its expiry, at every instant, over random rotation sequences) and resend allocation (no `(event, endpoint, resend_no)` twice; contiguous numbers); retry schedule; abandonment at 72 h (fake clock); endpoint disable; SSRF table (each blocked range); version rendering; ingest idempotency on duplicate domain events.

## 12. Stage mapping

| Stage | Scope |
|---|---|
| 1 | endpoints CRUD, ingest for payments + agentpay events, dispatch with HMAC + Ed25519, retries, events API, signing keys, egress proxy, delivery log (`GET …/deliveries`, step 1.4, spec 0.1.5) |
| 2 | refunds (payments' types) / account / review events, secret rotation, resend (step 2.1 contracts) |
| 3 | billing events (`invoice.*`, `customer.subscription.*`), `GET /v1/events/stream` (§5.1, spec 0.3.0) for the CLI and the dashboard live tail |
| 4 | `mandate.*` events (8 types, public `Mandate`; spec 0.4.0, billing §14 d10) — no new routes |

## Spec 0.2.1 contract changes (step 2.2/2.3 change requests, `nearpay-spec/CHANGELOG.md`)

- `rotateWebhookEndpointSecret` also emits **`evt.webhooks.endpoint.secret_rotated`** (`EndpointSecretRotatedEvent {endpoint_id, account_id, url, previous_secret_expires_at, actor_user_id}`, never the secret) through the outbox in the same transaction; consumer notify.

**Implementation (spec 0.2.1, v1.7).**

- `actor_user_id` is the caller's `usr_…` when the rotation came from a dashboard session (`ptype: session`), else null (API keys). The aggregate is `webhook_endpoint` and its version is the endpoint row version after the rotation. `previous_secret_expires_at` is the stored expiry in ISO form (millisecond precision). An idempotent replay returns the stored response and writes no second event.

## Step 3.2 implementation (spec 0.3.0, kit 0.2.1 with the kit 0.3.0 SSE API isolated, v1.9)

**Billing ingest (§4).** The 12 billing subjects (8 `invoice.*`, 4 `customer.subscription.*`) are ingested like every other type; `evt.billing.invoice.reminder` has no public type (notify only), mandates followed in Stage 4 (§"Step 4.2 implementation"). Two guards at ingest, both dead letters (`consumer:webhooks`), never a public event:
- an invoice whose `hosted_invoice_url` is not null (the URL carries the invoice secret; `InvoiceEvent` makes it null on the bus, the public `Invoice` schema alone would accept it);
- INV-8 is checked on every nested `livemode`, not only the object's own (a subscription's item `price` snapshots carry one).

**Events stream (§5.1) — decisions the LLD left open.**
- *Tail dedupe.* "Kept in memory for 10 s" is implemented as: the ids sent inside the re-read window `(floor, cursor]` with `floor = max(origin, uuidv7_floor(time(cursor) − 10 s))` are remembered until the floor passes them (wall-clock expiry would resend an id whose window is still re-read when no newer event moves the cursor). The re-read query excludes them in SQL (`not (id = any($sent))`) so a full page of already-sent ids cannot starve new rows; the forward query (`id > cursor`) and the re-read run on every poll, both with the settle filter (`inserted_at < now() − 2 s`) and the 30-day `created_at` filter. Guarantee (property-tested on a model with random commit delays and poll instants): every event once and in id order while ingest transactions are shorter than the settle window; every event once (order may differ) up to 10 s; never a duplicate and nothing at or below the start cursor whatever the timings.
- *Live-only start.* `origin` = UUIDv7 floor of the open instant (no cursor, or after `reset`): nothing generated before the stream opened is sent, also not by the re-read.
- *Cursor reasons.* A cursor of this account and mode older than 30 days, or an id this account does not have whose UUIDv7 time is older than 30 days (purged), is `cursor_expired`; anything else (other account or mode, random id) is `cursor_unknown`. A malformed id is rejected by the kit's header/query validation (`400 invalid_request`).
- *NOTIFY.* `POST /v1/webhook_endpoints/{id}/test` also sends `pg_notify` (pings are streamed). The api holds one LISTEN connection per process (`events_true`, `events_false`), re-established every 1 s after a failure; a notification schedules a poll right after the settle window (`+ 2.05 s`) because the new row is hidden until then.
- *Slots.* The "Valkey counter with a 30 s TTL" is a sorted set `wh:stream:<test|live>:<account uuid>` with one member per stream scored by its expiry (acquire: Lua `ZREMRANGEBYSCORE` stale, `ZCARD < 5`, `ZADD`), refreshed every heartbeat (15 s); a crashed replica's slots lapse after 30 s, which a plain counter cannot do. Valkey unset or down → 5 per replica. `NP_VALKEY_URL` is now an optional `webhooks-api` setting (§9).
- *Slow readers.* Closing a reader that does not read must also destroy the socket (`end()` cannot flush to it); `…_event_stream_closed_total{reason}` uses `client`, `max_duration`, `slow_client` (more than 1 MiB unsent at the last heartbeat sample), `shutdown`, `error`.
- *Backpressure.* `drain()` waits only when the last write returned false (`writableNeedDrain`); waiting whenever `writableLength > 0` hangs, because Node emits `drain` only after a refused write (kit 0.3.0 change request).
- *Rendering.* A stored event that no longer renders (schema mismatch) is skipped with an `ALERT` log instead of closing the stream (a close would repeat on every reconnect).
- *Kit.* Until kit 0.3.0 is published, `src/api/sse.ts` provides `ctx.sse(limits)` (duration, slow reader), `buffered`/`drain()` and `resumeCursor` on kit 0.2.1; it is the only file to change when switching.

## Step 4.2 implementation (spec 0.4.0, v1.11)

- **Mandate ingest (§4).** The ingest stage is 4: the 8 `evt.billing.mandate.*` subjects (`activated`, `collected`, `insufficient_funds`, `canceled`, `exhausted`, `updated`, `unlinked`, `released`) become `mandate.<same>` events whose `data.object` is the public `Mandate` billing published, validated against the spec schema; `sequence` is the mandate's own (merchants discard lower sequences). `evt.rails.vault.event` (raw vault events, billing's input) is never public. No new route or table: mandate events are listed by `GET /v1/events`, filtered by `type=`, subscribable in `enabled_events`, resent and streamed (§5.1) like every other type (`object_type` = the aggregate type `mandate`).
- **Guards.** The nested-livemode check (INV-8) applies unchanged: a `Mandate` whose `livemode` (sandbox vault stand-in = `false`) differs from the envelope's is dead-lettered (`consumer:webhooks`), never a public event, never delivered to an endpoint of the other mode. A payload that does not parse as `Mandate` is dead-lettered the same way. Mandates carry no secret (payer account and on-chain terms are public chain data), so no extra field guard as for `invoice.hosted_invoice_url`.
- **Tests.** Unit (mapping, pass-through per mode, guard, Event rendering incl. `previous_attributes` for `mandate.updated`), contract (40 ingested subjects, every type renders as the spec `Event`), integration `test/integration/stage4.test.ts` (signed delivery of all 8 types per subscription and mode, `GET /v1/events?type=`, dead letter on a mode mismatch, SSE tail per `types=` byte-for-byte like `GET /v1/events/{id}`).
