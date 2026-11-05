# Audit — platform

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | pass |
| G3 State | pass |
| G4 API | fixed: L-PL-02 |
| G5 Events | pass |
| G6 Concurrency | pass |
| G7 Failure | fixed: L-PL-01 |
| G8 Security | pass |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | fixed: L-PL-03, L-PL-04 |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-PL-01 | G7 | Medium | Compose table lacked the new `notify-api` service | Added | §1 |
| L-PL-02 | G4 | High | Overlapping path prefixes (`/v1/account` vs `/v1/account/kyb`, `/v1/checkout` vs `…/agent_pay`) had no priority rule | Explicit router priorities | §1 |
| L-PL-03 | G11 | Medium | Demo merchant's network reachability for webhooks undefined | Local-only egress allowance | §1 |
| L-PL-04 | G11 | High | Checkout pins the sentinel public key at build time but keys were generated after build | Defined start order: keys → static builds → compose | §1 |
