# Audit — Whole-system LLD

> Audited 2026-11-05 after all component audits. Result: **pass — LLD set is dev-ready.** Two passes; pass 2 found no new defects. Open external facts remain as spikes (SP-01, SP-02, SP-03, SP-04, SP-07, SP-14, SP-15), each with a defensive default already designed in. Updated 2026-11-06 for the step 1.2/1.3 implementation: SP-01, SP-14 and SP-15 resolved; traces T3, T5 and T7 adjusted below.

## 1. Method

1. **Automated cross-references** (`syscheck`): every `METHOD /internal/…` referenced anywhere is defined in exactly one owner's API table; every consumed event subject has a producer; every produced subject has a consumer or a public webhook mapping.
2. **Scope matrix:** every internal call's caller appears in the service scope catalogue (foundations §9) with the scope the callee checks.
3. **Route matrix:** every public path in any LLD maps to one Traefik router (platform §1) and is either authenticated or in identity's anonymous allow-list.
4. **End-to-end traces** of every Stage 1 flow, hop by hop, against the owning LLD sections (§3).
5. **Latency budget chains** for synchronous paths (§4).
6. **Invariant walk** INV-1…9 against the concrete mechanisms (§5).

## 2. Findings

| ID | Check | Severity | Finding | Fix | Where |
|---|---|---|---|---|---|
| S-01 | events | Medium | `evt.compliance.policy.updated` had no listed consumer although payments caches the geo policy | Added to payments consumers | payments §7 |
| S-02 | events | Low | `evt.payments.checkout_session.created`, `payment_attempt.updated`, `balance_transaction.created`, `payment_link.*`, `customer.*` had no consumer and no public mapping | Removed (objects readable via API) | payments §7 |
| S-03 | routes | High | Anonymous routes were implied in several LLDs but identity had no explicit allow-list | Exhaustive v1 allow-list in identity config | identity §11 |
| S-04 | scopes | High | Service scopes were checked by callees but never catalogued; callers could not be provisioned | Service scope catalogue | foundations §9 |
| S-05 | trace | High | Agents paying checkout sessions trusted `pay_to` from the API without the two-party release humans get | Requirements carry an attestation ticket; `@nearpay-money/agent-signer` verifies sentinel before signing | agentpay §5, sdks §2 |
| S-06 | trace | Medium | Stage 1 ops actions (KYB decision, kill-switch release) had no authenticated path before the console exists | In-container ops CLI via `docker compose exec` (local only), console in Stage 2 | platform §3 |
| S-07 | routes | Low | Email-provider bounce webhook (Stage 5) had no route | Router to notify-api + anonymous allow-list entry | platform §1, identity §11 |
| S-08 | budgets | Medium | Sentinel p99 400 ms was impossible with a live provider status call | p99 1.5 s (cached ≤ 50 ms); HLD, SLOs and risk R-17 aligned | sentinel §11, HLD §6, `operations/slos.md` |
| S-09 | trace | Medium | Amount tags could leak when a quote failed after allocation | Tag released on any failure after allocation | rails §8.1 |
| S-10 | scopes | Low | payments listed webhooks as a caller of `GET /internal/payment_intents/{id}` (webhooks never calls it) | Caller list corrected | payments §6.3 |
| S-11 | platform | High | Polyrepo with no remote registry in Stages 1–4: services could not consume `@nearpay-money/*` packages as published artifacts (ADR-0001) | Local Verdaccio registry in the Compose local profile; GitHub Packages/npm once remotes exist | platform §1 |
| S-12 | scopes | Medium | Sandbox endpoints had no authentication statement | `sandbox.use` service scope required | sandbox §2 |

Pass 2 re-ran checks 1–6 after the fixes: **no findings**.

## 3. End-to-end traces (Stage 1)

**T1 — Merchant onboarding:** dashboard passkey signup (identity §6) → magic link (notify `POST /internal/messages`, Mailpit) → destination create (identity §8.2; compliance `POST /internal/screenings`) → wallet attestation (dashboard §1, NEP-413/EIP-191) → cool-off job (identity §9; 0 s locally) → `evt.identity.settlement_destination.activated` → payments context cache (payments §7), rails watch-list (rails §7), compliance monitoring (compliance §6), sentinel cache (sentinel §7), notify security email (notify §4) → ops CLI KYB decision (platform §3 → compliance §5) → `evt.compliance.kyb.approved` → identity `live_enabled` (identity §5). ✔

**T2 — Cross-chain checkout (EVM payer → USDC to merchant):** `POST /v1/checkout/sessions` (payments §8.1; identity settlement context; rails `GET /internal/destination-minimums`) → SPA loads (checkout §5) → options (payments → rails `GET /internal/options`) → attempt (payments §8.2: geo, Turnstile, window rule) → rails `POST /internal/executions` (rails §8.1: kill-switch flag, compliance screen, pricing, tag, 1Click quote, integrity + signature) → ticket → sentinel co-sign (sentinel §5 R1–R7) → SPA verifies (checkout §5.2) → wallet ERC-20 transfer → `deposit_tx` → rails `POST /internal/executions/{id}/deposit_tx` (1Click submit) → hot poller (rails §8.4) ∥ EVM watcher tag match (rails §8.7) → evidence → `evt.rails.execution.settled` → payments apply (§8.3) → `payment_intent.succeeded` + `checkout.session.completed` → webhooks ingest/dispatch (webhooks §6) via egress proxy → `merchant-demo` → notify receipt (payments receipt-context) → SSE → redirect. ✔

**T3 — Checkout paid from a NEAR Intents balance (human):** attempt `method=intents_transfer` with `payer_wallet` (payments §6.2) → rails direct-intents requirement + binding (rails §8.1) → sentinel intents checks (sentinel §5) → wallet signs NEP-413/ERC-191 (checkout §5.4) → `signed_intent` → agentpay `POST /internal/relay` (agentpay §5) → batcher `execute_intents` (agentpay §7) → NEAR indexer binding match on the DIP-4 `transfer` memo (rails §8.6, SP-14) → settled → payments. ✔

**T4 — Agent pays an x402 API (merchant-demo `/premium`):** middleware → `POST /x402/requirements` (agentpay §5; identity context; rails assets) → `402 PAYMENT-REQUIRED` → `withX402` + agent-signer (sdks §2) → `PAYMENT-SIGNATURE` → `/x402/verify` (agentpay §6: signature, simulate, screening, quota) → `/x402/settle` → batcher → final outcome → `200` + `PAYMENT-RESPONSE`; `evt.agentpay.agent_payment.settled` → webhooks `agent_payment.settled`. ✔

**T5 — Agent pays a checkout session:** `POST …/agent_pay/requirements` (agentpay checks the client secret through payments' public session read, then payments `agent_attempts` → rails; agentpay §5) → sentinel verification by agent-signer (S-05) → `POST …/agent_pay {signed}` → matched to the handed-out requirement, verified against the rails execution → relay → indexer → payments success. ✔

**T6 — Test mode end to end:** `sk_test_` session → rails sandbox provider (rails §12 assertions) → sandbox quote/status (sandbox §3–§4) → sentinel sandbox verification (sentinel §9) → SPA "Simulate payment" (payments `simulate_deposit`) → sandbox timeline → rails sandbox feed/verifier → payments → webhooks. Scenarios 00–07 map to the outcome matrix (domain-model §5). ✔

**T7 — Kill switch:** genuine integrity mismatch (rails confirms issued address) → that merchant quarantined; corroborated mismatches across merchants (≥ 3 across ≥ 2 within 15 min, ADR-0028) → sentinel engages (sentinel §5 "Containment", §8) → `evt.sentinel.kill_switch.engaged` → rails `control_flags` (rails §7) and agentpay live x402 settles and live checkout relays paused (agentpay §8) → notify → ops CLI release (Stage 1). A merchant disabling a destination mid-checkout yields `stale`, never the switch (sentinel R5). Spoofed request → refused only (L-SN-01). ✔

## 4. Latency budget chains

| Path | Budget | Composition |
|---|---|---|
| Payer creates attempt | 10 s (public) | Turnstile 2 s ∥ geo (cached) → rails 8 s [compliance 1.5 s + 1Click quote 6 s + DB] → ticket sign < 5 ms |
| Sentinel co-sign | 1.5 s p99 | ticket verify < 1 ms → 1Click status ≤ 1.2 s typical (2 s timeout) → registry query 300 ms → sign < 1 ms |
| x402 verify | ≤ 1 s | signature < 5 ms + simulate ≤ 600 ms + screening (cached) |
| x402 settle | ≤ 8 s wait | batch tick ≤ 150 ms + NEAR finality ~1.2 s + RPC → typical 2–3 s |
| Authz (every request) | ≤ 5 ms p99 cached | LRU/Valkey lookup + JWT sign |
| Merchant create session | ≤ 300 ms p99 | context (cached) + minimum (cached) + one transaction |

## 5. Invariant walk

| Invariant | Enforced by |
|---|---|
| INV-1 Non-custody | No service holds fund-moving keys: relayer/keeper are function-call keys (agentpay §9, billing §8); rails/payments hold API credentials only; vault has no admin methods (vault §3) |
| INV-2 Evidence before success | rails §5 success rule + verifiers/watchers/indexer; payments applies only `rails.execution.settled`; x402 settled only from final tx outcome (agentpay §7) |
| INV-3 Truth wins | session `expiring`, late success, overpayments (payments §5); `uncollectible → paid` (billing §4); `expired_unpaid → refunded` (rails §5) |
| INV-4 Exactly-once effects | outbox/inbox, versions, idempotency keys (foundations §7–§8), unique constraints (statement `bt_once`, deliveries, tags) |
| INV-5 Integer money | `@nearpay-internal/money`, `numeric(78,0)`, bigint fiat (foundations §6) |
| INV-6 Immutable snapshots | destination/fee/settings on the intent; price snapshot per attempt (payments §4) |
| INV-7 Origin refunds | refund recipient fixed from the succeeded attempt (payments §8.4) |
| INV-8 Livemode isolation | three-point provider assertion (rails §12), sandbox rejects live, `livemode` predicate everywhere |
| INV-9 Append-only audit | transitions tables, provider observations, audit events, kill-switch log |

## 6. Verdict

All component audits pass after fixes; system pass 2 is clean; every remaining unknown is an external fact with a spike and a safe default. **The LLD set is ready to move to development** (Stage plan: `roadmap/development-plan.md`).
