# nearpay-brain

The design documentation of **NearPay** (https://nearpay.money): non-custodial, chain-abstracted merchant payments on NEAR Intents. This repo holds no code. The code repositories carry only a README and an AGENTS.md; everything cross-cutting (requirements, architecture, decisions, low-level designs, audits, runbooks) lives here.

Part of [NearPay](https://github.com/nearpay-money/nearpay) — start there for the live demo, the architecture diagram and the full "do X, see Y" walkthrough.

- **Where it sits:** every other NearPay repo links here for its design. Nothing is built or deployed from this repo.
- **Status:** NearPay is implemented through Stage 4 and runs on NEAR mainnet as a closed beta (site https://nearpay.money, docs https://docs.nearpay.money). The HLD is v1.1 with later owner decisions recorded as ADRs; each LLD is updated in place as its component is built, with a version line at the top. One part is not live: the auto-debit vault contract is built and tested but not deployed on mainnet yet (planned account `v1.nearpay-money.near`).

## Index for a reviewer

| You want to check | Read |
|---|---|
| What NearPay is and the invariants it must keep (INV-1 non-custody, INV-2 evidence before success, …) | [product/brief.md](product/brief.md), [product/requirements.md](product/requirements.md) §1 |
| The high-level design (HLD): containers, trust boundaries, flows, degradation | [architecture/hld.md](architecture/hld.md) (container diagram in §5), [architecture/flows.md](architecture/flows.md) |
| Every decision and why | [decisions/README.md](decisions/README.md): ADR-0001 to ADR-0028 (there is no 0023), one file each in [decisions/](decisions/) |
| The low-level design (LLD) of one service or app | [lld/README.md](lld/README.md) lists one LLD per component: [identity](lld/identity.md), [payments](lld/payments.md), [rails](lld/rails.md), [sentinel](lld/sentinel.md), [agentpay](lld/agentpay.md), [compliance](lld/compliance.md), [webhooks](lld/webhooks.md), [notify](lld/notify.md), [billing](lld/billing.md), [sandbox](lld/sandbox.md), [checkout and js](lld/checkout-and-js.md), [dashboard and console](lld/dashboard-and-console.md), [vault contract](lld/contracts-vault.md), [SDKs, CLI and site](lld/sdks-and-tools.md), [deploy, infra, testbed](lld/platform.md), [spec and kit](lld/00-foundations.md) |
| The audit of each LLD and of the whole system | [lld/audits/](lld/audits/) (same file names), [lld/audits/system.md](lld/audits/system.md) |
| How the design was attacked and what risk remains | [quality/stress-test.md](quality/stress-test.md), [quality/risks.md](quality/risks.md), [architecture/security.md](architecture/security.md) |
| An incident and what changed because of it | The kill-switch stop of 2026-11-11 in the beta: [lld/sentinel.md](lld/sentinel.md) §18 (what happened, root cause, new rules), [lld/audits/sentinel.md](lld/audits/sentinel.md) finding L-SN-01, [ADR-0028](decisions/ADR-0028-scoped-integrity-containment.md), runbook RB-03 in [operations/runbooks.md](operations/runbooks.md). There is no separate incidents folder. |
| How the closed beta is hosted | [lld/platform.md](lld/platform.md) §1.4 |
| Stages and what shipped in each | [roadmap/development-plan.md](roadmap/development-plan.md) |

### Decisions a reviewer is most likely to ask about

| ADR | Decision |
|---|---|
| [0022](decisions/ADR-0022-evm-l2-confirmations.md) | A payment confirms on one block of the chain its settlement evidence is read from (EVM 1 block, Solana `confirmed`, Bitcoin 1 confirmation, NEAR `final`), re-checked at finality afterwards |
| [0024](decisions/ADR-0024-open-onboarding.md) | Open onboarding: complete self-declared business details go live at once; no business review; registration number optional; the geo policy still blocks |
| [0025](decisions/ADR-0025-npm-scope-nearpay-money.md) | Public npm packages ship under `@nearpay-money/*`; internal ones stay `@nearpay-internal/*`. The `@nearpay` scope belongs to an unrelated company |
| [0026](decisions/ADR-0026-google-sign-in.md) | "Continue with Google" for the merchant dashboard alongside passkeys; passkeys still confirm sensitive actions |
| [0027](decisions/ADR-0027-payout-wallets-activate-immediately.md) | Payout wallets (settlement destinations) are active immediately after the ownership signature; no cool-off by default |
| [0028](decisions/ADR-0028-scoped-integrity-containment.md) | A sentinel integrity failure is contained per merchant; the global kill switch engages only on corroborated evidence, and stale automatic stops release themselves |

## Verify this repo

There is nothing to run: this repo is Markdown only (no package manifest, no build). What can be checked:

1. Every path in the tables above exists in this repo: open any of them.
2. `ls decisions/` → 27 ADR files (`ADR-0001` to `ADR-0028` without 0023), the template `ADR-0000-template.md` and the index `README.md`.
3. `ls lld/ lld/audits/` → 16 LLD files plus `README.md`, and 17 audit files (16 with matching names, plus `system.md`).
4. A design claim can be traced into code: each LLD names the tables, routes, events and tests of its component, and each code repo's README links back to its LLD. For example ADR-0022 → `config/chains.yaml` and `src/verifiers/` in [nearpay-rails](https://github.com/nearpay-money/nearpay-rails); ADR-0028 → `src/kill-switch/` in [nearpay-sentinel](https://github.com/nearpay-money/nearpay-sentinel).

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
| 9 | [architecture/repositories.md](architecture/repositories.md) | The 26 repos, dependency rules, directory structures |
| 10 | [decisions/](decisions/README.md) | ADR-0001…0028 |
| 11 | [quality/stress-test.md](quality/stress-test.md) · [capacity.md](quality/capacity.md) · [risks.md](quality/risks.md) | How the design was attacked, sized and what risk remains |
| 12 | [operations/slos.md](operations/slos.md) · [runbooks.md](operations/runbooks.md) | Running it |
| 13 | [lld/README.md](lld/README.md) | **Low-level designs** per component + audits (`lld/audits/`, incl. the system audit) — implement from these |
| 14 | [roadmap/development-plan.md](roadmap/development-plan.md) · [roadmap.md](roadmap/roadmap.md) · [open-questions.md](roadmap/open-questions.md) | Stages, parallel steps, demo scripts; spikes SP-01…15 |
| — | [reference/landscape.md](reference/landscape.md) · [glossary.md](reference/glossary.md) | Verified external facts (with dates) and terms |

## The design in five sentences

1. Merchants use a Stripe-style API, widget and webhooks; payers pay **any token on any chain**; merchants receive **any asset** on a chain or in their NEAR Intents account.
2. Value moves through **NEAR Intents** (1Click `EXACT_OUTPUT` quotes, or payer-signed Verifier transfers), so **NearPay never holds funds** and its outages never strand money.
3. A payment is `succeeded` only with **independent on-chain evidence**, detected by watchers (amount-tagged), an Intents indexer, provider polling and an Explorer feed.
4. Deposit instructions are released only with a **co-signature from an independent sentinel**, and a payout wallet (settlement destination) becomes active only on a verified ownership signature, plus a second approver where the account has more than one owner or admin; there is no cool-off (ADR-0027). These close the two ways money could be misdirected.
5. Ten TypeScript services + a Rust vault, each in its own repo with its own database, run as separate containers in one Docker Compose stack on a single host (ADR-0015). That is how the closed beta runs today (`lld/platform.md` §1.4); a netcup server is Stage 5 and the two-site k3s topology is the post-v1 target. The vault is not deployed on mainnet yet.

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

## Built with

Markdown and Mermaid diagrams; no libraries. Licence: Apache-2.0 (`LICENSE`).

Code and documentation are written with AI assistance (Claude) under human direction and review. Built for the NEAR Hackathon (Nov 2026).
