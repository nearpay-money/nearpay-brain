# ADR-0008: Subscriptions — `send_invoice` everywhere, `auto_debit` via immutable NEAR vault

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-1, INV-3

## Context
Crypto payments are push-based; recurring pull requires prior authorisation. NEAR function-call keys cannot attach deposits, so a merchant/NearPay key cannot pull tokens. Pre-signed Verifier intents cannot express "not before" and could be executed early. Contract-held balances are acceptable only if no party but the payer can withdraw them outside mandate rules.

## Decision
- **`send_invoice` (all chains, default):** billing creates a renewal invoice per period (job key `subscription × period_start`), emails a hosted invoice, runs dunning (T+1/3/7, configurable), then `past_due` → `canceled`/`unpaid` per policy.
- **`auto_debit` (NEAR, opt-in):** `nearpay-contracts/subscription-vault` deployed at `v1.vault.nearpay.near`:
  - `ft_on_transfer(msg=create_mandate{merchant_receiver, amount_per_period, period_secs, start_at, max_periods, fee_bps, fee_receiver, ref})` funds a mandate (payer pays storage).
  - `collect(mandate_id)` — permissionless; at most one period's amount per elapsed period; merchant share via `ft_transfer_call` to `intents.near` (msg = merchant Intents account) or `ft_transfer` to a NEAR account; fee to treasury; the resolve callback credits back any unused amount returned by `ft_resolve_transfer` and marks the period failed (not advanced) if the merchant share was not fully delivered.
  - `cancel` (payer or merchant) returns the remainder to the payer; `top_up`, `withdraw_unused` (payer only).
  - No owner/admin methods, no access keys after deploy, upgrades = new versioned account with payer opt-in migration; per-mandate cap and global TVL cap in v1; NEP-297 events for every state change.
  - Funding from non-NEAR chains via 1Click `customRecipientMsg` is **gated on spike SP-06**; until then funding requires a NEAR wallet (including EVM-keyed wallets that support NEAR).
- Insufficient vault balance → that period falls back to `send_invoice`.

## Consequences
- ➕ True auto-pay without custody by NearPay; payer can always exit.
- ➖ Auto-debit is NEAR-only in v1; merchant receives the vault's token in its Intents account (conversion is a separate merchant action).
- ➖ Smart-contract risk (T-10) → audits, caps, bug bounty, immutability.

## Alternatives considered
- **Pre-signed intents:** executable early; rejected.
- **NearPay-held allowance keys:** custodial in effect; rejected (INV-1).
- **EVM Permit2 mandates:** viable later behind the same `Mandate` abstraction (v2).

## Revisit when
NEAR Intents adds time-locked or allowance-style intents, or EVM auto-debit demand appears.
