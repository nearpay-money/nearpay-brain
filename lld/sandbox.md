# LLD — `nearpay-sandbox`

> Version 1.6 (step 4.1, spec 0.4.0: Solana and Bitcoin simulation — address and hash formats, `SimTx` chain fields, confirmations, scenarios `08`/`09`, Solana USDT — §10) · Version 1.5 (step 3.2b, spec 0.3.1: `SimTx.transfers[].from` §3, documented test addresses of the simulated payer and liquidity provider §4.1) · Version 1.4 (step 2.2 implementation: live 1Click shape details §3, Intents deposits by Verifier transfer §3, `/sim/intents/transfers` events §3, Explorer mirror rules §3, scenario `06` for refund legs §4, data §5, fixed prices and test-mode funds §8) · Version 1.3 (step 2.1, spec 0.2.0: refund legs — `scenario_group`/`scenario_kind` §3–§4, `POST /sim/intents/transfers` §3, Stage 2 §8; Explorer mirror and live 1Click shape §3) · Version 1.2 (step 1.2 implementation findings folded: quote signature, appFees split and scenario key §3, Verifier events §3, scenario 06 §4, configuration §6, ops CLI §9) · Version 1.1 audited (`audits/sandbox.md`) · ADR-0007 · API conventions §8 · Foundations `00-foundations.md`

## 1. Scope

Deterministic simulator for test mode, local development and E2E: a **1Click-compatible API**, simulated chains (evidence lookups), and a simulated Verifier (`intents.near`) for signed transfers. Rejects every `livemode=true` call.

## 2. Runtime

`sandbox-api` (1 replica, own DB `sandbox`). Exposed only on the `internal` network; every call requires a service token with `sandbox.use` (payments, rails, agentpay, sentinel); test helpers reach it through payments.

## 3. Endpoints

| Endpoint | Mirrors | Behaviour |
|---|---|---|
| `GET /v0/tokens` | 1Click | fixed registry using real asset IDs for Base/Arbitrum/Ethereum/NEAR/Solana USDC, USDT, ETH, wNEAR, BTC with fixed prices |
| `POST /v0/quote` | 1Click | validates the same schema; `amountIn = ceil(amountOut × price ratio × (1 + appFee + 20 bps))`; deterministic deposit address per chain format (`0x` + keccak(seed)[12..32] for EVM, `sbx-<hash>.near` for NEAR, base58 for Solana, bech32 for BTC); `signature` = Ed25519 (sandbox key) with the 1Click construction (the SDK's `quoteHash`), so `verifyQuoteSignature(quote, sandboxKey)` verifies it (SP-01); like 1Click, the signed request leaves `appFees` out and the response echoes `appFees` split 50/50 with a fee account of its own (rails §8.1); the scenario comes from `scenario_amount`, which rails sets to the payment's pricing amount (spec 0.1.1) |
| `GET /v0/status` | 1Click | scenario timeline (below) |
| `POST /v0/deposit/submit` | 1Click | marks deposit detected |
| `GET /sim/chains/{caip2}/tx/{hash}` | chain RPC | `{finalized, transfers:[{token, from, to, amount}]}` for sandbox verifiers; `from` (spec 0.3.1) is the sender per §4.1 — rails fills `Evidence.sender` from it (rails §20.1); rows recorded before v1.5 answer `from: null` (unknown) |
| `POST /sim/deposits` | — | `{deposit_address, memo?, amount_in?}` simulate the payer's deposit (used by checkout test-mode "Simulate payment" via payments) |
| `POST /sim/advance` | — | `{deposit_address, to: deposit\|processing\|success\|refund\|fail}` step control |
| `POST /sim/intents/simulate` · `/execute` | `simulate_intents` / `execute_intents` | real NEP-413 / ERC-191 / raw Ed25519 signature verification (`@nearpay-internal/intents`), nonces, simulated balances per account (seeded via `/sim/intents/fund`), mainnet rejection messages, atomic batches; named accounts register the first key that signs for them. Emits DIP-4 / NEP-245 events shaped like mainnet: the binding memo of a transfer intent is on the DIP-4 `transfer` event only, the NEP-245 `mt_transfer` carries none (SP-14) |
| `GET /sim/intents/events?cursor=` | indexer feed | event stream for rails' sandbox indexer adapter |
| `POST /sim/intents/transfers` (Stage 2) | — | `{from, to, token, amount, memo?}`: an executed transfer without a signature (credits `from` with any shortfall first — an `mt_mint` — then emits the same NEP-245 `mt_transfer` (no memo), DIP-4 `transfer` (with the memo) and `intents_executed` events as a signed execute, with a synthetic intent hash and nonce). Serialised with executes (the Verifier lock). `amount = 0` → `400` (`amount`), `from = to` → `400` (`to`). Used by payments' refund test helper for Intents legs and INTENTS-deposit route legs; test mode only by construction |
| `GET /api/v0/transactions` (Stage 2) | Explorer `/api/v0/transactions` | `ExplorerTransactionsQuery` → `ExplorerTransactionList`: the sandbox's quotes as Explorer rows, newest first, with the scenario's current status; filters `referral`, `statuses` (default without `*_DEPOSIT`, like the Explorer), `startTimestamp`/`endTimestamp` (exclusive, on creation time), `numberOfTransactions` ≤ 1000 and the `lastDepositAddress`/`lastDepositMemo` cursor; `amountIn`/`amountOut` read `0` until settled; no rate limit. Lets rails run its Explorer feed in test mode (rails §8.5). Rules (v1.4) below |

**Quote fields (Stage 2):** `SandboxQuoteRequest.scenario_group` (rails sets `pi_…` for payment attempts — from `CreateExecution.reference` — and `re_…` for refund legs) and `scenario_kind` (`payment` | `refund`). For refund legs rails sets `scenario_amount` to the refund amount (source base units), so a merchant picks a refund scenario with the amount's last two digits.

**Intents deposits (v1.4).** A Verifier transfer — a signed `execute` (agentpay relays the merchant's `sign_intent` refund leg) or `POST /sim/intents/transfers` — into the deposit address of a `depositType: INTENTS` quote is that quote's deposit, as 1Click sees transfers inside `intents.near`: the deposit is recorded with the Verifier transaction hash (`originChainTxHashes`) and the scenario timeline starts, including scenario `06` duplicates; a later `POST /v0/deposit/submit` with the same hash is a no-op. A transfer of another asset, or into a quote that already has a deposit or is final, is not attributed (logged; the funds stay on the deposit account). Only `INTENTS`-deposit quotes are matched (an `ORIGIN_CHAIN` NEAR deposit is an `ft_transfer`, not a Verifier transfer). Quote rows are locked before balances move (the materializer's lock order: quote row, then balances).

**Explorer mirror rules (v1.4).** `status` is the stored provider status (at most one materializer tick behind `GET /v0/status`) in the Explorer's vocabulary: `KNOWN_DEPOSIT_TX` reads `PROCESSING` (the Explorer does not list it), so `statuses=PROCESSING` matches both. `amountIn` is the deposit once the execution is final (`SUCCESS`, `REFUNDED`, `FAILED`), `amountOut` the delivered amount once it is `SUCCESS`; otherwise `"0"` with `*Formatted`/`*Usd` `"0"`. `appFees` are the echoed split fees (`null` without), `refundFee` the quote's, `refundReason` the execution's; `intentHashes`, `depositAddressAndMemo` are `null` and `senders` `[]` (the sandbox does not track senders); tx hash arrays are plain hashes (origin = deposit txs, destination = payout txs). `search` matches the deposit address, recipient, refund address or any of its tx hashes; `depositMemo` filters exactly; `showTestTxs` is ignored. Undocumented `statuses` values match nothing; an unknown cursor returns `[]`; `direction=prev` pages towards newer rows from the cursor (still returned newest first) and starts at the newest without one.

**Live 1Click shape (spec 0.2.0):** the `OneClick*` schemas accept `null` for unset optional fields and a status-nested `quoteResponse` without `correlationId` (`OneClickStatusQuoteResponse`). The sandbox serves the live shape: unset optionals as `null` (e.g. `swapDetails.amountOut` before settlement, `appFees[].limitOrderId`), and `GET /v0/status` nests the quote response without `correlationId` and its `quoteRequest` without `quoteWaitingTimeMs`, so adapters are tested against what mainnet sends (`roadmap/open-questions.md`, mainnet findings). Details as implemented (v1.4), parity-tested key by key against the recorded mainnet `PENDING_DEPOSIT` status body (spec fixture `oneclick-status-pending-live.json`): `swapDetails` always carries `depositedAmount*`, `amountIn*`, `amountOut*`, `slippage`, `refundReason` (`null` while unset), `refundedAmount*` (`"0"` until a refund), `refundFee` and `withdrawFee` (the quote's), and no `referral`; the request echo (quote and status) adds `virtualChainRecipient: null`, `virtualChainRefundRecipient: null` and `confidentiality: "public"` when unset, as live 1Click does — none of these change the signature (the SDK hashes nulls as unset and does not sign `confidentiality`). `scenario_amount`, `scenario_group` and `scenario_kind` are never echoed. `/v0/tokens` is unchanged (no recorded live body; native tokens keep omitting `contractAddress`).

## 4. Scenario engine

Scenario key = last two digits of `scenario_amount` (the fiat amount in minor units, or the asset amount in base units for asset-priced intents). The outcome also follows the deposit amount like 1Click `EXACT_OUTPUT`: below `minAmountIn` → refunded, above `amountIn` → excess refunded, after the deadline → refunded.

| Suffix / magic | Timeline after deposit |
|---|---|
| `00` or unlisted | `KNOWN_DEPOSIT_TX` (1 s) → `PROCESSING` (2 s) → `SUCCESS` (2 s), destination tx with exact amount |
| `01` | same, `SUCCESS` after 120 s |
| `02` | `INCOMPLETE_DEPOSIT` → `REFUNDED` at deadline (`refundReason: PARTIAL_DEPOSIT`) |
| `03` | `SUCCESS` with `refundedAmount` = excess |
| `04` | `FAILED` after `PROCESSING` |
| `05` | `SUCCESS` but `/sim/chains` returns amount − 1 (short) |
| `06` | like `00`; a deposit also pays the older open quotes of the same payment (duplicate settlement → overpayment). Stage 2: quotes are grouped by `scenario_group` when present; on refund legs a deposit to one leg also settles the other open legs of the same refund (duplicate refund → payments records a `duplicate` leg). Without `scenario_group` the Stage 1 grouping applies. As implemented (v1.4): group key = `sha256("group|" + kind + "|" + scenario_group)`, else the Stage 1 `sha256(recipient|destinationAsset|refundTo|scenario_amount)`; a payment deposit pays **older** open attempts only (Stage 1), a refund deposit pays **every other** open leg, older or newer; a refund leg's duplicate deposit counts as made **in time**, so an expired leg (deadline passed, still `PENDING_DEPOSIT`) settles `SUCCESS` — the "paid the expired leg and its replacement" case payments must see — while an expired payment attempt is still refunded (`DEPOSIT_AFTER_DEADLINE`, Stage 1). Every scenario applies to refund legs by the refund amount's last two digits (e.g. `02` underpays the leg → the merchant is refunded at the deadline) |
| `07` | success arrives 5 min after the session's `expires_at` |
| `08` (Stage 4) | slow finality: the destination tx is recorded not final for 60 s (§10) |
| `09` (Stage 4) | Solana: a failed destination tx precedes the real one; Bitcoin: the payout is split over two outputs (§10) |
| refund address contains `sanctioned` | compliance blocks (handled by compliance sandbox rule) |

### 4.1 Test addresses and transfer senders (v1.5)

Documented test addresses of the **simulated payer** (adopted from payments' test helper, payments §21.5, so one set exists platform-wide; `SIMULATED_PAYER` in `src/simulated-chains/addresses.ts`):

| Family | Address |
|---|---|
| EVM | `0x000000000000000000000000000000000000dEaD` |
| NEAR | `sandbox-payer.near` |
| Solana | `11111111111111111111111111111111` |
| Bitcoin | `bc1qar0srrr7xfkvy5l643lydnw9re59gtzzwf5mdq` (a chain with another bech32 HRP gets an address derived from seed `sandbox-payer`) |

The **simulated liquidity provider** is `sbx-solver.near` on NEAR (the `mt_transfer` `old_owner_id` / DIP-4 `account_id` of every Intents payout) and, on other chains, the address derived like a deposit address from seed `sbx-solver` (`solverAddress(chain)`).

Sender (`from`) of each simulated transfer recorded in `sim_txs`:

| Transfer | `from` |
|---|---|
| Deposit by `/sim/deposits`, `/v0/deposit/submit`, `/sim/advance` or a scenario `06` duplicate | the quote's `refundTo` when the refund address is on the deposit chain (`refundType: ORIGIN_CHAIN`, or `INTENTS` with a NEAR-origin quote) — the wallet that pays is the one refunds return to; otherwise the simulated payer of the origin chain |
| Deposit into an `INTENTS`-deposit quote by a Verifier transfer (`/sim/intents/execute`, `/sim/intents/transfers`) | the transfer's signer / `from` (a NEAR account) |
| Payout at `SUCCESS` and refund at `REFUNDED` / excess | the simulated liquidity provider (`sbx-solver.near` for Intents recipients, consistent with the indexer events rails attributes) |

On mainnet a 1Click delivery's sender is likewise the delivering solver, never the payer's origin wallet (rails §20.1).

## 5. Data

`sim_quotes(deposit_key pk, request jsonb, response jsonb, scenario text, state text, timeline jsonb, created_at)` (+ Stage 2, migration `0002`: `scenario_group text` null, `scenario_kind text` `payment`|`refund` default `payment`; indexes on `deposit_address` (Intents deposits) and `(request->>'referral', seq)` (Explorer mirror)), `sim_txs(hash pk, chain, transfers jsonb, finalized_at)`, `sim_intents_balances(account_id, token, amount numeric(78,0), primary key(account_id, token))`, `sim_intents_events(seq bigserial pk, payload jsonb)`.

## 6. Configuration

`NP_SANDBOX_SIGNING_KEY_FILE` (public key distributed to rails/sentinel as `NP_SANDBOX_SIGNING_PUBKEY`), `NP_SANDBOX_TIME_SCALE` (1.0; testbed may set 0.1), `NP_SANDBOX_TICK_MS` (200; how often due timeline steps are applied), `NP_JWKS_URL` (service tokens).

## 7. Tests

Schema parity: sandbox request/response validated against the same zod schemas as the 1Click adapter (contract test); each scenario timeline; signature verification by rails/sentinel sandbox verifiers.

## 8. Stage mapping

Stage 1: all of the above except scenario `06` duplicate behaviour for refunds (Stage 2). Stage 2 (implemented in step 2.2): `scenario_group` / `scenario_kind` on quotes, `06` for refund legs, `POST /sim/intents/transfers`; refund legs quote `EXACT_INPUT` (amountOut = `floor(amountIn × price ratio × (1 − 20 bps))`) and `EXACT_OUTPUT` like payments, with no app fee; the payer side of a refund settles on `/sim/chains` (route legs) or the simulated Verifier (Intents legs) so rails verifies payer-side evidence; the Explorer mirror (`GET /api/v0/transactions`) and the live 1Click body shape.

**Fixed prices for rails test mode (v1.4, rails §8.2).** Rails reads its test-mode primary *and* secondary token prices from sandbox `GET /v0/tokens` (`price` + `priceUpdatedAt` = now, so always fresh): USDC/USDT 1, ETH 2500, wNEAR 4, SOL 150, BTC 80000 USD (`src/domain/tokens.ts`), the same prices every quote uses. Fiat FX is rails' own fixed table `NP_TEST_FX_RATES` — the sandbox serves no FX. The registry has no EUR stablecoin, so test mode cannot exercise EUR par settlement (`NP_STABLE_PEGS`); EUR-priced test payments settle non-par in USD stablecoins. Adding one needs its real NEAR Intents asset id from live `/v0/tokens` (not reachable from the build container).

**Merchant funds for Intents-source refunds (v1.4).** A merchant's simulated Intents balance is what its test payments settled into its Intents account (payouts credit the simulated ledger), so a merchant-signed `sign_intent` refund leg executes end to end from it. An `EXACT_INPUT` (payer-absorbs) refund never needs more than was settled; an `EXACT_OUTPUT` (merchant-absorbs) full refund needs the 20 bps route cost on top, which the testbed funds with `POST /sim/intents/fund` or the ops CLI `intents-fund`. The Verifier stays strict (`insufficient balance or overflow`); only `/sim/intents/transfers` mints shortfalls.

## 9. Ops CLI

`docker compose exec sandbox-api /nodejs/bin/node dist/ops.js <command>`: `intents-fund <account> <token> <amount>` credits a simulated Intents balance (the same `mt_mint` as `POST /sim/intents/fund`; `seed.sh data` funds the demo agent with it, platform §3) · `intents-balance <account> <token>`. Test mode only by construction (the sandbox holds no live data); arguments are validated before the ledger is touched. Missing: a tx-status route, so agentpay cannot recover a sandbox execute that failed in transport (agentpay §7).

## 10. Stage 4 — Solana and Bitcoin simulation (step 4.1, spec 0.4.0)

Rails treats both chains as Tier A (rails §21); test mode must exercise the same verifier rules, so the simulated chains carry the chain-specific fields of `SimTx` (spec 0.4.0). Existing scenarios `00`–`07` apply unchanged to Solana/Bitcoin deposits (payer side), payouts (destination side) and refunds (refund address side).

**Formats** (as already implemented in `src/simulated-chains/addresses.ts`, `s = seedBytes(seed)`; recorded here as the contract):

| | Solana | Bitcoin |
|---|---|---|
| deposit address | `base58(s)` (32 bytes) | P2WPKH `bc1q…` with program `s[0..20]` |
| tx hash | `base58(seedBytes("tx:"+seed) ‖ seedBytes("tx2:"+seed))` (64 bytes, a signature) | `hex(seedBytes("tx:"+seed))` (txid) |
| `SimTx.block_ref` | slot `380000000 + seq` | height `900000 + seq` |
| `transfers[].token_account` | ATA(`to`, mint) for SPL (kit `chains`); null for SOL | — |
| `transfers[].vout` | — | output index (`0`; scenario `09`: `0` and `2`) |
| `confirmations` | null | `1` when recorded, `+1` per 30 s (× `NP_SANDBOX_TIME_SCALE`), capped at 6; `finalized = confirmations ≥ 1` |

Rows recorded before v1.6 answer the new fields absent (rails treats absent as unknown). Simulated payer addresses stay as §4.1; the solver address on these chains is `solverAddress(chain)` in the formats above.

**Scenarios added:**
- `08` — slow finality: the destination (payout or refund) tx is recorded with `finalized: false` for 60 s (Bitcoin: `confirmations: 0`, then the normal growth), then final. Rails retries (§8.8) and must not emit evidence early; with a settlement ≥ USD 1,000 on Bitcoin it needs the second confirmation 30 s later.
- `09` — Solana: `destinationChainTxHashes` lists two signatures; the first is recorded `failed: true` with no transfers, the second carries the payout (rails must skip the failed one). Bitcoin: the payout tx pays the destination in two outputs (`vout` 0 and 2; amounts `floor(a/2)` and the rest) plus a solver change output `vout` 1 (rails must sum). Other chains: like `00`.

**Registry:** add Solana USDT (`nep141:sol-c800a4bd850783ccb82c2b2c7e84175443606352.omft.near`, 6 decimals, `contractAddress` `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB`, price 1) next to Solana USDC, SOL and BTC; every `sol-` entry must satisfy the SPL id rule (rails §21.1, a unit test against `chains-solana-bitcoin.json`).

**Scenario suffixes in use:** `00`–`09` (payments/refunds), `91`–`94` (vault stand-in, OQ-T-03).

