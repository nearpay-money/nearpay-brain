# Public API and Webhook Conventions

> Version 1.3 (step 3.1, spec 0.3.0: `invoice.sent` in the event catalogue and the events SSE stream §9) · Version 1.2 (step 2.1, spec 0.2.0: `refund.pending` in the event catalogue §9) · Version 1.1 (list filters instead of search endpoints, ADR-0018) · Normative for `nearpay-spec`. Every public endpoint and event MUST follow this document; deviations need an ADR.

## 1. Surfaces

| Host | Purpose | Auth |
|---|---|---|
| `api.nearpay.money/v1/*` | Server-to-server REST | `Authorization: Bearer sk_…` or `rk_…` |
| `api.nearpay.money/v1/checkout/sessions/{id}/public*` | Checkout SPA calls | `client_secret` + `pk_…` (public-safe scopes only) |
| `dashboard.nearpay.money/api/*` | Dashboard (same-origin proxy to gateway) | Session cookie (`__Host-`, `Secure`, `HttpOnly`, `SameSite=Strict`) + CSRF token |
| `api.nearpay.money/x402/*` | x402 facilitator | Merchant `sk_`/`rk_` (`agentpay:facilitate` scope) |
| `js.nearpay.money/v1/nearpay.js` | Widget loader | `pk_…` |

## 2. Resource style

- REST, JSON, `snake_case`, plural nouns, form of Stripe's API (developers already know it).
- Every object has `id`, `object`, `created` (unix seconds), `livemode`, `metadata` (≤ 50 keys, key ≤ 40 chars, value ≤ 500 chars).
- Actions are sub-resources: `POST /v1/payment_intents/{id}/cancel`, `POST /v1/invoices/{id}/finalize`.
- `expand[]` for one-level expansion of referenced objects.
- Lists: `GET /v1/{resource}?limit=1..100&starting_after=&ending_before=` → `{object: "list", data, has_more, url}`, newest first. Filtering uses typed query parameters on the list itself — `field=value`, ranges `field[gte|gt|lte|lt]=`, value lists `field[]=`, `metadata[key]=value` — declared per route in `nearpay-spec`; there are no search endpoints or query language (ADR-0018).

## 3. Money fields

| Pricing in fiat | `{"amount": 4999, "currency": "usd"}` (minor units) |
|---|---|
| Pricing in an asset | `{"amount": "25000000", "currency": "asset", "asset_id": "nep141:…"}` |
| Settlement values | `{"amount": "25000000", "asset_id": "…", "decimals": 6, "display": "25.00 USDC (Base)"}` |

## 4. Idempotency

- `Idempotency-Key` header (≤ 255 chars) on every `POST`. Stored 24 h per `(account, livemode, key)` with a SHA-256 of method + path + canonical body.
- Same key + same body → replay original status and body (header `Idempotent-Replayed: true`). Same key + different body → `409 idempotency_key_reused`. Concurrent in-flight → `409 idempotency_in_progress` with `Retry-After`.
- SDKs generate a UUIDv4 key automatically and reuse it across their own retries.

## 5. Versioning

- Date-based versions: header `NearPay-Version: 2026-11-01`; default = version pinned at account creation; upgrades are explicit in the dashboard.
- Backward-compatible changes (new fields, new event types, new enum values documented as open) ship without a new version. Clients MUST tolerate unknown fields and enum values.
- Webhook endpoints have their own pinned version; payloads are rendered per endpoint version.

## 6. Errors

```json
{"error": {"type": "invalid_request_error", "code": "amount_below_destination_minimum",
  "message": "Amount must be at least 2.00 USD for destination sd_… (USDC on Ethereum).",
  "param": "amount", "doc_url": "https://docs.nearpay.money/errors/amount_below_destination_minimum",
  "request_id": "req_…"}}
```

| HTTP | `type` | Retry? |
|---|---|---|
| 400 | `invalid_request_error` | No |
| 401/403 | `authentication_error` / `permission_error` | No |
| 402 | `payment_error` (e.g. compliance block) | No |
| 404 | `invalid_request_error` (`resource_missing`) | No |
| 409 | `idempotency_error` / `state_conflict` | Per code |
| 429 | `rate_limit_error` | Yes, honour `Retry-After` |
| 5xx | `api_error` | Yes, same Idempotency-Key |
| 503 | `rail_unavailable` (upstream route provider down) | Yes, later |

Error codes are a closed catalogue in `nearpay-spec/errors.yaml`; each has a docs page.

## 7. Rate limits

Per key: 100 req/s live, 25 req/s test (bursts 2×); per checkout session: 5 live attempts; response headers `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` (IETF draft). Limits are data (per account tier).

## 8. Test mode

`sk_test_`/`pk_test_` keys route every object to the **sandbox rail** (ADR-0007). Deterministic scenarios are chosen by the **refund address** or **amount suffix** so they work from any client:

| Trigger (test mode) | Scenario |
|---|---|
| amount minor units end in `00`, or any suffix not listed below | success in ~5 s |
| ends in `01` | success after 2 min (slow chain) |
| ends in `02` | underpaid → `incomplete_deposit` → refunded at deadline |
| ends in `03` | overpaid → success + excess refund |
| ends in `04` | provider `FAILED` after deposit → `held` → resolved refunded |
| ends in `05` | `SUCCESS` but evidence mismatch → `requires_review` |
| ends in `06` | duplicate settlement on second attempt → `overpaid` |
| ends in `07` | late settlement after session expiry |
| refund address `sanctioned.test` | compliance block |

The sandbox also exposes `POST /v1/test_helpers/executions/{id}/advance` for step-by-step control.

## 9. Webhooks

**Envelope**

```json
{"id": "evt_…", "object": "event", "type": "payment_intent.succeeded", "api_version": "2026-11-01",
 "created": 1793900000, "livemode": true,
 "data": {"object": {"id": "pi_…", "object": "payment_intent", "sequence": 7, "...": "..."},
          "previous_attributes": {"status": "processing"}},
 "request": {"id": "req_…", "idempotency_key": "…"}}
```

- `data.object.sequence` increases with every change of that object; consumers discard events whose sequence ≤ the last applied.
- **Signature header:** `NearPay-Signature: t=1793900000,v1=<hex HMAC-SHA256(secret, t + "." + raw_body)>,v1a=<base64 Ed25519(sig over same string)>`. Ed25519 public keys at `https://api.nearpay.money/v1/webhook_signing_keys` (JWKS). Tolerance 300 s; constant-time compare; **verify the raw body bytes** (SDK helpers do this).
- **Delivery:** at-least-once, unordered, timeout 10 s, success = any 2xx. Retries with exponential backoff + jitter (≈ 1 m, 5 m, 30 m, 2 h, 5 h, 10 h, then every 12 h) up to 72 h. After 72 h of continuous failure the endpoint is disabled and owners are emailed.
- **Egress:** from published static IPs (`docs.nearpay.money/ips`); private/reserved ranges are refused; DNS resolution pinned per attempt.
- **Retention:** events 30 days; `GET /v1/events`, `POST /v1/events/{id}/resend`; `GET /v1/events/stream` (SSE, test-mode API keys or a dashboard session; webhooks LLD §5.1); CLI `nearpay listen` reads the stream and forwards to localhost.

**Event catalogue (v1)**

| Domain | Types |
|---|---|
| Checkout | `checkout.session.completed`, `checkout.session.expired` |
| Payments | `payment_intent.created`, `.processing`, `.succeeded`, `.payment_failed`, `.requires_review`, `.overpaid`, `.canceled` |
| Refunds | `refund.created`, `refund.requires_action`, `refund.pending`, `refund.succeeded`, `refund.failed`, `refund.canceled` |
| Billing | `invoice.created`, `.finalized`, `.sent`, `.paid`, `.payment_failed`, `.overdue`, `.voided`, `.marked_uncollectible`; `customer.subscription.created`, `.updated`, `.deleted`, `.renewal_upcoming`; `mandate.activated`, `.collected`, `.insufficient_funds`, `.canceled`, `.exhausted` |
| Agent | `agent_payment.settled`, `agent_payment.failed` |
| Account | `account.updated`, `settlement_destination.pending`, `.activated`, `.disabled`, `review.opened`, `review.closed` |
