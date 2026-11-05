# Audit — billing

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | pass |
| G3 State | fixed: L-BL-01 |
| G4 API | pass |
| G5 Events | fixed: L-BL-02 |
| G6 Concurrency | pass |
| G7 Failure | fixed: L-BL-03 |
| G8 Security | fixed: L-BL-04 |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-BL-01 | G3 | High | Hosted invoice links used the session client secret, which changes when a session expires — emailed links would break | Long-lived `invoice_secret`; public view lazily recreates the payments session | §3, §5 |
| L-BL-02 | G5 | Medium | Customer emails are owned by payments | payments `GET /internal/customers/{id}` (L-PAY-09) | §1 |
| L-BL-03 | G7 | Medium | Keeper could not distinguish insufficient balance from other collect failures | Vault error strings defined (contracts L-VT-01) and mapped | §7 |
| L-BL-04 | G8 | High | Any payer can create a vault mandate to any receiver; billing would trust it blindly | Mandates link only when `ext_ref`, receiver, token, amount and period match the subscription | §6 |
