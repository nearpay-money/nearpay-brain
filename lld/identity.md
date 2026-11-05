# LLD — `nearpay-identity`

> Version 1.1 (audited, `audits/identity.md`) · HLD §5.1, ADR-0005, ADR-0014 · Foundations: `00-foundations.md`

## 1. Scope

**Owns:** accounts, users, memberships/roles, dashboard sessions (passkeys, step-up, recovery), API keys, settlement destinations (attestation, approvals, cool-off, defaults), fee schedules, gateway authorisation (`/internal/authz`), internal JWT/JWKS, service tokens, account audit log.
**Does not own:** KYB decisions (compliance), screening (compliance), destination minimum amounts (rails computes; identity displays), checkout client secrets (payments validates).

## 2. Runtime

| Compose service | Command | Replicas (v1) | Notes |
|---|---|---|---|
| `identity-api` | `api` | 2 | Public `/v1/account…`, `/v1/auth…`, `/v1/api_keys…`, `/v1/settlement_destinations…`; internal `/internal/*` |
| `identity-worker` | `worker` | 1 | jobs (cool-off activation, purge), outbox relay, consumers |

Memory 256 MB each. `/internal/authz` p99 ≤ 5 ms (cache hit), ≤ 25 ms (miss).

## 3. Code layout

```
src/api/            routes: auth.ts, account.ts, api-keys.ts, destinations.ts, internal.ts
src/app/            use cases: signup, login, stepUp, recover, createKey, rollKey, revokeKey,
                    createDestination, attestDestination, approveDestination, activateDestination,
                    disableDestination, setDefault, authorize, mintServiceToken, settlementContext
src/domain/account/ user/ session/ passkey/ api-key/ settlement-destination/ approval/ authz/
src/infra/db/       repositories (Kysely), migrations in /migrations
src/infra/clients/  compliance.ts, rails.ts, notify.ts, near-rpc.ts
src/infra/crypto/   attestation verifiers: nep413.ts, eip191.ts, solana.ts, bip322.ts; jwt-signer.ts
src/consumers/      compliance-kyb.ts, compliance-destination.ts
src/workers/        activate-destinations.ts, purge.ts
```

## 4. Data

```sql
create table accounts (
  id uuid primary key, display_name text not null, verified_legal_name text,
  country char(2), status text not null default 'active' check (status in ('active','suspended','closed')),
  kyb_status text not null default 'not_started'
    check (kyb_status in ('not_started','pending','needs_info','approved','rejected')),
  live_enabled boolean not null default false,
  fee_schedule_id uuid not null references fee_schedules(id),
  fee_bearer text not null default 'payer' check (fee_bearer in ('payer','merchant')),
  api_version text not null,                        -- pinned at creation, e.g. '2026-11-01'
  settings jsonb not null default '{}',             -- late_payments, underpay_tolerance_bps, branding, session_ttl_default
  recovered_at timestamptz,                         -- last account recovery; blocks destination changes 48 h
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0);

create table users (
  id uuid primary key, email citext not null unique, email_verified_at timestamptz,
  name_enc bytea, totp_secret_enc bytea, status text not null default 'active',
  created_at timestamptz not null default now(), last_login_at timestamptz);

create table memberships (
  account_id uuid not null references accounts(id), user_id uuid not null references users(id),
  role text not null check (role in ('owner','admin','developer','finance','support','viewer')),
  created_at timestamptz not null default now(), primary key (account_id, user_id));

create table passkeys (
  id uuid primary key, user_id uuid not null references users(id), credential_id bytea not null unique,
  public_key bytea not null, sign_count bigint not null, transports text[] not null default '{}',
  aaguid uuid, name text not null, created_at timestamptz not null default now(), last_used_at timestamptz);

create table sessions (
  id uuid primary key, user_id uuid not null references users(id), account_id uuid not null references accounts(id),
  token_hash bytea not null unique, csrf_hash bytea not null, step_up_at timestamptz,
  created_at timestamptz not null default now(), expires_at timestamptz not null, last_seen_at timestamptz not null,
  revoked_at timestamptz, ip_enc bytea, user_agent text);
create index sessions_user on sessions (user_id) where revoked_at is null;

create table magic_links (
  token_hash bytea primary key, email citext not null, purpose text not null check (purpose in ('signup','recovery','invite')),
  account_id uuid, role text, expires_at timestamptz not null, used_at timestamptz);

create table api_keys (
  id uuid primary key, account_id uuid not null references accounts(id), livemode boolean not null,
  type text not null check (type in ('secret','publishable','restricted')),
  key_hash bytea not null unique,                   -- sha256(full key)
  display_prefix text not null, last4 char(4) not null, -- e.g. 'sk_live_' / '…a1b2'
  scopes text[] not null default '{}', name text not null, ip_allowlist cidr[] not null default '{}',
  created_by uuid not null, created_at timestamptz not null default now(),
  expires_at timestamptz, revoked_at timestamptz, last_used_at timestamptz);
create index api_keys_account on api_keys (account_id, livemode) where revoked_at is null;

create table fee_schedules (
  id uuid primary key, name text not null unique,
  rules jsonb not null,   -- {same_asset|stable_pair|volatile|intents_transfer: {total_bps:int, min_fee_usd_micros:bigint}} (classes defined in rails LLD §8.1)
  provider_split jsonb not null, -- {oneclick:{share_bps:5000, floor_bps:20, stable_floor_bps:1}}
  created_at timestamptz not null default now());

create table settlement_destinations (
  id uuid primary key, account_id uuid not null references accounts(id), livemode boolean not null,
  asset_id text not null, type text not null check (type in ('chain_address','intents_account')),
  chain text not null,                              -- CAIP-2; 'near:mainnet' for intents accounts
  address text not null, address_norm text not null, -- lowercase for EVM, as-is otherwise
  label text not null,
  status text not null check (status in ('pending_verification','pending_approval','cooling_off','active','disabled')),
  verification text not null check (verification in ('signature','unverified','test')),
  attestation jsonb,                                -- {standard, message, signature, public_key, verified_at}
  per_payment_cap_usd_micros bigint,                -- set when unverified
  approvals_required smallint not null, active_at timestamptz,
  screening_id text, created_by uuid not null, disabled_at timestamptz, disabled_reason text,
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0);
create unique index destinations_unique_live on settlement_destinations (chain, address_norm, asset_id)
  where status in ('pending_approval','cooling_off','active') and livemode and verification = 'signature';
-- unverified rows may not share (chain,address,asset) with ANY live non-disabled row of another account (checked in create, under
-- a transaction-level advisory lock on hashtext(chain||address_norm||asset_id)); sentinel treats residual duplicates as 'ambiguous'
create index destinations_account on settlement_destinations (account_id, livemode, status);

create table account_default_destinations (
  account_id uuid not null, livemode boolean not null, asset_id text,  -- null asset = account default
  destination_id uuid not null references settlement_destinations(id),
  primary key (account_id, livemode, coalesce_asset));                -- see migration: generated column coalesce(asset_id,'*')

create table destination_approvals (
  destination_id uuid not null references settlement_destinations(id), user_id uuid not null,
  decision text not null check (decision in ('approve','reject')), created_at timestamptz not null default now(),
  primary key (destination_id, user_id));

create table attestation_challenges (
  id uuid primary key, destination_id uuid not null references settlement_destinations(id),
  message text not null, nonce bytea not null, expires_at timestamptz not null, used_at timestamptz);

create table settlement_destination_transitions (… per foundations §7 …);

create table ops_operators (user_id uuid primary key references users(id), role text not null check (role in ('ops','ops_admin')),
  created_at timestamptz not null default now(), revoked_at timestamptz);   -- console access; passkeys must be hardware-backed (AAGUID allowlist)

create table service_clients (client_id text primary key, secret_hash bytea not null, scopes text[] not null,
  created_at timestamptz not null default now(), revoked_at timestamptz);

create table signing_keys (kid text primary key, public_key bytea not null, created_at timestamptz not null default now(),
  not_after timestamptz not null);    -- private keys live in secret files; this table publishes JWKS

create table audit_events (
  id uuid primary key, account_id uuid, actor_type text not null, actor_id text not null, action text not null,
  target_type text, target_id text, details jsonb not null default '{}', ip_enc bytea,
  created_at timestamptz not null default now());
create index audit_account on audit_events (account_id, created_at desc);

create view sentinel_destination_registry as        -- granted SELECT to role sentinel_ro only
  select d.id, d.account_id, d.livemode, d.asset_id, d.type, d.chain, d.address_norm, d.status, d.active_at,
         coalesce(a.verified_legal_name, a.display_name) as merchant_name, a.verified_legal_name is not null as kyb_verified
  from settlement_destinations d join accounts a on a.id = d.account_id where d.status = 'active';
```

Migration note: `account_default_destinations` uses a generated column `coalesce_asset text generated always as (coalesce(asset_id,'*')) stored` in the PK.

**Key formats:** `sk_live_` / `sk_test_` / `pk_live_` / `pk_test_` / `rk_live_` / `rk_test_` + 48 base62 chars (≈ 285 bits). Shown once. Stored as SHA-256. Session cookie `__Host-np_session` = 32 random bytes base64url (hash stored), `SameSite=Strict; Secure; HttpOnly; Path=/`, TTL 12 h sliding, absolute 7 days.

## 5. State machines

**SettlementDestination**

| From | Event / command | Guard | To | Side effects |
|---|---|---|---|---|
| — | `create` (session, step-up ≤ 10 min, role owner/admin) | address valid for chain; `recovered_at` older than 48 h; screening `allow` (sync); unique constraint; test mode → skip to `active` with `verification='test'` | `pending_verification` | challenge (10 min), audit, event `created` |
| `pending_verification` | `attest` | signature valid for message & key controls address (§8.2) | `pending_approval` or `cooling_off` | store attestation |
| `pending_verification` | `mark_unverified` (owner, step-up) | chain family has no message signing **or** owner opts in; no live non-disabled row of another account has the same `(chain,address,asset)` | `pending_approval`/`cooling_off` | cap = 1,000 USD per payment |
| `pending_approval` | `approve` (distinct admin/owner, step-up) | approvals ≥ required (`2` if account has ≥ 2 owner/admin members, else `1` = creator) | `cooling_off` | `active_at = now() + 24 h`; notify all owners/admins; job `destination.activate` |
| `cooling_off` | job `destination.activate` at `active_at` | not disabled; re-screen `allow` | `active` | event `activated`; if first destination for the asset → set default |
| any non-`disabled` | `disable` (owner/admin, step-up) or `compliance.destination.blocked` | — | `disabled` | if default → unset default; event `disabled`; notify |

**Account live mode:** `live_enabled = (kyb_status = 'approved') and exists(active live destination) and status = 'active'`; recomputed on every relevant event; change emits `identity.account.updated`.

## 6. API

All public routes follow `api-conventions.md`; dashboard routes are session-only (`ptype=session`), reached same-origin via `dashboard.nearpay.<tld>/api`.

| Method & path | Auth / role | Request → Response | Errors | Idem. |
|---|---|---|---|---|
| `POST /v1/auth/signup` | anonymous (Turnstile) | `{email}` → `202` (magic link sent) | `rate_limited` | — |
| `POST /v1/auth/magic_links/verify` | anonymous | `{token}` → `{registration_token}` (5 min) | `link_expired`, `link_used` | — |
| `POST /v1/auth/passkeys/registration/options` · `/verify` | registration_token or session | WebAuthn options / attestation → session cookie (new user: creates account `owner`) | `webauthn_failed` | — |
| `POST /v1/auth/passkeys/authentication/options` · `/verify` | anonymous | `{email?}` / assertion → session cookie + CSRF token | `webauthn_failed` | — |
| `POST /v1/auth/step_up/options` · `/verify` | session | assertion → `{step_up_until}` (10 min); any step-up-protected endpoint called without a fresh step-up returns `401 step_up_required` | `webauthn_failed` | — |
| `POST /v1/auth/recovery` | anonymous (Turnstile) | `{email}` → `202`; link → new passkey, sets `accounts.recovered_at` | — | — |
| `POST /v1/auth/logout` | session | `204` | — | — |
| `GET /v1/account` · `PATCH /v1/account` | session (PATCH: owner/admin) | account object (settings whitelist) | `invalid_setting` | yes |
| `GET /v1/account/requirements` | session | `{live_enabled, missing:['kyb','destination']}` | — | — |
| `GET /v1/account/members` · `POST` invite · `DELETE /{user}` | session owner/admin (+step-up for POST/DELETE) | members / invitation | `last_owner` | yes |
| `GET /v1/api_keys` | session developer+ | list (no secrets) | — | — |
| `POST /v1/api_keys` | session developer+ & step-up | `{type, livemode, name, scopes?, ip_allowlist?, expires_at?}` → key **with secret once** | `scope_unknown` | yes |
| `POST /v1/api_keys/{id}/roll` | session developer+ & step-up | `{expire_old_in_seconds ≤ 7 d}` → new key | — | yes |
| `DELETE /v1/api_keys/{id}` | session developer+ & step-up | `204` | — | — |
| `GET /v1/settlement_destinations` | session any role; secret key read-only (`destinations.read`) | list incl. `min_payment_amount` (from rails, cached 1 h) | — | — |
| `POST /v1/settlement_destinations` | session owner/admin & step-up | `{asset_id, type, chain, address, label}` → destination + `attestation_challenge {id, message, expires_at, signing_standard}` | `address_invalid`, `destination_in_use`, `recovery_lock`, `screening_blocked`, `memo_destination_unsupported`, `asset_unsupported` | yes |
| `POST /v1/settlement_destinations/{id}/attest` | session owner/admin | `{challenge_id, standard, signature, public_key?}` | `attestation_invalid`, `challenge_expired` | yes |
| `POST /v1/settlement_destinations/{id}/approve` | session owner/admin (≠ creator when 2 required) & step-up | `{decision}` | `approver_conflict` | yes |
| `POST /v1/settlement_destinations/{id}/set_default` · `/disable` | session owner/admin & step-up | — | `destination_not_active` | yes |
| `GET /v1/webhook_signing_keys` | anonymous | proxied by webhooks (not identity) | — | — |
| `GET /internal/authz` | Traefik only (network `internal`) | headers in → `200` + `X-NearPay-Token`, `X-Request-Id` / `401` / `403` / `429` | — | — |
| `GET /internal/jwks` | internal | JWKS | — | — |
| `POST /internal/service-tokens` | `client_id` + secret (Basic) | `{access_token, expires_in}` | `invalid_client` | — |
| `GET /internal/accounts/{id}/settlement-context?livemode=` | service (`identity.context.read`) | `{account:{id,status,live_enabled,kyb_status,display_name,verified_legal_name,fee_bearer,settings,api_version}, fee_schedule, destinations:[active…], defaults:{'*':sd_, <asset>:sd_}}` | `resource_missing` | — |
| `GET /internal/settlement-destinations/{id}` | service | destination (any status) | — | — |
| `GET /internal/accounts/{id}/notification-recipients?kind=security\|billing\|developer` | notify (`identity.recipients.read`) | `{recipients:[{user_id, email, role}]}` — security: owners+admins; developer: owners+admins+developers; billing: owners+finance | `resource_missing` | — |

**Scopes catalogue (restricted keys):** `checkout_sessions.read|write`, `payment_intents.read|write`, `refunds.read|write`, `payment_links.read|write`, `customers.read|write`, `invoices.read|write`, `subscriptions.read|write`, `products.read|write`, `webhook_endpoints.read|write`, `events.read`, `destinations.read`, `balance_transactions.read`, `agentpay.facilitate`. Secret keys hold all scopes except dashboard-only actions. Publishable keys: `checkout:public`, `elements:public`.

## 7. Events

| Produced subject | When | Consumers |
|---|---|---|
| `evt.identity.account.created` | signup completes | notify (welcome), compliance (create KYB placeholder) |
| `evt.identity.account.updated` | live_enabled, kyb_status, settings, status change | payments (context cache), webhooks (`account.updated`) |
| `evt.identity.settlement_destination.created` / `.cooling_off` / `.activated` / `.disabled` / `.default_changed` | transitions | payments (context cache), notify (security emails), webhooks (`settlement_destination.*`), sentinel (cache invalidation) |
| `evt.identity.api_key.created` / `.revoked` | key lifecycle | identity itself (cache purge across replicas), webhooks? no |
| `evt.identity.member.added` / `.removed` | team changes | notify |

| Consumed subject | Handler |
|---|---|
| `evt.compliance.kyb.approved` / `.rejected` / `.needs_info` | set `kyb_status`, recompute `live_enabled` |
| `evt.compliance.destination.blocked` | disable destination with reason `compliance` |
| `evt.compliance.account.suspended` / `.reinstated` | set account `status` |

## 8. Algorithms

### 8.1 `GET /internal/authz`
```
in: Authorization, X-NearPay-Client-Secret, Cookie, X-CSRF-Token, X-Forwarded-Method/Uri/For
1. route = match(X-Forwarded-Uri) against PUBLIC_ANON_ROUTES → if match and no credential: principal = anonymous
2. bearer sk_/rk_/pk_ → h = sha256(key); rec = lru.get(h) ?? valkey.get('ak:'+h) ?? dbReplica.apiKey(h); cache 30 s / 60 s
   reject if missing|revoked|expired → 401 invalid_api_key; ip_allowlist mismatch → 403
   pk_ keys allowed only on routes tagged publishable
3. cookie session → load (lru 10 s) → expired/revoked → 401; unsafe method requires X-CSRF-Token == csrf → else 403
   lm = header 'NearPay-Mode' == 'live' (default test); live requires account.live_enabled for unsafe methods on money routes
   console host (ops.*) → cookie '__Host-np_ops' and principal must be in ops_operators → ptype 'ops'
4. account = cache(acct) → status suspended & livemode & unsafe method → 403 account_suspended
5. rate limit: valkey token bucket key 'rl:'+principal (live 100 rps burst 200; test 25/50; anonymous 10/20 per IP) → 429 + Retry-After
   client IP = CF-Connecting-IP on the server stage (Cloudflare-only origin), X-Forwarded-For first hop from Traefik locally
   valkey down → per-pod bucket at 50% limits
6. jwt = sign({sub, acct, lm, ptype, scp, rid}, kid=current, exp=60 s)   // in-memory key
7. 200 headers: X-NearPay-Token, X-Request-Id (generate if absent); update last_used_at async (batched)
```
`api_key.revoked` events purge Valkey + broadcast purge of LRUs (NATS core subject `ctl.identity.cache.purge`).

### 8.2 Attestation verification
- Message (UTF-8, exact): `NearPay settlement destination\nAccount: acct_…\nMode: live\nAsset: <asset_id>\nNetwork: <caip2>\nAddress: <address>\nNonce: <hex32>\nExpires: <ISO>`.
- NEAR (named/implicit) & NEAR Intents account owned by a NEAR key: NEP-413 `signMessage({message, recipient:'nearpay.money', nonce})`; verify ed25519 over the NEP-413 borsh payload; for named accounts confirm the key is a **full-access key** of `address` via RPC `view_access_key` at final block; implicit accounts: `hex(pubkey) == address`.
- EVM addresses and `0x…` Intents accounts: EIP-191 `personal_sign`; `ecrecover == address`; ERC-1271 (`isValidSignature`) for contract wallets on the stated chain.
- Solana: ed25519 over message bytes; `base58(pubkey) == address`.
- Bitcoin: BIP-322 simple; other chain families → `unverified` path only.
- Challenge single-use, 10 min expiry; signature stored verbatim.

### 8.3 Settlement context (for payments)
Query account + fee schedule + active destinations (+ defaults) for `(account, livemode)`; response carries `context_version = max(updated_at)` so payments can cache and verify freshness.

## 9. Jobs

| Kind | Trigger | Behaviour |
|---|---|---|
| `destination.activate` | scheduled at `active_at` (dedupe `sd:<id>:activate`) | re-screen via compliance; activate or keep `cooling_off` with alert if screening unavailable (retry with backoff) |
| `purge.idempotency` / `purge.magic_links` / `purge.sessions` | hourly | delete expired rows |
| `keys.rotate_jwt` | daily | promote next signing key file (two keys mounted: current + next), publish JWKS, retire after 24 h |

## 10. Integrations

| Target | Call | Timeout | Retry | Failure outcome |
|---|---|---|---|---|
| compliance `POST /internal/screenings` | destination create/activate | 2 s | 2 | create → `503 screening_unavailable` (fail-closed); activate → stay `cooling_off`, retry |
| rails `GET /internal/destination-minimums` | list destinations | 1 s | 1 | show `min_payment_amount: null` |
| notify `POST /internal/messages` | magic links, security notices | 2 s | via outbox event instead where possible | magic link: `503` to user |
| NEAR RPC `view_access_key` | attestation | 2 s | 2 (pool) | `attestation_unverifiable` (retryable) |
| EVM RPC `eth_call` (ERC-1271) | attestation | 2 s | 2 | same |

## 11. Configuration

`NP_WEBAUTHN_RP_ID` (`nearpay.money` / `nearpay.localhost`), `NP_WEBAUTHN_ORIGINS`, `NP_JWT_KEYS_DIR` (current/next Ed25519 PEM files), `NP_COOLOFF_SECONDS` (86400; 0 allowed only when `NP_ENV=local`), `NP_RECOVERY_LOCK_SECONDS` (172800), `NP_TURNSTILE_SECRET_FILE`, `NP_PII_KEK_FILE`, `NP_PUBLIC_ANON_ROUTES` (file; v1 list: `POST /v1/auth/{signup,magic_links/verify,passkeys/*,recovery}`, `GET|POST /v1/checkout/sessions/{id}/public/**`, `POST /v1/checkout/sessions/{id}/agent_pay/**`, `GET|POST /v1/payment_links/{id}/public/**`, `GET /v1/invoices/{id}/public`, `POST /v1/attestations/deposit_instructions`, `GET /v1/attestations/keys`, `GET /v1/assets`, `GET /x402/supported`, `GET /v1/webhook_signing_keys`, `POST /webhooks/kyb/{vendor}`, `POST /webhooks/email/{provider}` — every other route requires a credential), `NP_RATE_LIMITS` (json), service client secrets file for each consumer service (`NP_SERVICE_CLIENTS_FILE`).

## 12. Security

- Passwords do not exist; passkeys + email magic link for signup/recovery only; recovery locks destination changes for 48 h and notifies all owners.
- Step-up (fresh WebAuthn assertion ≤ 10 min) for: key create/roll/revoke, destination create/approve/default/disable, member invite/remove, account settings that change money behaviour (`fee_bearer`, `late_payments`, tolerance).
- Destinations can never be changed by API keys (INV-1 practical).
- PII (`name`, IP) envelope-encrypted with the KEK file; emails stored plaintext (lookup key) but never logged.
- Audit every sensitive action into `audit_events` (append-only; daily export to object storage on server stage).

## 13. Observability

Metrics: `nearpay_identity_authz_duration_seconds{result}`, `…_authz_cache_hit_ratio`, `…_rate_limited_total{ptype}`, `…_destination_transitions_total{to}`, `…_attestation_failures_total{standard}`, `…_login_failures_total`. Alerts: authz p99 > 25 ms (5 min); destination creations > 20/h platform-wide (RB-06); login failure spike.

## 14. Failure modes

| Failure | Behaviour |
|---|---|
| Postgres primary down | authz keeps serving from LRU/Valkey/replica; writes `503` |
| Valkey down | authz uses LRU + DB; per-pod rate limits |
| JWT key file missing at boot | exit 1 (fail fast) |
| Compliance down | destination creation fails closed; activation waits |

## 15. Tests

- Unit: destination state table (every row), approvals rule, live-mode rule, key format/hash, CSRF.
- Property: random sequences of create/attest/approve/disable/activate never yield two active destinations with the same `(chain,address,asset)` in live mode; never `active` without screening allow and (signature or unverified cap).
- Attestation vectors: NEP-413 (named + implicit), EIP-191 EOA, ERC-1271 mock, Solana, BIP-322, wrong message, expired challenge, replayed challenge.
- Integration: authz for each `ptype`, revoked key propagation ≤ 30 s, rate limiter, recovery lock.
- Contract: settlement-context response vs `@nearpay-internal/spec`.

## 16. Stage mapping

| Stage | Scope |
|---|---|
| 1 | signup + passkeys, step-up, API keys (secret/publishable), destinations for NEAR/Intents/EVM with attestation, single-approver flow (cool-off configurable to 0 locally, 24 h on server), authz incl. mode header and ops principals, JWKS, service tokens, settlement context, notification recipients, test-mode destinations; live mode gated by the **manual** KYB decision from compliance (ops CLI) |
| 2 | team members & invitations, two-approver rule, restricted keys, recovery flow, KYB-driven live mode, audit export |
| 4 | Solana/Bitcoin attestation |
