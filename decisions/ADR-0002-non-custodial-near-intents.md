# ADR-0002: Non-custodial settlement through NEAR Intents

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-1, INV-2, INV-7

## Context
NearPay must accept any token on any chain and settle any asset (owner decision), without custody (owner decision). NEAR Intents provides solver-based any-to-any swaps across ~35 chains with the Verifier `intents.near` as settlement ledger; the 1Click API exposes this as quote → deposit address → status. 1Click "does not take custody" and funds move per quote rules (swap to recipient or refund to `refundTo`).

## Decision
- Cross-chain/any-asset payments use 1Click `EXACT_OUTPUT` quotes with `recipient` = merchant's snapshotted destination, `refundTo` = payer's origin address, `appFees` = NearPay fee.
- Payers/agents that already hold NEAR Intents balances pay by signing Verifier `transfer` intents that NearPay relays (gas only).
- NearPay holds no user funds and no key that can move them (HLD §6).

## Consequences
- ➕ No money-transmitter custody; outages never strand funds; chain coverage inherited from NEAR Intents.
- ➖ Refunds require the merchant's signature (no "refund from NearPay balance").
- ➖ Dependency on 1Click availability, fees and ToS (R-01); no webhooks; no testnet (handled in ADR-0003/0007/0012).
- ➖ NearPay must satisfy 1Click ToS duties (KYB, screening, end-user terms, unmasked origin address).

## Alternatives considered
- **Own deposit addresses via Chain Signatures + own solver/bridge integration:** maximal control, but NearPay would control deposit addresses (custodial in effect) and re-implement bridging/solver liquidity.
- **Custodial balances with batch payouts:** simpler refunds, rejected for licensing and trust.
- **Chain-specific direct transfers only:** no any-to-any; fails the product goal.

## Revisit when
1Click materially changes ToS/fees, or a direct-intents route (ADR-0003) covers ≥ 80% of volume.
