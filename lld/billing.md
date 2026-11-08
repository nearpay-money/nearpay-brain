# LLD — `nearpay-billing`

> Version 4.1-vault/1.6 (vault re-check 2: `prepare` must refuse relayer/gasless funding in its instructions §5) · Version 4.1-vault/1.5 (vault re-audit: `prepare` signer and gas notes §5) · Version 4.1-vault/1.4 (step 4.1, AI vault audits: mandate columns bigint + linkage columns §3, linkage rules §6, keeper rules §7.1, keeper gas 250 Tgas) · Version 4.1-vault/1.3 (step 4.1: keeper attaches 150 Tgas to vault `collect`, §7 — `contracts-vault.md` v1.3 §3.3) · Version 1.4 (spec 0.3.1: test-mode billing clock §13, 0.3.1 error lists supersede §12 d18) · Version 1.3 (step 3.2, implementation decisions §12; data model §3 as built in `migrations/0001_billing.sql`) · Version 1.2 (step 3.1, Stage 3 contracts in `nearpay-spec` 0.3.0: API §5, events §6, decisions §11) · Version 1.1 (audited, `audits/billing.md`) · HLD §12 · ADR-0008 · Flows §5–§6 · Foundations `00-foundations.md`

## 1. Scope

**Owns:** Product, Price, TaxRate, Invoice (+ lines, numbering, PDF), Subscription (+ items), dunning, Mandate mirror of the vault, the keeper that triggers `collect`.
**Uses:** payments for collection (internal checkout sessions) and customers (`cus_` is owned by payments); rails `evt.rails.vault.event` for vault evidence; notify for emails.

## 2. Runtime

| Compose service | Command | Replicas |
|---|---|---|
| `billing-api` | `api` | 2 |
| `billing-worker` | `worker` | 1 (scheduler jobs, keeper, consumers, outbox relay) |

## 3. Data

```sql
create table products (id uuid primary key, account_id uuid not null, livemode boolean not null, name text not null,
  description text, active boolean not null default true, metadata jsonb not null default '{}',
  created_at timestamptz not null default now(), updated_at timestamptz not null default now(), version integer not null default 0);

create table prices (id uuid primary key, account_id uuid not null, livemode boolean not null, product_id uuid not null references products(id),
  type text not null check (type in ('one_time','recurring')),
  unit_amount bigint, currency text not null, asset_id text, asset_amount numeric(78,0),
  interval text check (interval in ('day','week','month','year')), interval_count smallint check (interval_count between 1 and 12),
  active boolean not null default true, metadata jsonb not null default '{}', created_at timestamptz not null default now(),
  check ((type = 'recurring') = (interval is not null)));

create table tax_rates (id uuid primary key, account_id uuid not null, livemode boolean not null, display_name text not null,
  rate_ppm integer not null check (rate_ppm between 0 and 1000000), inclusive boolean not null,   -- §11 d1 (was percentage_bps)
  jurisdiction text, country char(2), description text, metadata jsonb not null default '{}',
  active boolean not null default true, created_at timestamptz not null default now());

create table invoice_sequences (account_id uuid not null, livemode boolean not null, next_number integer not null default 1,
  prefix text not null, primary key (account_id, livemode));

create table invoices (
  id uuid primary key, account_id uuid not null, livemode boolean not null, customer_id text not null,  -- cus_ from payments
  subscription_id uuid, number text, status text not null check (status in ('draft','open','paid','void','uncollectible')),
  collection_method text not null check (collection_method in ('send_invoice','auto_debit')),
  currency text not null, asset_id text, subtotal bigint not null, tax bigint not null, total bigint not null,  -- fiat minor units
  asset_total numeric(78,0),                                     -- when currency = 'asset'
  due_date timestamptz, period_start timestamptz, period_end timestamptz,
  checkout_session_id text, payment_intent_id text, hosted_invoice_url text, pdf_object_key text,
  invoice_secret_hash bytea,                                     -- hosted URL /i/<in_id>#<invoice_secret>; long-lived, independent of sessions
  invoice_secret_enc bytea, checkout_client_secret_enc bytea,    -- §11 d3 (envelope-encrypted)
  billing_reason text not null default 'manual', default_tax_rate_ids uuid[] not null default '{}',
  description text, footer text, days_until_due smallint, reminder_days smallint[] not null default '{-3,0,1,3,7}',
  uncollectible_reason text, sent_at timestamptz, marked_uncollectible_at timestamptz, payment_intent_ids text[] not null default '{}',
  mandate_id uuid, collect_period integer,                       -- auto_debit
  finalized_at timestamptz, paid_at timestamptz, voided_at timestamptz, reminders_sent smallint not null default 0,
  metadata jsonb not null default '{}', created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0, unique (account_id, livemode, number));
create unique index invoices_sub_period on invoices (subscription_id, period_start) where subscription_id is not null;

create table invoice_lines (id uuid primary key, invoice_id uuid not null references invoices(id), price_id uuid references prices(id),
  description text not null, quantity integer not null check (quantity > 0), unit_amount bigint not null, amount bigint not null,
  tax_rate_ids uuid[] not null default '{}', tax_amount bigint not null default 0, period_start timestamptz, period_end timestamptz,
  tax_amounts jsonb not null default '[]', subscription_item_id uuid, product_id uuid);   -- tax snapshot per rate (§11 d2)

create table subscriptions (
  id uuid primary key, account_id uuid not null, livemode boolean not null, customer_id text not null,
  status text not null check (status in ('incomplete','trialing','active','past_due','canceled','unpaid')),
  collection_method text not null check (collection_method in ('send_invoice','auto_debit')),
  days_until_due smallint not null default 7, current_period_start timestamptz not null, current_period_end timestamptz not null,
  trial_end timestamptz, cancel_at_period_end boolean not null default false, canceled_at timestamptz,
  latest_invoice_id uuid, mandate_id uuid, dunning jsonb not null default '{"reminder_days":[-3,0,1,3,7],"on_exhausted":"cancel"}',
  billing_cycle_anchor timestamptz not null, interval text not null, interval_count smallint not null, currency text not null, asset_id text,
  default_tax_rate_ids uuid[] not null default '{}', cancellation_reason text, ended_at timestamptz, trial_start timestamptz,
  settlement_destination_id text, description text,
  metadata jsonb not null default '{}', created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0);
create table subscription_items (id uuid primary key, subscription_id uuid not null references subscriptions(id),
  price_id uuid not null references prices(id), quantity integer not null default 1, tax_rate_ids uuid[]);

create table mandates (
  id uuid primary key, account_id uuid not null, livemode boolean not null, subscription_id uuid references subscriptions(id),
  vault_contract text not null, onchain_id bigint, payer text not null, token text not null,
  amount_per_period numeric(78,0) not null, period_secs bigint not null, start_at timestamptz not null, max_periods bigint,
  collected_periods bigint not null default 0, balance numeric(78,0) not null default 0,
  merchant_receiver text, receiver_kind text, fee_bps integer, ext_ref text,   -- v1.4: from mandate_created, for linkage (§6)
  link_status text not null default 'pending' check (link_status in ('pending','linked','unlinked')), unlinked_reason text,
  collect_failures integer not null default 0,                                    -- consecutive collect_failed (§7.1)
  status text not null check (status in ('pending_funding','active','insufficient','exhausted','canceled')),
  last_event_height bigint, created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0, unique (vault_contract, onchain_id));
-- transitions tables for invoices, subscriptions, mandates (foundations §7)
```
Invoice numbers: `<prefix>-<5-digit>` where prefix = first 6 chars of account display slug + mode letter; allocated with `update invoice_sequences … returning` at finalize.

**As built (v1.3, `nearpay-billing/migrations/0001_billing.sql`; supersedes the sketch above where they differ, §12 d1):** every amount column is `numeric(78,0)` in units of the row's currency (fiat minor units or asset base units), so `prices.unit_amount` replaces `unit_amount bigint` + `asset_amount`, and `invoices.asset_total` is dropped; `customer_id` is the payments customer's uuid; invoices add `total_excluding_tax`, `total_tax_amounts jsonb`, `amount_paid`, `settlement_destination_id`, `overdue_at`, `bill_to_enc` (encrypted `{name, email}` at finalize), `merchant_name` (snapshot), `checkout_session_expires_at`, `checkout_sessions_created` (the `n` of `in:<id>:s<n>`), `uncollectible_reason`; lines add `position`, `type`, `amount_excluding_tax`, `tax_rates_explicit`; subscriptions keep `reminder_days smallint[]` + `on_exhausted` (instead of `dunning jsonb`) and `next_period_index` (the `n` the next renewal invoices); items add `position`; `invoice_transitions` has no FK (a deleted draft keeps its history); a trigger refuses deleting non-draft invoices; `rate_counters` backs the hosted-page limits. `collection_method` admits only `send_invoice`; `mandates`, `mandate_id` and `collect_period` arrive with the Stage 4 migration.

## 4. State machines

`domain-model.md` §4.5 plus guards: finalize requires ≥ 1 line, `customer_id` exists (payments internal), total > 0 and ≥ destination minimum; `void` only from `open` and only when no session can still settle (§11 d10); `uncollectible → paid` allowed (late payment, INV-3); `paid`/`void` are final. Subscriptions: `incomplete → active` (first invoice paid) · `incomplete → canceled` (first invoice exhausted) · `trialing → active` (trial end, first invoice issued) · `active → past_due` (renewal unpaid at due date) · `past_due → active` (paid) · `past_due → canceled | unpaid` (dunning exhausted, `on_exhausted`) · `unpaid → active` (paid) · any non-canceled `→ canceled` (cancel now or at period end).

## 5. API

Exact contracts: `nearpay-spec` `public/billing-catalog.ts`, `public/invoices.ts`, `public/subscriptions.ts`, `internal/billing.ts` (§11).

| Method & path | Scope · roles | Notes |
|---|---|---|
| `POST/GET /v1/products…`, `/v1/prices…`, `/v1/tax_rates…`, `PATCH …/{id}` | `products.*` · owner/admin/developer/finance | create/get/list/update; no delete (archive with `active: false`); prices immutable except `active`, `nickname`, `metadata`; tax rates immutable except labels, `active`, `metadata` |
| `POST /v1/invoices` · `PATCH /v1/invoices/{id}` · `DELETE /v1/invoices/{id}` | `invoices.write` · owner/admin/finance | draft with `lines[{price, quantity?} \| {unit_amount, description, quantity?}]` (+ `tax_rates?`), `customer`, `collection_method='send_invoice'`, `days_until_due` (default 30) or `due_date`, `default_tax_rates`, `reminder_days` (default `[-3,0,1,3,7]`); update = drafts only except `metadata`; delete = drafts only |
| `POST /v1/invoices/{id}/finalize` · `/send` · `/void` · `/mark_uncollectible` | `invoices.write` · owner/admin/finance | finalize assigns number, due date, invoice secret, hosted URL, PDF (no payments session yet, §11 d3); send emits `invoice.sent` (notify mails `invoice.issued`); void expires the session first (`attempt_in_flight` while money may move) |
| `GET /v1/invoices…` · `GET /v1/invoices/{id}/pdf` | `invoices.read` | PDF `application/pdf`, drafts `409` |
| `GET /v1/invoices/{id}/public` · `POST …/public/sessions` · `GET …/public/pdf` | anonymous + `X-NearPay-Invoice-Secret` | hosted invoice `/i/:id`: read-only `PublicInvoice`; create-or-continue the invoice's payments session (`InvoicePaymentSession {checkout_session, client_secret, expires_at, reused}`); PDF |
| `POST /v1/subscriptions` | `subscriptions.write` · owner/admin/finance | `{customer, items[{price, quantity?, tax_rates?}], collection_method?, days_until_due? (7), trial_period_days?, default_tax_rates?, dunning?{reminder_days, on_exhausted}, description?, settlement_destination?, metadata?}` (`mandate` arrives in Stage 4) |
| `PATCH /v1/subscriptions/{id}` · `POST …/cancel` | `subscriptions.write` · owner/admin/finance | update applies from the next renewal, `proration_behavior` only `none`; cancel `{at_period_end?}` |
| `POST /v1/mandates/prepare` | `subscriptions.write` | Stage 4: returns `{receiver_id: vault, method:'ft_transfer_call', args:{amount, msg}, gas: '100 Tgas', deposit: '1 yoctoNEAR'}` for the payer to sign **from their own NEAR account, with their own gas** (the vault refunds deposits whose sender is not the transaction signer — `contracts-vault.md` v1.6 §3.1). The response and the checkout flow must say so: no relayer/gasless wallets (NEP-366 meta-transaction deposits are refused, because their signer is the relayer), no Intents balance, bridge or smart wallet; such deposits are refunded. Cancel/withdraw may go through a meta-transaction. Cancel/withdraw prepared for the payer attach 100 Tgas (`MIN_EXIT_GAS` 50 Tgas, §3.4) |
| `GET /v1/mandates…` | `subscriptions.read` | Stage 4 |
| `GET /internal/invoices/{id}/notification-context` · `GET /internal/subscriptions/{id}/notification-context` | notify (`billing.read`) | customer email/name (via payments), merchant name, amounts, hosted link (with secret), reminder number |

## 6. Events

Produced: `evt.billing.invoice.{created,finalized,sent,paid,payment_failed,overdue,voided,marked_uncollectible}` (`InvoiceEvent`, `hosted_invoice_url` null), `evt.billing.invoice.reminder` (`InvoiceReminderEvent`, notify only), `evt.billing.subscription.{created,updated,deleted,renewal_upcoming}`, Stage 4: `evt.billing.mandate.{activated,collected,insufficient_funds,canceled,exhausted}`. Public webhook types: `invoice.<same>` and `customer.subscription.<same>` (not `invoice.reminder`).
Consumed: `evt.payments.payment_intent.succeeded` (`invoice` set → invoice paid), `evt.payments.payment_intent.payment_failed` (→ `invoice.payment_failed`; invoice stays open), `evt.payments.checkout_session.expired` and `evt.payments.payment_intent.expired` (clear `checkout_session_id`; a fresh session is created lazily on the next `POST …/public/sessions`), Stage 4: `evt.rails.vault.event` (mandate created/collected/canceled/withdrawn). On `mandate_created` billing links the mandate to a subscription only if **all** hold (v1.4, AI vault audits A1-L-2 / A2-M-6): `ext_ref` = the subscription ID; `receiver_kind = "intents_account"`; `merchant_receiver` = an active Intents-account destination of that account (the one `POST /v1/mandates/prepare` put in the `msg`); `token`, `amount_per_period` and `period_secs` match the subscription's price; `fee_bps` = the account's agreed NearPay fee (ADR-0006) — a payer-edited `fee_bps` (e.g. 0) bypasses the fee; `start_at` ≤ the subscription's `current_period_end`; `max_periods` is null or ≥ the subscription's remaining periods (null when it has no end). Otherwise the mandate is recorded with `link_status = 'unlinked'` and `unlinked_reason`, and the payer is told to cancel (the contract cannot refuse payer-signed terms). Vault events reach billing only through rails' forwarding rules (`rails.md` §8.6: vault account, successful receipts); billing dedupes on `receipt_id:log_index` and never blocks the consumer on one event — an event that fails validation or storage is parked (`vault_events_parked`, alert) and the stream continues. Mandate numbers are stored as bigint/numeric (§3), so every value the vault accepts fits (`contracts-vault.md` v1.4 §3.1 bounds them anyway).

## 7. Jobs

| Kind (dedupe key) | When | Behaviour |
|---|---|---|
| `subscription.renew` (`sub:<id>:<period_start>`) | `current_period_end` | create invoice for next period (draft → finalize); `send_invoice` → payments session + email; `auto_debit` → `mandate.collect`; advance period |
| `mandate.collect` (`md:<id>:<period>`) | after renew | keeper calls vault `collect(mandate_id)` (function-call key, **250 Tgas attached**, unused gas refunded — the vault refuses less than 170 Tgas with `ERR_GAS` and reserves static gas for the token's `ft_transfer_call` and its callbacks, `contracts-vault.md` v1.4 §3.3; see §7.1 for outcomes); success waits for `vault.event collected` (evidence) → invoice paid; tx failure with `ERR_INSUFFICIENT_BALANCE` → invoice switches to `send_invoice` path + `mandate.insufficient_funds`; `ERR_NOT_DUE`/`ERR_IN_FLIGHT` → retry at `next_collect_at`; other errors → retry ×5 then case |
| `invoice.remind` (`in:<id>:r<n>`) | per the invoice's `reminder_days` relative to `due_date` | publish `evt.billing.invoice.reminder` (notify mails); after the last → dunning exhausted (§11 d8) |
| `invoice.overdue` (`in:<id>:overdue`) | `due_date` | unpaid → `invoice.overdue`; subscription invoice → subscription `past_due` |
| `subscription.renewal_upcoming` | 3 days before period end | event + email |

### 7.1 Keeper outcomes (Stage 4, v1.4)

A `collect` transaction ends in exactly one of: a failure with a vault error string; or success with `collected` (optionally followed by `exhausted`) or `collect_failed` in its vault receipts (forwarded by rails). The job acts on the forwarded event, not on the RPC response alone (INV-2).

| Outcome | Action |
|---|---|
| `collected`, `merchant_share` = expected share (`floor(amount × (10000 − fee_bps) / 10000)`) | invoice paid (evidence = the event); `mandate.collected`; reset `collect_failures` |
| `collected` with a **short** `merchant_share` (partial delivery, `contracts-vault.md` §3.2) | do not mark paid: record the amount received, open a case, send the shortfall as a `send_invoice` balance invoice; the period is counted on-chain |
| `collected` with `fee = "0"` while `fee_bps > 0` | invoice paid; alert ops (treasury not registered on the token, or blocklisted) |
| `collect_failed` | `collect_failures += 1`; retry on the next keeper tick (≥ 1 block later, jittered, at most once per minute); after **N = 5** consecutive failures (or 24 h) the invoice falls back to `send_invoice`, `mandate.insufficient_funds` is not emitted, a case is opened for the merchant destination |
| tx failure `ERR_INSUFFICIENT_BALANCE` | invoice → `send_invoice` path + `mandate.insufficient_funds` (unchanged) |
| tx failure `ERR_NOT_DUE` | retry at `next_collect_at` (view) |
| tx failure `ERR_IN_FLIGHT` | another collect is pending (possibly someone else's — `collect` is permissionless): wait for its forwarded outcome; re-check after 30 s, never in a tight loop |
| tx failure `ERR_GAS` | keeper misconfiguration: alert, fix gas, retry |
| tx failure `ERR_NOT_ACTIVE` / `ERR_MAX_PERIODS` / `ERR_NOT_FOUND` | reconcile the mirror from the view / events; no retry |
| other errors | retry ×5 then case (unchanged) |

## 8. Configuration

`NP_VAULT_CONTRACT` (`v1.vault.nearpay.near`), `NP_KEEPER_ACCOUNT`, `NP_KEEPER_KEY_FILE`, `NP_NEAR_RPC_URLS`, `NP_OBJECT_STORE` (`fs:/data/invoices` local · `s3://…` server), `NP_PAY_ORIGIN`.

## 9. Tests

Proration is out of scope (no mid-period changes in v1: updates apply next period) — tested as rejected. Renewal idempotency across restarts; dunning schedule with fake clock; tax inclusive/exclusive rounding (per line, round half-up in minor units); auto-debit with sandbox vault simulation; late payment on `uncollectible`.

## 10. Stage mapping

Stage 3: products, prices, tax rates, invoices, hosted invoice, `send_invoice` subscriptions, dunning (contracts §11). Stage 4: mandates + keeper + auto-debit.

## 11. Stage 3 contracts (step 3.1, `nearpay-spec` 0.3.0)

Spec modules: public `billing-catalog.ts` (products, prices, tax rates, tax and period helpers), `invoices.ts` (invoices + hosted payer routes), `subscriptions.ts`; internal `billing.ts` (notification contexts, `BILLING_EMAIL_TEMPLATES`), invoice sessions and customer read in `internal/payments.ts`; events in `events/catalogue.ts`. Decisions made by the contract (not in earlier versions of this LLD):

1. **Tax rates are exact decimals, not bps.** `percentage` is a decimal string 0–100 with ≤ 4 decimals (`"8.875"`), mirrored as `rate_ppm` (= percentage × 10 000, the rational `ppm / 10⁶`); the column becomes `rate_ppm integer check (rate_ppm between 0 and 1000000)` instead of `percentage_bps` (bps cannot express 8.875 %). Tax rates gain `description`, `country`, `metadata`; `percentage` and `inclusive` are immutable.
2. **Normative tax rule** (`computeLineTax` / `computeInvoiceTotals` in `@nearpay/spec`, vectors `test-vectors/billing-tax.json`): line `amount = unit_amount × quantity` (integer units of the invoice currency: fiat minor units or asset base units); all rates of one line are inclusive or all exclusive (`tax_rates_mixed`); exclusive `tax_i = round_half_up(amount × ppm_i / 10⁶)`; inclusive `tax_i = round_half_up(amount × ppm_i / (10⁶ + Σppm))`, each rate rounded on its own; `subtotal = Σ amount`, `tax = Σ tax_i`, `total = subtotal + Σ exclusive tax`, `total_excluding_tax = total − tax`; `total_tax_amounts` sums the already rounded line taxes per rate. Round half-up = ⌊n/d + ½⌋ (all values ≥ 0). Max 5 rates per line. Invoice `default_tax_rates` apply to lines created without `tax_rates` (`[]` = untaxed). No discounts in v1.
3. **Finalize does not open the payments session.** Finalize assigns number, due date, invoice secret, hosted URL and PDF; the hosted page opens the session with `POST /v1/invoices/{id}/public/sessions` (create, or return the open one with `reused: true`). `GET …/public` stays read-only (no side effects on a GET). Billing stores the session's client secret envelope-encrypted (`checkout_client_secret_enc bytea`) to hand it back for a reused session, and checks `getInternalCheckoutSession` is still `open` first. The invoice secret is stored as hash **and** envelope-encrypted (`invoice_secret_enc bytea`) so `hosted_invoice_url` can be returned on every merchant read and in the notification context.
4. **Internal invoice session contract (payments).** `POST /internal/checkout_sessions` `{account_id, livemode, invoice, amount (= amount_remaining), currency, asset_id?, customer, description, settlement_destination?, success_url, expires_at (5 min–30 d), locale?, metadata?}` → `CheckoutSession` with `client_secret`/`url`; `Idempotency-Key` `in:<invoice>:s<n>`; at most one `open` session per invoice (`409 state_conflict`). `POST /internal/checkout_sessions/{id}/expire {reason}` (void), `GET /internal/checkout_sessions/{id}`, `GET /internal/customers/{id}` (`InternalCustomer` with `deleted`). `PaymentIntent.invoice` / `CheckoutSession.invoice` carry the link; billing never reads payments' tables.
5. **Billing emails are event-driven** (notify §4): `invoice.sent` → `invoice.issued`, `invoice.reminder` → `invoice.reminder`, `invoice.paid` → `invoice.paid`, `invoice.voided` → `invoice.voided`, `subscription.renewal_upcoming` → `subscription.renewal_upcoming`; notify reads `getInvoiceNotificationContext` (`?reminder=n`) / `getSubscriptionNotificationContext`. The hosted link embeds the invoice secret, so it is served only by the context route, never put in events (`InvoiceEvent.hosted_invoice_url` is `null`). Billing does not call `POST /internal/messages`. `send` is rate-limited to one mail per invoice per 10 minutes.
6. **No personal data on invoices.** `Invoice` carries `customer` (`cus_`) only; name/email are read from payments when rendering the PDF, the hosted page (`bill_to`, taken at finalize) and mails. Billing's service client gains `identity.context.read` (merchant name, destinations) and `rails.read` (asset decimals/symbol).
7. **Lines and drafts.** Custom-amount lines use `unit_amount` (not `amount`) so `amount = unit_amount × quantity` is unambiguous; ≤ 100 lines; price lines need active `one_time` prices in the invoice currency (`price_type_invalid`, `currency_mismatch`, `price_inactive`); `currency` defaults to the first price line's. Drafts are editable (`PATCH`, `lines` replaces all) and deletable; issued invoices change only `metadata`. Line ids `il_`, item ids `si_`, tax-rate ids `txr_`.
8. **Dunning.** Every invoice has `reminder_days` (strictly increasing, −30…+60, ≤ 10; subscription invoices copy `subscription.dunning.reminder_days`; manual default `[-3,0,1,3,7]`; offsets already past at finalize are skipped). At `due_date` an unpaid invoice emits `invoice.overdue` (subscription → `past_due`). When the last reminder has gone out and the invoice is unpaid: subscription invoices → `uncollectible` (`uncollectible_reason: dunning_exhausted`) and the subscription → `canceled` (`dunning_exhausted`) or `unpaid` per `on_exhausted` (a first invoice of an `incomplete` subscription → `canceled`, `incomplete_expired`); manual invoices stay `open` (the merchant may mark them uncollectible). `unpaid` subscriptions create no invoices; paying the latest invoice returns them to `active`, and the next renewal runs at `current_period_end` (immediately if past).
9. **Subscriptions.** Periods follow `periodBoundary(anchor, interval, count, n)` (UTC; `day`/`week` add seconds; `month`/`year` add calendar months from the anchor, clamped to the month's last day, never chained); the anchor is creation time or `trial_end`. Invoiced in advance at each period start, due after `days_until_due` (default 7). All items share currency and `recurring` (`interval_mismatch`). Updates use `PATCH` (spec convention since Stage 2) and apply from the next renewal; `proration_behavior` accepts only `none`. Immediate cancel leaves open invoices as they are. Trials: `trialing` until `trial_end`, then the first invoice and `active`.
10. **Void and late money.** Void is refused with `attempt_in_flight` while any session of the invoice could still settle. Settlements that arrive for an already `paid` or `void` invoice (INV-3) are listed in `Invoice.payment_intents` and refunded by the merchant like overpayments; the invoice status does not change. No "paid out of band" (INV-2).
11. **PDF.** `GET /v1/invoices/{id}/pdf` (merchant) and `GET /v1/invoices/{id}/public/pdf` (invoice secret) serve `application/pdf` (`binaryResponse` in the route table), rendered at finalize and re-rendered with a status stamp on each later status change.
12. **New errors** (0.3.0): `currency_mismatch`, `interval_mismatch`, `price_inactive`, `price_type_invalid`, `tax_rate_inactive`, `tax_rates_mixed`, `invoice_empty`, `customer_email_missing`, `invoice_secret_invalid`, `invoice_not_payable`.
13. **Webhook types** `invoice.{created,finalized,sent,paid,payment_failed,overdue,voided,marked_uncollectible}` and `customer.subscription.{created,updated,deleted,renewal_upcoming}` (`invoice.sent` added to the api-conventions §9 catalogue); no product/price webhooks.

## 12. Implementation decisions (step 3.2)

Made while building `nearpay-billing` against spec 0.3.0; not decided by §1–§11.

1. **Data model** as noted under §3 (one `numeric(78,0)` amount column for fiat and asset pricing; dunning settings as columns; Stage 4 columns deferred to its own forward-only migration).
2. **Reminder numbers are positional.** Reminder `n` is the 1-based position in `reminder_days` (job key `invoice.remind:<id>:r<n>`, event `number`, `?reminder=n`); offsets already past at finalize are skipped without renumbering. A reminder is sent only if it is the next unsent one and due (repeated or out-of-order jobs do nothing).
3. **Dunning exhaustion** is its own job (`invoice.dunning_exhausted`) at `max(due_date, last remaining reminder)` — never before the due date, `[]` exhausts at the due date. The last reminder still goes out if exhaustion ran first in the same instant (invoice `uncollectible` with `dunning_exhausted`). An `active` subscription passes through `past_due` before `canceled`/`unpaid`.
4. **No billing in arrears.** A renewal invoices the period containing "now" (`n` = first period whose end is after now, from `next_period_index`): after a pause (`incomplete` and `unpaid` create no invoices) or a late job, periods that ended uninvoiced are skipped. Paying the invoice that ends a pause enqueues a renewal at `max(now, current_period_end)` with its own dedupe key.
5. **`past_due → active`** when the paid invoice leaves no other open overdue invoice of the subscription; `unpaid → active` only for the latest invoice (§11 d8); `incomplete → active` for the first invoice.
6. **Pricing at renewal.** Manual finalize and subscription creation need active prices and tax rates (`price_inactive`, `tax_rate_inactive`; an archived product makes its prices inactive for new use). Renewals keep billing items whose price or tax rate was deactivated later (current unit amount, current rate). Invoices are repriced from current rates at finalize, then frozen.
7. **Finalize guards.** Live mode enabled when live (`account_not_live`), the invoice's destination or the asset/account default active (`destination_not_active`), currency supported (`currency_unsupported`), customer present and not deleted. The destination **minimum** is enforced by payments when the hosted page opens the session (`amount_below_destination_minimum` from `createInvoiceCheckoutSession`): billing has no read path for minimums in pricing-currency terms.
8. **Hosted sessions.** `expires_at = max(now + 1 h, min(now + 30 d, last reminder))` (uncollectible invoices stay payable after their last reminder). Session metadata = the invoice's metadata (≤ 49 keys kept) + `invoice_number`. A stored session is reused only when payments reports it `open` and unexpired; otherwise a new one is created with the next `in:<id>:s<n>` key. A `409` from payments returns the session a concurrent request stored. A session that opened while the invoice was paid/voided is expired again (`superseded`) and the request answers `invoice_not_payable`. Limits: 10/min per client network and 30/min per invoice (configurable), counted in Postgres.
9. **Payment evidence (INV-2).** On `payment_intent.succeeded` billing re-checks the intent: `invoice` link, account and mode, `status = succeeded`, pricing equal to the invoice's, amount ≥ `amount_remaining`. Anything else is recorded in `payment_intents` and alerted (`ALERT` log), never turned into `paid`. Settlements for `paid`/`void` invoices are listed only (§11 d10).
10. **Ordering of payments' events** is handled by the rules (paid is final; `payment_failed` never changes status; an expired session is forgotten only if it is still the current one), on top of inbox dedupe.
11. **Event versions.** Each change bumps the aggregate `version` (= `sequence`) once, plus once per extra event in the same transaction, so every event carries its own increasing version; changes without events (session stored, PDF key) also bump it.
12. **Renewal without customer email.** Subscription creation requires an email (`customer_email_missing`); a later renewal for a customer who lost it is finalized but not sent (warning log), so dunning still runs.
13. **PDF storage.** One object per status (`invoices/<account>/<mode>/<in_…>/<status>.pdf`) rendered by job `invoice.render_pdf` after finalize and each status change; a missing object is rendered on demand (deterministic from the frozen invoice).
14. **Numbering.** Prefix = first 6 `[A-Z0-9]` of the NFKD-upper-cased display name (`INV` if none) + `L`/`T`, fixed by the first allocation of the account and mode.
15. **Metadata updates replace the whole map** (as in payments).
16. **Consumed subjects** follow the spec catalogue: `evt.payments.checkout_session.expired` (not `payment_intent.expired`, §6) clears the stored session.
17. **Job time.** Every job payload carries its scheduled time `at`; reminder, overdue, exhaustion and renewal handlers decide at `max(now, at)`, so a job the database starts a moment before the service clock reaches it (clock skew) still acts; a renewal that is genuinely not due fails and is retried by the job runner, never dropped.
18. **Pre-0.3.1 error mappings** (superseded by spec 0.3.1, which lists the codes: answer them as-is once billing builds on 0.3.1) (lead decision 2026-11-08): `createSubscription` answers a zero-total first invoice with `invalid_request` (param `items`) until `invoice_empty` is listed; `createPublicInvoiceSession` relays payments' `amount_below_destination_minimum`, `amount_above_unverified_cap` and `customer_deleted` as `invoice_not_payable` (the reason in `status`) and a concurrent-open `state_conflict` as `rate_limited` (retry after 1 s) until 0.3.1 lists them.
19. **Consumed subjects (0.3.0 catalogue):** `evt.payments.payment_intent.expired` is routed to billing too; it forgets the invoice's session when the intent's `checkout_session` is the stored one (same rule as `checkout_session.expired`). Consumers take the scope from the envelope; an intent whose own `livemode` disagrees with the envelope can never pay.
20. **Object store:** v1 ships `fs:/<dir>` only (a volume on the single host, ADR-0015); `s3://` is refused at boot until after v1. `NP_API_ORIGIN` (new) builds `Invoice.invoice_pdf`.
21. **Merchant name** for numbering, PDF, hosted page and mails: identity's `verified_legal_name` once ops approved it (ADR-0020 d3), else `display_name`.
22. **Renewal sweep** (recurring every 5 min): re-enqueues the renewal of any `active`/`past_due`/`trialing` subscription whose period ended more than 10 minutes ago (idempotent per period; guards against a lost job). Kit retention purge hourly, plus `rate_counters` older than a day.

## 13. Test-mode billing clock (spec 0.3.1)

`POST /v1/test_helpers/subscriptions/{id}/advance_clock` (`advanceSubscriptionClock`): test mode only (`livemode_forbidden`), `secret_key` · `restricted_key:subscriptions.write` · session (owner/admin/developer), `Idempotency-Key`. Body `{target: renewal_upcoming | renewal | overdue}` → `SubscriptionClockAdvance {subscription, target, clock_time, advanced_by}`. It lets merchants, `nearpay` CLI and the testbed E2E exercise renewals, renewal notices, reminders, overdue and dunning exhaustion without waiting real days.

1. **Per-subscription clock.** `subscriptions.clock_offset_s bigint not null default 0` with `check (livemode = false or clock_offset_s = 0)`; clock time = `now() + clock_offset_s`. Only test subscriptions ever get a non-zero offset, so live behaviour is unchanged by construction. Invoices of the subscription use the subscription's offset (read through `subscription_id`; manual invoices have none).
2. **What runs on clock time:** the subscription's `subscription.renew`, `subscription.renewal_upcoming`, and its invoices' `invoice.remind`, `invoice.overdue`, `invoice.dunning_exhausted` jobs (handlers decide at `max(clock now, at)`, §12 d17), and dates computed for its invoices (`due_date`, period dates, reminder offsets). Not on clock time: event `occurred_at`/`created`, payments sessions (`expires_at` from §12 d8 is computed in real time: `max(now + 1 h, …)` with the last reminder translated back by the offset), rate counters, retention, PDF rendering.
3. **Targets.** `renewal_upcoming` = `current_period_end − 3 d`; `renewal` = `current_period_end` (`trial_end` while `trialing`); both need `trialing`/`active`/`past_due`. `overdue` = the earliest `due_date` after the clock among the subscription's `open` invoices. A target not strictly ahead of the clock, or a missing one (wrong status, no such invoice), is `409 state_conflict`. `cancel_at_period_end` subscriptions may advance to `renewal` (they end, `customer.subscription.deleted`).
4. **Advance.** In one transaction (subscription row locked `FOR UPDATE`): `delta = target − clock now`; `clock_offset_s += delta`; every pending kit job of the subscription and its invoices with `run_at > now()` gets `run_at = greatest(now(), run_at − delta)`, so jobs that are due at the new clock time run immediately in their normal order (reminder before exhaustion at equal times, §12 d3). Then a `subscription.renewal_upcoming`/`subscription.renew` job is enqueued for the target if none is pending (same dedupe keys as the scheduler, so the sweep and the helper never double-invoice). Response: `clock_time` = new clock now, `advanced_by = delta`. The resulting events and emails follow asynchronously through the outbox.
5. **Safety.** Forward-only (never decreases `clock_offset_s`). One target per call (the clock never jumps further than the next boundary, so repeat `renewal` to step through periods). It doesn't touch other subscriptions, manual invoices or payments state, and it isn't an ops route. Logged as `billing.test_clock.advanced {subscription, target, delta}` (info; no ops audit record). Idempotency-Key replays return the stored response.
6. **Tests.** Advance to each target from `trialing`, `active`, `past_due`; `state_conflict` cases; live key → `livemode_forbidden`; renewal via helper and via scheduler/sweep produce exactly one invoice per period; a reminder schedule `[-3,0,1]` fires in order across two `overdue`/`renewal` advances.

