---
book: The Complete Modern PHP Engineering Book
volume: 19
volume_title: SMALL ENGINEERING PROJECTS
chapter: 285
title: Notification Dispatcher
slug: notification-dispatcher
status: complete
summary: ../../_ai/chapter-summaries/285-notification-dispatcher-summary.md
---

# Chapter 285 — Notification Dispatcher

## Why This Matters

A notification dispatcher turns a business event into an email, SMS, push message, webhook, or in-app alert. The first implementation often calls a provider directly from a controller. That couples the user request to provider latency, leaks template and preference decisions into business code, and creates an awkward failure question: what if the database commits but the provider call times out?

This project builds a dispatcher for the reservation and import systems from Chapters 280 and 283. It separates notification intent from transport, applies recipient and preference policy, renders bounded content, persists an outbox job, delivers with provider-aware retries and idempotency, and records enough evidence to reconcile unknown outcomes.

## Define the Intent Contract

The producer should create a semantic intent, not a provider-specific payload:

~~~text
event: reservation.confirmed or import.completed
tenant and actor: trusted application context
recipient: authorized destination reference, not arbitrary user input
template: named versioned template
locale: explicit or policy-derived locale
channels: allowed channel preferences, with fallback policy
operation_id: stable business operation identity
effect: one notification intent may have channel-specific attempts
privacy: data minimization and retention apply to intent and delivery logs
~~~

“Send an email” is an implementation choice. “Tell the member their reservation was confirmed” is a domain intent. The dispatcher can choose email, push, or in-app delivery according to consent, capability, urgency, and channel policy without making the reservation service know provider APIs.

## Keep the Outbox Boundary

Write the business state and notification intent in one database transaction:

~~~text
reservation update + notification_outbox row
                         ↓ commit
                  dispatcher worker
                         ↓
              provider attempt + evidence
~~~

If the process dies after commit but before publishing a queue message, an outbox scanner can recover the intent. If the provider call succeeds and the worker dies before marking it delivered, the next attempt must use a provider idempotency key or reconciliation process. The database transaction cannot make an external provider atomic.

An outbox record should contain an opaque ID, event type, aggregate reference, operation identity, template/policy version, channel, recipient reference, safe payload reference, status, attempt count, next-attempt time, lease, and timestamps. Avoid placing secrets or unnecessary personal data into a queue or log when a protected reference can be resolved at delivery time.

## Model Channel Differences

Channels are not interchangeable:

| Channel | Main contract | Typical failure |
| --- | --- | --- |
| email | provider accepts a message for delivery | accepted does not mean inbox delivery |
| SMS | provider accepts a message to a number | quota, carrier, or number policy |
| push | token and platform-specific payload | expired or revoked token |
| webhook | signed HTTP request to a consumer | timeout with unknown completion |
| in-app | durable product notification | storage and unread-state races |

Use a narrow port:

~~~php
<?php

declare(strict_types=1);

enum DeliveryStatus: string
{
    case Accepted = 'accepted';
    case Rejected = 'rejected';
    case RetryableFailure = 'retryable_failure';
    case Unknown = 'unknown';
}

final readonly class NotificationIntent
{
    public function __construct(
        public string $intentId,
        public string $operationId,
        public string $channel,
        public string $recipientRef,
        public string $templateName,
        public string $templateVersion,
    ) {
        if ($intentId === '' || $operationId === '' || $channel === '' || $recipientRef === '' || $templateName === '' || $templateVersion === '') {
            throw new InvalidArgumentException('Notification intent is incomplete');
        }
    }
}

final readonly class NotificationDelivery
{
    public function __construct(
        public string $deliveryId,
        public string $intentId,
        public int $attempt,
    ) {
        if ($deliveryId === '' || $intentId === '' || $attempt < 1) {
            throw new InvalidArgumentException('Notification delivery is incomplete');
        }
    }
}

final readonly class DeliveryReceipt
{
    public function __construct(
        public DeliveryStatus $status,
        public ?string $providerRequestId,
        public ?string $providerMessageId,
    ) {
    }
}

interface NotificationProvider
{
    public function deliver(NotificationIntent $intent, NotificationDelivery $delivery, string $idempotencyKey): DeliveryReceipt;
}
~~~

The provider status is about the provider boundary, not necessarily the end user. `Accepted` may mean queued by the provider, while `Unknown` means the dispatcher cannot safely know whether the provider accepted the request. A channel adapter maps provider-specific responses into these stable categories and returns provider request/message IDs as protected evidence in a `DeliveryReceipt`. `RetryableFailure` means the provider request was known not to be accepted; it is distinct from `Unknown`, where acceptance cannot be established.

## Preferences, Consent, and Authorization

Before rendering or sending, evaluate:

* whether the actor may cause this notification;
* whether the recipient belongs to the same tenant or has an explicit relationship;
* whether the channel is verified and currently usable;
* marketing consent, transactional-message rules, and unsubscribe state;
* quiet hours, locale, time zone, urgency, and accessibility preferences;
* data classification and whether the channel is allowed to carry it;
* rate, quota, and duplicate-notification policy.

Do not treat a queue message as proof that a recipient is authorized. In this project, authorization is checked when the intent is created and again at dispatch; consent, unsubscribe, suppression, and channel capability are checked immediately before provider submission. A transactional reservation confirmation may have different consent rules from a marketing message, but both need an explicit policy and audit record.

A user-controlled display name belongs in a safely escaped template value. It must never become a template expression, header, HTML fragment, SQL statement, shell argument, or provider credential. Keep template selection and variables separate.

## Render Bounded Content

Templates should be named, versioned, reviewed, and rendered with context-specific escaping. Define subject length, body size, attachment limits, URL policy, localization fallback, and missing-variable behavior. A missing translation should select a documented fallback or fail the notification; it should not emit an empty security-sensitive message.

Avoid rendering a full business aggregate inside the worker. Resolve a stable, authorized view or snapshot so a later retry does not accidentally disclose fields added after the event. If the message must reflect the state at event time, persist the minimal event data or a versioned payload reference under the retention policy.

## Idempotency and Duplicate Delivery

Use a stable key such as `intent_id:channel` for the effect. The same notification intent may legitimately produce one email and one in-app alert, so a global key of only `intent_id` would suppress the wrong channel. Store provider request ID, result status, and last attempt.

The state machine might be:

~~~text
pending → delivering → accepted
                    ├→ rejected
                    ├→ unknown
                    └→ retryable failure → pending
~~~

Only a lease owner may transition its attempt. If the worker times out after sending, the next worker first checks durable delivery evidence and the provider’s idempotency or lookup API. Blindly sending again can duplicate an SMS or webhook. For providers with no idempotency support, make the uncertainty visible and choose a reconciliation or manual-review path.

## Retry by Channel and Failure

Classify failures rather than retrying every exception:

| Result | Action |
| --- | --- |
| invalid recipient or revoked token | record permanent failure; update capability state |
| policy or consent denial | record suppressed; do not retry |
| provider rate limit | delayed retry respecting provider guidance |
| transient network failure before send | bounded retry |
| timeout after request may have sent | unknown; reconcile before retry |
| provider accepted | record accepted; no blind duplicate |
| template or serialization bug | stop class, alert, and dead-letter |

Use a channel-specific retry budget, backoff, and maximum age. Email and webhook latency, provider quotas, and business urgency differ. A single global retry policy can flood a provider or delay urgent security notices behind marketing traffic. Chapter 240 covers backoff; Chapter 284 covers worker leases and dead letters.

## Protect Providers and Recipients

Apply separate limits for tenant, channel, provider, recipient, and message class. A notification storm from one import must not exhaust the provider quota for reservation confirmations. Use queues or worker pools by channel when latency and failure domains differ. Bulkheads protect email provider failure from blocking in-app notifications.

Provider credentials belong in secret management, not templates, messages, or logs. Verify TLS; for outbound webhooks, sign the exact request bytes and support secret rotation, while verifying incoming provider callbacks only where that provider contract requires it. Restrict destinations for webhook channels, and bound connection, response, redirect, and payload sizes. A webhook dispatcher is an outbound network client and must not let recipient-controlled URLs bypass the SSRF and egress policy from Chapter 147.

## Privacy and Retention

Notification content can contain names, reservation times, import results, reset links, or personal data. Minimize what is persisted, encrypt protected records, restrict operator access, and define retention separately for intent, rendered content, provider response, audit record, and analytics. Do not log full bodies, tokens, phone numbers, email addresses, or signed URLs by default.

An audit record should answer who or what caused a notification, which policy and template version applied, which channel was selected, and which provider outcome was observed. It need not copy the entire message into a broadly accessible log.

## Tests That Matter

Test policy and effects separately:

* authorized and unauthorized recipient/tenant combinations;
* consent, unsubscribe, quiet hours, locale, and fallback policy;
* missing template variables and output-size limits;
* context-specific escaping and header injection attempts;
* transaction commit with outbox creation;
* outbox recovery after queue publication failure;
* duplicate delivery and idempotency by intent/channel;
* provider accepted, rejected, rate-limited, transient, and unknown outcomes;
* expired push token and webhook signature failure;
* retry age, backoff, lease expiry, and dead-letter routing;
* provider outage, quota, and channel bulkhead behavior;
* secret redaction and retention cleanup;
* tenant fairness and notification-storm load.

Use provider fakes for deterministic response classes, but use contract or sandbox tests for provider request shape, authentication, idempotency, and response interpretation. A unit test that sees `200 OK` cannot prove that a real provider accepted or delivered the message.

## Observe the Dispatcher

Measure notification intents, suppression reasons, accepted/rejected/unknown outcomes, retry age, outbox age, lease expiry, provider latency, quota responses, dead letters, and channel backlog. Use bounded labels such as channel, message class, provider, template version, outcome, and tenant class. Do not label metrics with recipient addresses, intent IDs, raw URLs, or message bodies.

Alert on outbox age, unknown outcomes, provider errors, quota saturation, suppression spikes, template failures, dead-letter growth, and channel-specific backlog. A high email accepted count does not prove inbox delivery; provider acceptance, bounce, complaint, and downstream delivery signals have different meanings.

## Rollout and Recovery

Deploy intent and outbox schema before enabling a new channel. Canary by tenant or notification class, send to a controlled recipient set, and verify template, consent, provider identity, rate, and duplicate behavior. Version templates and policies; do not silently reinterpret an old intent with a new template when the original content is contractually important.

If a provider is degraded, pause or slow that channel while preserving intents and retry budgets. Do not drop transactional notifications merely because a marketing provider is unavailable. If a provider result is unknown, reconcile by idempotency key or provider lookup before replay. A rollback must preserve outbox records and understand their template/policy versions.

## Common Mistakes

* Calling a provider synchronously inside the business transaction or request.
* Treating a queue message as authorization to notify its recipient.
* Using one idempotency key for all channels of an intent.
* Treating provider acceptance as proof of end-user delivery.
* Retrying an unknown timeout as a definite failure.
* Applying one retry budget to email, SMS, push, and webhooks.
* Rendering user input as template syntax or placing it in headers unsafely.
* Logging message bodies, tokens, recipient addresses, or provider secrets.
* Letting one tenant or channel exhaust every provider quota.
* Sending webhook requests to arbitrary destinations without egress controls.
* Changing template semantics during retry or rollback without versioning.

## Senior Engineer Thinking

The senior question is not “did the email send?” It is “what business intent exists, who is authorized to receive it, which channel policy applies, what exactly does the provider outcome mean, how does a duplicate attempt converge, and what evidence remains when completion is unknown?”

A notification dispatcher is an effect boundary. Keep intent, preference, rendering, provider delivery, retry, privacy, and audit ownership separate. Durable outbox state and provider idempotency cannot guarantee human receipt, but they can keep the system honest about what it knows and what it must reconcile.

## Exercises

1. Design an intent and outbox schema for reservation confirmation and import completion. Include tenant, template, policy, channel, and operation identity.
2. Implement provider adapters that map accepted, rejected, transient, and unknown results into stable delivery states.
3. Define consent, quiet-hour, locale, and fallback policy for transactional and marketing notifications.
4. Draw the recovery path after an SMS timeout where the provider may have accepted the request.
5. Load-test an import notification storm and design channel, tenant, provider, retry, and recipient limits.

## Review Questions

* Why should a business event create an intent rather than call a provider directly?
* What does the outbox transaction solve, and what does it not solve?
* Why are channel contracts different?
* Which authorization and preference checks belong at dispatch time?
* Why must template variables be separate from template selection?
* Why is idempotency scoped by intent and channel?
* How should an unknown provider result affect retries?
* Which privacy data should be retained in intent, delivery, and logs?
* How do bulkheads and quotas protect unrelated notification classes?
* What evidence distinguishes provider acceptance from human delivery?

## Summary

A notification dispatcher turns business intent into controlled external effects. Define semantic intents, preserve them through an outbox, separate channel adapters, enforce recipient authorization and consent, render named versioned templates safely, use intent/channel idempotency, classify retries and unknown outcomes, isolate provider quotas and failures, protect secrets and personal data, test real boundary contracts, observe outbox and delivery age, and roll out templates and providers with reconciliation and recovery.

## References

- [Chapter 136 — Webhooks](../09-http-and-application-development/136-webhooks.md)
- [Chapter 147 — SSRF](../10-security/147-ssrf.md)
- [Chapter 155 — Secrets](../10-security/155-secrets.md)
- [Chapter 240 — Backoff](../16-distributed-systems/240-backoff.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 242 — Idempotency](../16-distributed-systems/242-idempotency.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 248 — Bulkheads](../16-distributed-systems/248-bulkheads.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 280 — Tennis Reservation Service](./280-tennis-reservation-service.md)
- [Chapter 283 — File Importer](./283-file-importer.md)
- [Chapter 284 — Queue Worker](./284-queue-worker.md)
