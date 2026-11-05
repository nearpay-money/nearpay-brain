# Audit — subscription vault contract

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | fixed: L-VT-02 |
| G3 State | pass |
| G4 API | pass |
| G5 Events | pass |
| G6 Concurrency | pass |
| G7 Failure | fixed: L-VT-01 |
| G8 Security | fixed: L-VT-03 |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-VT-01 | G7 | Medium | Keeper needed machine-readable failure reasons | Stable error strings | §3 |
| L-VT-02 | G2 | Medium | Storage staking for mandates unspecified (unbounded state growth by any caller) | NEP-145 storage with per-mandate minimum; refund on missing storage | §3 |
| L-VT-03 | G8 | High | Mandates can target any receiver; linkage risk sat in billing | Billing linkage rules (L-BL-04); contract stays receiver-agnostic by design | billing §6 |
