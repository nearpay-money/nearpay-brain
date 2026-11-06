# LLD — `nearpay-notify`

> Version 1.2 (step 1.2 implementation findings folded: data §3, templates and copy rules §4, event fan-out §5, SMTP sending §6, configuration §7, stage map §9) · Version 1.1 audited (`audits/notify.md`) · Foundations `00-foundations.md`

## 1. Scope

**Owns:** transactional email (templates, rendering, sending, suppression, delivery status). No marketing email.

## 2. Runtime

`notify-api` (`api`, 2 replicas): internal API `/internal/messages` (login magic links depend on it). `notify-worker` (`worker`, 1 replica): consumers, send queue (jobs table), outbox relay. Local profile uses **Mailpit** (SMTP sink + web UI at `mail.nearpay.localhost`); server stage uses Postmark (primary) and Amazon SES (fallback).

## 3. Data

```sql
create table messages (
  id uuid primary key, account_id uuid, livemode boolean, template text not null, locale text not null default 'en',
  to_hash bytea not null,                 -- HMAC-SHA256 of the lower-cased address, key derived from the KEK
  to_enc bytea not null, variables_enc bytea not null,   -- envelope-encrypted
  status text not null check (status in ('queued','sent','failed','suppressed')), provider text, provider_message_id text,
  attempts smallint not null default 0, dedupe_key text unique,   -- '<template>:<caller dedupe key>'
  source text not null,                   -- 'api' | 'event:<subject>'
  last_error text, created_at timestamptz not null default now(), sent_at timestamptz, updated_at timestamptz not null default now());
create index messages_recipient on messages (to_hash, created_at desc);   -- per-recipient limit
create index messages_status on messages (status, created_at);
create table suppressions (to_hash bytea primary key, reason text not null check (reason in ('hard_bounce','complaint','manual')),
  detail text, created_at timestamptz not null default now());
```
Retention: messages are deleted after `NP_MESSAGE_RETENTION_DAYS` (90). A sent `auth.magic_link` has its variables wiped at once (the link is single-use and must not stay readable at rest).

## 4. Templates (compiled MJML in `templates/email`, variables validated by zod)

| Template | Trigger | Recipient |
|---|---|---|
| `auth.magic_link` | identity `POST /internal/messages` | user |
| `account.welcome` | `evt.identity.account.created` | owner |
| `security.destination_pending` · `_activated` · `_disabled` · `_blocked` | `evt.identity.settlement_destination.*`, `evt.compliance.destination.blocked` | all owners/admins |
| `security.member_changed` | `evt.identity.member.*` | owners |
| `kyb.decision` | `evt.compliance.kyb.*` | owners |
| `account.suspended` · `account.reinstated` | `evt.compliance.account.*` | owners |
| `review.opened` · `review.closed` | `evt.compliance.review.*` | owners/admins |
| `payment.receipt` | `evt.payments.payment_intent.succeeded` when the session has a customer email and the account enables receipts; test-mode receipts only when `NP_TEST_MODE_RECEIPTS=true` (default `false`; Compose local sets `true` because Mailpit catches every mail) | customer |
| `refund.receipt` | `evt.payments.refund.succeeded` | customer |
| `invoice.issued` · `invoice.reminder` · `invoice.paid` | `evt.billing.invoice.*` | customer |
| `subscription.renewal_upcoming` · `mandate.insufficient_funds` | `evt.billing.*` | customer |
| `webhooks.endpoint_disabled` | `evt.webhooks.endpoint.disabled` | developers + owners |
| `ops.kill_switch` · `ops.audit_finding` | `evt.sentinel.*` | security contacts (config) |

**Copy rules:** security mails always show the destination fingerprint; receipts show the fingerprint of the destination the payment settled to; internal compliance reasons are never shown to merchants (a `kyb.decision` mail shows a reason only for `needs_info`); test-mode mails say so; every value is HTML-escaped and only absolute http(s) links render. The worker renders every template sample at startup and refuses to start if one fails.

**Event fan-out:** the consumer only enqueues an `event.notify` job inside its inbox transaction; the job resolves the audience and queues one message per recipient with dedupe key `<event id>:<recipient>`. Identity or payments outages delay mail (job backoff) and never block the consumer.

Recipient resolution (never from event payloads, which carry no PII): identity `GET /internal/accounts/{id}/notification-recipients?kind=security|billing|developer`; payments `GET /internal/payment_intents/{id}/receipt-context`; billing `GET /internal/invoices/{id}/notification-context`.

## 5. API

| Method & path | Caller (scope) | Request → Response |
|---|---|---|
| `POST /internal/messages` | identity, billing (`notify.send`) | `{template, to, variables, locale?, dedupe_key}` → `202 {message_id}` |

## 6. Sending

Job `message.send` per message (dedupe `template:dedupe_key`): suppression check → per-recipient limit 20/hour → render (MJML, strict) → provider call 5 s. HTTP providers (Stage 5): 4xx permanent → `failed`; 5xx/timeouts transient. SMTP (Stage 1) is classified by reply code the other way round: 5xx permanent, 4xx transient, and a rejected recipient (550/551/553) is suppressed. Transient failures retry with backoff for 8 attempts, then the fallback provider (Stage 5) for 2. Every mail carries `Message-ID: <message uuid@domain>` so an at-least-once re-send is recognisable. Bounces/complaints (Postmark webhook `/webhooks/email/postmark`, Stage 5) → `suppressions`.

## 7. Configuration

`NP_EMAIL_PROVIDER` (`smtp` local · `postmark` server), `NP_SMTP_URL` (`smtp://mailpit:1025`), `NP_POSTMARK_TOKEN_FILE`, `NP_SES_*_FILE`, `NP_FROM` (`NearPay <no-reply@nearpay.money>`), `NP_SECURITY_CONTACTS` (comma-separated), `NP_PII_KEK_FILE`, `NP_TEST_MODE_RECEIPTS` (`false`), `NP_DASHBOARD_ORIGIN` (links in merchant mails), `NP_IDENTITY_URL`, `NP_PAYMENTS_URL` (recipient resolution), `NP_SEND_TIMEOUT_MS` (5000), `NP_RECIPIENT_HOURLY_LIMIT` (20), `NP_MESSAGE_RETENTION_DAYS` (90).

## 8. Tests

Template snapshot tests per locale; dedupe; suppression; provider fallback; recipient resolution contract tests against identity/payments/billing.

## 9. Stage mapping

Stage 1: Mailpit, every subject the event catalogue routes to notify (magic link, welcome, security destination emails, KYB decision, suspension, reviews, payment receipt, kill switch, audit finding, webhooks endpoint disabled). Stage 2: refund receipts, member emails. Stage 3: billing templates. Stage 5: Postmark/SES, bounce handling.
