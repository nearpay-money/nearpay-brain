# ADR-0025: Public npm packages ship under `@nearpay-money`

- **Status:** Accepted
- **Date:** 2026-11-11
- **Deciders:** owner (Adarsh)
- **Invariants touched:** none (package names only; contracts, wire formats and versions' semantics unchanged)

## Context
The public SDK packages were named `@nearpay/*` (`spec`, `node`, `js`, `react`, `cli`, `x402`, `agent-signer`, `mcp`)
and lived only on the local Verdaccio registry (ADR-0001, system audit S-11). Open question Q-BIZ-03 kept
`@nearpay-money` as the fallback if the `@nearpay` npm scope was unavailable. It is: the `@nearpay` scope on the public
npm registry belongs to an unrelated company. The packages are about to be published publicly.

## Decision
1. Every public package is named `@nearpay-money/<name>` (npm org `nearpay-money`, matching the GitHub org and the
   domain `nearpay.money`): `@nearpay-money/spec`, `/node`, `/js`, `/react`, `/cli`, `/x402`, `/agent-signer`, `/mcp`.
   The bins keep their names (`nearpay`, `nearpay-mcp`).
2. Nothing is ever published under `@nearpay`. Internal packages stay `@nearpay-internal/*` and are never published
   to the public registry.
3. The rename is a minor bump of each package (spec 0.5.0, node 0.4.0, js/react 0.2.0, cli 0.2.0, x402/agent-signer/mcp
   0.3.0; kit's spec-dependent packages 0.5.0). Old `@nearpay/*` versions remain on the local registry only so older
   lockfiles keep installing.
4. Services still pinned to older `@nearpay-internal/*` tarballs (which import `@nearpay/spec`) alias that name to the
   renamed package with a pnpm override `"@nearpay/spec": "npm:@nearpay-money/spec@<version>"` until they move to kit
   ≥ 0.5.0. No `@nearpay` package is ever fetched; the alias resolves `@nearpay-money/spec` from the local registry.
5. Publishing to npm is a tagged GitHub Actions release in each publishing repo (nearpay-spec, nearpay-node,
   nearpay-js, nearpay-cli, nearpay-agentkit); steps in `nearpay/RELEASING.md`.

Package names in earlier ADRs (0009, 0019, 0021, 0022) read as `@nearpay-money/*` from this decision on.

## Consequences
Every repository's imports, manifests, lockfiles, `.npmrc` scope routing, Dockerfiles and docs move to the new scope in
one change set. Old internal kit tarballs need the alias override (decision 4) until their consumers upgrade.

## Alternatives considered
- Keep `@nearpay` locally and rename only at publish time: two names for one package, and dependency confusion
  with the unrelated `@nearpay` scope on npmjs.
- Unscoped names (`nearpay-node`, …): unscoped names are first-come and harder to keep consistent.

## Revisit when
The `@nearpay` scope becomes available to NearPay (not expected).
