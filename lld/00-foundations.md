# LLD 00 — Foundations (`nearpay-spec`, `nearpay-kit`, conventions)

> Version 1.0 · Applies to every service and client. Component LLDs only state deviations from this file.

## 1. Scope

Defines the toolchain, the contract system (`nearpay-spec`), the shared runtime libraries (`nearpay-kit`), and the conventions for IDs, money, databases, events, internal auth, HTTP, errors, configuration, observability, containers and tests.

## 2. Toolchain baseline

| Area | Choice (major versions pinned; minor/patch via lockfile) |
|---|---|
| Runtime | Node.js 22 LTS, TypeScript 5 `strict`, ESM only, pnpm 9 (`packageManager` field) |
| HTTP server | Fastify 5 with `fastify-type-provider-zod` |
| Validation / contracts | zod 4 (authored in `nearpay-spec`), OpenAPI 3.1 emitted with `zod-openapi` |
| SQL | PostgreSQL 17, `pg` 8, Kysely (typed query builder), plain-SQL migrations |
| Events | NATS 2.10+ JetStream via `@nats-io/jetstream` |
| Cache | Valkey 8 via `iovalkey` |
| Crypto | `@noble/curves` (ed25519, secp256k1), `@noble/hashes` (sha256, keccak), `bs58` |
| NEAR | `@near-js/*` (providers, transactions, signers), `@hot-labs/near-connect` (browser wallets) |
| EVM | `viem` 2 |
| 1Click | `@defuse-protocol/one-click-sdk-typescript` ≥ 0.1.24 (types + `verifyQuoteSignature`) |
| Logging / telemetry | pino 9; OpenTelemetry JS SDK (traces, metrics) exporting OTLP |
| Tests | Vitest 2, fast-check (property), Testcontainers (Postgres, NATS, Valkey), Playwright (browser E2E), k6 (load) |
| Frontends | Vite 6, React 19, TanStack Router + Query, Tailwind 4 |
| Contracts | Rust stable, `near-sdk` 5, `near-workspaces` |
| Lint/format | Biome (TS), rustfmt + clippy (Rust) |

## 3. `nearpay-spec` — the contract system

**Authoring model:** contracts are authored as zod schemas in TypeScript (one source of truth, consumed directly by TS code), and emitted as OpenAPI 3.1 / JSON Schema for docs, other languages and diffing.

```
nearpay-spec/
├── src/
│   ├── common/        ids.ts, money.ts, pagination.ts, errors.ts, metadata.ts, enums.ts
│   ├── public/        <resource>.ts — request/response schemas + route table for /v1
│   ├── internal/      <service>.ts — service-to-service route tables
│   ├── events/        <domain>.ts — domain event envelopes and payloads (NATS)
│   ├── webhooks/      public event types + payloads (rendered per API version)
│   ├── x402/          NearPay x402 scheme schemas (intents-transfer, near-intents)
│   └── errors/        catalogue.ts — every error code, HTTP status, message template, docs slug
├── openapi/           GENERATED public.yaml, internal/<service>.yaml
├── asyncapi/          GENERATED events.yaml
├── schemas/           GENERATED JSON Schemas (webhooks, events, x402)
├── codegen/           emit scripts
├── packages/spec-public/    @nearpay/spec (public schemas, types, error codes)
├── packages/spec-internal/  @nearpay-internal/spec (internal routes + events)
└── test/compat/       oasdiff against the last released tag; fails on breaking change
```

**Route table format** (each route is data, so servers and clients are generated from it):

```ts
export const createCheckoutSession = route({
  method: "POST", path: "/v1/checkout/sessions", service: "payments",
  auth: ["secret_key", "restricted_key:checkout_sessions.write"],
  idempotent: true, body: CheckoutSessionCreate, response: CheckoutSession,
  errors: ["amount_below_destination_minimum", "account_not_live", "destination_not_active"],
});
```

Rules: additive changes only within a version; new enum values are allowed only for enums declared `open()`; every schema field has `.describe()`; every route lists its error codes; `pnpm emit && pnpm compat` must pass in CI. Releases: semver per package; services pin exact versions.

## 4. `nearpay-kit` — shared runtime packages

| Package | Public surface (signatures abbreviated) |
|---|---|
| `@nearpay-internal/ids` | `newId(prefix): PrefixedId`, `toUuid(id)`, `fromUuid(prefix, uuid)`, `isId(prefix, s)` — UUIDv7, Crockford base32 (26 chars) |
| `@nearpay-internal/money` | `type Amount = bigint`; `Fiat {amount: bigint, currency}`; `AssetAmount {amount: bigint, assetId}`; `parseUnits`, `formatUnits`, `mulDivFloor(a,b,c)`, `bpsOf(amount,bps)`, `minorUnits(currency)`; throws on any `number` input |
| `@nearpay-internal/service` | `createService({name, config, routes, consumers, jobs})` → Fastify app with health (`/healthz`, `/readyz`), metrics, OTel, pino, graceful shutdown (30 s drain), request-id, error mapping; `loadConfig(zodSchema)` reading env + `/run/secrets/*` |
| `@nearpay-internal/db` | `createDb(url)` (Kysely + pg pool), `withTx(fn, {isolation})`, `migrate(dir)`, **outbox** `emit(tx, event)` + `startRelay()`, **inbox** `consume(tx, eventId)`, **idempotency** `withIdempotency(ctx, key, hash, fn)`, `advisoryLeader(name, fn)` |
| `@nearpay-internal/bus` | `connectBus()`, `ensureStreams(defs)`, `subscribe({durable, filter, handler, maxDeliver})`, `publish(subject, envelope, msgId)` |
| `@nearpay-internal/auth` | `verifyInternalJwt(token, jwks)`, `requireScope(ctx, scope)`, `serviceTokenClient({clientId, secretFile})`, header constants |
| `@nearpay-internal/http` | `httpClient({baseUrl, timeoutMs, retries, breaker})` with jittered backoff, idempotent-only retries, circuit breaker (opossum semantics), trace + request-id propagation |
| `@nearpay-internal/intents` | NEAR Intents payloads: `buildTransferIntents`, `buildPayload({signerId, deadline, nonce, intents})`, `makeNonce(saltHex)`, `encodeNep413`, `encodeErc191`, `verifySigned(multiPayload)`, `parseDip4Events(logs)`, `intentsRpc({currentSalt, simulateIntents, executeIntents, mtBatchBalanceOf})` |
| `@nearpay-internal/chains` | Chain registry (CAIP-2 ↔ 1Click `blockchain` ↔ display), address validation per chain family, finality rules, explorer URL builders |
| `@nearpay-internal/testing` | Testcontainers fixtures, fake clock, factories for every aggregate, `expectEvent(subject, matcher)` |

Kit packages contain **no domain rules** (ADR-0001). Semver; a breaking change bumps major and ships a codemod note.

## 5. Identifiers

- Format `<prefix>_<26 Crockford base32 chars>` encoding a UUIDv7 (time-ordered). DB columns store `uuid`; the API renders/accepts the prefixed form. Prefix catalogue: `architecture/domain-model.md` §1 plus `req_` (request), `ses_` (session), `pk_`/`sk_`/`rk_` (keys, see identity LLD), `obs_` (observation), `job_` (job).
- IDs never leak `livemode`; cross-mode lookups return `404 resource_missing`.

## 6. Money and assets

- In code: `bigint` only. In JSON: fiat `amount` is an integer (`number`, must be ≤ 2^53 − 1, validated); asset amounts are decimal **strings** of base units.
- Asset identity = NEAR Intents asset ID (`nep141:…`, `nep245:…`). Chain identity = CAIP-2 where it exists (`eip155:8453`, `near:mainnet`, `solana:5eykt4Us…`, `bip122:000000000019d6689c085ae165831e93`), with 1Click's `blockchain` code kept as an attribute.
- Rounding: conversions to the merchant's settlement amount round **down**; fees round **up**; payer-facing totals round **up**. All helpers in `@nearpay-internal/money`.

## 7. Database conventions

- One database and one login role per service (`<svc>_app`), one migration role (`<svc>_migrator`); `app` roles have no DDL rights. Extensions: `pgcrypto`, `citext`.
- Table names plural `snake_case`; PK `id uuid`; every merchant-owned row has `account_id uuid not null` and `livemode boolean not null`; `created_at`, `updated_at timestamptz not null default now()`; aggregates have `version integer not null default 0` (optimistic concurrency: `update … where id = $1 and version = $2`).
- Every money aggregate has a `<aggregate>_transitions` table: `(id uuid pk, <aggregate>_id uuid, from_state text, to_state text, cause text, actor text, evidence jsonb, created_at)` — append-only (INV-9).
- Shared tables created by `@nearpay-internal/db` migrations in every service:

```sql
create table outbox (
  id uuid primary key,                 -- event id
  subject text not null,               -- evt.<domain>.<aggregate>.<event>
  aggregate_type text not null, aggregate_id uuid not null, aggregate_version integer not null,
  account_id uuid, livemode boolean,
  payload jsonb not null, trace jsonb,
  created_at timestamptz not null default now(), published_at timestamptz);
create index outbox_pending on outbox (created_at) where published_at is null;

create table inbox (event_id uuid primary key, subject text not null, received_at timestamptz not null default now());

create table idempotency_keys (
  account_id uuid not null, livemode boolean not null, key text not null,
  request_hash bytea not null, state text not null check (state in ('in_progress','completed')),
  locked_until timestamptz not null, response_status integer, response_body jsonb,
  created_at timestamptz not null default now(),
  primary key (account_id, livemode, key));
create index idempotency_expiry on idempotency_keys (created_at);

create table jobs (
  id uuid primary key, kind text not null, run_at timestamptz not null, payload jsonb not null,
  dedupe_key text unique, attempts integer not null default 0, max_attempts integer not null default 20,
  locked_by text, locked_until timestamptz, last_error text, done_at timestamptz,
  created_at timestamptz not null default now());
create index jobs_due on jobs (run_at) where done_at is null;

create table dead_letters (id uuid primary key, source text not null, subject text, payload jsonb not null,
  error text not null, created_at timestamptz not null default now(), resolved_at timestamptz);
```

- **Idempotency algorithm** (`withIdempotency`): (1) `insert … state='in_progress', locked_until=now()+60s on conflict do nothing`; if a row exists: same hash & completed → replay stored response (`Idempotent-Replayed: true`); same hash & in_progress & `locked_until > now()` → `409 idempotency_in_progress`; same hash & lock expired → take over and re-run; different hash → `409 idempotency_key_reused`. (2) Run the handler; every internal side effect uses a derived deterministic key (`<idempotency-key>:<step>`). (3) Store status + body, `state='completed'`. Keys purged after 24 h by a job.
- **Jobs** (`jobs` table): workers poll `select … where done_at is null and run_at <= now() and (locked_until is null or locked_until < now()) order by run_at limit $n for update skip locked`; lease 60 s; exponential backoff `min(2^attempts × 5 s, 1 h)` ± 20%; after `max_attempts` → `dead_letters` + alert.
- **Leadership** for singleton loops: `pg_try_advisory_lock(hashtext('<svc>:<loop>'))` on a dedicated connection; lost connection = lost leadership.
- Retention: `outbox` published rows 7 days; `inbox` 14 days; `idempotency_keys` 24 h; transitions forever (AML window per policy).

## 8. Events (NATS JetStream)

- **Envelope** (`@nearpay-internal/spec` `EventEnvelope`):

```json
{"id":"<uuid>","subject":"evt.payments.payment_intent.succeeded","occurred_at":"2026-11-05T10:00:00.000Z",
 "account_id":"<uuid>","livemode":true,
 "aggregate":{"type":"payment_intent","id":"pi_…","version":7},
 "data":{"object":{…full snapshot…},"previous":{…changed fields…}},
 "trace":{"traceparent":"00-…"}}
```

- **Streams:** one per producing domain — `EVT_IDENTITY` (`evt.identity.>`), `EVT_PAYMENTS`, `EVT_BILLING`, `EVT_RAILS`, `EVT_AGENTPAY`, `EVT_COMPLIANCE`, `EVT_SENTINEL`. File storage, limits retention 7 days, duplicate window 120 s (publish with `Nats-Msg-Id = event id`).
- **Consumers:** durable pull consumers named `<consumer-svc>--<stream>`, explicit filter subjects, `ack_wait 30s`, `max_deliver 20`, backoff `[1s,5s,30s,2m,10m…]`; after max deliver the handler writes `dead_letters` and acks.
- **Handler contract:** in one DB transaction: `inbox.consume(event.id)` (skip if seen) → load aggregate → ignore if `aggregate.version` ≤ last applied version stored in the read model → apply → commit → ack.
- **Ordering:** not guaranteed across subjects; consumers rely on `aggregate.version`.

## 9. Internal authentication

| Token | Minted by | Claims | TTL | Use |
|---|---|---|---|---|
| **Request token** | identity `/internal/authz` (gateway ForwardAuth) | `iss=nearpay-identity`, `aud=nearpay`, `sub` (principal id), `acct`, `lm`, `ptype` (`secret_key`·`restricted_key`·`publishable_key`·`client_secret`·`session`·`ops`), `scp[]`, `rid` | 60 s | Forwarded as `X-NearPay-Token` to the routed service; services may forward it on calls made *for that request* |
| **Service token** | identity `/internal/service-tokens` (client credentials: `client_id` + secret file) | `ptype=service`, `sub=<svc>`, `scp[]` (static per service, see each LLD) | 5 min, cached until 30 s before expiry | Background jobs and service-to-service calls not tied to a request |

EdDSA (Ed25519) signatures, `kid` header, JWKS at identity `/internal/jwks` (cached 5 min, refetch on unknown `kid`). Every handler calls `requireScope`. Headers: `X-NearPay-Token`, `X-Request-Id`, `traceparent`, `Idempotency-Key`, `NearPay-Version`.

## 10. HTTP conventions (internal and outbound)

| Default | Value |
|---|---|
| Inbound request budget | 10 s public, 5 s internal (Fastify `connectionTimeout`/handler deadline) |
| Outbound timeout | min(remaining budget − 100 ms, per-integration timeout) |
| Retries | only idempotent methods or requests with an idempotency key; 3 attempts, 100 ms × 2^n ± 20% |
| Circuit breaker | open after 50% failures over ≥ 20 calls in 30 s; half-open after 15 s |
| Pagination | `limit` 1–100 (default 10), `starting_after`/`ending_before` (IDs), stable by `(created_at desc, id desc)` |

Every service listens on `:8080` (HTTP) and exposes `/healthz` (process up), `/readyz` (DB + bus reachable, migrations at expected version), `/metrics` on `:9464`.

## 11. Errors

Public envelope per `architecture/api-conventions.md` §6. Internal calls use the same envelope with `type`/`code`. Unknown internal errors map to `500 api_error` with a generated `req_` id; stack traces only in logs.

## 12. Observability names

- Logs: pino JSON with `svc`, `env`, `rid`, `trace_id`, `span_id`, `acct`, `lm`; PII fields redacted by path list (`email`, `name`, `ip`, `*.secret`, `authorization`).
- Metrics: `nearpay_<svc>_<thing>_<unit>`; mandatory set: `http_requests_total{route,code}`, `http_request_duration_seconds{route}`, `outbox_lag_seconds`, `consumer_lag_messages{durable}`, `jobs_due{kind}`, `dead_letters_open`, `external_calls_total{target,outcome}`, `external_call_duration_seconds{target}`.
- Traces: one span per inbound request, outbound call, DB transaction, event publish/consume; outbox rows carry `traceparent` so consumer spans link to the producer.

## 13. Configuration and secrets

- Env vars `NP_<NAME>`; secrets only as files `/run/secrets/<name>` (Docker secrets, ADR-0015), referenced by `NP_<NAME>_FILE`.
- Common keys: `NP_ENV` (`local`·`server`), `NP_SERVICE`, `NP_DB_URL_FILE`, `NP_NATS_URL`, `NP_VALKEY_URL`, `NP_IDENTITY_URL`, `NP_JWKS_URL`, `NP_SERVICE_CLIENT_SECRET_FILE`, `NP_OTEL_ENDPOINT`, `NP_LOG_LEVEL`, `NP_EGRESS_PROXY`.
- Config is validated at boot with zod; invalid config = exit 1.

## 14. NEAR Intents payloads (implemented in `@nearpay-internal/intents`)

- **Message JSON** (Verifier): `{"signer_id","verifying_contract":"intents.near","deadline":"<ISO>","nonce":"<base64 32B>","intents":[…]}`; NEP-413 variant keeps `nonce` and `recipient` in the NEP-413 `payload` and puts `{signer_id, deadline, intents}` in `message` (string).
- **Nonce:** 32 bytes = 4-byte current salt (view `current_salt` on `intents.near`, hex) + 28 random bytes; base64.
- **Envelope (`MultiPayload`):** `{"standard":"nep413","payload":{"recipient":"intents.near","nonce","message"},"public_key":"ed25519:…","signature":"ed25519:…"}` or `{"standard":"erc191","payload":"<json string>","signature":"secp256k1:…"}` (v normalised to 0/1). Keys/signatures are `<curve>:<base58>`; secp256k1 pubkeys 64 B uncompressed, signatures 65 B r‖s‖v.
- **Transfer intent:** `{"intent":"transfer","receiver_id":"<acct>","tokens":{"nep141:<token>":"<amount>"},"memo":"<binding>"}`.
- **Calls:** `simulate_intents({"signed":[…]})` → `{intents_executed[{intent_hash,account_id,nonce}], logs[], min_deadline, state{fee,current_salt}}`; `execute_intents({"signed":[…]})` (gas: from simulation + margin, SP-07); `mt_batch_balance_of({account_id, token_ids})` → `string[]`.
- **Events:** log lines prefixed `EVENT_JSON:`; `{"standard":"dip4","version":"0.3.0","event":"intents_executed"|"token_diff"|"transfer"|…,"data":[…]}` plus NEP-245 `mt_transfer`/`mt_mint`/`mt_burn`. The `transfer` event data shape is not documented → the parser treats `intents_executed` + `mt_transfer` (`old_owner_id`, `new_owner_id`, `token_ids`, `amounts`, `memo`) as authoritative and is fixture-tested against mainnet receipts (SP-14).

## 15. Containers

- Base `node:22-bookworm-slim` (multi-stage; final stage `gcr.io/distroless/nodejs22-debian12` where native deps allow), non-root UID 10001, read-only root FS, `tmpfs /tmp`, `HEALTHCHECK` → `/healthz`.
- One image per service; process type selected by command: `node dist/main.js api|worker|relay|migrate`.

## 16. Testing conventions

| Level | Scope | Tooling | Gate |
|---|---|---|---|
| Unit | pure domain (state machines, money, mapping) | Vitest | ≥ 90% branch coverage on `src/domain` |
| Property | state machines and money: random event sequences never violate invariants | fast-check | required for every money aggregate |
| Integration | service + real Postgres/NATS/Valkey | Testcontainers | every endpoint and consumer |
| Contract | provider/consumer against `nearpay-spec` routes and events | Vitest + spec fixtures | in CI of both sides |
| E2E | whole stack via Compose | `nearpay-testbed` (Playwright, scripts) | per stage exit |
| Mainnet canary | live rails, capped budget | `nearpay-testbed/canary` | stage demos |
