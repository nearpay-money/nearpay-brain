# Audit — sandbox

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | pass |
| G3 State | pass |
| G4 API | pass |
| G5 Events | pass |
| G6 Concurrency | pass |
| G7 Failure | pass |
| G8 Security | pass |
| G9 Observability | pass |
| G10 Testability | fixed: L-SB-01 |
| G11 Stage fit | pass |
| G12 Decision-free | fixed: L-SB-02 |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-SB-01 | G10 | Medium | Scenario selection undefined for asset-priced intents | Last two digits of base units | §4 |
| L-SB-02 | G12 | Medium | "Same canonical message as 1Click" was ambiguous (signature field inclusion) | Exact construction specified, signature excluded | §3 |
