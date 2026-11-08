# Open Questions and Spikes

> Version 1.7 (step 4.1: OQ-T-03 resolved — sandbox vault stand-in decided by the lead and specified in `nearpay-spec` 0.4.0, sandbox v1.6 §3.1/§4.2, billing v1.7 §14 d12) · Version 1.6 (step 4.1: proposed resolution for OQ-T-03, not decided) · Version 1.5 (owner decisions 2026-11-07, ADR-0020: Q-VEN-01 superseded, Q-VEN-02 deferred, Q-VEN-05 decided for v1; OQ-T-03 added) · Version 1.4 (step 1.4: CR-15-01, 02, 03, 05, 08, 09, 11, 12, 13 resolved in `@nearpay/spec` 0.1.5) · Version 1.3 (SP-01, SP-14, SP-15 resolved in step 1.2; planned spec 0.1.5 change requests from steps 1.2/1.3) · Version 1.2 OQ-T-01 and OQ-T-02 decided by the owner 2026-11-05 · Every ⚠ in the brain points here. A spike closes with a short write-up appended below its row and, if a decision changes, an ADR.

## Technical spikes

| ID | Question | Why it matters | Resolve by | Blocks |
|---|---|---|---|---|
| ~~**SP-01**~~ ✔ | Quote signatures are verifiable via `verifyQuoteSignature()` in `@defuse-protocol/one-click-sdk-typescript` ≥ 0.1.24 — confirm algorithm/key from SDK source, browser bundle size, and whether browsers can call `/v0/status` (CORS) | Client-side verification of deposit instructions (T-03 layer 4) | Read SDK source; test | Stage 1 (sentinel covers if unresolved) |
| **SP-02** | Does `EXACT_OUTPUT` always deliver exactly `amountOut` (no surplus/shortfall)? | Amount tags, evidence checks | Mainnet tests + vendor confirmation | P0 |
| **SP-03** | `INCOMPLETE_DEPOSIT`: are top-ups to the same address accepted? Behaviour for deposits after `timeWhenInactive`? | Underpay UX | Vendor + tests | P1 |
| **SP-04** | Partner rate limits for `/v0/quote` and `/v0/status`; any webhooks/streams for partners? Meaning of `KNOWN_DEPOSIT_TX`? | Capacity (D), polling design | Partner agreement | P0 |
| **SP-05** | Recovery policy for wrong-chain EVM sends and missing memos | Support playbook, R-14 | Vendor | P1 |
| **SP-06** | Reliability of `customRecipientMsg` to fund vault mandates from non-NEAR chains | Cross-chain auto-debit funding | Mainnet tests | P3 |
| **SP-07** | Gas per `transfer` intent and failure isolation inside one `execute_intents` batch; realistic shard headroom | Agent throughput (A), batching | Mainnet measurements | P2 |
| **SP-08** | Can the treasury Intents account be controlled by a multisig contract calling Verifier withdraw methods? | Treasury custody design | Contract test | P1 |
| **SP-09** | netcup: IPv6 per server, root server availability per site, Cloud VLAN MTU/price, AUP for payment businesses | Infra build | netcup support | P0 |
| **SP-10** | Explorer API: filter semantics for time windows (created vs updated) and `referral` reliability | Explorer feed design | Tests (contract read 2026-11-06: rows have no `updatedAt` and the time basis is undocumented; reference/landscape.md §2) | P1 |
| **SP-11** | x402 PR #3370 status; upstream appetite for `intents-transfer` | Interop | x402 foundation | P2 |
| **SP-12** | Self-hosted NEAR RPC node sizing on netcup | RPC independence | Benchmark | P3 |
| **SP-13** | Amount-tag behaviour per asset (decimals, minimum transfer units, chains that round) | Watcher attribution | Tests per chain | P1 |
| ~~**SP-14**~~ ✔ | Exact DIP-4 `transfer` event and NEP-245 `mt_transfer` log shapes emitted by `intents.near` (incl. `memo`) on mainnet | Indexer attribution by binding/tag | Record mainnet receipts as fixtures | Stage 1 |
| ~~**SP-15**~~ ✔ | OFAC SDN advanced XML source URL and digital-currency feature identifiers | Local sanctions list (compliance Stage 1) | Download + parse fixture | Stage 1 |

**SP-01 write-up (2026-11-05, step 1.2).** `verifyQuoteSignature` of `@defuse-protocol/one-click-sdk-typescript` verifies 1Click quotes with an Ed25519 key pinned in the SDK (0.1.26: `ed25519:reYaWhvwu8Jzo3WUM3zhn6VrhuMEF4eADL17qtRVifc`). The signed request sets `appFees: undefined`, so appFees are **not** covered and are checked separately; 1Click echoes them split 50/50 with its own account (rails §8.1, sentinel R6). Rails and sentinel verify server-side; browser bundle size and `/v0/status` CORS were not examined, because the checkout relies on sentinel's co-signature (T-03 layer 4) as planned.

**SP-14 write-up (2026-11-06, step 1.2).** Recorded from mainnet blocks 223270688–223270709 (neardata): the DIP-4 `transfer` event is `{intent_hash, account_id, receiver_id, tokens{asset: amount}, memo?}`; the NEP-245 `mt_transfer` of a transfer intent carries no memo. Binding attribution uses the DIP-4 memo (foundations §14 corrected, rails §8.6); the sandbox emits the same shapes. Fixtures: `nearpay-rails/test/fixtures/near/`.

**SP-15 write-up (step 1.2).** Source `https://sanctionslistservice.ofac.treas.gov/api/PublicationPreview/exports/SDN_ADVANCED.XML` (redirects to a signed S3 URL; ~128 MB). Digital-currency addresses are the features named "Digital Currency Address - <CODE>"; their type ids are read from the document itself rather than hard-coded (compliance §7).

## Contract gaps found while implementing

| ID | Question | Default until answered | Resolve by |
|---|---|---|---|
| **OQ-T-01** | ~~Query grammar of `GET /v1/payment_intents/search`~~ | **Decided:** no search endpoints; typed filters on list endpoints (ADR-0018) | Closed 2026-11-05 |
| **OQ-T-02** | ~~Request/response shapes of ops HTTP routes for compliance cases, blocklist edits and sentinel findings~~ | **Decided:** specified together with the Stage 2 console, their first network consumer; Stage 1 keeps these actions in the in-process ops CLI with no network path | Closed 2026-11-05 |
| ~~**OQ-T-03**~~ ✅ resolved 2026-11-08 | How are `auto_debit` mandates exercised in test mode, now that Stage 4 exits on a test-mode E2E instead of a mainnet demo (ADR-0020)? Neither the sandbox nor billing specifies a test-mode vault | **Decided (lead): sandbox vault stand-in** — see the write-up below; specified in `nearpay-spec` 0.4.0 (step 4.1, `stage-4/4.1-autodebit`) and `lld/sandbox.md` v1.6 §3.1/§4.2 | Step 4.1 |


**OQ-T-01 write-up.** The owner chose typed list filters over a Stripe-style query language: one validated mechanism, no parser, each filter backed by a deliberate index. `GET /v1/payment_intents` adds `amount`, `currency`, `asset_id` and `metadata[key]` filters in Stage 2; `/search` is removed from the conventions and the payments LLD.

**OQ-T-02 write-up.** The owner chose to shape these endpoints with the Stage 2 console screens rather than guess them now. Stage 1 operators use `nearpay-testbed/ops` (in-container CLI). The KYB decision and kill-switch endpoints stay in the spec as already specified.

**OQ-T-03 — resolved (lead decision, step 4.1, 2026-11-08).** The proposal below was adopted as written, with these contract details (spec 0.4.0): the JSON-RPC endpoint is `POST /sim/near-rpc` (also serving `block`, `query view_access_key` and `tx`, which a signing client needs); signed transactions are decoded and their ed25519 signature checked (keys register on first use); payer actions without a wallet use `POST /sim/vault/call`; `POST /sim/vault/advance {mandate_id, secs?}` moves **one mandate's** clock (no cross-test interference; billing's `advanceSubscriptionClock` aligns it); scenario `93` fails only the first refund (so the later `withdraw` path is testable); `94` delivers `floor(share / 2)`; the indexer feed items gain `receipt_id`, `log_index`, `executor_id`, `block_hash`, `block_timestamp`, `receipt_status`; the fixtures are `nearpay-spec/test-vectors/vault.json`, byte-checked against strings produced by the contract's own code. The neard-sandbox alternative stays available for later, higher-fidelity runs. Original proposal:

**OQ-T-03 — proposed resolution (step 4.1, 2026-11-08).** The sandbox serves a **vault stand-in** that billing's keeper and rails' indexer reach through their normal code paths, with byte-identical shapes:
- **NEAR JSON-RPC subset** for one test vault account (e.g. `vault.sandbox.near`, the test-mode `NP_VAULT_CONTRACT`): `broadcast_tx_commit` / `send_tx` for `collect`, `cancel`, `withdraw`, `storage_deposit`; `query` `call_function` for `get_mandate`, `get_mandates_by_payer`, `next_collect_at`, `get_config`. The keeper points `NP_NEAR_RPC_URLS` at the sandbox in test mode (INV-8: live keys and RPC never see it).
- **Same rules** as `lld/contracts-vault.md` §3.1–§3.8 (msg schema, check orders, caps, periods, refunds) and the same view JSON (§3.6).
- **Failures** have the NEAR RPC shape: `status.Failure.ActionError.kind.FunctionCallError.ExecutionError = "Smart contract panicked: ERR_…"`. The keeper matches the `ERR_[A-Z_]+` code.
- **Successes** carry the contract's exact `EVENT_JSON:{"standard":"nearpay_vault","version":"1.0.0",…}` lines (§4.1) in `receipts_outcome[].outcome.logs` with `executor_id` = the vault, and `status.SuccessValue` = base64 of `true`/`false`. The same receipt outcomes are appended to `GET /sim/intents/events`, so rails' sandbox indexer adapter forwards `evt.rails.vault.event` unchanged.
- **Funding:** a test helper `POST /sim/vault/ft_transfer_call {payer, token, amount, msg}` stands in for the payer's wallet. It applies §3.1, and a rejection appears as a plain `ERR_…[: detail]` log plus a refund, as on chain.
- **Outcome scenarios** are chosen like the 1Click scenario engine, by the last two digits of `amount_per_period`: `00` success, `91` merchant leg fails (`collect_failed`), `92` fee leg fails (`collected` with `fee: "0"`), `93` the cancel/withdraw refund fails (`canceled` with `refunded: "0"`), `94` partial merchant delivery.
- **Time:** `POST /sim/vault/advance {secs}` moves the stand-in's clock so period 2 becomes due without lowering the 86400 s minimum.
- **Parity:** the contract's event lines, error strings and view JSON are exported as fixtures to `nearpay-spec`; the vault's unit tests and the stand-in's contract tests both assert against them.
- **Alternative:** run the real wasm on a neard sandbox in the testbed Compose (highest fidelity). It needs the sandbox binary in Compose and an indexer block source for the sandbox RPC (neardata/Lake do not exist there), so it is heavier for the v1 date.

## Planned `@nearpay/spec` 0.1.5 (additive) — change requests from steps 1.2 and 1.3

Collected from the builders of steps 1.2/1.3 (2026-11-06). Each needs a spec PR (additive within 0.1.x) and a consumer bump, or is rejected. Items marked *(kit)*, *(sandbox)* or *(deploy)* are not spec changes and are listed so they are decided in the same pass.

**0.1.5 write-up (step 1.4).** Released and implemented by producers and readers (`nearpay-spec/CHANGELOG.md`): CR-15-01 (identity §6, dashboard §1), CR-15-02 (webhooks §5, `whd_` prefix), CR-15-03 (identity §5–§6), CR-15-05 (payments §6.2, checkout §6), CR-15-08 (webhooks §5), CR-15-09 (payments §6.1, agentpay §5/§7, webhooks §4), CR-15-11 and CR-15-12 (agentpay §3/§6/§7, sdks-and-tools §2), CR-15-13 (foundations §3). Still open: CR-15-04, 06, 07, 10 and 15; CR-15-14, 16 and 17 are not spec changes.

**From the dashboard and `@nearpay/node`**

| ID | Request | Why |
|---|---|---|
| ~~CR-15-01~~ ✔ 0.1.5 | `GET /v1/auth/session` → `DashboardSession` | A reload or new tab cannot recover the CSRF token; the dashboard keeps the session in `sessionStorage` (dashboard §1). Identity can re-issue the CSRF token from `csrf_enc` |
| ~~CR-15-02~~ ✔ 0.1.5 | `GET /v1/webhook_endpoints/{id}/deliveries` (delivery log) | Endpoint page shows only the test-ping result today; coordinate with webhooks (LLD lists it as Stage 2) |
| ~~CR-15-03~~ ✔ 0.1.5 | `POST /v1/settlement_destinations/{id}/challenge` | Re-issue an attestation challenge for a destination left in `pending_verification` (today: disable and re-add) |
| CR-15-04 | Optional: `fingerprint` on the payment intent's destination snapshot; typed `Event.data.object` union | The dashboard computes the fingerprint client-side; `@nearpay/node` derives the event union itself |

**From the checkout and `nearpay-js`**

| ID | Request | Why |
|---|---|---|
| ~~CR-15-05~~ ✔ 0.1.5 | `PublicCheckoutSession.payment_intent: id("pi")` | Embedded `np:complete` needs the intent id; today it is known only from an SSE `intent` event (`null` when the page only polled) |
| CR-15-06 | Optional `PaymentOption.token: {standard: "erc20" \| "native" \| "nep141", address \| null} \| null` | The wallet path derives the EVM token contract from omft asset-id naming, so non-omft EVM assets are manual-path only |
| CR-15-07 | (sentinel, later) attest the origin asset's `decimals` / `symbol` with the deposit | The checkout displays `amount_in` with decimals from the unattested payment options |

**From webhooks**

| ID | Request | Why |
|---|---|---|
| ~~CR-15-08~~ ✔ 0.1.5 | `enabled_events` items as `z.string()` (max 100) on create/update endpoint, or drop `event_type_unknown` from the routes | Today an unknown type is `400 invalid_request` before the handler runs; `event_type_unknown` is unreachable (webhooks §5) |
| ~~CR-15-09~~ ✔ 0.1.5 | `sequence` on `CheckoutSession` and `AgentPayment` (public and bus objects) | api-conventions §9 requires it; webhooks fills it from `aggregate.version` meanwhile |
| CR-15-10 | Optional: `ping` in `WebhookTypeSchema` | Lets merchants subscribe to `ping` (harmless; it is never fanned out) |

**From `nearpay-agentkit` and `nearpay-testbed`**

| ID | Request | Why |
|---|---|---|
| ~~CR-15-11~~ ✔ 0.1.5 | Optional absolute expiry (`extra.deadline` / `expiresAt`) on `X402IntentsTransferExtra` | Requirements carry only `maxTimeoutSeconds`; agents sign `now + maxTimeoutSeconds/2` to stay inside the window |
| ~~CR-15-12~~ ✔ 0.1.5 | `SettleResponse.replayed?: boolean` (also requested by agentpay) | Settle is idempotent on the payload hash, so one payment can be delivered several times across merchant replicas |
| ~~CR-15-13~~ ✔ 0.1.5 | Tolerant parsing of open enums for SDKs | `openEnum` is a plain `z.enum`; api-conventions §5 says clients tolerate new values (foundations §3) |
| CR-15-14 | Funding a sandbox Intents balance: sandbox ops CLI `intents-fund` *(sandbox; no spec change, preferred by the requester)* vs a test-helper route | The demo agent needs a test balance and there is no public route. State: `seed.sh data` currently uses the sandbox ops CLI (`nearpay-sandbox` 313739e, step 1.4) |

**From agentpay**

| ID | Request | Why |
|---|---|---|
| ~~CR-15-12~~ ✔ 0.1.5 | `SettleResponse.replayed?: boolean` | Same as above |
| CR-15-15 | Internal payments route to verify a client secret / read the session for agentpay (e.g. `GET /internal/checkout_sessions/{id}/payer`) | Today agentpay calls payments' public session route with the payer's secret (agentpay §5) |
| CR-15-16 | *(kit)* intents: send an already-signed tx and return the outcome block hash; http client `call` with extra headers | `executeIntents` loses the tx hash on timeout; agentpay works around both (foundations §4) |
| CR-15-17 | *(sandbox)* tx-status route | A test-mode execute that fails in transport is requeued and then hits `nonce_used` (agentpay §7) |

**Deploy follow-ups reported with these requests** *(deploy)*: nginx per-response CSP nonce for the checkout and dashboard `index.html` (applied in `nearpay-deploy` 1a9a9c4, platform §1); agentpay and webhooks-dispatcher `depends_on` valkey, batcher `NP_RELAYER_KEY_FILES=/run/secrets/relayer_key_*` with owner-provided `relayer_key_1…4`, `NP_WEBHOOK_SIGNING_KID` from `keys.env` (applied in `nearpay-deploy` 5329223, step 1.4; the relayer keys come from the `seed.sh live` overlay; platform §1.2). Agentpay's kill switch also blocks live checkout relays — a conservative reading to revisit (agentpay §8).

## Business

| ID | Question | Default until answered |
|---|---|---|
| **Q-BIZ-01** | Trademark clearance for "NearPay" (name collisions exist) | Keep identifiers domain-derived |
| **Q-BIZ-02** | Fee schedule by tier/route class (net revenue ≈ half of configured `appFees` on 1Click routes) | 100 bps `volatile`, 50 bps `stable_pair`, 20 bps `same_asset`, 30 bps `intents_transfer` with USD 0.001 minimum (all-in) |
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
| ~~Q-VEN-01~~ | ~~KYB vendor~~ — **superseded for v1:** self-declared KYB + ops decision (ADR-0020) | ~~Sumsub, Persona~~ |
| Q-VEN-02 | Address screening — **deferred:** v1 uses the local OFAC list + blocklist (ADR-0020) | Chainalysis (sanctions API + KYT), TRM Labs, Elliptic |
| Q-VEN-03 | Off-provider EU object storage | Hetzner Object Storage, Backblaze B2 (EU), Scaleway |
| Q-VEN-04 | Transactional email (primary/fallback) | Postmark, Amazon SES, Resend |
| ~~Q-VEN-05~~ | FX and price feeds — **decided for v1:** ECB reference rates + a keyless public crypto price feed (ADR-0020) | ~~ECB reference + commercial FX API; secondary crypto price source (e.g. Pyth, CoinGecko)~~ |
| Q-VEN-06 | On-call paging + status page | external SaaS (must be outside Cloudflare and netcup) |

**Q-VEN-01 write-up (2026-11-07, superseded by ADR-0020).** No KYB vendor in v1. Merchants self-declare minimum business details (legal name, country of incorporation, registration number, business type, website, business address, representative name/role, expected monthly volume band); ops approves or rejects them in the console through the existing manual KYB decision endpoint, which gates live mode (FR-ACC-05 unchanged). No hosted flow, vendor applicant or vendor webhook. Reopen when the launch jurisdiction (Q-LEG-01), 1Click or a partner requires verified KYB.

**Q-VEN-02 write-up (2026-11-07, deferred by ADR-0020).** Every screening context (FR-CMP-02) runs on the Stage 1 local OFAC SDN digital-currency list + blocklist (SP-15) behind a vendor-neutral screening interface with one provider, `local`; the fail policy is unchanged. A vendor becomes a second provider behind the same interface without a contract change. Reopen when volume, non-NEAR exposure or a partner calls for exposure (KYT) screening.

**Q-VEN-05 write-up (2026-11-07, decided for v1 by ADR-0020).** Fiat FX: ECB euro reference rates (daily XML, 29 currencies, published ≈ 16:00 CET on TARGET working days, "for information purposes only"; `reference/landscape.md` §8). Secondary token price for the dual-source deviation check: a keyless public crypto price feed. Both sit behind provider interfaces in rails; no keys, no paid plans; stale or missing data → `pricing_unavailable`. Rails §8/§10 pick the concrete feed and set staleness bounds that tolerate weekends and TARGET holidays. A real-time FX source is a later provider.

## Mainnet findings carried into Stage 2 (2026-11-07)

**Stage 2 step 2.1 note.** The 1Click nullish finding below is being addressed in `@nearpay/spec` in step 2.1 (nullish `OneClick*` fields, status-only quote response with optional `correlationId`); once released, rails and sentinel can drop their pre-validation normalisation and the sandbox serves the live shape.

- **Spec transcription of live 1Click.** Live 1Click sends `null` for unset optional fields and omits the nested `quoteResponse.correlationId` on `GET /v0/status` (reference/landscape.md §2). rails and sentinel normalise bodies before validation (rails #2, sentinel #2); the sandbox cannot mirror these two quirks because the spec schemas reject them. Step 2.1: make those `OneClick*` fields nullish and add a status-only quote response with an optional `correlationId`, then let the sandbox serve the live shape.

