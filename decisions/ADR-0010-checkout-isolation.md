# ADR-0010: Checkout as an isolated static SPA; widget = iframe + postMessage

- **Status:** Accepted · **Date:** 2026-11-05 · **Invariants:** INV-1 (practical), INV-2

## Context
The most direct way to steal a crypto payment is to change the deposit address the payer sees. Merchant pages run arbitrary third-party scripts. The checkout must also survive a compromise of NearPay's API tier as far as possible.

## Decision
- `pay.nearpay.money` is a static SPA (Vite + React) served from the CDN with strict CSP (nonces, no `unsafe-inline`, `connect-src` limited to the NearPay API, wallet relays and Turnstile), HSTS preload, and no third-party runtime scripts except Cloudflare Turnstile and bundled wallet adapters.
- The widget (`js.nearpay.money/v1/nearpay.js`, SRI-pinned, immutable versioned URLs) only mounts an iframe of the checkout and exchanges a versioned postMessage protocol (origin-checked both ways). Deposit addresses, amounts and wallet interactions never touch the merchant's DOM.
- Checkout is built and deployed by a pipeline separate from backend services (distinct credentials); it shows the merchant's KYB-verified name and the destination fingerprint.
- **Two-party release:** checkout renders a deposit address only with a valid `nearpay-sentinel` Ed25519 co-signature over `(attempt_id, deposit_address, memo, origin_asset, amount_in, recipient, destination_asset, deadline)`, verified against a public key pinned at build time. Once SP-01 confirms the scheme, it additionally verifies 1Click's own quote signature.
- Wallets: NEAR via NEAR Connect, EVM via WalletConnect/injected (EIP-6963), Solana wallet standard; other chains via deposit address.

## Consequences
- ➕ Merchant-page compromise cannot swap addresses; smaller attack surface; CDN-fast.
- ➖ Less layout freedom for merchants (themeable via appearance API, not arbitrary CSS).

## Alternatives considered
- **Inline DOM components (Elements-style without iframe):** better styling, unacceptable address-swap risk.
- **SSR checkout:** adds a server in the payer path without security benefit.

## Revisit when
Browser platform offers stronger isolation primitives, or merchants need native mobile SDKs (v2: native sheets wrapping the same SPA).
