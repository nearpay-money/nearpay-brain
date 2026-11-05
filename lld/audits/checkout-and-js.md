# Audit — checkout and js

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | pass |
| G3 State | pass |
| G4 API | fixed: L-CK-01 |
| G5 Events | pass |
| G6 Concurrency | pass |
| G7 Failure | fixed: L-CK-02 |
| G8 Security | fixed: L-CK-03 |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-CK-01 | G4 | Medium | SSE via `EventSource` cannot send the client-secret header | fetch-based SSE client | §3 |
| L-CK-02 | G7 | Medium | Sentinel now requires a payments-issued ticket | Ticket forwarded with the attestation request | §5 |
| L-CK-03 | G8 | Low | Merchant name could have been taken from the (untrusted) payments session | Only the attested merchant name and fingerprint are rendered (already §5.2; re-verified) | §5 |
