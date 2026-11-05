# HLD Stress Test

> Version 1.0 · Method below · Result: **all findings closed in the design or tracked as owned spikes/risks**. Re-run this document's scenario list on every ADR that touches money paths.

## 1. Method

Each pass walks concrete scenarios against the invariants (INV-1…9), the requirements and the degradation matrix, and asks three questions: *Can money be lost or misdirected? Can state lie? Does a failure cascade?* Every defect becomes a finding (`F-nn`) with a design change and a pointer to where it now lives.

| Pass | Focus | Scenarios | Findings |
|---|---|---|---|
| 1 | A — money correctness · B — concurrency and consistency | 24 | F-01…F-09 |
| 2 | C — dependency failure · D — security and abuse · E — operations | 26 | F-10…F-20 |
| 3 | F — product and compliance edges · G — capacity | 14 | F-21…F-27 |
| 4 | Consistency and verification (cross-doc links, IDs, diagrams render, repo list matches disk) | automated | see `README.md` §Verification |

Legend: ✅ handled by the design as drafted · 🔧 design changed (finding) · ⚠ depends on an external answer (spike/risk).

## 2. Scenarios and outcomes

### A. Money correctness

| # | Scenario | Result |
|---|---|---|
| A1 | Exact payment, cross-chain | ✅ EXACT_OUTPUT → verify → succeeded |
| A2 | Payer overpays in one attempt | ✅ excess refunded by provider to `refund_to`; recorded |
| A3 | Payer underpays within slippage band (≥ `minAmountIn`) | ✅ treated as paid; verifier checks evidence ≥ expected |
| A4 | Payer underpays below `minAmountIn` | 🔧 F-03 top-up UX made conditional on SP-03 |
| A5 | Payer starts a 90-min Bitcoin route 2 minutes before the session expires; merchant releases inventory at expiry | 🔧 F-01 attempt deadline ≤ session window; slow routes hidden; opt-in late payments |
| A6 | Payer creates attempt 2 after sending to attempt 1's address | 🔧 F-02 one visible attempt; old address hidden with warning; `overpaid` path |
| A7 | Two attempts both settle | ✅ first wins; `overpayments[]`; refund recommended |
| A8 | Settlement arrives after session expiry | ✅ `expiring` state; session `complete` with `late: true` (INV-3) |
| A9 | Provider `SUCCESS`, destination shows less than expected | ✅ `requires_review`, merchant tolerance |
| A10 | Provider `FAILED` after deposit detected | 🔧 F-05 → `held` + case (funds exist somewhere) instead of terminal `failed` |
| A11 | Unknown provider status value | ✅ `held` + alert; never terminal |
| A12 | Partial refund then second refund exceeding the remainder, concurrently | ✅ row lock + refundable formula |
| A13 | Refund fees — who pays? | 🔧 F-04 `fee_mode`, no NearPay fee on refunds |
| A14 | Merchant changes destination while sessions are open | ✅ snapshot per intent; cool-off |
| A15 | Stablecoin depegs during checkout | ✅ depeg guard disables par policy |
| A16 | Fee charged differs from expected (1Click split/floor) | ✅ `fee_adjustment` statement line; fee schedule as data |

### B. Concurrency and consistency

| # | Scenario | Result |
|---|---|---|
| B1 | `payments` crashes after `rails` obtained a live quote | 🔧 F-06 attempt row written first (`quoting`), idempotent execution, recovery worker |
| B2 | Observations arrive out of order (Explorer older than hot poll) | ✅ provider timestamp + version checks; stale = no-op |
| B3 | Same event delivered twice to billing | ✅ inbox dedupe |
| B4 | Compromised `payments`/`rails` returns attacker deposit address; watchdog alerts only *after* payer paid | 🔧 F-07 sentinel **co-signs** instructions; checkout refuses without co-signature |
| B5 | Watchdog compares against the payments snapshot (same blast radius) | 🔧 F-08 sentinel compares against identity's registry via its own read-replica role |
| B6 | Postgres primary failover makes `/authz` fail → every API call fails | 🔧 F-09 authz reads cache → Valkey → replica |
| B7 | Billing generates the same renewal twice after scheduler restart | ✅ job key `(subscription, period_start)` unique |
| B8 | Webhook consumer receives `processing` after `succeeded` | ✅ `sequence` on object; docs instruct discard |

### C. Dependency failure

| # | Scenario | Result |
|---|---|---|
| C1 | 1Click quotes down for 2 h | ✅ cross-chain hidden; Intents-native still works; status page |
| C2 | 1Click status down, swaps progressing | ✅ watchers/indexer/Explorer detect; verify independently |
| C3 | Explorer API down | ✅ hot poller + watchers |
| C4 | OpenBao sealed after a power event | 🔧 F-10 OpenBao off hot path (in-memory keys, envelope encryption) |
| C5 | Price feeds down | 🔧 F-11 only non-par settlement assets blocked |
| C6 | Turnstile down | 🔧 F-12 degrade to stricter per-IP limits |
| C7 | NEAR block stream provider down | 🔧 F-13 two block sources + RPC catch-up |
| C8 | Screening vendor down | ✅ policy: fail-closed ≥ USD 1k, post-screen below |
| C9 | Email provider down | ✅ fallback provider |
| C10 | NATS cluster lost | ✅ outbox buffers; rebuild + republish |
| C11 | NUE site lost mid-checkout | ✅ funds safe (not in our path); VIE promotion ≤ 30 min; trackers resume from cursor − 30 min |
| C12 | netcup account suspended | ✅ off-provider backups + IaC rebuild ≤ 8 h; DNS at Cloudflare |

### D. Security and abuse

| # | Scenario | Result |
|---|---|---|
| D1 | Account takeover changes destination | ✅ passkey step-up, attestation, 2 approvers, cool-off, notifications |
| D2 | Malicious script on merchant page | ✅ iframe isolation; address never in merchant DOM |
| D3 | Leaked `sk_live_` key | ✅ cannot touch destinations/team; scopes; secret scanning |
| D4 | Webhook URL points at `169.254.169.254` | ✅ egress proxy blocks; DNS pinning |
| D5 | Agent replays a signed payment for another resource | ✅ Verifier nonce + binding memo |
| D6 | Agent spams tiny valid payments to burn relayer gas | 🔧 F-15 minimum absolute fee + quotas + allowance caps |
| D7 | Merchant registers an exchange deposit address needing a memo | 🔧 F-14 memo-requiring destinations unsupported |
| D8 | Payer gives an exchange deposit address as refund address | 🔧 F-16 UI guard + docs; ToS question Q-LEG-03 |
| D9 | Quote flooding via payment links | ✅ Turnstile + per-session/IP/merchant limits; indicative prices |
| D10 | AI coding agent weakens a state-machine guard | ✅ CODEOWNERS on money paths, property tests, no prod credentials |
| D11 | Insider suspends a competitor's merchant account | ✅ two-person rule + audit |
| D12 | Sanctioned address pays a merchant | ✅ post-screen case; cannot claw back (documented, R-13) |

### E. Operations

| # | Scenario | Result |
|---|---|---|
| E1 | In-site CNPG failover flips the Cloudflare LB to unpromoted VIE | 🔧 F-17 probe thresholds > failover time; NUE as fallback pool |
| E2 | Merchant allow-lists webhook IPs; failover changes source IP | 🔧 F-18 publish both sites' IPs |
| E3 | Canary runs new code against old schema | 🔧 F-19 expand/contract migrations, pre-sync hook, separate role |
| E4 | Split brain after manual promotion | ✅ fencing step in RB-02; health only green on writable primary |
| E5 | GHCR outage during incident | ✅ Zot pull-through cache |
| E6 | HLD and repo docs disagree on the service list (sentinel missing) | 🔧 F-20 single generator for repo docs + skeletons; HLD updated |

### F. Product and compliance edges

| # | Scenario | Result |
|---|---|---|
| F1 | EU merchant needs VAT on invoices | 🔧 F-21 manual `TaxRate` in billing |
| F2 | Vault `ft_transfer_call` partially refunded by `intents.near` | 🔧 F-22 resolve callback credits unused amount, period not advanced |
| F3 | Payer cancels auto-debit mid-period | ✅ remainder returned; merchant notified |
| F4 | GDPR erasure vs. AML retention | ✅ pseudonymise PII, retain on-chain records per policy |
| F5 | Jurisdiction chosen later | ✅ geo policy and retention as data |
| F6 | Merchant wants fees borne by payer vs merchant | ✅ `fee_bearer` |
| F7 | Wrong-chain send to an EVM deposit address | ⚠ prevention UX; recovery depends on provider (SP-05) |

### G. Capacity

| # | Scenario | Result |
|---|---|---|
| G1 | 30+ status polls per payment at scale exceed any plausible provider limit | 🔧 F-23 deposit-signal polling (≈ 10 calls/payment) |
| G2 | Success detection throttled by provider limits | 🔧 F-24 destination watchers + amount tags (provider-independent success on Tier A) |
| G3 | Agent batch of 50 intents exceeds max gas per transaction | 🔧 F-25 gas-bounded batching |
| G4 | 500 agent payments/s on a single shared contract shard | 🔧 F-26 target provisional (SP-07); v2 channel scheme |
| G5 | Provider observations bloat the primary DB (≈ 30 GB/month at 1M payments) | 🔧 F-27 archive observations > 90 days to S3 (Parquet), keep hash |
| G6 | Flash sale: 10k sessions in 5 min | ✅ live quotes only on confirmation; quote pool concurrency; Turnstile |
| G7 | Webhook backlog after a 1 h outage | ✅ per-endpoint concurrency + rate limits; oldest-first with jitter |

## 3. Findings log

| ID | Severity | Finding | Change | Where |
|---|---|---|---|---|
| F-01 | High | Attempt deadlines could outlive the merchant's session window | Deadline ≤ `expires_at`; hide slow routes; opt-in late payments | FR-PAY-09, HLD §10, flows §2 |
| F-02 | Medium | Multiple visible deposit addresses invite double payment | One visible attempt; warning copy | flows §2, domain-model §5 |
| F-03 | Medium | Top-up after `INCOMPLETE_DEPOSIT` not confirmed by provider docs | Conditional UX; SP-03 | domain-model §4.3, §5 |
| F-04 | Medium | Refund fee semantics undefined | `fee_mode`; no NearPay fee on refunds | FR-REF-04, flows §4 |
| F-05 | High | `FAILED` after a deposit would drop tracking of real funds | Map to `held` + case | domain-model §4.3 |
| F-06 | High | Crash window between quote and attempt persistence | Attempt-first + idempotent execution + recovery worker | flows §2, domain-model §4.3 |
| F-07 | Critical | Watchdog could only alert after a payer paid an attacker | Sentinel co-signature required before rendering | HLD §6, ADR-0010, FR-PAY-10, security T-03 |
| F-08 | High | Watchdog shared the payments blast radius | Compare against identity registry via own replica role | HLD §6, security T-03 |
| F-09 | High | Auth depended on the Postgres primary | Cache → Valkey → replica | ADR-0014 |
| F-10 | High | OpenBao on the request hot path | In-memory keys, envelope encryption | HLD §8, security §3 |
| F-11 | Medium | Price feeds blocked all payments | Only non-par settlement needs pricing | HLD §10, §18 |
| F-12 | Medium | Turnstile outage blocked checkout | Degrade to stricter limits | HLD §18 |
| F-13 | Medium | Single NEAR block source | Two sources + RPC catch-up | infrastructure §5 |
| F-14 | High | Memo-requiring merchant destinations can lose funds | Unsupported in v1 | ADR-0005, FR-SET-05 |
| F-15 | Medium | Relayer gas griefing | Minimum absolute fee, quotas | HLD §11, ADR-0009 |
| F-16 | Medium | Exchange refund addresses | UI guard; legal question | flows §2, open-questions |
| F-17 | High | LB flapping during in-site failover | Thresholds + fallback pool | infrastructure §4 |
| F-18 | Low | Webhook IP allow-lists break on failover | Publish both sites | infrastructure §2 |
| F-19 | Medium | Schema changes vs. canary | Expand/contract | infrastructure §6 |
| F-20 | Low | Docs drift on repo list | Generated from one definition | repositories.md |
| F-21 | Low | No tax on invoices | Manual tax rates | FR-BIL-01 |
| F-22 | High | Vault callback partial refund unhandled | Credit back; period not advanced | ADR-0008 |
| F-23 | High | Polling budget unsustainable | Deposit-signal polling | HLD §10 |
| F-24 | High | Success detection bound to provider limits | Watchers + amount tags | HLD §10, domain-model §3, ADR-0012 |
| F-25 | Medium | Batch size could exceed max gas | Gas-bounded batches | ADR-0009 |
| F-26 | Medium | Shared shard throughput | Provisional target; v2 channels | ADR-0009, NFR-SCL-01 |
| F-27 | Medium | Observation storage growth | Archive > 90 days | capacity §4 |

## 4. Exit criteria (met)

- No open Critical/High finding without a design change. ✅
- Every ⚠ item has an owner and a spike or risk entry (`roadmap/open-questions.md`, `quality/risks.md`). ✅
- Degradation matrix shows *Safe* money outcome for every single-component failure. ✅
- Docs consistency verified (pass 4). ✅
