# ADR-0015: v1 runs every service as its own container in one Docker Compose stack on a single host

- **Status:** Accepted · **Date:** 2026-11-05 · **Supersedes:** ADR-0011 *for v1 only* (ADR-0011 remains the post-v1 target topology) · **Invariants:** INV-1, INV-4, INV-8, INV-9

## Context
Owner decisions (2026-11-05): keep separate repos and separate services; run all services inside one `docker-compose`; develop and demo on the local machine first (against NEAR mainnet), and move to a single netcup server only in the last stage, after v1 is complete. NEAR Intents has no testnet (hackathon FAQ Q-007), so "local" still means real mainnet rails for live mode.

## Decision
1. **Runtime unit = container.** Each of the 10 services (identity, payments, billing, rails, agentpay, compliance, webhooks, notify, sandbox, sentinel) is its own image and Compose service, with its own Postgres database + role, its own env/secret files and its own healthcheck. Process types (api, workers) are separate Compose services built from the same image with a different command.
2. **Platform in the same Compose stack:** Traefik (gateway + ForwardAuth, ADR-0014), PostgreSQL 17 (one instance, database per service), NATS JetStream (single node, file storage), Valkey, an SSRF-safe egress proxy, and an observability profile (OpenTelemetry Collector, VictoriaMetrics, Loki, Grafana).
3. **Secrets in v1:** SOPS + age-encrypted env files in `nearpay-deploy`, decrypted at deploy time into Docker secrets (files mounted at `/run/secrets/*`, never env vars for keys). OpenBao is deferred to the post-v1 topology; the `secrets` port in `@nearpay-internal/service` reads files, so swapping to OpenBao later is a config change.
4. **Hosts:** `local` profile on the developer machine (`*.nearpay.localhost` hostnames, browser-resolvable without DNS); `server` profile on one netcup server behind Cloudflare in the final stage (root server recommended over VPS for dedicated cores and the 99.9% guarantee).
5. **Data safety on one host:** WAL-G continuous WAL archiving + nightly base backups to off-host EU S3 (RPO ≤ 60 s), weekly automated restore test; NATS remains rebuildable from outboxes (ADR-0004).
6. **Separation that survives one host:** sentinel runs with its own DB role (read-only on identity's registry), its own secret files and its own image pipeline; checkout stays a static build. Same-host isolation is weaker than separate machines — accepted for v1 (R-18).
7. **Compose ownership:** `nearpay-deploy/compose/` holds the stack, profiles and env templates; the entry repo `nearpay` (ADR-0016) clones siblings and runs it.

## Consequences
- ➕ One command brings up the whole system locally; identical topology later on the server; no Kubernetes in the critical path to v1.
- ➖ v1 availability is bounded by one host: targets drop to 99.5% (API/checkout) until the ADR-0011 topology is adopted; host loss → restore on a new host (RTO ≤ 4 h).
- ➖ Same-host blast radius for secrets and DB; mitigated by per-service DB roles, per-service secret files, read-only root filesystems, non-root users and no Docker socket access for app containers.
- Money safety is unchanged: NearPay is never in the funds path, so host loss delays information, never funds.

## Alternatives considered
- **k3s from day one (ADR-0011):** stronger HA, slower to v1; kept as post-v1 target.
- **Modular monolith:** rejected by owner; separate services retained.

## Revisit when
v1 is live and either paid merchants require > 99.5%, or sustained load exceeds one host's headroom.
