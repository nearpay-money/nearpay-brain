# Audit — sentinel

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | fixed: L-SN-04 |
| G3 State | pass |
| G4 API | pass |
| G5 Events | pass |
| G6 Concurrency | fixed: L-SN-02 |
| G7 Failure | fixed: L-SN-03 |
| G8 Security | fixed: L-SN-01 |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-SN-01 | G8 | Critical | Any request with a foreign or crafted deposit address failed a `security` check and engaged the global kill switch — a one-request DoS of all live checkouts | Kill switch only when rails confirms the address was issued by NearPay; spoofs are refused and rate-limited | §5 |
| L-SN-02 | G6 | High | Duplicate registry rows (unverified destinations) made recipient→merchant resolution undefined | `ambiguous` class (refuse, no kill switch); identity prevents most duplicates (L-ID-06) | §5 R5 |
| L-SN-03 | G7 | High | Anonymous endpoint could exhaust sentinel's provider budget, blocking checkout (fail-closed) | Payments-signed `attestation_ticket` gate before any provider call + per-IP limits | §5 |
| L-SN-04 | G2 | Low | Deadline equality compared strings with possibly different ISO formatting | Compare as instants | §5 R4 |
