# ADR-0019: One `nearpay-site` repo for the docs and marketing sites

- **Status:** Accepted
- **Date:** 2026-11-05
- **Deciders:** Owner; Principal Engineer
- **Amends:** ADR-0001 (repo list: `nearpay-docs` + `nearpay-web` → `nearpay-site`)
- **Invariants touched:** none

## Context
`nearpay-docs` (developer docs) and `nearpay-web` (marketing, pricing, legal) are both public, static, content-only Astro sites with no secrets, sessions or money paths, the same tooling and the same reviewers. Repos are split by trust level, audience and release cycle (ADR-0001); these two share all three.

## Decision
Merge them into **`nearpay-site`**: one Astro project building `dist/docs` (served at `docs.<domain>`) and `dist/web` (apex domain). `nearpay-checkout`, `nearpay-js`, `nearpay-dashboard` and `nearpay-console` stay separate (payment-page isolation, published SDK, session/money settings, internal ops tool).

## Consequences
- ➕ One toolchain, shared layout and components, one CI; 26 repos instead of 27.
- ➖ Docs and marketing release together (acceptable for static content).

## Alternatives considered
- **All frontends in one repo** — rejected: weakens checkout isolation (ADR-0010), tangles the published `@nearpay/js` releases with app deploys, mixes the internal console with public code.

## Revisit when
Docs need versioned releases independent of marketing pages.
