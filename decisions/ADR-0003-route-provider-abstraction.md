# ADR-0003: Route-provider abstraction (1Click first)

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-2, INV-6, INV-8

## Context
1Click has no SLA, no webhooks, discretionary recovery, prospective fee changes and no testnet. Test mode needs a simulator. Intents-native payers do not need 1Click at all.

## Decision
`nearpay-rails` defines a `RouteProvider` port: `listAssets`, `quote(dry)`, `createExecution`, `submitDepositTx`, `getStatus`, `listChanges(cursor)`. Implementations: `oneclick` (live), `sandbox` (test mode, ADR-0007), `direct-intents` (signed transfers; same-asset Intents settlements). Provider choice is recorded per execution. Provider-specific statuses are mapped to the execution state machine (`domain-model.md` §4.3); unknown statuses map to `held`.

## Consequences
- ➕ A second provider can be added without touching payments; test mode is a provider swap (INV-8).
- ➖ Lowest-common-denominator risk; mitigated by provider capability flags (e.g. `supportsExactOutput`, `supportsMemo`).

## Alternatives considered
- **Call 1Click directly from payments:** fastest to build, but couples merchant state to one provider's quirks.

## Revisit when
A second live provider is integrated (review the port's shape against it).
