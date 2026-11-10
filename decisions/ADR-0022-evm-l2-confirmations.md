# ADR-0022: EVM L2 payments confirm on L2 block confirmations, re-checked at L1 finality

- **Status:** Accepted
- **Date:** 2026-11-10
- **Deciders:** Owner (decision); Principal Engineer (lead agent, implementation)
- **Invariants touched:** INV-2 (evidence before success), INV-3 (truth wins)
- **Amends:** ADR-0012 decision "EVM chains … `finalized` block tag" for Base and Arbitrum (Ethereum keeps it)

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
