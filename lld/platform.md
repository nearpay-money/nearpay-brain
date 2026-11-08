# LLD — Platform (`nearpay-deploy`, `nearpay-infra`, `nearpay-testbed`, `nearpay` entry)

> Version 1.6 (step 2.4 integration: Stage 2 Compose wiring and findings §1.4, Stage 2 E2E §3) · Version 1.5 (step 2.1: console `/ops/*` routing and unused `/webhooks/kyb` route in §1, ADR-0020) · Version 1.4 (step 1.4: Compose follow-ups applied and `seed.sh live` §1/§1.2, E2E suites and confirmed scenario outcomes §3) · Version 1.3 (steps 1.2 and 1.3 implemented; changes in §1.2, local stand-in images in §1.3, testbed in §3) · Version 1.1 audited (`audits/platform.md`) · ADR-0015, ADR-0016 · Infrastructure §0 · Foundations `00-foundations.md`

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
├── (../images/)                 # local stand-ins for third-party images, same tags (§1.3)
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
| `agentpay-api`, `agentpay-batcher` | nearpay-agentpay | internal, data, egress | postgres, nats, valkey (step 1.4) |
| `compliance-api`, `compliance-worker` | nearpay-compliance | internal, data, egress | postgres, nats |
| `webhooks-api`, `webhooks-ingest`, `webhooks-dispatcher` | nearpay-webhooks | internal, data (dispatcher → egress-proxy only) | postgres, nats; dispatcher: egress-proxy, valkey (step 1.4) |
| `notify-api`, `notify-worker` | nearpay-notify | internal, data, egress | postgres, nats |
| `billing-api`, `billing-worker` | nearpay-billing | internal, data, egress (RPC) | postgres, nats (Stage 3+) |
| `sandbox-api` | nearpay-sandbox | internal, data | postgres |
| `sentinel-api`, `sentinel-auditor` | nearpay-sentinel | internal, data, egress | postgres, nats |
| `migrate-<svc>` (one-shot) | each service image, `dist/main.js migrate` with the migrator URL | data | postgres-init (completed) |
| `static` | `nginxinc/nginx-unprivileged:1.29.8-alpine` behind Traefik; sites mounted read-only from the checkout, js, dashboard and console `dist/` and `nearpay-site/dist/docs` (placeholder page until built) | edge | — |
| `mailpit` (local) | `axllent/mailpit:v1.27.11` | internal | — |
| `registry` (local) | `verdaccio/verdaccio:6.10.6` | edge (+ host port `127.0.0.1:4873`) | — | local npm registry for `@nearpay/*` and `@nearpay-internal/*` packages so sibling repos consume **published artifacts** (ADR-0001) without a remote; GitHub Packages / npm replace it when remotes exist |
| `merchant-demo` (local, profile `merchant-demo`) | `nearpay/merchant-demo:local`, built from `nearpay-testbed/merchant-demo` (port 3000; §3) | demo (static IP `NP_DEMO_IP`, `.2` of `NP_DEMO_SUBNET`; dynamic addresses come from `NP_DEMO_IP_RANGE` `.4/30`, so traefik or the egress proxy can never take `.2`; the egress proxy allows exactly this private address, webhooks L-WH-02); mounts `.secrets/merchant-demo` read-only at `/run/secrets/merchant-demo` | payments-api |

**Networks:** `edge` (traefik, static), `internal` (all services; no internet), `data` (postgres, nats, valkey + their clients), `egress` (only containers that call the internet), `demo` (local: merchant-demo, egress-proxy, traefik). Containers not on `egress` cannot reach the internet.

**Traefik routing (local hostnames; server uses `*.nearpay.money`):**

| Host / path | Target | Middleware |
|---|---|---|
| `api.…/v1/checkout`, `/v1/payment_intents`, `/v1/refunds`, `/v1/payment_links`, `/v1/customers`, `/v1/balance_transactions`, `/v1/test_helpers` | payments-api | authz |
| `api.…/v1/account`, `/v1/auth`, `/v1/api_keys`, `/v1/settlement_destinations` | identity-api | authz |
| `api.…/v1/account/kyb`, `/webhooks/kyb` (router priority above identity's `/v1/account`) | compliance-api | authz |
| `ops.…/api/ops/*` (step 2.1: `/api` stripped; `/ops/merchants/{id}/suspension*` → compliance at priority 200, then by prefix: `/ops/auth`, `/ops/merchants`, `/ops/audit_events` → identity; `/ops/kyb_cases`, `/ops/cases`, `/ops/blocklist`, `/ops/screenings`, `/ops/rescreens` → compliance; `/ops/kill_switch`, `/ops/findings` → sentinel; `/ops/executions` → rails; `/ops/reconciliation_runs` → payments — generated from spec `OPS_ROUTES`) | owning service | authz (ops cookie) |
| `api.…/v1/products`, `/v1/prices`, `/v1/tax_rates`, `/v1/invoices`, `/v1/subscriptions`, `/v1/mandates` | billing-api | authz |
| `api.…/v1/webhook_endpoints`, `/v1/events`, `/v1/webhook_signing_keys` | webhooks-api | authz |
| `api.…/x402`, `/v1/agent_payments`, `/v1/checkout/sessions/{id}/agent_pay*` (priority over payments) | agentpay-api | authz |
| `api.…/v1/attestations` | sentinel-api | authz (anonymous allowed) + IP rate limit |
| `api.…/webhooks/email` (Stage 5) | notify-api | authz (anonymous allowed; provider signature verified by notify) |
| `api.…/v1/assets` | rails-api | authz (anonymous allowed) |
| `pay.…`, `dashboard.…` `/api/*` | the API routers above (prefix stripped before ForwardAuth): the checkout and dashboard call the API same-origin, so the API has no CORS surface; unrouted `/api/*` answers 404 | authz |
| `pay.…`, `js.…`, `dashboard.…`, `ops.…`, `docs.…` | static (nginx) | per-site headers: `frame-ancestors 'none'` (pay `/embed/*`: `https:` + local http), nosniff, referrer and permissions policies; js served with CORS + cross-origin CORP and `Cache-Control: public, max-age=300` (`/v1/` is the protocol major, not a release: the loader changes with releases, so it is never immutable); pay (`/` and `/embed/`) and dashboard `index.html` get a per-response CSP nonce — nginx `sub_filter` replaces `{{NP_CSP_NONCE}}` (pay) and `NPCSPNONCE` (dashboard) with the 128-bit `$request_id`, `sub_filter_once off` (NEAR Connect wallet frames need it; checkout §2, dashboard §1) |
| `api.…/healthz` | Traefik `ping@internal` (`OK`) | — |
| `mail.…`, `merchant-demo.…` (local) | mailpit, merchant-demo | — |

The `/webhooks/kyb` route is not used in v1 (no KYB vendor, ADR-0020): compliance implements no handler and the router may be dropped. ForwardAuth middleware: `http://identity-api:8080/internal/authz`, `authResponseHeaders: [X-NearPay-Token, X-Request-Id]`, `trustForwardHeader: false` — ForwardAuth derives `X-Forwarded-Method/Uri` from the request as routed (after `/api` stripping), so a client cannot influence the URI identity authorizes. A headers middleware removes any client-supplied `X-NearPay-Token` before ForwardAuth; only identity's response sets it. Every route matches `Path(p) || PathPrefix(p/)` (no accidental prefix matches); `/internal/*` is never routed. Router priorities are explicit so `/v1/checkout/sessions/{id}/agent_pay*` (200) → agentpay and `/v1/account/kyb` (200) → compliance win over the broader prefixes (100); static sites are 10. Verified for every Stage-1 public route on `api.…` and both `/api` hosts.

**Postgres init:** databases `identity, payments, billing, rails, agentpay, compliance, webhooks, notify, sandbox, sentinel`; roles `<db>_app` (DML), `<db>_migrator` (DDL); `sentinel_ro` with `CONNECT` on `identity` and `SELECT` on `identity.public.sentinel_destination_registry` only (granted by identity's migration through a `grant` step run by the migrator role). Implemented by `init/postgres/roles.sh`, run by the one-shot `postgres-init` on **every** `up` (idempotent; passwords re-synchronised from the URL secret files, the single source): the migrator owns its database (DDL, trusted extensions such as `pgcrypto`/`citext`); default privileges give the app role DML on tables and sequences the migrator creates; `CONNECT` is revoked from `PUBLIC` on every database (including `postgres`), so no role reaches another service's data. `migrate-<svc>` one-shots run `dist/main.js migrate` with the migrator URL after `postgres-init`; service processes start after their migrator completed.

**Secrets:** `nearpay-deploy/secrets/<env>/*.sops.env|*.sops.key` (age recipients = owner + server); `scripts/secrets-decrypt.sh <env>` writes `./.secrets/<name>` (git-ignored) mounted as Docker secrets. Local `env` ships **test-mode-only** secrets generated by `scripts/seed.sh` (dev keys), so the stack runs without any real credentials; live mode requires the owner to add 1Click API key, Explorer token, treasury/relayer/keeper accounts and RPC keys.

**Local secrets (`seed.sh keys`, test-only, never overwritten):** `postgres_password`; `db_url_<svc>` / `db_migrate_url_<svc>` (app / migrator URLs) and `db_registry_url_sentinel`; `jwt/current.pem`, `jwt/next.pem` (mounted at `/run/secrets/jwt/`), `sentinel_signing_key.pem` (mounted as `/run/secrets/sentinel-<kid>.pem`), `payments_ticket_key.pem`, `webhook_signing_key.pem`, `sandbox_signing_key.pem` (Ed25519 PKCS#8 PEM, generated with Node so macOS works); `service_client_<svc>` and identity's `service_clients.json` (`{"clients":[{"client_id","secret_sha256"}]}`); `kek_<svc>` (32 bytes base64, one per service for all envelope encryption: PII, idempotency bodies, endpoint secrets); `turnstile_secret` (Cloudflare's always-pass test secret; sitekey `1x00000000000000000000AA`); `registry_password`. Public material in `.secrets/public/`: `keys.env` (`NP_SENTINEL_KID`, `NP_JWT_CURRENT_KID`, `NP_PAYMENTS_TICKET_PUBKEY`, `NP_SANDBOX_SIGNING_PUBKEY`, `NP_WEBHOOK_SIGNING_KID` — public keys as raw 32-byte base64url, kids `local-<12 hex of sha256(pubkey)>`) and `sentinel_keys.json` (`[{kid, public_key}]` for `VITE_SENTINEL_KEYS`). TLS: `tls/{cert,key,ca}.pem` from mkcert when installed (trusted after `mkcert -install`), otherwise a self-signed local CA + leaf (browsers warn; passkeys need the trusted certificate).

**Build/start order (local, `scripts/up.sh local`):** `seed.sh keys` (secrets + TLS) → registry up → publish `nearpay-spec` then `nearpay-kit`, then the client packages `nearpay-node` and `nearpay-agentkit` (single-package repos supported; the merchant demo installs `@nearpay/node`, `@nearpay/x402`, `@nearpay/agent-signer`) when a package version in those repositories is missing (release scripts accept only a loopback registry and pack reproducibly, so lockfile integrity hashes stay valid when a fresh registry is re-seeded) → static builds (`pnpm install && pnpm build` in checkout, then js, then dashboard) read the **public** sentinel key from `.secrets/public/sentinel_keys.json` (pinned at build; checkout §2) → `docker build --network host --build-arg NP_REGISTRY_URL=http://127.0.0.1:4873 -t nearpay/<svc>:local ../nearpay-<svc>` for every service whose repo has a `Dockerfile` (Compose's bake builder refuses `network: host` without an interactive grant, so Compose never builds or pulls service images) → `docker compose up --wait` (postgres-init → migrate-<svc> → processes healthy) → `seed.sh data` (step 1.2+). Each service has a Compose profile named after it; `up.sh` enables it once `../nearpay-<svc>/Dockerfile` exists and lists the services not built yet. Requires Docker Compose ≥ 2.30, Node 22 + pnpm.

**Service image contract:** `ARG NP_REGISTRY_URL` used for `@nearpay/*` and `@nearpay-internal/*` installs; `node dist/main.js <process>` (Compose passes `command: [dist/main.js, <process>]`, so the image's entrypoint is `node`); `HEALTHCHECK` runs `dist/main.js healthcheck` (built into `@nearpay-internal/service`, GET `:8080/healthz`); non-root UID 10001, read-only root FS (Compose enforces `read_only`, `tmpfs /tmp`, `cap_drop: ALL`, `no-new-privileges`, 35 s stop grace).

**Scripts:** `up.sh [local] [--no-build]` (`--no-build` reuses existing images), `down.sh [--volumes]` (stops every profile), `seed.sh keys [--renew-tls]` (local keys and TLS; also creates `.secrets/merchant-demo/`, readable by UID 10001), `seed.sh data` and `seed.sh live` (step 1.4: validates the owner-provided mainnet credentials in `.secrets/live/` — `oneclick_api_key`, `explorer_token`, `relayer_key_1…4` — and writes `.secrets/live/compose.live.yaml` with only those present, which `up.sh`/`down.sh` include; what is missing stays off; nothing is generated or signed), `deploy.sh server` (Stage 5: pull digests, decrypt secrets, `docker compose up -d`, run migrations, smoke test), `backup-verify.sh`.

**`seed.sh data`** (test mode, stack running): (1) runs identity's ops CLI `seed-demo` inside `identity-api` (owner, account, active test destination, test secret + publishable keys; idempotent on the email; identity §17) and writes its JSON to `.secrets/demo.json`; (2) writes the demo's secret key to `.secrets/merchant-demo/secret_key` and creates (once) the webhook endpoint `http://merchant-demo:3000/webhooks` for `payment_intent.*`, `checkout.session.*`, `agent_payment.*` through the API, storing its signing secret as `.secrets/merchant-demo/webhook_secret`; (3) tops the demo agent's simulated Intents balance up to 100 USDC with the sandbox ops CLI `intents-fund` (§3).

### 1.1 Changes in version 1.2 (from the step 1.1 implementation)

- Static sites are served by nginx behind Traefik (Traefik's file provider does not serve files).
- The checkout joins the dashboard in calling the API same-origin under `/api` (no CORS anywhere).
- identity and payments need the `egress` network (RPC for attestations; Turnstile siteverify).
- ForwardAuth uses `trustForwardHeader: false`; client-supplied `X-NearPay-Token` is stripped.
- Postgres roles are applied by an idempotent `postgres-init` one-shot on every `up` (was a first-boot SQL file).
- Service images are built by `up.sh` (host network for the registry), not by Compose; the image contract above.
- `/healthz` on the API host is Traefik's ping (`OK`); service health is per container.

### 1.2 Changes in version 1.3 (from the steps 1.2 and 1.3 implementation)

- All nine Stage 1 services run in the stack (27 containers healthy in the build workspace); `seed.sh data` is implemented through identity's `seed-demo` (above).
- `up.sh` publishes the client SDK repos too; the merchant demo is built from its own package directory.
- Static sites: per-response CSP nonces for the checkout and the dashboard; the js loader is cached for 5 minutes instead of a year.
- Compose follow-ups reported by the service builds, applied in step 1.4 (`nearpay-deploy` 5329223): `agentpay-api`/`agentpay-batcher` and `webhooks-dispatcher` depend on a healthy `valkey` (settle waiters, quotas, endpoint semaphores); the batcher gets `NP_RELAYER_KEY_FILES=/run/secrets/relayer_key_*` from the `seed.sh live` overlay when the owner provides `relayer_key_1…4` (live relaying is off without them); the webhooks services receive `NP_WEBHOOK_SIGNING_KID` from `keys.env` (otherwise their default kid `wh-…` differs from the `local-…` kid deploy derives for the same key). Still open: the egress proxy should restrict ports to 80/443.

### 1.4 Changes in version 1.6 (step 2.4 integration)

- **Console:** its own image (`nearpay/console:<tag>`, nginx from nearpay-console's Dockerfile, profile `console`, built by `up.sh` like a service image); the static nginx no longer serves `ops.`. Traefik answers `ops.<domain>` for local clients only (`ipAllowList`: loopback + Docker bridge ranges; Cloudflare Access on the server). `ops.<domain>/api/ops/*` routers are generated from spec `OPS_ROUTES` by `scripts/lib/ops-routes.mjs` into `traefik/dynamic/ops.yaml` (`--check` detects drift); the five anonymous `/ops/auth/*` routes have their own rate-limited router. **Query strings:** Traefik cannot drop only the query from its access log, so ops GET routers are not access-logged (console list queries can carry personal data, e.g. `listOpsMerchants q`); writes carry no query string and stay logged.
- Anonymous `GET|POST /v1/payment_links/{id}/public/**` router (per-IP limit before payments' own limiter); `pay.` `/c|/l|/i` get the hosted `frame-ancestors 'none'` header (meta CSP from the build).
- Env: `NP_ENV` has no default (`local.env` sets `local`; kit 0.2.1 requires it); `NP_SCREENING_PROVIDER=local`; `NP_OPS_ORIGIN`/`NP_CONSOLE_ORIGIN`/`NP_DASHBOARD_ORIGIN`; identity `NP_OPS_AAGUID_ALLOWLIST_FILE` (Compose config; the local list holds the two virtual/self-attesting AAGUIDs identity honours only with `NP_ENV` local/test); `NP_PRICING_CURRENCIES` for payments and rails; rails `NP_STABLE_PEGS`, `NP_TEST_FX_RATES`, `NP_OBS_ARCHIVE_STORE=file:/data/obs-archive` on volume `rails-obs-archive` (chowned by the one-shot `volume-init`); payments `NP_NEAR_RPC_URLS` and `NP_LINK_SESSIONS_*`.
- Egress: the local `egress` network is not filtered per host. Rails' hosts for the server allow-list are documented in Compose (1Click, RPCs, neardata, Explorer, `api.coingecko.com`, `www.ecb.europa.eu`).
- **Health checks** of NearPay services run every 15 s (timeout 10 s) instead of the images' 5 s: each check boots Node, and ~28 containers on 2 CPUs saturated the machine after a restart so that every check timed out.
- `seed.sh data` refreshes rails' token registry before `seed-demo` and subscribes merchant-demo to `refund.*`.
- Offline machines: compliance answers live screening with `screening_unavailable` until an OFAC list was loaded; the testbed imports a synthetic list (`fixtures/ofac`, not OFAC data) when compliance has none.

### 1.3 Local stand-in images (machines that cannot pull registries)

`nearpay-deploy/images/` builds local images under the **same tags** as every third-party image the stack, the service Dockerfiles and the Testcontainers suites use, so no repository changes when Docker Hub, gcr.io or ghcr.io are unreachable (a sandboxed CI or cloud workspace). On a normal machine Docker pulls the official images and this folder is not used.

- Tags covered: `node:22-bookworm-slim`, `gcr.io/distroless/nodejs22-debian12:nonroot`, `postgres:17.11`, `nats:2.10.29-alpine`, `valkey/valkey:8.1.11-alpine`, `traefik:v3.5.6`, `axllent/mailpit:v1.27.11`, `nginxinc/nginx-unprivileged:1.29.8-alpine`, `verdaccio/verdaccio:6.10.6`, `ghcr.io/foundry-rs/foundry:v1.4.4`, and `nearpay/egress-proxy:smokescreen-v0.1.0` (Smokescreen built offline from vendored sources).
- Base: `nearpay-local/ubuntu:25.10`, a debootstrap made by `images/rootfs.sh`; official release binaries or npm packages on top, downloads verified against `images/SHA256SUMS`. Distro packages can trail a tag's patch level (e.g. PostgreSQL 17.10 under `postgres:17.11`), so each image records its real contents in the label `org.nearpay.contents`.
- `images/build.sh [--force] [<image>]` builds what is missing; `NP_IMAGES_EXTRA_CA=<pem>` trusts an intercepting proxy's CA inside the node images (so `pnpm install` works during service builds); `NP_IMAGES_UPDATE_SUMS=1` records checksums after a version bump.
- Such machines **build the service images first, then start with `scripts/up.sh --no-build`**: a plain `up.sh` asks Compose to rebuild the egress proxy from `compose/egress-proxy/Dockerfile`, which pulls `golang`/`alpine`.
- Testcontainers' reaper image is not built: run tests with `TESTCONTAINERS_RYUK_DISABLED=true`.
- Remove a stand-in with `docker image rm <tag>` once the registries are reachable.

## 2. `nearpay-infra` (Stage 5)

Ansible: `base-hardening` (Debian 12, unattended upgrades, SSH keys only, nftables default-deny, fail2ban), `docker-host` (Docker CE, compose plugin, log rotation, WAL-G timer, node exporter), `wireguard` (admin access). Terraform: Cloudflare (DNS, proxy, origin certs, Authenticated Origin Pulls, WAF managed rules, Turnstile sitekeys, Access app for `ops.`), object storage bucket (WAL-G, backups, audit) at an EU provider, GitHub org settings (branch protection, CODEOWNERS enforcement).

## 3. `nearpay-testbed`

```
compose/compose.test.yaml   # overrides: time scale 0.1 for sandbox, test secrets
merchant-demo/              # own package (own lockfile, Dockerfile; image nearpay/merchant-demo:local), Express 5,
                            # @nearpay/node 0.1.0 (sessions, constructEvent) + @nearpay/x402 0.1.1 (GET /premium)
demo/checkout.ts            # pnpm demo:checkout → session via merchant-demo, scripted payer, waits for the signed webhook
demo/agent.ts               # pnpm demo:agent → agent fetches /premium with withX402 + agent-signer (--live: mainnet, step 1.5)
lib/                        # spec-route HTTP client, stack locations, merchant-demo client, scripted payers, scenarios 00–07
e2e/api|checkout|dashboard|agent/   # Vitest + Playwright suites per stage
test/                       # unit + mock-backed tests (spec-validated fake API, sentinel test key, webhooks)
contract-tests/             # consumer-driven tests across services from spec fixtures
load/                       # k6: sessions/attempts, webhook fan-out, x402 settle
chaos/                      # kill containers, block egress, stall NATS; assert money-safety invariants
canary/                     # mainnet canary runner (budget-capped, human-started)
ops/                        # Stage 1 ops CLI (local only, `pnpm ops <cmd>`): `docker compose … exec -T <svc>-api /nodejs/bin/node
                            # dist/ops.js …` with up.sh's project, files, env files and the service profile; no network auth
                            # path exists for them; Stage 2 replaces them with the console (ops principals, two-person rules)
fixtures/oneclick|near/     # recorded provider responses and mainnet receipts (SP-14)
```

**merchant-demo:** reads `secret_key` / `webhook_secret` lazily (5 s cache) from `NP_DEMO_SECRETS_DIR` (default `/run/secrets/merchant-demo`), so it starts healthy before `seed.sh data` (Compose runs `up --wait` first) and picks the secrets up without a restart. Routes: `/`, `POST /checkout` (USD 2.00; test-mode-only `amount` and `expires_in` overrides for sandbox scenarios, incl. 07), `/orders/:cs`, `/success`, `POST /webhooks` (v1 HMAC, any `v1`, 300 s tolerance, dedupe by event id), `/events` (only when `NP_ENV` is local/test), `/premium` (x402, 10000 base units). Its public URL defaults to the API URL with `api.` → `merchant-demo.`; x402 resource URLs use it, not the container URL.

**Ops CLI commands:** `kyb-show`, `kyb-decide` (compliance, `--actor`), `kill-switch` / `kill-switch-engage` / `kill-switch-release` (sentinel; release needs `--yes`; sentinel's CLI has no `--actor`, so the operator's name is prefixed to the reason), `destination-cooloff-skip` and `account-show` (identity; T1's "account live" check uses `account-show`, since `GET /v1/account` is session-only), `rails-flags`.

**Demo agent (test only):** fixed key `sha256("nearpay demo agent (test only)")`, implicit account `0857962d91335176f92257df38175401936a4ca31b832f0c98c77b836800c655`. Test-mode x402 and T3/T5 need its sandbox Intents balance funded; there is no public route for it, so `seed.sh data` funds it with the sandbox ops CLI (`ops.js intents-fund <account> <token> <amount>`, test mode by construction; sandbox §9).

**Scripted payer = checkout rules:** creates the attempt, verifies sentinel's co-signature with the pinned key (fail closed), then "Simulate payment" (`route_quote`) or signs the attested requirement (`intents_transfer`); Turnstile uses Cloudflare's always-pass test token.

**E2E modes:** `NP_E2E=1` runs against the real stack; `NP_E2E=mock` runs the same suites (scenario 00, T2–T5) against the spec-validated mock stack; slow scenarios (01, 02, 07, run concurrently) only with `NP_E2E_SLOW=1`. Set `NODE_EXTRA_CA_CERTS=../nearpay-deploy/.secrets/tls/ca.pem` explicitly.

**E2E suites (step 1.4, all green against the full Compose stack in test mode):** `pnpm test:e2e` — `e2e/api/checkout` (T2/T6: session → sentinel-verified instructions → simulated deposit → signed webhooks; INV-2 evidence), `e2e/api/scenarios` (sandbox 00–07: final intent/attempt/session state, required and forbidden webhooks, statement lines), `e2e/api/kill-switch` (T7: a spoofed request is refused without engaging; drill engage → rails stops quoting and agentpay pauses → ops CLI release → both resume), `e2e/api/onboarding` (T1 ops half), `e2e/checkout/intents` and `e2e/agent/*` (T3 Intents balance with NEP-413, T4 x402 `/premium`, T5 agent pays a session); `NP_E2E=1 pnpm test:pw` (system Chrome) — `e2e/checkout/hosted.pw.ts` (hosted checkout: manual path, verified merchant + fingerprint, address after network confirmation, simulate, redirect, webhook) and `e2e/dashboard/onboarding.pw.ts` (T1: sign-up → magic link via Mailpit → passkey on a CDP virtual authenticator → live NEAR Intents destination signed with NEP-413 → cool-off skip → KYB → ops decision → `live_enabled`). The browser tests stub Turnstile with the always-pass test sitekey's dummy token (services answer the local test secret themselves) and serve an empty NEAR Connect manifest, so no third-party script loads; Chrome trusts the local certificate through its pinned SPKI (WebAuthn needs it). The scripted payer asks sentinel again (up to 4 tries) while it answers `503 attestation_unavailable`; every answer is still verified against the pinned key.

**Scenario outcomes (confirmed in step 1.4 against domain-model §4–§5; `lib/scenarios.ts`):** 00 succeeded · 01 processing then succeeded · 02 attempt refunded at the 30 min quote deadline, intent back to `requires_payment` with `payment_intent.payment_failed`, session open (1 h session) · 03 succeeded, excess refunded (no overpayment) · 04 provider `FAILED` after the deposit → attempt `held` (`failed_after_deposit`), intent `requires_review` (`verification_held`) with `payment_intent.requires_review`, no succeeded or `payment_failed` event (the step 1.3 expectation of `requires_payment` + `payment_failed` was wrong) · 05 `requires_review` (`short_settlement`), no succeeded event · 06 succeeded + `overpaid` (`duplicate`) · 07 settles after the session's `expires_at` with `late: true`; it needs a **420 s** session, because USDC on Base has `min_deadline_s` 300 and a 300 s session is correctly refused as `route_too_slow`.

**Stage 2 E2E (step 2.4, green in test mode):** `e2e/stage2/` — refunds (route refund to the payer's origin chain via the test-mode signing path `simulateRefundDeposit` → `succeeded` with evidence, signed `refund.succeeded`, receipt in Mailpit; Intents refund signed NEP-413 verbatim by the merchant's Intents account after the dashboard's consistency checks → `submit_tx {signed_intent}` → agentpay relay), payment link paid from Base while settling on NEAR (session complete, `uses` + 1, webhook, hosted `/l/` header), reconciliation through the console (completed day `ok`/`diff_count 0`, today `reconciliation_day_open`; evidence side stopped → run `error` → `reconciliation_diff` case — a true amount `diff` cannot be produced on a same-day stack without writing service databases), console smoke (two-person kill-switch release incl. `approver_conflict` and `ops_role` refusal; self-declared KYB → console approval with step-up → `live_enabled`). Operators and merchants authenticate with a software WebAuthn authenticator (`lib/webauthn.ts`: ES256, packed self attestation, all-zero AAGUID — local/test only) through the same hosts and routes the console and dashboard use (`lib/ops-console.ts`: cookie, CSRF, step-up replay with the same Idempotency-Key). Node does not resolve `*.localhost` on glibc: `lib/localhost-dns.ts` maps them to loopback (RFC 6761) in E2E workers and Playwright's Node side. The ops CLI adds `operator-add <email> ops|ops_admin` (kit 0.2.0 enforces `ops_role`), `operator-revoke`, `sanctions-import`.

## 4. `nearpay` entry repo

`README.md` (hackathon format), `HACKATHON.md`, `repos.json`, `scripts/bootstrap.sh`, `scripts/up.sh` (delegates to `nearpay-deploy/compose`), `AGENTS.md`, `LICENSE`. At each stage exit the README's "Try it" commands are executed verbatim in CI (`nearpay-testbed` job `readme-check`).

## 5. Stage mapping

Stage 1: compose local profile, Postgres init, Traefik routes, seed script, testbed demos + E2E for checkout and agent, entry repo live steps. Stage 2–4: added services/routes. Stage 5: server profile, infra, backups, observability profile on server, Cloudflare.
