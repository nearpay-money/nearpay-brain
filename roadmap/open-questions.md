# Open Questions and Spikes

> Version 1.0 · Every ⚠ in the brain points here. A spike closes with a short write-up appended below its row and, if a decision changes, an ADR.

## Technical spikes

| ID | Question | Why it matters | Resolve by | Blocks |
|---|---|---|---|---|
| **SP-01** | Quote signatures are verifiable via `verifyQuoteSignature()` in `@defuse-protocol/one-click-sdk-typescript` ≥ 0.1.24 — confirm algorithm/key from SDK source, browser bundle size, and whether browsers can call `/v0/status` (CORS) | Client-side verification of deposit instructions (T-03 layer 4) | Read SDK source; test | Stage 1 (sentinel covers if unresolved) |
| **SP-02** | Does `EXACT_OUTPUT` always deliver exactly `amountOut` (no surplus/shortfall)? | Amount tags, evidence checks | Mainnet tests + vendor confirmation | P0 |
| **SP-03** | `INCOMPLETE_DEPOSIT`: are top-ups to the same address accepted? Behaviour for deposits after `timeWhenInactive`? | Underpay UX | Vendor + tests | P1 |
| **SP-04** | Partner rate limits for `/v0/quote` and `/v0/status`; any webhooks/streams for partners? Meaning of `KNOWN_DEPOSIT_TX`? | Capacity (D), polling design | Partner agreement | P0 |
| **SP-05** | Recovery policy for wrong-chain EVM sends and missing memos | Support playbook, R-14 | Vendor | P1 |
| **SP-06** | Reliability of `customRecipientMsg` to fund vault mandates from non-NEAR chains | Cross-chain auto-debit funding | Mainnet tests | P3 |
| **SP-07** | Gas per `transfer` intent and failure isolation inside one `execute_intents` batch; realistic shard headroom | Agent throughput (A), batching | Mainnet measurements | P2 |
| **SP-08** | Can the treasury Intents account be controlled by a multisig contract calling Verifier withdraw methods? | Treasury custody design | Contract test | P1 |
| **SP-09** | netcup: IPv6 per server, root server availability per site, Cloud VLAN MTU/price, AUP for payment businesses | Infra build | netcup support | P0 |
| **SP-10** | Explorer API: filter semantics for time windows (created vs updated) and `referral` reliability | Explorer feed design | Tests | P1 |
| **SP-11** | x402 PR #3370 status; upstream appetite for `intents-transfer` | Interop | x402 foundation | P2 |
| **SP-12** | Self-hosted NEAR RPC node sizing on netcup | RPC independence | Benchmark | P3 |
| **SP-13** | Amount-tag behaviour per asset (decimals, minimum transfer units, chains that round) | Watcher attribution | Tests per chain | P1 |

## Business

| ID | Question | Default until answered |
|---|---|---|
| **Q-BIZ-01** | Trademark clearance for "NearPay" (name collisions exist) | Keep identifiers domain-derived |
| **Q-BIZ-02** | Fee schedule by tier/route class (net revenue ≈ half of configured `appFees` on 1Click routes) | 100 bps volatile, 50 bps stable-cross, 20 bps stable-same (all-in) |
| **Q-BIZ-03** | npm scope `@nearpay` availability | Fallback `@nearpay-money` |
| **Q-BIZ-04** | Default `fee_bearer` (payer vs merchant) | `payer` |

## Legal and compliance

| ID | Question |
|---|---|
| **Q-LEG-01** | Regulatory status of a non-custodial, fee-taking gateway (and the vault) in the chosen launch jurisdiction(s): VASP/CASP, money transmission, travel rule |
| **Q-LEG-02** | Data residency with Cloudflare in the path (EU data localisation needs) |
| **Q-LEG-03** | 1Click ToS 4.1(h) "unmasked originating wallet" when payers pay from exchanges and supply a refund address |
| **Q-LEG-04** | Surcharging rules if `fee_bearer = payer` in specific jurisdictions |

## Vendor selection

| ID | Choice | Shortlist |
|---|---|---|
| Q-VEN-01 | KYB vendor | Sumsub, Persona |
| Q-VEN-02 | Address screening | Chainalysis (sanctions API + KYT), TRM Labs, Elliptic |
| Q-VEN-03 | Off-provider EU object storage | Hetzner Object Storage, Backblaze B2 (EU), Scaleway |
| Q-VEN-04 | Transactional email (primary/fallback) | Postmark, Amazon SES, Resend |
| Q-VEN-05 | FX and price feeds | ECB reference + commercial FX API; secondary crypto price source (e.g. Pyth, CoinGecko) |
| Q-VEN-06 | On-call paging + status page | external SaaS (must be outside Cloudflare and netcup) |
