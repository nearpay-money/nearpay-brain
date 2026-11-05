# Risk Register

> Version 1.0 · Likelihood/Impact: L/M/H. Review monthly and at every phase exit.

| ID | Risk | L | I | Mitigation | Monitor / trigger | Owner |
|---|---|---|---|---|---|---|
| R-01 | **1Click dependency** — no SLA, no webhooks, discretionary recovery (< USD 300 may be refused), fee changes, liability cap ≈ USD 1k | M | H | Provider abstraction (ADR-0003); direct-intents path; watchers make success provider-independent; partner agreement (SP-04) | Provider error rate, quote latency, ToS changes | Architecture |
| R-02 | **Regulatory classification** of a fee-taking non-custodial gateway (MiCA/VASP, money transmission, travel rule) | M | H | Legal opinion per launch jurisdiction before live (Q-LEG-01); compliance as data | Jurisdiction decision | Founders |
| R-03 | **Competition** (PingPay already live on NEAR Intents, x402/MPP) | H | M | Differentiate: provable non-custody, evidence-before-success, DX, instant agent payments, any-asset settlement | Feature parity reviews | Product |
| R-04 | **Name/trademark collision** ("NearPay" used by other companies) | M | M | Trademark search before public launch (Q-BIZ-01); identifiers derive from domain | Legal response | Founders |
| R-05 | **Cloudflare single edge** | L | H | Status page outside Cloudflare; documented direct-origin emergency DNS (requires manual switch) | Edge incidents | Platform |
| R-06 | **netcup single provider** | L | H | Two sites; off-provider backups; IaC rebuild ≤ 8 h, rehearsed twice a year | DR drill results | Platform |
| R-07 | **Vault contract bug** | L | H | Minimal code, immutability, caps, two audits, fuzzing, bug bounty | Audit findings; TVL cap | Contracts |
| R-08 | **Bridge risk** for merchants holding bridged assets in Intents accounts | L | H | Disclosure at destination setup; sweep reminders; recommend chain-address destinations for large balances | Bridge incidents | Product |
| R-09 | **AI-agent-introduced defects** in money paths | M | H | CODEOWNERS two-human review, property tests, invariant checks, no prod credentials, GitOps | Escaped defects | Engineering |
| R-10 | **Relayer key compromise** | L | L | Function-call keys only (cannot steal); allowance caps; rotation | Allowance drain alerts | Platform |
| R-11 | **x402 NEAR methods non-standard** | M | M | Implement PR #3370 method too; upstream `intents-transfer` | Upstream status | Agents |
| R-12 | **Shared shard throughput** for agent payments | M | M | SP-07; quotas; v2 channel scheme | Gas/s on `intents.near` | Agents |
| R-13 | **Sanctioned funds** reach merchants (non-custodial cannot block in flight) | M | M | Pre-screening, geo policy, post-screen cases, KYB | Screening hit rate | Compliance |
| R-14 | **Payer self-inflicted loss** (wrong chain, missing memo) harms reputation | M | M | Wallet-connect default, guards, support playbook with provider (SP-05) | Support tickets | Product |
| R-15 | **Oracle/peg failure** | L | M | Dual source, deviation and depeg guards | Deviation alerts | Rails |
| R-16 | **EU-only hosting latency** for US/Asia payers | M | L | Static checkout on CDN; API latency budget includes +150–250 ms; MNZ/SIN sites later (ADR-0011 revisit) | p95 by country | Platform |
| R-18 | **v1 single host** — host loss = full outage; weaker same-host isolation between services | M | M | WAL-G off-host (RPO ≤ 60 s), restore runbook (RTO ≤ 4 h), per-service DB roles and secrets, hardened containers; move to ADR-0011 topology after v1 | Host health, restore drill | Platform |
| R-19 | **Hackathon deadline** (2026-11-14 23:59 UTC) forces scope cuts in Stage 1 | H | M | Stage 1 scoped to checkout + agent pay; every step demoable; cut list pre-agreed in the development plan | Daily burn-down | Founders |
| R-17 | **Sentinel on the critical path** (fail-closed) reduces availability | L | M | ≥ 2 replicas per site on different nodes; stateless; 400 ms budget; SLO tracked | Co-sign error rate | Security |
