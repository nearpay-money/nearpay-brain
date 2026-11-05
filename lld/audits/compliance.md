# Audit — compliance

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | fixed: L-CMP-02 |
| G2 Data | pass |
| G3 State | pass |
| G4 API | pass |
| G5 Events | pass |
| G6 Concurrency | pass |
| G7 Failure | pass |
| G8 Security | pass |
| G9 Observability | pass |
| G10 Testability | fixed: L-CMP-01 |
| G11 Stage fit | fixed: L-CMP-03 |
| G12 Decision-free | fixed: L-CMP-04 |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-CMP-01 | G10 | Medium | Sandbox blocking rule (`dead0000` suffix) disagreed with api-conventions (`sanctioned` refund address) | Single rule: subject contains `sanctioned` | §4 |
| L-CMP-02 | G1 | Medium | Region-level blocking assumed a header Cloudflare only sends with a managed transform | Server stage enables `cf-region-code`; local has no geo headers | §1 |
| L-CMP-03 | G11 | Low | Post-screening of senders silently depended on Explorer (disabled in Stage 1) | Stage 1 scope stated explicitly | §12 |
| L-CMP-04 | G12 | Medium | OFAC source URL/format assumed | Spike SP-15 | §8 |
