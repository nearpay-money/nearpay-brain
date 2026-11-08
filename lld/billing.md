# LLD — `nearpay-billing`

> Version 1.2 (step 3.1, Stage 3 contracts in `nearpay-spec` 0.3.0: API §5, events §6, decisions §11) · Version 1.1 (audited, `audits/billing.md`) · HLD §12 · ADR-0008 · Flows §5–§6 · Foundations `00-foundations.md`

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
  amount_per_period numeric(78,0) not null, period_secs integer not null, start_at timestamptz not null, max_periods integer,
  collected_periods integer not null default 0, balance numeric(78,0) not null default 0,
  status text not null check (status in ('pending_funding','active','insufficient','exhausted','canceled')),
  last_event_height bigint, created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0, unique (vault_contract, onchain_id));
-- transitions tables for invoices, subscriptions, mandates (foundations §7)
```
Invoice numbers: `<prefix>-<5-digit>` where prefix = first 6 chars of account display slug + mode letter; allocated with `update invoice_sequences … returning` at finalize.

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
| `POST /v1/mandates/prepare` | `subscriptions.write` | Stage 4: returns `{receiver_id: vault, method:'ft_transfer_call', args:{amount, msg}}` for the payer to sign |
| `GET /v1/mandates…` | `subscriptions.read` | Stage 4 |
| `GET /internal/invoices/{id}/notification-context` · `GET /internal/subscriptions/{id}/notification-context` | notify (`billing.read`) | customer email/name (via payments), merchant name, amounts, hosted link (with secret), reminder number |

## 6. Events

Produced: `evt.billing.invoice.{created,finalized,sent,paid,payment_failed,overdue,voided,marked_uncollectible}` (`InvoiceEvent`, `hosted_invoice_url` null), `evt.billing.invoice.reminder` (`InvoiceReminderEvent`, notify only), `evt.billing.subscription.{created,updated,deleted,renewal_upcoming}`, Stage 4: `evt.billing.mandate.{activated,collected,insufficient_funds,canceled,exhausted}`. Public webhook types: `invoice.<same>` and `customer.subscription.<same>` (not `invoice.reminder`).
Consumed: `evt.payments.payment_intent.succeeded` (`invoice` set → invoice paid), `evt.payments.payment_intent.payment_failed` (→ `invoice.payment_failed`; invoice stays open), `evt.payments.checkout_session.expired` and `evt.payments.payment_intent.expired` (clear `checkout_session_id`; a fresh session is created lazily on the next `POST …/public/sessions`), Stage 4: `evt.rails.vault.event` (mandate created/collected/canceled/withdrawn). On `mandate_created` billing links the mandate to a subscription only if `ext_ref` = the subscription ID, `merchant_receiver` = an active Intents-account destination of that account, token/amount/period match the subscription's price — otherwise the mandate is recorded as `unlinked` and the payer is told to cancel.

## 7. Jobs

| Kind (dedupe key) | When | Behaviour |
|---|---|---|
| `subscription.renew` (`sub:<id>:<period_start>`) | `current_period_end` | create invoice for next period (draft → finalize); `send_invoice` → payments session + email; `auto_debit` → `mandate.collect`; advance period |
| `mandate.collect` (`md:<id>:<period>`) | after renew | keeper calls vault `collect(mandate_id)` (function-call key, 60 Tgas); success waits for `vault.event collected` (evidence) → invoice paid; tx failure with `ERR_INSUFFICIENT_BALANCE` → invoice switches to `send_invoice` path + `mandate.insufficient_funds`; `ERR_NOT_DUE`/`ERR_IN_FLIGHT` → retry at `next_collect_at`; other errors → retry ×5 then case |
| `invoice.remind` (`in:<id>:r<n>`) | per the invoice's `reminder_days` relative to `due_date` | publish `evt.billing.invoice.reminder` (notify mails); after the last → dunning exhausted (§11 d8) |
| `invoice.overdue` (`in:<id>:overdue`) | `due_date` | unpaid → `invoice.overdue`; subscription invoice → subscription `past_due` |
| `subscription.renewal_upcoming` | 3 days before period end | event + email |

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
