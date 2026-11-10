# Core Flows

> Version 1.2 (§1 onboarding: open onboarding — automatic KYB decision, no ops step, ADR-0024) · Version 1.1 (§1 onboarding: v1 self-declared KYB, ADR-0020) · Version 1.0 · Sequence-level detail for HLD §7. Service names = repos without the `nearpay-` prefix.

## 1. Merchant onboarding and settlement destination

```mermaid
sequenceDiagram
  autonumber
  actor M as Merchant owner
  participant D as Dashboard SPA
  participant ID as identity
  participant C as compliance
  participant O as Ops console
  participant W as Merchant wallet
  M->>D: sign up (email + passkey)
  D->>ID: create account (test mode on immediately)
  M->>D: save business details (registration number optional)
  D->>C: submit self-declared business details (KYB case)
  C->>C: same tx: kyb.submitted + automatic decision (actor system; geo policy hit = rejected, ADR-0024)
  Note over O,C: NP_KYB_MODE=manual only: ops KYB decision in the console
  C-->>ID: evt compliance.kyb.approved
  M->>D: add destination (asset, chain or Intents account, address)
  D->>ID: POST /v1/settlement_destinations (step-up passkey)
  ID->>C: screen address (sync)
  ID-->>D: attestation challenge (account, destination, nonce, expiry)
  D->>W: signMessage (NEP-413 / EIP-191 / Solana / BIP-322)
  W-->>D: signature
  D->>ID: submit attestation
  ID->>ID: verify signature, status = cooling_off (24 h), second approver if required
  ID-->>M: email + dashboard notice to all owners
  Note over ID: after cool-off: status = active, evt identity.settlement_destination.activated
  ID-->>ID: live_enabled when KYB approved and >= 1 active destination
```

## 2. Checkout — cross-chain, any asset (the main path)

```mermaid
sequenceDiagram
  autonumber
  participant MS as Merchant server
  participant P as payments
  participant ID as identity
  actor U as Payer
  participant CK as Checkout SPA
  participant SN as sentinel
  participant R as rails
  participant C as compliance
  participant OC as 1Click
  participant CH as Chains / Verifier
  MS->>P: POST /v1/checkout/sessions (Idempotency-Key, sk_live)
  P->>ID: settlement context (destination, fee schedule, live_enabled)
  P-->>MS: cs_ + url + client_secret (intent snapshots fixed)
  U->>CK: open pay.nearpay.money/c/cs_...
  CK->>P: GET session (client_secret) + Turnstile
  CK->>P: GET payment options (connected wallet balances or chosen chain)
  P->>R: indicative prices (token registry, no live quote)
  U->>CK: choose asset (wallet-connect gives refund_to, manual path asks for it)
  CK->>P: POST attempt (origin_asset, refund_to)
  P->>R: create execution (EXACT_OUTPUT, recipient = snapshot, appFees = snapshot, deadline)
  R->>C: screen refund_to (+ geo verdict from edge)
  R->>OC: POST /v0/quote (dry = false, partner JWT)
  OC-->>R: depositAddress, memo, amountIn, minAmountIn, deadline, signature
  R->>R: assert recipient, asset, amount, appFees match request, store observation
  R-->>P: execution awaiting_deposit
  P-->>CK: deposit instructions (address, memo, amount, deadline)
  CK->>SN: request co-signature (attempt id, deposit address)
  SN->>OC: GET /v0/status by depositAddress (own credentials)
  SN->>SN: recipient and asset match an active destination in identity registry, amount and deadline match attempt
  SN-->>CK: Ed25519 co-signature over instructions
  CK->>CK: verify with pinned sentinel key, only then render address and QR
  alt wallet-connect
    CK->>U: wallet prompt (transfer to depositAddress)
    U->>CH: sign and broadcast
    CK->>P: tx hash
    P->>R: submit tx hash
    R->>OC: POST /v0/deposit/submit
  else send from anywhere
    U->>CH: send from exchange/wallet (QR, memo)
  end
  loop hot poll + Explorer feed
    R->>OC: GET /v0/status
  end
  OC-->>R: SUCCESS (+ destination tx hashes)
  R->>CH: verify destination tx (recipient, asset, amount, finality)
  R-->>P: evt rails.execution.settled (evidence)
  P->>P: attempt succeeded, intent succeeded, session complete, statement line
  P-->>CK: SSE: succeeded
  CK->>U: redirect success_url
  P-->>MS: webhook payment_intent.succeeded, checkout.session.completed (via webhooks)
```

**Rules embedded in the flow**
- Live quotes are created only on explicit payer confirmation (Turnstile-protected, ≤ 5 live attempts per session, per-IP and per-merchant limits) — browsing uses indicative prices.
- **Crash safety:** `payments` inserts the attempt (`quoting`) *before* calling `rails`; the execution call is idempotent on `attempt:{id}`; a recovery worker resumes attempts stuck in `quoting`. An orphaned quote is harmless (no one sees its address).
- **Two-party release:** no address is rendered without sentinel's co-signature (fail-closed).
- The deposit address is displayed only inside `pay.nearpay.money` (never in the merchant's DOM).
- **One visible attempt:** creating a new attempt hides the previous address with "If you already sent to the previous address, don't send again — we'll detect it"; the previous execution stays tracked until terminal.
- **Window rule:** an attempt's deadline never exceeds the session's `expires_at` (unless the merchant allows late payments); slow routes are hidden when the remaining window is too short.
- After `deadline` the address is hidden and replaced by "expired — do not send"; a new attempt can be created while the session is `open`.
- **Payer guards:** wallet-connect is the default; the manual path shows network name + icon prominently, requires confirming the network, makes memos impossible to miss (checkbox "I included the memo"), and asks for a refund address the payer controls (not an exchange deposit address).

## 3. Intents-native payment (payer already holds a NEAR Intents balance)

```mermaid
sequenceDiagram
  autonumber
  actor U as Payer (wallet or agent)
  participant CK as Checkout SPA / agent SDK
  participant AP as agentpay
  participant V as intents.near
  participant IX as rails near-indexer
  participant P as payments
  CK->>AP: payment requirement for cs_/pi_ (payTo = merchant Intents account, fee, binding)
  CK->>U: sign Verifier payload: transfer(merchant), transfer(treasury), memo = binding
  U-->>CK: signed payload (NEP-413 or ERC-191)
  CK->>AP: submit signed payload
  AP->>AP: verify signature, nonce unused, amounts, binding, screening
  AP->>V: simulate_intents
  AP->>V: execute_intents (batched, relayer function-call key)
  V-->>IX: transfer events (finalized block)
  IX-->>P: evidence for pi_ (via rails.execution.settled)
  P-->>CK: succeeded (about 1-2 s)
```

Only valid when the merchant destination is an Intents account in the paid asset (or the merchant enables "accept Intents balance in asset X" — then settlement asset = X for that payment, recorded on the attempt).

## 4. Refund (merchant-authorised)

```mermaid
sequenceDiagram
  autonumber
  actor M as Merchant (dashboard) or merchant server
  participant P as payments
  participant R as rails
  participant OC as 1Click
  participant MW as Merchant wallet / key
  M->>P: POST /v1/refunds (pi_, amount, reason, Idempotency-Key)
  P->>P: lock intent, check refundable, refund requires_action
  P->>R: prepare leg: origin = merchant settlement asset, destination = payer origin asset, recipient = refund_to
  R->>OC: quote (EXACT_INPUT of refund amount, refundTo = merchant destination)
  OC-->>R: depositAddress (on merchant's chain or Intents)
  R-->>P: next_action sign_and_send {chain, to, amount, memo, deadline}
  P-->>M: refund.requires_action
  M->>MW: sign transfer (or Verifier transfer intent if Intents account)
  MW->>OC: deposit lands
  R->>OC: track status, verify payer-side evidence
  R-->>P: rails.execution.settled
  P-->>M: refund.succeeded (webhook + payer email)
```

If the quote deadline passes unsigned, the leg expires and `payments` prepares a new leg on demand; a merchant cannot change `recipient` except via an owner override with step-up auth and fresh screening (INV-7).

**Refund fees:** NearPay charges no fee on refunds (no `appFees`) and does not return the original fee. `fee_mode = payer_absorbs` (default, `EXACT_INPUT`: payer receives the refund amount minus route costs) or `merchant_absorbs` (`EXACT_OUTPUT`: payer receives the equivalent of the refunded amount; merchant sends slightly more). Refunds of Intents-native payments are a merchant-signed Verifier `transfer` back to the payer's Intents account — no route provider involved.

## 5. Invoice and `send_invoice` subscription renewal

```mermaid
sequenceDiagram
  autonumber
  participant S as billing scheduler
  participant B as billing
  participant P as payments
  participant N as notify
  actor U as Customer
  participant CK as Checkout SPA (hosted invoice)
  S->>B: period boundary for sub_ (job key = sub_ + period_start)
  B->>B: create invoice (draft) then finalize
  B->>P: create PaymentIntent (metadata invoice = in_)
  B->>N: invoice email (hosted_invoice_url)
  U->>CK: open invoice, pay (flow 2 or 3)
  P-->>B: evt payments.payment_intent.succeeded
  B->>B: invoice paid, subscription active, next period scheduled
  Note over S,B: unpaid at due date: past_due, dunning reminders T+1, T+3, T+7 (configurable), then cancel or mark unpaid
```

## 6. `auto_debit` subscription via vault

```mermaid
sequenceDiagram
  autonumber
  actor U as Payer (NEAR wallet)
  participant T as Token contract (e.g. USDC on NEAR)
  participant VA as v1.vault.nearpay.near
  participant KP as billing keeper
  participant V as intents.near
  participant IX as rails near-indexer
  participant B as billing
  U->>T: ft_transfer_call(vault, amount, msg = create_mandate{...})
  T->>VA: ft_on_transfer: mandate md_ created, funded
  VA-->>IX: event nearpay_vault.mandate_created
  IX-->>B: mandate active
  KP->>VA: collect(md_) after period boundary (function-call key, gas only)
  VA->>T: ft_transfer_call(intents.near, merchant share, msg = merchant Intents account)
  VA->>T: ft_transfer(treasury, fee)
  VA-->>IX: event nearpay_vault.collected (period n)
  IX-->>B: evidence, invoice paid
  U->>VA: cancel(md_) any time, remainder returned to payer
```

`collect` is idempotent per `(mandate, period)` on-chain; failed transfers roll back the period in the resolve callback.

## 7. Agent payment over x402 (`exact` + `intents-transfer`)

```mermaid
sequenceDiagram
  autonumber
  participant A as Agent (x402 client)
  participant RS as Merchant API + @nearpay/x402
  participant AP as agentpay facilitator
  participant V as intents.near
  A->>RS: GET /resource
  RS-->>A: 402 PAYMENT-REQUIRED (accepts: exact, near:mainnet, asset, amount, payTo, extra.binding)
  A->>A: sign transfer intents (merchant + fee), memo = binding, deadline <= maxTimeoutSeconds
  A->>RS: GET /resource + PAYMENT-SIGNATURE
  RS->>AP: POST /x402/verify
  AP-->>RS: isValid (signature, nonce, simulate, screening, binding)
  RS->>AP: POST /x402/settle
  AP->>V: execute_intents (batch <= 150 ms)
  V-->>AP: final outcome (about 1.2 s)
  AP-->>RS: success, transaction, network near:mainnet
  RS-->>A: 200 + resource + PAYMENT-RESPONSE
```

`settlement_pending` (non-terminal) is returned if finality is not reached within the request budget; the middleware retries `settle` with the same payload (idempotent on payload hash).

## 8. Event and webhook delivery

```mermaid
sequenceDiagram
  autonumber
  participant SVC as Owning service
  participant DB as Service DB
  participant RL as Outbox relay
  participant NT as NATS JetStream
  participant WH as webhooks
  participant EG as Egress proxy
  participant ME as Merchant endpoint
  SVC->>DB: tx: state change + transition row + outbox row
  RL->>DB: poll outbox (SKIP LOCKED)
  RL->>NT: publish evt.<domain>.<aggregate>.<event> (msg id = event id)
  RL->>DB: mark relayed
  NT->>WH: deliver (durable consumer)
  WH->>WH: inbox dedupe, persist public Event, fan out per subscribed endpoint
  WH->>EG: POST signed payload (NearPay-Signature t, v1, v1a)
  EG->>ME: deliver (private IPs blocked, DNS pinned)
  ME-->>WH: 2xx (or retry with backoff up to 72 h)
```
