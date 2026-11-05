# LLD — `nearpay-billing`

> Version 1.1 (audited, `audits/billing.md`) · HLD §12 · ADR-0008 · Flows §5–§6 · Foundations `00-foundations.md`

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
  percentage_bps integer not null check (percentage_bps between 0 and 10000), inclusive boolean not null,
  jurisdiction text, active boolean not null default true, created_at timestamptz not null default now());

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
  mandate_id uuid, collect_period integer,                       -- auto_debit
  finalized_at timestamptz, paid_at timestamptz, voided_at timestamptz, reminders_sent smallint not null default 0,
  metadata jsonb not null default '{}', created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0, unique (account_id, livemode, number));
create unique index invoices_sub_period on invoices (subscription_id, period_start) where subscription_id is not null;

create table invoice_lines (id uuid primary key, invoice_id uuid not null references invoices(id), price_id uuid references prices(id),
  description text not null, quantity integer not null check (quantity > 0), unit_amount bigint not null, amount bigint not null,
  tax_rate_ids uuid[] not null default '{}', tax_amount bigint not null default 0, period_start timestamptz, period_end timestamptz);

create table subscriptions (
  id uuid primary key, account_id uuid not null, livemode boolean not null, customer_id text not null,
  status text not null check (status in ('incomplete','trialing','active','past_due','canceled','unpaid')),
  collection_method text not null check (collection_method in ('send_invoice','auto_debit')),
  days_until_due smallint not null default 7, current_period_start timestamptz not null, current_period_end timestamptz not null,
  trial_end timestamptz, cancel_at_period_end boolean not null default false, canceled_at timestamptz,
  latest_invoice_id uuid, mandate_id uuid, dunning jsonb not null default '{"reminders_days":[-3,0,1,3,7],"on_exhausted":"cancel"}',
  metadata jsonb not null default '{}', created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  version integer not null default 0);
create table subscription_items (id uuid primary key, subscription_id uuid not null references subscriptions(id),
  price_id uuid not null references prices(id), quantity integer not null default 1);

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

`domain-model.md` §4.5 plus guards: finalize requires ≥ 1 line, `customer_id` exists (payments internal), total > 0 and ≥ destination minimum; `void` only from `open`; `uncollectible → paid` allowed (late payment, INV-3).

## 5. API

| Method & path | Scope | Notes |
|---|---|---|
| `POST/GET /v1/products…`, `/v1/prices…`, `/v1/tax_rates…` | `products.*` | CRUD (prices immutable except `active`) |
| `POST /v1/invoices` | `invoices.write` | draft with `lines[{price?|amount+description, quantity, tax_rates?}]`, `customer`, `collection_method='send_invoice'`, `days_until_due` |
| `POST /v1/invoices/{id}/finalize` · `/send` · `/void` · `/mark_uncollectible` | `invoices.write` | finalize creates the payments session and PDF |
| `GET /v1/invoices…` | `invoices.read` | |
| `GET /v1/invoices/{id}/public` | anonymous + `X-NearPay-Invoice-Secret` | hosted invoice data for the checkout SPA `/i/:id`; if the invoice is `open` and has no open payments session (expired), creates one via payments `POST /internal/checkout_sessions` and returns `{session_id, client_secret}` so the SPA continues with the normal checkout flow |
| `POST /v1/subscriptions` | `subscriptions.write` | `{customer, items[{price, quantity}], collection_method, days_until_due?, trial_period_days?, mandate?}` |
| `POST /v1/subscriptions/{id}` (update) · `/cancel` | `subscriptions.write` | `cancel_at_period_end` or immediate |
| `POST /v1/mandates/prepare` | `subscriptions.write` | Stage 4: returns `{receiver_id: vault, method:'ft_transfer_call', args:{amount, msg}}` for the payer to sign |
| `GET /v1/mandates…` | `subscriptions.read` | |
| `GET /internal/invoices/{id}/notification-context` | notify | customer email (via payments), amounts, links |

## 6. Events

Produced: `evt.billing.invoice.{created,finalized,paid,payment_failed,overdue,voided,marked_uncollectible}`, `evt.billing.subscription.{created,updated,deleted,renewal_upcoming}`, `evt.billing.mandate.{activated,collected,insufficient_funds,canceled,exhausted}`.
Consumed: `evt.payments.payment_intent.succeeded` (source invoice → paid), `evt.payments.checkout_session.expired` (clear `checkout_session_id`; a fresh session is created lazily on the next public view), `evt.rails.vault.event` (mandate created/collected/canceled/withdrawn). On `mandate_created` billing links the mandate to a subscription only if `ext_ref` = the subscription ID, `merchant_receiver` = an active Intents-account destination of that account, token/amount/period match the subscription's price — otherwise the mandate is recorded as `unlinked` and the payer is told to cancel.

## 7. Jobs

| Kind (dedupe key) | When | Behaviour |
|---|---|---|
| `subscription.renew` (`sub:<id>:<period_start>`) | `current_period_end` | create invoice for next period (draft → finalize); `send_invoice` → payments session + email; `auto_debit` → `mandate.collect`; advance period |
| `mandate.collect` (`md:<id>:<period>`) | after renew | keeper calls vault `collect(mandate_id)` (function-call key, 60 Tgas); success waits for `vault.event collected` (evidence) → invoice paid; tx failure with `ERR_INSUFFICIENT_BALANCE` → invoice switches to `send_invoice` path + `mandate.insufficient_funds`; `ERR_NOT_DUE`/`ERR_IN_FLIGHT` → retry at `next_collect_at`; other errors → retry ×5 then case |
| `invoice.remind` (`in:<id>:r<n>`) | per `dunning.reminders_days` relative to `due_date` | email; after last → `past_due` handling: `on_exhausted` cancel or `unpaid` |
| `subscription.renewal_upcoming` | 3 days before period end | event + email |

## 8. Configuration

`NP_VAULT_CONTRACT` (`v1.vault.nearpay.near`), `NP_KEEPER_ACCOUNT`, `NP_KEEPER_KEY_FILE`, `NP_NEAR_RPC_URLS`, `NP_OBJECT_STORE` (`fs:/data/invoices` local · `s3://…` server), `NP_PAY_ORIGIN`.

## 9. Tests

Proration is out of scope (no mid-period changes in v1: updates apply next period) — tested as rejected. Renewal idempotency across restarts; dunning schedule with fake clock; tax inclusive/exclusive rounding (per line, round half-up in minor units); auto-debit with sandbox vault simulation; late payment on `uncollectible`.

## 10. Stage mapping

Stage 3: products, prices, tax rates, invoices, hosted invoice, `send_invoice` subscriptions, dunning. Stage 4: mandates + keeper + auto-debit.
