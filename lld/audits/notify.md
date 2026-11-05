# Audit — notify

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | pass |
| G3 State | pass |
| G4 API | pass |
| G5 Events | fixed: L-NT-02 |
| G6 Concurrency | pass |
| G7 Failure | fixed: L-NT-01 |
| G8 Security | pass |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-NT-01 | G7 | High | The internal send API ran on a single worker replica, making login (magic links) depend on one container | Separate `notify-api` (2 replicas) and `notify-worker` | §2 |
| L-NT-02 | G5 | Low | Recipient resolution APIs did not exist in owner services | Added to identity (L-ID-03), payments (L-PAY-03), billing (`notification-context`) | §4 |
