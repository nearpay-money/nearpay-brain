# Low-Level Design (LLD)

> Status: **dev-ready** — all component audits and the system audit (`audits/system.md`) pass · Owner: Principal Engineer · Baseline: HLD v1.1, ADR-0001…0021. The version in the table is the latest version line at the top of each LLD (step 4.1, spec 0.4.0).

An LLD is **dev-ready** when an engineer (human or AI agent) can implement the component without making a design decision: every table, endpoint, event, state transition, job, timeout, config key and test case is named. Anything not decided is an explicit open item linked to a spike.

## Documents

| LLD | Component(s) | Audit | Status |
|---|---|---|---|
| [00-foundations.md](00-foundations.md) | Cross-cutting: `nearpay-spec`, `nearpay-kit`, conventions every service follows | [audit](audits/00-foundations.md) | dev-ready (v1.7 + step 3.1 and spec 0.4.0 updates) |
| [identity.md](identity.md) | `nearpay-identity` | [audit](audits/identity.md) | dev-ready (v1.12: Solana and BIP-322 ownership attestations — spec 0.4.0) |
| [payments.md](payments.md) | `nearpay-payments` | [audit](audits/payments.md) | dev-ready (v1.12) |
| [rails.md](rails.md) | `nearpay-rails` | [audit](audits/rails.md) | dev-ready (v1.15: step 4.2 as built — tier pinning and pool health, Solana/Bitcoin deposit signals, sender rule, vault forwarding storage and outbox (§21.6); Solana and Bitcoin Tier A, vault event forwarding, `VaultEventRecord` — spec 0.4.0) |
| [sentinel.md](sentinel.md) | `nearpay-sentinel` | [audit](audits/sentinel.md) | dev-ready (v1.10: Solana and Bitcoin coverage — spec 0.4.0) |
| [agentpay.md](agentpay.md) | `nearpay-agentpay` | [audit](audits/agentpay.md) | dev-ready (v1.9) |
| [compliance.md](compliance.md) | `nearpay-compliance` | [audit](audits/compliance.md) | dev-ready (v1.7 + spec 0.2.1 update) |
| [webhooks.md](webhooks.md) | `nearpay-webhooks` | [audit](audits/webhooks.md) | dev-ready (v1.10: `mandate.*` types — spec 0.4.0) |
| [notify.md](notify.md) | `nearpay-notify` | [audit](audits/notify.md) | dev-ready (v1.8: payer mandate mails — spec 0.4.0) |
| [billing.md](billing.md) | `nearpay-billing` | [audit](audits/billing.md) | dev-ready (v1.10: Stage 4 keeper, linkage rules and auto-debit contracts §14 — spec 0.4.0) |
| [sandbox.md](sandbox.md) | `nearpay-sandbox` | [audit](audits/sandbox.md) | dev-ready (v1.8.1: step 4.2 as built — stand-in "As implemented" with the serde_json input layer, commit-ordered feed, scenario `94` on `near_account`, Solana/Bitcoin as built; subscription-vault stand-in (OQ-T-03), Solana and Bitcoin simulation — spec 0.4.0) |
| [checkout-and-js.md](checkout-and-js.md) | `nearpay-checkout`, `nearpay-js` | [audit](audits/checkout-and-js.md) | dev-ready (v1.7: Solana wallet path, Solana Pay / BIP-21 QR — spec 0.4.0) |
| [dashboard-and-console.md](dashboard-and-console.md) | `nearpay-dashboard`, `nearpay-console` | [audit](audits/dashboard-and-console.md) | dev-ready (v1.9) |
| [contracts-vault.md](contracts-vault.md) | `nearpay-contracts` (subscription vault) | [audit](audits/contracts-vault.md) | dev-ready (v1.6: step 4.1 implementation decisions, per-mandate cap, AI audits, re-audit and re-check 2 folded in) |
| [sdks-and-tools.md](sdks-and-tools.md) | `nearpay-node`, `nearpay-agentkit`, `nearpay-cli`, `nearpay-site` | [audit](audits/sdks-and-tools.md) | dev-ready (v1.9) |
| [platform.md](platform.md) | `nearpay-deploy`, `nearpay-infra`, `nearpay-testbed`, `nearpay` (entry) | [audit](audits/platform.md) | dev-ready (v1.6) |
| — | Whole system | [audits/system.md](audits/system.md) | dev-ready (v1.1) |

## Section template (every component LLD)

1. Scope — owns / does not own
2. Runtime — Compose services, process types, ports, resources
3. Code layout — directories → responsibilities
4. Data — DDL (tables, constraints, indexes), retention
5. State machines — transition table with guards and side effects
6. API — public and internal endpoints (auth, request, response, errors, idempotency)
7. Events — produced and consumed (subject, payload, handler, idempotency)
8. Algorithms — pseudocode for critical paths
9. Jobs — trigger, cadence, leadership, batch, failure behaviour
10. Integrations — endpoint, timeout, retries, breaker, budget
11. Configuration — env vars and secret files
12. Security — authz matrix, data classification
13. Observability — metrics, logs, alerts
14. Failure modes
15. Tests — named cases per level
16. Stage mapping — what ships in which stage (`roadmap/development-plan.md`)

## Audit method (applied to each LLD, then system-wide)

Each component audit checks 12 gates and records findings (`L-<comp>-nn`) with the fix applied to the LLD:

| Gate | Question |
|---|---|
| G1 Requirements | Every FR/NFR/INV the component touches is implemented by a named mechanism |
| G2 Data | Every field has a type, nullability and owner; every query path has an index; money is integer |
| G3 State | Every transition has a guard, a side effect list, and an event; no unreachable/terminal-trap states |
| G4 API | Every endpoint has auth, schemas, errors, idempotency; matches `api-conventions.md` |
| G5 Events | Every produced event has ≥ 1 consumer or a public mapping; every consumed event has a producer |
| G6 Concurrency | Locks, versions, unique constraints cover every race in the flow |
| G7 Failure | Every external call has timeout, retry, breaker; every failure has a defined outcome |
| G8 Security | Authz per endpoint; secrets in files; PII classified; invariants INV-1/2/7/8 enforced in code paths |
| G9 Observability | Metrics/alerts exist for every SLO the component affects |
| G10 Testability | Every rule has a named test; sandbox scenarios cover every outcome |
| G11 Stage fit | Stage mapping exists; Stage-1 subset is coherent and demoable |
| G12 Decision-free | No "TBD" without a spike ID; an implementer makes no design choices |
