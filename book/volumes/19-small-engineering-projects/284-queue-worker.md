---
book: The Complete Modern PHP Engineering Book
volume: 19
volume_title: SMALL ENGINEERING PROJECTS
chapter: 284
title: Queue Worker
slug: queue-worker
status: complete
summary: ../../_ai/chapter-summaries/284-queue-worker-summary.md
---

# Chapter 284 — Queue Worker

## Why This Matters

A queue worker turns a durable message into work performed later. The loop can look trivial—receive, handle, acknowledge—but the process can die after the side effect and before the acknowledgement. The queue then delivers the message again. A worker that assumes exactly-once execution will eventually send a duplicate notification, apply a duplicate import batch, or lose a job during deployment.

This project builds a worker for the file-import jobs from Chapter 283. It makes delivery, lease, acknowledgement, retry, idempotency, dead-letter, shutdown, and observability contracts explicit. The goal is not to pretend that a queue removes failure; it is to make failure states durable and recoverable.

## Define the Queue Contract

Write down what the broker and worker promise:

~~~text
delivery: at-least-once
message: opaque import job ID, operation ID, policy version
visibility: message is hidden during a renewable lease
ack: only after durable batch progress or terminal outcome commits
retry: transient failures use bounded attempts and backoff
dead letter: poison/permanent failures are retained for authorized review
ordering: no global ordering; per-import ordering is required
shutdown: stop claiming, finish or release current lease safely
effect: handler is idempotent or uses a durable operation identity
~~~

The queue is a transport boundary, not an authorization boundary. The message must identify work, but the worker must load the import and verify tenant, state, policy, and permissions before changing data. Do not put raw file rows, credentials, or unrestricted user input in a queue payload when an opaque reference is enough.

## A Message Has More Than a Payload

Useful envelope fields include:

| Field | Purpose |
| --- | --- |
| `message_id` | stable published-message identity |
| `delivery_id` | per-delivery lease/ack identity |
| `operation_id` | logical work identity across retries |
| `job_id` | import or business object reference |
| `attempt` | bounded retry accounting |
| `available_at` | delayed delivery time |
| `schema_version` | payload compatibility |
| `trace_id` | bounded diagnostic correlation |
| `created_at` | age and retention |

The broker may assign a new delivery ID each time it redelivers. Keep that per-delivery identity separate from the stable `message_id`, and do not use either changing delivery metadata as the business idempotency key. A crash after a database commit can produce a new delivery for the same logical operation.

## Build the Smallest Worker Loop

Separate broker mechanics from handler decisions. A worker should claim one message, create a bounded lease, invoke a handler, classify the outcome, and acknowledge or reschedule deliberately.

~~~php
<?php

declare(strict_types=1);

enum WorkResult: string
{
    case Completed = 'completed';
    case Retry = 'retry';
    case DeadLetter = 'dead_letter';
    case Unknown = 'unknown';
}

final readonly class WorkMessage
{
    public function __construct(
        public string $messageId,
        public string $deliveryId,
        public string $operationId,
        public string $jobId,
        public int $attempt,
    ) {
        if ($messageId === '' || $deliveryId === '' || $operationId === '' || $jobId === '' || $attempt < 1) {
            throw new InvalidArgumentException('Work message is invalid');
        }
    }
}

interface WorkHandler
{
    public function handle(WorkMessage $message): WorkResult;
}

final class PermanentWorkFailure extends RuntimeException
{
}

final class UnknownWorkOutcome extends RuntimeException
{
}

function classifyFailure(Throwable $failure, int $attempt, int $maxAttempts): WorkResult
{
    if ($failure instanceof UnknownWorkOutcome) {
        return WorkResult::Unknown;
    }

    if ($failure instanceof PermanentWorkFailure || $attempt >= $maxAttempts) {
        return WorkResult::DeadLetter;
    }

    return WorkResult::Retry;
}
~~~

This is a decision example, not a broker client. `PermanentWorkFailure` and `UnknownWorkOutcome` stand for application-specific failure types. It assumes the handler classifies failures deliberately. Catching every `Throwable` and retrying it forever turns programming bugs and poison input into an endless redelivery loop. Catching every error and dead-lettering it immediately can lose recoverable work.

## Lease and Acknowledge in the Right Order

A visibility timeout or lease suppresses normal redelivery for a period; it is not a correctness lock and does not guarantee that only one worker processes the message. It also does not guarantee that a handler finishes before the lease expires. Long jobs renew the lease, and the renewal itself can fail.

The safe sequence is:

~~~text
claim message + lease
        ↓
load and authorize job
        ↓
perform one idempotent batch
        ↓
commit durable progress/effect
        ↓
acknowledge message
~~~

If the process dies after the commit and before the ack, redelivery is expected. The handler must recognize the durable batch identity or operation ID and converge without repeating the effect. If the lease expires while the first worker still runs, a second worker may start. Lease ownership is a coordination hint; the database invariant or effect idempotency must provide the safety property.

Set the lease longer than normal batch work, renew before the deadline, and include a safety margin for network and scheduler delay. A worker must stop extending a lease after it has lost ownership. Otherwise a stalled or partitioned process can hold work indefinitely.

## Make Import Handling Idempotent

For Chapter 283, use `(import_id, batch_start, batch_end)` or a durable batch number as the logical work identity. The batch writer can:

1. claim or verify the batch state;
2. read the immutable source object from the pinned cursor;
3. write rows using tenant-scoped unique keys and deterministic upserts;
4. record row outcomes and counters;
5. commit the batch and progress together;
6. acknowledge the message.

If a message is redelivered after step 5, the worker sees the committed batch and returns completed without applying it again. If a database timeout makes commit status unknown, query the batch identity before retrying. If durable state still cannot establish the outcome, return `WorkResult::Unknown` and retain the message for reconciliation; do not assume a client exception means rollback.

An idempotency key does not solve non-idempotent downstream effects automatically. If a successful batch emits a notification, use an outbox record with a stable event identity. If it calls a provider, use the provider’s idempotency contract or reconcile unknown completion.

## Classify Retryable Failures

Failure classification belongs to the operation, not just the exception class:

| Failure | Worker action |
| --- | --- |
| malformed import row | record row error; continue or terminally complete |
| missing source object | fail import and preserve evidence |
| database deadlock | roll back and retry with bounded backoff |
| transient storage timeout | retry if lease and budget permit |
| invalid message schema | dead-letter and alert |
| programming error | stop or dead-letter under operator policy |
| provider timeout | mark effect unknown and reconcile |
| cancellation | stop at a safe batch boundary |

Use exponential backoff with jitter and a maximum attempt count. A retry budget should account for queue age, operation deadline, and downstream capacity. Retrying faster than a dependency can recover is load amplification; Chapter 240 covers backoff and Chapter 246 covers backpressure.

Do not sleep inside a worker while holding a database transaction or a lease that cannot be renewed. Prefer delayed delivery or release the lease before a long backoff, according to broker semantics.

## Dead Letters Are a Workflow

A dead-letter queue is not a trash can. Preserve the original message, failure category, attempt history, timestamps, and safe diagnostic context. Restrict access because the envelope may identify tenant work. Operators need actions:

* inspect without executing;
* repair source data or configuration;
* replay with explicit authorization and a new audit record;
* discard when policy permits;
* quarantine a poison message or version;
* verify resulting state after replay.

Replay must use the original logical operation identity or a deliberately new repair identity. Replaying a message does not prove that its prior attempt had no effect. Query durable batch and outbox evidence before authorizing replay.

## Shutdown and Long-Lived PHP Processes

A PHP worker is a long-lived process, unlike a normal FPM request. It accumulates memory, static state, file descriptors, database connections, stale configuration, and object graphs. Bound its lifetime and recycle after a controlled number of messages, elapsed time, memory watermark, or failure count.

On deploy or termination:

1. mark the worker not ready to claim new messages;
2. stop polling or pause claims;
3. finish a bounded current unit or release its lease;
4. flush logs and metrics;
5. close resources and exit before the grace deadline.

If the process is killed during a handler, the broker should redeliver after the lease, and the handler should use idempotent progress. Do not acknowledge before shutdown merely to make the queue appear empty. A graceful drain is useful only if the lease, timeout, and effect contract support it.

## Fairness and Backpressure

One tenant can fill a queue and starve everyone else. Choose a policy: separate queues, weighted scheduling, per-tenant concurrency, a fair dispatcher, or admission limits at submission. Measure and document the trade-off between throughput and fairness.

Bound in-flight messages, batch sizes, memory, database connections, provider calls, and retry volume. A queue depth of zero does not prove health if workers are blocked or messages are disappearing. Track age of the oldest message, processing latency, lease expirations, and terminal outcomes.

## Tests That Matter

Test pure decisions and real process boundaries:

* valid and invalid message envelopes;
* completed, retryable, permanent, and unknown failures;
* ack only after durable commit;
* redelivery after crash before ack;
* lease renewal, expiry, and lost ownership;
* concurrent workers claiming the same message;
* idempotent batch replay after committed progress;
* ambiguous database commit and provider outcome;
* delayed retry, maximum attempts, and dead-letter routing;
* malformed payload and poison-message quarantine;
* signal-driven graceful shutdown and forced termination;
* worker memory, descriptor, connection, and configuration recycling;
* queue fairness, hot tenant, backpressure, and recovery ramp.

Use a real broker or faithful test container for visibility, ack, redelivery, ordering, and delay semantics. A mocked queue cannot prove those behaviors. Use a real database for commit/ack ordering and concurrent batch identity. Run failure injection around every boundary where an effect may commit before the process reports success.

## Observe the Worker

Record bounded fields: queue name, message type, schema version, outcome, failure category, attempt, age bucket, lease renewals, handler duration, batch size, and worker revision. Keep message IDs, tenant IDs, file names, and arbitrary payload values out of unbounded metric labels. Logs may include a short-lived correlation ID and an authorization-safe reference.

Alert on oldest-message age, lease-expiry rate, retry growth, dead-letter growth, handler error ratio, memory high-water mark, worker restart rate, database lock time, and outbox lag. Distinguish no messages from a broken poller, and distinguish a normal business rejection from a poison-message storm.

## Rollout and Recovery

Deploy a compatible worker before publishing messages that require it. During mixed deployment, old workers must understand the envelope and either safely ignore or dead-letter newer schema versions. Prefer additive fields and explicit schema versions; do not silently reinterpret a field.

Canary one worker or message class, watch duplicate effects and lease behavior, then expand. If a rollout fails, pause publishing or drain the affected queue, preserve messages, and choose between code rollback, replay under a compatible handler, or forward repair. Do not delete the queue to “clear” the incident.

## Common Mistakes

* Assuming queue delivery is exactly once because acknowledgements exist.
* Acknowledging before the database transaction or effect evidence is durable.
* Using the changing delivery ID as the idempotency identity.
* Treating a lease as a permanent lock or renewing after losing ownership.
* Retrying every exception forever or dead-lettering every exception immediately.
* Sleeping through backoff while holding locks or a non-renewable lease.
* Treating the dead-letter queue as an unreviewed trash can.
* Replaying a message without checking whether the earlier attempt committed.
* Letting long-lived PHP workers retain unbounded memory or stale configuration.
* Draining by acknowledging work that was never completed.
* Measuring queue depth without message age, lease expiry, or terminal outcomes.
* Deploying a new message schema before old workers have a safe behavior.

## Senior Engineer Thinking

The senior question is not “did the worker process the message?” It is “what delivery guarantee exists, which durable evidence makes a redelivery safe, when does the lease stop protecting us, how are failures classified, and how can an operator replay or repair work without duplicating an effect?”

A queue worker is a small distributed system. Keep the broker contract, handler contract, database transaction, external effect, process lifecycle, and recovery workflow visible. At-least-once delivery is workable when operations converge, progress is durable, retries are bounded, and dead letters remain an accountable workflow.

## Exercises

1. Draw the crash points before commit, after commit, before acknowledgement, and after acknowledgement. State the recovery behavior for each.
2. Design a message envelope and schema-version policy for the Chapter 283 importer during a mixed deployment.
3. Implement a durable batch identity and prove that redelivery after commit does not duplicate rows or notifications.
4. Choose retry and dead-letter rules for six failures, including a malformed message, deadlock, provider timeout, and programming error.
5. Design a graceful shutdown and worker-recycling policy with lease, memory, descriptor, and grace-period limits.

## Review Questions

* Why is at-least-once delivery often safer than pretending to have exactly-once execution?
* What must be durable before a message is acknowledged?
* Why can a lease expire while the original worker is still running?
* Which identity should survive message redelivery?
* How do retryable, permanent, and unknown failures differ?
* What operational actions make a dead-letter queue useful?
* Why do long-lived PHP workers need recycling and graceful drain?
* Which measurements reveal queue health beyond depth?
* How should old and new workers coexist during schema rollout?
* When is replay unsafe without reconciliation?

## Summary

A queue worker is a durable failure-handling loop, not just a polling script. Define delivery and message contracts, claim work with renewable leases, acknowledge only after durable progress or terminal evidence, make handlers idempotent across redelivery, classify failures with bounded retries and backoff, operate dead letters as an authorized workflow, handle signals and long-lived PHP resources, enforce fairness and backpressure, test real broker/database boundaries, observe age and lease health, and roll out schemas compatibly with replay and recovery plans.

## References

- [Chapter 239 — Retries](../16-distributed-systems/239-retries.md)
- [Chapter 240 — Backoff](../16-distributed-systems/240-backoff.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 242 — Idempotency](../16-distributed-systems/242-idempotency.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 245 — Dead-Letter Queues](../16-distributed-systems/245-dead-letter-queues.md)
- [Chapter 246 — Backpressure](../16-distributed-systems/246-backpressure.md)
- [Chapter 254 — Linux for PHP Engineers](../17-production-engineering/254-linux-for-php-engineers.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 283 — File Importer](./283-file-importer.md)
