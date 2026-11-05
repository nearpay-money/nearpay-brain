# ADR-0012: Evidence before success — independent settlement verification

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-2, INV-3, INV-9

## Context
Merchants fulfil orders on `payment_intent.succeeded`. A false positive (provider bug, spoofed response, compromised dependency) is a direct loss. 1Click status is the only real-time provider signal and it has no SLA.

## Decision
- On provider `SUCCESS`, `rails` moves the execution to `verifying` and checks evidence independently:
  - **Tier A** — NEAR & Intents (`intents.near` events via own indexer), EVM chains (receipt + ERC-20 `Transfer`/native value to recipient, `finalized` block tag), Solana (finalized tx, token balance delta), Bitcoin (output to address, ≥ 2 confirmations for amounts ≥ USD 1k, else 1).
  - **Tier B** — other chains: provider-attested, with per-payment and daily per-merchant limits, plus reconciliation via Explorer.
- Evidence = `{chain, tx_hash, block, recipient, asset, amount, finality}`, stored append-only.
- Amount < expected → `requires_review` (auto-accept within merchant tolerance). Missing evidence after the chain's verification timeout → `held` + page.
- `nearpay-sentinel` (separate code, credentials and pipeline) co-signs every deposit instruction before checkout shows it (ADR-0010) and independently re-verifies a sample of succeeded payments (100% for ≥ USD 10k) against chain evidence.
- Hot poller, Explorer feed, NEAR indexer and **destination watchers** (amount-tag attribution, `domain-model.md` §3) are independent detectors; any of them can trigger verification. For tagged executions on Tier-A chains, a watcher match *is* the evidence, so success no longer depends on provider status.

## Consequences
- ➕ No success without proof; resilient to provider status outages.
- ➖ Adds seconds (EVM finality can add minutes on L1) and RPC costs; acceptable for correctness.

## Alternatives considered
- **Trust provider status:** simplest; unacceptable single point of truth.

## Revisit when
Provider offers cryptographic settlement receipts verifiable without RPC calls.
