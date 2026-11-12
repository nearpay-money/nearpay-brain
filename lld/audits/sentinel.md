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
| L-SN-01 (amended 2026-11-12, ADR-0028) | G8 | Critical | Residual: a single `security` refusal on a genuine address still stopped every merchant (beta incident 2026-11-11 18:58 UTC); caller-supplied R4 fields (amount, origin asset, chain, deadline) let a payer with a ticket for their own attempt trigger it in one request | A refusal never stops the platform: pipeline evidence quarantines the account(s) involved (30 min TTL); the global switch engages only on ≥ 3 pipeline breaches across ≥ 2 accounts within 15 min; request evidence is refused, recorded and rate-limited only; R5 re-reads the registry; configuration and provider-shape errors are `unavailable` | §5 "Containment" |
| L-SN-02 | G6 | High | Duplicate registry rows (unverified destinations) made recipient→merchant resolution undefined | `ambiguous` class (refuse, no kill switch); identity prevents most duplicates (L-ID-06) | §5 R5 |
| L-SN-03 | G7 | High | Anonymous endpoint could exhaust sentinel's provider budget, blocking checkout (fail-closed) | Payments-signed `attestation_ticket` gate before any provider call + per-IP limits | §5 |
| L-SN-04 | G2 | Low | Deadline equality compared strings with possibly different ISO formatting | Compare as instants | §5 R4 |
