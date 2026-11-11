# Infrastructure, Deployment and Operations Architecture

> Version 1.2 (static sites served by nginx behind Traefik) · **v1 runtime: §0 (ADR-0015).** §1–§7 describe the post-v1 target topology (ADR-0011). Everything here is code in `nearpay-infra` (hosts, edge) and `nearpay-deploy` (workloads).

## 0. v1 topology — one Docker Compose stack, one host (ADR-0015)

```mermaid
flowchart LR
  subgraph Host["Host: developer machine (Stages 1..N-1) / netcup server (final stage)"]
    subgraph edge["network: edge"]
      TR[traefik<br/>:443 / :80]
    end
    subgraph internal["network: internal"]
      SVC[identity, payments, billing, rails,<br/>agentpay, compliance, webhooks, notify,<br/>sandbox, sentinel - api + worker containers]
    end
    subgraph data["network: data"]
      PG[(postgres 17<br/>db per service)]
      NA[(nats jetstream)]
      VA[(valkey)]
    end
    subgraph egress["network: egress"]
      EP[egress-proxy]
    end
    ST[static: checkout, js, dashboard, console<br/>nginx behind traefik]
  end
  BR[Browser / merchant / agent] --> TR
  TR --> SVC
  TR --> ST
  SVC --> PG & NA & VA
  SVC --> EP
  EP --> NET[(1Click, NEAR RPC, chain RPCs,<br/>vendors, merchant webhooks)]
  PG -. WAL-G .-> S3[(off-host EU S3)]
```

| Item | v1 choice |
|---|---|
| Stack location | `nearpay-deploy/compose/` (`compose.yaml` + `compose.local.yaml` + `compose.server.yaml` + `compose.observability.yaml`) |
| Images | Built from sibling repos (`build.context: ../../nearpay-<svc>`) locally; GHCR digests on the server |
| Hostnames (local) | `api.nearpay.localhost`, `pay.nearpay.localhost`, `js.nearpay.localhost`, `dashboard.nearpay.localhost`, `ops.nearpay.localhost`, `merchant-demo.nearpay.localhost` (browsers resolve `*.localhost` to 127.0.0.1); local TLS via Traefik default cert or `mkcert` |
| Hostnames (server) | `*.nearpay.money` via Cloudflare proxy, Authenticated Origin Pulls, origin firewall to Cloudflare ranges |
| Postgres | One container; `init/` creates a database and a role per service (+ `sentinel_ro` with SELECT on `identity.settlement_destinations` view only); WAL-G continuous archiving + nightly base backup to off-host S3; weekly restore test in `nearpay-testbed` |
| NATS | Single node JetStream, file store volume; streams per domain, 7-day retention |
| Secrets | `nearpay-deploy/secrets/<env>/<service>.enc.env` (SOPS + age) → `scripts/deploy.sh` decrypts to Docker secrets; keys never in env vars |
| Egress | Only `egress-proxy` and the services that need the internet are attached to `egress`; webhook delivery always via the proxy (SSRF rules) |
| Container hardening | non-root user, read-only root FS, `no-new-privileges`, dropped capabilities, memory/CPU limits, healthchecks; no container mounts the Docker socket |
| Observability | `observability` profile: OpenTelemetry Collector → VictoriaMetrics, Loki, Tempo, Grafana (same host in v1; external uptime check on the server stage) |
| Server sizing (final stage) | netcup root server RS 8000 G12.5 (16 dedicated cores, 64 GB, 1 TB NVMe) recommended; RS 4000 is the floor |

## 1. Sites and failure domains (post-v1 target)

```mermaid
flowchart LR
  subgraph CF["Cloudflare: DNS, WAF, Turnstile, CDN, Load Balancer, Access"]
    LB{LB pools<br/>NUE primary, VIE fallback}
  end
  subgraph NUE["NUE Nuremberg - PRIMARY"]
    N1[RS 4000 #1] --- N2[RS 4000 #2] --- N3[RS 4000 #3]
  end
  subgraph VIE["VIE Vienna - WARM STANDBY"]
    V1[RS 4000 #1] --- V2[RS 4000 #2] --- V3[RS 4000 #3]
  end
  subgraph AMS["AMS Amsterdam - WITNESS + OBSERVABILITY"]
    A1[VPS 2000]
  end
  S3[(Off-provider EU S3<br/>backups, WAL, audit)]
  LB --> NUE
  LB -. after promotion .-> VIE
  NUE <-. WireGuard: Postgres replica, OpenBao raft .-> VIE
  NUE <-. WireGuard .-> AMS
  VIE <-. WireGuard .-> AMS
  NUE --> S3
  VIE --> S3
```

| Site | Hosts | Role |
|---|---|---|
| **NUE** | 3 × Root Server RS 4000 G12.5 (12 dedicated EPYC cores, 32 GB, 512 GB NVMe) | k3s HA control plane + all workloads; Postgres primary + sync replica + async replica; NATS ×3; OpenBao ×2 |
| **VIE** | 3 × RS 4000 G12.5 | k3s HA; CNPG **replica cluster** (streaming from NUE); services deployed at min replicas in standby mode; NATS ×3 (idle); OpenBao ×2 |
| **AMS** | 1 × VPS 2000 G12 | OpenBao witness (5th Raft voter), observability stack, external synthetic probes, WireGuard hub for admin access |
| **Staging** | 1 × VPS 4000 G12 (NUE) | Single-node k3s; sandbox rail + tiny-mainnet canaries |

Rationale: root servers for anything stateful or latency-critical (dedicated cores, 99.9% guarantee); VPS only where an outage is tolerable. Two EU sites ~400 km apart on the same backbone survive a DC loss; the off-provider bucket + IaC survive a netcup-wide event (RTO ≤ 8 h by rebuilding on another provider).

## 2. Network

- **Public:** each node has a public IP; nftables default-deny; `443/tcp` accepted only from Cloudflare ranges with **Authenticated Origin Pulls** (mTLS); WireGuard UDP only from peer IPs; **no public SSH** (admin via WireGuard hub in AMS + hardware-key SSH certificates).
- **Private in-site:** netcup Cloud VLAN (same-location only); host firewall rules apply because the netcup firewall cannot filter VLAN traffic.
- **Cross-site:** WireGuard full mesh NUE–VIE–AMS (Postgres streaming, OpenBao Raft, metrics/logs shipping).
- **Pod network:** Cilium (eBPF) with transparent WireGuard encryption and default-deny `CiliumNetworkPolicy` per service (only edges declared in HLD §5 are allowed).
- **Egress:** webhooks via a dedicated egress-proxy deployment pinned to nodes whose IPs are published — **both NUE and VIE IPs are published** so merchant allow-lists survive failover; other egress (1Click, RPCs, vendors) through an allow-listed egress gateway.

## 3. Platform components (per site)

| Component | Choice | Notes |
|---|---|---|
| Orchestrator | **k3s** (embedded etcd, 3 servers) | Lightweight, single binary, fits root servers; no nested virtualisation needed |
| Ingress | **Traefik** (hostNetwork on all 3 nodes) | Cloudflare LB targets all node IPs → no floating IP required |
| Postgres | **CloudNativePG** | 1 primary + 1 sync + 1 async per site; `synchronous_commit = remote_apply` in-site; VIE as replica cluster; barman-cloud WAL archive to S3 |
| Events | **NATS JetStream** | 3 nodes, R3 streams, 7-day retention |
| Cache | **Valkey** | Primary + replica + Sentinel; ephemeral |
| Secrets | **OpenBao** | 5-voter Raft across NUE(2)/VIE(2)/AMS(1); Shamir 3-of-5 unseal; KV for service secrets and signing keys (loaded at pod start-up), transit only for envelope-encryption KEKs — never on the request hot path |
| GitOps | **Argo CD** + **Argo Rollouts** | Canary 10% → 50% → 100% gated on SLIs |
| Registry cache | **Zot** pull-through cache | Survives GHCR outages; images verified by cosign policy (Kyverno) |
| Policy | **Kyverno** | Signed images only, no privileged pods, resource limits required |
| Time | chrony (NTS) | Deadlines and signatures depend on time |

## 4. Data protection and DR

| Mechanism | Target |
|---|---|
| In-site sync replica (CNPG, quorum commit) | RPO 0 for single-node loss; automatic failover ≈ 30 s |
| VIE replica cluster (async over WireGuard) | RPO ≤ 60 s for site loss; manual promotion (runbook RB-02) RTO ≤ 30 min |
| WAL archive + daily base backup to off-provider S3 (35 days, encrypted, object-lock on audit bucket) | PITR anywhere in 35 days; provider-loss RTO ≤ 8 h |
| OpenBao Raft snapshots (hourly) → S3 | Secrets recoverable without quorum |
| **Weekly automated restore drill** into staging + ledger checksum comparison | Backups proven, not assumed |

**Failover safety:** each site's deep health check (`/healthz/deep`) is green only if *its* Postgres is writable. Thresholds are set so an in-site CNPG failover (≈ 30 s) never flips the LB: probe every 15 s, unhealthy after 4 consecutive failures (60 s); NUE is also the LB fallback pool, so if both pools look unhealthy traffic still goes to NUE. The Cloudflare LB therefore sends traffic to VIE automatically **only after** VIE has been promoted, and stops sending to NUE if NUE is demoted. Promotion runbook fences NUE first (stop CNPG primary, revoke NUE WireGuard peer for Postgres) to prevent split-brain.

**Why NearPay outages are safe for money:** in-flight swaps are executed by NEAR Intents, not NearPay. On recovery, trackers resume from the persisted Explorer cursor minus a 30-minute overlap and re-poll every non-terminal execution; the outbox republishes anything not yet acknowledged; consumers dedupe by event ID.

## 5. External dependencies and redundancy

| Dependency | Redundancy |
|---|---|
| NEAR block data (indexer) | Two independent finalized-block sources (FastNEAR `neardata` stream and NEAR Lake), de-duplicated by block hash; RPC catch-up as a third path |
| NEAR RPC | Pool of ≥ 3 providers with health scoring and hedged reads; self-hosted RPC node on a dedicated root server when traffic justifies (SP-13) |
| Chain RPCs (verifiers) | ≥ 2 providers per Tier-A chain family; results must agree for amounts ≥ USD 10k |
| 1Click / Explorer | Single provider (R-01); circuit breakers; direct-intents path for Intents-native payments |
| Email | Primary + fallback transactional provider |
| Screening / KYB vendors | None in v1 (ADR-0020: local OFAC list + blocklist, self-declared KYB); later single each; fail policy per §13 of HLD |
| Cloudflare | Single edge (accepted R-05); status page hosted outside Cloudflare and netcup |
| GitHub / GHCR | Zot cache; deploys pause, runtime unaffected |

## 6. Environments and delivery

| Env | Rails | Data | Deploy trigger |
|---|---|---|---|
| `local` | sandbox | ephemeral (testbed compose) | developer / agent |
| `staging` | sandbox + mainnet canaries (≤ USD 5) | synthetic | merge to `main` (auto) |
| `production` | 1Click mainnet (live) + sandbox (test mode) | real | PR to `nearpay-deploy` approved by a human |

**Schema changes** follow expand → migrate → contract across at least two releases; migrations run as a pre-sync Argo hook with a separate DB role and must be backward compatible with the previous release (canaries run old and new code side by side).

Pipeline per service repo: lint → typecheck → unit → property tests on state machines → contract tests vs `nearpay-spec` → build image → SBOM (syft) → scan (grype, fail on critical) → cosign sign (keyless, GitHub OIDC) → push GHCR → bot PR to `nearpay-deploy` (staging overlay) → testbed E2E on staging → promotion PR (production overlay) → Argo Rollouts canary.

## 7. Observability

- **Instrumentation:** OpenTelemetry SDK in every service (`@nearpay-internal/service` sets it up); trace context propagated through HTTP, NATS headers and outbox rows.
- **Backends (AMS):** VictoriaMetrics, Loki, Tempo, Grafana, Alertmanager. Placed outside NUE/VIE so they observe failures of either. A dead-man's-switch heartbeat to an external SaaS alerts if AMS itself goes dark.
- **Paging:** Alertmanager → external on-call SaaS (SMS/push), with severity routing.
- **Synthetic:** checkout journey every minute from 3 regions (external), API probes, mainnet canary payments (funds circulate between NearPay-owned canary wallets; cost = route fees, hard daily cap) every 15 min across: NEAR USDC → Base USDC, Base USDC → NEAR Intents, Solana USDC → Arbitrum USDT, BTC → USDC (hourly, cost-bounded).
- **Status page:** `status.nearpay.money` on an external provider; components map 1:1 to SLOs plus "Upstream: NEAR Intents".
