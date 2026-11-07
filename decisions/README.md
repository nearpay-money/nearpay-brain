# Architecture Decision Records

| ADR | Title | Status |
|---|---|---|
| [0001](ADR-0001-polyrepo-contract-first.md) | Polyrepo by bounded context, contract-first, central brain | Accepted |
| [0002](ADR-0002-non-custodial-near-intents.md) | Non-custodial settlement through NEAR Intents | Accepted |
| [0003](ADR-0003-route-provider-abstraction.md) | Route-provider abstraction (1Click first) | Accepted |
| [0004](ADR-0004-postgres-outbox-nats.md) | Postgres per service + outbox → NATS JetStream | Accepted |
| [0005](ADR-0005-settlement-destinations.md) | Any-asset settlement destinations with attestation and cool-off | Accepted |
| [0006](ADR-0006-fees-via-appfees.md) | Revenue via in-flight appFees; fee schedule as data | Accepted |
| [0007](ADR-0007-test-mode-sandbox.md) | Test mode backed by a deterministic sandbox rail | Accepted |
| [0008](ADR-0008-subscriptions.md) | Subscriptions: send_invoice everywhere, auto_debit via NEAR vault | Accepted |
| [0009](ADR-0009-agent-payments-x402.md) | Agent payments via x402 v2 | Accepted |
| [0010](ADR-0010-checkout-isolation.md) | Checkout as isolated static SPA; widget = iframe | Accepted |
| [0011](ADR-0011-netcup-k3s-two-site.md) | netcup root servers, k3s, NUE/VIE/AMS | Accepted (post-v1 target; v1 → 0015) |
| [0012](ADR-0012-independent-verification.md) | Evidence before success | Accepted |
| [0013](ADR-0013-typescript-rust.md) | TypeScript services/frontends, Rust contracts | Accepted |
| [0014](ADR-0014-auth-gateway.md) | Traefik gateway + ForwardAuth, internal JWTs | Accepted |
| [0015](ADR-0015-v1-single-host-compose.md) | v1: every service a container in one Compose stack, single host | Accepted |
| [0016](ADR-0016-entry-repo-and-hackathon.md) | Thin `nearpay` entry repo; NEAR Hackathon compliance | Accepted (decision 5 amended by 0017 and 0020; decision 7 amended by 0020) |
| [0017](ADR-0017-git-workflow.md) | Git workflow: `main` + feature branches merged by pull request | Accepted (decisions 3–5 amended by 0020: merges delegated to the lead agent) |
| [0018](ADR-0018-list-filters-not-search.md) | Typed filters on list endpoints instead of search endpoints | Accepted |
| [0019](ADR-0019-site-repo.md) | One `nearpay-site` repo for the docs and marketing sites | Accepted |
| [0020](ADR-0020-v1-delivery-and-vendor-scope.md) | v1 by 2026-11-14: test-mode stage gates + one mainnet demo, self-declared KYB, local screening, keyless FX/prices, AI vault audits, merges delegated to the lead agent | Accepted |
| [0021](ADR-0021-shared-ui-package.md) | Shared UI package `@nearpay-internal/ui` in nearpay-kit (CSS custom properties + plain classes + thin React components, CSP-safe, self-hosted DM Sans) | Accepted |

New ADRs copy [ADR-0000-template.md](ADR-0000-template.md). Never edit an accepted ADR's decision — supersede it.
