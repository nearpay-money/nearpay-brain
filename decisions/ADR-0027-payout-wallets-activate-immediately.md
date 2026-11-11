# ADR-0027: Payout wallets activate immediately — no cool-off by default

- **Status:** Accepted
- **Date:** 2026-11-11
- **Deciders:** Owner (decision, 2026-11-11, final); Principal Engineer (lead agent, implementation)
- **Invariants touched:** none changed (INV-1 non-custodial, INV-6 destination snapshot per intent still hold)
- **Amends:** ADR-0005 decision 2 ("24 h cool-off with notification to all owners"); FR-SET-03 as amended below

## Context
ADR-0005 put a 24 h cool-off between a settlement destination's ownership attestation (and, where required, its second approval) and its activation, against account takeover. On 2026-11-09 the owner made the cool-off configurable down to 0 on every stage and the beta set 0, but identity still moved the destination to `cooling_off` with `active_at = now + NP_COOLOFF_SECONDS` and a `destination.activate` job re-screened and activated it later: even with 0 the merchant saw a cool-off state, the dashboard spoke of "a short security hold", and activation depended on the job worker.

Owner decision 2026-11-11 (final): **payout wallets activate immediately after attestation/approval; no cool-off by default.** A business accepting blockchain payments must be able to take live payments the moment its payout wallet is added and its ownership is proven.

## Decision
1. `NP_COOLOFF_SECONDS` defaults to **0** in identity and in the Compose stack (beta: 0, unchanged). With 0, a successful attestation — or the last required approval under the adaptive two-approver rule — screens the address synchronously (the same compliance screening the activation job ran, called **before** the transaction; unavailable → `503 screening_unavailable`, nothing changes, the unused challenge can be retried) and moves the destination straight to **`active` in the same request and transaction**: transition row (`pending_verification|pending_approval → active`, cause `attested`/`approved`), outbox `evt.identity.settlement_destination.activated` (+ `default_changed` when it becomes the first default), `live_enabled` recomputed. A screening `block` disables it (`disabled_reason: compliance`) as activation did. No `cooling_off` event and no activation job are produced.
2. Everything else that protected destination changes stays: stepped-up owner/admin session, API keys can never touch destinations, screening at creation and at activation, ownership attestation, the adaptive two-approver rule (application, approvals primary key and the `destination_approvals_complete` trigger, which already guards `pending_* → active`), the 48 h recovery lock, the audit log, and notify's single "payout wallet active" security email (`security.destination_activated`) to the account's security contacts. The "cool-off started" email (`security.destination_pending`, on `cooling_off`) is no longer sent in the default flow.
3. An operator may still set a cool-off > 0 (up to 30 days); then the earlier path applies unchanged (`cooling_off`, `active_at`, job `destination.activate` with re-screen and approval re-check). The boot line about the cool-off is informational (`info`), never a warning; the gauge `nearpay_identity_destination_cooloff_seconds` stays.
4. The dashboard says "Payout wallet active" right after verification and carries no cool-off / security-hold wording in the default flow; a `cooling_off` destination (non-zero configuration) gets neutral "Activating" copy.

## Consequences
- ➕ Live payments are possible right after onboarding; no dependency on the job worker for activation; one notice for the merchant.
- ➕ Consumers keep working unchanged: payments refreshes its settlement context on any destination event (`activated` included), compliance and rails start monitoring / watching on `activated`, sentinel's reconciliation compares `active_at` (now the activation instant) with the execution's creation time, webhooks map `activated`.
- ➖ An account takeover that passes step-up, attestation and (where required) a second approver redirects future payments without a waiting window; the security email arrives at activation, not before. Accepted by the owner: the attacker must control the session, a fresh passkey step-up and the destination key, and every intent snapshots its destination (INV-6).
- ➖ Two activation paths remain in identity (instant and job); both are covered by tests.

## Alternatives considered
- **Keep `cooling_off` with `active_at = now` and the job (the 2026-11-09 state):** lost — the merchant still sees a cool-off state and activation waits for the worker.
- **Remove the cool-off option entirely:** lost — operators of a later deployment may want a delay; keeping the configuration costs little.

## Revisit when
Destination-change attacks are observed (runbook RB-06), or a regulated launch requires a waiting period.
