---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 260
title: Logging
slug: logging
status: complete
summary: ../../_ai/chapter-summaries/260-logging-summary.md
---

# Chapter 260 — Logging

## Why This Matters

Logs are a record of what an application observed and decided at a point in time. They help explain failures, trace a request across boundaries, and support audit and debugging. They are not a substitute for metrics or traces, and they become a liability when they leak secrets, contain unbounded data, or cannot be correlated with the operation that failed.

Design logs for a question: which request failed, which release handled it, what decision was made, and what dependency or resource was involved? Record enough context to answer the question without copying the entire request.

## Structured Events

Prefer structured fields over prose that operators must parse:

```json
{
  "timestamp": "2026-09-16T12:00:00Z",
  "level": "warning",
  "event": "provider_timeout",
  "request_id": "req-123",
  "operation": "checkout",
  "provider": "billing",
  "elapsed_ms": 740,
  "release": "2026-09-16.2"
}
```

Use stable event names and bounded field values. Do not use raw user IDs, full URLs, exception messages, or arbitrary payload fields as metric-like labels. A log backend may index every field and create cost or query-cardinality problems.

## Levels and Meaning

Levels need an operational policy:

* **debug:** detailed development or focused diagnosis;
* **info:** important lifecycle and business milestones;
* **notice/warning:** unusual or degraded behavior that may need attention;
* **error:** an operation failed and evidence should be investigated;
* **critical:** a broad or immediate operational impact.

Do not log every successful request at error level to make an alert work. Alert on metrics and use logs as evidence. Avoid logging a stack trace for expected client validation failures unless a bounded audit requires it.

## Correlation

Propagate a request or trace ID through Nginx, PHP-FPM, application code, database spans, HTTP calls, and queue messages. A correlation ID identifies a diagnostic path; it is not a secret or authorization token. Generate a safe ID when the caller does not provide one, and validate or replace unsafe caller input.

For asynchronous work, include both the originating request ID and durable operation or message ID where useful. The request may finish before the job fails. Use release, worker, host or container, and tenant class dimensions to locate the failure without logging personal data.

## A Small Logger Boundary

Keep structured logging behind a narrow adapter and redact known sensitive keys:

~~~php
<?php

declare(strict_types=1);

interface Logger
{
    /** @param array<string, scalar|null> $context */
    public function warning(string $event, array $context = []): void;
}

/** @param array<string, scalar|null> $context @return array<string, scalar|null> */
function redact(array $context): array
{
    $sensitive = ['authorization', 'cookie', 'password', 'token', 'secret'];

    foreach ($context as $key => $value) {
        if (in_array(strtolower($key), $sensitive, true)) {
            $context[$key] = '[REDACTED]';
        }
    }

    return $context;
}

function logProviderTimeout(Logger $logger, string $requestId, int $elapsedMs): void
{
    $logger->warning('provider_timeout', redact([
        'request_id' => $requestId,
        'provider' => 'billing',
        'elapsed_ms' => $elapsedMs,
    ]));
}
~~~

The example handles a small flat context. Real applications need nested redaction, value-length limits, allowed-field schemas, and a policy for exception objects. Redaction is a defense in depth; do not put secrets into the context and hope a formatter catches every representation.

## Request and Business Logs

Log technical events such as timeouts, retries, worker shutdown, and configuration failure. Log business milestones such as order accepted or payment reconciled when the domain requires evidence. Keep audit records separate when they need stronger durability, immutability, retention, or access controls than operational logs.

Do not log a payment card number, password, session cookie, access token, or complete uploaded document. Be cautious with email addresses, addresses, search terms, and free-form user content. Hashing or truncating does not automatically make personal data non-sensitive.

## Exceptions

An exception is an object with a message, code, trace, previous exception, and sometimes arguments or response bodies. Serialize only safe fields. Log the causal category, operation, release, and a bounded diagnostic reference. The user-facing error should not be the raw log event.

Expected failures should have stable categories: validation, authorization, unavailable, timeout, conflict, duplicate, and bug. This makes dashboards and alerts useful and prevents operators from searching prose variations.

## Volume and Cost

High-volume logs consume CPU, disk, network, storage, indexing, and retention budget. Sample repetitive success events, aggregate them into metrics, and retain detailed events for errors and slow operations. Sampling must not remove required audit evidence or make rare tenant impact invisible.

Use bounded message and field lengths. A malicious request should not create a multi-megabyte log line. Apply rate limits to repeated errors while preserving counts and exemplars.

## Collection and Delivery

A process may write to standard output, a file, a socket, or a logging agent. Decide what happens when the destination is slow or unavailable. Blocking on logging can increase request latency; dropping all logs can remove incident evidence. Keep audit records on a durability path appropriate to their requirement.

Use a common timestamp format, synchronized host time where possible, release identity, and source role. Log rotation, disk capacity, permissions, transport encryption, and retention are production concerns.

## PHP Runtime and Workers

Configure PHP errors and application logs deliberately. In long-running workers, reset request context and do not retain a logger buffer or trace span across messages. Flush logs before graceful shutdown within a bounded policy. A process crash may lose buffered output; important state belongs in a durable store.

Correlate PHP logs with Nginx access logs, FPM slow logs, database query records, queue messages, and provider traces. Use the same request or operation identity without copying sensitive payloads.

## Security and Privacy

Treat logs as sensitive production data. Restrict readers, encrypt transport and storage, audit access, define retention and deletion, and separate tenant access. A log search interface can become a data-exfiltration tool if it allows arbitrary queries or raw payload display.

Redact before the event leaves the process when possible. Protect crash reports and support bundles too; they often contain the same context as logs.

## Testing

Test stable event names, required fields, missing correlation IDs, exception mapping, nested and oversized values, redaction, sampling, rate limiting, worker shutdown, log-destination failure, rotation, and tenant access. Assert that synthetic secrets never occur in emitted output.

Use a schema or consumer contract for logs that dashboards depend on. Changing a field from milliseconds to seconds without a version or migration can invalidate alerts.

## Common Mistakes

* Logging complete requests, headers, exceptions, or URLs by default.
* Using prose-only logs with no stable operation or event name.
* Treating correlation IDs as authorization.
* Alerting on log volume instead of failure and latency metrics.
* Letting unbounded user input become a log field or line.
* Blocking requests on a slow logging destination.
* Sharing audit and debug retention policies.
* Forgetting queue operation IDs and worker identity.

## Senior Engineer Thinking

Ask which question the log answers, who may read it, how long it survives, and what happens if logging fails. Good logs reduce time to recovery while minimizing copied data. They complement metrics, traces, audit records, and durable business state.

## Exercises

1. Design structured events for a request timeout, queue retry, deployment failure, and payment reconciliation.
2. Build a redaction test with nested secrets, long values, headers, and exception context.
3. Define sampling and retention for debug, operational, and audit events.
4. Correlate one request across Nginx, PHP-FPM, PHP application, database, and queue logs.

## Review Questions

* What question should a useful log answer?
* Why are logs not a replacement for metrics and traces?
* Which identifiers help asynchronous debugging?
* Why should audit records be separated from ordinary logs?
* How can logging harm latency and availability?
* Which data must never be logged by default?

## Summary

Logging is structured operational evidence with privacy, cost, and availability constraints. Use stable events, bounded fields, safe correlation and operation IDs, explicit failure categories, redaction, sampling, retention, and access control. Keep audit and business durability separate, reset worker context, and test emitted output—including the absence of synthetic secrets.

## References

- [OWASP: Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [OpenTelemetry: Logs](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Chapter 254 — Linux for PHP Engineers](./254-linux-for-php-engineers.md)
- [Chapter 262 — Tracing](./262-tracing.md)
