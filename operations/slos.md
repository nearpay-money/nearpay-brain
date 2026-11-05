# Service Level Objectives

> Version 1.0 · Window: rolling 28 days unless stated. "NearPay-attributable" excludes requests failed with `rail_unavailable` caused by a documented upstream outage (those are tracked separately and published on the status page).

> **v1 (single host, ADR-0015):** availability SLOs are **99.5%**; all latency, correctness, reconciliation and freshness SLOs below apply unchanged.

| SLO | SLI (good / valid) | Target (post-v1) | Page when |
|---|---|---|---|
| **API availability** | non-5xx responses / all requests (excl. upstream-attributed 503) | 99.95% (v1: 99.5%) | 2% budget burn in 1 h, or 5% in 6 h |
| **Checkout availability** | synthetic checkout journeys passing / all (3 regions, every minute) | 99.95% (v1: 99.5%) | 3 consecutive failures in 2 regions |
| **API latency** | create-session requests < 300 ms / all | 99% | p99 > 300 ms for 15 min |
| **Live quote latency** | quotes < 4 s / all | 95% | p95 > 4 s for 15 min |
| **Sentinel co-sign** | co-signatures < 1.5 s and non-error / all | 99.9% | error rate > 1% for 5 min |
| **Event freshness** | state change → webhook first attempt < 60 s / all | 99.9% | p99 > 60 s for 10 min |
| **Settlement detection** | provider SUCCESS (or watcher match) → `payment_intent.succeeded` < 120 s on Tier A / all | 99% | p95 > 120 s for 15 min |
| **Correctness** | succeeded intents with verified evidence / all succeeded | **100%** | any violation (sev-1) |
| **Reconciliation** | merchants with zero unexplained diffs / all merchants (daily) | 100% | any diff (sev-2) |
| **Verifier holds** | executions not in `held` / all terminal-bound | ≥ 99.5% | hold rate > 1% in 15 min |
| **Dashboard availability** | non-5xx / all | 99.9% | 5% burn in 6 h |

**Rail SLIs (reported, not owned):** 1Click quote success rate, quote latency, status error rate, time-to-settle by route, provider refund rate — shown on the status page as "Upstream: NEAR Intents".

**Error budget policy:** when an SLO's budget is exhausted, feature deploys to the owning service pause (only reliability and security fixes ship) until the budget recovers.
