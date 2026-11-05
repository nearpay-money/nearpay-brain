# Development Plan — stages and steps

> Version 1.0 · Baseline: HLD v1.1, LLD set v1.1 (dev-ready), ADR-0015/0016 · Owner: Principal Engineer
> v1 = Stages 1–4 (local Docker Compose against NEAR mainnet). Stage 5 moves v1 onto one netcup server.

## 1. Rules for every stage

- **Each stage ends in a mainnet demo** that a human runs from the entry repo's README, recorded on video.
- **Steps are gates.** Inside a step, every listed repo is built **in parallel** (one AI agent or engineer per repo). A step starts only when the previous step's exit criteria pass.
- **Contract freeze:** the first step of each stage releases `@nearpay/spec` / `@nearpay-internal/spec` (and `nearpay-kit`) at a tagged version to the local registry; repos in later steps of that stage build against that version only. A contract change mid-stage is a change request: spec PR → new patch version → affected repos bump in the same step.
- **Definition of done per repo:** LLD sections for the stage implemented; unit + property + integration + contract tests green; `AGENTS.md` rules respected; money-path changes reviewed by a human (CODEOWNERS); brain updated in the same change set; commit per finished feature.
- **Human-only actions** (rules + ADR-0016): providing credentials and accounts, signing mainnet transactions, spending funds, pushing, making repos public, posting, submitting.

## 2. Stage overview

```mermaid
flowchart LR
  S1[Stage 1<br/>Pay from any chain + agent pay<br/>NEAR Hackathon] --> S2[Stage 2<br/>Merchant-ready payments<br/>refunds, links, KYB, console]
  S2 --> S3[Stage 3<br/>Billing + agent commerce<br/>invoices, subscriptions, MCP, cross-chain x402]
  S3 --> S4[Stage 4<br/>Auto-debit + chain coverage<br/>vault, Solana, Bitcoin]
  S4 --> S5[Stage 5<br/>v1 live on netcup<br/>server, Cloudflare, backups, launch]
```

## 3. Stage 1 — Pay from any chain + agent pay (NEAR Hackathon)

**Deadline:** code, README and demo ready by **2026-11-14 23:59 UTC** (repos public, X post with video, submission — human actions). Tracks: Real-World Money, Agent Economy.

| Step | Repos built in parallel | Scope (LLD sections tagged Stage 1) | Exit criteria |
|---|---|---|---|
| **1.1 Foundations** | `nearpay-spec`, `nearpay-kit`, `nearpay-deploy`, `nearpay` | spec: Stage 1 public/internal routes, events, webhook types, x402 scheme, error catalogue, test vectors · kit: all packages · deploy: Compose local profile (Postgres init, NATS, Valkey, Traefik routes + ForwardAuth, egress proxy, Mailpit, Verdaccio registry), key seeding, `up.sh` · entry: scripts | `@nearpay/spec@0.1.0` + kit `0.1.0` in local registry; `scripts/up.sh local` boots platform containers healthy; spec compat test green |
| **1.2 Services** | `nearpay-identity`, `nearpay-payments`, `nearpay-rails`, `nearpay-sentinel`, `nearpay-agentpay`, `nearpay-compliance`, `nearpay-webhooks`, `nearpay-notify`, `nearpay-sandbox` | each service's Stage 1 scope (LLD §16) with Testcontainers integration tests and spec contract tests; mocks of peers generated from spec | every service image builds, migrates, passes its tests, and is healthy in Compose |
| **1.3 Clients** (starts with 1.2, against spec mocks) | `nearpay-checkout`, `nearpay-js`, `nearpay-dashboard`, `nearpay-agentkit`, `nearpay-node`, `nearpay-testbed` | checkout hosted + embed, NEAR/EVM wallets, manual path, sentinel verification · js loader + React · dashboard onboarding (signup, destination + attestation, keys, webhooks, payments list, agent payments) · `@nearpay/x402` + `@nearpay/agent-signer` · node SDK subset (sessions, intents, webhook verify) · testbed: merchant-demo, `demo:checkout`, `demo:agent`, E2E suites | components pass unit + mock-backed Playwright tests |
| **1.4 Integration** | `nearpay-testbed` (lead), fixes in any repo | full stack in Compose; sandbox scenarios 00–07 green; traces T1–T7 (`lld/audits/system.md`) green in test mode; README "Try it" steps 1–5 verified by the `readme-check` job | all E2E green on a clean machine from `scripts/bootstrap.sh` |
| **1.5 Mainnet demo** | `nearpay-testbed`, `nearpay` (README), owner | owner provides: NEAR accounts (merchant demo, treasury, relayer + 4 function-call keys), small funds (USDC on Base or Arbitrum, USDC in a NEAR Intents balance for the agent), optionally a 1Click API key; ops CLI KYB approval locally | **Demo script below** passes on mainnet; video recorded; README step 6 verified |

**Stage 1 mainnet demo script**
1. Merchant signs up, adds a USDC **NEAR Intents account** destination, signs the attestation with a NEAR wallet; ops CLI approves KYB → account live.
2. Merchant backend (`merchant-demo`) creates a USD 2.00 checkout session.
3. Payer opens the hosted checkout, connects an EVM wallet holding USDC on Base, sees the sentinel-verified merchant name and destination fingerprint, pays.
4. Within ~1–2 minutes the merchant's Intents balance shows USD 2.00 USDC + tag; `payment_intent.succeeded` arrives at `merchant-demo` with a valid signature; dashboard shows the evidence link.
5. An AI agent calls `merchant-demo/premium`, receives `402`, signs a NEAR Intents transfer, receives the resource with `PAYMENT-RESPONSE` in ~2 s; `agent_payment.settled` webhook arrives.

**Stage 1 cut list (applied in this order only if 1.4 is late):** dashboard pages beyond onboarding → seed CLI · embedded mode → hosted only · human Intents-balance path → agents only · EVM watchers → provider SUCCESS + receipt verifier only. Never cut: sentinel co-signing, evidence-before-success, webhook signatures, test mode.

## 4. Stage 2 — Merchant-ready payments

| Step | Repos in parallel | Scope |
|---|---|---|
| 2.1 Contracts | `nearpay-spec`, `nearpay-kit` | refunds, payment links, customers, team, restricted keys, KYB, console routes, reconciliation, Explorer, FX |
| 2.2 Services | `nearpay-payments`, `nearpay-rails`, `nearpay-identity`, `nearpay-compliance`, `nearpay-webhooks`, `nearpay-notify`, `nearpay-agentpay`, `nearpay-sentinel` | refunds + legs (route + Intents), payment links, customers, review decisions, reconciliation · rails refunds, Explorer feed (owner token), secondary prices + FX (non-par settlement, multi-currency) · identity team, invitations, two-approver rule, restricted keys, recovery · compliance vendor screening + KYB vendor, re-screening · webhooks rotation/resend/delivery API · notify KYB/refund/member templates · agentpay Intents refunds relay · sentinel two-person kill-switch release |
| 2.3 Clients | `nearpay-dashboard`, `nearpay-console`, `nearpay-checkout`, `nearpay-node`, `nearpay-docs` | refunds with wallet signing, team, KYB, customers, links · console (merchants, KYB, cases, holds, findings, reconciliation) · payment-link route · full node SDK · docs site |
| 2.4 Integration + mainnet demo | `nearpay-testbed` | **Demo:** refund a mainnet payment back to the payer's origin chain (merchant signs in the dashboard); pay a payment link from a different chain; show the daily reconciliation with zero diffs |

## 5. Stage 3 — Billing and agent commerce

| Step | Repos in parallel | Scope |
|---|---|---|
| 3.1 Contracts | `nearpay-spec`, `nearpay-kit` | billing resources and events, hosted invoice, events stream, `near-intents` x402 method |
| 3.2 Services | `nearpay-billing`, `nearpay-payments`, `nearpay-agentpay`, `nearpay-rails`, `nearpay-webhooks`, `nearpay-notify` | products/prices/tax rates/invoices/subscriptions (`send_invoice`), dunning, PDFs · internal invoice sessions · cross-chain x402 (rails owner `agentpay`) · events SSE stream · billing emails |
| 3.3 Clients | `nearpay-checkout`, `nearpay-dashboard`, `nearpay-agentkit`, `nearpay-cli`, `nearpay-docs` | hosted invoices · billing pages · MCP server · CLI (`login`, `listen`, `trigger`) · billing + agent guides |
| 3.4 Integration + mainnet demo | `nearpay-testbed` | **Demo:** invoice paid from Solana/EVM into a NEAR destination; subscription renewal invoice issued and paid; an MCP-driven agent pays a checkout session and a cross-chain x402 resource |

## 6. Stage 4 — Auto-debit and chain coverage (completes v1)

| Step | Repos in parallel | Scope |
|---|---|---|
| 4.1 Contract + contracts | `nearpay-contracts`, `nearpay-spec` | subscription vault (tests, fuzzing, gas profile), vault event schemas; **two external audits start** |
| 4.2 Services | `nearpay-billing`, `nearpay-rails`, `nearpay-identity`, `nearpay-sentinel` | mandates + keeper + auto-debit, vault event forwarding · Solana + Bitcoin watchers/verifiers (Tier A) · Solana/BIP-322 attestations · sentinel coverage for new chains |
| 4.3 Clients | `nearpay-checkout`, `nearpay-dashboard`, `nearpay-agentkit` | Solana wallets, mandate funding UI, mandate management |
| 4.4 Integration + mainnet demo | `nearpay-testbed`, owner | audits closed; vault deployed (keys deleted, human-signed); **Demo:** payer funds a mandate, keeper collects period 1 on mainnet, merchant receives into its Intents account, payer cancels and withdraws the remainder; a Solana payer completes a checkout |

**v1 complete** when Stage 4 exits: every LLD section tagged Stages 1–4 is implemented and every stress-test scenario is automated in the testbed.

## 7. Stage 5 — v1 live on a netcup server

| Step | Repos in parallel | Scope |
|---|---|---|
| 5.1 Platform | `nearpay-infra`, `nearpay-deploy` | Ansible host hardening + docker-host + WireGuard admin; Terraform Cloudflare (DNS, proxy, AOP, WAF, Turnstile, Access) + EU object storage; Compose server profile; WAL-G + restore drill; observability on server; GitHub remotes + GHCR/registry switch (owner pushes) |
| 5.2 Production readiness | `nearpay-notify`, `nearpay-web`, `nearpay-docs`, `nearpay-testbed` | Postmark/SES + bounces; marketing + legal pages; public docs; external status page; canary schedule; load test at P1 peak ×3 on the server |
| 5.3 Security + go-live | all, owner | external pentest (checkout, identity, sentinel), legal opinion for launch jurisdiction (Q-LEG-01), trademark check (Q-BIZ-01), runbook drills RB-01/RB-02/RB-07/RB-11 |
| 5.4 Launch demo | owner | **Demo:** `https://pay.nearpay.money` live checkout paid on mainnet end to end on the netcup server; hosted agent-payment demo; status page green |

## 8. Repo × stage matrix

| Repo | S1 | S2 | S3 | S4 | S5 |
|---|---|---|---|---|---|
| nearpay (entry) | ● | ○ | ○ | ○ | ● |
| nearpay-spec, nearpay-kit | ● | ● | ● | ● | ○ |
| nearpay-identity | ● | ● | | ● | |
| nearpay-payments | ● | ● | ● | | |
| nearpay-rails | ● | ● | ● | ● | |
| nearpay-sentinel | ● | ● | | ● | |
| nearpay-agentpay | ● | ● | ● | | |
| nearpay-compliance | ● | ● | | | |
| nearpay-webhooks | ● | ● | ● | | |
| nearpay-notify | ● | ● | ● | | ● |
| nearpay-sandbox | ● | ○ | ○ | ○ | |
| nearpay-billing | | | ● | ● | |
| nearpay-checkout, nearpay-js | ● | ● | ● | ● | |
| nearpay-dashboard | ● | ● | ● | ● | |
| nearpay-console | | ● | | | |
| nearpay-agentkit | ● | | ● | ● | |
| nearpay-node | ● | ● | | | |
| nearpay-cli | | | ● | | |
| nearpay-docs | | ● | ● | | ● |
| nearpay-web | | | | | ● |
| nearpay-contracts | | | | ● | |
| nearpay-deploy | ● | ○ | ○ | ○ | ● |
| nearpay-infra | | | | | ● |
| nearpay-testbed | ● | ● | ● | ● | ● |

● main work · ○ small additions (new routes, scenarios, config)
