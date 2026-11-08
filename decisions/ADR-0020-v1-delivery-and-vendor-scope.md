# ADR-0020: v1 by 2026-11-14 — test-mode stage gates, one mainnet demo, no KYB/screening/FX vendors, AI audits, delegated merges

- **Status:** Accepted
- **Date:** 2026-11-07
- **Deciders:** Owner; Principal Engineer
- **Amends:** ADR-0016 decision 7 (one v1 mainnet demo instead of a Stage 1 demo); ADR-0017 decisions 3–5 (merging pull requests, including money-path pull requests, is delegated to the lead agent); `roadmap/development-plan.md` v1.1 §1 and Stages 2–4; FR-CMP-01 and FR-CMP-02 for v1 (vendor wording only)
- **Supersedes:** Q-VEN-01 (KYB vendor); the compliance Stage 2 scope "KYB vendor (hosted flow, webhooks)" (`lld/compliance.md` §12)
- **Resolves:** Q-VEN-05 for v1 · **Defers:** Q-VEN-02 (`roadmap/open-questions.md`)
- **Invariants touched:** none are changed. INV-1, INV-2, INV-7 and INV-8 are now enforced before merge by automated checks and lead-agent review rather than human review (decision 7).

## Context
- Step 1.4 is green in test mode (traces T1–T7, sandbox scenarios 00–07; HLD revision log 2026-11-07). The Stage 1 mainnet demo (step 1.5) has not run yet.
- The NEAR Hackathon deadline is 2026-11-14 23:59 UTC (ADR-0016). The owner wants all of v1 (Stages 2–4) by that date, built by parallel AI agents, one agent per repo per step.
- Development plan v1.1 has steps that depend on outside parties or on the owner's time, and these cannot fit in seven days:
  - a human-run mainnet demo at the end of every stage;
  - a KYB vendor and an address-screening vendor in Stage 2 (Q-VEN-01, Q-VEN-02). Each needs a business account, a contract and credentials, and all three are human-only (ADR-0016 decision 5);
  - a commercial FX API (Q-VEN-05);
  - two external audits of the Stage 4 vault, which take weeks and cost money;
  - an owner merge for every pull request, plus human CODEOWNERS review on money paths (ADR-0017 decisions 3 and 5; `architecture/security.md` T-15).
- Stage 1 already ships the parts that can replace the vendors: a local OFAC SDN digital-currency list and a blocklist behind the screening API (SP-15, compliance §7), and a manual KYB decision taken by ops (ops CLI; console route in Stage 2).
- 1Click's terms require the integrator to KYB merchants and to screen end users (`reference/landscape.md` §2).
- The ECB publishes its euro reference rates for 29 currencies once per TARGET working day at about 16:00 CET. It publishes them "for information purposes only" and strongly discourages using them for transactions (`reference/landscape.md` §8).

## Decision
1. **Target.** v1 (Stages 2–4) is targeted for **2026-11-14**. Each step of each stage is built by AI agents in parallel, one per repo, under the step gates of the development plan.
2. **Stage gates.** A stage exits when its **test-mode E2E suite is green** against the full Compose stack. The suite runs in `nearpay-testbed`, on the sandbox rail and from a clean `scripts/bootstrap.sh`. The per-stage mainnet demos are dropped. Each demo script becomes a test-mode E2E scenario of its stage. v1 ends with **one mainnet demo**: a script that covers Stages 1–4, run by a human from the entry repo's README and recorded on video. That demo replaces step 1.5 and the step 2.4/3.4/4.4 mainnet demos. It also serves as the hackathon video (ADR-0016 decision 7 as amended).
3. **KYB without a vendor.** The merchant owner submits **minimum business details, self-declared**: legal name, country of incorporation, registration number, business type, website, business address, a representative's name and role, and an expected monthly volume band. No documents are collected, and there is no vendor hosted flow, vendor applicant or vendor webhook. The KYB case records the submission. **Live mode is enabled by an ops decision in the console** that approves the self-declared details. The decision uses the existing manual KYB decision endpoint, is audited (INV-9), and drives `kyb_status` as it does today. FR-ACC-05 is unchanged: live mode still needs `kyb_status = approved` and an active destination. The merchant name shown at checkout is the legal name approved by ops. It is not a third-party-verified identity, so the UI must not claim one.
4. **Address screening stays local.** Every screening context in FR-CMP-02 uses the Stage 1 local OFAC list and blocklist, behind a **vendor-neutral screening interface**: one provider, `local`, with the fail policy unchanged. A vendor (Q-VEN-02) is a new provider behind the same interface later. It needs no contract change.
5. **FX and secondary prices come from keyless public sources**, each behind a provider interface in rails. Fiat FX uses the **ECB reference rates** (daily XML). The independent secondary token price for the dual-source deviation check uses **a public, keyless crypto price feed**. Rails §8 and §10 choose the feed and set the staleness bounds. Those bounds must allow for the ECB's daily, working-days-only publication. Missing or stale data gives `pricing_unavailable`, never a guess (HLD §10, unchanged). No API keys and no paid plans are used.
6. **AI audits for the vault.** The two external audits of the Stage 4 subscription vault are replaced by **independent AI-agent audits**. An audit is a separate agent run that did not write the contract. It works against `lld/contracts-vault.md` V1–V5 and T-10. Each finding is recorded with its fix or an accepted-risk note. Property tests, fuzzing, the gas profile and the on-chain caps stay required. **Mainnet deployment of the vault stays a human-signed action**, as does every item on the ADR-0016/0017 human-only list except merging (decision 7).
7. **Merges are delegated to the lead agent.** The lead agent reviews and merges every pull request, **including money-path pull requests**. The CODEOWNERS files stay in place to name the sensitive paths. For v1 their "human review / two human approvals" rule (ADR-0017 decision 3; T-15; `architecture/repositories.md` §5) is met by the lead agent's review **plus** green CI on those paths: invariant property tests, contract tests and the stage's test-mode E2E suite. A money-path pull request with a red or skipped check is never merged. ADR-0017 decision 5's human-only list loses exactly one item, merging pull requests. Everything else stays human-only: the first push of `main` to a new remote, making repos public, posting, submitting, signing transactions, spending funds, mainnet deployment and providing credentials.

## Consequences
- ➕ v1 has no external calendar dependency apart from the owner's human-only actions: funds, keys, the final demo and the vault deployment.
- ➕ Test-mode E2E gates are repeatable and automated, and they run on every change. The v1 mainnet demo still proves the whole product on mainnet once, under human control.
- ➕ The screening, KYB, FX and price provider interfaces keep vendors pluggable. Adding a vendor later is an implementation change, not a contract change.
- ➖ **Mainnet coverage is thinner.** Stages 2–4 behaviour is first exercised on mainnet in the final demo, so live-provider quirks of the kind found in Stage 1 (live 1Click nulls, `open-questions.md`) surface late and all at once. Refunds, payment links, invoices, auto-debit and the Solana path get no separate mainnet rehearsal. Leave time before the deadline to fix what the demo finds.
- ➖ **Self-declared KYB is weak KYB.** Nothing checks a business's existence, ownership or sanctions status except the ops reviewer and the local OFAC list. Whether self-declared details plus an ops decision meet 1Click's integrator-KYB term and any launch jurisdiction's rules is open (Q-LEG-01). Until then, approve live mode deliberately and only for known merchants, and keep volumes small.
- ➖ **Local screening has narrower coverage than a commercial vendor.** It covers sanctioned addresses only, with no exposure analysis (KYT) and no other risk categories (T-14 and R-13 residuals rise). The OFAC list is refreshed daily (compliance §7).
- ➖ **The ECB rates are for information purposes only, daily, and absent at weekends and on TARGET holidays.** Prices in non-USD fiat currencies drift from market rates between publications, and the ECB discourages transactional use. Merchants pricing in non-USD fiat carry that drift. USD-priced payments that settle in stablecoins are unaffected. Keyless crypto feeds have low rate limits and no SLA, so a feed outage gives `pricing_unavailable` for non-par settlement assets, which fails safe.
- ➖ **AI audits are not equivalent to independent human audits.** Auditor agents share failure modes with the agents that wrote the code, carry no professional liability and have no track record on NEAR contracts. T-10 (vault bug drains mandates) stays at Medium residual with weaker assurance, held down by immutability, per-mandate and global TVL caps, and payer-only withdrawal. **A human audit is recommended before the vault holds material value.** Until then, keep the global TVL cap low.
- ➖ **No human reviews money-path code before it reaches `main`.** A defect that the lead agent and CI both miss reaches the v1 demo build (T-15 and R-09 residuals rise from Low to Medium). Remaining human controls: the owner runs and watches the v1 mainnet demo, signs every mainnet transaction and deployment, and can audit the merged history at any time. **A human review of money-path diffs is recommended before material volume.**
- ➖ The brain and per-repo `AGENTS.md` files still say "human review on money paths". They must be read through this ADR until they are updated.

## Addendum — owner decisions 2026-11-08
- **Approver tenure stays 7 d / 30 d** (identity §5 two-approver rule, v1.8 audit fix): a new owner/admin's approval of a settlement destination counts after 7 days, or after 30 days when two or more other managers held the role in the 30 days before the grant. The owner chose it over a plain 7-day wait, accepting that newly added managers in established teams wait a month before approving payout-wallet changes.
- **Website hero fee example: $0.25** (0.50% stable pair on a $50 example), not the Figma's $0.49.

## Alternatives considered
- **Keep per-stage mainnet demos.** This needs owner time and funds four times in one week, and it makes the stages run one after another. The owner rejected it.
- **Integrate a KYB vendor and a screening vendor (Sumsub/Persona, Chainalysis/TRM).** Contracts, credentials and onboarding are human-only and take longer than the window, and per-check fees apply before there is revenue.
- **Commercial FX API.** Needs an account and a key. A real-time rate adds little for v1's USD-first merchants.
- **Postpone the vault, and with it Stage 4, until external audits finish.** This misses the v1 target. AI audits plus immutability and caps are the owner's chosen trade-off.
- **Keep owner merges for money paths only.** Most Stage 2–4 pull requests touch money paths, so the owner would again be the bottleneck.

## Revisit when
- Before the vault holds material value or any launch beyond known merchants: commission a human audit of the vault, and run a human review of money-path code.
- When the launch jurisdiction is decided (Q-LEG-01), or when 1Click or a partner asks for verified KYB: choose a KYB vendor and reopen Q-VEN-01.
- When volume, non-NEAR-chain exposure or a partner requirement calls for exposure screening: choose a vendor and reopen Q-VEN-02.
- When merchants price in non-USD fiat at volume: add a real-time FX source behind the FX provider interface.
- When a second human engineer joins: restore human CODEOWNERS approval on money paths (ADR-0017 "Revisit when").
