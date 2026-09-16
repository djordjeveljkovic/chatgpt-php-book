---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 245
title: Dead-Letter Queues
slug: dead-letter-queues
status: complete
summary: ../../_ai/chapter-summaries/245-dead-letter-queues-summary.md
---

# Chapter 245 — Dead-Letter Queues

## Why This Matters

A dead-letter queue (DLQ) is a failure path for messages that cannot be processed safely within the normal delivery policy. It protects healthy work from a poison message, preserves evidence for investigation, and gives operators a controlled replay or discard decision.

A DLQ is not a trash can and not proof that the system handled the failure. A message moved there is still unfinished business. Define ownership, retention, alerting, privacy, replay authorization, and the conditions under which it may be repaired or permanently discarded.

## Why Messages Fail

Failures usually fall into different classes:

* malformed encoding or unsupported schema;
* invalid domain data that cannot succeed without correction;
* authorization or tenant mismatch;
* transient dependency outage after the retry budget;
* handler bug or incompatible deployment;
* duplicate or stale command;
* resource exhaustion or message too large.

The class determines the next action. A transient provider outage may be delayed and retried. A bad schema needs a decoder fix or a producer correction. A duplicate may be safely marked complete. An authorization failure should not be replayed by an operator who lacks the original authority.

## Failure Envelope

Preserve the original identity and enough safe context to diagnose:

```text
original_message_id
operation_id
event_type and schema version
first_seen_at and failed_at
attempt_count
failure category and bounded reason
consumer and release version
safe payload reference
```

Avoid copying secrets or full personal-data payloads into an indefinitely retained failure store. A reference to encrypted durable storage may be safer, but the reference itself needs access control and retention.

## Normal Flow and DLQ

```text
main queue → retry delay → main queue
     ↓ permanent or exhausted failure
     └──────────────→ dead-letter queue → inspect → repair/replay/discard
```

The transition should be observable and durable. Count moves, record the final failure category, and retain the original operation identity. A DLQ that silently accepts messages can hide a growing customer-impacting backlog.

## A Failure Record

Keep the failure path typed and bounded:

~~~php
<?php

declare(strict_types=1);

enum FailureCategory: string
{
    case InvalidMessage = 'invalid_message';
    case PermanentDomainFailure = 'permanent_domain_failure';
    case RetryExhausted = 'retry_exhausted';
    case IncompatibleSchema = 'incompatible_schema';
}

final readonly class DeadLetter
{
    /** @param array<string, scalar|null> $metadata */
    public function __construct(
        public string $messageId,
        public string $operationId,
        public FailureCategory $category,
        public int $attempts,
        public array $metadata,
    ) {
        if ($messageId === '' || $operationId === '' || $attempts < 1) {
            throw new InvalidArgumentException('Invalid dead-letter record');
        }
    }
}

interface DeadLetterStore
{
    public function put(DeadLetter $letter): void;
}

function quarantine(
    DeadLetterStore $store,
    string $messageId,
    string $operationId,
    FailureCategory $category,
    int $attempts,
): void {
    $store->put(new DeadLetter(
        $messageId,
        $operationId,
        $category,
        $attempts,
        ['handler' => 'orders'],
    ));
}
~~~

The example stores bounded metadata rather than an arbitrary exception object. A real failure record may include a safe payload reference, release, timestamps, and a redacted reason. Never make replay depend on unserializing an untrusted exception.

## Replay Is a Write Operation

Replay can create an external effect, so it needs authorization, rate limits, audit, and idempotency. An operator should be able to inspect a message without being able to replay payments or change another tenant's data by default.

Before replay:

1. validate the current schema and required fields;
2. confirm the original operation and tenant are still valid;
3. determine whether the failure was fixed;
4. preserve the original message and operation identities;
5. choose a bounded destination, rate, and attempt budget;
6. record who requested the replay and its outcome.

Replay to the main queue may be correct for a transient outage. A repaired payload may need a new message envelope while retaining the original operation ID. A command whose business precondition has expired should be reconciled or discarded under domain policy, not blindly replayed.

## Poison Messages and Code Bugs

A poison message fails repeatedly for the same deterministic reason. Requeueing it immediately consumes workers and can starve healthy work. A high DLQ rate after a deployment may indicate a code or schema incompatibility; stop or roll back the producer/consumer change before replaying thousands of records.

Fixing the code does not automatically make all old messages safe. Test a sample, replay at a controlled rate, monitor outcomes, and stop if the failure pattern returns. Keep a quarantine copy until the repair is verified and retention policy permits removal.

## Retention and Privacy

Failure data can contain customer and security information. Encrypt it where required, restrict access, redact logs, set retention by business and regulatory need, and delete expired records through an auditable process. Do not let a debugging convenience create a permanent copy of credentials or payment data.

The DLQ itself needs capacity monitoring. A full DLQ can cause the original failure path to block, reject, or lose messages depending on transport behavior. Define what happens when both the main queue and failure store are unavailable.

## What PHP Does

A PHP worker may throw an exception, release a message, or move it to a failure transport. The exception object is local process state; it is not a durable diagnostic by itself. Serialize a safe, structured failure category and context before the worker exits.

Reset request and tenant context after handling a failure. A long-running worker that retains a large payload or trace object while processing the next message can turn a DLQ incident into a memory incident. See [Chapter 234 — Queue Performance](../../volumes/15-performance/234-queue-performance.md).

## Testing

Test invalid messages, retry exhaustion, schema mismatch, duplicate messages, authorization failure, DLQ unavailability, replay authorization, repaired payloads, current-version validation, rate limits, retention, and graceful shutdown during a move. Verify that the original effect is not repeated unexpectedly.

Run a replay drill with synthetic data. Measure how quickly operators can identify the cause, fix it, replay a bounded sample, stop the replay, and prove the business result.

## Security

Protect DLQ contents and management APIs as production data. Require strong authentication, tenant-aware authorization, audit records, and separate permissions for view, edit, replay, and discard. Treat payload repair as untrusted input and validate it again. Do not expose stack traces, credentials, or internal topology in public errors.

## Common Mistakes

* Moving all failures to a DLQ without classifying them.
* Treating DLQ depth as an operationally harmless metric.
* Replaying every message after a deployment without a sample or rate limit.
* Letting operators replay another tenant's command.
* Storing raw exceptions and secrets indefinitely.
* Dropping original message and operation identities during repair.
* Forgetting that a full DLQ is itself a failure mode.
* Marking a message discarded without an audit or business decision.

## Senior Engineer Thinking

Ask what decision the DLQ enables: retry, repair, reconcile, or discard. Then make that decision safe, authorized, observable, and reversible where possible. A dead-letter path is part of the normal system design because every durable queue eventually encounters malformed input, changing code, and exhausted dependencies.

## Exercises

1. Define failure categories and actions for a notification, payment, and search-indexing queue.
2. Design a dead-letter envelope that preserves identity while minimizing sensitive data.
3. Run a replay drill for 10,000 messages and define sampling, rate, stop conditions, and audit evidence.
4. Model what happens when the main queue and DLQ are both unavailable.

## Review Questions

* What distinguishes a DLQ from a trash can?
* Which failure classes should be repaired, delayed, reconciled, or discarded?
* Why is replay a write operation?
* Which identities must survive a replay?
* How can a deployment bug make a DLQ grow quickly?
* Which permissions should be separate for DLQ operations?

## Summary

A dead-letter queue isolates messages that cannot be processed safely under normal policy while preserving evidence for an authorized decision. Classify failures, retain bounded and protected context, preserve message and operation identity, validate and rate-limit replay, monitor both main and failure stores, and test repair, reconciliation, discard, and outage behavior.

## References

- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/)
- [Enterprise Integration Patterns: Dead Letter Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DeadLetterChannel.html)
- [Chapter 243 — Message Delivery](./243-message-delivery.md)
- [Chapter 244 — Queues](./244-queues.md)
