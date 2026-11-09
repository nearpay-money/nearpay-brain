# LLD — `nearpay-notify`

> Version 1.9 (step 4.2 implementation, spec 0.4.0: mandate mails — §"Stage 4 mandate mails (step 4.2)"; renewal copy for `auto_debit`) · Version 1.8 (step 4.1, spec 0.4.0: mandate mails to the payer — §4, `getMandateNotificationContext`) · Version 1.7 (step 3.2 implementation, spec 0.3.0: billing mails — §"Stage 3 billing mails (step 3.2)"; `NP_BILLING_URL` §7) · Version 1.6 (step 3.1, spec 0.3.0: billing templates §4) · Version 1.5 (spec 0.2.1 implementation: manager changes to owners and admins, recovery and rotation audiences, names via ?user= — §"Spec 0.2.1 contract changes") · Spec 0.2.1 update (spec 0.2.1: recovery and secret-rotation mails, recipients by user, `created_by` on destinations — §"Spec 0.2.1 contract changes") · Version 1.4 (step 2.2 implementation, spec 0.2.0: Stage 2 audiences, refund receipt and approval-request rules, credential wipe, brand §4; console link §7) · Version 1.3 (step 2.1 contracts: member, invitation, operator-enrolment, KYB-submitted and approval-request mails §4–§5, stage map §9) · Version 1.2 (step 1.2 implementation findings folded: data §3, templates and copy rules §4, event fan-out §5, SMTP sending §6, configuration §7, stage map §9) · Version 1.1 audited (`audits/notify.md`) · Foundations `00-foundations.md`

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
| `payment.receipt` | `evt.payments.payment_intent.succeeded` when the session has a customer email and the account enables receipts (step 3.1: not for `source: invoice` intents — `invoice.paid` replaces it); test-mode receipts only when `NP_TEST_MODE_RECEIPTS=true` (default `false`; Compose local sets `true` because Mailpit catches every mail) | customer |
| `refund.receipt` | `evt.payments.refund.succeeded` | customer |
| `invoice.issued` · `invoice.reminder` · `invoice.paid` · `invoice.voided` (step 3.1) | `evt.billing.invoice.sent` · `.reminder` · `.paid` · `.voided` — data from billing `getInvoiceNotificationContext` (`?reminder=<n>` for reminders; includes the hosted link with the invoice secret, never in events) | customer (`customer_email` of the context; null → no mail) |
| `subscription.renewal_upcoming` (step 3.1) · `mandate.insufficient_funds` (Stage 4) | `evt.billing.subscription.renewal_upcoming` (`getSubscriptionNotificationContext`; nothing when `cancel_at_period_end`) · `evt.billing.mandate.insufficient_funds` | customer |
| `mandate.activated` · `mandate.insufficient_funds` · `mandate.canceled` · `mandate.unlinked` · `mandate.released` (step 4.1, spec 0.4.0) | `evt.billing.mandate.activated` · `.insufficient_funds` · `.canceled` · `.unlinked` · `.released` — data from billing `getMandateNotificationContext` (`MandateNotificationContext`: token display, amounts, balance, next due time, `refunded`, the payer's `cancel_transaction`/`withdraw_transaction`, `manage_url` = the latest invoice's hosted link with its secret — never in events). Copy: `activated` confirms auto-debit and always shows how to cancel (vault, mandate id, the cancel call — FR-BIL-04); `insufficient_funds` asks to top up or pay the emailed invoice; `canceled` is the refund receipt (`refunded`, or "the refund failed — withdraw from the page" when `"0"`); `unlinked` says NearPay will not collect this mandate and asks the payer to cancel it to get the funds back; `released` says the subscription ended and asks the payer to cancel to get the balance back. Every mandate mail states NearPay never holds the funds and cannot move them (INV-1) | customer (`customer_email` of the context; null → no mail; context `livemode` ≠ event's → nothing) |
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

`NP_EMAIL_PROVIDER` (`smtp` local · `postmark` server), `NP_SMTP_URL` (`smtp://mailpit:1025`), `NP_POSTMARK_TOKEN_FILE`, `NP_SES_*_FILE`, `NP_FROM` (`NearPay <no-reply@nearpay.money>`), `NP_SECURITY_CONTACTS` (comma-separated), `NP_PII_KEK_FILE`, `NP_TEST_MODE_RECEIPTS` (`false`), `NP_DASHBOARD_ORIGIN` (links in merchant mails), `NP_CONSOLE_ORIGIN` (`https://ops.<domain>`, links in ops approval mails; step 2.2), `NP_IDENTITY_URL`, `NP_PAYMENTS_URL`, `NP_BILLING_URL` (recipient resolution; billing since step 3.2), `NP_SEND_TIMEOUT_MS` (5000), `NP_RECIPIENT_HOURLY_LIMIT` (20), `NP_MESSAGE_RETENTION_DAYS` (90).

## 8. Tests

Template snapshot tests per locale; dedupe; suppression; provider fallback; recipient resolution contract tests against identity/payments/billing.

## 9. Stage mapping

Stage 1: Mailpit, every subject the event catalogue routes to notify (magic link, welcome, security destination emails, KYB decision, suspension, reviews, payment receipt, kill switch, audit finding, webhooks endpoint disabled). Stage 2: refund receipts (payments), member, invitation, destination-approval, KYB-submitted, operator-enrolment and approval-request emails (step 2.1 contracts; implemented in step 2.2 with the brand restyle and self-declared KYB wording). Stage 3: billing templates (`invoice.issued`, `invoice.reminder`, `invoice.paid`, `invoice.voided`, `subscription.renewal_upcoming`; spec `BILLING_EMAIL_TEMPLATES`; a mail is skipped when the context's `livemode` differs from the event's or the status no longer fits, e.g. a reminder for an invoice paid meanwhile; test-mode mails follow `NP_TEST_MODE_RECEIPTS`). Stage 4: mandate mails to the payer (`mandate.activated`, `.insufficient_funds`, `.canceled`, `.unlinked`, `.released`; §"Stage 4 mandate mails (step 4.2)"). Stage 5: Postmark/SES, bounce handling.

## Spec 0.2.1 contract changes (step 2.2/2.3 change requests, `nearpay-spec/CHANGELOG.md`)

- `security.recovery` on **`evt.identity.user.recovered`**: per account in `account_ids`, owners (and the recovered user via `notification-recipients?user=`), showing `lock_until`.
- `webhooks.secret_rotated` on **`evt.webhooks.endpoint.secret_rotated`**: developers + owners, URL and `previous_secret_expires_at` (never the secret).
- `notification-recipients` accepts `user` (one member, any role) — e.g. to mail the affected member of a `member.*` event.
- `InternalSettlementDestination.created_by` is available for destination mails (e.g. excluding the creator).

**Implementation (spec 0.2.1, v1.5).**

- `security.member_changed`: when the member's `role` or `previous_role` is owner or admin, owners **and admins** are mailed (identity audit fix d: the other managers see every manager change); other roles reach owners only. `member_email` and `actor_email` come from the recipient list, else from `notification-recipients?user=` (any role, and members removed in the last 7 days), else the id is shown.
- `security.recovery`: audience `accounts`, the owners and admins of every account in `account_ids`, mailed once per user across the accounts (dedupe key `<event id>:<user id>`; the recovered user is one of them). The user is named through the same lookup. An empty `account_ids` sends nothing.
- `webhooks.secret_rotated`: identity `kind=developer`, i.e. owners, admins and developers (admins are included, unlike the earlier "developers + owners" line). The mail shows the URL, the overlap end or "dropped at once", the rotating user (by email when listed) and a test-mode banner for test endpoints. It never shows a secret.
- `security.destination_approval` also excludes `created_by` (deduplicated with `approval.approved_by`).
- Snapshots and samples: `security.recovery` (2) and `webhooks.secret_rotated` (2).

## Stage 3 billing mails (step 3.2)

Implementation decisions (spec 0.3.0, kit 0.2.1):

- **Subjects → templates** exactly as `BILLING_EMAIL_TEMPLATES`: `evt.billing.invoice.sent|reminder|paid|voided` → `invoice.issued|reminder|paid|voided`, `evt.billing.subscription.renewal_upcoming` → `subscription.renewal_upcoming`. `evt.billing.invoice.overdue` is not routed to notify by the catalogue, so the overdue state is mailed through the reminders with a positive offset ("Overdue: …"), not a separate mail.
- **Audience** `invoice_customer` / `subscription_customer`: the `event.notify` job reads the context route (`?reminder=<event number>` for reminders); 404 → nothing sent, other errors → job backoff (billing outages delay mail, never block the consumer). The message is filed under the context's `account_id`; dedupe key `<template>:<event id>:customer` (a merchant's re-`send` is a new event and a new mail, by design).
- **Fail closed** (nothing sent, reason logged without data): the context names another invoice/subscription than the event; context `livemode` ≠ event `livemode` (a null-mode envelope never matches); `customer_email` null; status no longer fits — `invoice.issued`/`invoice.reminder` need `open` or `uncollectible` (an uncollectible invoice can still be paid, INV-3, and the last reminder may race the dunning transition), `invoice.paid` needs `paid`, `invoice.voided` needs `void`; renewal needs `active|trialing|past_due` and `cancel_at_period_end: false` (also checked on the event); issue/reminder mails need `hosted_invoice_url`. A reminder takes the context's `reminder` block, else the event's `number/offset_days/last`.
- **Test mode:** billing mails follow `NP_TEST_MODE_RECEIPTS` (like receipts) and carry the test-mode banner and `[test]` subject tag.
- **`invoice.paid`** reads payments `receipt-context` of the context's `payment_intent` for "Settled" and the on-chain proof links (404 → mail without them). Invoice mails do not apply the account's `email_receipts` switch (that setting governs `payment.receipt`/`refund.receipt`; billing controls invoice mail recipients through `customer_email`). `payment.receipt` is skipped for intents with `source: invoice` or a non-null `invoice` (in the consumer, no job).
- **Copy:** merchant name from the context; customer greeting only when `customer_name` is known; amounts stay integers in the variables (fiat minor units, asset base units + `{symbol, decimals}` from `pricing_asset`) and are formatted by the template (asset: at least 2 fraction digits); the variables schema rejects a fiat/asset mix-up. Lines show `description × quantity → amount`; because the context has no subtotal/tax, a "Tax" row = `total − Σ lines` is shown when fewer than 20 lines are listed (all lines present; inclusive tax is inside the line amounts), else "More lines and tax: On the invoice page". Reminder copy depends on `offset_days` (< 0 "due in N days", 0 "due today", > 0 "Overdue"); the last reminder says so (subscription invoices: "may be canceled or paused"). Every invoice mail says payments settle directly to the merchant and NearPay never holds the funds (INV-1); the void mail says a payment already sent is refunded by the merchant from their own wallet (INV-3). Renewal mails say nothing is charged automatically (v1 is `send_invoice` only).
- **Invoice secret:** the hosted link exists only inside the envelope-encrypted variables until the mail is sent; `invoice.issued`, `invoice.reminder` and `invoice.paid` are wiped like single-use credential mails once sent; the void mail's schema has no link field. `LinkError` never echoes a refused URL (it reaches logs and `last_error`). Integration tests assert no notify log line contains the sample secret.
- **Gaps (change requests for spec 0.3.1):** see the step 3.2 report — `InvoiceNotificationContext` lacks `subtotal`/`tax`/line count (tax row is derived) and a settlement destination fingerprint for `invoice.paid` (the receipt copy rule "show the fingerprint of the destination the payment settled to" cannot be met: `ReceiptContext` carries no destination and notify holds no `payments.read`).


## Stage 4 mandate mails (step 4.2)

Implementation decisions (spec 0.4.0, kit 0.2.1):

- **Subjects → templates** exactly as `BILLING_EMAIL_TEMPLATES`: `evt.billing.mandate.activated|insufficient_funds|canceled|unlinked|released` → `mandate.<same>`. `collected`, `exhausted` and `updated` are webhooks-only (the catalogue does not route them to notify).
- **Routing (consumer).** Nested-livemode guard: the payload `Mandate.livemode` must equal the envelope's, else nothing is enqueued. Test-mode mandate mails follow `NP_TEST_MODE_RECEIPTS`. Audience `mandate_customer {mandate, subscription, envelope account}`.
- **Fail closed (job, nothing sent, reason logged without data):** 404 → nothing; other errors → job backoff. The context must name the event's mandate and subscription, and its `account_id` must be the envelope's account (when the envelope has one); context `livemode` = event `livemode`; `customer_email` not null. Status fits — `activated`: `active|insufficient` and `linked`; `insufficient_funds`: `insufficient` and `linked` (a top-up or a cancel meanwhile makes it moot); `canceled`: `canceled` with `refunded` not null; `unlinked`: `active|insufficient` and `unlinked`; `released`: `active|insufficient` (a mandate canceled meanwhile needs no "cancel to get your funds back" mail). `activated`, `unlinked` and `released` need the `cancel_transaction` (FR-BIL-04: the confirmation always shows how to cancel; the other two ask the payer to cancel). Every payer transaction billing hands over must be exactly one `FunctionCall` on the context's `vault_contract` with method `cancel` (resp. `withdraw`) and `signer_id` null or the payer; otherwise the whole mail is skipped (a mail must never send the payer to another contract).
- **Copy.** Amounts are token base units formatted with the token's decimals (≥ 2 fraction digits) and symbol; the period reads `per month` / `every 2 weeks` from the subscription's `recurring` (`per period` when null); "covers N periods" = ⌊balance / amount_per_period⌋. Payer accounts longer than 32 characters (implicit) are shortened `8…8`. Payer calls are spelled out as contract, method, deposit (`1 yoctoNEAR`; ≥ 10¹⁸ yocto in NEAR), gas (`100 Tgas`) and the JSON arguments in a code box, signed by the payer's account, so the payer can cancel from any wallet without the hosted page. `canceled`: "Returned to you" when `refunded` > 0; `refunded` = `"0"` with a `withdraw_transaction` → "the refund failed — withdraw" with the withdraw call; without one → "No balance remained". `unlinked` maps `unlinked_reason` to plain words (unknown values → "its terms do not match the subscription's"). Every mandate mail ends with the INV-1 line (NearPay never holds the funds and cannot move them; only the payer or the merchant's receiving account can cancel) and, with a `manage_url`, the personal-link warning. Test mode: `[test]` subject tag and the "sandbox vault stand-in; no real funds moved" banner.
- **Secret.** `manage_url` (latest invoice's hosted link with its secret) exists only inside the envelope-encrypted variables until sent; all five mandate templates are wiped once sent, like the invoice mails. Integration tests assert no notify log line contains the sample secret or the payer's email.
- **Renewal mail (`subscription.renewal_upcoming`).** The variables gain `auto_debit` (from the event's `Subscription.collection_method`; default `false` for queued Stage 3 messages): auto-debit copy says the renewal is collected from the mandate and that an invoice is emailed if the mandate cannot pay; otherwise "Nothing is charged automatically" as before. (Fixes the Stage 3 copy that rendered a literal `$v.merchant_name … $day`.)
- **Gaps (CR candidates, not blocking).** `SubscriptionNotificationContext` has no `collection_method` (the renewal mail reads it from the event; a context field would keep the context the single authority). `MandateNotificationContext.withdraw_transaction` does not say the withdrawable amount after a failed refund (`balance` is shown when > 0, else "your remaining balance").
