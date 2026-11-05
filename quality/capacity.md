# Capacity Model

> Version 1.0 · Numbers are planning estimates; re-baseline with load tests in `nearpay-testbed/load` before each phase exit.

## 1. Load points

| Point | Definition | Value |
|---|---|---|
| **P1 — year-one plan** | 1 M payments/month, peak factor 20× | avg 0.39 /s, **peak ≈ 8 payments/s** |
| **D — design point** (NFR-SCL-01) | burst capacity of v1 hardware | **100 payment creations/s**, 1,000 API req/s |
| **A — agent target** | intents-transfer settlements | 500 /s (provisional, SP-07) |

## 2. Per-payment work (cross-chain checkout)

| Resource | Per payment | At P1 peak (8/s) | At D (100/s) |
|---|---|---|---|
| Public API requests (session, options, attempt, SSE, tx submit) | ≈ 6 | 48 rps | 600 rps |
| Attempts | 1.3 | 10 /s | 130 /s |
| 1Click quotes (live) | 1.3 | 10 rps | 130 rps |
| 1Click status calls (deposit-signal polling) | ≈ 10 | 80 rps | 1,000 rps ⚠ SP-04 |
| Explorer calls | fixed 0.2 rps | 0.2 rps | 0.2 rps |
| Postgres row writes (all services) | ≈ 40 | 320 /s | 4,000 /s |
| Domain events | ≈ 6 | 48 /s | 600 /s |
| Webhook deliveries (1 endpoint avg) | ≈ 4 | 32 /s | 400 /s |
| Sentinel co-signatures (1 status call each) | 1.3 | 10 /s | 130 /s |

**Interpretation**
- Postgres at 4k writes/s on NVMe with sync replication is comfortably within a single primary's capability (tens of thousands of small writes/s); not the bottleneck.
- At D, 1Click status volume (≈ 1,000 rps) is the binding constraint. Because success on Tier-A destinations comes from destination watchers (F-24), throttling status polling degrades **progress UX and refund detection latency**, not success detection. Policy when the status budget saturates: drop to watcher-only for success, poll only executions past their expected settle time, and rely on Explorer for refunds.
- At P1 the whole platform runs at < 10% of v1 hardware.

## 3. Agent payments

| Item | Estimate |
|---|---|
| Verify cost | 1 signature check + 1 `simulate_intents` RPC (≈ 30–80 ms) |
| Batching | every ≤ 150 ms, gas-bounded (≤ 80% max prepaid gas per tx) |
| Keys | N function-call keys; each key ≈ 1 tx per block (600 ms) in flight → N ≥ ceil(tx/s × 0.6 × 2) |
| At A = 500/s with k intents per tx | tx/s = 500 / k; e.g. k = 10 → 50 tx/s → N ≥ 60 keys |
| Shard bound | Σ gas/s on `intents.near` shard shared with all NEAR Intents users — **unknown until SP-07** |

## 4. Storage growth

| Data | Size per payment | Retention | Growth at P1 |
|---|---|---|---|
| Core rows (intents, attempts, transitions, statements, events) | ≈ 10 KB | 5 years (AML policy default) | ≈ 10 GB/month |
| Provider observations (raw JSON) | ≈ 20 KB | 90 days hot, then Parquet in S3 with hash in DB (F-27) | ≈ 20 GB rolling |
| Webhook deliveries | ≈ 4 KB | 30 days | ≈ 4 GB rolling |
| Logs/traces (AMS) | — | 14 d logs, 7 d traces, 13 mo metrics | sized on AMS host / object storage |

Per node: 512 GB NVMe → Postgres volume budget 300 GB → ≈ 2 years at P1 before adding netcup local block storage (up to 8 TB) or partitioning cold years to S3.

## 5. Hardware allocation (per NUE node, RS 4000: 12 cores / 32 GB)

| Workload | CPU req | RAM req |
|---|---|---|
| Postgres instance (1 per node: primary / sync / async) | 4 | 12 GB |
| NATS + Valkey | 1 | 2 GB |
| Application pods (≈ 2 replicas × 10 services spread over 3 nodes) | 4 | 10 GB |
| k3s, Cilium, Traefik, Argo, Kyverno, OpenBao, agents | 2 | 5 GB |
| Headroom | 1 | 3 GB |

Scale-out trigger: sustained > 60% CPU on application pods or p99 latency SLO burn → add a 4th node per site (k3s agent) before vertical changes.
