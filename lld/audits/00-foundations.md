# Audit — LLD 00 Foundations

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | fixed: L-F-01 |
| G3 State | pass |
| G4 API | pass |
| G5 Events | fixed: L-F-04 |
| G6 Concurrency | fixed: L-F-05 |
| G7 Failure | fixed: L-F-06 |
| G8 Security | fixed: L-F-02, L-F-03 |
| G9 Observability | pass |
| G10 Testability | fixed: L-F-07 |
| G11 Stage fit | pass |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-F-01 | G2 | High | Asset amount column type unspecified; 24-decimal tokens overflow `bigint` | `numeric(78,0)` mandated for asset base units | §6 |
| L-F-02 | G8 | High | Event snapshots could carry PII (customer email) into NATS and webhook logs | No-PII rule for events; notify resolves recipients via owner APIs | §8 |
| L-F-03 | G8 | High | Idempotency replay stores full responses in plaintext, including `client_secret` | `response_body_enc` (envelope-encrypted) | §7 |
| L-F-04 | G5 | Medium | `evt.webhooks.*` had no stream | `EVT_WEBHOOKS` added | §8 |
| L-F-05 | G6 | Low | Global unique `dedupe_key` could collide across job kinds | Kind prefix mandatory | §7 |
| L-F-06 | G7 | Low | Time source for deadlines unstated | DB `now()` + NTP host requirement | §10 |
| L-F-07 | G10 | Medium | Cross-repo signature fixtures had no home (sentinel↔checkout, webhooks↔node, agentpay↔agent-signer) | `nearpay-spec/test-vectors/` | §3 |
