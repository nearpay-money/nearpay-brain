# Glossary

| Term | Meaning in NearPay |
|---|---|
| **Asset ID** | Canonical asset identifier = NEAR Intents ID, e.g. `nep141:eth-0xa0b8…omft.near`. Display metadata comes from the token registry in `rails`. |
| **Amount tag** | Unique sub-cent offset added to a quoted output so a destination transfer can be attributed to exactly one execution without the provider. |
| **Attempt** (`pa_`) | One route-specific try to pay a PaymentIntent (one quote, one deposit address). An intent can have many attempts. |
| **Attestation** | A signature by the settlement destination's key over a NearPay-defined message proving the merchant controls it. |
| **Auto-debit / Mandate** (`md_`) | Payer-funded, payer-cancellable subscription balance held by the immutable vault contract; anyone may trigger `collect` after each period. |
| **CheckoutSession** (`cs_`) | A payer-facing, expiring wrapper around a PaymentIntent with UI options and redirect URLs. |
| **Deadline** | 1Click quote deadline after which un-swapped deposits are refunded to the payer. |
| **Evidence** | Verified on-chain proof of settlement: destination tx/receipt, or Verifier event for Intents-account recipients. |
| **Execution** (`ex_`) | `rails`' provider-specific record of a route run (quote, deposit address, provider statuses, tx hashes). 1:1 with an attempt or refund leg. |
| **Facilitator** | x402 role that verifies and settles payment payloads for resource servers; NearPay's lives in `agentpay`. |
| **Intents account** | A balance holder inside `intents.near`, identified by a NEAR AccountId (incl. `0x…` EVM-derived). |
| **Livemode** | Boolean on every object; `false` routes to the sandbox rail. |
| **Outbox** | Table written in the same DB transaction as a state change; relayed to NATS. Guarantees no lost events. |
| **Route provider** | Adapter that can quote and execute any-to-any value movement: `oneclick`, `sandbox`, later `direct-intents`. |
| **Settlement destination** (`sd_`) | Where a merchant receives funds: `{asset_id, type: chain_address \| intents_account, chain, address}`. |
| **Tier A / Tier B chain** | Tier A: NearPay independently verifies settlement on-chain. Tier B: provider-attested only; lower per-payment limits. |
| **Treasury** | `treasury.nearpay.near` Intents account (multisig-controlled) that receives `appFees`. |
