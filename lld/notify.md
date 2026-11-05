# LLD — `nearpay-notify`

> Version 1.1 (audited, `audits/notify.md`) · Foundations `00-foundations.md`

## 1. Scope

**Owns:** transactional email (templates, rendering, sending, suppression, delivery status). No marketing email.

## 2. Runtime

`notify-api` (`api`, 2 replicas): internal API `/internal/messages` (login magic links depend on it). `notify-worker` (`worker`, 1 replica): consumers, send queue (jobs table), outbox relay. Local profile uses **Mailpit** (SMTP sink + web UI at `mail.nearpay.localhost`); server stage uses Postmark (primary) and Amazon SES (fallback).

## 3. Data

```sql
create table messages (
  id uuid primary key, account_id uuid, livemode boolean, template text not null, locale text not null default 'en',
  to_hash bytea not null, to_enc bytea not null, variables_enc bytea not null,
  status text not null check (status in ('queued','sent','failed','suppressed')), provider text, provider_message_id text,
  attempts smallint not null default 0, dedupe_key text unique, last_error text,
  created_at timestamptz not null default now(), sent_at timestamptz);
create table suppressions (to_hash bytea primary key, reason text not null, created_at timestamptz not null default now());
```

## 4. Templates (compiled MJML in `templates/email`, variables validated by zod)

| Template | Trigger | Recipient |
|---|---|---|
| `auth.magic_link` | identity `POST /internal/messages` | user |
| `account.welcome` | `evt.identity.account.created` | owner |
| `security.destination_pending` · `_activated` · `_disabled` | `evt.identity.settlement_destination.*` | all owners/admins |
| `security.member_changed` | `evt.identity.member.*` | owners |
| `kyb.decision` | `evt.compliance.kyb.*` | owners |
| `payment.receipt` | `evt.payments.payment_intent.succeeded` when the session has a customer email and the account enables receipts | customer |
| `refund.receipt` | `evt.payments.refund.succeeded` | customer |
| `invoice.issued` · `invoice.reminder` · `invoice.paid` | `evt.billing.invoice.*` | customer |
| `subscription.renewal_upcoming` · `mandate.insufficient_funds` | `evt.billing.*` | customer |
| `webhooks.endpoint_disabled` | `evt.webhooks.endpoint.disabled` | developers + owners |
| `ops.kill_switch` · `ops.audit_finding` | `evt.sentinel.*` | security contacts (config) |

Recipient resolution (never from event payloads, which carry no PII): identity `GET /internal/accounts/{id}/notification-recipients?kind=security|billing|developer`; payments `GET /internal/payment_intents/{id}/receipt-context`; billing `GET /internal/invoices/{id}/notification-context`.

## 5. API

`POST /internal/messages` (identity, billing; scope `notify.send`): `{template, to, variables, locale?, dedupe_key}` → `202 {message_id}`.

## 6. Sending

Job per message (dedupe `template:dedupe_key`); suppression check; provider call 5 s; 4xx permanent → `failed`; 5xx/timeouts retry with backoff (max 8) then fallback provider; bounces/complaints (Postmark webhook `/webhooks/email/postmark`, Stage 5) → `suppressions`. Per-recipient limit 20/hour.

## 7. Configuration

`NP_EMAIL_PROVIDER` (`smtp` local · `postmark` server), `NP_SMTP_URL` (`smtp://mailpit:1025`), `NP_POSTMARK_TOKEN_FILE`, `NP_SES_*_FILE`, `NP_FROM` (`NearPay <no-reply@nearpay.money>`), `NP_SECURITY_CONTACTS`, `NP_PII_KEK_FILE`.

## 8. Tests

Template snapshot tests per locale; dedupe; suppression; provider fallback; recipient resolution contract tests against identity/payments/billing.

## 9. Stage mapping

Stage 1: Mailpit, magic link, security destination emails, payment receipt. Stage 2: KYB, refund receipts, member emails, webhooks disabled. Stage 3: billing templates. Stage 5: Postmark/SES, bounce handling.
