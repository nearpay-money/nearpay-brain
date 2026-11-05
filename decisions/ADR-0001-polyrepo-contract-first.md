# ADR-0001: Polyrepo by bounded context, contract-first, central brain

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** — (enables all)

## Context
The product owner does not want a monorepo for everything. Development is led by AI coding agents that work best in small, self-describing codebases. A payments platform has hard security boundaries (on-chain code, deposit-address display, production access, independent auditing) that benefit from separate review and credentials.

## Decision
1. One repo per bounded context / deployable / publishable unit / security boundary (26 repos at acceptance; 27 after ADR-0016 added the `nearpay` entry repo — see `architecture/repositories.md`).
2. `nearpay-spec` is the single source of truth for all contracts; types and validators are generated and published; no hand-written contract types elsewhere.
3. Repos depend only on published artifacts at pinned versions.
4. `nearpay-brain` holds all cross-cutting documentation; repos carry only `README.md`, `AGENTS.md`, `CODEOWNERS`.
5. Cross-repo verification lives in `nearpay-testbed` (consumer-driven contract tests, E2E).

## Consequences
- ➕ Independent deploys, blast-radius isolation, clear ownership, agent-sized contexts.
- ➕ Contract drift is caught at the spec, not in production.
- ➖ Cross-cutting changes need ordered PRs (spec → kit → services). Mitigated by automation bots that open bump PRs.
- ➖ More CI pipelines; mitigated by reusable workflows in `nearpay-infra/.github`.

## Alternatives considered
- **Monorepo (Nx/Turborepo):** atomic cross-cutting changes, but rejected by the owner and blurs security boundaries.
- **Fewer repos (one "backend" repo):** fewer pipelines, but a single blast radius for money paths and auth.

## Revisit when
Cross-repo PR chains exceed ~20% of all changes for a quarter.
