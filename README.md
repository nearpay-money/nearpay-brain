# nearpay-brain

The single source of truth for **NearPay** (`nearpay.money`) — non-custodial, chain-abstracted merchant payments on NEAR Intents. Code repositories stay surgical (README + AGENTS.md only); everything cross-cutting lives here.

> Status: **HLD v1.1 accepted and stress-tested; LLD v1.1 dev-ready (all audits pass)** (2026-11-05). Scope: production, delivered in 5 stages (Stage 1 = NEAR Hackathon). Implementation not started.

## Reading order

| # | Doc | Read when |
|---|---|---|
| 1 | [product/brief.md](product/brief.md) | Always first — what and why |
| 2 | [product/requirements.md](product/requirements.md) | Invariants **INV-1…9**, FR/NFR IDs used everywhere |
| 3 | [architecture/hld.md](architecture/hld.md) | The system design |
| 4 | [architecture/domain-model.md](architecture/domain-model.md) | Objects, IDs, state machines, payment outcome matrix |
| 5 | [architecture/flows.md](architecture/flows.md) | Sequence diagrams for every core flow |
| 6 | [architecture/api-conventions.md](architecture/api-conventions.md) | Public API, idempotency, versioning, webhooks, test mode |
| 7 | [architecture/security.md](architecture/security.md) | Trust boundaries, keys, threats T-01…T-16 |
| 8 | [architecture/infrastructure.md](architecture/infrastructure.md) | netcup sites, k3s platform, HA/DR, delivery, observability |
| 9 | [architecture/repositories.md](architecture/repositories.md) | The 27 repos, dependency rules, directory structures |
| 10 | [decisions/](decisions/README.md) | ADR-0001…0014 |
| 11 | [quality/stress-test.md](quality/stress-test.md) · [capacity.md](quality/capacity.md) · [risks.md](quality/risks.md) | How the design was attacked, sized and what risk remains |
| 12 | [operations/slos.md](operations/slos.md) · [runbooks.md](operations/runbooks.md) | Running it |
| 13 | [lld/README.md](lld/README.md) | **Low-level designs** per component + audits (`lld/audits/`, incl. the system audit) — implement from these |
| 14 | [roadmap/development-plan.md](roadmap/development-plan.md) · [roadmap.md](roadmap/roadmap.md) · [open-questions.md](roadmap/open-questions.md) | Stages, parallel steps, demo scripts; spikes SP-01…15 |
| — | [reference/landscape.md](reference/landscape.md) · [glossary.md](reference/glossary.md) | Verified external facts (with dates) and terms |

## The design in five sentences

1. Merchants use a Stripe-style API, widget and webhooks; payers pay **any token on any chain**; merchants receive **any asset** on a chain or in their NEAR Intents account.
2. Value moves through **NEAR Intents** (1Click `EXACT_OUTPUT` quotes, or payer-signed Verifier transfers), so **NearPay never holds funds** and its outages never strand money.
3. A payment is `succeeded` only with **independent on-chain evidence**, detected by watchers (amount-tagged), an Intents indexer, provider polling and an Explorer feed.
4. Deposit instructions are released only with a **co-signature from an independent sentinel**, and settlement destinations are attested, cooled-off and two-person-approved — the two ways money could be misdirected are closed.
5. Ten TypeScript services + a Rust vault, each in its own repo with its own database, run as separate containers in one Docker Compose stack — locally against NEAR mainnet first, then on a single netcup server behind Cloudflare (ADR-0015); the two-site k3s topology is the post-v1 target.

## Conventions for this repo

- Every doc carries a version line; bump it and add a revision-log row in `architecture/hld.md` for material changes.
- IDs are stable: `INV-n`, `FR-…`, `NFR-…`, `ADR-nnnn`, `T-nn` (threats), `F-nn` (stress-test findings), `R-nn` (risks), `SP-nn` (spikes), `RB-nn` (runbooks).
- Decisions change only through a new ADR that supersedes the old one.
- External facts go in `reference/landscape.md` with source and date; ⚠ marks anything unverified and must link to a spike.

## Verification (stress-test pass 4)

Automated checks run on this folder before every release of the brain:

- all relative links resolve; all Mermaid diagrams render;
- every repo in `architecture/repositories.md` exists as a sibling folder with README, AGENTS.md and CODEOWNERS;
- every ⚠ in the brain has a matching `SP-`/`Q-` entry;
- LLD cross-references (internal endpoints, event producers/consumers, scopes, routes) are consistent (`lld/audits/system.md`).
