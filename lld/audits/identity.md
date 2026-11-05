# Audit — identity

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | pass |
| G2 Data | fixed: L-ID-01 |
| G3 State | pass |
| G4 API | fixed: L-ID-02 |
| G5 Events | fixed: L-ID-03 |
| G6 Concurrency | fixed: L-ID-06 |
| G7 Failure | pass |
| G8 Security | fixed: L-ID-04, L-ID-07 |
| G9 Observability | pass |
| G10 Testability | pass |
| G11 Stage fit | fixed: L-ID-05 |
| G12 Decision-free | pass |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-ID-01 | G2 | Medium | Fee-schedule route classes (`stable_same/stable_cross`) disagreed with rails classification | Classes unified: `same_asset`, `stable_pair`, `volatile`, `intents_transfer` (ADR-0006 wording aligned, Q-BIZ-02 defaults updated) | §4 fee_schedules |
| L-ID-02 | G4 | High | Session principals had no way to choose test vs live mode | `NearPay-Mode` header → `lm` claim; live unsafe calls require `live_enabled` | §8.1 |
| L-ID-03 | G5 | Medium | Notify needed owner/developer emails but no API existed (events carry no PII) | `GET /internal/accounts/{id}/notification-recipients` | §6 |
| L-ID-04 | G8 | High | `ptype=ops` existed in foundations but identity had no operator model | `ops_operators` table, ops cookie/host, hardware-key AAGUID allowlist | §4, §8.1 |
| L-ID-05 | G11 | Medium | Stage 1 live mode depended on KYB vendor not built until Stage 2 | Stage 1 uses compliance manual KYB decision via ops CLI | §16 |
| L-ID-06 | G6 | Critical | Unverified-destination path let an attacker squat a victim's address (uniqueness) or create duplicate rows that sentinel could not disambiguate | Unique index limited to signature-verified rows; unverified creation blocked if any other account has the same live row (advisory lock); sentinel `ambiguous` class never trips the kill switch | §4, §5, sentinel §5 |
| L-ID-07 | G8 | Medium | Per-IP rate limits trusted an unspecified client-IP header | `CF-Connecting-IP` on server (Cloudflare-only origin), first XFF hop locally | §8.1 |
