# ADR-0022: Payments confirm on one block of their own chain, re-checked at finality

- **Status:** Accepted 2026-11-10; **amended by owner 2026-11-10 17:10 IST** (one block on every chain — see "Amendment")
- **Date:** 2026-11-10
- **Deciders:** Owner (decision); Principal Engineer (lead agent, implementation)
- **Invariants touched:** INV-2 (evidence before success), INV-3 (truth wins)
- **Amends:** ADR-0012 decision "EVM chains … `finalized` block tag" — originally for Base and Arbitrum; since the amendment for every EVM chain, and rails §21 for Solana (`finalized` → `confirmed`) and Bitcoin (USD confirmation tiers → 1 confirmation)

## Amendment (owner, 2026-11-10 17:10 IST)

> "Payment confirmation speed is of the essence. Every chain should need only ONE block of its own chain's confirmation (Base, any other chain)."

### Rule per chain family (defaults; `config/chains.yaml` and the environment still override)

| Family | Evidence counts when | Evidence `finality` | Override |
|---|---|---|---|
| EVM — every chain: Ethereum `eip155:1`, Base, Arbitrum (Tier A); Optimism, Polygon, BSC, Avalanche, Gnosis (Tier B, displayed) | receipt status 1 in a block on the chain's own `latest` head (`latest − block + 1 ≥ 1`), transfer matched as before; the receipt is **re-read once right before the evidence applies** and must sit in the same block number and hash with status 1 (otherwise nothing counts and the next attempt starts over). Profile `evm_confirmation: {kind: confirmations, blocks: 1}` | `confirmed` | `NP_EVM_CONFIRMATIONS_<CHAINID>` = `finalized` \| N |
| Solana | `getTransaction` / `getSignaturesForAddress` at **`confirmed`** commitment (the slot was voted on by a supermajority — Solana's one-block confirmation). `processed` is never used: it is not a confirmation | `confirmed` | `NP_SOLANA_COMMITMENT=finalized` |
| Bitcoin (BTC; LTC/DOGE profiles, Tier B, displayed) | **1 confirmation for every amount** (the spec 0.4.0 USD tiers — 1 below USD 1,000, else 2 — are no longer the default) | `confirmed` | profile `confirmation_tiers` (`NP_CHAINS_FILE`) |
| NEAR | **unchanged: `final`** | `final` | — |

**Why NEAR stays `final`.** NEAR's Doomslug finality makes a block final once the next block carries the endorsements of validators holding more than ⅔ of the stake — "the block plus one", about 1–2 s after the transfer. That already is NEAR's one-block confirmation. `optimistic` (the block before endorsement) would save about one second and give up the only finality guarantee NEAR offers, so it is not used. Nothing else in rails waits longer than `final` for NEAR: the indexer follows neardata's `last_block/final` every 1 s and the verifier asks `EXPERIMENTAL_tx_status` for `FINAL` (all receipts executed in final blocks). NEAR evidence needs no post-success re-check.

### Post-success re-check, every Tier-A family (rails §8.12)

| Family | Re-checked against | Cadence / window | `failed` when |
|---|---|---|---|
| EVM | the chain's `finalized` tag | 60 s / 2 h (`NP_FINALITY_RECHECK_INTERVAL_S`, `…_WINDOW_S`) | at finality the receipt is gone, in another block (hash changed) or status ≠ 1 |
| Solana | `finalized` commitment (`getTransaction` + `getSlot` at `finalized`, ≈ 13 s after `confirmed`) | 15 s / 30 min (`…_SOLANA_INTERVAL_S`, `…_SOLANA_WINDOW_S`) | served at `finalized` in another slot or with `meta.err`; or the finalized slot passed the evidence slot and the transaction is gone (not even `confirmed` in that slot — dropped with its fork) |
| Bitcoin | 6 confirmations of the evidence block (`…_BITCOIN_CONFIRMATIONS`) | 60 s / 6 h (`…_BITCOIN_INTERVAL_S`, `…_BITCOIN_WINDOW_S`) | at 6 confirmations the transaction is unknown, back in the mempool or in another block hash (orphaned, even if re-mined) |

Not final within the window → `timeout`. `failed` and `timeout` alert the same way for every family: `ALERT finalized re-check …` error log, `nearpay_rails_finality_recheck_total{chain,outcome}` (deploy rule `FinalityRecheckFailed`, no chain filter → Telegram), `finality_recheck_status` / `finality_recheck_detail` on the execution row and a `verifier` observation (applied = false) in the evidence trail. A succeeded payment is **never moved back** (unchanged). Tier B (LTC, DOGE, Tier B EVM chains) and test-mode evidence are not re-checked: there is no on-chain evidence to re-check.

### Accepted risks (owner)

- **EVM L1 1-block reorg (Ethereum).** A 1-block Ethereum reorg happens a few times a day network-wide; the transfer is usually re-included in the replacing block, but a payment can read `succeeded` before its block is orphaned. Ethereum's `finalized` (≈ 13 min) now comes after "Paid" instead of before it. Surfaced by the re-check within minutes (same block hash required at finality, so even a re-included transfer alerts; ops then confirm it arrived).
- **EVM L2 1-block reorg (Base, Arbitrum).** A sequencer reorg of the newest block (failover, operator fault) before batch posting. Previously bounded by 10/40 blocks, now by one block plus the same-hash re-read. Rare; surfaced by the `finalized` re-check.
- **Solana `confirmed` → fork.** A supermajority-voted slot has never been rolled back on mainnet, but `confirmed` is not `finalized`: a minority fork dropping the slot is possible in principle. Surfaced by the `finalized` re-check ≈ 13 s later.
- **Bitcoin 1-confirmation orphan risk, including larger amounts.** A stale (orphaned) block occurs roughly once every few weeks; a 1-confirmation transfer in it usually returns to the mempool and is re-mined, but could be double-spent by the sender in between. The USD tiers that required 2 confirmations at USD 1,000 and above are gone by default, so this exposure now applies to any amount (no USD limit applies to Tier-A Bitcoin evidence). Surfaced by the 6-confirmation re-check (up to ~1 h later); ops may restore tiers via the profile file.
- In all cases the payment stays `succeeded` and the merchant may already have delivered: the alert is the only signal (gap below).

### Time to "Paid"

Measured in rails' integration test `test/integration/latency.test.ts` (Anvil, fake 1Click, real Postgres/NATS, **production loop cadences**), from the payout's block (receipt first readable) to `evt.rails.execution.settled` on NATS — the event payments turns into `succeeded` and pushes to the checkout stream (Valkey notify → SSE; no polling on that path):

| Path | Before (main, Base 10 confirmations, 2 s blocks) | After (one block) |
|---|---|---|
| Watcher (tagged USDC on Base) | median 18.5 s (18.2–18.7 s) | median 0.1–1.0 s, max 1.02 s |
| Provider SUCCESS → hot poller → verifier (swap settling ~30 s) | median 19.4 s (19.0–19.7 s) | median 0.1–0.5 s, max 0.74 s |

Cadence changes behind it: hot poller every 2 s for the first 150 polls of an in-flight deposit (`deposit_detected` / `incomplete_deposit` / `settling`), then 4, 8, 15 s (the 1Click status token bucket still caps the rate); verifier retry 2 s (Bitcoin 5 s) and 500 ms idle tick (was 10 s / 1 s); watchers Base/Arbitrum 1 s (was 2 s), Ethereum 3 s (was 12 s), Solana 2 s (was 4 s), Bitcoin 30 s (was 60 s); the Bitcoin watcher makes a `verifying` execution's verification due now when a new confirmed payment to its destination appears. The EVM watcher's first sight of demand starts 10 blocks below the counted head (with one block the counted head is `latest`; a payout mined in that block would otherwise be skipped — the old 10/40-block rules left this margin implicitly). Target met: < 5 s from the confirming block to checkout "Paid" (rails ≤ ~1 s; payments' consumer and the checkout stream are push-driven).

Checkout copy while the attempt is `verifying`: "Payment received — confirming on {chain}… usually a few seconds" for every EVM chain (Ethereum included), NEAR and Solana; Bitcoin (one ~10-minute block) keeps the copy without a time promise.

### Consequences of the amendment

- ➕ Every Tier-A chain confirms on its own next block; Ethereum payments no longer wait ≈ 13 min.
- ➖ The accepted risks above; the re-check is the safety net for all of them.
- ➖ **Spec follow-up:** `@nearpay/spec` 0.4.0 still ships `BITCOIN_CONFIRMATION_TIERS` (1 below USD 1,000, else 2) and vectors with Solana profile `finality: "solana: finalized commitment"` and the Bitcoin tiers. The `ChainProfile` schema accepts the new values (rails' contract tests assert everything else against the vectors); a spec change should update the constant/vectors. Sentinel's independent settlement audit keeps reading Solana at `finalized` and Bitcoin with the spec tiers — it never gated success (it starts after `evt.rails.execution.settled`) and stays a second, stricter check.
- ➖ The post-success contradiction gap (below) now covers Solana and Bitcoin too.

---

*Original decision (2026-11-10, before the amendment), kept for the record:*

## Context
Rails accepted EVM evidence only once the transfer's block was at or below the chain's `finalized` tag (rails §8.7, §8.8). On an L2 that tag follows Ethereum L1 finality (batch posted, two L1 epochs): about 15–20 minutes. On the closed beta a USD 1 Base → Base payment showed 1Click `SUCCESS` at 08:39:35 and rails' verifier succeeded at 08:55:06; the checkout sat on "Confirming your payment" for over 15 minutes. That is not the product experience the owner wants for USD stablecoin payments on L2s.

An L2 block is produced by the chain's sequencer every 2 s (Base) or 0.25 s (Arbitrum One). Before the batch containing it is posted to L1, a sequencer could in principle reorganise it (operator fault, sequencer failover); such reorgs are rare and shallow in practice. After batch posting the L2 block can only change with an L1 reorg, and after L1 finality not at all.

## Decision
1. **Owner decision (2026-11-10):** a payment that settles on an EVM L2 is confirmed when the transfer is in a **Base / Arbitrum block with enough L2 confirmations**, not when Ethereum L1 finalizes it.
2. Chain profiles carry `evm_confirmation`: `{kind: finalized}` or `{kind: confirmations, blocks: N}`. Defaults: **Base `eip155:8453` 10 blocks (≈ 20 s)**, **Arbitrum `eip155:42161` 40 blocks (≈ 10 s)**, **Ethereum `eip155:1` `finalized`** (unchanged). Per-chain override `NP_EVM_CONFIRMATIONS_<CHAINID>` = `finalized` | N (1–100 000); an invalid value fails the boot.
3. Under `confirmations`, Tier-A evidence passes when the receipt has status 1, the transfer log matches exactly as before (recipient, asset contract, amount), and `latest − receipt.blockNumber + 1 ≥ N`; at that moment the receipt is **read again and must be in the same block (same number and block hash) with status 1** (reorg check) — otherwise nothing passes and the next attempt starts over. The same rule applies to the EVM destination watcher (scan up to `latest − N + 1`, re-read each attributed receipt; a failed re-read stops the cursor before that block). Evidence is recorded with `finality: confirmed` and `details.confirmations`, `details.required_confirmations`, `details.block_hash`.
4. **Post-success `finalized` re-check:** every execution settled on `confirmed` EVM evidence is followed until the chain's `finalized` tag covers its block, for at most `NP_FINALITY_RECHECK_WINDOW_S` (2 h), every `NP_FINALITY_RECHECK_INTERVAL_S` (60 s). At finality the receipt must still exist with status 1 in the same block hash.
   - Pass → `finality_recheck_status = finalized`.
   - Receipt gone, block hash changed, or status ≠ 1 at finality → `failed`; not finalized within the window → `timeout`. Either: `ALERT …` error log, `nearpay_rails_finality_recheck_total{chain,outcome}` (Prometheus rule `FinalityRecheckFailed` → Telegram), `finality_recheck_status` on the execution row and a `verifier` observation (applied = false) in the execution's evidence trail (console).
   - The execution is **never moved out of `succeeded`** (terminal states stay terminal, rails §5): the merchant may already have delivered; ops decide (refund, contact, write-off).
5. Sentinel's independent settlement audit (sentinel §7) keeps reading at `finalized`: it never gated success or delivery, it starts after `evt.rails.execution.settled` and retries for ≈ 85 minutes, so it is a second, independent L1-finality check of the same payments.
6. Checkout says what happens: while the attempt is `verifying` the confirming frame reads "Payment received — confirming on {destination chain} (usually under a minute)" for fast chains (Base, Arbitrum, NEAR, Solana).

## Consequences
- ➕ Base/Arbitrum payments complete in seconds instead of ≈ 15–20 minutes; the checkout no longer sits on "confirming".
- ➕ Every early confirmation is still proven against L1 finality afterwards, and a contradiction pages ops.
- ➖ **Accepted risk:** an L2 sequencer reorg before batch posting that drops or moves the transfer after NearPay showed success. The payment then reads `succeeded` without money having arrived, until ops act on the alert. Rare (no deep Base/Arbitrum sequencer reorg in normal operation), bounded by the 10/40-block depth and the reorg check, and surfaced within minutes by the re-check.
- ➖ **Gap:** the LLD and spec have no post-success contradiction state or event. The flag lives in rails (`finality_recheck_status`, evidence trail, metric) and is not on the spec `Execution`/`Evidence` objects or webhooks; payments, dashboard and console show nothing beyond the evidence trail. A spec change request (e.g. `Evidence.finality_recheck` or an `evt.rails.execution.finality_contradicted` event payments could turn into a dispute/review) is needed to close it.
- ➖ One more tracker loop and RPC reads (a `finalized` head + a receipt per confirmed payment per minute for ≈ 15–20 minutes).

## Alternatives considered
- **Keep `finalized` everywhere** — lost: 15–20 minutes per L2 payment, the beta showed the UX cost; the owner decided against it.
- **Wait for the L2 batch to be posted to L1 (Base `safe` tag)** — lost: still minutes (batch cadence) and provider-dependent tag semantics; the residual risk it removes is the rare sequencer reorg the re-check already detects.
- **Move a contradicted payment back (`succeeded → held`)** — lost: breaks the terminal-state rule every consumer relies on (payments, webhooks, reconciliation); a contradiction needs a human decision, not an automatic reversal.
- **Trust 1Click `SUCCESS` for L2s** — lost: violates INV-2 (evidence before success).

## Revisit when
- A Base or Arbitrum sequencer reorg deeper than the configured depth is observed, or `FinalityRecheckFailed` fires.
- Base/Arbitrum move to decentralised sequencing or change block times (re-derive N).
- Payment sizes grow so that the reorg exposure matters (consider `finalized` above a USD threshold, like Bitcoin's confirmation tiers).
- The spec gains a post-success contradiction field/event (close the gap above).
