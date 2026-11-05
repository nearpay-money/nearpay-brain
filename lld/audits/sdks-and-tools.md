# Audit — SDKs and tools

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | pass |
| G3 State | pass |
| G4 API | pass |
| G5 Events | pass |
| G6 Concurrency | fixed: L-SDK-01 |
| G7 Failure | pass |
| G8 Security | pass |
| G9 Observability | pass |
| G10 Testability | fixed: L-SDK-02 |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-SDK-01 | G6 | Critical | x402 middleware cached requirements per resource, reusing single-use bindings across agents | One requirements call per 402 response | §2 |
| L-SDK-02 | G10 | Low | Webhook and agent-signer vectors needed a shared source | Consume `nearpay-spec/test-vectors` | §6 |
