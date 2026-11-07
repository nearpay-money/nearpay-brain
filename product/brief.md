# NearPay — Product Brief

> Status: **Accepted v1.0** · Owner: Architecture · Last reviewed: 2026-11-05
> Domain: `nearpay.money` · Audience: founders, engineers, AI coding agents

## 1. One line

**NearPay is Stripe for chain-abstracted crypto:** a merchant prices in USD (or any asset), the payer pays with *any token on any chain*, and the merchant receives *the asset they chose on the chain they chose* — without NearPay ever holding the money.

## 2. Problem (first principles)

A payment has exactly three jobs: **move value** from payer to merchant, **prove it moved**, and **tell the merchant's systems** so they can fulfil. Crypto solved the first job per chain, but fragmented it across 30+ chains and thousands of tokens:

| Who | Pain today |
|---|---|
| Merchant | Must pick chains, run wallets per chain, watch each chain, reconcile by hand, and still loses payers who hold the "wrong" token. |
| Payer | Must already hold the exact token on the exact chain, or bridge/swap first (minutes, fees, risk). |
| Developer | No Stripe-grade primitives: idempotent API, signed webhooks, test mode, refunds, invoices, subscriptions. |
| AI agent | Needs to pay per request, programmatically, in seconds, for cents — card rails can't. |

**NEAR Intents** (solver network + Verifier contract `intents.near` + bridges, exposed through the **1Click API**) solves *move value across chains and assets* non-custodially. What is missing is the **merchant-grade layer** on top: objects, state machines, proofs, webhooks, refunds, billing, compliance and developer experience. That layer is NearPay.

## 3. Users and actors

| Actor | Goal | Primary surface |
|---|---|---|
| **Merchant developer** | Integrate in < 1 hour; correct, idempotent, testable | API, `@nearpay/js`, `@nearpay/node`, docs, CLI |
| **Merchant operator** (owner, finance, support) | See payments, refund, invoice, manage settlement, team | Dashboard |
| **Payer** (human) | Pay with what they hold, from wallet or exchange, in one step | Hosted checkout / embedded widget / hosted invoice |
| **Payer agent** (AI) | Pay per call/resource automatically within budget | x402 headers, MCP server, agent SDK |
| **NearPay ops** | Onboard (KYB), monitor, investigate, support | Internal console |

## 4. Product scope — v1 (production)

**Core**
- Merchant accounts, teams, RBAC, passkey auth, KYB-gated live mode, test mode.
- Settlement destinations: **any asset**, either on a destination chain address or in the merchant's NEAR Intents account; ownership-attested.
- Payments: `PaymentIntent`, `CheckoutSession` (hosted page + embeddable widget), `PaymentLink`, `Customer`.
- Payer pays any supported token on any supported chain (wallet-connect or send-from-anywhere with deposit address/QR).
- Webhooks (signed, retried, replayable), Events API, idempotent REST API, date-versioned.
- Dashboard, docs, Node SDK, JS widget/React components.

**Extended (in v1 scope, per decision on 2026-11-05)**
- **Refunds** (full/partial, merchant-authorised, to the payer's origin address).
- **Invoices** (hosted invoice page, reminders).
- **Subscriptions** — `send_invoice` renewals on every chain; opt-in `auto_debit` through a non-custodial NEAR vault contract.
- **AI-agent payments** — x402 facilitator (instant NEAR-Intents transfer scheme + cross-chain deposit scheme), x402 middleware, MCP server.

**Compliance**: automated address screening on every payment + KYB before live mode (v1: local OFAC list and self-declared business details approved by ops — ADR-0020).

**Revenue**: in-flight basis-point fee taken through NEAR Intents `appFees`, routed directly to the NearPay treasury account (no custody of merchant funds).

## 5. Explicit non-goals (v1)

| Non-goal | Why | Revisit |
|---|---|---|
| Holding merchant/payer balances (custody) | Licensing, security, trust model | Never by default |
| Fiat on/off-ramp | Licensing; partner later | v2 |
| Card payments, chargebacks | Crypto is final; different rail | — |
| E-commerce plugins (Shopify, WooCommerce), in-store POS | Out of v1 scope by decision | v2 |
| Marketplace splits / Connect-style platforms | Needs multi-party fee model | v2 (`appFees` already supports multiple recipients) |
| Agent wallets / spend policies on the payer side | Wallet's job, not merchant processor's | v2 |
| SDKs other than TS/JS | Focus | v2 (Python, Go) |

## 6. Differentiation (vs. PingPay and custodial gateways)

1. **Provably non-custodial** — no NearPay key can move user funds; hot keys are NEAR function-call keys that can only *submit already-signed* intents.
2. **Any-asset settlement**, chosen per merchant, with ownership attestation and change cool-off (anti-takeover).
3. **Independent settlement verification** — a payment is `succeeded` only with on-chain evidence, not just a provider status.
4. **Stripe-grade DX** — idempotency on every POST, test mode with a deterministic sandbox (NEAR Intents has no testnet), signed webhooks with replay, CLI.
5. **Instant agent payments** — signed NEAR-Intents transfers settle in ~1.2 s (NEAR finality) with near-zero fees, not a cross-chain deposit per call.

## 7. Success metrics (v1)

| Metric | Target |
|---|---|
| Time-to-first-live-payment for a developer | < 60 min |
| Checkout conversion (session opened → paid) | ≥ 55% |
| Median time payer-confirm → merchant webhook (same-chain stable) | ≤ 30 s; cross-chain p95 ≤ 15 min |
| Payments marked succeeded without verified evidence | **0** |
| Unexplained reconciliation differences (daily) | **0** |
| Webhook delivery within 60 s of state change | ≥ 99.9% |
| NearPay-attributable API availability | ≥ 99.95% monthly |

## 8. Naming note

"NearPay" is also used by at least two unrelated companies (a Saudi SoftPOS fintech and a crypto-card product in the NEAR ecosystem). Trademark clearance is an open business question — see `roadmap/open-questions.md` (Q-BIZ-01). The architecture is name-agnostic: all identifiers derive from the `nearpay.money` domain.
