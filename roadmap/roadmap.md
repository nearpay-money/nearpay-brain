# Roadmap

> Version 2.0 · Superseded detail lives in [development-plan.md](development-plan.md) (stages, parallel steps, exit criteria, demo scripts).

| Stage | Outcome (each ends in a mainnet demo) |
|---|---|
| **1 — Pay from any chain + agent pay** | NEAR Hackathon submission (deadline 2026-11-14 23:59 UTC): hosted checkout from any chain via NEAR Intents, sentinel-verified deposit instructions, on-chain evidence, signed webhooks, x402 agent payments settled with signed NEAR Intents transfers — local stack against mainnet |
| **2 — Merchant-ready payments** | Refunds, payment links, customers, team and approvals, KYB/screening vendors, console, reconciliation, multi-currency pricing, docs |
| **3 — Billing + agent commerce** | Invoices, subscriptions (`send_invoice`), dunning, hosted invoices, MCP server, cross-chain x402, CLI |
| **4 — Auto-debit + chain coverage** | Audited subscription vault and keeper, Solana and Bitcoin Tier A — **v1 complete** |
| **5 — v1 live on netcup** | Single netcup server behind Cloudflare, backups, observability, pentest, launch |

## Definition of done (every stage)

- Contracts released in `nearpay-spec` before implementation; consumer contract tests green.
- Invariant property tests (INV-1…9) green for touched aggregates.
- Runbooks for new failure modes added to `operations/runbooks.md`.
- `nearpay-brain` updated in the same change set (ADR if a decision changed).
- Stage demo executed from the entry repo README on mainnet and recorded.
