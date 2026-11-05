# LLD — Platform (`nearpay-deploy`, `nearpay-infra`, `nearpay-testbed`, `nearpay` entry)

> Version 1.1 (audited, `audits/platform.md`) · ADR-0015, ADR-0016 · Infrastructure §0 · Foundations `00-foundations.md`

## 1. `nearpay-deploy/compose` — the v1 stack

```
compose/
├── compose.yaml                 # all services, networks, volumes, secrets (profile-independent)
├── compose.local.yaml           # build contexts ../../nearpay-<svc>, *.nearpay.localhost, mailpit, merchant-demo, ops ports
├── compose.server.yaml          # GHCR image digests, Cloudflare origin certs, WAL-G sidecar, restart policies
├── compose.observability.yaml   # otel-collector, victoriametrics, loki, tempo, grafana
├── init/postgres/00-roles.sql   # databases + roles per service, sentinel_ro grant
├── traefik/dynamic.yaml         # routers, ForwardAuth middleware, static file servers
├── env/local.env.example        # non-secret config; secrets via ../secrets
└── scripts/                     # (in ../scripts) up, down, seed, deploy, secrets-decrypt
```

**Services (container names = Compose service names):**

| Service | Image / build | Networks | Depends on (healthy) |
|---|---|---|---|
| `traefik` | `traefik:v3` | edge, internal | — |
| `postgres` | `postgres:17` | data | — |
| `nats` | `nats:2.10` (`-js -sd /data`) | data | — |
| `valkey` | `valkey/valkey:8` | data | — |
| `egress-proxy` | Smokescreen | internal, egress | — |
| `identity-api`, `identity-worker` | nearpay-identity | internal, data | postgres, nats, valkey |
| `payments-api`, `payments-worker` | nearpay-payments | internal, data | identity-api, rails-api |
| `rails-api`, `rails-tracker`, `rails-feeds` | nearpay-rails | internal, data, egress | postgres, nats, compliance-api |
| `agentpay-api`, `agentpay-batcher` | nearpay-agentpay | internal, data, egress | postgres, nats |
| `compliance-api`, `compliance-worker` | nearpay-compliance | internal, data, egress | postgres, nats |
| `webhooks-api`, `webhooks-ingest`, `webhooks-dispatcher` | nearpay-webhooks | internal, data (dispatcher → egress-proxy only) | postgres, nats |
| `notify-api`, `notify-worker` | nearpay-notify | internal, data, egress | postgres, nats |
| `billing-api`, `billing-worker` | nearpay-billing | internal, data, egress (RPC) | postgres, nats (Stage 3+) |
| `sandbox-api` | nearpay-sandbox | internal, data | postgres |
| `sentinel-api`, `sentinel-auditor` | nearpay-sentinel | internal, data, egress | postgres, nats |
| `migrate-<svc>` (one-shot) | each service image `migrate` | data | postgres |
| `static` (Traefik file provider) | volumes from checkout, js, dashboard, console, docs builds | edge | — |
| `mailpit` (local) | `axllent/mailpit` | internal | — |
| `merchant-demo` (local) | nearpay-testbed `merchant-demo` | internal (reachable by `egress-proxy` under the local-only allowance, webhooks L-WH-02) | payments-api |

**Networks:** `edge` (traefik, static), `internal` (all services; no internet), `data` (postgres, nats, valkey + their clients), `egress` (only containers that call the internet). Containers not on `egress` cannot reach the internet.

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
| `api.…/v1/assets` | rails-api | authz (anonymous allowed) |
| `pay.…`, `js.…`, `dashboard.…` (+ `/api/*` → api routes with prefix strip), `ops.…`, `docs.…` | static | security headers per site |

ForwardAuth middleware: `http://identity-api:8080/internal/authz`, `authResponseHeaders: [X-NearPay-Token, X-Request-Id]`, `trustForwardHeader: true`. Router priorities are explicit (longest-prefix first) so `/v1/checkout/sessions/{id}/agent_pay*` → agentpay and `/v1/account/kyb` → compliance win over the broader prefixes.

**Postgres init:** databases `identity, payments, billing, rails, agentpay, compliance, webhooks, notify, sandbox, sentinel`; roles `<db>_app` (DML), `<db>_migrator` (DDL); `sentinel_ro` with `CONNECT` on `identity` and `SELECT` on `identity.public.sentinel_destination_registry` only (granted by identity's migration through a `grant` step run by the migrator role).

**Secrets:** `nearpay-deploy/secrets/<env>/*.sops.env|*.sops.key` (age recipients = owner + server); `scripts/secrets-decrypt.sh <env>` writes `./.secrets/<name>` (git-ignored) mounted as Docker secrets. Local `env` ships **test-mode-only** secrets generated by `scripts/seed.sh` (dev keys), so the stack runs without any real credentials; live mode requires the owner to add 1Click API key, Explorer token, treasury/relayer/keeper accounts and RPC keys.

**Build/start order (local):** `seed.sh keys` generates all local key pairs first (JWT, sentinel, payments ticket, webhook, sandbox) → static builds of checkout/js/dashboard read the **public** sentinel key from `.secrets/public/` (pinned at build) → `docker compose up` → migrations → `seed.sh data`.

**Scripts:** `up.sh [profile]`, `down.sh`, `seed.sh` (generate local keys, create a demo merchant via the API in test mode, register `merchant-demo` webhook), `deploy.sh server` (Stage 5: pull digests, decrypt secrets, `docker compose up -d`, run migrations, smoke test), `backup-verify.sh`.

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
ops/                        # Stage 1 ops CLI: kyb-decide, kill-switch-release, destination-cooloff-skip (local only)
fixtures/oneclick|near/     # recorded provider responses and mainnet receipts (SP-14)
```

## 4. `nearpay` entry repo

`README.md` (hackathon format), `HACKATHON.md`, `repos.json`, `scripts/bootstrap.sh`, `scripts/up.sh` (delegates to `nearpay-deploy/compose`), `AGENTS.md`, `LICENSE`. At each stage exit the README's "Try it" commands are executed verbatim in CI (`nearpay-testbed` job `readme-check`).

## 5. Stage mapping

Stage 1: compose local profile, Postgres init, Traefik routes, seed script, testbed demos + E2E for checkout and agent, entry repo live steps. Stage 2–4: added services/routes. Stage 5: server profile, infra, backups, observability profile on server, Cloudflare.
