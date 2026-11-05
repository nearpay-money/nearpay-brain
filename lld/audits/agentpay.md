# Audit — agentpay

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | pass |
| G3 State | fixed: L-AP-01 |
| G4 API | fixed: L-AP-03 |
| G5 Events | pass |
| G6 Concurrency | fixed: L-AP-02 |
| G7 Failure | pass |
| G8 Security | pass |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-AP-01 | G3 | High | Checkout agent requirements were fetched with a GET although they must create an attempt/execution (binding) | `POST …/agent_pay/requirements` → payments `agent_attempts` | §5 |
| L-AP-02 | G6 | Critical | Single-use bindings combined with requirement caching in the middleware would reject every concurrent agent after the first | Requirements issued per 402, never shared; issuance rate-limited (SDK LLD updated) | §3, sdks §2 |
| L-AP-03 | G4 | High | Minimum fee conversion needed asset decimals and price; `payTo` default needed the merchant's Intents destination — neither source was specified | rails `GET /internal/assets/{id}`, identity settlement context | §10 |
