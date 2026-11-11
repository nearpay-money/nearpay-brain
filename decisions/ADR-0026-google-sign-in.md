# ADR-0026: Continue with Google for the merchant dashboard (optional, OIDC + PKCE, keyed on the Google subject)

- **Status:** Accepted
- **Date:** 2026-11-11
- **Deciders:** Owner (decisions, 2026-11-11); Principal Engineer (lead agent, design details and implementation)
- **Invariants touched:** none changed. INV-1 (non-custody) and INV-7 (origin refunds) still depend on step-up, which stays a passkey assertion; INV-9 (append-only audit) gains `user.google_linked` / `user.google_unlinked` and the sign-in method on `login`.
- **Amends:** FR-ACC-02 ("MUST authenticate dashboard users with passkeys"): passkeys remain the only way to **confirm** sensitive actions; Google becomes an optional way to **sign up and sign in** (see Decision 1). Console (ops) sign-in is unchanged (identity LLD §21).

## Context
Merchants asked for a one-click sign-up. Today the dashboard signs up with an emailed magic link plus a passkey, and signs in with a passkey (identity LLD §6). The owner decided (2026-11-11) to add "Continue with Google" next to the passkey flow, without weakening what protects money: settlement destinations, refund recipients, API keys and team changes all require a fresh passkey step-up (§12) and NearPay never holds funds (INV-1), so the risk is account takeover, not theft from NearPay.

External facts (`reference/landscape.md` §10): Google's OpenID Connect endpoints and issuers (`https://accounts.google.com` and the legacy `accounts.google.com`); Google recommends keying users on `sub`, not `email`; `email_verified` is a boolean claim; supported `prompt` values are `none`, `consent`, `select_account` (no `login`); `auth_time` is returned only when requested through `claims` and only for verified apps.

## Decision
1. **Optional sign-in method.** OpenID Connect authorization code flow with PKCE (S256), `state` and `nonce`, run server-side by identity (§22). Off unless identity has `NP_GOOGLE_CLIENT_ID` **and** a non-empty client secret file (`NP_GOOGLE_CLIENT_SECRET_FILE`); identity starts either way. The dashboard learns whether to show the button from `GET /v1/auth/methods` (no build flag).
2. **ID token verification:** signature against Google's JWKS (`jose` remote key set, cached 10 min, refetched on an unknown `kid`), RS256 only; `iss` ∈ {`https://accounts.google.com`, `accounts.google.com`}; `aud` = our client id (and `azp`, when present); `exp`/`iat` with 30 s tolerance; `nonce` = the flow's nonce; `email_verified === true` (the boolean). No Google SDK.
3. **Identity = Google `sub`**, stored in `linked_identities (provider, subject, user_id, email_at_link, created_at)`, at most one Google identity per user. Never keyed on email alone.
4. **New email (no user)** → user + account (owner) exactly as the sign-up path (shared code), the email counted as verified (Google verified it), no magic link; the browser lands on onboarding (`/setup`). Open onboarding (ADR-0024) unchanged.
5. **Existing user with that email and no Google link → refused, never auto-linked** (account-takeover risk: whoever controls a Google account for that address would otherwise get the NearPay user). The dashboard says: "This email already has a NearPay account. Sign in with your passkey, then connect Google in Settings." Settings offers **Connect Google** (stepped-up session; the callback links the `sub` unless it belongs to another user) and **Disconnect** (stepped-up; only while the user still has a passkey; ends the user's other Google-created sessions).
6. **Sessions** after a Google login are ordinary dashboard sessions (`__Host-np_session`, same TTLs) with `auth_method = google` and **no step-up**. Step-up for sensitive actions stays a passkey assertion; `stepUpOptions` answers `403 passkey_required` for a user without a passkey.
7. **First passkey of a Google-only user (owner asked us to pick the safest consistent option):** registration from a session is allowed only within **10 minutes of the Google sign-in** that created the session (`DashboardSession.sign_in.passkey_setup_until`); afterwards the user continues with Google again (`401 reauthentication_required`). We chose this over a separate "Google re-auth" because Google offers no verifiable forced re-authentication for our app (no `prompt=login`; `auth_time` only for verified apps), so a "re-auth" would prove no more than a fresh sign-in. **Consistency rule for everyone:** adding a passkey from a session now needs a fresh confirmation — a step-up (≤ 10 min; a passkey sign-in counts) when the user already has a passkey. This closes the older gap where any session could add a passkey and thereby gain step-up. Recovery (magic link → passkey) is unchanged.
8. **Callback** `https://dashboard.<domain>/api/auth/google/callback` (the redirect URI registered at Google); the gateway rewrites it to identity's spec route `GET /v1/auth/google/callback` (identity router chain: ForwardAuth, rate limit). The callback always answers `303` to a dashboard page (success or `?google_error=<code>`).
9. **Flow binding:** a `oauth_flows` row per start (10 min, single use; `state`, cookie binding and `nonce` stored as SHA-256, the PKCE verifier envelope-encrypted) and a cookie `__Host-np_google` (HttpOnly, Secure, SameSite=Lax, Path=/, Max-Age 600). The callback consumes the flow on **any** attempt before exchanging the code; it needs the matching state, the matching cookie and an unexpired, unused flow. A link flow also records the stepped-up session that started it and is refused if that session has ended (the Strict session cookie is not sent on Google's cross-site redirect, so the flow, not the cookie, identifies the user).
10. **Audit and events:** `login` carries `details.method` (`passkey` | `google`); `account.created` carries `method: google` for Google sign-ups; `user.google_linked` (`via: signup | settings`) / `user.google_unlinked` in every account of the user; events `evt.identity.user.google_linked` / `.google_unlinked` (ids only) for notify (security notice: follow-up in notify).
11. **Rate limits:** gateway per-IP limit on the start and callback routes (like the CLI login routes), identity's anonymous limit, plus identity's own 30 starts / 10 min per client IP.
12. **Branding:** both auth pages show "Continue with Google" above the passkey option as Google's standard button (white, Google "G" mark, text "Continue with Google"), rendered from bundled SVG — no Google script (CSP unchanged, ADR-0010 not touched).

## Consequences
- ➕ One-click sign-up and sign-in; no email round trip for Google users.
- ➕ Money controls are unchanged: every sensitive action still needs a passkey; a Google-only user must add a passkey before touching destinations, keys, refunds or the team.
- ➕ Passkey users gain protection: a stolen session cookie can no longer add a passkey without a fresh step-up.
- ➖ **Accepted risk:** a compromised Google account of a Google-linked user signs in to NearPay (read access, non-sensitive settings). Bounded by: step-up needs a passkey; a Google-only user's first-passkey window is 10 minutes after a Google sign-in, so a Google-account takeover of a Google-only user can register a passkey — the same strength as that user's only factor; recovery still emails the address.
- ➖ Google-only users cannot sign in while Google sign-in is switched off; email recovery (adds a passkey) exists in the API (`POST /v1/auth/recovery`) but the dashboard has no recovery page yet (follow-up), so the dashboard nudges Google-only users to add a passkey at their first sensitive action.
- ➖ Disconnect requires a passkey, so a Google-only user must add one first.
- ➖ Adding a passkey now prompts a step-up for passkey users whose sign-in is older than 10 minutes (one extra tap).
- ➖ The kit's route bindings emit only 2xx; identity writes the callback's 303 itself (documented in identity §22; a kit change can remove that later).

## Alternatives considered
- **Auto-link by verified email** — lost: anyone controlling a Google account for the address (re-registered domain, workspace admin, recycled address) would take over the NearPay user.
- **Google sign-in counts as a step-up** — lost: owner decision; would put money controls behind Google account security.
- **Client-side Google Identity Services (One Tap / GIS script)** — lost: third-party runtime script in the dashboard (strict CSP, ADR-0010) and the ID token would pass through page JavaScript; the server-side code flow keeps the client secret, code and token off the browser.
- **SPA callback page that posts the code to the API** — lost: puts the code in page JavaScript and browser history; the server callback keeps it to one redirect.
- **"Re-authenticate with Google" before adding the first passkey** — lost (for now): not verifiable without `prompt=login` / `auth_time` (see Context); revisit if the app is verified and `auth_time` is available.
- **Separate Google-only session kind** — lost: same cookie, TTLs and authz path keep the gateway hot path unchanged.

## Revisit when
- The Google OAuth app is verified and `auth_time` can be required: tighten the first-passkey rule to "Google authentication within 5 minutes".
- More providers are wanted (Apple, Microsoft): the `linked_identities.provider` enum and the routes are provider-generic.
- Account-takeover attempts through Google appear in the audit log (`google_error=email_exists` spikes, unexpected links).
