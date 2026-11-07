# ADR-0021: Shared UI package in nearpay-kit

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** Owner; Principal Engineer (lead agent)
- **Invariants touched:** INV-5 (money is formatted from integer base units only)

## Context
Four frontends render the NearPay design (the owner's Figma): `nearpay-checkout` (isolated payment page, strict CSP with `style-src 'self' 'nonce-…'`, ADR-0010), `nearpay-dashboard`, `nearpay-console` and `nearpay-site` (ADR-0019). Stage 2 builds the dashboard and console screens in parallel with checkout changes, so tokens, components and money formatting must come from one versioned source or they drift. The kit already publishes versioned `@nearpay-internal/*` packages to the registry the frontends install from (foundations §4); creating a new repo needs a human first push.

## Decision
1. The design system ships as **`@nearpay-internal/ui`**, a versioned package in **`nearpay-kit`** (first release 0.2.0), consumed by checkout, dashboard, console and site like any other kit package (pinned version, no source imports).
2. Styling is **CSS custom properties** (`tokens.css`) + **plain CSS classes** (`styles.css`, `np-*` prefix, cascade layers) + **thin React 19 components** that only compose those classes. **No runtime CSS-in-JS** and no inline `style` attributes, so every consumer stays within its CSP (checkout: no `'unsafe-inline'`); merchant appearance is applied by setting token values on a root element, with contrast checked before use.
3. Fonts are **self-hosted DM Sans** (`fonts.css`, `font-src 'self'`); no third-party font CDN.
4. Amounts render through the package's `Money` (base units as bigint/integer string + decimals; no floats, INV-5).
5. The package carries a static gallery (Figma reference compositions, light and dark) used for visual review; it is not deployed.
6. Kit rules apply: no domain rules in the package, semver, one shared kit version.

## Consequences
- ➕ One source for tokens, components and money display; frontends upgrade by bumping a version.
- ➕ CSP-safe by construction; works in the checkout iframe and the static site alike.
- ➖ UI changes need a kit release before apps can use them (mitigated: frontends may run ahead with app-local components and upstream them).
- ➖ The kit now carries React as a peer dependency of one package (`react@^19`), and its tooling builds TSX and runs jsdom tests for `packages/ui`.

## Alternatives considered
- **Separate `nearpay-ui` repo** — lost: needs a human first push and its own release pipeline, for no isolation benefit (the package is internal and content-free).
- **In the `nearpay-js` monorepo** — lost: that repo publishes the public SDK (`@nearpay/js`, public scope); internal app UI would tangle public releases with app needs.
- **Copy-paste per app** — lost: tokens and components drift across four frontends immediately; no single fix for a bug such as a money-format or contrast defect.

## Revisit when
A frontend needs a UI release cadence the kit cannot follow, or the package grows enough to warrant its own repo and pipeline.
