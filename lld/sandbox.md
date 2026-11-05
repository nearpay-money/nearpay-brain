# LLD — `nearpay-sandbox`

> Version 1.1 (audited, `audits/sandbox.md`) · ADR-0007 · API conventions §8 · Foundations `00-foundations.md`

## 1. Scope

Deterministic simulator for test mode, local development and E2E: a **1Click-compatible API**, simulated chains (evidence lookups), and a simulated Verifier (`intents.near`) for signed transfers. Rejects every `livemode=true` call.

## 2. Runtime

`sandbox-api` (1 replica, own DB `sandbox`). Exposed only on the `internal` network; every call requires a service token with `sandbox.use` (payments, rails, agentpay, sentinel); test helpers reach it through payments.

## 3. Endpoints

| Endpoint | Mirrors | Behaviour |
|---|---|---|
| `GET /v0/tokens` | 1Click | fixed registry using real asset IDs for Base/Arbitrum/Ethereum/NEAR/Solana USDC, USDT, ETH, wNEAR, BTC with fixed prices |
| `POST /v0/quote` | 1Click | validates the same schema; `amountIn = ceil(amountOut × price ratio × (1 + appFee + 20 bps))`; deterministic deposit address per chain format (`0x` + keccak(seed)[12..32] for EVM, `sbx-<hash>.near` for NEAR, base58 for Solana, bech32 for BTC); `signature` = Ed25519 (sandbox key) over `base58(sha256(stable-stringify({...quoteRequest, ...quoteResponseWithoutSignature, timestamp})))` — the documented 1Click construction (SP-01 confirms 1Click's own algorithm); scenario chosen from amount suffix / refund address |
| `GET /v0/status` | 1Click | scenario timeline (below) |
| `POST /v0/deposit/submit` | 1Click | marks deposit detected |
| `GET /sim/chains/{caip2}/tx/{hash}` | chain RPC | `{finalized, transfers:[{token, to, amount}]}` for sandbox verifiers |
| `POST /sim/deposits` | — | `{deposit_address, memo?, amount_in?}` simulate the payer's deposit (used by checkout test-mode "Simulate payment" via payments) |
| `POST /sim/advance` | — | `{deposit_address, to: deposit\|processing\|success\|refund\|fail}` step control |
| `POST /sim/intents/simulate` · `/execute` | `simulate_intents` / `execute_intents` | real signature verification (`@nearpay-internal/intents`), simulated balances per account (seeded via `/sim/intents/fund`), emits DIP-4 / NEP-245 style events |
| `GET /sim/intents/events?cursor=` | indexer feed | event stream for rails' sandbox indexer adapter |

## 4. Scenario engine

Scenario key = last two digits of the fiat amount in minor units, or of the asset amount in base units for asset-priced intents.

| Suffix / magic | Timeline after deposit |
|---|---|
| `00` or unlisted | `KNOWN_DEPOSIT_TX` (1 s) → `PROCESSING` (2 s) → `SUCCESS` (2 s), destination tx with exact amount |
| `01` | same, `SUCCESS` after 120 s |
| `02` | `INCOMPLETE_DEPOSIT` → `REFUNDED` at deadline (`refundReason: PARTIAL_DEPOSIT`) |
| `03` | `SUCCESS` with `refundedAmount` = excess |
| `04` | `FAILED` after `PROCESSING` |
| `05` | `SUCCESS` but `/sim/chains` returns amount − 1 (short) |
| `06` | every second attempt of the same session also succeeds on deposit |
| `07` | success arrives 5 min after the session's `expires_at` |
| refund address contains `sanctioned` | compliance blocks (handled by compliance sandbox rule) |

## 5. Data

`sim_quotes(deposit_key pk, request jsonb, response jsonb, scenario text, state text, timeline jsonb, created_at)`, `sim_txs(hash pk, chain, transfers jsonb, finalized_at)`, `sim_intents_balances(account_id, token, amount numeric(78,0), primary key(account_id, token))`, `sim_intents_events(seq bigserial pk, payload jsonb)`.

## 6. Configuration

`NP_SANDBOX_SIGNING_KEY_FILE` (public key distributed to rails/sentinel as `NP_SANDBOX_SIGNING_PUBKEY`), `NP_SANDBOX_TIME_SCALE` (1.0; testbed may set 0.1).

## 7. Tests

Schema parity: sandbox request/response validated against the same zod schemas as the 1Click adapter (contract test); each scenario timeline; signature verification by rails/sentinel sandbox verifiers.

## 8. Stage mapping

Stage 1: all of the above except scenario `06` duplicate behaviour for refunds (Stage 2).
