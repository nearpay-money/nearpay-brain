# Audit — payments

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | fixed: L-PAY-01 |
| G2 Data | pass |
| G3 State | fixed: L-PAY-05 |
| G4 API | fixed: L-PAY-04, L-PAY-07 |
| G5 Events | fixed: L-PAY-03, L-PAY-09 |
| G6 Concurrency | pass |
| G7 Failure | fixed: L-PAY-08 |
| G8 Security | fixed: L-PAY-06 |
| G9 Observability | pass |
| G10 Testability | fixed: L-PAY-02 |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-PAY-01 | G1 | High | FR-CMP-03 geo policy was not enforced anywhere in the payer path | Geo check on attempt creation from gateway-forwarded country/region; `geo_blocked` | §8.2, §6.2 |
| L-PAY-02 | G10 | High | Test helpers referenced by api-conventions and checkout had no endpoint | `simulate_deposit` (payer, test only) and `/v1/test_helpers/payment_attempts/{pa}/advance` → sandbox | §6.2 |
| L-PAY-03 | G5 | Medium | Receipts needed customer email but events carry no PII | `GET /internal/payment_intents/{id}/receipt-context` | §6.3 |
| L-PAY-04 | G4 | High | `intents_transfer` attempts had no defined refund destination | `payer_wallet` (payer's Intents account) mandatory for that method; refunds go there | §6.2 |
| L-PAY-05 | G3 | High | Agent flow needed a binding before signing, but bindings are created with the execution; a read-only GET could not provide it | `POST /internal/checkout_sessions/{id}/agent_attempts` creates/reuses the attempt and returns the requirement; GET removed | §6.3, agentpay §5 |
| L-PAY-06 | G8 | Medium | Embedded sessions could be created without allowed origins (clickjacking risk) | `allowed_origins` required for `ui_mode=embedded` | §8.1 |
| L-PAY-07 | G4 | Medium | SSE with client-secret header is impossible with `EventSource` | Documented fetch-based SSE client; no secrets in URLs | §6.2, checkout §5 |
| L-PAY-08 | G7 | High | Sentinel's anonymous endpoint could be flooded, exhausting its 1Click status budget and blocking all checkouts (fail-closed) | Payments issues an Ed25519 `attestation_ticket` per attempt; sentinel requires it (correctness still independent of payments) | §8.2, sentinel §5 |
| L-PAY-09 | G5 | Medium | Billing needed customer emails owned by payments | `GET /internal/customers/{id}` | §6.3 |
