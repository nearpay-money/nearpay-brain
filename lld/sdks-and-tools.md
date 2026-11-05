# LLD — SDKs and tools (`nearpay-node`, `nearpay-agentkit`, `nearpay-cli`, `nearpay-site`)

> Version 1.1 (audited, `audits/sdks-and-tools.md`) · Foundations `00-foundations.md` · API conventions

## 1. `@nearpay/node` (Stage 1 subset, full in Stage 2)

- Generated resource clients from `@nearpay/spec` route tables: `nearpay.checkout.sessions.create(params, {idempotencyKey?})`, `paymentIntents`, `refunds`, `paymentLinks`, `customers`, `invoices`, `subscriptions`, `products`, `prices`, `webhookEndpoints`, `events`, `agentPayments`.
- Transport: `fetch` with timeout 30 s, automatic `Idempotency-Key` (UUIDv4) on POST reused across up to 2 retries (network errors, 409 `idempotency_in_progress`, 429, 5xx) with exponential backoff; `NearPay-Version` pinned per client.
- `nearpay.webhooks.constructEvent(rawBody: string|Buffer, signatureHeader, secret, {toleranceSec=300})` — verifies `v1` (any of multiple) with constant-time compare; optional `verifyEd25519(rawBody, header, jwks)`.
- `nearpay.refunds.signers`: helpers that take a refund `next_action` and a merchant-provided signer (viem `Account`, NEAR `KeyPair`) and broadcast — never constructs a different recipient/amount.
- Errors: typed `NearPayError` subclasses by `type`; `requestId` exposed.

## 2. `nearpay-agentkit`

**`@nearpay/x402`** (Stage 1):
- Server middleware (Express, Hono, Next route handler): `paywall({amount, asset?, description?, maxTimeoutSeconds?}, {secretKey})`. Flow: no `PAYMENT-SIGNATURE` → call `POST /x402/requirements` (**one call per 402 response, no caching** — bindings are single-use) → `402` with `PAYMENT-REQUIRED: base64(PaymentRequired)` and body `{}`. With header → decode → `POST /x402/verify` → invalid → `402` with `PAYMENT-RESPONSE` failure; valid → `POST /x402/settle` → success → call handler, set `PAYMENT-RESPONSE: base64(SettlementResponse)`; `settlement_pending` → one retry after 1 s, then `402` with pending response.
- Client: `withX402(fetch, signer, {maxAmountPerRequest, allowedPayTo?})` → on 402 picks the NearPay `intents-transfer` requirement, enforces client-side spend limits, signs via `@nearpay/agent-signer`, retries with `PAYMENT-SIGNATURE`.

**`@nearpay/agent-signer`** (Stage 1): `nearSigner({accountId, keyPair})` (NEP-413), `evmSigner(viemAccount)` (ERC-191), `buildTransferPayment(requirements, signer, {nonceSaltProvider})` → MultiPayload; fetches `current_salt` via RPC (cached 60 s). For NearPay **checkout** requirements it first obtains and verifies a sentinel attestation (pinned keys shipped in the package, `kid` allow-list) and refuses to sign on mismatch; x402 requirements come from the merchant's own server and are signed as received (client spend limits apply).

**`@nearpay/mcp`** (Stage 3): MCP server (stdio + streamable HTTP) with tools `nearpay_create_checkout_session`, `nearpay_get_payment_intent`, `nearpay_list_payments` (merchant key), and payer tools `nearpay_pay_checkout_session(session_url)` and `nearpay_fetch_paid(url)` (uses `withX402` with a configured signer and budget).

## 3. `nearpay-cli` (Stage 3)

Commands: `login` (device code via dashboard session → restricted test key stored in OS keychain), `listen --forward-to http://localhost:3000/webhooks` (consumes `GET /v1/events/stream`, re-signs with a local secret printed at start), `trigger <scenario>` (creates a test session and advances the sandbox), `events list|resend`, `logs tail`. Distributed via npm (`@nearpay/cli`) and single-file binaries (`bun build --compile`).

## 4. `nearpay-site` — docs (Stage 2; Stage 1 = READMEs)

One Astro project (ADR-0019) builds both public sites: `dist/docs` → `docs.<domain>`, `dist/web` → apex domain.

Astro Starlight site: guides (quickstart, checkout, webhooks, test mode, refunds, agents/x402, settlement destinations), API reference rendered from `nearpay-spec/openapi/public.yaml`, error pages generated from the error catalogue, webhook IP list, changelog. Built statically; deployed with the stack.

## 5. `nearpay-site` — marketing (Stage 5)

Astro static marketing site at `nearpay.money`: product, pricing (from fee schedule doc), legal pages (terms incl. 1Click flow-down provisions, privacy), status link.

## 6. Tests

SDK: contract tests against spec fixtures; webhook and agent-signer vectors from `nearpay-spec/test-vectors` (same files used by `nearpay-webhooks` and `nearpay-agentpay`); x402 middleware E2E against sandbox in testbed.
