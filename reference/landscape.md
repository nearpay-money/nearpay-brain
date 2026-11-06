# External Landscape — Verified Facts

> Facts the design depends on, with source and date read. **Re-verify before relying on anything older than 90 days.** Items marked ⚠ are unconfirmed and tracked as spikes in `roadmap/open-questions.md`.
> Last verified: 2026-11-06 (step 1.2 observations: 1Click quote signature and appFees split, DIP-4 transfer shape, Explorer API contract, NEAR tx status); 2026-11-05 (1Click OpenAPI v0.1.10, hackathon rules added).

## 1. NEAR Intents (settlement fabric)

| Fact | Design consequence | Source |
|---|---|---|
| Verifier contract **`intents.near`** is the on-chain settlement ledger. Balances are ledger entries keyed by NEAR `AccountId` (named, implicit Ed25519 hex, or ECDSA `0x…` implicit). "Intents Technology does not custody these balances — control stays with the keys that can sign for each account." | A merchant can settle into its own Intents account (EVM merchants use their `0x…` address — no NEAR wallet needed). | docs.near-intents.org › verifier-contract |
| Asset IDs are prefixed multi-token IDs: `nep141:wrap.near`, `nep141:eth-0x…omft.near` (bridged), `nep245:…`. Withdraw intents use unprefixed IDs. | Canonical `asset_id` in NearPay = NEAR Intents asset ID. | same |
| Intent types: `add_public_key`, `remove_public_key`, `transfer`, `token_diff`, `ft_withdraw`, `nft_withdraw`, `mt_withdraw`, `native_withdraw`, `storage_deposit`. Signed payload: `signer_id`, `verifying_contract`, `deadline`, `nonce` (32 B: 4 B salt + 28 B unique), `intents`. Standards: NEP-413, ERC-191, others. | Agent payments = payer-signed `transfer` intents; replay prevented by Verifier nonces. | intent-types-and-execution |
| ⚠ `execute_intents` executes a list in order; cross-contract atomicity is not guaranteed; batch failure semantics for one bad intent are not documented. | Facilitator simulates each intent before batching and falls back to single submission (SP-07). | same |
| **No testnet deployment** of the Verifier. | Test mode needs a sandbox simulator (ADR-0007). | verifier introduction |
| Mainnet behaviour observed with read-only `simulate_intents` view calls (2026-11-05): NEP-413, ERC-191 and raw Ed25519 payloads built by `@nearpay-internal/intents` are accepted; DIP-4 event version is `0.4.3`; `simulate_intents` does **not** check the nonce salt (only execution does); rejection messages include `invalid signature`, `insufficient balance or overflow`, `public key '…' doesn't exist for account '…'`, `nonce was already used`, `deadline has expired`. | Signers are tested against the live contract; the sandbox reproduces these messages; salt validity must be checked by the caller before submitting (`current_salt`). | mainnet `intents.near` view calls |
| DIP-4 `transfer` event data = `{intent_hash, account_id, receiver_id, tokens: {<asset>: <amount>}, memo?}`; the NEP-245 `mt_transfer` emitted for a transfer intent carries **no memo** (SP-14 resolved). | Binding attribution reads the DIP-4 `transfer` memo, never `mt_transfer` (rails §8.6, foundations §14); the sandbox emits the same shapes. | mainnet blocks 223270688–223270709 via neardata, 2026-11-06 |
| Bridges: **Omni Bridge** (Near One; ETH, Base, Arbitrum, BNB, Solana, Bitcoin), **POA Bridge** (many chains incl. Tron, XRP, Sui, Aptos, Cardano, Starknet), **HOT Bridge** (BNB, Polygon, Optimism, Avalanche, TON, Stellar, …). Protocol picks the bridge by destination chain. | Bridged balances (`*.omft.near`) carry bridge risk; disclose to merchants settling into Intents accounts. | bridging/overview |
| Supported chains (≈35): EVM (Ethereum, Base, Arbitrum, Optimism, Polygon, BNB, Avalanche, Gnosis, Bera, Scroll, Monad, Plasma, XLayer, Aurora, ADI), Bitcoin, Dogecoin, Litecoin, BCH, Dash, Zcash (transparent), NEAR, Solana, Stellar, Sui, TON, Tron, XRP, Aptos, Cardano, Starknet, Aleo. Lists disagree across pages. | Treat `GET /v0/tokens` as the only live source; never hard-code chains. | chain-support |

## 2. 1Click API (route provider #1)

| Fact | Design consequence |
|---|---|
| Base `https://1click.chaindefuser.com`; `GET /v0/tokens`, `POST /v0/quote`, `POST /v0/deposit/submit`, `GET /v0/status` (OpenAPI v0.1.10). Auth: header **`X-API-Key`** (recommended; JWT bearer is legacy); `/v0/tokens` needs none. Errors: `400 {message}`, `401`, `404` (status: deposit address not found). No rate-limit headers documented. | `OneClickProvider` adapter in `nearpay-rails`; treat 429/5xx defensively. |
| Quote response top level: `correlationId`, `timestamp`, `signature`, `quoteRequest`, `quote`; quote adds `refundFee`, `withdrawFee`. Status `swapDetails` adds `depositedAmount*`, `refundReason` (e.g. `PARTIAL_DEPOSIT`), `withdrawFee`. Request adds `rebates`, `confidentiality`, `insured` (+0.02%). | Persist `correlationId`; map `refundReason`. |
| **Quote signature** = Ed25519 signature over Base58(SHA-256(stable-stringify({...quoteRequest, ...quoteResponse, timestamp}))); verify with `verifyQuoteSignature()` from `@defuse-protocol/one-click-sdk-typescript` ≥ 0.1.24; the key is pinned in the SDK (0.1.26: `ed25519:reYaWhvwu8Jzo3WUM3zhn6VrhuMEF4eADL17qtRVifc`). The signed request sets `appFees: undefined`, so **appFees are not covered by the signature** (SP-01 resolved 2026-11-05). | Rails and sentinel verify via the SDK and check the echoed appFees themselves (rails §8.1, sentinel R6). |
| 1Click echoes `appFees: [{treasury, 50}]` **split** as `[{treasury, 25}, {<1Click implicit account>, 25}]` (recorded dry quote, 2026-11-05). | Integrity checks are split-aware: total equals the requested fee, one treasury entry keeping its share, ≤ 2 entries. |
| Quote fields: `dry`, `swapType` (`EXACT_INPUT`, `EXACT_OUTPUT`, `FLEX_INPUT`, partner-only `ANY_INPUT`), `slippageTolerance` (bps), `originAsset`, `destinationAsset`, `amount`, `depositType`/`refundType` (`ORIGIN_CHAIN`/`INTENTS`), `refundTo`, `recipient`, `recipientType` (`DESTINATION_CHAIN`/`INTENTS`), `deadline` (required), `referral`, `appFees[{recipient, fee}]`, `depositMode` (`SIMPLE`/`MEMO`). | Checkout uses `EXACT_OUTPUT` so the merchant receives the exact amount. |
| Response: `quote.depositAddress`, `depositMemo`, `amountIn`, `minAmountIn`, `amountOut`, `minAmountOut`, `deadline`, `timeWhenInactive`, `timeEstimate`; top-level `signature`. | Store the full response; the signature is verified server-side (rails, sentinel); the checkout relies on sentinel's co-signature. |
| `GET /v0/tokens` also lists `1cs_v1:…` asset ids (27 on 2026-11-06) that are not NEAR Intents asset ids. | Rails drops them from the registry (the spec `AssetId` rejects them). |
| `EXACT_OUTPUT`: deposits below `minAmountIn` are refunded at the deadline; excess is refunded to `refundTo`. | Underpay/overpay outcomes are provider-enforced; NearPay mirrors them. |
| Statuses: `PENDING_DEPOSIT`, `KNOWN_DEPOSIT_TX`, `PROCESSING`, `SUCCESS`, `INCOMPLETE_DEPOSIT`, `REFUNDED`, `FAILED` (`KNOWN_DEPOSIT_TX` appears in examples but not every enum). | Unknown statuses map to `held_unknown` + alert; never assumed terminal. |
| **No webhooks.** Poll `/v0/status`; Explorer API `GET explorer.near-intents.org/api/v0/transactions` (JWT, **1 req / 5 s**, ≤ 1,000 rows, filter by `referral`, `statuses`, time). Contract read from its `openapi.yaml` (2026-11-06): Bearer JWT; bare array body; `numberOfTransactions` ≤ 1000; comma-separated `statuses` whose default omits `PENDING`/`INCOMPLETE`; `startTimestamp` ISO-8601, exclusive; no `updatedAt` field; ⚠ whether the time filter uses creation or update time is undocumented (SP-10). | Hybrid tracker: hot per-attempt polling + Explorer change feed + own NEAR indexer (ADR-0012). |
| Fees: unauthenticated +25 bps. With partner JWT, `appFees` total ≤ 500 bps, **split 50/50 with 1Click**, 1Click half min 20 bps (1 bp on stable/same-asset routes). Fee is taken from the input token. Quote improvement within ~30 min shared 50/50. | NearPay sets total `appFees` = its all-in price; net revenue ≈ half. Fee schedule is data, not code. |
| ToS: integrator must KYB, screen end users (sanctions/AML), keep end-user terms, pass the **unmasked originating wallet** as refund address, not cache API data > 24 h (except tx history), not present 1Click as a payment processor. Fees non-refundable; recovery of misdirected funds discretionary (may refuse < USD 300); liability cap ≈ max(USD 1,000, 12-month fees). | Compliance service is mandatory; refund address is a hard input; risk register R-01. |

## 3. NEAR chain

| Fact | Source |
|---|---|
| 600 ms blocks, ~1.2 s finality on mainnet. | NEAR announcement (2025) |
| Chain Signatures MPC signer `v1.signer` (mainnet), secp256k1 + ed25519 domains; one-way outbound signing. Cost/latency unpublished. | docs.near.org › chain-signatures |
| Wallet login: **NEAR Connect** (`@hot-labs/near-connect`) is the documented connector. | docs.near.org › web-login |
| Function-call access keys cannot attach deposits (so cannot call `ft_transfer`); they can call zero-deposit methods with a gas allowance. | NEAR protocol docs |
| `EXPERIMENTAL_tx_status` with `wait_until: NONE` answers at once (`UNKNOWN_TRANSACTION` for an unknown hash); all-shard nodes (FastNEAR) accept any `sender_account_id`, so a verifier needs only the tx hash. | observed 2026-11-06 (rails verifier) |

## 4. Agent payments

| Fact | Source |
|---|---|
| x402 v2 headers: `PAYMENT-REQUIRED` (402 response), `PAYMENT-SIGNATURE` (client retry), `PAYMENT-RESPONSE` (settlement result); Base64 JSON. Facilitator `POST /verify`, `POST /settle`; `settlement_pending` is non-terminal. Schemes: `exact`, `upto`, `batch-settlement`. Repo moved to `x402-foundation/x402`. | docs.x402.org; Cloudflare Agents docs (Oct 2026) |
| ⚠ NEAR is not an upstream x402 network; a `near-intents` transfer method is proposed in x402 PR #3370 (PingPay implements it) — SP-11. | PingPay docs; x402 README |
| Adoption is real but noisy: Aug 2026 ≈ 18.3 M payments / $2.59 B across 5 EVM chains, 88% of value one bridging contract; median Base seller earned $2.50/month. | Bitquery (Sep 2026) |
| Stripe/Tempo **MPP** (Machine Payments Protocol) is the main alternative; PingPay ships an MPP endpoint. | Zuplo, Formo (2026) |

## 5. Competitors

| Product | Notes (from public docs, Nov 2026) |
|---|---|
| **PingPay** (pingpay.io) | On NEAR Intents. Hosted checkout (`/checkout/sessions`, statuses `CREATED→PENDING→COMPLETED/EXPIRED/CANCELLED`), headless payments (`PENDING/SUCCESS/FAILED`), HMAC webhooks (`x-ping-signature`), x402 v2 facilitator (`near-intents`, upfront deposit), MPP. Undocumented: refunds, subscriptions mechanics, custody, fees. |
| Custodial crypto gateways (Coinbase Commerce, NOWPayments, etc.) | Custodial or chain-specific; not chain-abstracted any-to-any. |

## 6. Hosting — netcup

| Fact | Consequence |
|---|---|
| Locations: Nuremberg (NUE), Vienna (VIE), Amsterdam (AMS), Manassas (MNZ), Singapore (SIN). Location fixed at order. | Primary NUE, secondary VIE, witness/observability in AMS. |
| Root servers (RS G12.5, AMD EPYC 9645): **dedicated cores**, 99.9% availability guarantee. VPS: shared resources, no availability guarantee. ⚠ Root servers listed for VIE/NUE/MNZ/SIN, not AMS (SP-09). | Production data/compute on root servers only; VPS only for witness/tooling. |
| Cloud VLAN is **same-location only**; firewall cannot filter VLAN traffic. | WireGuard mesh between sites; host firewall (nftables) + Cilium policies. |
| DDoS filter up to L4 (Anexia, 2 Tbps). No L7. No floating IPs found. No S3-compatible object storage found. | Cloudflare in front for L7/WAF/LB; backups to an off-provider EU S3 bucket. |
| No nested virtualisation. | Containers on k3s, not VMs-in-VMs. |

## 7. NEAR Hackathon (rules read 2026-11-05 from https://nearhackathon.com/HACKATHON.md — re-read before submitting)

| Fact | Consequence |
|---|---|
| Window 2026-11-01 → **2026-11-14 23:59 UTC**; repo public by then; judging 11-16, results by 11-18. Pre-existing code allowed; window counts from the 2026-10-19 announcement. | Stage 1 must be demoable and documented by Nov 14 (ADR-0016). |
| One submission per project; `projectUrl` = one public repo, unique. | Thin `nearpay` entry repo. |
| README must serve machine and human readers, "do X → see Y", no marketing; list libraries/templates/AI-generated code; LICENSE required (any). | Apache-2.0 everywhere; "Built with" sections. |
| Local-only demo OK with reproducible README and a video in the X post; hosted demo a plus (keep up through 11-18, ideally 11-25). | Stage 1 demo local against mainnet. |
| No Intents testnet (Q-007): use mainnet, separate dev accounts, tiny amounts, `dry: true` quotes. 1Click without key: +0.2% fee (Q-006). | Sandbox for test mode; canaries cost-capped. |
| Tracks used: `realworld` (stablecoins, NEAR Intents, meta transactions) and `agents` (agent commerce, NEAR Intents). | Stage 1 = checkout + agent payments. |
| Never without explicit human confirmation: submit, post, make public, push, sign, spend, deploy to mainnet, commit secrets. Do not invent accounts. | Agents prepare; humans execute. |
