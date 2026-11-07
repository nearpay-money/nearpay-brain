# LLD — `nearpay-notify`

> Spec 0.2.1 update (spec 0.2.1: recovery and secret-rotation mails, recipients by user, `created_by` on destinations — §"Spec 0.2.1 contract changes") · Version 1.4 (step 2.2 implementation, spec 0.2.0: Stage 2 audiences, refund receipt and approval-request rules, credential wipe, brand §4; console link §7) · Version 1.3 (step 2.1 contracts: member, invitation, operator-enrolment, KYB-submitted and approval-request mails §4–§5, stage map §9) · Version 1.2 (step 1.2 implementation findings folded: data §3, templates and copy rules §4, event fan-out §5, SMTP sending §6, configuration §7, stage map §9) · Version 1.1 audited (`audits/notify.md`) · Foundations `00-foundations.md`

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
Retention: messages are deleted after `NP_MESSAGE_RETENTION_DAYS` (90). A sent `auth.magic_link`, `team.invitation` or `ops.enrolment` has its variables wiped at once (the link is single-use and must not stay readable at rest).

## 4. Templates (compiled MJML in `templates/email`, variables validated by zod)

| Template | Trigger | Recipient |
|---|---|---|
| `auth.magic_link` | identity `POST /internal/messages` | user |
| `account.welcome` | `evt.identity.account.created` | owner |
| `security.destination_pending` · `_activated` · `_disabled` · `_blocked` | `evt.identity.settlement_destination.*`, `evt.compliance.destination.blocked` | all owners/admins |
| `security.member_changed` | `evt.identity.member.added` · `.removed` · `.role_changed` (step 2.1; `MemberEvent` has ids only — names/emails via identity recipients) | owners |
| `team.invitation` (step 2.1) | identity `POST /internal/messages` (`TeamInvitationMessage`: link, account name, role, 7-day validity) | invitee |
| `ops.enrolment` (step 2.1) | identity `POST /internal/messages` (`OpsEnrolmentMessage`, from `operator-add`) | operator |
| `security.destination_approval` (step 2.1) | `evt.identity.settlement_destination.pending_approval` | owners/admins other than the creator |
| `kyb.submitted` (step 2.1) | `evt.compliance.kyb.submitted` | owners |
| `kyb.decision` | `evt.compliance.kyb.approved` · `.rejected` · `.needs_info` (step 2.1: shows `requested_info` for `needs_info`; never `reason`) | owners |
| `ops.approval_requested` (step 2.1) | `evt.compliance.account.action_requested`, `evt.sentinel.kill_switch.release_requested` | security contacts (config) |
| `account.suspended` · `account.reinstated` | `evt.compliance.account.*` | owners |
| `review.opened` · `review.closed` | `evt.compliance.review.*` | owners/admins |
| `payment.receipt` | `evt.payments.payment_intent.succeeded` when the session has a customer email and the account enables receipts; test-mode receipts only when `NP_TEST_MODE_RECEIPTS=true` (default `false`; Compose local sets `true` because Mailpit catches every mail) | customer |
| `refund.receipt` | `evt.payments.refund.succeeded` | customer |
| `invoice.issued` · `invoice.reminder` · `invoice.paid` | `evt.billing.invoice.*` | customer |
| `subscription.renewal_upcoming` · `mandate.insufficient_funds` | `evt.billing.*` | customer |
| `webhooks.endpoint_disabled` | `evt.webhooks.endpoint.disabled` | developers + owners |
| `ops.kill_switch` · `ops.audit_finding` | `evt.sentinel.*` | security contacts (config) |

**Copy rules:** security mails always show the destination fingerprint; receipts show the fingerprint of the destination the payment settled to; internal compliance reasons are never shown to merchants (a `kyb.decision` mail shows only the event's `requested_info`, for `needs_info`; the decision `reason` is never mailed, not even as a fallback for a Stage 1 producer); KYB is self-declared business details reviewed by NearPay ops (ADR-0020 d3), so no mail says "verified" or "verification"; refund mails say the merchant sent the refund from their own wallet and NearPay never held the funds (INV-1); test-mode mails say so ("Test mode — no real funds moved."); every value is HTML-escaped and only absolute http(s) links render. The worker renders every template sample at startup and refuses to start if one fails.

**Look (step 2.2):** one MJML layout in the NearPay brand (`product/design-system.md` §2): deep ink `#132D31` text, NearPay teal `#087A68` buttons and links, warm canvas `#F7F9F8` background, fine borders `#DCE5E3`, 12 px card radius, 8 px controls; DM Sans when installed, else the system sans (no remote fonts or images: a remote asset is a tracking request). Blocks: paragraph, note, facts table, code box (addresses, fingerprints), button, links, callout (soft mint for requests such as KYB `requested_info`, warm cream for test mode).

**Stage 2 audience rules (step 2.2):**
- `refund.receipt`: payments `GET /internal/refunds/{id}/receipt-context` (`getRefundReceiptContext`); nothing is sent when `email_receipts` is false, `customer_email` is null, or the context's `livemode` differs from the event's (fail closed). Test-mode refund receipts follow `NP_TEST_MODE_RECEIPTS`, like payment receipts. The mail shows what reached the payer (`received`, else `refunded`), what the merchant sent when it differs, the shortened refund address, the original priced amount (fiat only) and the payer-side evidence links.
- `security.member_changed`: owners. `MemberEvent` carries ids only; the member's and actor's emails are looked up by user id in the same `notification-recipients?kind=security` response (owners and admins), else the mail shows the user id (a removed member or a non-admin role is never listed). Only owners are mailed (an owner who was just added receives it too).
- `security.destination_approval`: owners/admins minus every user in `approval.approved_by` (the creator counts as the first approval, identity §5), so neither the creator nor an earlier approver is asked again; a destination with `rejected_by` sends nothing.
- `ops.approval_requested`: only for an `OpsApproval` with `status: pending`; it shows action, target, the requester's reason (ops-only audience), requester id/role, expiry and a console link (`NP_CONSOLE_ORIGIN`).
- Every audience is de-duplicated per user, so the message dedupe key `<template>:<event id>:<user id>` never collides (INV-4).

**Event fan-out:** the consumer only enqueues an `event.notify` job inside its inbox transaction; the job resolves the audience and queues one message per recipient with dedupe key `<event id>:<recipient>`. Identity or payments outages delay mail (job backoff) and never block the consumer.

Recipient resolution (never from event payloads, which carry no PII): identity `GET /internal/accounts/{id}/notification-recipients?kind=security|billing|developer`; payments `GET /internal/payment_intents/{id}/receipt-context`; billing `GET /internal/invoices/{id}/notification-context`.

## 5. API

| Method & path | Caller (scope) | Request → Response |
|---|---|---|
| `POST /internal/messages` | identity, billing (`notify.send`) | `{template, to, variables, locale?, dedupe_key}` → `202 {message_id}`; templates `auth.magic_link`, `team.invitation`, `ops.enrolment` (step 2.1; links with single-use tokens never travel in events). The invitation mail names the account, never the inviter's email; KYB mails never show internal reasons |

## 6. Sending

Job `message.send` per message (dedupe `template:dedupe_key`): suppression check → per-recipient limit 20/hour → render (MJML, strict) → provider call 5 s. HTTP providers (Stage 5): 4xx permanent → `failed`; 5xx/timeouts transient. SMTP (Stage 1) is classified by reply code the other way round: 5xx permanent, 4xx transient, and a rejected recipient (550/551/553) is suppressed. Transient failures retry with backoff for 8 attempts, then the fallback provider (Stage 5) for 2. Every mail carries `Message-ID: <message uuid@domain>` so an at-least-once re-send is recognisable. Bounces/complaints (Postmark webhook `/webhooks/email/postmark`, Stage 5) → `suppressions`.

## 7. Configuration

`NP_EMAIL_PROVIDER` (`smtp` local · `postmark` server), `NP_SMTP_URL` (`smtp://mailpit:1025`), `NP_POSTMARK_TOKEN_FILE`, `NP_SES_*_FILE`, `NP_FROM` (`NearPay <no-reply@nearpay.money>`), `NP_SECURITY_CONTACTS` (comma-separated), `NP_PII_KEK_FILE`, `NP_TEST_MODE_RECEIPTS` (`false`), `NP_DASHBOARD_ORIGIN` (links in merchant mails), `NP_CONSOLE_ORIGIN` (`https://ops.<domain>`, links in ops approval mails; step 2.2), `NP_IDENTITY_URL`, `NP_PAYMENTS_URL` (recipient resolution), `NP_SEND_TIMEOUT_MS` (5000), `NP_RECIPIENT_HOURLY_LIMIT` (20), `NP_MESSAGE_RETENTION_DAYS` (90).

## 8. Tests

Template snapshot tests per locale; dedupe; suppression; provider fallback; recipient resolution contract tests against identity/payments/billing.

## 9. Stage mapping

Stage 1: Mailpit, every subject the event catalogue routes to notify (magic link, welcome, security destination emails, KYB decision, suspension, reviews, payment receipt, kill switch, audit finding, webhooks endpoint disabled). Stage 2: refund receipts (payments), member, invitation, destination-approval, KYB-submitted, operator-enrolment and approval-request emails (step 2.1 contracts; implemented in step 2.2 with the brand restyle and self-declared KYB wording). Stage 3: billing templates. Stage 5: Postmark/SES, bounce handling.

## Spec 0.2.1 contract changes (step 2.2/2.3 change requests, `nearpay-spec/CHANGELOG.md`)

- `security.recovery` on **`evt.identity.user.recovered`**: per account in `account_ids`, owners (and the recovered user via `notification-recipients?user=`), showing `lock_until`.
- `webhooks.secret_rotated` on **`evt.webhooks.endpoint.secret_rotated`**: developers + owners, URL and `previous_secret_expires_at` (never the secret).
- `notification-recipients` accepts `user` (one member, any role) — e.g. to mail the affected member of a `member.*` event.
- `InternalSettlementDestination.created_by` is available for destination mails (e.g. excluding the creator).
