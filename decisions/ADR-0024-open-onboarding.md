# ADR-0024: Open onboarding — no NearPay approval before live mode

- **Status:** Accepted
- **Date:** 2026-11-10
- **Deciders:** Owner (decision, 2026-11-10 17:18 IST); Principal Engineer (lead agent, implementation)
- **Invariants touched:** none changed (INV-1 non-custodial, INV-2 evidence before success, INV-8 test mode isolation still hold)
- **Amends:** ADR-0020 decision 3 ("self-declared minimum business details + an ops decision in the console"): the ops decision is no longer a required step; FR-CMP-01 / FR-ACC-05 as amended below

## Context
Since ADR-0020 a merchant self-declares minimum business details (no vendor, no documents) and a NearPay operator approves them in the console (`decideOpsKybCase`, or the Stage 1 `kyb-decide` CLI) before identity turns live mode on. In the closed beta the owner is the only operator. Every new merchant therefore waits for a human who checks nothing a machine could not: the details are self-declared, there is no register lookup and no document. The owner's words: "We operate on an open blockchain network; a NearPay approval only adds friction. A new merchant fills in their business details, no approval from NearPay is needed, onboarding is smooth, anyone can join and start working (live mode) without delay. Business registration/reference number becomes optional."

NearPay is non-custodial (INV-1): it never holds merchant or payer funds, so an approval step protects no balance. The protections that matter are automatic and instant: sanctions screening of payout addresses (OFAC digital-currency list + blocklist, compliance §4) and the geo policy (blocked countries/regions, FR-CMP-03).

## Decision
1. **Owner decision (2026-11-10):** submitting complete business details activates the account for live mode immediately. No human step is in the default path.
2. Compliance `NP_KYB_MODE` = `auto` (default) | `manual`.
   - `auto`: `submitKyb` moves the case `→ pending` (event `kyb.submitted`) and, **in the same transaction**, records the system's decision — actor `system`, transition cause `auto_decision`, `decided_by = system`:
     - `approved` with reason `auto-approved: open onboarding (ADR-0024)` → `kyb.approved`; identity copies the declared legal name/country and recomputes live mode exactly as before (live = KYB approved + an active live destination + account active).
     - `rejected` with reason `auto-rejected: geo_blocked (…)` (the existing `geo_blocked` reason code) when the country of incorporation, the business address country, or an ISO 3166-2 address region is in the geo policy (`policies.blocked_countries` / `blocked_regions`) → `kyb.rejected`; live mode stays off.
     - A `kyb.auto_decide` job (every minute) decides complete cases still `pending` (submitted while the mode was `manual`, or before this release).
   - `manual`: today's flow — `pending` waits for an ops decision (console or `kyb-decide`). Switching needs only the env var and a restart.
3. **Required fields:** legal name, country of incorporation, business type, business address, representative, expected monthly volume (as before). **`registration_number` and `website` are optional** everywhere (compliance completeness check, dashboard form, console "Not provided"). No spec schema change: the field was already nullable/optional; only descriptions and the `kyb_profile_incomplete` example changed (docs-only, unreleased).
4. **What stays (automatic, instant, not an approval):**
   - Payout-address sanctions screening when a destination is added and the daily re-screen (a hit blocks with the existing reasons `sanctions` / `blocklist` and disables the destination).
   - Geo policy on the business country/region at submission (above) and on payers as before.
   - Operators can **suspend / reinstate** a merchant (existing suspension tools, adaptive two-person rule) and **reject a business for abuse** from the console (`decideOpsKybCase` `rejected` from `approved`, audited as an operator action). An operator may also reverse a rejection (`approved` from `rejected`).
5. **Console:** the "Business reviews" queue becomes the **Businesses** list (opens on active businesses; screening result from the system decision; reject for abuse; link to suspend/reinstate). Approve / request-info remain for manual mode.
6. **Dashboard and mail:** no "under review" / waiting copy; after saving, "You're live — you can accept real payments". `kyb.submitted` sends no mail; the `kyb.decision` approval mail is the "You're live" welcome. Security mails are unchanged.
7. **Copy rule (amends ADR-0020 d3 wording):** `kyb_verified` / `verified_legal_name` keep their names (additive contract) but now mean "the legal name the owner self-declared and that passed the automatic checks". No surface says "reviewed by NearPay" or "verified"; checkout says "Business details declared by the merchant".
8. The data model and audit trail are unchanged: the case, its encrypted profile, `declared_by`, `submitted_at`, every transition (actor, cause, reason) and the events stay.

## Consequences
- ➕ Onboarding is self-serve end to end: sign-up → destination → business details → live in minutes, with no operator on the path; a one-person NearPay can run the beta.
- ➕ Every decision is still recorded and auditable; the manual review is one env var away.
- ➖ **Accepted risk:** NearPay no longer looks at a business before its first live payment. A fraudulent or prohibited business can take live payments until an operator notices (merchant reports, cases, screening hits) and suspends or rejects it. Bounded by: non-custody (NearPay holds no funds), payout-address sanctions screening, geo policy, payer-side screening, the kill switch, suspension.
- ➖ **Accepted risk:** declared details (legal name shown at checkout) are unverified; payers see "declared by the merchant", never "verified".
- ➖ Fewer data points per merchant (no registration number by default) for later investigations.
- ➖ Name screening of the declared legal name/representative is not done (the local provider screens addresses only); revisit with a screening vendor (Q-VEN-01).

## Alternatives considered
- **Keep the manual review** — lost: owner decision; adds friction without a check a human can actually perform on self-declared data.
- **Auto-approve after a delay (cool-off) so ops can intervene** — lost: same waiting UX the owner rejected for destinations ("no waiting period", 2026-11-09).
- **Remove the KYB case entirely** — lost: identity's live gate, the legal name shown at checkout, sentinel's `kyb_verified` and the audit trail all hang off it; keeping the case with an automatic decision is additive and reversible.

## Revisit when
- Abuse (fraud reports, chargeback-style disputes, screening hits on merchants) appears in the beta.
- A jurisdiction or partner requires KYB before processing, or a screening vendor is chosen (Q-VEN-01) — then `manual`, or an automatic vendor decision, per market.
