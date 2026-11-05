# Low-Level Design (LLD)

> Status: see table · Owner: Principal Engineer · Baseline: HLD v1.1, ADR-0001…0016.

An LLD is **dev-ready** when an engineer (human or AI agent) can implement the component without making a design decision: every table, endpoint, event, state transition, job, timeout, config key and test case is named. Anything not decided is an explicit open item linked to a spike.

## Documents

| LLD | Component(s) | Audit | Status |
|---|---|---|---|
| [00-foundations.md](00-foundations.md) | Cross-cutting: `nearpay-spec`, `nearpay-kit`, conventions every service follows | [audit](audits/00-foundations.md) | see audit |
| [identity.md](identity.md) | `nearpay-identity` | [audit](audits/identity.md) | see audit |
| [payments.md](payments.md) | `nearpay-payments` | [audit](audits/payments.md) | see audit |
| [rails.md](rails.md) | `nearpay-rails` | [audit](audits/rails.md) | see audit |
| [sentinel.md](sentinel.md) | `nearpay-sentinel` | [audit](audits/sentinel.md) | see audit |
| [agentpay.md](agentpay.md) | `nearpay-agentpay` | [audit](audits/agentpay.md) | see audit |
| [compliance.md](compliance.md) | `nearpay-compliance` | [audit](audits/compliance.md) | see audit |
| [webhooks.md](webhooks.md) | `nearpay-webhooks` | [audit](audits/webhooks.md) | see audit |
| [notify.md](notify.md) | `nearpay-notify` | [audit](audits/notify.md) | see audit |
| [billing.md](billing.md) | `nearpay-billing` | [audit](audits/billing.md) | see audit |
| [sandbox.md](sandbox.md) | `nearpay-sandbox` | [audit](audits/sandbox.md) | see audit |
| [checkout-and-js.md](checkout-and-js.md) | `nearpay-checkout`, `nearpay-js` | [audit](audits/checkout-and-js.md) | see audit |
| [dashboard-and-console.md](dashboard-and-console.md) | `nearpay-dashboard`, `nearpay-console` | [audit](audits/dashboard-and-console.md) | see audit |
| [contracts-vault.md](contracts-vault.md) | `nearpay-contracts` (subscription vault) | [audit](audits/contracts-vault.md) | see audit |
| [sdks-and-tools.md](sdks-and-tools.md) | `nearpay-node`, `nearpay-agentkit`, `nearpay-cli`, `nearpay-docs`, `nearpay-web` | [audit](audits/sdks-and-tools.md) | see audit |
| [platform.md](platform.md) | `nearpay-deploy`, `nearpay-infra`, `nearpay-testbed`, `nearpay` (entry) | [audit](audits/platform.md) | see audit |
| — | Whole system | [audits/system.md](audits/system.md) | see audit |

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
