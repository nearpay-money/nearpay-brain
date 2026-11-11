# ADR-0005: Any-asset settlement destinations with attestation and cool-off

- **Status:** Accepted (cool-off amended by [ADR-0027](ADR-0027-payout-wallets-activate-immediately.md): owner decision 2026-11-11 (final) — payout wallets activate immediately after attestation/approval; no cool-off by default) · **Date:** 2026-11-05 · **Invariants:** INV-1, INV-6

## Context
Merchants may settle in **any asset** (owner decision) on a destination chain or into their NEAR Intents account. In a non-custodial system the most valuable attack is changing where future payments go (account takeover, insider, compromised API key). Small payments to expensive chains are uneconomic. Bridged assets held in Intents accounts (`*.omft.near`) carry bridge risk.

## Decision
- `SettlementDestination = {asset_id, type: chain_address | intents_account, chain, address}`, owned by `identity`.
- Activation requires: step-up passkey; screening pass; **ownership attestation** (signature by the destination key over `nearpay:settlement-destination:v1|acct|asset_id|chain|address|nonce|expiry`) where the chain supports message signing (otherwise `unverified`: per-payment cap and warnings); **two approvers** if the account has ≥ 2 admins; **24 h cool-off** with notification to all owners (configurable down to 1 h only for KYB tier 2+ with two approvers).
- API keys can never create or modify destinations (session-only endpoints).
- Each destination has a computed `min_payment_amount` (destination network costs × safety factor), refreshed hourly; payments below it are rejected with a clear error suggesting an Intents-account destination.
- Intents-account destinations holding bridged assets show a bridge-risk disclosure and an optional sweep reminder.
- Destinations that require a memo/tag to credit the recipient (typical exchange deposit addresses on XRP, Stellar, TON, Cosmos-style chains) are **not supported** in v1; merchants use a self-custodied address or an Intents account.
- PaymentIntents snapshot the destination at creation (INV-6).

## Consequences
- ➕ Account takeover cannot redirect funds silently or instantly.
- ➖ Onboarding friction (signature + wait); acceptable for a money-routing change.

## Alternatives considered
- **Free-text payout addresses (typical gateways):** fast, but the #1 theft vector.
- **Allow API-key updates:** convenient for platforms; deferred to a scoped, two-key flow in v2.

## Revisit when
Platform/Connect use cases (v2) need programmatic destination management.
