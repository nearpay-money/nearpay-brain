# ADR-0011: Self-hosted on netcup root servers with k3s; NUE primary, VIE standby, AMS witness

- **Status:** Accepted as the **post-v1 target topology**; v1 is superseded by [ADR-0015](ADR-0015-v1-single-host-compose.md) · **Date:** 2026-11-05 · **Invariants:** INV-4, INV-9

## Context
Owner preference: remote machines, mostly netcup. netcup facts (Nov 2026): root servers have dedicated cores and a 99.9% availability guarantee, VPS do not; Cloud VLAN is same-location only; DDoS filtering to L4; no floating IPs or S3 object storage found; sites NUE, VIE, AMS, MNZ, SIN.

## Decision
- Production on root servers: 3 × RS 4000 G12.5 in NUE (primary) and 3 in VIE (warm standby); 1 VPS in AMS as OpenBao Raft witness + observability.
- k3s HA per site; Cilium; Traefik on hostNetwork behind Cloudflare LB (no floating IP needed); CloudNativePG (sync replica in-site, replica cluster cross-site); NATS JetStream; Valkey; OpenBao (5 voters across 3 sites); Argo CD + Rollouts; Kyverno; Zot cache.
- Cloudflare for DNS, WAF, L7 DDoS, Turnstile, LB with deep health checks, Access; WireGuard between sites.
- Off-provider EU S3 for WAL/backups/audit; IaC sufficient to rebuild on another provider within 8 h.

## Consequences
- ➕ Low cost, full control, EU residency, no hyperscaler lock-in.
- ➖ We operate databases, secrets and Kubernetes ourselves → strong automation, runbooks and restore drills are mandatory.
- ➖ Single edge provider (Cloudflare) and single hosting provider are accepted risks (R-05, R-06).

## Alternatives considered
- **Docker Compose on VMs:** simpler, but weak failover, no policy enforcement, harder GitOps.
- **Nomad + Consul:** viable; smaller ecosystem for Postgres operators and policy tooling.
- **Hyperscaler managed services:** contradicts owner preference.

## Revisit when
Sustained load needs more than 6 production nodes per site, or a region outside the EU is needed (MNZ/SIN).
