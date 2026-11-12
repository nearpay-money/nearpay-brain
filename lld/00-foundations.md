# LLD 00 — Foundations (`nearpay-spec`, `nearpay-kit`, conventions)

> Version 1.8 (kit 0.6.0, backend resilience audit 2026-11-12 KIT-1…KIT-7: supervised consumers, outbox dead-letter and replay §8; breaker probe settlement, service tokens through identity outages §10; liveness from loop heartbeats, DB and leadership timeouts §15; changes in §22) · Spec 0.4.0 update (releases 0.3.0, 0.3.1, 0.4.0 in §3 "Releases so far") · Step 3.1 update (§13: `NP_ENV` ∈ {local, server, test}, required, no default — kit 0.2.1; kit SSE limits and x402 `near-intents` helpers, agentpay §3.1, webhooks §5.1) · Spec 0.2.1 update (releases §"Releases so far") · Version 1.7 (kit 0.2.0: ops-principal enforcement §9, `@nearpay-internal/ui` §4, ADR-0021; changes in §21) · Version 1.6 (step 2.1 integration: Stage 2 service and ops scopes §9, spec 0.2.0 §3) · Version 1.5 (step 2.1 rails-fx: kit money FX helpers, http call headers, intents `sendSignedTx`; spec exact rates; changes in §20) · Version 1.4 (step 1.4: `nearpay-spec` 0.1.5 and tolerant parsing §3; changes in §19) · Version 1.3 (steps 1.2 and 1.3 implemented against `nearpay-spec` 0.1.4 and `nearpay-kit` 0.1.2; changes in §18) · Version 1.2 step 1.1 (§17) · Version 1.1 audited (`audits/00-foundations.md`) · Applies to every service and client. Component LLDs only state deviations from this file.

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
├── test-vectors/       shared fixtures: JCS+Ed25519 attestations, webhook signatures, NEAR Intents payloads (consumed by producer and consumer tests)
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
├── packages/spec-public/    @nearpay-money/spec (public schemas, types, error codes)
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

**Tolerant parsing (0.1.5):** `openEnum` schemas stay strict `z.enum`s (servers validate what they produce; the typed unions are unchanged). Clients that parse responses, events or webhook payloads use `safeParseTolerant(schema, input)` / `parseTolerant(schema, input)`: an unknown string value of an **open** enum is accepted, kept verbatim and listed in `unknown` with its path; closed enums, literals, types, formats and missing fields still fail with the strict error. The dashboard, checkout and agentkit use it; `@nearpay-money/node` uses it from 0.2.0 (step 2.3), returning a contract-breaking body as received by default and throwing only in `strict` mode (sdks-and-tools §1).

**Releases so far (all additive):** 0.1.0 Stage 1 contracts (step 1.1) · 0.1.1 sandbox quote carries `scenario_amount` · 0.1.2 `pol_` ids and execution review subjects · 0.1.3 optional `livemode` on rails asset and options reads · 0.1.4 `ks_` id prefix (kill switch) · 0.1.5 change requests from steps 1.2/1.3 (step 1.4; `nearpay-spec/CHANGELOG.md`): `PublicCheckoutSession.payment_intent`, x402 `SettleResponse.replayed` and `extra.deadline`, open `enabled_events` request items, `sequence` on `CheckoutSession` and `AgentPayment`, routes `GET /v1/auth/session`, `GET /v1/webhook_endpoints/{id}/deliveries` (`whd_` prefix) and `POST /v1/settlement_destinations/{id}/challenge`, tolerant parsing. · 0.2.0 Stage 2 contract freeze (step 2.1) · 0.2.1 Stage 2 change requests (`nearpay-spec/CHANGELOG.md` table): service-readable kill switch (`sentinel.read`), `abandonExecution`, observed fees/deposited amounts on executions, declared error codes, CLI actors on `OpsActionEvent`, `evt.identity.user.recovered`, `evt.webhooks.endpoint.secret_rotated`, `getInternalOpsOperator` (`identity.operators.read`), display data for checkout, dashboard and console. · 0.3.0 Stage 3 contract freeze (step 3.1): billing (products, prices, tax rates, invoices and hosted invoices, subscriptions, billing emails; `txr_`, `il_`, `si_` prefixes) and agent commerce (cross-chain x402 `near-intents`, events SSE stream, CLI device login, `nearpay trigger` helper) · 0.3.1 Stage 3.2 change requests: `getEvidenceSummary?owner=`, `SimTx.transfers[].from`, declared error codes on invoice sessions, subscriptions and FX/pricing routes, invoice-mail totals and destination fingerprint, test-mode billing clock `advanceSubscriptionClock` · 0.4.0 Stage 4 contract freeze (step 4.1): auto-debit (subscription-vault mirror and `test-vectors/vault.json` shared with the contract and the sandbox stand-in, `md_` mandates and `/v1/mandates*`, hosted mandate funding, `evt.rails.vault.event`, `evt.billing.mandate.*` and `mandate.*` webhooks, `CollectionMethod` `auto_debit`, sandbox vault stand-in — OQ-T-03) and Solana + Bitcoin as Tier A chains (`src/common/chains.ts`, `Evidence.solana`/`Evidence.bitcoin`, `solana`/`bip322` destination attestation, chain test vectors).

## 4. `nearpay-kit` — shared runtime packages

| Package | Public surface (signatures abbreviated) |
|---|---|
| `@nearpay-internal/ids` | `newId(prefix): PrefixedId`, `toUuid(id)`, `fromUuid(prefix, uuid)`, `isId(prefix, s)` — UUIDv7, Crockford base32 (26 chars) |
| `@nearpay-internal/money` | `type Amount = bigint`; `Fiat {amount: bigint, currency}`; `AssetAmount {amount: bigint, assetId}`; `parseUnits`, `formatUnits`, `mulDivFloor(a,b,c)`, `bpsOf(amount,bps)`, `minorUnits(currency)`; exact rationals `parseDecimal`, `mul`/`div`, `floor`/`ceil`; **Stage 2:** `RoundingMode` (`floor`·`ceil`·`half_even`·`half_up`) + `roundRational`, `toDisplayDecimal`, `ratioToWire`/`ratioFromWire` (spec `Ratio`), `convertUnits(amount, fromExp, rate, toExp, mode)`, `toUsdMicros`, `parseJsonDecimals` (JSON numbers kept as source text); throws on any `number` input |
| `@nearpay-internal/service` | `createService({name, config, routes: [implement(route, handler)], idempotency, readiness, background})` → Fastify app with spec-route auth + validation, health (`/healthz`, `/readyz`), metrics, OTel, pino, graceful shutdown (30 s drain), request-id, error mapping, SSE; consumers and jobs run as `background` tasks; `loadConfig(shape)` reading `NP_*` env + `NP_*_FILE` secrets (`/run/secrets/*`); `runMain({api, worker, migrate, …})` |
| `@nearpay-internal/db` | `createDb({url\|urlFile})` (Kysely + pg pool), `withTx(db, fn, {isolation})`, `migrate(db, dir)`, **outbox** `emit(tx, event)` + `startOutboxRelay({db, pool, publish})`, **inbox** `consume(tx, eventId)`, **idempotency** `withIdempotency(db, ctx, fn)` / `idempotencyStore(db, cipher)`, **jobs** `enqueueJob`/`startJobWorker`, `advisoryLeader({pool, name, run})`, envelope encryption, retention; **0.1.2:** `inboxHandler` (inbox dedupe + handler in one transaction), `deadLetterSink`, background tasks `outboxRelayTask`, `jobWorkerTask`, `leaderTask`, `recurringJobsTask` (leader-elected, one job per slot through the dedupe key) |
| `@nearpay-internal/bus` | `connectBus()`, `ensureStreams(defs)`, `subscribe({durable, filter, handler, maxDeliver})`, `publish(subject, envelope, msgId)`; **0.1.2:** `startDomainConsumers` creates one durable per producing stream with the handled subjects as filters and refuses subjects the event catalogue does not route to the service; `unhandledSubjects`, `durablesOf` |
| `@nearpay-internal/auth` | `verifyInternalJwt(token, jwks)`, `authorizeRoute(claims, route)`, `requireScope(ctx, scope)`, `serviceTokenClient({clientId, secretFile})`, header constants; **0.2.0:** ops principals enforced in `authorizeRoute` (§9) |
| `@nearpay-internal/http` | `httpClient({baseUrl, timeoutMs, retries, breaker})` with jittered backoff, idempotent-only retries, circuit breaker (opossum semantics), trace + request-id propagation; `call(route, {headers})` adds route-declared headers (e.g. a forwarded client secret) but never replaces the token (Stage 2, CR-15-16) |
| `@nearpay-internal/intents` | NEAR Intents payloads: `buildTransferIntents`, `buildPayload({signerId, deadline, nonce, intents})`, `makeNonce(saltHex)`, `encodeNep413`, `encodeErc191`, `verifySigned(multiPayload)`, `parseDip4Events(logs)`, `intentsRpc({currentSalt, simulateIntents, executeIntents, buildExecuteTx, mtBatchBalanceOf})`. `executeIntents` loses the tx hash when the send times out, so agentpay signs with `buildExecuteTx` and stores the hash first; since Stage 2 it sends with `sendSignedTx({bytes, txHash})`, whose `ExecuteResult` carries `blockHash` (CR-15-16) |
| `@nearpay-internal/chains` | Chain registry (CAIP-2 ↔ 1Click `blockchain` ↔ display), address validation per chain family, finality rules, explorer URL builders |
| `@nearpay-internal/testing` | Testcontainers fixtures, fake clock, factories for every aggregate, `eventProbe(natsUrl)` + `expectEvent(probe, subject, matcher)`, test tokens; **0.1.1:** `mockPeer` serves a peer's spec routes through the real service bootstrap (requests validated, caller tokens verified and authorized from each route's auth list, mocked responses validated, calls recorded), `mockError`, `mintServiceToken`, `testServiceToken`; **0.1.2:** `provisionServiceDatabase` mirrors `postgres-init` (migrator owns, app role gets DML through default privileges); **0.2.0:** ops test principals `opsClaims`, `mintOpsToken({role, scopes, stepUp})`, `testOpsToken` (token provider for `httpClient`), `opsHeaders`; `mockPeer` serves console routes (`OPS_ROUTES`) |
| `@nearpay-internal/ui` | **0.2.0** (ADR-0021): the design system shared by checkout, dashboard, console and site — `tokens.css` (CSS custom properties) + `styles.css` (plain `np-*` classes) + `fonts.css` (self-hosted DM Sans), thin React 19 components (`ThemeRoot`, `AppShell`, `CheckoutFrame`, `Button`, `Field`, `Money`, `Stat`/`StatGroup`, `Table`, `Timeline`, `Modal`, `ToastProvider`, …), integer `Money` formatting from base units (INV-5), runtime merchant appearance with contrast checks; no runtime CSS-in-JS, no inline styles (CSP-safe); static gallery (`pnpm --filter @nearpay-internal/ui gallery:build`). Peer `react@^19` |

Kit packages contain **no domain rules** (ADR-0001). Semver; a breaking change bumps major and ships a codemod note.

## 5. Identifiers

- Format `<prefix>_<26 Crockford base32 chars>` encoding a UUIDv7 (time-ordered). DB columns store `uuid`; the API renders/accepts the prefixed form. Prefix catalogue: `architecture/domain-model.md` §1 plus `req_` (request), `ses_` (session), `pk_`/`sk_`/`rk_` (keys, see identity LLD), `obs_` (observation), `job_` (job).
- IDs never leak `livemode`; cross-mode lookups return `404 resource_missing`.

## 6. Money and assets

- In code: `bigint` only. In JSON: fiat `amount` is an integer (`number`, must be ≤ 2^53 − 1, validated); asset amounts are decimal **strings** of base units.
- Rates (prices, FX) on the wire are exact rationals `Ratio {num, den}` (integer strings) with a display-only `decimal` (spec 0.2.0 `PriceRate`); feed values are parsed from their text (`parseDecimal`, `parseJsonDecimals`), never via floats.
- Asset identity = NEAR Intents asset ID (`nep141:…`, `nep245:…`). Chain identity = CAIP-2 where it exists (`eip155:8453`, `near:mainnet`, `solana:5eykt4Us…`, `bip122:000000000019d6689c085ae165831e93`), with 1Click's `blockchain` code kept as an attribute.
- Storage: fiat minor units `bigint`; **asset base units `numeric(78,0)`** (24-decimal tokens exceed `bigint`).
- Rounding: conversions to the merchant's settlement amount round **down**; fees round **up**; payer-facing totals round **up**; display renderings of rates round **half-even**. Every helper takes the mode explicitly. All helpers in `@nearpay-internal/money`.

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
  locked_until timestamptz not null, response_status integer, response_body_enc bytea,   -- envelope-encrypted (responses may carry secrets, e.g. client_secret)
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
- **Jobs** (`jobs` table): `dedupe_key` must be prefixed by the job kind (`<kind>:<id>…`). Workers poll `select … where done_at is null and run_at <= now() and (locked_until is null or locked_until < now()) order by run_at limit $n for update skip locked`; lease 60 s; exponential backoff `min(2^attempts × 5 s, 1 h)` ± 20%; after `max_attempts` → `dead_letters` + alert.
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

- **Streams:** one per producing domain — `EVT_IDENTITY` (`evt.identity.>`), `EVT_PAYMENTS`, `EVT_BILLING`, `EVT_RAILS`, `EVT_AGENTPAY`, `EVT_COMPLIANCE`, `EVT_SENTINEL`, `EVT_WEBHOOKS`.
- **No PII in events:** payloads carry IDs and non-personal fields only; consumers needing PII (notify) call the owner's internal API. File storage, limits retention 7 days, duplicate window 120 s (publish with `Nats-Msg-Id = event id`).
- **Consumers:** durable pull consumers named `<consumer-svc>--<stream>`, explicit filter subjects, `ack_wait 30s`, `max_deliver 20`, backoff `[1s,5s,30s,2m,10m…]`; after max deliver the handler writes `dead_letters` and acks. The backoff is applied by `nak(delay)` on handler failure, **not** with the consumer's `backoff` setting: NATS replaces `ack_wait` with `backoff[0]` when `backoff` is set (verified on 2.10.29), which would redeliver any message whose handler runs longer than 1 s. Unparseable envelopes and envelopes whose `subject` differs from the NATS subject are dead-lettered at once.
- **Supervised consume loop (kit 0.6.0, KIT-1):** a durable's consume loop that ends or throws without `stop()` (durable deleted, stream recreated after a NATS data loss, terminal consume error) is reported (`onError`) and re-created — consumer `info`/`add` + `consume` — after an exponential back-off with jitter (1 s → 60 s), until it consumes again; `stop()` interrupts the back-off. Consumes use `abort_on_missing_resource` (a vanished durable ends the iterator instead of spinning on "consumer not found") and 10 s pull expiry (idle heartbeat 5 s), so a durable deleted while no pull is pending is noticed within ~10 s. Only the first setup at boot throws. Metric `consumer_running{durable}` (0 while restarting); the loop beats its liveness heartbeat (§15) while consuming or backing off, never while a handler hangs or once the connection is closed for good (a NATS outage is readiness, not liveness). Alert `ConsumerLagHigh` (`consumer_lag_messages > 0 for 10m`, nearpay-deploy).
- **Handler contract:** in one DB transaction: `inbox.consume(event.id)` (skip if seen) → load aggregate → ignore if `aggregate.version` ≤ last applied version stored in the read model → apply → commit → ack.
- **Ordering:** not guaranteed across subjects; consumers rely on `aggregate.version`.
- **Outbox poison rows (kit 0.6.0, KIT-6):** the relay publishes in order and stops at the first failure, counting it on the row (`outbox.attempts`, `last_error`, `first_failed_at`; kit migration `0000_kit_outbox_attempts`). Failures are classified: `retry` (bus down, timeout — retried forever, the row keeps its place, `OutboxBacklog` alerts), `poison` (payload over NATS `max_payload`, invalid subject — deterministic) and `unroutable` (no stream takes the subject). A `poison` row is moved to `dead_letters` (source `outbox`, payload `{outbox_id, envelope}`) after 5 attempts, an `unroutable` one after 5 attempts **and** 5 min of failing (a JetStream restart never dead-letters a backlog); the relay then continues with the next row and logs `ALERT outbox_dead_lettered` (the `DeadLetters` alert fires on `dead_letters_open`). Consumers already ignore stale `aggregate.version`s.
- **Dead-letter replay (kit 0.6.0, KIT-7):** `@nearpay-internal/db` `replayDeadLetter` / `dropDeadLetter` / `listDeadLetters`, and every service ops CLI exposes them through `runKitOps`: `ops <svc> dead-letters list [--source …] | show <id> | replay <id> | drop <id> --note …` and `ops <svc> outbox show | skip <id> --note …`. Replay by source: `consumer:*` → the stored envelope (validated) goes through the service's handler, which dedupes through the inbox, so a replay is idempotent; `outbox` → the envelope is re-queued under its event id (bus de-duplication and consumer inboxes keep it idempotent); `job:<kind>` → the job is re-armed. `dead_letters` records `resolution` (`replayed`·`dropped`), `resolved_by`, `resolution_note`. Unparseable envelopes cannot be replayed, only dropped.

## 9. Internal authentication

| Token | Minted by | Claims | TTL | Use |
|---|---|---|---|---|
| **Request token** | identity `/internal/authz` (gateway ForwardAuth) | `iss=nearpay-identity`, `aud=nearpay`, `sub` (principal id), `acct`, `lm`, `ptype` (`secret_key`·`restricted_key`·`publishable_key`·`client_secret`·`session`·`ops`·`anonymous`), `scp[]`, `rid`; sessions add `role` and `step_up_until` (unix s) | 60 s | Forwarded as `X-NearPay-Token` to the routed service. Calls to other services authenticate with the **service token** (every internal route authorizes `service:`/`ops:` scopes); forwarding the request token is opt-in (`httpClient({forwardRequestToken: true})`) for a target that authorizes the original caller |
| **Service token** | identity `/internal/service-tokens` (client credentials: `client_id` + secret file) | `ptype=service`, `sub=<svc>`, `scp[]` (static per service, see each LLD) | 5 min; refreshed in the background from 60 s before expiry and used until 5 s before expiry while identity is unavailable (§10, kit 0.6.0) | Background jobs and service-to-service calls not tied to a request |

**Service scope catalogue** (`service_clients.scopes`; a call outside this table is a defect):

| Client | Scopes |
|---|---|
| identity | `compliance.screen`, `rails.read`, `notify.send`, `compliance.read` (Stage 2: approved KYB details) |
| payments | `identity.context.read`, `rails.executions.write`, `rails.read`, `agentpay.relay`, `compliance.read`, `sandbox.use`, `compliance.cases.write` (Stage 2: `reconciliation_diff` cases), `sentinel.read` (spec 0.2.1: kill-switch state at boot) |
| rails | `compliance.screen`, `sandbox.use`, `sentinel.read` (spec 0.2.1) |
| agentpay | `payments.attempts.create`, `payments.read`, `rails.read`, `identity.context.read`, `compliance.screen`, `sandbox.use`, `sentinel.read` (spec 0.2.1) |
| sentinel | `rails.read`, `sandbox.use` (+ DB role `sentinel_ro`) |
| billing | `payments.sessions.create`, `payments.read`, `notify.send` |
| notify | `identity.recipients.read`, `payments.receipts.read`, `billing.read` |
| compliance | `identity.operators.read` (spec 0.2.1: active-operator check on case assignment) |
| webhooks, sandbox | none outbound |
| ops principals (`ptype=ops`) | `compliance.kyb.decide`, `compliance.cases.write`, `sentinel.killswitch`, `rails.read`, `payments.read`; Stage 2 (spec 0.2.0, `OPS_SCOPES`): `console.session`, `identity.merchants.read`, `identity.audit.read`, `compliance.read`, `compliance.accounts.write`, `compliance.blocklist.write`, `compliance.screen`, `sentinel.read`, `sentinel.findings.write`, `payments.reconcile`, `rails.verify`. Both console roles hold all of them; what only `ops_admin` may do is set per route by `opsRoles` (checked against the `ops_role` claim) |

EdDSA (Ed25519) signatures, `kid` header, JWKS at identity `/internal/jwks` (cached 5 min, refetch on unknown `kid`). `@nearpay-internal/service` verifies the token and authorizes every route from its spec definition before the handler runs (principal type, key/service/ops scopes, session roles and step-up, test-mode-only); routes open at the gateway (`anonymous`, `client_secret`, `gateway`, `internal`, `service_client`) accept requests without a token and expose an authorized principal only when one matches. Handlers call `requireScope` for finer checks. Headers: `X-NearPay-Token`, `X-Request-Id`, `traceparent`, `Idempotency-Key`, `NearPay-Version`.

**Ops principals (kit 0.2.0, spec 0.2.0).** `authorizeRoute` fails closed for `ptype=ops`: the token must carry a known `ops_role` (`OPS_ROLES`; no other principal type may carry one); the route must list an `ops:<scope>` the principal holds and its role is granted (`OPS_ROLE_SCOPES`); `ops_role` must be in the route's `opsRoles` (an `ops` route without `opsRoles` admits nobody; internal routes with `opsRoles`, e.g. `decideKyb`, are checked too); `stepUp` routes need a fresh `step_up_until`, as for sessions (`401 step_up_required`). `ops` routes (`/ops/…`) refuse every other principal type (`403 permission_denied`) except anonymous sign-in routes under `/ops/auth/`; non-ops routes admit ops principals only through an explicit `ops:` entry (e.g. `listEvidence` with `ops:rails.read`). `ops` routes get the 10 s request budget (§10).

## 10. HTTP conventions (internal and outbound)

| Default | Value |
|---|---|
| Inbound request budget | 10 s public and ops (console), 5 s internal (Fastify `connectionTimeout`/handler deadline) |
| Outbound timeout | min(remaining budget − 100 ms, per-integration timeout) |
| Retries | only idempotent methods or requests with an idempotency key; 3 attempts, 100 ms × 2^n ± 20% |
| Circuit breaker | open after 50% failures over ≥ 20 calls in 30 s; half-open after 15 s; exactly one probe, always settled (below) |
| Pagination | `limit` 1–100 (default 10), `starting_after`/`ending_before` (IDs), stable by `(created_at desc, id desc)` |

**Breaker probe settlement (kit 0.6.0, KIT-2).** Everything that can fail without contacting the target — the service token, body serialisation — happens before the breaker grants a permit. The permit is settled on every exit: a verdict on success, 5xx, network error and timeout (the body read included), a release without verdict when the caller aborts. Results that belong to an earlier breaker period are ignored, and a probe that never reports re-opens the breaker after max(30 s, 2 × timeout). The breaker therefore always closes or re-opens; it can never stay half-open refusing every call. Metric `http_breaker_open{target}` (1 while open or half-open); alert `HttpBreakerOpen` > 2 min (nearpay-deploy).

**Service tokens through identity outages (kit 0.6.0, KIT-3).** `serviceTokenClient` refreshes in the background from `exp − 60 s` while callers keep the cached token; a failed refresh is retried with back-off (1 s → 30 s, jitter) and the cached token is served until `exp − 5 s`. Only a caller without a usable token waits for identity and gets its error. `ALERT service_token_refresh_failing` is logged once per failure streak. Verifiers keep enforcing `exp`. An identity outage now stops service-to-service calls only when the cached token really expires: between ≈ 55 s (outage begins just before a refresh) and ≈ 4.9 min (just after one) with the 300 s TTL, instead of failing at the first refresh attempt. Longer tolerance needs a longer service-token TTL (spec `TOKEN_TTL_SECONDS`).

State transitions and deadlines use database time (`now()`); containers share the host clock, which must be NTP-synchronised (chrony on the server stage).

Every service listens on `:8080` (HTTP) and exposes `/healthz` (liveness: process up and every registered background loop beating, §15), `/readyz` (DB + bus reachable, migrations at expected version), `/metrics` on `:9464`.

## 11. Errors

Public envelope per `architecture/api-conventions.md` §6. Internal calls use the same envelope with `type`/`code`. Unknown internal errors map to `500 api_error` with a generated `req_` id; stack traces only in logs.

## 12. Observability names

- Logs: pino JSON with `svc`, `env`, `rid`, `trace_id`, `span_id`, `acct`, `lm`; PII fields redacted by path list (`email`, `name`, `ip`, `*.secret`, `authorization`).
- Metrics: `nearpay_<svc>_<thing>_<unit>`; mandatory set: `http_requests_total{route,code}`, `http_request_duration_seconds{route}`, `outbox_lag_seconds`, `consumer_lag_messages{durable}`, `jobs_due{kind}`, `dead_letters_open`, `external_calls_total{target,outcome}`, `external_call_duration_seconds{target}`.
- Traces: one span per inbound request, outbound call, DB transaction, event publish/consume; outbox rows carry `traceparent` so consumer spans link to the producer.

## 13. Configuration and secrets

- Env vars `NP_<NAME>`; secrets only as files `/run/secrets/<name>` (Docker secrets, ADR-0015), referenced by `NP_<NAME>_FILE`.
- Common keys: `NP_ENV` (`local`·`server`·`test`; **required, no default** — a process without it fails at boot, kit 0.2.1), `NP_SERVICE`, `NP_DB_URL_FILE`, `NP_NATS_URL`, `NP_VALKEY_URL`, `NP_IDENTITY_URL`, `NP_JWKS_URL`, `NP_SERVICE_CLIENT_SECRET_FILE`, `NP_OTEL_ENDPOINT`, `NP_LOG_LEVEL`, `NP_EGRESS_PROXY`.
- Config is validated at boot with zod; invalid config = exit 1.
- **Pitfall:** `secret(schema)` of `@nearpay-internal/service` marks the schema object it wraps. Wrapping a shared schema (e.g. a common URL schema) silently turns every other key using it into a secret that is accepted only from a file; always pass a fresh schema (found in sentinel's Compose config; fixed with a regression test).

## 14. NEAR Intents payloads (implemented in `@nearpay-internal/intents`)

- **Message JSON** (Verifier): `{"signer_id","verifying_contract":"intents.near","deadline":"<ISO>","nonce":"<base64 32B>","intents":[…]}`; NEP-413 variant keeps `nonce` and `recipient` in the NEP-413 `payload` and puts `{signer_id, deadline, intents}` in `message` (string).
- **Nonce:** 32 bytes = 4-byte current salt (view `current_salt` on `intents.near`, hex) + 28 random bytes; base64.
- **Envelope (`MultiPayload`):** `{"standard":"nep413","payload":{"recipient":"intents.near","nonce","message"},"public_key":"ed25519:…","signature":"ed25519:…"}` or `{"standard":"erc191","payload":"<json string>","signature":"secp256k1:…"}` (v normalised to 0/1). Keys/signatures are `<curve>:<base58>`; secp256k1 pubkeys 64 B uncompressed, signatures 65 B r‖s‖v.
- **Transfer intent:** `{"intent":"transfer","receiver_id":"<acct>","tokens":{"nep141:<token>":"<amount>"},"memo":"<binding>"}`.
- **Calls:** `simulate_intents({"signed":[…]})` → `{intents_executed[{intent_hash,account_id,nonce}], logs[], min_deadline, state{fee,current_salt}}`; `execute_intents({"signed":[…]})` (gas: from simulation + margin, SP-07); `mt_batch_balance_of({account_id, token_ids})` → `string[]`.
- **Events:** log lines prefixed `EVENT_JSON:`; `{"standard":"dip4","version":"0.4.3","event":"intents_executed"|"token_diff"|"transfer"|…,"data":[…]}` (version observed on mainnet 2026-11-05; the parser accepts any `dip4` version) plus NEP-245 `mt_transfer`/`mt_mint`/`mt_burn`. **SP-14 resolved** (mainnet blocks 223270688–223270709 via neardata, 2026-11-06): the DIP-4 `transfer` event data is `{intent_hash, account_id, receiver_id, tokens: {<asset>: <amount>}, memo?}`, and the NEP-245 `mt_transfer` (`old_owner_id`, `new_owner_id`, `token_ids`, `amounts`) emitted for a transfer intent carries **no memo**. Binding attribution therefore reads the memo of the DIP-4 `transfer` event, never `mt_transfer` (the earlier assumption that the `mt_transfer` memo was authoritative was wrong); the parser is fixture-tested against these receipts and the sandbox emits the same shapes.

## 15. Containers

- Base `node:22-bookworm-slim` (multi-stage; final stage `gcr.io/distroless/nodejs22-debian12` where native deps allow), non-root UID 10001, read-only root FS, `tmpfs /tmp`, `HEALTHCHECK` → `node dist/main.js healthcheck` (built into `runMain`: GET `127.0.0.1:$NP_PORT/healthz`, no curl needed in distroless).
- One image per service; process type selected by command: `node dist/main.js api|worker|relay|migrate|…` (`runMain` exits 2 on an unknown process, 1 on invalid configuration or startup failure, drains 30 s on SIGTERM). The image's entrypoint is `node`; Compose passes `[dist/main.js, <process>]`.
- Builds take `ARG NP_REGISTRY_URL` for `@nearpay-money/*` and `@nearpay-internal/*` (platform §1).
- **Liveness (kit 0.6.0, KIT-4).** `/healthz` is what Docker's `HEALTHCHECK` and autoheal act on, so it must fail when the process is up but its work has stopped. Each background loop registers a heartbeat with a silence budget (`BackgroundTask.maxSilenceMs`, which hands `start(heartbeat)` a `Heartbeat`, or `service.heartbeat(name, maxSilenceMs)`) and calls `beat()` once per iteration — **including iterations that failed because a dependency is down** — and `idle()` while it is not responsible (a leader follower). `/healthz` answers 503 `stalled: <loop> …` while a loop is active and silent beyond its budget; idle loops never fail it, and a dependency outage (DB, NATS, a peer) never fails it either — that is `/readyz`. Kit defaults: outbox relay 120 s (beats per batch and per published row), job worker max(3 × lease, 120 s) (`extendLease` beats; a handler hung without it blocks its batch and goes silent), domain consumers 120 s (§8), leader loops max(12 × heartbeat, 60 s) (beats on each answered leadership heartbeat, or with `livenessFrom: "run"` only when `run` calls its `beat`), recurring jobs max(3 × tick, 60 s). Hand-written loops (e.g. the rails hot poller and verifier) adopt it by declaring `maxSilenceMs` and beating per tick. Metrics `background_last_beat_seconds{task}`, `background_task_stalled{task}`; `ALERT background_task_stalled` is logged on each transition. With the 15 s × 8 healthcheck and 60 s autoheal grace, a loop stalled beyond its budget is restarted within about 3 more minutes.
- **DB connections and leadership (kit 0.6.0, KIT-5).** Pools connect (and wait for a free slot) for at most 5 s, use TCP keepalive (first probe after 10 s) and set `idle_in_transaction_session_timeout = 60 s`, so a dead DB host never hangs a loop and a hung client cannot hold locks forever. The leadership heartbeat (`select 1`, §7) must answer within 2 × its interval or the leader gives up leadership (the lock-holding connection is destroyed); a leadership term shorter than `retryMs` (a `run` that returns or throws at once) is followed by `retryMs` + up to 20 % jitter before competing again, so re-election never hot-loops.

## 16. Testing conventions

| Level | Scope | Tooling | Gate |
|---|---|---|---|
| Unit | pure domain (state machines, money, mapping) | Vitest | ≥ 90% branch coverage on `src/domain` |
| Property | state machines and money: random event sequences never violate invariants | fast-check | required for every money aggregate |
| Integration | service + real Postgres/NATS/Valkey | Testcontainers | every endpoint and consumer |
| Contract | provider/consumer against `nearpay-spec` routes and events | Vitest + spec fixtures | in CI of both sides |
| E2E | whole stack via Compose | `nearpay-testbed` (Playwright, scripts) | per stage exit |
| Mainnet canary | live rails, capped budget | `nearpay-testbed/canary` | stage demos |

## 17. Changes in version 1.2 (from the step 1.1 implementation)

- **Events (§8):** consumer backoff is applied with `nak(delay)`; the JetStream `backoff` setting is never used (it overrides `ack_wait`).
- **Auth (§9):** claims gained `role` and `step_up_until` (sessions) and `ptype=anonymous`; service-to-service calls use service tokens by default; route authorization happens in the service bootstrap.
- **Kit details:** `@nearpay-internal/db` `idempotencyStore(db, cipher)` feeds the service bootstrap's automatic idempotency for `idempotent` routes called by account principals (replays carry `Idempotent-Replayed: true`; in-flight keys answer `409` + `Retry-After: 1`); `@nearpay-internal/service` `runMain` with a built-in `healthcheck` process; `@nearpay-internal/testing` `expectEvent(probe, subject, matcher)` (the probe subscribes before the action).
- **Tracing (§12):** a tracer provider always runs (trace ids in logs and propagation); spans are exported over OTLP/HTTP only when `NP_OTEL_ENDPOINT` is set.
- **Configuration (§13):** keys are camelCase in code and `NP_<SNAKE_CASE>` in the environment; values marked secret are accepted only from `NP_<NAME>_FILE`.
- **Intents (§14):** DIP-4 version on mainnet is `0.4.3`.

## 18. Changes in version 1.3 (from the steps 1.2 and 1.3 implementation)

- **Intents (§14):** SP-14 resolved — binding memos are on the DIP-4 `transfer` event; `mt_transfer` of a transfer intent has none.
- **Kit (§4):** 0.1.1 spec-driven peer mocks (`mockPeer`); 0.1.2 consumer glue (`inboxHandler`, `startDomainConsumers`), background tasks and provisioned test databases. Requested: an intents call that sends a pre-signed tx and returns its outcome; extra headers on `httpClient` calls.
- **Spec (§3):** releases 0.1.1–0.1.4 (additive); open enums are not yet tolerant in client parsing.
- **Configuration (§13):** never wrap a shared schema in `secret()`.

## 19. Changes in version 1.4 (from the step 1.4 implementation)

- **Spec (§3):** release 0.1.5 (additive; `pnpm compat` reports no breaking change since 0.1.4) with the step 1.2/1.3 change requests CR-15-01, 02, 03, 05, 08, 09, 11, 12 and 13 (`roadmap/open-questions.md`); new id prefix `whd_` (webhook delivery, owner webhooks; domain-model §1); tolerant parsing of open enums for clients (`safeParseTolerant` / `parseTolerant`). Not in 0.1.5: CR-15-04, 06, 07, 10 and 15.

## 20. Changes in version 1.5 (step 2.1, rails-fx)

- **Money (§6):** rates travel as exact `Ratio {num, den}`; explicit rounding modes incl. banker's rounding for display; kit `convertUnits`/`toUsdMicros`/`parseJsonDecimals` for rails FX and price feeds (rails §8.2).
- **Kit (§4):** `@nearpay-internal/http` `call` with extra headers and `@nearpay-internal/intents` `sendSignedTx` (CR-15-16, both resolved); kit stays at 0.1.2 until the lead releases.
- **Spec (§3):** step 2.1 additions are additive over 0.1.5 and ship as **0.2.0** (Stage 2 contract freeze); new error codes record `since: "0.2.0"` (`ErrorDefinition.since` is now `` `${number}.${number}.${number}` ``).

## 21. Changes in version 1.7 (kit 0.2.0)

- **Kit (§4):** every package moves to spec 0.2.0 and ships as **0.2.0** together with the rails-fx additions (§20); new package `@nearpay-internal/ui` (ADR-0021).
- **Auth (§9):** ops-principal enforcement (`ops_role` ∈ `opsRoles`, role-granted ops scope, step-up, ops/non-ops separation) lives in `@nearpay-internal/auth` `authorizeRoute`, which `@nearpay-internal/service` runs for every route; ops tokens without `ops_role` are refused. `@nearpay-internal/testing` mints ops test tokens.
- **HTTP (§10):** ops routes use the 10 s budget of gateway-routed routes.

## 22. Changes in version 1.8 (kit 0.6.0, backend resilience audit 2026-11-12)

- **Events (§8):** supervised consume loop with restart back-off and `consumer_running{durable}` (KIT-1); outbox per-row failure counts and poison/unroutable rows moved to `dead_letters` (KIT-6); dead-letter replay/drop and outbox show/skip for every ops CLI (KIT-7).
- **Database (§7, via kit migration `0000_kit_outbox_attempts`):** `outbox` gains `attempts integer not null default 0`, `last_error text`, `first_failed_at timestamptz`; `dead_letters` gains `resolution text check (resolution in ('replayed','dropped'))`, `resolved_by text`, `resolution_note text`. Kit migrations may now share the reserved `0000_` number (applied in kit order). Services must run `migrate` before the new code (readiness reports the migration pending until then); a rollback to kit ≤ 0.5 after migrating needs that `schema_migrations` row deleted (old code reports it as drift).
- **Auth (§9, §10):** service tokens are used past their refresh point while identity is unavailable (KIT-3).
- **HTTP (§10):** circuit-breaker probes are always settled; `http_breaker_open{target}` (KIT-2).
- **Containers (§15):** `/healthz` reflects background-loop heartbeats (KIT-4); DB pool connect timeout, keepalive and idle-in-transaction limit; leadership heartbeat timeout and no hot re-election (KIT-5).
