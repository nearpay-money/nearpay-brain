# ADR-0006: Revenue via in-flight `appFees`; fee schedule as data

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-1, INV-6

## Context
Non-custodial means NearPay cannot deduct fees from balances. 1Click `appFees` route a bps fee from the input token to a recipient in-flight. Per docs (Nov 2026), with a partner JWT the configured fee is split 50/50 with 1Click, 1Click's half has a 20 bps floor (1 bp on stable/same-asset routes), total ≤ 500 bps; unauthenticated calls pay an extra 25 bps.

## Decision
- Every live quote sets `appFees = [{recipient: treasury.nearpay.near, fee: total_bps}]` from the merchant's **fee schedule** (table keyed by tier × route class: `stable_same`, `stable_cross`, `volatile`), snapshotted on the PaymentIntent.
- `fee_bearer` per merchant: `payer` (output = settlement amount) or `merchant` (output reduced by the fee; `domain-model.md` §3).
- Intents-native payments include a second signed `transfer` to the treasury for the fee amount.
- Vault `collect` sends the fee share to the treasury on-chain.
- Fee revenue is reconciled daily against Explorer `appFees` and treasury ledger events.
- The fee-split model is configuration; it is never hard-coded.

## Consequences
- ➕ Revenue without custody; atomic with settlement.
- ➖ Net revenue ≈ half of the configured fee on 1Click routes; pricing must account for it.
- ➖ Fees arrive in many input tokens → treasury sweeps via partner `ANY_INPUT` aggregation to USDC.

## Alternatives considered
- **Monthly invoicing of merchants:** collection risk, and still needs a payment rail; kept as the v2 plan for enterprise tiers.

## Revisit when
A commercial agreement with the 1Click operator changes the split, or fee floors change.
