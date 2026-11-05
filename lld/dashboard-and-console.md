# LLD — `nearpay-dashboard` and `nearpay-console`

> Version 1.1 (audited, `audits/dashboard-and-console.md`) · Foundations `00-foundations.md` · Identity LLD §6

## 1. Dashboard (`dashboard.nearpay.<tld>`, static SPA)

- **Auth:** passkey login/registration (`@simplewebauthn/browser`) against identity; session cookie is same-origin (`/api` → gateway); CSRF token header on unsafe methods; step-up modal (WebAuthn) when an endpoint returns `401 step_up_required`.
- **Mode switch:** test/live toggle (stored in memory + URL `?mode=`); every request sends `NearPay-Mode: test|live` and identity's authz mints `lm` accordingly for session principals (identity §8.1); live writes are disabled in the UI until `GET /v1/account/requirements` reports `live_enabled`.
- **Pages (stage):** Home (requirements checklist, recent payments) S1 · Payments list/detail (attempts timeline, evidence links to explorers) S1 · Settlement destinations (add → wallet attestation via NEAR Connect / EIP-191, cool-off status, fingerprint) S1 · Developers: API keys (create/roll/revoke), webhooks (endpoints, deliveries), events S1 · Agent payments S1 · Refunds (create; wallet signing of `next_action`; status) S2 · Team & approvals S2 · KYB S2 · Customers, payment links S2 · Invoices, subscriptions, products S3 · Mandates S4 · Settings (fee bearer, late payments, tolerance, branding) S1/S2.
- **Wallet signing (refunds, attestations):** builds exactly the payload returned by the API (`next_action` / challenge message) and shows a human-readable summary before signing; never edits recipient or amount.
- **Tests:** Playwright flows in testbed for signup → destination → key → webhook → test payment visible.

## 2. Console (`ops.nearpay.<tld>`, internal SPA behind Cloudflare Access in server stage; local: separate port bound to localhost)

- **Auth:** `ptype=ops` sessions issued by identity for users in the `ops` allowlist with hardware-key passkeys; every action audited.
- **Pages:** Merchants (search, status, suspend/reinstate with two-person approval) · KYB decisions (Stage 1 manual approve/reject) · Compliance cases · Verifier holds & executions explorer (rails) · Sentinel findings & kill switch (engage; release requires second approver) · Reconciliation runs · Audit log.
- **Stage mapping:** Stage 1 ships ops **CLI scripts** in `nearpay-testbed/ops` (KYB decision, kill-switch release) instead of UI; console UI arrives in Stage 2.
