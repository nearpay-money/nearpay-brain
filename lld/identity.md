# LLD — `nearpay-identity`

> Version 1.13 (step 4.2 implementation, spec 0.4.0 (Phase A on 0.3.1) / kit 0.3.1: Solana/BIP-322 verifier decisions — strict ed25519, small-order keys refused, off-chain/off-curve hints, canonical BIP-322 encodings, Bitcoin type from kit-validated address, migration `0008_bitcoin_destination_types` — §20 "Implementation decisions") · Version 1.12 (step 4.1, spec 0.4.0: Solana and BIP-322 ownership attestations — formats, verification rules, Bitcoin destination address types, vectors — §8.2, §20) · Version 1.11 (step 3.2 implementation, spec 0.3.0 / kit 0.2.1: CLI login decisions — migration `0007_cli_logins`, IP envelope-encrypted, login IP on the audit row (`origin_ip_enc`), uncollected keys revoked after 15 min, poll jitter 1 s, `verification_url_complete` = `…?code=`, visibility of decided logins — §6.2, §9, §16, §19) · Version 1.10 (step 3.1, spec 0.3.0: CLI login — device authorization minting a restricted test key §6.2, anonymous allow-list §11: CLI login and hosted invoice routes) · Version 1.9 (spec 0.2.1 implementation: per-user recovery columns, recipients by user including recently removed members, branding validation and defaults — §"Spec 0.2.1 contract changes") · Spec 0.2.1 update (spec 0.2.1: recovery event, operator read for compliance, recipients by user, website/theme/corner radius, `created_by`, member security, `expired` disabled reason, `recovery_lock` declared — §"Spec 0.2.1 contract changes") · Version 1.8 (stage 2 audit fixes: two-approver rule re-derived at every step with a 30-day manager high-water mark, manager tenure via `manager_grants` (7 d / 30 d), pending destinations expire after 7 days, recovery lock covers owner/admin team changes, relaxed behaviour only for `NP_ENV` local/test — §5, §9, §11, §12, §18 "Audit fixes", migration 0005) · Version 1.7 (step 2.2 implementation, spec 0.2.0 / kit 0.2.0: decisions in §18 — creator approval row + trigger, recovery lock on owned and administered accounts, invite links for existing users, KYB name lifecycle, console key policy, migration 0004) · Version 1.6 (step 2.1 integration, spec 0.2.0: unified console audit trail — identity consumes `evt.compliance.ops_action.recorded` / `evt.sentinel.ops_action.recorded` into `audit_events` §6.1/§7; `kyb_verified` / `verified_legal_name` keep their names, meaning "self-declared, ops-approved" §4) · Version 1.5 (step 2.1 contracts: team, invitations, multi-account sessions, recovery, restricted keys, two-approver rule, audit export, console sessions and console routes, self-declared KYB semantics per ADR-0020 — §1, §4, §5, §6, §6.1, §7, §8.1, §11, §12, §16, §17) · Version 1.4 (step 1.4, spec 0.1.5: `GET /v1/auth/session` and `POST /v1/settlement_destinations/{id}/challenge` §6, `challenge` command §5, `attestation_challenges.superseded_at` §4) · Version 1.3 (step 1.2 implementation findings folded: sessions and sign-up §4/§6, authz §8.1, jobs §9, configuration §11, ops CLI §17) · Version 1.1 audited (`audits/identity.md`) · HLD §5.1, ADR-0005, ADR-0014 · Foundations: `00-foundations.md`

## 1. Scope

**Owns:** accounts, users, memberships/roles, dashboard sessions (passkeys, step-up, recovery), API keys (secret, publishable, restricted), settlement destinations (attestation, approvals, cool-off, defaults), fee schedules, gateway authorisation (`/internal/authz`), internal JWT/JWKS, service tokens, account audit log; from step 2.1 also invitations, console operators and console sessions, and the console's merchant and audit-log reads (§6.1).
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
  -- verified_legal_name: set from the KYB profile when compliance approves (step 2.1, getInternalKybCase); it is the
  -- ops-approved, self-declared legal name (ADR-0020), not a third-party-verified identity
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
  aaguid uuid, name text not null, created_at timestamptz not null default now(), last_used_at timestamptz,
  kind text not null default 'dashboard' check (kind in ('dashboard','ops')));   -- step 2.1: ops = hardware key registered
  -- through operator enrolment (attestation verified, AAGUID on NP_OPS_AAGUID_ALLOWLIST); only ops passkeys sign in to the console

create table sessions (
  id uuid primary key, kind text not null default 'dashboard' check (kind in ('dashboard','ops')),
  user_id uuid not null references users(id), account_id uuid references accounts(id),   -- null for ops sessions
  token_hash bytea not null unique, csrf_hash bytea not null,
  csrf_enc bytea not null,                          -- envelope-encrypted copy: the CSRF token is re-issued with the session object
  step_up_at timestamptz,
  created_at timestamptz not null default now(), expires_at timestamptz not null, last_seen_at timestamptz not null,
  revoked_at timestamptz, ip_enc bytea, user_agent text,
  check ((kind = 'dashboard') = (account_id is not null)));
create index sessions_user on sessions (user_id) where revoked_at is null;

create table magic_links (
  token_hash bytea primary key, email citext not null, purpose text not null check (purpose in ('signup','recovery','invite','ops_enrol')),
  account_id uuid, role text, expires_at timestamptz not null, used_at timestamptz,
  invitation_id uuid);                              -- step 2.1: invite links (7 days); ops_enrol links (24 h, role = ops role)

create table invitations (                          -- step 2.1 (inv_)
  id uuid primary key, account_id uuid not null references accounts(id), email citext not null,
  role text not null check (role in ('owner','admin','developer','finance','support','viewer')),
  status text not null check (status in ('pending','accepted','revoked','expired')),
  invited_by uuid not null, expires_at timestamptz not null, accepted_at timestamptz, accepted_by uuid,
  created_at timestamptz not null default now());
create unique index invitations_one_pending on invitations (account_id, email) where status = 'pending';
-- memberships gain last_used_at timestamptz (sign-in picks the most recently used account; switchSessionAccount sets it)

create table registration_tokens (                  -- verified email → passkey registration (5 min, single use)
  token_hash bytea primary key, email citext not null, user_handle bytea not null,  -- WebAuthn user.id of the future user
  expires_at timestamptz not null, used_at timestamptz, created_at timestamptz not null default now());

create table webauthn_challenges (                  -- one row per ceremony; single use
  challenge text primary key, purpose text not null check (purpose in ('registration','authentication','step_up')),
  registration_token_hash bytea, user_id uuid, session_id uuid, host text,   -- dashboard or ops host the ceremony started on
  expires_at timestamptz not null, used_at timestamptz, created_at timestamptz not null default now());

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
  created_at timestamptz not null default now(), updated_at timestamptz not null default now());
-- the migration seeds schedule 'standard' with the Q-BIZ-02 defaults: same_asset 20, stable_pair 50, volatile 100,
-- intents_transfer 30 bps with min_fee_usd_micros 1000 (USD 0.001); the others have no minimum

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
  message text not null, nonce bytea not null, expires_at timestamptz not null, used_at timestamptz,
  superseded_at timestamptz);   -- migration 0003: set when a re-issued challenge replaces this unused one (kept apart from used_at)

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

-- step 2.1: `kyb_verified` below means "legal name approved by NearPay ops from self-declared details" (ADR-0020 §3). Clients
-- must not present it (or verified_legal_name) as third-party verification: the checkout shows "Business details reviewed by NearPay".
create view sentinel_destination_registry as        -- granted SELECT to role sentinel_ro only (v2, migration 0002)
  select d.id, d.account_id, d.livemode, d.asset_id, d.type, d.chain, d.address_norm, d.status, d.active_at,
         coalesce(a.verified_legal_name, a.display_name) as merchant_name, a.verified_legal_name is not null as kyb_verified,
         d.verification, d.disabled_at
  from settlement_destinations d join accounts a on a.id = d.account_id
  where d.status = 'active' or (d.status = 'disabled' and d.active_at is not null);
```

Registry view v2: `verification` lets sentinel R5 prefer the single signature-verified row; destinations that were active once and are now disabled are listed with `disabled_at`, so sentinel's audit can check that a settlement reached a destination active when the execution was created, and the co-signer can answer `stale` (never a security breach) for a recipient whose destination was disabled while a checkout was open (sentinel §5, §7). The co-signer filters `status = 'active'` itself. Columns are only appended (a replaced view may only add columns at the end); pending rows are never listed (contract test pins the columns and the filter).

Migration note: `account_default_destinations` uses a generated column `coalesce_asset text generated always as (coalesce(asset_id,'*')) stored` in the PK.

**Key formats:** `sk_live_` / `sk_test_` / `pk_live_` / `pk_test_` / `rk_live_` / `rk_test_` + 48 base62 chars (≈ 285 bits). Shown once. Stored as SHA-256. Session cookie `__Host-np_session` (console: `__Host-np_ops`) = 32 random bytes base64url (hash stored), `SameSite=Strict; Secure; HttpOnly; Path=/`, TTL 12 h sliding, absolute 7 days. The cookie name decides the session kind: a dashboard token presented as a console cookie is no session.

## 5. State machines

**SettlementDestination**

| From | Event / command | Guard | To | Side effects |
|---|---|---|---|---|
| — | `create` (session, step-up ≤ 10 min, role owner/admin) | address valid for chain; `recovered_at` older than 48 h; screening `allow` (sync); unique constraint; test mode → skip to `active` with `verification='test'` | `pending_verification` | challenge (10 min), audit, event `created` |
| `pending_verification` | `challenge` (session owner/admin, step-up) | — | `pending_verification` | fresh challenge (10 min); earlier unused challenges superseded in the same tx; audit `destination.challenge_issued`; no transition row or event |
| `pending_verification` | `attest` | signature valid for message & key controls address (§8.2) | `pending_approval` or `cooling_off` | store attestation |
| `pending_verification` | `mark_unverified` (owner, step-up) | chain family has no message signing **or** owner opts in; no live non-disabled row of another account has the same `(chain,address,asset)` | `pending_approval`/`cooling_off` | cap = 1,000 USD per payment |
| `pending_approval` | `approve` (distinct admin/owner, step-up) | qualifying approvals ≥ required (§5 "Two-approver rule"; v1.8) | `cooling_off` | `active_at = now() + 24 h`; notify all owners/admins; job `destination.activate` |
| `cooling_off` | job `destination.activate` at `active_at` | not disabled; re-screen `allow`; approvals still complete on today's team (v1.8) | `active` | event `activated`; if first destination for the asset → set default |
| `cooling_off` | job `destination.activate`, approvals no longer complete (v1.8) | — | `pending_approval` | `active_at` cleared; event `pending_approval`; audit `destination.approval_required` |
| `pending_verification` / `pending_approval` | job `destination.expire`, 7 days after creation (v1.8) | — | `disabled` (`merchant`, cause `expired`) | audit `destination.expired`; attest/approve/challenge answer `409 state_conflict` |
| any non-`disabled` | `disable` (owner/admin, step-up) or `compliance.destination.blocked` | — | `disabled` | if default → unset default; event `disabled`; notify |

**Account live mode:** `live_enabled = (kyb_status = 'approved') and exists(active live destination) and status = 'active'`; recomputed on every relevant event; change emits `identity.account.updated`. From step 2.1 `kyb_status` follows the compliance events including `kyb.submitted` (`pending`); `approved` comes only from an ops decision on self-declared details (ADR-0020); on `kyb.approved` identity reads the case (`GET /internal/kyb_cases/{acct}`, `compliance.read`) and copies `legal_name` → `verified_legal_name` and `country_of_incorporation` → `country`.

**Two-approver rule (step 2.1).** `approvals_required` is fixed at creation: `2` when the account has ≥ 2 owner/admin members, else `1` (the creator's own creation counts). `POST /v1/settlement_destinations/{id}/approve` is checked server-side on the session's user id: with `2` required the approver must differ from the creator and from earlier approvers (`403 approver_conflict`); approvals are rows of `destination_approvals` (primary key `(destination_id, user_id)` repeats the rule); a `reject` disables the destination (`disabled_reason: merchant`). Creating a destination that needs a second approval emits `settlement_destination.pending_approval` (notify asks the other owners/admins). The public object shows `approval {required, approved_by, rejected_by}`. **v1.8 (audit fix):** the requirement is re-derived at attestation, approval and activation as the greatest of the stored value, `2` when ≥ 2 users hold owner/admin now, and `2` when ≥ 2 users held owner/admin at any time in the last 30 days (high-water mark; the stored value only rises). With `2` required an approval — the creator's included — counts only from a user who holds owner/admin now under a grant that predates the destination's `created_at`, has reached its `qualifies_at` and is older than the cool-off. `qualifies_at` = grant + 7 days, or + 30 days when ≥ 2 other users held owner/admin in the 30 days before the grant (a manager added next to existing ones cannot vote for a month); owner ↔ admin keeps the grant, leaving the roles closes it, coming back opens a new one (`manager_grants`, written by a trigger on `memberships`). Rejections need no tenure. Live destinations still pending 7 days after creation are disabled. Effect: one person — any number of sessions and invited users — cannot activate a destination on an account that had ≥ 2 managers in the last 30 days; the other managers are notified of every owner/admin change (`member.*` → notify) and of the pending destination. Cost: a new two-founder team waits 7 days after the second founder joined before its first live destination can be approved; a team that replaces a manager waits 30 days.

**Invitation** (step 2.1): `pending` → `accepted` (verifyMagicLink + passkey registration for a new user, or `acceptInvitation` by the signed-in user whose email matches; emits `member.added`) · `revoked` (owner/admin) · `expired` (7 days, job `purge.auth` marks it). One pending invitation per (account, email) (`409 member_exists`, also when already a member). Only owners invite, promote or remove owners; the last owner can be neither demoted nor removed (`409 last_owner`). Removing a member revokes their sessions for that account at once (cache purge as for logout).

**Recovery** (step 2.1): `POST /v1/auth/recovery` → magic link (`purpose: recovery`) → `verifyMagicLink` (`Registration.purpose = recovery`) → passkey registration adds a passkey to the existing user (existing passkeys stay) and creates a session; every account the user owns gets `recovered_at = now()` (48 h destination lock) and all its owners are notified (`security.member_changed`-style recovery notice, audit `user.recovered`). v1.8: the 48 h lock also refuses inviting an owner/admin and promoting, demoting or removing one (`403 recovery_lock`, message "Owners and admins cannot change until …").

## 6. API

All public routes follow `api-conventions.md`; dashboard routes are session-only (`ptype=session`), reached same-origin via `dashboard.nearpay.<tld>/api`.

| Method & path | Auth / role | Request → Response | Errors | Idem. |
|---|---|---|---|---|
| `POST /v1/auth/signup` | anonymous (Turnstile) | `{email}` → `202`: a single-use magic link (15 min) `<NP_DASHBOARD_ORIGIN>/auth/verify#token=…` (token in the fragment, never in a server log); at most 5 links per address per hour; an address that already has a user gets the same `202` and **no** mail (no enumeration) | `rate_limited` | — |
| `POST /v1/auth/magic_links/verify` | anonymous | `{token}` → `{registration_token}` (5 min) | `link_expired`, `link_used` | — |
| `POST /v1/auth/passkeys/registration/options` · `/verify` | registration_token or session | WebAuthn options / attestation (discoverable credential, user verification required, ES256/EdDSA/RS256) → session cookie (new user: creates user + account `owner` + session, emits `account.created`); adding a passkey to a session re-issues the session object with its CSRF token (from `csrf_enc`) | `webauthn_failed` | — |
| `POST /v1/auth/passkeys/authentication/options` · `/verify` | anonymous | assertion of a discoverable credential, no email asked (no enumeration) → session cookie + CSRF token; signing in counts as a step-up (`step_up_until` = sign-in + 10 min) | `webauthn_failed` | — |
| `POST /v1/auth/step_up/options` · `/verify` | session | assertion → `{step_up_until}` (10 min); any step-up-protected endpoint called without a fresh step-up returns `401 step_up_required` | `webauthn_failed` | — |
| `POST /v1/auth/recovery` | anonymous (Turnstile) | `{email}` → `202`; link → new passkey, sets `accounts.recovered_at` | — | — |
| `GET /v1/auth/session` | session cookie, any role (API keys `403`) | → the current `DashboardSession` with its CSRF token, re-issued from `csrf_enc` (never rotated, so open tabs keep working); sets no cookie (the sliding TTL applies as for any session request). The handler re-checks that the cookie names a valid (not revoked, expired or idle) dashboard session of the request principal's user and account, else `401 authentication_required` (also when the membership was removed) — spec 0.1.5 | `authentication_required` | — |
| `POST /v1/auth/logout` | session | `204` | — | — |
| `GET /v1/account` · `PATCH /v1/account` | session (PATCH: owner/admin) | account object (settings whitelist) | `invalid_setting` | yes |
| `GET /v1/account/requirements` | session | `{live_enabled, missing:['kyb','destination']}` | — | — |
| `GET /v1/account/members` · `PATCH /{user}` · `DELETE /{user}` (`listMembers`, `updateMember`, `removeMember`, step 2.1) | session owner/admin (+step-up for PATCH/DELETE) | `Member {user{id,email}, role, created, last_login_at}` | `last_owner` | yes |
| `GET` · `POST /v1/account/invitations` · `DELETE /{inv}` (`listInvitations`, `createInvitation`, `revokeInvitation`) | session owner/admin (+step-up for POST) | `{email, role}` → `Invitation`; the link is mailed through notify `team.invitation` (token only in the link fragment) | `member_exists`, `state_conflict` | yes |
| `POST /v1/auth/invitations/accept` (`acceptInvitation`) | session (the invited user) | `{token}` → `DashboardSession` re-issued for the joined account | `invitation_invalid` | — |
| `GET /v1/auth/accounts` · `POST /v1/auth/session/account` (`listMemberships`, `switchSessionAccount`) | session | memberships / `{account}` → `DashboardSession` for that account (step-up cleared) | — | — |
| `PATCH /v1/api_keys/{id}` (`updateApiKey`) | session developer+ & step-up | `{name?, scopes?, ip_allowlist?}`; `scopes` only for restricted keys | `scope_unknown`, `state_conflict` | yes |
| `GET /v1/account/audit_events` (`listAccountAuditEvents`) | session owner/admin | cursor list of `AuditEvent` (`aud_`), filters `created[…]`, `action`, `actor`; the dashboard's audit export walks it and writes CSV client-side (no server-side file export; the platform's daily object-storage export is an ops job) | — | — |
| `GET /v1/api_keys` | session developer+ | list (no secrets) | — | — |
| `POST /v1/api_keys` | session developer+ & step-up | `{type, livemode, name, scopes?, ip_allowlist?, expires_at?}` → key **with secret once**; step 2.1: `type: restricted` (`rk_live_`/`rk_test_`) needs ≥ 1 scope from the catalogue below (`<resource>.read\|write`, write implies read), other types refuse `scopes` | `scope_unknown` | yes |
| `POST /v1/api_keys/{id}/roll` | session developer+ & step-up | `{expire_old_in_seconds ≤ 7 d}` → new key | — | yes |
| `DELETE /v1/api_keys/{id}` | session developer+ & step-up | `204` | — | — |
| `GET /v1/settlement_destinations` | session any role; secret key read-only (`destinations.read`) | list incl. `min_payment_amount` (from rails, cached 1 h) | — | — |
| `POST /v1/settlement_destinations` | session owner/admin & step-up | `{asset_id, type, chain, address, label}` → destination + `attestation_challenge {id, message, expires_at, signing_standard}` | `address_invalid`, `destination_in_use`, `recovery_lock`, `screening_blocked`, `memo_destination_unsupported`, `asset_unsupported` | yes |
| `POST /v1/settlement_destinations/{id}/challenge` | session owner/admin & step-up | → destination + a fresh `attestation_challenge` (same helper and shape as on create) while `pending_verification`; earlier unused challenges are superseded in the same transaction; audited (`destination.challenge_issued`, number superseded); an idempotent replay returns the same challenge and supersedes nothing — spec 0.1.5 | `state_conflict` (any other status) | yes |
| `POST /v1/settlement_destinations/{id}/attest` | session owner/admin | `{challenge_id, standard, signature, public_key?}`; a used, superseded or expired challenge is refused (checked again in the locking transaction) | `attestation_invalid`, `challenge_expired` ("…request a new challenge for the destination.") | yes |
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

### 6.1 Console routes (step 2.1, spec `OPS_ROUTES`, service `identity`)

Visibility `ops`, paths `/ops/…`, reached only through the console host (§8.1). `opsRoles` `ops`+`ops_admin`.

| operationId | Method & path | Auth | Notes |
|---|---|---|---|
| `opsEnrolmentVerify` | `POST /ops/auth/enrolment/verify` | anonymous | `{token}` (from `operator-add`) → `OpsRegistration {registration_token}`; `link_expired`, `link_used` |
| `opsPasskeyRegistrationOptions` · `opsPasskeyRegistrationVerify` | `POST /ops/auth/passkeys/registration/options` · `/verify` | anonymous + registration token | attestation `direct`, cross-platform, discoverable, UV required; AAGUID must be allow-listed (`403 authenticator_not_allowed`) → `OpsSession` + `__Host-np_ops` |
| `opsPasskeyAuthenticationOptions` · `opsPasskeyAuthenticationVerify` | `POST /ops/auth/passkeys/authentication/options` · `/verify` | anonymous | only `kind='ops'` passkeys of active operators; every refusal `401 webauthn_failed`; sign-in counts as step-up |
| `opsStepUpOptions` · `opsStepUpVerify` · `getOpsSession` · `opsLogout` | `POST /ops/auth/step_up/options` · `/verify` · `GET /ops/auth/session` · `POST /ops/auth/logout` | `ops:console.session` | `OpsSession {operator, role, csrf_token, step_up_until, idle_expires_at, expires_at}` |
| `listOpsMerchants` · `getOpsMerchant` | `GET /ops/merchants` · `/{acct}` | `ops:identity.merchants.read` | `q` (exact `acct_` id or member email, name prefix), `status`, `kyb_status`, `live_enabled`; `OpsMerchant` = `Account` + owners, member count, active live destinations, `recovered_at` |
| `listOpsAuditEvents` | `GET /ops/audit_events` | `ops:identity.audit.read` | platform audit log (`AuditEvent`), filters account, actor, actor type, action, target, created |

Every console write (here and in the other services) is audited with `actor_type = operator`, `actor_id = usr_…`. Identity's own operator actions (sign-in, enrolment, step-up) are audited in `audit_events` directly. **Unified audit trail (T-12, lead decision at step 2.1 integration):** compliance and sentinel publish every operator action as `evt.compliance.ops_action.recorded` / `evt.sentinel.ops_action.recorded` (`OpsActionEvent`, ids only); identity's consumer inserts one `audit_events` row per event (`id` = the event's action id, so redelivery is a no-op; `actor_type = operator`, `actor_id` = `actor.sub`, `details` = `{ops_role, approval, source: compliance|sentinel}`), so `listOpsAuditEvents` is one trail across services. Their own append-only logs (case events, `ops_approvals`, `kill_switch_log`) stay the detail source on the respective console pages. Payments' and rails' console writes (reconciliation, reverify) are audited locally for now; publishing them the same way is a follow-up (open question for the lead).

### 6.2 CLI login (Stage 3, spec 0.3.0 `src/public/cli.ts`)

`nearpay login` (sdks-and-tools §3) uses a device-authorization flow in the RFC 8628 shape; the result is an **ordinary restricted test key**, not a new session type, so the gateway, authz, rate limits, listing (`GET /v1/api_keys`) and revocation need nothing new.

| Route | Auth | Behaviour |
|---|---|---|
| `POST /v1/cli/logins` (`createCliLogin`) | anonymous (allow-list §11) | `{device_name, client_version?}` → `201 CliLogin {device_code npdc_…, user_code XXXX-XXXX, verification_url, verification_url_complete, expires_at (+15 min), interval 5}`. Rate limit 10 per 10 min per client IP. Stores sha256(device_code), the user code (unique among live logins), IP, `CF-IPCountry`, user agent |
| `POST /v1/cli/logins/poll` (`pollCliLogin`) | anonymous | `{device_code}` → `CliLoginPoll {status, interval, key?, account?}`. Faster than `interval` → `429 rate_limited` + `Retry-After` (`slow_down`); unknown code → `404`. The first poll that sees `approved` receives the key **with its secret** and moves the login to `redeemed` in the same transaction (the secret is encrypted at rest with `NP_PII_KEK_FILE` between approval and that poll, then erased) |
| `GET /v1/cli/logins/{code}` (`getCliLogin`) | session owner/admin/developer | `CliLoginRequest` (device name, where it came from, status). Logins approved for another account and logins expired > 24 h → `404` |
| `POST /v1/cli/logins/{code}/approve` (`approveCliLogin`) | same + **step-up** | mints a `restricted` key in **test mode** for the session's account regardless of `NearPay-Mode`: scopes `CLI_KEY_SCOPES`, name `CLI · <device_name>`, `expires_at` = +90 days; audit `api_key.created` (`via: cli_login`, login IP); not `pending` → `409 state_conflict` |
| `POST /v1/cli/logins/{code}/deny` (`denyCliLogin`) | session owner/admin/developer | `denied`; not `pending` → `409 state_conflict` |

Data (migration `0007_cli_logins` — 0006 was taken by spec 0.2.1; v1.11): `cli_logins (id uuid pk, device_code_sha256 bytea unique, user_code text, device_name, client_version, ip_enc bytea (envelope-encrypted, §12; was `ip inet`), country, user_agent, status check in (pending, approved, denied, expired, redeemed), account_id, key_id, key_secret_enc bytea, approved_by, decided_at, created_at, expires_at, last_polled_at, updated_at)` with a unique index on `user_code` where `status = 'pending'`; job `cli_logins.expire` (1 min) moves overdue `pending` rows to `expired` and expires approvals whose key was not collected within 15 min (key revoked, §19), purge after 7 days (`purge.auth`). `verification_url` = `NP_DASHBOARD_ORIGIN` + `/cli/authorize`; the dashboard page (Stage 3 dashboard) shows `getCliLogin` and calls approve/deny. Phishing guard: the page shows the device name, IP/country and creation time and asks the user to compare the code with the terminal.

## 7. Events

| Produced subject | When | Consumers |
|---|---|---|
| `evt.identity.account.created` | signup completes | notify (welcome), compliance (create KYB placeholder) |
| `evt.identity.account.updated` | live_enabled, kyb_status, settings, status change | payments (context cache), webhooks (`account.updated`) |
| `evt.identity.settlement_destination.created` / `.cooling_off` / `.activated` / `.disabled` / `.default_changed` | transitions | payments (context cache), notify (security emails), webhooks (`settlement_destination.*`), sentinel (cache invalidation) |
| `evt.identity.api_key.created` / `.revoked` | key lifecycle | identity itself (cache purge across replicas), webhooks? no |
| `evt.identity.member.added` / `.removed` / `.role_changed` (step 2.1, `MemberEvent`: ids and roles only) | team changes | notify (`security.member_changed`) |
| `evt.identity.settlement_destination.pending_approval` (step 2.1) | a destination waits for a second approver | notify (other owners/admins) |

| Consumed subject | Handler |
|---|---|
| `evt.compliance.kyb.approved` / `.rejected` / `.needs_info` | set `kyb_status`, recompute `live_enabled`; on `approved` read the case and set `verified_legal_name`, `country` (step 2.1) |
| `evt.compliance.kyb.submitted` (step 2.1) | set `kyb_status = pending` |
| `evt.compliance.destination.blocked` | disable destination with reason `compliance` |
| `evt.compliance.account.suspended` / `.reinstated` | set account `status` |
| `evt.compliance.ops_action.recorded` · `evt.sentinel.ops_action.recorded` · `evt.payments.ops_action.recorded` · `evt.rails.ops_action.recorded` (step 2.1 integration) | insert into `audit_events` (one console audit trail, §6.1); idempotent on the action id |

## 8. Algorithms

### 8.1 `GET /internal/authz`
```
in: Authorization, X-NearPay-Client-Secret, Cookie, X-CSRF-Token, X-Forwarded-Method/Uri/For
1. route = match(X-Forwarded-Uri) against PUBLIC_ANON_ROUTES → if match and no credential: principal = anonymous
2. bearer sk_/rk_/pk_ → h = sha256(key); rec = lru.get(h) ?? valkey.get('ak:'+h) ?? dbReplica.apiKey(h); cache 30 s / 60 s
   reject if missing|revoked|expired → 401 invalid_api_key; ip_allowlist mismatch → 403
   pk_ keys allowed only on routes tagged publishable
3. cookie session → load (lru 10 s) → expired/revoked → 401; unsafe method requires X-CSRF-Token == csrf → else 403
   lm = header 'NearPay-Mode' == 'live' (default test); live requires account.live_enabled on money routes, i.e. the routes whose
   spec definition declares the error `account_not_live` (derived from the route table, not a hand-kept list) → 403 account_not_live
   console host (ops.*) → cookie '__Host-np_ops' and principal must be in ops_operators → ptype 'ops'
   (authz accepts console sessions; issuing them arrives with the console in Stage 2 — Stage 1 has no ops sign-in route, ops work runs through the ops CLIs)
   step 2.1: on the console host only `/ops/*` is routed; anonymous allow-list = spec `OPS_ANONYMOUS_ROUTES` (enrolment + passkey
   sign-in); every other `/ops/*` route needs an ops session (kind 'ops', not revoked, idle ≤ 30 min, absolute ≤ 12 h — spec
   OPS_SESSION_TTL_SECONDS) whose user has an active ops_operators row; the dashboard cookie is ignored there and the ops cookie
   is ignored on every other host; CSRF as for dashboard sessions; claims: sub = usr id, ptype 'ops', ops_role, scp =
   OPS_ROLE_SCOPES[role], step_up_until; no acct/lm (ops routes take account and mode as parameters)
4. account = cache(acct) → status suspended & livemode & unsafe method → 403 account_suspended
5. rate limit: valkey token bucket key 'rl:'+principal (live 100 rps burst 200; test 25/50; anonymous 10/20 per IP) → 429 + Retry-After
   client IP = CF-Connecting-IP on the server stage (Cloudflare-only origin), X-Forwarded-For first hop from Traefik locally
   valkey down → per-pod bucket at 50% limits
6. jwt = sign({sub, acct, lm, ptype, scp, rid}, kid=current, exp=60 s)   // in-memory key
7. 200 headers: X-NearPay-Token, X-Request-Id (generate if absent); update last_used_at async (batched)
```
`api_key.revoked` events purge Valkey + broadcast purge of LRUs (NATS core subject `ctl.identity.cache.purge`); step-ups and logouts purge the session the same way. The api's NATS connection is optional: without it the LRU TTLs bound staleness (keys 30 s, sessions 10 s).

### 8.2 Attestation verification
- Message (UTF-8, exact): `NearPay settlement destination\nAccount: acct_…\nMode: live\nAsset: <asset_id>\nNetwork: <caip2>\nAddress: <address>\nNonce: <hex32>\nExpires: <ISO>`.
- NEAR (named/implicit) & NEAR Intents account owned by a NEAR key: NEP-413 `signMessage({message, recipient:'nearpay.money', nonce})`; verify ed25519 over the NEP-413 borsh payload; for named accounts confirm the key is a **full-access key** of `address` via RPC `view_access_key` at final block; implicit accounts: `hex(pubkey) == address`.
- EVM addresses and `0x…` Intents accounts: EIP-191 `personal_sign`; `ecrecover == address`; ERC-1271 (`isValidSignature`) for contract wallets on the stated chain.
- Solana: ed25519 over message bytes; `base58(pubkey) == address`. Stage 4 details: §20.
- Bitcoin: BIP-322 simple (P2WPKH and P2TR destinations only); other chain families → `unverified` path only. Stage 4 details: §20.
- Stage 1 verifies NEP-413 and EIP-191 (+ ERC-1271) only: a **live** destination whose chain needs Solana or BIP-322 attestation is refused with `400 invalid_request` (`param: chain`) until Stage 4.
- Challenge single-use, 10 min expiry; signature stored verbatim.

### 8.3 Settlement context (for payments)
Query account + fee schedule + active destinations (+ defaults) for `(account, livemode)`; response carries `context_version = max(updated_at)` so payments can cache and verify freshness.

## 9. Jobs

| Kind | Trigger | Behaviour |
|---|---|---|
| `destination.activate` | scheduled at `active_at` (dedupe `sd:<id>:activate`) | re-screen via compliance; activate or keep `cooling_off` with alert if screening unavailable (retry with backoff) |
| `destination.expire` (v1.8) | hourly (+20 min offset) | disable live destinations still `pending_verification`/`pending_approval` 7 days after creation |
| `purge.auth` | hourly (+10 min offset) | delete magic links, registration tokens and WebAuthn challenges expired > 1 day, unused attestation challenges expired > 30 days, sessions expired > 30 days |
| `kit.purge` | hourly | published outbox, old inbox, expired idempotency keys (foundations §7) |
| `cli_logins.expire` (Stage 3) | every minute | overdue `pending` CLI logins → `expired`; `approved` logins not collected within 15 min → `expired`, key revoked (`api_key.revoked` event + audit, actor `system`), secret erased (§6.2, §19); `purge.auth` deletes settled logins 7 days after creation |
| `keys.rotate_jwt` | operator action in Stage 1 (no job) | both mounted keys (current + next) are published in the JWKS and `current` signs; promoting `next` is a secret swap + restart (platform runbook) |

## 10. Integrations

| Target | Call | Timeout | Retry | Failure outcome |
|---|---|---|---|---|
| compliance `POST /internal/screenings` | destination create/activate | 2 s | 2 | create → `503 screening_unavailable` (fail-closed); activate → stay `cooling_off`, retry |
| rails `GET /internal/destination-minimums` | list destinations | 1 s | 1 | show `min_payment_amount: null` |
| notify `POST /internal/messages` | magic links, security notices | 2 s | via outbox event instead where possible | magic link: `503` to user |
| NEAR RPC `view_access_key` | attestation | 2 s | 2 (pool) | `attestation_unverifiable` (retryable) |
| EVM RPC `eth_call` (ERC-1271) | attestation | 2 s | 2 | same |

## 11. Configuration

`NP_WEBAUTHN_RP_ID` (`nearpay.money` / `nearpay.localhost`), `NP_WEBAUTHN_ORIGINS`, step 2.1: `NP_OPS_ORIGIN` (console origin, enrolment links and ops WebAuthn origin), `NP_OPS_AAGUID_ALLOWLIST` (file of hardware-key AAGUIDs accepted for ops passkeys; local stacks may add the CDP virtual authenticator's AAGUID only when `NP_ENV` is `local`/`test`), `NP_JWT_KEYS_DIR` (current/next Ed25519 PEM files), `NP_COOLOFF_SECONDS` (86400; 0 allowed only when `NP_ENV=local`), `NP_RECOVERY_LOCK_SECONDS` (172800), `NP_TURNSTILE_SECRET_FILE`, `NP_PII_KEK_FILE`, `NP_PUBLIC_ANON_ROUTES` (file; v1 list: `POST /v1/auth/{signup,magic_links/verify,passkeys/*,recovery}`, `GET|POST /v1/checkout/sessions/{id}/public/**`, `POST /v1/checkout/sessions/{id}/agent_pay/**`, `GET|POST /v1/payment_links/{id}/public/**`, `GET|POST /v1/invoices/{id}/public/**` (Stage 3: `GET …/public`, `POST …/public/sessions`, `GET …/public/pdf`; billing checks `X-NearPay-Invoice-Secret`), `POST /v1/attestations/deposit_instructions`, `GET /v1/attestations/keys`, `GET /v1/assets`, `GET /x402/supported`, `GET /v1/webhook_signing_keys`, `POST /v1/cli/logins` and `POST /v1/cli/logins/poll` (Stage 3, §6.2), `POST /webhooks/kyb/{vendor}`, `POST /webhooks/email/{provider}` — every other route requires a credential), `NP_RATE_LIMITS` (json), service client secrets file for each consumer service (`NP_SERVICE_CLIENTS_FILE`).

Step 1.2 additions: `NP_DASHBOARD_ORIGIN` (magic-link target), `NP_COMPLIANCE_URL`, `NP_RAILS_URL`, `NP_NOTIFY_URL` (peers), `NP_NEAR_RPC_URLS` (NEAR `view_access_key` pool), `NP_EVM_RPC_URLS` (`{"<chain id>": [urls]}` for ERC-1271), `NP_VALKEY_URL` (key cache, rate limits; down → per-pod limits at half rate), `NP_NATS_URL` (worker: events; api, optional: cache purge). Cloudflare's Turnstile test secrets are answered locally only when `NP_ENV` is `local` or `test` (v1.8: every relaxed path — Turnstile test secrets, `X-Forwarded-For` client IPs, zero cool-off, virtual AAGUIDs, relaxed console attestation, local-only CLI commands — is gated on `NP_ENV ∈ {local, test}`; any other value gets server behaviour).

Step 1.1 wiring (platform §1): the anonymous allow-list is `PUBLIC_ANONYMOUS_ROUTES` exported by `@nearpay/spec` (derived from route auth `anonymous`/`client_secret`), so `NP_PUBLIC_ANON_ROUTES` is not needed while it matches the spec; `NP_JWT_KEYS_DIR=/run/secrets/jwt` holds `current.pem` and `next.pem` (kid = `local-` + 12 hex of SHA-256 of the raw public key locally); `NP_SERVICE_CLIENTS_FILE` is `{"clients":[{"client_id","secret_sha256"}]}` with scopes taken from the spec `SERVICE_CLIENT_SCOPES`; identity joins the `egress` network for attestation RPC (§8.2) and Turnstile.

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
| 1 | signup + passkeys, step-up, API keys (secret/publishable), destinations for NEAR/Intents/EVM with attestation, single-approver flow (cool-off configurable to 0 locally, 24 h on server), authz incl. mode header and ops principals (accepted, not issued), JWKS, service tokens, settlement context, notification recipients, test-mode destinations; live mode gated by the **manual** KYB decision from compliance (ops CLI); ops CLI (§17) |
| 2 | team members & invitations, multi-account sessions, two-approver rule, restricted keys, recovery flow, console sessions (enrolment + hardware passkeys) and console reads (§6.1), live mode driven by the ops KYB decision on self-declared details (ADR-0020), audit export (audit-events list) |
| 3 | CLI login (device authorization → restricted test key, §6.2, §19); anonymous allow-list += CLI login and hosted invoice routes (§11, from the spec); billing's service client (scopes from spec `SERVICE_CLIENT_SCOPES`) |
| 4 | Solana/Bitcoin attestation (§20) |

## 17. Ops CLI (Stage 1)

`docker compose exec identity-api /nodejs/bin/node dist/ops.js <command>`: `account-show <acct|email>` (also the testbed's "account live" check, since `GET /v1/account` is session-only) · `operator-add <email> ops|ops_admin` (step 2.1: also emails a 24 h enrolment link, notify `ops.enrolment`, for the operator's hardware passkey) · `operator-revoke <email>` · `destination-cooloff-skip <sd_…>` (local) · `seed-demo [--email] [--name] [--destination]` (`NP_ENV=local` only; idempotent on the email: owner, account "Demo Shop", an active test NEAR Intents-account destination for USDC (`merchant-demo.near` by default) and test secret + publishable keys, printed as JSON; `seed.sh data` uses it, platform §1).

## 18. Stage 2 implementation decisions (step 2.2, migration 0004)

- **Two-approver rule, three layers.** The creator's creation is stored as a `destination_approvals` row (`approve`), so the primary key `(destination_id, user_id)` itself refuses a second decision by the creator; the domain (`decide`) answers `403 approver_conflict` first; the `destination_approvals_complete` trigger refuses any live, non-test transition from `pending_verification`/`pending_approval` to `cooling_off`/`active` with fewer distinct approving users than `approvals_required`. `destination_approvals` is append-only (trigger). Existing live destinations got their creator row in the migration. The creator cannot reject either (`approver_conflict`; they disable instead). The approve handler re-reads the approver's membership in the transaction (the token role may be 10 s old). `approvals_required` counts active owner/admin users at creation. (v1.8: the starting value; see "Audit fixes" below.)
- **Recovery lock covers owners and admins** (§5 said "owns"): admins can change destinations too, so a recovered admin must not redirect funds either. An **approval** is a destination change and is refused with `recovery_lock` during the 48 h (a reject is not) — `recovery_lock` is not in `approveSettlementDestination.errors` (change request). Recovery sessions carry no step-up. Owners are not yet notified: there is no recovery event or notify template in spec 0.2.0 (change request); recovery is audited (`user.recovered` per account) and logged.
- **Invitation links never add a passkey to an existing user.** `verifyMagicLink` on an invitation for an address that already has a user answers `401 authentication_required` without consuming the link; the user signs in and calls `acceptInvitation` (otherwise anyone controlling a victim's mailbox could invite the victim to their own account and gain a passkey on the victim's user, bypassing recovery's lock). Accepting re-issues the same session for the joined account with step-up cleared.
- **KYB name lifecycle.** `kyb.approved` is applied only after `getInternalKybCase` answered (compliance down → redelivery; live mode waits, fail closed); a case no longer `approved` makes the approval superseded (skipped; its own event follows); an approved case without a legal name dead-letters. Leaving `approved` (rejected / needs_info) clears `verified_legal_name`, so `kyb_verified` (sentinel view) and checkout never show a name ops no longer stands behind. Meaning unchanged: reviewed by NearPay ops, self-declared, never "verified identity".
- **Audit.** Operator rows use `actor_type = 'operator'` (Stage 1 CLI rows `ops` read as `operator`). The merchant's `listAccountAuditEvents` excludes operator actions (operator ids stay internal); `listOpsAuditEvents` shows everything. `*.ops_action.recorded` rows use the action id as `audit_events.id` and `occurred_at` as `created_at`.
- **Console keys.** Accepted only when: attestation format is not `none`; the credential is device-bound (multi-device/backed-up refused); the AAGUID is on `NP_OPS_AAGUID_ALLOWLIST_FILE` (empty → all refused; the all-zero and CDP virtual-authenticator AAGUIDs are dropped unless `NP_ENV` is `local`/`test`); on the server stage additionally a full attestation (`x5c`, formats packed/fido-u2f/tpm) chaining to `NP_OPS_ATTESTATION_ROOTS_FILE` (vendor roots, required — without anchors the AAGUID is self-asserted). RP id = host of `NP_OPS_ORIGIN`, so dashboard passkeys cannot assert on the console. `operator-add` creates the user when missing (operators need no merchant account) and mails the enrolment link (`NP_OPS_ORIGIN`). Console refusals are `webauthn_failed`, which the catalogue defines as `400` (the spec text says 401).
- **Gateway.** The console host routes only `/ops/*` and `/ops/*` only on the console host (`403` otherwise); bearer keys on the console host are `403`. Ops claims: `ops_role` and `OPS_ROLE_SCOPES[role]`. Restricted keys get `scp` = their scopes plus the read of every write scope (catalogue only) and are refused at the gateway on routes without a matching `restricted_key:` entry (same rule as the kit; property-tested against it). The cached path is unchanged (pure computation on the cached key record).
- **Keys.** `updateApiKey` is scoped to the session's mode (`livemode`), refuses revoked/expired/rolled keys (`409 state_conflict`) and purges the caches.
- **Member events.** Aggregate `membership` with id = the member's `usr_` id (the envelope carries the account); versions come from the sequence `membership_event_version` (memberships are deleted on removal, so a row version would restart on re-join).
- **Invitations.** The mail is queued inside the creating transaction (nothing is stored if notify refuses); a stale pending invitation of the same address is marked expired first; `purge.auth` marks expired invitations.

### Audit fixes (stage 2 audit, migration 0005)

- **Two-approver rule against one person** (finding 1). `manager_grants (account_id, user_id, granted_at, qualifies_at, revoked_at)` is written only by the `memberships_manager_grants` trigger (every path: sign-up, invitations, role changes, removals, ops CLI); `manager_qualifies_at()` computes the 7 d / 30 d tenure (mirrored by `grantQualifiesAt` in `domain/approval`). Backfill: current managers granted at `greatest(membership created_at, last member.role_changed audit)`; managers removed/demoted in the last 30 days (from the audit log, `typeid_uuid()` decodes `usr_` ids) as closed grants. The `destination_approvals_complete` trigger now repeats requirement (stored / current / 30-day high-water mark) and tenure (grant before `created_at`, `qualifies_at` reached, approver holds the role now, no rejection) and also guards `cooling_off → active`; the cool-off part of the tenure is checked by the application only (the 7-day floor exceeds the default 24 h cool-off). The rule is evaluated on the database clock. The finding's suggested tenure ("older than the cool-off") alone would not stop a puppet invited ≥ cool-off before the destination; the 30-day tenure for contested grants is what makes the one-person property hold within the 30-day window (model-based property test `two-approver rule against one human`).
- **Expiry** reuses `disabled_reason: merchant` (the spec enum has no `expired`; change request) with transition cause `expired` and audit `destination.expired`.
- **Recovery lock on team changes** uses `recovery_lock` on `createInvitation` / `updateMember` / `removeMember`, which do not list it (change request, like `approveSettlementDestination`). Accepting an owner/admin invitation issued before the recovery is not blocked (the new grant still waits its tenure).
- **Stage gate** (finding 2): `domain/env.ts isDevEnv` — relaxed only for `NP_ENV ∈ {local, test}`.

## Spec 0.2.1 contract changes (step 2.2/2.3 change requests, `nearpay-spec/CHANGELOG.md`)

- **`evt.identity.user.recovered`** (`UserRecoveredEvent {user_id, account_ids, recovered_at, lock_until}`, platform event, aggregate `user` = `usr_…`, version = the user's recovery count) after a recovery passkey registration; consumer notify (`security.recovery`).
- **`getInternalOpsOperator` `GET /internal/ops_operators/{id}`** (`service:identity.operators.read`, held by compliance) → `InternalOpsOperator {user_id, role, active, created_at, revoked_at}`; never an operator → 404.
- **`getNotificationRecipients` `user`**: only that user if currently a member (any role; `kind` not applied), else empty.
- `Account.website` and `settings.branding.theme` (`auto|light|dark`) / `corner_radius` (0–24 px) stored and settable via `updateAccount` (no step-up).
- `SettlementDestination.created_by` (creator user; null for Stage 1 rows), `Member.security {passkeys, recovered_at}`.
- Pending-destination expiry now uses `disabled_reason: expired` (supersedes "reuses `merchant`" in §18).
- `recovery_lock` is declared on `createInvitation`, `updateMember`, `removeMember`, `approveSettlementDestination` (§18 change requests closed). `webauthn_failed` stays `400` (catalogue; status changes are breaking).

**Implementation (spec 0.2.1, v1.9).**

- Recovery bookkeeping per user (migration 0006): `users.recovered_at` (shown as `Member.security.recovered_at`) and `users.recovery_count` (the `evt.identity.user.recovered` aggregate version). The account-level `accounts.recovered_at` still drives the 48 h lock. The event is written in the recovery transaction, also when the user manages no account (`account_ids: []`; notify skips it).
- `getNotificationRecipients?user=` also answers for a member **removed in the last 7 days**, with the role they had (from the `member.removed` audit row; index `audit_account_action_target`), so notify can name or mail the removed member. This widens the spec description ("currently a member") on the lead's instruction; it is a description change only, no schema change. Users who are disabled or have no verified email are never returned.
- `Member.security.passkeys` counts dashboard passkeys only (console hardware keys are not the member's sign-in).
- `settings.branding.theme` accepts `auto|light|dark` only on input (the contract enum is open for future values) and `corner_radius` must be an integer from 0 to 24 (`400 invalid_setting` with the field as `param`). `website` must be `https://` (`null` clears it). Neither needs a step-up. Accounts created from now on store `theme: auto`, `corner_radius: 8`; older rows keep no keys until they are set, and reads fill the same defaults.
- `SettlementDestination.created_by` is the stored creator (`usr_…`): every row (Stage 1 included) was created by a dashboard session, so it is never null in practice.
- `webauthn_failed` answers the catalogue `400` everywhere, the console included (the console's code comment said `401`).
- `/ops/*` query strings: the kit's request log records the route id, never the URL, and fastify's own request logging is off. Identity logs no URLs, so there was nothing to strip. Deploy proxies must still drop query strings from their access logs.

## 19. Stage 3 implementation decisions (step 3.2, spec 0.3.0 / kit 0.2.1, migration 0007)

- **Migration number.** `0007_cli_logins` (§6.2 said 0006, which spec 0.2.1 already used).
- **Login IP encrypted.** `cli_logins.ip_enc` (envelope, AAD `cli_login:<id>:ip`) instead of `ip inet`: IPs are PII at rest (§12, repo AGENTS). `getCliLogin` decrypts it for the approving user. `country` (from `CF-IPCountry`, `^[A-Z0-9]{2}$`) and the user agent (≤ 256 chars, control characters stripped) stay plain.
- **Audit of the minted key.** `api_key.created` with `details {type, livemode: false, scopes, via: "cli_login", login_country, client_version}`; the login IP is **not** in `details` (the contract says details carry no personal data) but in the new nullable column `audit_events.origin_ip_enc` (AAD `audit:<id>:origin_ip`), so it stays provable after the login row is purged; `ip_enc` remains the approver's IP. A denial is audited as `cli_login.denied` (target `cli_login` / user code). No security email: the LLD names none and `api_key.created` has no notify consumer; adding one would be a notify change (open question for the lead).
- **Uncollected keys are revoked.** An `approved` login whose first poll did not come within 15 min of the approval (`REDEEM_WINDOW`) reads as `expired` at once (no key is handed out) and the job revokes the key, erases `key_secret_enc` and audits `api_key.revoked` (`via: cli_login, reason: not_collected`). A secret nobody received must neither stay usable nor stay stored.
- **Status is derived, the job makes it durable.** Every read and command uses the effective status: `pending` past `expires_at` and uncollected approvals are `expired` before the job runs; all time checks use the database clock (`now()`), so replicas and the job agree.
- **slow_down.** A poll is on time when it comes ≥ `interval − 1 s` after the previous **accepted** poll (1 s timer/network jitter); refused polls do not move `last_polled_at`, so a client that waits `Retry-After` always gets through. The check runs on the row lock, so concurrent polls of one device code cannot both pass. Unknown device codes are `404 resource_missing` with the id shown as `npdc_…` (never echoed).
- **Create limit.** 10 per 10 min per client IP as a token bucket (burst 10, one back per 60 s) in Valkey; the per-pod fallback (Valkey down, half limits) now keeps slow buckets until they would be full again instead of 60 s (it used to forget them).
- **User codes.** Uniform by rejection sampling (bytes ≥ 240 dropped; 20⁸ ≈ 2.6·10¹⁰ codes); a clash with a pending login (unique partial index) draws again (≤ 5 tries). `verification_url_complete` = `verification_url?code=<user_code>` (the Stage 3 dashboard page reads `code`).
- **Visibility.** A pending login is visible to any owner/admin/developer session (the code is the capability; enumeration is bounded by the per-user gateway limit: 25 rps × 15 min ≈ 2·10⁴ ≪ 2.6·10¹⁰); approved, redeemed and denied logins only to the deciding account (`account_id` is stored on denial too; the object shows `account`/`key` only when a key was minted); expired logins for 24 h after they stopped being actionable. Approve and deny re-read the user's membership role in the transaction (the token's role may be 10 s old).
- **Idempotency.** As the spec says: `createCliLogin` / `pollCliLogin` are not idempotent routes (the device code is the poll credential; a replayed poll must never replay a key), approve/deny are (replays return the stored `CliLoginRequest`, which carries no secret).
- **Anonymous allow-list.** Still derived from the spec (`PUBLIC_ROUTES` with `anonymous`/`client_secret`), so the CLI login and hosted invoice routes are open at the gateway with no configuration change; `NP_PUBLIC_ANON_ROUTES` is not read (§11 step 1.1 wiring). Integration tests pin both groups.
- **Billing's service client** gets `SERVICE_CLIENT_SCOPES.billing` from the spec (0.3.0: `payments.sessions.create`, `payments.read`, `notify.send`, `identity.context.read`, `rails.read`); its secret comes from `NP_SERVICE_CLIENTS_FILE` (deploy already seeds `billing`). Tested: token scopes, settlement context `200`, recipients `403`.

## 20. Stage 4 — Solana and Bitcoin ownership attestations (step 4.1, spec 0.4.0)

Contracts: `attestSettlementDestination.body.standard` += `solana`, `bip322`; `src/common/chains.ts` (`BitcoinDestinationAddress`, `DESTINATION_SIGNING_STANDARDS`); vectors `test-vectors/destination-attestation.json` (message, Solana cases, BIP-322 upstream vectors + NearPay P2WPKH/P2TR cases, each labelled valid/invalid) and `chains-solana-bitcoin.json` (`addresses`). The Stage 1 refusal of live Solana/Bitcoin destinations (`400 invalid_request`, `param: chain`) is lifted; `STAGE1_STANDARDS` becomes all four.

**Destinations.** Chains `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` and `bip122:000000000019d6689c085ae165831e93`, type `chain_address` only (`intents_account` stays NEAR), assets from rails' registry for that chain (`asset_unsupported` otherwise; rails §21.1). Validation and `address_norm` from kit `chains`:
- Solana: base58 decoding to 32 bytes; `address_norm` = as given. An off-curve (program-derived) address — e.g. a Squads vault — is a valid destination but cannot sign, so it can only go the `mark_unverified` path (USD 1,000 cap).
- Bitcoin: P2WPKH (`bc1q…`, v0 20-byte, bech32) or P2TR (`bc1p…`, v1 32-byte, bech32m), mainnet HRP, single case; `address_norm` = lower-case. P2PKH, P2SH (incl. nested segwit) and P2WSH are refused with `address_invalid` ("Use a native SegWit (bc1q…) or Taproot (bc1p…) address"), in both modes, because BIP-322 "simple" cannot prove ownership of them (P2SH/P2WSH scripts are unknown from the address; legacy P2PKH needs the legacy message format, not supported). Existing test-mode rows are untouched.

**Challenge.** Same message as §8.2 (the address exactly as entered on `Address:`); `signing_standard` `solana` / `bip322`; `nonce` and `recipient` null.

**`standard: solana`.** `signature` = base58 of the 64-byte ed25519 signature over the exact UTF-8 bytes of the message (wallet-standard `solana:signMessage`; Phantom, Solflare, Backpack sign raw bytes). Valid iff it base58-decodes to 64 bytes and `ed25519.verify(sig, message, base58decode(address))`. Not accepted in v1: base64 encoding, the Solana off-chain message format (`\xffsolana offchain` header — what the Ledger Solana app signs; Ledger users use `mark_unverified`), Sign-In-With-Solana messages. `public_key` is ignored.

**`standard: bip322`.** `signature` = base64 of the BIP-322 "simple" encoding (the witness stack of the virtual `to_sign` transaction spending `to_spend`; message hash = tagged hash `BIP0322-signed-message` of the message bytes):
- P2WPKH: witness exactly `[DER ECDSA signature ‖ 0x01, 33-byte compressed pubkey]`; `hash160(pubkey)` = witness program; ECDSA over the BIP-143 sighash of input 0 (scriptCode = P2PKH of the program, amount 0, `SIGHASH_ALL` only); high-S accepted (verification, not relay).
- P2TR: witness exactly `[64-byte Schnorr signature]` (`SIGHASH_DEFAULT`) or 65 bytes ending `0x01` (`SIGHASH_ALL`); BIP-340 verify over the BIP-341 key-path sighash (amount 0, the destination's scriptPubKey) against the output key = witness program. Script-path spends and annexes are refused.
- Refused (`attestation_invalid`): the "full" format (a serialized transaction), the legacy `signmessage` format, any other witness shape or sighash type.
- Implementation: pure verifier in `src/infra/crypto/bip322.ts` on `@noble/curves` + `@scure/btc-signer` (sighash helpers `preimageWitnessV0/V1`), checked against every case of the vector file. Wallets: Unisat, Xverse, Leather, OKX (`signMessage(…, 'bip322-simple')`), Sparrow (segwit); the dashboard also accepts a pasted base64 signature.

**Errors.** Bad signature, wrong key, wrong encoding/format → `400 attestation_invalid`; challenge rules unchanged (`challenge_expired`, single use). No RPC is needed for either chain (unlike NEP-413 named accounts), so neither standard can be "unavailable".

**Implementation decisions (step 4.2, v1.13).**
- Verifiers: `src/infra/crypto/solana.ts` and `src/infra/crypto/bip322.ts`, pure, on `@noble/curves` 2.4.0 (already used for NEP-413/EIP-191), `@scure/btc-signer` 2.4.1 (sighashes, address → scriptPubKey, witness serialization) and `@scure/base` 2.4.0 (strict base64). Every case of `destination-attestation.json` and the address rules of `chains-solana-bitcoin.json` are unit tests.
- Solana: the signature must match `^[1-9A-HJ-NP-Za-km-z]+$` and decode to 64 bytes; verification is RFC 8032 strict (`zip215: false`: canonical `R`/`S`). The address must decode to an on-curve point that is **not small-order** — a small-order key (e.g. the all-zero System Program address, which `chains-solana-bitcoin.json` lists as on-curve) verifies forged signatures, so such an address has no provable owner and is refused like an off-curve one. Off-curve → `attestation_invalid` with the message pointing to the unverified path. A signature over the short off-chain envelope (`\xff"solana offchain"`, version 0, format 0, u16 LE length, message) is recognised only to give the merchant a better message; it is never accepted.
- BIP-322: base64 must be canonical (padded) and the witness must re-serialize to the exact bytes (no trailing data, minimal CompactSize); P2WPKH needs a 33-byte compressed key and **strict DER** (BIP-66: the parsed signature re-encodes to the same bytes), high-S accepted; P2TR accepts 64 bytes, or 65 ending `0x01` (an explicit `0x00` hash type is not a BIP-341 encoding and is refused). `to_spend` / `to_sign` txids are checked against the BIP's vectors.
- Bitcoin destination type: kit 0.3.1 has no type helper, so `bitcoinAddressType` (domain) reads it from the kit-validated address (`bc1q` + 42 chars = P2WPKH, `bc1q` + 62 = P2WSH, `bc1p` + 62 = P2TR, `1…` P2PKH, `3…` P2SH, else `other`); swap for the kit helper when kit ships it. Refusal: `400 address_invalid`, `param: address`, message "Use a native SegWit (bc1q…) or Taproot (bc1p…) address: … is a <type> address …", in both modes.
- Migration `0008_bitcoin_destination_types`: check constraint `settlement_destinations_bitcoin_live_type` — a **live** Bitcoin row's `address_norm` must be lower-case P2WPKH/P2TR; validated at once (no live Bitcoin rows could exist before Stage 4); test-mode rows untouched.
- The challenge message keeps the address as entered (an upper-case bech32 address stays upper-case on `Address:`); ownership is checked against `address_norm` (lower-case). `attestation.method` gains `solana_ed25519`, `bip322_p2wpkh`, `bip322_p2tr`. The Stage 1 refusal is gone: `VERIFIABLE_STANDARDS` = all four.
- Not in identity v1: the `mark_unverified` command (§5) has no route in spec 0.4.0, so off-curve Solana and Ledger users have no self-serve path yet (their error messages ask for a wallet that signs raw bytes).

