# LLD — Platform (`nearpay-deploy`, `nearpay-infra`, `nearpay-testbed`, `nearpay` entry)

> Version 1.2 (step 1.1 implemented; changes in §1.1) · Version 1.1 audited (`audits/platform.md`) · ADR-0015, ADR-0016 · Infrastructure §0 · Foundations `00-foundations.md`

## 1. `nearpay-deploy/compose` — the v1 stack

```
compose/
├── compose.yaml                 # all services, networks, volumes, secrets (profile-independent)
├── compose.local.yaml           # ports on 127.0.0.1, *.nearpay.localhost, registry, mailpit, merchant-demo, demo network
├── compose.server.yaml          # (Stage 5) GHCR image digests, Cloudflare origin certs, WAL-G sidecar, restart policies
├── compose.observability.yaml   # (Stage 5) otel-collector, victoriametrics, loki, tempo, grafana
├── init/postgres/roles.sh       # databases + roles per service, sentinel_ro grant; idempotent, run by postgres-init
├── traefik/traefik.yaml         # static config: entry points :80/:443, ping, file provider
├── traefik/dynamic/routes.yaml  # routers, ForwardAuth and other middlewares, TLS (Go-templated by NP_BASE_DOMAIN)
├── static/                      # nginx config for pay, js, dashboard, ops, docs (+ placeholder page)
├── egress-proxy/Dockerfile      # Smokescreen v0.1.0 built from source
├── registry/config.yaml         # Verdaccio (@nearpay/*, @nearpay-internal/*)
├── env/local.env.example        # non-secret config (copied to local.env on first up); secrets in ../.secrets
└── scripts/                     # (in ../scripts) up, down, seed (+ deploy, secrets-decrypt in Stage 5)
```

**Services (container names = Compose service names):**

| Service | Image / build | Networks | Depends on (healthy) |
|---|---|---|---|
| `traefik` | `traefik:v3.5.6` (non-root; binds :80/:443 via `net.ipv4.ip_unprivileged_port_start=0`) | edge, internal (+ demo locally) | — |
| `postgres` | `postgres:17.11` | data | — |
| `postgres-init` (one-shot) | `postgres:17.11` running `init/postgres/roles.sh` | data | postgres |
| `nats` | `nats:2.10.29-alpine` (`-js -sd /data -m 8222`) | data | — |
| `valkey` | `valkey/valkey:8.1.11-alpine` | data | — |
| `egress-proxy` | Smokescreen v0.1.0 (built from `egress-proxy/Dockerfile`) | internal, egress (+ demo locally) | — |
| `identity-api`, `identity-worker` | nearpay-identity | internal, data, egress (NEAR/EVM RPC for attestations §8.2, Turnstile) | postgres, nats, valkey |
| `payments-api`, `payments-worker` | nearpay-payments | internal, data, egress (Turnstile siteverify) | identity-api, rails-api |
| `rails-api`, `rails-tracker`, `rails-feeds` | nearpay-rails | internal, data, egress | postgres, nats, compliance-api |
| `agentpay-api`, `agentpay-batcher` | nearpay-agentpay | internal, data, egress | postgres, nats |
| `compliance-api`, `compliance-worker` | nearpay-compliance | internal, data, egress | postgres, nats |
| `webhooks-api`, `webhooks-ingest`, `webhooks-dispatcher` | nearpay-webhooks | internal, data (dispatcher → egress-proxy only) | postgres, nats |
| `notify-api`, `notify-worker` | nearpay-notify | internal, data, egress | postgres, nats |
| `billing-api`, `billing-worker` | nearpay-billing | internal, data, egress (RPC) | postgres, nats (Stage 3+) |
| `sandbox-api` | nearpay-sandbox | internal, data | postgres |
| `sentinel-api`, `sentinel-auditor` | nearpay-sentinel | internal, data, egress | postgres, nats |
| `migrate-<svc>` (one-shot) | each service image, `dist/main.js migrate` with the migrator URL | data | postgres-init (completed) |
| `static` | `nginxinc/nginx-unprivileged:1.29.8-alpine` behind Traefik; sites mounted read-only from the checkout, js, dashboard and console `dist/` and `nearpay-site/dist/docs` (placeholder page until built) | edge | — |
| `mailpit` (local) | `axllent/mailpit:v1.27.11` | internal | — |
| `registry` (local) | `verdaccio/verdaccio:6.10.6` | edge (+ host port `127.0.0.1:4873`) | — | local npm registry for `@nearpay/*` and `@nearpay-internal/*` packages so sibling repos consume **published artifacts** (ADR-0001) without a remote; GitHub Packages / npm replace it when remotes exist |
| `merchant-demo` (local) | nearpay-testbed `merchant-demo` (port 3000) | demo (static IP `NP_DEMO_IP`; the egress proxy allows exactly this private address, webhooks L-WH-02) | payments-api |

**Networks:** `edge` (traefik, static), `internal` (all services; no internet), `data` (postgres, nats, valkey + their clients), `egress` (only containers that call the internet), `demo` (local: merchant-demo, egress-proxy, traefik). Containers not on `egress` cannot reach the internet.

**Traefik routing (local hostnames; server uses `*.nearpay.money`):**

| Host / path | Target | Middleware |
|---|---|---|
| `api.…/v1/checkout`, `/v1/payment_intents`, `/v1/refunds`, `/v1/payment_links`, `/v1/customers`, `/v1/balance_transactions`, `/v1/test_helpers` | payments-api | authz |
| `api.…/v1/account`, `/v1/auth`, `/v1/api_keys`, `/v1/settlement_destinations` | identity-api | authz |
| `api.…/v1/account/kyb`, `/webhooks/kyb` (router priority above identity's `/v1/account`) | compliance-api | authz |
| `api.…/v1/products`, `/v1/prices`, `/v1/tax_rates`, `/v1/invoices`, `/v1/subscriptions`, `/v1/mandates` | billing-api | authz |
| `api.…/v1/webhook_endpoints`, `/v1/events`, `/v1/webhook_signing_keys` | webhooks-api | authz |
| `api.…/x402`, `/v1/agent_payments`, `/v1/checkout/sessions/{id}/agent_pay*` (priority over payments) | agentpay-api | authz |
| `api.…/v1/attestations` | sentinel-api | authz (anonymous allowed) + IP rate limit |
| `api.…/webhooks/email` (Stage 5) | notify-api | authz (anonymous allowed; provider signature verified by notify) |
| `api.…/v1/assets` | rails-api | authz (anonymous allowed) |
| `pay.…`, `dashboard.…` `/api/*` | the API routers above (prefix stripped before ForwardAuth): the checkout and dashboard call the API same-origin, so the API has no CORS surface; unrouted `/api/*` answers 404 | authz |
| `pay.…`, `js.…`, `dashboard.…`, `ops.…`, `docs.…` | static (nginx) | per-site headers: `frame-ancestors 'none'` (pay `/embed/*`: `https:` + local http), nosniff, referrer and permissions policies; js served with CORS + cross-origin CORP |
| `api.…/healthz` | Traefik `ping@internal` (`OK`) | — |
| `mail.…`, `merchant-demo.…` (local) | mailpit, merchant-demo | — |

ForwardAuth middleware: `http://identity-api:8080/internal/authz`, `authResponseHeaders: [X-NearPay-Token, X-Request-Id]`, `trustForwardHeader: false` — ForwardAuth derives `X-Forwarded-Method/Uri` from the request as routed (after `/api` stripping), so a client cannot influence the URI identity authorizes. A headers middleware removes any client-supplied `X-NearPay-Token` before ForwardAuth; only identity's response sets it. Every route matches `Path(p) || PathPrefix(p/)` (no accidental prefix matches); `/internal/*` is never routed. Router priorities are explicit so `/v1/checkout/sessions/{id}/agent_pay*` (200) → agentpay and `/v1/account/kyb` (200) → compliance win over the broader prefixes (100); static sites are 10. Verified for every Stage-1 public route on `api.…` and both `/api` hosts.

**Postgres init:** databases `identity, payments, billing, rails, agentpay, compliance, webhooks, notify, sandbox, sentinel`; roles `<db>_app` (DML), `<db>_migrator` (DDL); `sentinel_ro` with `CONNECT` on `identity` and `SELECT` on `identity.public.sentinel_destination_registry` only (granted by identity's migration through a `grant` step run by the migrator role). Implemented by `init/postgres/roles.sh`, run by the one-shot `postgres-init` on **every** `up` (idempotent; passwords re-synchronised from the URL secret files, the single source): the migrator owns its database (DDL, trusted extensions such as `pgcrypto`/`citext`); default privileges give the app role DML on tables and sequences the migrator creates; `CONNECT` is revoked from `PUBLIC` on every database (including `postgres`), so no role reaches another service's data. `migrate-<svc>` one-shots run `dist/main.js migrate` with the migrator URL after `postgres-init`; service processes start after their migrator completed.

**Secrets:** `nearpay-deploy/secrets/<env>/*.sops.env|*.sops.key` (age recipients = owner + server); `scripts/secrets-decrypt.sh <env>` writes `./.secrets/<name>` (git-ignored) mounted as Docker secrets. Local `env` ships **test-mode-only** secrets generated by `scripts/seed.sh` (dev keys), so the stack runs without any real credentials; live mode requires the owner to add 1Click API key, Explorer token, treasury/relayer/keeper accounts and RPC keys.

**Local secrets (`seed.sh keys`, test-only, never overwritten):** `postgres_password`; `db_url_<svc>` / `db_migrate_url_<svc>` (app / migrator URLs) and `db_registry_url_sentinel`; `jwt/current.pem`, `jwt/next.pem` (mounted at `/run/secrets/jwt/`), `sentinel_signing_key.pem` (mounted as `/run/secrets/sentinel-<kid>.pem`), `payments_ticket_key.pem`, `webhook_signing_key.pem`, `sandbox_signing_key.pem` (Ed25519 PKCS#8 PEM, generated with Node so macOS works); `service_client_<svc>` and identity's `service_clients.json` (`{"clients":[{"client_id","secret_sha256"}]}`); `kek_<svc>` (32 bytes base64, one per service for all envelope encryption: PII, idempotency bodies, endpoint secrets); `turnstile_secret` (Cloudflare's always-pass test secret; sitekey `1x00000000000000000000AA`); `registry_password`. Public material in `.secrets/public/`: `keys.env` (`NP_SENTINEL_KID`, `NP_JWT_CURRENT_KID`, `NP_PAYMENTS_TICKET_PUBKEY`, `NP_SANDBOX_SIGNING_PUBKEY`, `NP_WEBHOOK_SIGNING_KID` — public keys as raw 32-byte base64url, kids `local-<12 hex of sha256(pubkey)>`) and `sentinel_keys.json` (`[{kid, public_key}]` for `VITE_SENTINEL_KEYS`). TLS: `tls/{cert,key,ca}.pem` from mkcert when installed (trusted after `mkcert -install`), otherwise a self-signed local CA + leaf (browsers warn; passkeys need the trusted certificate).

**Build/start order (local, `scripts/up.sh local`):** `seed.sh keys` (secrets + TLS) → registry up → publish `nearpay-spec` then `nearpay-kit` when a package version in those repositories is missing (release scripts accept only a loopback registry and pack reproducibly, so lockfile integrity hashes stay valid when a fresh registry is re-seeded) → static builds of checkout/js/dashboard read the **public** sentinel key from `.secrets/public/` (pinned at build) → `docker build --network host --build-arg NP_REGISTRY_URL=http://127.0.0.1:4873 -t nearpay/<svc>:local ../nearpay-<svc>` for every service whose repo has a `Dockerfile` (Compose's bake builder refuses `network: host` without an interactive grant, so Compose never builds or pulls service images) → `docker compose up --wait` (postgres-init → migrate-<svc> → processes healthy) → `seed.sh data` (step 1.2+). Each service has a Compose profile named after it; `up.sh` enables it once `../nearpay-<svc>/Dockerfile` exists and lists the services not built yet. Requires Docker Compose ≥ 2.30, Node 22 + pnpm.

**Service image contract:** `ARG NP_REGISTRY_URL` used for `@nearpay/*` and `@nearpay-internal/*` installs; `node dist/main.js <process>` (Compose passes `command: [dist/main.js, <process>]`, so the image's entrypoint is `node`); `HEALTHCHECK` runs `dist/main.js healthcheck` (built into `@nearpay-internal/service`, GET `:8080/healthz`); non-root UID 10001, read-only root FS (Compose enforces `read_only`, `tmpfs /tmp`, `cap_drop: ALL`, `no-new-privileges`, 35 s stop grace).

**Scripts:** `up.sh [local] [--no-build]`, `down.sh [--volumes]` (stops every profile), `seed.sh keys [--renew-tls]` (local keys and TLS) and `seed.sh data` (create a demo merchant via the API in test mode, register `merchant-demo` webhook — step 1.2+), `deploy.sh server` (Stage 5: pull digests, decrypt secrets, `docker compose up -d`, run migrations, smoke test), `backup-verify.sh`.

### 1.1 Changes in version 1.2 (from the step 1.1 implementation)

- Static sites are served by nginx behind Traefik (Traefik's file provider does not serve files).
- The checkout joins the dashboard in calling the API same-origin under `/api` (no CORS anywhere).
- identity and payments need the `egress` network (RPC for attestations; Turnstile siteverify).
- ForwardAuth uses `trustForwardHeader: false`; client-supplied `X-NearPay-Token` is stripped.
- Postgres roles are applied by an idempotent `postgres-init` one-shot on every `up` (was a first-boot SQL file).
- Service images are built by `up.sh` (host network for the registry), not by Compose; the image contract above.
- `/healthz` on the API host is Traefik's ping (`OK`); service health is per container.

## 2. `nearpay-infra` (Stage 5)

Ansible: `base-hardening` (Debian 12, unattended upgrades, SSH keys only, nftables default-deny, fail2ban), `docker-host` (Docker CE, compose plugin, log rotation, WAL-G timer, node exporter), `wireguard` (admin access). Terraform: Cloudflare (DNS, proxy, origin certs, Authenticated Origin Pulls, WAF managed rules, Turnstile sitekeys, Access app for `ops.`), object storage bucket (WAL-G, backups, audit) at an EU provider, GitHub org settings (branch protection, CODEOWNERS enforcement).

## 3. `nearpay-testbed`

```
compose/compose.test.yaml   # overrides: time scale 0.1 for sandbox, test secrets
merchant-demo/              # tiny Express app: creates sessions via @nearpay/node, receives webhooks,
                            # exposes GET /premium behind @nearpay/x402 (agent demo)
demo/checkout.ts            # npm run demo:checkout → creates a session, prints URL, waits for webhook
demo/agent.ts               # npm run demo:agent → agent fetches /premium with withX402 + agent-signer
e2e/api|checkout|dashboard|agent/   # Vitest + Playwright suites per stage
contract-tests/             # consumer-driven tests across services from spec fixtures
load/                       # k6: sessions/attempts, webhook fan-out, x402 settle
chaos/                      # kill containers, block egress, stall NATS; assert money-safety invariants
canary/                     # mainnet canary runner (budget-capped, human-started)
ops/                        # Stage 1 ops CLI (local only): wrappers around `docker compose exec <svc> node dist/ops.js <cmd>` —
                            # kyb-decide, kill-switch-release, destination-cooloff-skip; no network auth path exists for them;
                            # Stage 2 replaces them with the console (ops principals, two-person rules)
fixtures/oneclick|near/     # recorded provider responses and mainnet receipts (SP-14)
```

## 4. `nearpay` entry repo

`README.md` (hackathon format), `HACKATHON.md`, `repos.json`, `scripts/bootstrap.sh`, `scripts/up.sh` (delegates to `nearpay-deploy/compose`), `AGENTS.md`, `LICENSE`. At each stage exit the README's "Try it" commands are executed verbatim in CI (`nearpay-testbed` job `readme-check`).

## 5. Stage mapping

Stage 1: compose local profile, Postgres init, Traefik routes, seed script, testbed demos + E2E for checkout and agent, entry repo live steps. Stage 2–4: added services/routes. Stage 5: server profile, infra, backups, observability profile on server, Cloudflare.
