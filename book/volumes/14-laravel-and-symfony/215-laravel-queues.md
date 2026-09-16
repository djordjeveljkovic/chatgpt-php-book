---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 215
title: Laravel Queues
slug: laravel-queues
status: complete
summary: ../../_ai/chapter-summaries/215-laravel-queues-summary.md
---

# Chapter 215 — Laravel Queues

## Why This Matters

Laravel queues move slow or retryable work out of an HTTP request. A queued job can send mail, resize an image, call a provider, rebuild a projection, or process an import without holding a web worker open. The queue also introduces a distributed boundary: a job can be delivered more than once, run after a deploy, fail after a side effect, or remain invisible while a worker is unavailable.

Queue design is about the job contract and failure policy, not only dispatch syntax. Laravel's drivers, worker commands, retry methods, middleware, and testing helpers vary across framework versions, so verify exact names against the target release.

## A Job Is a Message

A job should carry the smallest stable data needed to perform its work, usually identifiers and an operation key rather than an ORM object graph. Resolve current state inside the handler and re-check authorization or ownership where policy requires it.

~~~php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Models\Invoice;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Bus\Queueable;

final class GenerateInvoicePdf implements ShouldQueue
{
    use Queueable;

    public function __construct(
        public readonly int $invoiceId,
        public readonly int $tenantId,
        public readonly string $operationId,
    ) {
    }

    public function handle(InvoicePdfGenerator $generator): void
    {
        $invoice = Invoice::query()
            ->where('tenant_id', $this->tenantId)
            ->whereKey($this->invoiceId)
            ->firstOrFail();

        $generator->generate($invoice, $this->operationId);
    }
}
~~~

Job base conventions and some helper names differ between Laravel majors; the architectural points are stable. Do not serialize passwords, bearer tokens, payment data, or stale authorization decisions into a payload. Queue identifiers, then load current records with tenant scope.

## At-Least-Once and Idempotency

A worker can crash after a provider commits and before the queue acknowledges the job. The queue may deliver the job again. Make the handler idempotent with a unique operation key, a provider idempotency key, or a durable effect record.

~~~php
<?php

declare(strict_types=1);

final class InvoicePdfGenerator
{
    public function __construct(private PdfStore $files)
    {
    }

    public function generate(Invoice $invoice, string $operationId): void
    {
        if ($this->files->alreadyGenerated($operationId)) {
            return;
        }

        $temporaryPath = $this->files->renderTemporary($invoice);
        $this->files->publishAtomically($operationId, $temporaryPath);
        $this->files->markGenerated($operationId);
    }
}
~~~

The three operations need their own crash policy. If publishing succeeds and marking fails, a retry must detect the published artifact by operation ID. If a remote provider is involved, use its idempotency mechanism or reconcile by operation ID before retrying. A job's unique dispatch option, where available, does not replace idempotent effects after execution begins.

## Retry, Backoff, and Failure

Classify failures before choosing retries:

- transient network, lock, or capacity failures may retry;
- invalid input or missing permanent data usually needs a failed-job path;
- a provider timeout may be ambiguous and needs reconciliation;
- authorization revocation may require dropping or quarantining the job;
- a poison payload should not consume all worker capacity.

Set a bounded attempt count, backoff with jitter where supported, and a total age or deadline. A job that retries for days can deliver stale business behavior. Failed jobs need an owner, retention, alert, and safe replay policy. Replay should preserve the operation ID and re-check current authorization.

A retry can amplify a dependency outage. Use queue-specific concurrency, rate limits, circuit-breaking or provider backoff, and load shedding when the dependency is unhealthy. The worker should not run an unbounded loop inside one job.

## Transactions and Dispatch Timing

If a job refers to a database row, dispatch it only after the transaction that creates or updates the row commits. Many Laravel versions provide an after-commit dispatch option or queue configuration; otherwise, use an explicit application boundary. Dispatching before commit can produce a job that cannot find data that later commits, or a job that runs after a rollback.

An outbox table is useful when the database state and message enqueue must be durably coordinated. A queue dispatch call alone does not make a database transaction and broker operation atomic. Keep local writes, outbox records, and idempotency records in the transaction that owns the invariant.

## Ordering, Concurrency, and Uniqueness

Do not assume jobs run in dispatch order unless the chosen queue and keying policy explicitly guarantee it. Two jobs for one aggregate can race; include a version, use a lock, or serialize the operation by a stable key. A worker restart or multiple queue consumers can reorder work.

Job uniqueness controls duplicate enqueueing under a defined window, while idempotency controls duplicate execution. They solve different races. Locks should have an expiry and should not be held across long provider calls unless the design can tolerate lease loss. Prefer a state machine with operation IDs for workflows that span several jobs.

## Batches, Chains, and Workflows

A chain expresses ordered steps; a batch groups work and reports aggregate progress. They are useful for imports and fan-out processing, but each step can fail, retry, or complete while another step is running. Persist workflow state if users or operators need to resume, cancel, or reconcile it.

For a long business process, an explicit workflow or saga is clearer than nested jobs that implicitly depend on one another. Define compensation and deadlines. Do not put a large collection of IDs into one payload; partition work into bounded jobs.

## Worker Lifecycle and Security

Queue workers are long-running PHP processes. They need memory and time limits, graceful restarts, signal handling, and code deployment behavior. Reset request-scoped or tenant-scoped state after each job. Do not assume a process restart occurs after every deployment.

Protect queue transport credentials and encrypt or restrict payloads where the framework and threat model require it. Treat payload data as untrusted at the handler boundary. Log job class, operation ID, tenant, attempt, duration, and outcome, but redact tokens and personal data.

## Testing and Operations

Unit-test job branching and idempotency with fakes; integration-test serialization, database transactions, real queue visibility, and provider adapters. Laravel testing helpers can assert that a job was dispatched without running it, while worker integration tests should prove retry, timeout, failure, and duplicate behavior.

Monitor queue depth and age, throughput, attempts, failed jobs, worker memory, visibility timeouts, lock contention, and dependency latency. Alert on oldest job age and failed-job growth, not just process liveness. A queue that accepts messages but never makes progress is an outage.

## Common Mistakes

- Passing mutable ORM graphs or secrets in job payloads.
- Assuming a job runs exactly once or in dispatch order.
- Dispatching before the creating transaction commits.
- Treating unique dispatch as execution idempotency.
- Retrying permanent or ambiguous failures without a reconciliation plan.
- Running unbounded imports in one job.
- Keeping tenant or request state in a long-lived worker.
- Monitoring worker processes without measuring queue age.

## Senior Engineer Thinking

A Laravel job is a durable message processed by an unreliable worker. Keep payloads small and scoped, make effects idempotent, coordinate dispatch with transactions, bound retries and concurrency, define ordering and workflow state, and monitor age and failure. Queue convenience APIs do not remove distributed-systems semantics.

## Exercises

1. Design an idempotent invoice PDF job with operation state, artifact publication, retry, and cleanup behavior.
2. Model a job dispatched inside a database transaction and choose an after-commit or outbox solution.
3. Define retry and dead-letter policies for a timeout, validation error, authorization revocation, and provider rate limit.
4. Design a queue workflow for a million-row import with bounded payloads and resumable progress.

## Review Questions

1. Why should a job carry identifiers instead of a mutable ORM graph?
2. What crash window makes idempotency necessary?
3. How do unique dispatch and idempotent execution differ?
4. Why can dispatch before commit create a missing-data race?
5. Which metrics reveal a queue outage even when workers are running?
6. Why do long-lived workers need state cleanup and graceful restarts?

## Summary

Laravel queues move slow work out of HTTP but introduce at-least-once delivery, retries, ordering, visibility, and worker lifecycle concerns. Use small tenant-scoped payloads, idempotent effects, after-commit or outbox dispatch, bounded retry and concurrency policies, explicit workflows, secure logging, and queue-age monitoring. Verify exact queue APIs against the target Laravel version.

## References

- [Laravel documentation: Queues](https://laravel.com/docs/queues)
- [Laravel documentation: Dispatching after database transactions](https://laravel.com/docs/queues#jobs-and-database-transactions)
- [Laravel documentation: Task scheduling](https://laravel.com/docs/scheduling)
- [Laravel documentation: Testing queues](https://laravel.com/docs/mocking#queue-fake)
- [microservices.io: Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html)
