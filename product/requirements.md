# NearPay — Requirements

> Status: **Accepted v1.0** · IDs are stable; reference them in PRs, ADRs and tests (`FR-PAY-03`).
> MUST / SHOULD / MAY per RFC 2119.

## 1. Invariants (non-negotiable; every design and PR is checked against these)

| ID | Invariant |
|---|---|
| **INV-1 Non-custody** | No key held by NearPay can unilaterally move payer or merchant funds. NearPay-held keys may only (a) pay gas, (b) submit payloads already signed by the fund owner, (c) trigger contract methods whose rules the payer pre-agreed. Fee revenue lands in the NearPay treasury in-flight. |
| **INV-2 Evidence before success** | A `PaymentIntent`/`Refund` becomes `succeeded` only when settlement is backed by verified evidence (destination tx/receipt or Verifier ledger event) for at least the expected amount. A provider status alone is never sufficient on Tier-A chains. |
| **INV-3 Truth wins** | Internal timers (expiry, cancellation) never contradict on-chain truth. If money moved, the system records it — even late. |
| **INV-4 Exactly-once effects, at-least-once delivery** | Every state transition is idempotent and versioned; every outbound message (event, webhook, email) is at-least-once with a stable ID consumers can dedupe on. |
| **INV-5 Integer money** | Amounts are integers in base units with an explicit asset/currency. No floats anywhere in money paths. |
| **INV-6 Immutable snapshots** | Prices, fees, settlement destination and route used for a payment are snapshotted at creation and never mutated. |
| **INV-7 Origin refunds** | Refunds go to the payer's recorded origin/refund address unless an owner overrides with step-up auth and fresh screening. |
| **INV-8 Livemode isolation** | Test-mode objects can never touch mainnet rails; live objects can never touch the sandbox. |
| **INV-9 Append-only audit** | Every money-relevant fact (provider observation, verification, transition, admin action) is stored append-only with actor and timestamp. |

## 2. Functional requirements

### 2.1 Accounts and access (`identity`)
- **FR-ACC-01** MUST support merchant accounts with multiple users and roles: `owner`, `admin`, `developer`, `finance`, `support`, `viewer`.
- **FR-ACC-02** MUST authenticate dashboard users with passkeys (WebAuthn); email magic-link MAY be used for recovery only; TOTP as backup factor.
- **FR-ACC-03** MUST issue API keys per mode: secret (`sk_live_`/`sk_test_`), publishable (`pk_…`), restricted (`rk_…` with scopes). Secrets shown once, stored hashed.
- **FR-ACC-04** Sensitive actions (settlement destination changes, key creation, team/role changes) MUST require step-up auth; destination changes MUST require two approvers when the account has ≥ 2 admins.
- **FR-ACC-05** Live mode MUST be gated on KYB status `approved` and at least one `active` settlement destination.

### 2.2 Settlement destinations
- **FR-SET-01** A destination is `{asset_id, type: chain_address | intents_account, chain, address}` for **any** supported asset.
- **FR-SET-02** Ownership MUST be attested by a signature from the destination key (NEP-413, EIP-191, Solana `signMessage`, BIP-322) where the chain supports message signing; otherwise flagged `unverified` with lower limits and a warning.
- **FR-SET-03** New/changed destinations MUST pass screening and a cool-off (default 24 h) before becoming usable; all owners are notified.
- **FR-SET-04** Each destination exposes a computed `min_payment_amount` (destination network costs make tiny payments uneconomic).
- **FR-SET-05** Destinations that need a memo/tag to credit the recipient are not supported in v1.

### 2.3 Payments (`payments`, `rails`)
- **FR-PAY-01** Merchants create `CheckoutSession`s and `PaymentIntent`s priced in a fiat currency (ISO 4217) or an asset; amounts in minor units/base units.
- **FR-PAY-02** Payers can pay with any token on any chain supported by the active route provider(s), filtered by policy (liquidity, min amount, risk).
- **FR-PAY-03** Two payer paths: **wallet-connect** (NEAR, EVM, Solana, others as adapters exist) and **send-from-anywhere** (deposit address + memo + QR, payer-supplied refund address).
- **FR-PAY-04** The merchant MUST receive at least the snapshotted settlement amount; payer-side fees, slippage buffer and excess are handled by the route (excess refunded to payer).
- **FR-PAY-05** Under-, over-, duplicate-, late- and wrong-amount payments MUST each have a defined outcome (see `architecture/domain-model.md` §5).
- **FR-PAY-06** Payment status MUST stream to the checkout page in real time (SSE with polling fallback).
- **FR-PAY-07** `PaymentLink`s create a fresh session per visit; hosted invoice pages reuse the checkout app.
- **FR-PAY-08** Every API `POST` MUST honour `Idempotency-Key` (24 h, body-fingerprinted).
- **FR-PAY-09** An attempt's deadline MUST NOT exceed the session's `expires_at` unless the merchant enabled late payments; routes that cannot fit the remaining window MUST be hidden.
- **FR-PAY-10** Deposit instructions MUST NOT be rendered without a valid sentinel co-signature (ADR-0010).

### 2.4 Refunds
- **FR-REF-01** Full and partial refunds; Σ(refunds pending + succeeded) ≤ settled amount, enforced transactionally.
- **FR-REF-02** Refunds are merchant-authorised: NearPay prepares the route and the merchant signs (dashboard wallet or merchant backend key). NearPay never signs for merchant funds (INV-1).
- **FR-REF-03** Default refund basis is the **settled amount** (merchant returns what it received); payer receives it in the original asset/chain at the then-current rate.
- **FR-REF-04** Refunds carry no NearPay fee; `fee_mode` is `payer_absorbs` (default) or `merchant_absorbs`.

### 2.5 Billing (`billing`)
- **FR-BIL-01** Products, Prices (one-time / recurring), manual Tax Rates, Invoices (draft → open → paid | void | uncollectible), hosted invoice URL + PDF.
- **FR-BIL-02** Subscriptions with `collection_method = send_invoice` (any chain) and `auto_debit` (NEAR vault mandate).
- **FR-BIL-03** Dunning schedule configurable; renewal generation idempotent per `(subscription, period)`.
- **FR-BIL-04** Auto-debit MUST be cancellable by the payer at any time with unused funds withdrawable by the payer alone.

### 2.6 Agent payments (`agentpay`, `agentkit`)
- **FR-AGT-01** x402 v2 facilitator endpoints `/supported`, `/verify`, `/settle`.
- **FR-AGT-02** Scheme `exact` with transfer method `intents-transfer` (payer-signed NEAR Intents `transfer` intents; NearPay relays) — settle-before-serve by default.
- **FR-AGT-03** Scheme `exact` with transfer method `near-intents` (cross-chain upfront deposit, compatible with x402 PR #3370) for agents without Intents balances.
- **FR-AGT-04** Payment payload MUST be bound to the specific resource requirement (hash in the intent memo) to prevent reuse.
- **FR-AGT-05** Agents can pay normal `CheckoutSession`s/invoices via API/MCP using the same signed-transfer path.

### 2.7 Webhooks and events (`webhooks`)
- **FR-WH-01** Events have stable IDs, `created`, `livemode`, API version, full object snapshot and an object `sequence` for ordering.
- **FR-WH-02** Deliveries are signed (HMAC-SHA256 `v1` + Ed25519 `v1a`), retried with exponential backoff for 72 h, then the endpoint is disabled and owners alerted.
- **FR-WH-03** Events retained 30 days; list/retrieve/resend via API, dashboard and CLI.
- **FR-WH-04** Webhook egress MUST go through an SSRF-safe egress proxy from published static IPs.

### 2.8 Compliance (`compliance`)
- **FR-CMP-01** KYB through a vendor before live mode; periodic refresh by risk tier.
- **FR-CMP-02** Screen every settlement destination (on add + daily), every payer origin/refund address (pre-quote) and actual sender addresses (post-deposit).
- **FR-CMP-03** Geo-policy (IP country, configurable lists) on checkout; jurisdiction is undecided, so policy is data, not code.
- **FR-CMP-04** Case management for hits; actions: block address, suspend merchant, report.

### 2.9 Merchant operations
- **FR-OPS-01** Dashboard: payments, refunds, invoices, subscriptions, customers, events/webhook logs, settlement, team, keys, exports (CSV).
- **FR-OPS-02** Daily reconciliation report per merchant: expected vs. observed settlement and fees.

## 3. Non-functional requirements

| ID | Requirement |
|---|---|
| **NFR-AV-01** | NearPay-attributable availability ≥ 99.95%/month for create-session, checkout and webhooks; ≥ 99.9% dashboard. Upstream rail outages are reported separately. |
| **NFR-LAT-01** | API p99 < 300 ms excluding upstream quote time; live quote p95 < 4 s; checkout first paint < 1.5 s on 4G. |
| **NFR-LAT-02** | State change → webhook enqueue p99 < 5 s; → first delivery attempt p99 < 60 s. |
| **NFR-DUR-01** | RPO 0 in-region (synchronous replica); ≤ 60 s cross-region; RTO ≤ 30 min region loss; ≤ 8 h provider loss. |
| **NFR-SCL-01** | v1 hardware sustains 100 payment creations/s and 1,000 API req/s; cross-chain success detection independent of provider limits on NEAR/EVM destinations (watchers + amount tags), ≈ 10 provider status calls per payment; agent settlements 500/s target, provisional on SP-07 (shard gas). Stateless tiers scale out linearly. |
| **NFR-SEC-01** | OWASP ASVS L2 for all web surfaces; secrets only in OpenBao/SOPS; signed container images; least-privilege everywhere. |
| **NFR-PRV-01** | GDPR-ready: EU data residency by default, PII field-encrypted, erasure via pseudonymisation that preserves AML records. |
| **NFR-OBS-01** | Every request traced end-to-end (W3C trace context) across services, queue hops and provider calls. |
| **NFR-DX-01** | Test mode with deterministic scenarios; every scenario documented; SDKs typed from the single spec. |
| **NFR-MNT-01** | Each repo builds, tests and runs locally in isolation with one command; AI agents can work in any repo with only that repo + `nearpay-brain` + published specs. |
