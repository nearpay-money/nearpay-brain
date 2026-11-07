# Roadmap

> Version 2.1 (ADR-0020: test-mode stage gates, one v1 mainnet demo, no vendors in v1; v1 targeted for 2026-11-14) · Version 2.0 · Superseded detail lives in [development-plan.md](development-plan.md) (stages, parallel steps, exit criteria, demo scripts).

| Stage | Outcome (each exits on green test-mode E2E; one mainnet demo at the end of v1 — ADR-0020) |
|---|---|
| **1 — Pay from any chain + agent pay** | NEAR Hackathon submission (deadline 2026-11-14 23:59 UTC): hosted checkout from any chain via NEAR Intents, sentinel-verified deposit instructions, on-chain evidence, signed webhooks, x402 agent payments settled with signed NEAR Intents transfers — local stack against mainnet |
| **2 — Merchant-ready payments** | Refunds, payment links, customers, team and approvals, self-declared KYB with ops approval and local OFAC screening (no vendors in v1, ADR-0020), console, reconciliation, multi-currency pricing (ECB FX + keyless public crypto price feed), docs |
| **3 — Billing + agent commerce** | Invoices, subscriptions (`send_invoice`), dunning, hosted invoices, MCP server, cross-chain x402, CLI |
| **4 — Auto-debit + chain coverage** | AI-audited subscription vault (human audit recommended before material value, ADR-0020) and keeper, Solana and Bitcoin Tier A — **v1 complete** after the single v1 mainnet demo (target 2026-11-14) |
| **5 — v1 live on netcup** | Single netcup server behind Cloudflare, backups, observability, pentest, launch |

## Definition of done (every stage)

- Contracts released in `nearpay-spec` before implementation; consumer contract tests green.
- Invariant property tests (INV-1…9) green for touched aggregates.
- Runbooks for new failure modes added to `operations/runbooks.md`.
- `nearpay-brain` updated in the same change set (ADR if a decision changed).
- Stage E2E suite green in test mode from a clean bootstrap (ADR-0020). Mainnet: one v1 demo after Stage 4, executed from the entry repo README and recorded; Stage 5 ends with its launch demo on the netcup server.
