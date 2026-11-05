# LLD — `nearpay-contracts` / subscription vault

> Version 1.1 (audited, `audits/contracts-vault.md`) · ADR-0008 · Flow §6 · Stage 4 · Rust `near-sdk` 5

## 1. Scope

`subscription-vault` deployed at `v1.vault.nearpay.near` with **no access keys** after deployment. Holds NEP-141 balances per mandate; payer-funded, payer-cancellable; permissionless `collect`.

## 2. State

```rust
pub struct Vault { mandates: LookupMap<u64, Mandate>, next_id: u64, allowed_tokens: UnorderedSet<AccountId>,
                   fee_receiver: AccountId, max_fee_bps: u16, tvl_cap: LookupMap<AccountId, U128>, tvl: LookupMap<AccountId, U128> }
pub struct Mandate { payer: AccountId, merchant_receiver: AccountId, receiver_kind: ReceiverKind /* IntentsAccount | NearAccount */,
  token: AccountId, amount_per_period: U128, period_secs: u64, start_at_ns: u64, max_periods: Option<u32>,
  fee_bps: u16, collected_periods: u32, balance: U128, status: Status /* Active | Canceled | Exhausted */,
  in_flight: bool, ext_ref: String /* ≤ 64 bytes, e.g. sub_ id */ }
```
Init (once, at deploy): `allowed_tokens` (USDC, USDT on NEAR), `fee_receiver = treasury.nearpay.near`, `max_fee_bps = 500`, per-token `tvl_cap`. These are constructor parameters only — no setters exist.

## 3. Methods

| Method | Caller | Rules |
|---|---|---|
| `ft_on_transfer(sender_id, amount, msg)` | token contract (predecessor ∈ allowed_tokens) | `msg = {"create_mandate":{…}}` → new mandate (payer = sender_id), balance = amount; or `{"top_up":{"mandate_id"}}` (sender must be payer). Rejects (returns full amount) on invalid msg, cap exceeded, `fee_bps > max_fee_bps`, `amount_per_period == 0`, `period_secs < 86400`. Storage: payer pre-pays via `storage_deposit` (NEP-145) |
| `collect(mandate_id)` | anyone | `Active`, not `in_flight`, `now ≥ start_at + (collected_periods) × period`, `balance ≥ amount_per_period`, max_periods not reached → set `in_flight`, `merchant_share = amount × (10000 − fee_bps) / 10000`, `fee = amount − merchant_share` → promises: merchant share via `ft_transfer_call(intents.near, share, msg = merchant_receiver)` (IntentsAccount) or `ft_transfer(merchant_receiver)` (NearAccount); fee via `ft_transfer(fee_receiver)`; callback `on_collected` |
| `on_collected(mandate_id, share, fee)` (private) | self | inspect results: merchant leg unused amount (from `ft_resolve_transfer` return) refunded to the vault → credit back; if merchant leg fully succeeded → `collected_periods += 1`, `balance −= amount`, emit `collected`; if failed → `balance` unchanged except fee leg outcome (fee leg is only sent after the merchant leg succeeds: chained `then`), emit `collect_failed`; clear `in_flight`; exhausted when max reached |
| `cancel(mandate_id)` | payer or merchant_receiver | status `Canceled`; remaining balance `ft_transfer` to payer (callback restores balance on failure for retry via `withdraw`) |
| `withdraw(mandate_id)` | payer | only when `Canceled`/`Exhausted` with balance > 0 |
| views | anyone | `get_mandate`, `get_mandates_by_payer(from, limit)`, `next_collect_at(mandate_id)` |

Collect ordering uses `then` chaining: merchant leg → (on success) fee leg → callback; fee is never sent if the merchant leg fails.

**Error strings (stable, parsed by the keeper):** `ERR_NOT_FOUND`, `ERR_NOT_ACTIVE`, `ERR_NOT_DUE`, `ERR_IN_FLIGHT`, `ERR_INSUFFICIENT_BALANCE`, `ERR_MAX_PERIODS`, `ERR_UNAUTHORIZED`, `ERR_TOKEN_NOT_ALLOWED`, `ERR_BAD_MSG`, `ERR_CAP_EXCEEDED`, `ERR_FEE_TOO_HIGH`, `ERR_NO_STORAGE`.

**Storage:** NEP-145 `storage_deposit` / `storage_balance_of` / `storage_withdraw`; a mandate needs ≥ 0.01 NEAR of the payer's storage balance; `ft_on_transfer` without enough storage returns the full amount (refund) and logs `ERR_NO_STORAGE`; storage is released to the payer on `withdraw` of a closed mandate.

## 4. Events (NEP-297, `standard: "nearpay_vault"`, `version: "1.0.0"`)

`mandate_created {mandate_id, payer, merchant_receiver, token, amount_per_period, period_secs, start_at, max_periods, fee_bps, ext_ref}` · `topped_up {mandate_id, amount, balance}` · `collected {mandate_id, period, amount, merchant_share, fee}` · `collect_failed {mandate_id, period, reason}` · `canceled {mandate_id, by, refunded}` · `withdrawn {mandate_id, amount}` · `exhausted {mandate_id}`.

## 5. Invariants (property-tested and fuzzed)

V1 Σ balances + Σ in-flight ≤ token balance of the vault. V2 a payer can always reach `Canceled` and recover `balance`. V3 per mandate, Σ collected ≤ `collected_periods × amount_per_period` and periods ≤ elapsed periods. V4 no method transfers to any account other than payer, merchant_receiver (via intents or directly) and fee_receiver. V5 no method changes `merchant_receiver`, amount, period or fee after creation.

## 6. Deployment

Build reproducibly (`cargo near build reproducible-wasm`), publish wasm hash, deploy + `new(...)`, then delete all access keys from `v1.vault.nearpay.near` (human-signed). Upgrades = new account `v2.vault…` with payer opt-in migration (cancel + recreate).

## 7. Tests

Unit (`near-sdk` unit env), integration (`near-workspaces` sandbox with mock FT + mock `intents.near` receiver), proptest sequences for V1–V5, cargo-fuzz on `msg` parsing, gas profile per method (target `collect` ≤ 60 Tgas total). Two independent audits before mainnet (Stage 4 exit).
