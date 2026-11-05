# ADR-0007: Test mode backed by a deterministic sandbox rail

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-8

## Context
Stripe-grade DX requires test mode. The NEAR Intents Verifier has **no testnet deployment**; mainnet tests cost money and are slow and non-deterministic.

## Decision
- `nearpay-sandbox` implements the `RouteProvider` contract and simulated chains with deterministic scenarios (triggered by amount suffix / magic refund address, `api-conventions.md` §8) plus `test_helpers` to advance executions step by step.
- `livemode = false` objects always route to the sandbox; the check sits in `rails` provider selection and is asserted again in the sandbox (rejects live-mode calls) and in the 1Click adapter (rejects test-mode calls).
- Staging additionally runs **mainnet canaries** (≤ USD 5 budget) to catch drift between sandbox and reality; recorded 1Click responses become fixtures.

## Consequences
- ➕ Fast, free, reproducible integration testing for merchants and our own E2E.
- ➖ Simulator drift; mitigated by canaries + fixture refresh + contract tests on the provider adapter.

## Alternatives considered
- **Mainnet with tiny amounts for test mode:** costs, non-determinism, fee/ToS implications for merchant tests.

## Revisit when
NEAR Intents ships an official testnet.
