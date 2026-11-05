# Audit — webhooks

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
| G7 Failure | fixed: L-WH-01 |
| G8 Security | pass |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | fixed: L-WH-02 |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-WH-01 | G7 | Low | Per-endpoint concurrency relied on Valkey with no fallback | 1 in flight per endpoint per replica when Valkey is down | §6 |
| L-WH-02 | G11 | Medium | Local demo merchant runs as a Compose service (`merchant-demo`), not on localhost — SSRF rules would block the Stage 1 demo | Local-only allowance for `merchant-demo` | §8 |
