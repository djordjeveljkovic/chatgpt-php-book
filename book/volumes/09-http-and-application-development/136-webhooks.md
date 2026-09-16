---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 136
title: Webhooks
slug: webhooks
status: complete
summary: ../../_ai/chapter-summaries/136-webhooks-summary.md
---

# Chapter 136 — Webhooks

## Why This Matters

A webhook is an outbound HTTP notification about a state change. It lets one system react without polling, but it also turns a local transaction into a distributed operation: the receiver can be slow, unavailable, duplicated, or compromised. A reliable webhook design makes delivery, authentication, replay, ordering, and recovery explicit.

The event is a message, not a remote procedure call. The sender should be able to retry it safely, and the receiver should be able to acknowledge it before doing slow work. A successful HTTP response means that the receiver accepted the message; it does not prove that the business action has finished.

## Event Contract

Give every event a stable identifier, type, creation time, and version. Include the aggregate identifier and enough data for the consumer to make a decision, while avoiding secrets and accidental database internals.

```json
{
  "id": "evt_01J8Z6K4J7Q2",
  "type": "invoice.paid",
  "version": 1,
  "occurred_at": "2026-09-16T12:34:56Z",
  "data": {
    "invoice_id": "inv_204",
    "account_id": "acct_9",
    "amount_minor": 12500,
    "currency": "EUR"
  }
}
```

Document whether `data` is a snapshot or only an identifier. A snapshot makes consumers less dependent on a follow-up API call; an identifier keeps payloads smaller but requires the resource to remain available. Treat an event type and version as a compatibility contract. Add fields compatibly and retain old versions long enough for consumers to migrate.

## Recording and Delivering Events

Do not make a customer request wait for an arbitrary third-party endpoint. Commit the domain change and an outbox row in one database transaction, then let a worker deliver pending rows. The outbox closes the gap between “the invoice was committed” and “the process crashed before enqueueing a notification.” A unique event ID also gives every delivery attempt the same deduplication key.

```php
<?php

declare(strict_types=1);

final class EventOutbox
{
    public function record(PDO $db, string $eventId, string $type, array $payload): void
    {
        $statement = $db->prepare(
            'INSERT INTO webhook_events (id, type, payload, status, next_attempt_at)
             VALUES (:id, :type, :payload, :status, CURRENT_TIMESTAMP)'
        );
        $statement->execute([
            'id' => $eventId,
            'type' => $type,
            'payload' => json_encode($payload, JSON_THROW_ON_ERROR),
            'status' => 'pending',
        ]);
    }
}
```

A worker claims a bounded batch, sends each event with a strict connect and total timeout, and records the result. Claiming must be safe when workers run concurrently: use a database lock/lease or a queue with visibility timeout. Never hold a database transaction open while waiting on the network.

## Signing and Verifying

Sign the exact bytes sent over the wire. A common contract combines a timestamp and body, such as `timestamp + '.' + rawBody`, and sends a key identifier plus an HMAC-SHA-256 digest. The receiver must read the raw request body before parsing JSON; re-encoding parsed data can change whitespace, escaping, or key order.

```php
<?php

declare(strict_types=1);

function verifyWebhook(string $rawBody, string $header, string $secret, int $now): bool
{
    // Header format: t=1710000000,v1=hex-digest
    $parts = [];
    foreach (explode(',', $header) as $part) {
        [$key, $value] = array_pad(explode('=', trim($part), 2), 2, '');
        $parts[$key] = $value;
    }

    $timestamp = filter_var($parts['t'] ?? null, FILTER_VALIDATE_INT);
    $signature = $parts['v1'] ?? '';
    if ($timestamp === false || $signature === '' || abs($now - $timestamp) > 300) {
        return false;
    }

    $expected = hash_hmac('sha256', $timestamp . '.' . $rawBody, $secret);
    return hash_equals($expected, $signature);
}
```

Use constant-time comparison and reject timestamps outside a small, documented tolerance. Timestamp checking limits replay, but it does not identify a message as already processed. Store event IDs (or a provider's delivery IDs) in a unique table and make the state transition idempotent. Rotate secrets with a key ID and accept the previous key only during a planned migration window.

## Receiver Processing

The receiver should validate the signature, content type, schema, and event type before enqueueing. It should persist the delivery ID before acknowledging it, using a unique constraint. If the ID already exists, return the same success response after confirming the prior outcome. Process the queue with a transaction that records the business effect and marks the inbox row handled.

```php
<?php

declare(strict_types=1);

function acceptWebhook(PDO $db, string $rawBody, string $deliveryId): int
{
    try {
        $event = json_decode($rawBody, true, 512, JSON_THROW_ON_ERROR);
    } catch (JsonException) {
        return 400;
    }
    if (!is_array($event) || !isset($event['id'], $event['type'])) {
        return 400;
    }

    $db->beginTransaction();
    try {
        $insert = $db->prepare(
            'INSERT INTO webhook_inbox (delivery_id, event_id, payload, status)
             VALUES (?, ?, ?, ?)
             ON CONFLICT (delivery_id) DO NOTHING'
        );
        $insert->execute([$deliveryId, $event['id'], $rawBody, 'pending']);
        $db->commit();
    } catch (Throwable $exception) {
        $db->rollBack();
        throw $exception;
    }

    return 202; // A worker will perform the domain action.
}
```

The `ON CONFLICT` syntax is PostgreSQL-specific; use the equivalent unique insert behavior for the chosen database. A duplicate delivery is normal, not an incident. If processing is permanently invalid, retain the payload and move it to a dead-letter state rather than retrying forever.

## Retries, Ordering, and Responses

Retry transient network errors, timeouts, and usually 5xx responses with exponential backoff and jitter. Respect a receiver's `Retry-After` when it is within policy. Do not retry most 4xx responses until the endpoint contract says the condition is temporary. Cap attempts and retain the final failure for operator replay.

Events can arrive out of order. Include a per-aggregate sequence number when ordering matters, and make the consumer detect gaps or discard stale versions. Global ordering is expensive and often unnecessary. A retry can also race with a later event, so consumers should compare versions or use domain timestamps under a transaction.

Return `2xx` only when the message has been durably accepted. `202 Accepted` is appropriate for queued work; `204 No Content` is fine when there is no response body. Return `400` for an invalid contract and `401`/`403` for authentication policy failures. Keep error bodies free of secrets and include a correlation ID for support.

## Security and Operations

Outbound delivery needs an allowlist or controlled endpoint registration. Validate URLs at registration, resolve and re-check addresses when connecting, block private and link-local destinations where server-side request forgery is a concern, and restrict redirects. Use TLS verification and limit response size. A webhook URL is an integration credential in practice; protect it and redact it from logs.

Record event ID, endpoint ID, attempt number, status code, latency, response classification, and correlation ID. Metrics should show queue age, delivery success rate, retry count, and dead-letter count. Preserve signed payloads only under a retention policy because they may contain personal data.

## Testing

Test exact-byte signature verification, stale timestamps, key rotation, malformed JSON, duplicate delivery IDs, retry classification, `Retry-After`, timeout behavior, out-of-order versions, and dead-letter replay. Use a fake clock and fake HTTP client so backoff tests are deterministic. A contract test should verify that a consumer can parse each supported event version and that a sender does not publish undocumented fields as required fields.

## Common Mistakes

- Performing webhook delivery inside the request transaction.
- Signing parsed or re-encoded JSON instead of the raw body.
- Treating one successful delivery as proof that all consumers processed the event.
- Omitting a unique inbox/outbox key and applying a business action twice.
- Retrying permanent validation failures forever.
- Logging secrets, full authorization headers, or sensitive payloads.

## Senior Engineer Thinking

Design webhooks as durable messages crossing an unreliable boundary. The outbox, signature, inbox, idempotent consumer, bounded retry policy, and observable dead-letter path form one system. Each part answers a different failure: a crash before send, tampering, duplicate delivery, transient outage, and an event that cannot be processed.

## Exercises

1. Design an outbox and inbox schema with uniqueness, attempt history, lease expiry, and dead-letter state.
2. Implement key rotation so a receiver accepts both current and previous secrets for a bounded interval.
3. Create a retry policy table for timeouts, `429`, `400`, `401`, and `503`, including operator replay behavior.

## Review Questions

1. Why must the domain transaction and outbox insert commit together?
2. What problem does a timestamped HMAC solve, and what problem does an inbox uniqueness constraint solve?
3. When is `202 Accepted` a truthful webhook response?
4. How should a consumer handle an event that arrives after a newer version?

## Summary

Webhooks are distributed messages delivered over HTTP. Record them durably with the domain change, sign the exact body, verify timestamps and schemas, acknowledge only durable acceptance, make consumers idempotent, retry transient failures with limits, and observe the queue and dead-letter path.

## References

- [RFC 2104: HMAC](https://www.rfc-editor.org/rfc/rfc2104)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [PHP `hash_hmac` documentation](https://www.php.net/manual/en/function.hash-hmac.php)
- [OWASP: Server-Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
