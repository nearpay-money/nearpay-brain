# LLD — `nearpay-contracts` / subscription vault

> Version 1.6 (re-check 2: meta-transaction deposits are refused §3.1 — correction of v1.5; funding UX rule §3.1; deployable hash only from the final CI job §6) · Version 1.5 (re-audit 2026-11-08, N-1…N-6: payer must be the transaction signer §3.1, `period_secs` ≤ 12 × 366 days §3.1, `MIN_EXIT_GAS` for cancel/withdraw §3.4, gas wording §3.3/§3.10, deploy only push-run hashes §6) · Version 1.4 (independent AI audits 2026-11-08 folded in — `nearpay-contracts/audits/2026-11-08-ai-audits.md`: relayed senders refused §3.1, input bounds §3.1, `MIN_COLLECT_GAS` and new static gas §3.3, constructor checks §3.5, callback failure §3.10, accepted risks §3.11, event validity §4.1, deploy runbook §6) · Version 1.3 (per-mandate cap `mandate_cap` per token — ADR-0008 / T-10, §3.8; cap rejection log detail §3.1; V10) · Version 1.2 (step 4.1 implementation decisions: §3.1–§3.9, §4.1, §5 V6–V9; `intents_account` constructor parameter, `refund_in_flight`, `on_merchant_leg`, closed-mandate removal, three payer/deploy-only error strings, keeper gas) · Version 1.1 (audited, `audits/contracts-vault.md`) · ADR-0008 · Flow §6 · Stage 4 · Rust `near-sdk` 5

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
Init (once, at deploy): `allowed_tokens` (USDC, USDT on NEAR), `fee_receiver = treasury.nearpay.near`, `max_fee_bps = 500`, per-token `tvl_cap` and `mandate_cap` (v1.3, §3.8), `intents_account = intents.near` (v1.2; a parameter so sandbox/testnet can point at a stand-in). These are constructor parameters only — no setters exist.

**v1.2 additions to state.** `Mandate.refund_in_flight: U128` (amount of a `cancel`/`withdraw` refund awaiting its callback; 0 when none). `Vault` also holds NEP-145 `storage_accounts: LookupMap<AccountId, {total, mandates}>` and `payer_mandates: LookupMap<AccountId, IterableSet<u64>>` (index for `get_mandates_by_payer`). `tvl[token]` = Σ over the token's mandates of `balance + (in_flight ? amount_per_period : 0) + refund_in_flight` — the tokens the vault holds for mandates. A mandate that is closed (`Canceled`/`Exhausted`) with `balance = 0` and nothing in flight is **removed** from state (`get_mandate` → `null`), and its storage lock returns to the payer's NEP-145 available balance (§3.7).

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

**Error strings added in v1.2** (payer- or deploy-only paths; the keeper never sees them): `ERR_STILL_ACTIVE` (`withdraw` on an active mandate), `ERR_OPEN_MANDATES` (`storage_unregister` with open mandates), `ERR_BAD_CONFIG` (constructor). near-sdk's own messages remain for its guards: `Method <m> is private`, `Requires attached deposit of exactly 1 yoctoNEAR`, `Method <m> doesn't accept deposit`.

**Error strings added in v1.4:** `ERR_UNSUPPORTED_SENDER` (`ft_on_transfer` whose `sender_id` is not the transaction signer — v1.5 — or is `intents_account` or the vault itself; logged, refunded) and `ERR_GAS` (`collect` with less than `MIN_COLLECT_GAS`, or — v1.5 — `cancel`/`withdraw` with less than `MIN_EXIT_GAS`; the keeper sees the `collect` case, `billing.md` §7.1).

### 3.1 `msg` schema (v1.2, v1.4)

**Who may fund (v1.4, v1.5; AI audits A1-M-2 / A2-M-1, re-audit N-1).** The payer is the NEP-141 `sender_id`, and since v1.5 it **must equal the transaction signer** (`env::signer_account_id()`): a direct `ft_transfer_call` signed by the payer. Any deposit relayed by a contract — a bridge, router, aggregator or smart wallet — is refunded with `ERR_UNSUPPORTED_SENDER`, so the payer is always the human who can cancel and whom refunds reach. **Correction (v1.6, re-check 2 R-1):** v1.5 said NEP-366 meta-transactions pass; they do not. In a meta-transaction the inner receipt's predecessor is the user but its signer is the relayer (nearcore `apply_delegate_action`), so a meta-tx deposit is refused and refunded. **Gasless / relayer wallets cannot fund or top up mandates in v1.** `cancel` and `withdraw` do work via meta-transaction, because they authorise the predecessor, not the signer. **Smart-wallet, DAO and multisig payers are unsupported in v1.**

**Funding UX rule (v1.6).** Billing's `POST /v1/mandates/prepare` and the checkout auto-debit flow must tell payers to sign the funding (and top-up) transaction **from their own NEAR account with their own gas** — not through a relayer/gasless wallet, an Intents balance, a bridge or a smart wallet — and say that such deposits are refunded. Wallet selectors in that flow should offer only wallets that sign plain transactions. The explicit `intents_account` check below is subsumed but kept. A deposit whose `sender_id` is `intents_account` (Intents `ft_withdraw` with `msg`, 1Click `customRecipientMsg`) would make intents.near the payer — the human could never cancel and a refund (plain `ft_transfer` to intents.near) would credit nobody — so it is refunded with `ERR_UNSUPPORTED_SENDER`, checked right after the token allow-list and before parsing (create and top_up). Funding from Intents balances or other chains (SP-06) needs its own design (explicit payer and refund-to in `msg`, accepted only from `intents_account`, refunds as `ft_transfer_call(intents_account, msg = payer)`) and an ADR before it is enabled. `POST /v1/mandates/prepare` must tell payers to sign from their own NEAR account.

**Bounds (v1.4, A2-M-3; v1.5, re-audit N-3).** `period_secs` ≤ 12 × 366 days (379 468 800 s — every billing price up to `year × 12`, still below 2^31 − 1), `max_periods` ≤ 2^31 − 1, `start_at` ≤ block time + 366 days; larger values → `ERR_BAD_MSG`. Every number the vault accepts therefore fits int4/bigint columns and is exact in JavaScript. Number strings follow near-sdk's `U128`/`U64` parsing, which also accepts a leading `+` and leading zeros (`"+0010"`); events and views re-serialise canonically, so consumers only ever see canonical decimals.

```json
{"create_mandate":{"merchant_receiver":"m.near","receiver_kind":"intents_account","amount_per_period":"10000000",
  "period_secs":2592000,"start_at":"1731000000000000000","max_periods":12,"fee_bps":100,"ext_ref":"sub_123"}}
{"top_up":{"mandate_id":7}}
```
- Exactly one of the two keys; **unknown or duplicate fields are rejected** (`ERR_BAD_MSG`). `payer` = `sender_id` and `token` = the calling token contract — neither is read from `msg`; `fee_receiver` is the constructor's (ADR-0008's `create_mandate{…, fee_receiver, ref}` is superseded by §2; `ref` is `ext_ref`). A `msg` that names `fee_receiver`, `token` or `payer` is rejected.
- `receiver_kind`: `"intents_account"` | `"near_account"` (snake_case, also in views/events; status values `"active"`, `"canceled"`, `"exhausted"`). `amount_per_period` is a decimal string (u128), `start_at` a decimal string of **nanoseconds**, optional; an omitted, `null` or past `start_at` becomes the block timestamp (payer-favourable clamp). `period_secs` ≥ 86400, `max_periods` absent/`null` (unlimited) or ≥ 1, `fee_bps` ≤ `max_fee_bps` (else `ERR_FEE_TOO_HIGH`).
- `ext_ref` ≤ 64 bytes of printable ASCII (0x20–0x7E), may be empty. `merchant_receiver` ≠ the vault itself. The merchant share of one period must be > 0 (e.g. `amount_per_period = 1` with any fee is rejected). `msg` longer than 2048 bytes is rejected unparsed.
- Order of checks (first failure is the logged code): token allowed (panic `ERR_TOKEN_NOT_ALLOWED`; the token refunds) → sender (`ERR_UNSUPPORTED_SENDER`, v1.4) → `amount = 0` / parse / shape (`ERR_BAD_MSG`) → `ERR_FEE_TOO_HIGH` → zero share (`ERR_BAD_MSG`) → create: `ERR_NO_STORAGE` → per-mandate cap (`amount_per_period` and the funding, v1.3) → TVL cap; top_up: `ERR_NOT_FOUND` → `ERR_UNAUTHORIZED` (sender ≠ payer) → `ERR_TOKEN_NOT_ALLOWED` (other token than the mandate's) → `ERR_NOT_ACTIVE` → per-mandate cap → TVL cap. A rejection logs one plain line (no event) and returns the full amount; nothing else changes.
- **Rejection log line (v1.3):** the stable code, optionally followed by `": <detail>"`. Consumers match the text before the first `:`. Only `ERR_CAP_EXCEEDED` carries a detail: `ERR_CAP_EXCEEDED: mandate` (per-mandate cap) or `ERR_CAP_EXCEEDED: tvl` (per-token TVL cap). Every other rejection logs the bare code.

### 3.2 `collect` (v1.2)

- Check order: `ERR_GAS` (v1.4: less than `MIN_COLLECT_GAS` attached) → `ERR_NOT_FOUND` → `ERR_MAX_PERIODS` (status `Exhausted`) → `ERR_NOT_ACTIVE` (`Canceled`) → `ERR_IN_FLIGHT` → `ERR_MAX_PERIODS` (defensive) → `ERR_NOT_DUE` → `ERR_INSUFFICIENT_BALANCE`. Not-due wins over insufficient balance, so the keeper falls back to `send_invoice` only for a period that is actually due.
- Periods are 1-based: period *n* is due at `start_at + (n − 1) × period_secs` (period 1 at `start_at`). One period per call; a late keeper may catch up with consecutive calls (never more than the elapsed periods, V3).
- On success `amount_per_period` moves out of `balance` immediately (`in_flight = true`), so a concurrent `cancel`/`withdraw` can never refund money that is in flight; the callback credits back whatever did not leave.
- **Callback chain.** A NEAR `then` runs regardless of the previous result, so a token call cannot be made conditional by chaining alone. The chain is: merchant leg → `on_merchant_leg(mandate_id, share, fee)` (private) → only if the full share was delivered and `fee > 0`: fee leg `ft_transfer(fee_receiver, fee)` → `on_collected(mandate_id, share, fee)` (private); otherwise the period is settled in `on_merchant_leg`. Callbacks recompute share/fee from the immutable mandate terms (the arguments are informational).
- **Merchant-leg outcome** (promise result, read bounded to 64 bytes): failed → delivered 0. `near_account` success → full share. `intents_account` success → the `U128` returned by `ft_transfer_call` (= used amount after `ft_resolve_transfer`), capped at the share; an unparseable or over-long success value is treated as the full share (the token reported success; assuming the tokens left keeps V1, at the cost of leaving them unattributed in the vault if they did not).
- **Settlement.** Full share: `collected_periods += 1`, emit `collected`; then `exhausted` if `max_periods` is reached and the mandate is still `Active`. **Partial** share (0 < used < share — not expected from `intents.near`): the period **counts** with the delivered amount, no fee is sent, the rest stays in `balance` (keeps V3; billing compares `merchant_share` with the expected share). **Nothing** delivered: period not counted, full amount credited back, emit `collect_failed {reason: "merchant_transfer_failed"}`. **Fee leg failed:** the period counts (the merchant was paid), the fee stays in the mandate's `balance` (payer-favourable, never retried), `collected.fee = "0"`.
- Keeper contract: a `collect` transaction either fails with one of the error strings above, or succeeds with exactly one of `collected` (optionally followed by `exhausted`) or `collect_failed` in its receipts' logs. The final return value is `true` when the period was counted.

### 3.3 Gas (v1.2, re-derived in v1.4)

`collect` is permissionless and the caller chooses the gas, so every budget that matters is **static** (independent of the attached gas) and `collect` refuses too little gas (AI audits A1-M-1 / A2-M-5 / A1-L-1 / A2-L-5).

| Receipt | Static gas | Why |
|---|---|---|
| merchant `ft_transfer_call` (`intents_account`) | 100 Tgas | deployed tokens (near-contract-standards 4.x logic) require `> 30 Tgas` and forward exactly `prepaid − 30 Tgas` to the receiver: `intents.near::ft_on_transfer` gets ≥ 70 Tgas whoever calls (NEAR Intents recommends 100 Tgas for a whole deposit) |
| merchant `ft_transfer` (`near_account`), fee leg, refunds | 15 Tgas | plain NEP-141 transfer |
| `on_merchant_leg` | 50 Tgas | its work plus scheduling fee leg (15) and `on_collected` (20) |
| `on_collected`, `on_refunded` | 20 Tgas each | worst case (exhaust/close) measured at ≈ 1.1 Tgas host gas under mainnet fees, plus wasm |

`MIN_COLLECT_GAS = 170 Tgas` (100 + 50 static + 20 for `collect` itself); below it `collect` fails with `ERR_GAS` before any state change. Gas above the static amounts is spread by weight (merchant leg first). **The keeper attaches 250 Tgas** (`billing.md` §7; within the protocol's prepaid-gas limit — 1 PGas since protocol 83, 300 Tgas before; check the live protocol version at deploy; unused gas is refunded). The guarantee is the **static gas on each receipt**: a receipt with less cannot be created, so the whole call reverts before any write. `MIN_COLLECT_GAS` only turns an under-gassed call into an early, named `ERR_GAS` (re-audit N-6); never lower static budgets "because the floor covers it". Payer-signed `cancel`/`withdraw` need 35 Tgas static plus their own work (≈ 40 Tgas measured in CI); `MIN_EXIT_GAS = 50 Tgas` (§3.4).

The §7 target "`collect` ≤ 60 Tgas" applies to gas **burnt by the vault's design** — the vault's and the token's receipts — and excludes the merchant receiver's own `ft_on_transfer` (the Verifier's cost; the sandbox mock burns 30 Tgas to stand in for it). **Before mainnet deployment (human):** measure the real `intents.near::ft_on_transfer` cost of a USDC/USDT deposit (testnet or a mainnet dry run) and confirm it is well under 70 Tgas; the sandbox cannot import mainnet `intents.near`/USDC with their state.

### 3.4 `cancel` / `withdraw` (v1.2)

- **Gas (v1.5, re-audit N-4):** both refuse less than `MIN_EXIT_GAS = 50 Tgas` with `ERR_GAS` (checked first, no state change). Payers attach ≥ 50 Tgas, **100 Tgas recommended**; many tools default to 30 Tgas (near-api-js, near-cli JS, explorer forms), which now fails with the explainable `ERR_GAS` instead of "Exceeded the prepaid gas". NearPay's frontend attaches 100 Tgas; payer help text and the spec state the requirement.
- Both require exactly 1 yoctoNEAR (a full-access key of the payer/merchant). `cancel` checks `ERR_GAS` → `ERR_NOT_FOUND` → `ERR_UNAUTHORIZED` → `ERR_NOT_ACTIVE`; `withdraw` checks `ERR_GAS` → `ERR_NOT_FOUND` → `ERR_UNAUTHORIZED` (payer only) → `ERR_STILL_ACTIVE` → `ERR_IN_FLIGHT` (a refund is pending) → `ERR_INSUFFICIENT_BALANCE` (nothing to withdraw).
- `cancel` sets `Canceled` at once and refunds `balance` (which excludes an in-flight collect). The `canceled {by, refunded}` event is emitted **when the refund outcome is known** (`refunded` = amount actually returned, `"0"` on failure; immediately with `"0"` when the balance is zero). On a failed refund the amount returns to `balance` for `withdraw`; `withdrawn` is emitted only on success (a failed withdraw restores the balance silently).
- A collect in flight at cancel time still settles normally (the period was due); its credit-back lands in `balance` for `withdraw`.
- Merchant cancel requires `merchant_receiver` to be a NEAR account that can sign. Intents accounts that cannot (e.g. EVM-keyed implicit accounts without a wallet contract) cannot cancel; the payer always can (A2-I-5).

### 3.5 Constructor (v1.2)

`new(allowed_tokens, fee_receiver, max_fee_bps, tvl_caps: [{token, cap, mandate_cap}], intents_account)` is `#[init] #[private]` (called by the vault account in the deploy batch, before its keys are deleted). Rejected with `ERR_BAD_CONFIG`: no tokens, duplicate tokens, not exactly one cap entry per allowed token, a cap for an unknown token, `mandate_cap > cap` (v1.3), a zero `mandate_cap` (hence a zero `cap`), `fee_receiver = intents_account`, `fee_receiver` or `intents_account` among the tokens (v1.4, A2-L-2), `max_fee_bps > 10000`, `fee_receiver`/`intents_account`/a token equal to the vault.

### 3.6 Views (v1.2)

`get_mandate(mandate_id) → MandateView | null` (`{mandate_id, payer, merchant_receiver, receiver_kind, token, amount_per_period, period_secs, start_at, max_periods, fee_bps, collected_periods, balance, status, in_flight, refund_in_flight, ext_ref, next_collect_at}`), `get_mandates_by_payer(payer, from_index?, limit?)` (open mandates only, default 50, max 100, stable but unordered), `next_collect_at(mandate_id) → ns string | null` (null unless `Active` below `max_periods`), plus `get_config()` (constructor parameters, incl. `caps: [{token, cap, mandate_cap}]` in `allowed_tokens` order) and `get_token_tvl(token) → {token, tvl, cap, mandate_cap}`.

### 3.7 Storage (v1.2)

Registration minimum (`storage_balance_bounds.min`) 0.002 NEAR; each open mandate locks 0.01 NEAR; `available = total − 0.002 − 0.01 × open mandates`. Both cover worst-case bytes (64-char account ids, 64-byte `ext_ref`; asserted by a unit test). Storage is released when the mandate is removed — on the successful refund of `cancel`/`withdraw`, or on the collect that exhausts a mandate to zero — into the payer's available balance (`storage_withdraw`, 1 yoctoNEAR). `storage_unregister` (1 yoctoNEAR) returns everything once the account has no open mandates (`ERR_OPEN_MANDATES` otherwise; `force` is not supported). `storage_withdraw` errors: `ERR_NO_STORAGE` (not registered), `ERR_INSUFFICIENT_BALANCE` (more than available).

### 3.8 Per-mandate cap (v1.3)

ADR-0008 and T-10 require a per-mandate cap in v1 besides the per-token TVL cap. Each allowed token has a constructor-only `mandate_cap` (`≤ cap`; no setter), stored in `Vault.mandate_cap: LookupMap<AccountId, U128>`:
- `create_mandate` is refunded with `ERR_CAP_EXCEEDED: mandate` if `amount_per_period > mandate_cap` or the funding `amount > mandate_cap`.
- `top_up` is refunded with `ERR_CAP_EXCEEDED: mandate` if the mandate's held amount after the deposit — `balance + (in_flight ? amount_per_period : 0) + amount` — exceeds `mandate_cap`. The in-flight period counts because a failed merchant leg credits it back.
- Collect credit-backs only return tokens the mandate already held, so a mandate never holds more than its cap (V10). The cap applies to deposits only: a mandate created under a cap stays valid; there is no way to change caps after deploy.
- The deploy values are an owner decision for the deploy runbook (§6), as with `tvl_cap`.

### 3.10 Callback failure (v1.4)

There is **no recovery** for a callback that fails (out of gas, or a future protocol fee change on an immutable contract): its state changes are reverted while the token transfer before it stands (A1-L-1). A failed `on_merchant_leg`/`on_collected` leaves `in_flight = true` forever: further collects fail with `ERR_IN_FLIGHT`, the period is never counted or credited back, the payer can still cancel and recover `balance` but not the in-flight amount, the mandate never closes (storage lock kept, `storage_unregister` → `ERR_OPEN_MANDATES`) and `tvl` keeps counting the period. A failed `on_refunded` leaves `refund_in_flight` set and blocks `withdraw` forever. No permissionless "unstick" is safe, because the vault cannot learn the token outcome afterwards. The protection is margin: the §3.3 static budgets are 2.5–10× the burn measured in CI (`on_merchant_leg` ≈ 39 of 50 Tgas incl. its children's static gas, `on_collected` 1.9 of 20, `on_refunded` 2.5 of 20, transfers 1.7 of 15), the static gas on each receipt guarantees them (a receipt with less is never created; the gas floors only make the failure early and named), and the sandbox tests run every path at the minimum accepted gas and assert the callbacks succeeded (`integration-tests/tests/min_gas.rs`). The native tests `failed_collect_callback_leaves_the_period_stuck` and `failed_refund_callback_blocks_withdraw_forever` document the resulting state.

### 3.11 Accepted risks (v1.4)

Recorded with the AI audits; no code change:
- **TVL-cap squatting** (A1-L-3, A2-M-2): one capital holder can fill a token's cap with refundable mandates and block new mandates and top-ups at no loss. The `start_at` bound does **not** mitigate it (re-audit N-9): a due mandate paying the attacker 1 base unit per long period holds the cap just as well. Per-payer limits are sybil-able. Mitigation: monitor `tvl/cap`, mandates with `merchant_receiver = payer`, tiny `amount_per_period` relative to the balance, and far `start_at`; keep the `v2.vault…` deployment runbook pre-staged (§6).
- **Unparseable merchant-leg success counts as delivered** (§3.2; A1-I-1): solvency over fairness; tokens actually refunded would stay unattributed. Allowed tokens return `U128` strings.
- **NEP-141 resolve assumption** (A1-L-4): a failed `ft_transfer_call` is read as "nothing moved"; a token whose `ft_resolve_transfer` itself fails after the receiver kept the tokens would break V1 by one share. Only audited near-contract-standards tokens are allowed.
- **Token blocklists and pauses** (A1-I-3): if a token blocks the payer, refunds fail and stay withdrawable later; if it blocks the vault or the treasury, mandates in that token freeze or fees stay on mandates. Listed in T-10's residual risk.
- **Stray tokens** (A2-I-4): a plain `ft_transfer` to the vault (no call) is never attributed and cannot be recovered (no admin, V4).
- **Partial delivery counts the period** (§3.2; A1-I-2, A2-L-1): kept as specified (no fee, remainder stays with the payer, billing handles the short share — `billing.md` §7.1). ADR-0008 ("marks the period failed (not advanced) if the merchant share was not fully delivered") and `quality/stress-test.md` F-22 say the opposite; **an ADR-0008 amendment is needed** so the code is not "fixed" back.
- **Pagination** (A2-I-3): `get_mandates_by_payer` is stable only between writes (swap-remove on close).

### 3.9 Spec

The event, view and `msg` shapes in §3.1, §3.6 and §4.1 are the source for the `nearpay-spec` vault schemas of step 4.1.

**As specified (spec 0.4.0, step 4.1 auto-debit contracts):** `nearpay-spec/src/vault/contract.ts` mirrors events (`VaultEvent`, `formatVaultEventLog`), the 17 error strings in `errors::ALL` order, check orders, `msg` and view shapes, limits and gas floors. `nearpay-spec/test-vectors/vault.json` holds every event line, error string, rejection line and view result byte for byte; the spec's test suite checks it against strings produced by this contract's own code (`events::format`, serde of the view structs, `errors::ALL`) at `nearpay-contracts` `7e2585a`. **Follow-up for `nearpay-contracts`:** a unit test that reads `test-vectors/vault.json` (vendored or from the published `@nearpay/spec` package) and asserts `events::format(...)` and the view serialisations equal it, so a contract change that alters the wire fails CI on both sides.

## 4. Events (NEP-297, `standard: "nearpay_vault"`, `version: "1.0.0"`)

`mandate_created {mandate_id, payer, merchant_receiver, token, amount_per_period, period_secs, start_at, max_periods, fee_bps, ext_ref}` · `topped_up {mandate_id, amount, balance}` · `collected {mandate_id, period, amount, merchant_share, fee}` · `collect_failed {mandate_id, period, reason}` · `canceled {mandate_id, by, refunded}` · `withdrawn {mandate_id, amount}` · `exhausted {mandate_id}`.

### 4.1 Wire format (v1.2)

One log line per event: `EVENT_JSON:{"standard":"nearpay_vault","version":"1.0.0","event":"<name>","data":[{…}]}` — `data` is a one-element array (NEP-141/171 convention), fields in the order listed. Encoding: `mandate_id`, `period`, `period_secs`, `max_periods` (or `null`), `fee_bps` are JSON numbers; amounts and `balance` are decimal strings; `start_at` is a decimal string of nanoseconds; accounts are strings. Additive fields: `mandate_created` ends with `receiver_kind` and `balance` (the initial funding — billing's mirror needs both; billing's linkage rule should also require `receiver_kind = "intents_account"`); `collected` ends with `balance` (after the period). `collected.amount = merchant_share + fee` actually paid (see §3.2 for partial delivery and fee-leg failure). `collect_failed.reason` is `"merchant_transfer_failed"` (the only reason in v1). Example:
```
EVENT_JSON:{"standard":"nearpay_vault","version":"1.0.0","event":"collected","data":[{"mandate_id":0,"period":1,"amount":"10000000","merchant_share":"9900000","fee":"100000","balance":"20000000"}]}
```
`ft_on_transfer` rejections are not events: a plain log line with the error string.

**Validity (v1.4, A2-M-4).** NEAR keeps the logs of failed receipts, and a payer can make a vault receipt log an event and then run out of gas. An event is valid **only** when it is in a successful receipt (`SuccessValue`/`SuccessReceiptId`) executed by the vault account itself; consumers must check both (`rails.md` §8.6). The `standard` field alone proves nothing.

## 5. Invariants (property-tested and fuzzed)

V1 Σ balances + Σ in-flight ≤ token balance of the vault. V2 a payer can always reach `Canceled` and recover `balance`. V3 per mandate, Σ collected ≤ `collected_periods × amount_per_period` and periods ≤ elapsed periods. V4 no method transfers to any account other than payer, merchant_receiver (via intents or directly) and fee_receiver. V5 no method changes `merchant_receiver`, amount, period or fee after creation.

v1.2 precision and additions: V1 holds with "in-flight" = in-flight collects (`amount_per_period`) + in-flight refunds, and `tvl[token]` equals that sum exactly; between the receipts of one collect, tokens that already left are subtracted from the in-flight term. V3's Σ collected counts the in-flight period once its tokens have left. V5 also covers payer, token, receiver kind, `start_at`, `max_periods` and `ext_ref`. V6 `in_flight` ⇔ exactly one collect awaiting callbacks; `refund_in_flight` = the pending refund. V7 `tvl ≤ tvl_cap` per token. V8 each payer's storage lock count = its open mandates = its index entries. V9 `collect` succeeds iff its §3.2 preconditions hold, else fails with the documented code. V1–V10 are unchanged in v1.4 and v1.5; the property tests now also cover callback failure (documented stuck states) and every audit PoC is a regression test. V10 (v1.3) per mandate, `balance + in-flight collect + in-flight refund ≤ mandate_cap[token]` and `amount_per_period ≤ mandate_cap[token]`. Where each is tested: `nearpay-contracts/audits/README.md`.

## 6. Deployment

Build reproducibly (`cargo near build reproducible-wasm`), publish wasm hash, deploy + `new(...)`, then delete all access keys from `v1.vault.nearpay.near` (human-signed). Upgrades = new account `v2.vault…` with payer opt-in migration (cancel + recreate).

**Runbook (v1.4, A2-L-4, A2-L-2, A2-M-7):**
1. Take the wasm and its sha256 **only from a `push` run on `main`** (or a release tag) whose **overall conclusion is `success`** — never from a `pull_request` run (re-audit N-2: cargo-near embeds the source commit in the wasm's NEP-330 metadata, so a PR run's hash belongs to a commit that may not survive the merge and cannot be verified later). The hash to use is the one printed as `DEPLOYABLE` by the final `deployable` job, which `needs:` every other job (lint, unit, wasm, integration, reproducible, fuzz) and runs only on push to main (re-check 2 R-2); the `reproducible` job alone only shows a candidate. That run built the wasm twice with equal hashes and ran the sandbox suite against these bytes. Record the hash with the commit and the run URL.
2. Before deploying, check on **every** allowed token: `storage_balance_of(treasury.nearpay.near)` and `storage_balance_of(intents.near)` are registered, and register `v1.vault.nearpay.near` itself (`storage_deposit`). An unregistered treasury makes every fee leg fail forever.
3. Owner decides the caps (`cap`, `mandate_cap` per token, both > 0) and `max_fee_bps`.
4. **One batch transaction**, signed by the vault account's full-access key: `DeployContract(wasm)` + `FunctionCall new(...)` + `DeleteKey` for every access key. There is no window in which the code can be replaced.
5. Publish `view_code` hash = the reproducible hash and `view_access_key_list` = `[]`, and call `get_config` to show the constructor values.
6. Fund the account so the code and base state are covered (≈ 3 NEAR + buffer); payers fund their own mandates' storage.
7. Keep the `v2.vault…` runbook (same steps, new account) ready for the accepted risks in §3.11.

## 7. Tests

Unit (`near-sdk` unit env), integration (`near-workspaces` sandbox with mock FT + mock `intents.near` receiver), proptest sequences for V1–V5, cargo-fuzz on `msg` parsing, gas profile per method (target `collect` ≤ 60 Tgas total). Two independent audits before mainnet (Stage 4 exit).

v1.2: unit tests run each call as the chain does (state written and re-read per call, reverted on panic); proptests interleave every method with randomly delayed and failing promise outcomes against a token-ledger model (V1–V9) and end with a drain that must close every mandate; the parser is fuzzed natively (proptest, with an independent schema oracle) and by a cargo-fuzz target in CI. The mock FT enforces the `> 30 Tgas` `ft_transfer_call` rule of deployed tokens. Wasm builds, sandbox tests, the gas profile and the reproducible build run only in CI (`.github/workflows/ci.yml`). ADR-0020: the two audits are independent AI-agent audits.
