# ADR-0009: Agent payments via x402 v2 — `intents-transfer` and `near-intents` methods

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-1, INV-2, INV-4

## Context
AI agents need to pay per request programmatically. x402 v2 is the most adopted HTTP-native protocol (`PAYMENT-REQUIRED` / `PAYMENT-SIGNATURE` / `PAYMENT-RESPONSE`; facilitator `/verify`, `/settle`, `/supported`). NEAR is not an upstream x402 network; PR #3370 proposes `assetTransferMethod: near-intents` with upfront cross-chain deposits (PingPay implements it). A cross-chain deposit per call is too slow and costly for micropayments, while a signed Verifier `transfer` settles in ~1.2 s for gas only.

## Decision
- `nearpay-agentpay` is an x402 v2 facilitator supporting scheme `exact` with two transfer methods:
  1. **`intents-transfer`** (default, NearPay-native): network `near:mainnet`; payer signs a Verifier payload containing `transfer(payTo, amount)` + `transfer(treasury, fee)` with `memo = binding`, where `binding = sha256(canonical(requirements) ‖ resource URL ‖ merchant account)`; `deadline ≤ maxTimeoutSeconds`.
  2. **`near-intents`** (compatibility with PR #3370): two-phase quote → deposit → settle via rails/1Click.
- `/verify`: signature (NEP-413/ERC-191), nonce unused, `simulate_intents` passes, amounts/payTo/binding match, payer screened, merchant quota OK.
- `/settle`: settle-before-serve by default; the batcher submits payloads every ≤ 150 ms with batch size bounded by simulated gas (≤ 80% of max prepaid gas per transaction), using a pool of function-call keys (parallel nonces); on batch failure, re-simulate and submit individually; idempotent on payload hash; returns `settlement_pending` if finality exceeds the request budget.
- **Economics:** minimum absolute fee per payment ≥ 2 × its share of relayer gas; per-merchant and per-payer quotas; relayer allowance caps (T-16).
- **Throughput:** `intents.near` is a single contract on one shard shared with all NEAR Intents traffic; sustained throughput is bounded by that shard's gas. SP-07 measures gas per transfer intent; above the bound, the v2 path is a channel-style scheme (payer pre-funds a NearPay channel contract, signs cumulative `upto` vouchers off-chain, merchant settles periodically).
- Optional merchant setting **verify-then-serve** for amounts ≤ a merchant-chosen cap (risk accepted by merchant).
- Merchant-side `@nearpay/x402` middleware and agent-side `@nearpay/agent-signer`; MCP server `@nearpay/mcp` for discovering and paying sessions/invoices. MPP adapter is an extension point, not v1-critical.
- NearPay will propose `intents-transfer` upstream to the x402 foundation.

## Consequences
- ➕ ~1–2 s, near-zero-fee agent payments; non-custodial (we relay signed payloads only).
- ➖ Agents must hold a NEAR Intents balance (fundable from any chain via 1Click once).
- ➖ Non-standard method until upstreamed; we also support the PR #3370 method for interoperability.

## Alternatives considered
- **Only `near-intents` deposits:** interoperable but minutes and higher fees per call.
- **EVM `exact` (EIP-3009) only:** ignores NEAR's speed/cost advantage and chain abstraction.

## Revisit when
x402 upstream adopts a NEAR network/scheme, or `batch-settlement`/`upto` schemes become standard for micropayments.
