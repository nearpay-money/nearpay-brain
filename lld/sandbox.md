# LLD — `nearpay-sandbox`

> Version 1.3 (step 2.1: refund legs — `scenario_group`/`scenario_kind` §3–§4, `POST /sim/intents/transfers` §3, Stage 2 §8) · Version 1.2 (step 1.2 implementation findings folded: quote signature, appFees split and scenario key §3, Verifier events §3, scenario 06 §4, configuration §6, ops CLI §9) · Version 1.1 audited (`audits/sandbox.md`) · ADR-0007 · API conventions §8 · Foundations `00-foundations.md`

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
| `GET /sim/chains/{caip2}/tx/{hash}` | chain RPC | `{finalized, transfers:[{token, to, amount}]}` for sandbox verifiers |
| `POST /sim/deposits` | — | `{deposit_address, memo?, amount_in?}` simulate the payer's deposit (used by checkout test-mode "Simulate payment" via payments) |
| `POST /sim/advance` | — | `{deposit_address, to: deposit\|processing\|success\|refund\|fail}` step control |
| `POST /sim/intents/simulate` · `/execute` | `simulate_intents` / `execute_intents` | real NEP-413 / ERC-191 / raw Ed25519 signature verification (`@nearpay-internal/intents`), nonces, simulated balances per account (seeded via `/sim/intents/fund`), mainnet rejection messages, atomic batches; named accounts register the first key that signs for them. Emits DIP-4 / NEP-245 events shaped like mainnet: the binding memo of a transfer intent is on the DIP-4 `transfer` event only, the NEP-245 `mt_transfer` carries none (SP-14) |
| `GET /sim/intents/events?cursor=` | indexer feed | event stream for rails' sandbox indexer adapter |
| `POST /sim/intents/transfers` (Stage 2) | — | `{from, to, token, amount, memo?}`: an executed transfer without a signature (credits `from` with any shortfall first, then emits the same DIP-4 `transfer` + NEP-245 `mt_transfer` events as a signed execute). Used by payments' refund test helper for Intents legs; test mode only by construction |

**Quote fields (Stage 2):** `SandboxQuoteRequest.scenario_group` (rails sets `pi_…` for payment attempts — from `CreateExecution.reference` — and `re_…` for refund legs) and `scenario_kind` (`payment` | `refund`). For refund legs rails sets `scenario_amount` to the refund amount (source base units), so a merchant picks a refund scenario with the amount's last two digits.

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
| `06` | like `00`; a deposit also pays the older open quotes of the same payment (duplicate settlement → overpayment). Stage 2: quotes are grouped by `scenario_group` when present; on refund legs a deposit to one leg also settles the other open legs of the same refund (duplicate refund → payments records a `duplicate` leg). Without `scenario_group` the Stage 1 grouping applies |
| `07` | success arrives 5 min after the session's `expires_at` |
| refund address contains `sanctioned` | compliance blocks (handled by compliance sandbox rule) |

## 5. Data

`sim_quotes(deposit_key pk, request jsonb, response jsonb, scenario text, state text, timeline jsonb, created_at)`, `sim_txs(hash pk, chain, transfers jsonb, finalized_at)`, `sim_intents_balances(account_id, token, amount numeric(78,0), primary key(account_id, token))`, `sim_intents_events(seq bigserial pk, payload jsonb)`.

## 6. Configuration

`NP_SANDBOX_SIGNING_KEY_FILE` (public key distributed to rails/sentinel as `NP_SANDBOX_SIGNING_PUBKEY`), `NP_SANDBOX_TIME_SCALE` (1.0; testbed may set 0.1), `NP_SANDBOX_TICK_MS` (200; how often due timeline steps are applied), `NP_JWKS_URL` (service tokens).

## 7. Tests

Schema parity: sandbox request/response validated against the same zod schemas as the 1Click adapter (contract test); each scenario timeline; signature verification by rails/sentinel sandbox verifiers.

## 8. Stage mapping

Stage 1: all of the above except scenario `06` duplicate behaviour for refunds (Stage 2). Stage 2: `scenario_group` / `scenario_kind` on quotes, `06` for refund legs, `POST /sim/intents/transfers`; refund legs quote `EXACT_INPUT` (amountOut = `floor(amountIn × price ratio × (1 − 20 bps))`) and `EXACT_OUTPUT` like payments, with no app fee; the payer side of a refund settles on `/sim/chains` (route legs) or the simulated Verifier (Intents legs) so rails verifies payer-side evidence.

## 9. Ops CLI

`docker compose exec sandbox-api /nodejs/bin/node dist/ops.js <command>`: `intents-fund <account> <token> <amount>` credits a simulated Intents balance (the same `mt_mint` as `POST /sim/intents/fund`; `seed.sh data` funds the demo agent with it, platform §3) · `intents-balance <account> <token>`. Test mode only by construction (the sandbox holds no live data); arguments are validated before the ledger is touched. Missing: a tx-status route, so agentpay cannot recover a sandbox execute that failed in transport (agentpay §7).
