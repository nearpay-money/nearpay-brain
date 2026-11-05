# Audit — rails

> Audited 2026-11-05 against `lld/README.md` gates G1–G12. Result: **pass after fixes** — every finding below is fixed in the LLD (version bumped to 1.1). Re-audit after any change to the component.

## Gate results

| Gate | Result |
|---|---|
| G1 Requirements | fixed: L-RL-02 |
| G2 Data | pass |
| G3 State | pass |
| G4 API | fixed: L-RL-05, L-RL-08 |
| G5 Events | fixed: L-RL-01 |
| G6 Concurrency | pass |
| G7 Failure | fixed: L-RL-06 |
| G8 Security | pass |
| G9 Observability | pass |
| G10 Testability | fixed: L-RL-04 |
| G11 Stage fit | fixed: L-RL-03 |
| G12 Decision-free | fixed: L-RL-07 |

## Findings

| ID | Gate | Severity | Finding | Fix applied | Where |
|---|---|---|---|---|---|
| L-RL-01 | G5 | Medium | Agentpay's cross-chain executions (Stage 3) would be consumed by payments, which has no attempt for them | `owner` column; consumers filter by owner | §4 |
| L-RL-02 | G1 | Critical | Sentinel's kill switch had no enforcement point in rails | `control_flags.quoting_enabled` set from sentinel events and checked before every live quote | §4, §7, §8.1 |
| L-RL-03 | G11 | Medium | Stage 1 has no Explorer token (owner provides credentials later), but the feed was mandatory | `NP_EXPLORER_ENABLED`; hot poller + watchers cover Stage 1 | §11 |
| L-RL-04 | G10 | High | Test-mode `intents_transfer` executions had no evidence source (no NEAR indexer for the sandbox) | Sandbox feed loop reusing the indexer matching code | §9 |
| L-RL-05 | G4 | Medium | Options could not advertise the Intents-balance path | `method` in options; `destination_type` input | §6 |
| L-RL-06 | G7 | High | Stage 1 has no secondary price source; dual-source rule would block everything or be silently skipped | Explicit Stage 1 posture: stable settlement only, `usd` pricing only | §11 |
| L-RL-07 | G12 | Medium | Tier-B limits were described but not valued | `NP_TIER_B_LIMITS` defaults | §11 |
| L-RL-08 | G4 | High | Agentpay needed decimals and fresh price to convert minimum fees; sentinel needed to tell genuine from spoofed deposit addresses | `GET /internal/assets/{id}`, `GET /internal/executions?deposit_address=` | §6 |
