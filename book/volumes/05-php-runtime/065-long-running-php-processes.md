---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 65
title: Long-Running PHP Processes
slug: long-running-php-processes
status: complete
summary: ../../_ai/chapter-summaries/065-long-running-php-processes-summary.md
---

# Chapter 65 — Long-Running PHP Processes

## Why This Matters

Normal PHP web work is usually request-shaped: start with a request, do bounded work, produce a response, and return the worker to the pool. A queue consumer, daemon, scheduler loop, websocket server, or streaming command changes the contract. One process may execute thousands of jobs without restarting.

That can reduce startup overhead and make event-driven work possible, but it also makes process state, memory, connections, signals, deployment, and retries explicit engineering concerns. A local variable, static property, singleton, open transaction, or logging context that would disappear at the end of a normal CLI invocation can remain alive for hours.

```text
supervisor
    ↓ start / stop / restart
long-running PHP process
    ├─ receive job
    ├─ create job scope
    ├─ perform bounded work
    ├─ acknowledge or reject
    ├─ release/reset job resources
    └─ repeat until drain requested
```

## Mental Model

There are two nested lifecycles:

```text
process lifetime:  start ───────────────────────────── stop
job lifetime:             job 1 → cleanup → job 2 → cleanup → ...
```

A request worker may be reused, but the runtime and framework often provide a strong request teardown boundary. A deliberate long-running process must create its own job boundary. The process is an optimization and a reliability unit, not a place to put arbitrary shared mutable state.

Long-running does not mean immortal. A healthy process has a planned shutdown path, bounded work per iteration, telemetry, and a supervisor that can restart it. Planned recycling is often safer than allowing slow memory growth or stale connections to continue indefinitely.

## Core Concept

### Scope dependencies to one job

Construct stable, read-only dependencies once where appropriate, but create mutable job state inside the loop:

```php
<?php

declare(strict_types=1);

$queue = connectToQueue();
$logger = createLogger();
$stop = false;

while (!$stop) {
    $message = $queue->receive(timeout: 5);

    if ($message === null) {
        continue;
    }

    $jobId = $message->id;
    $startedAt = hrtime(true);

    try {
        $command = decodeAndValidate($message->body);
        handleCommand($command);
        $queue->ack($message);
    } catch (RetryableFailure $exception) {
        $logger->warning('job_retry', ['job_id' => $jobId]);
        $queue->release($message, delay: retryDelay($message));
    } catch (Throwable $exception) {
        $logger->error('job_failed', ['job_id' => $jobId, 'exception' => $exception]);
        $queue->reject($message);
    } finally {
        clearJobContext();
        closeJobTransactionIfOpen();
        $logger->info('job_finished', [
            'job_id' => $jobId,
            'elapsed_ms' => (hrtime(true) - $startedAt) / 1_000_000,
        ]);
    }
}
```

The names represent an application contract, not a universal queue API. The important properties are explicit acknowledgment, a classification of retryable versus permanent failures, bounded job scope, and cleanup that runs for ordinary exceptions. A process killed by the OS can skip `finally`, so the queue's visibility timeout/lease and redelivery policy must make recovery safe.

### Memory is cumulative

In a short-lived script, memory released at process exit is a coarse safety net. In a worker, a collection appended on every iteration, a static cache keyed by unbounded input, a retained closure, or a library buffer can grow for the whole process:

```php
// Unbounded process-lifetime state: a production risk.
static $seen = [];
$seen[$message->id] = true;
```

Use bounded caches, delete per-job references, avoid retaining full payloads, and measure memory before and after jobs. `gc_collect_cycles()` can collect eligible cyclic garbage, but it cannot free objects still reachable from a cache or global. Garbage collection is a tool, not a leak diagnosis.

### Connections become stale

Database, Redis, HTTP, and message-broker connections can be closed by an idle timeout, load balancer, failover, or network event while the PHP object remains. Check or reconnect according to the client library's contract. Do not blindly retry a write after an ambiguous network failure; use an operation ID or durable idempotency mechanism.

Transactions must be short and job-scoped. An exception that leaves a transaction open can hold locks across unrelated jobs. Ensure the connection is rolled back or discarded before the next message. A persistent connection is not the same thing as a healthy transaction.

### Configuration and code can become stale

A long-lived process usually does not see changed environment variables, configuration files, secrets, or deployed source until it is restarted or explicitly reloaded. This is desirable for consistency during one process lifetime, but it means deployment must drain and replace old processes. Do not assume changing a `.env` file changes a running worker.

## How It Works

At startup, the process loads code and configuration, opens stable infrastructure connections, installs logging and signal behavior, and reports readiness. It then waits for work. Each job passes through receive, decode, validate, execute, acknowledge/reject, cleanup, and metrics. A stop request changes the process from accepting new work to draining current work, after which it exits with a supervisor-visible status.

```text
start → initialize → ready
                    ↓
              receive job
                    ↓
       decode → validate → execute
          │         │         │
          └── failure/classification
                    ↓
          ack / retry / dead letter
                    ↓
                 cleanup
                    ↓
          stop requested? ── no → receive
                    │
                   yes
                    ↓
                 drain → exit
```

The queue delivery contract matters. At-least-once delivery means the same job can be delivered again; exactly-once business effect must be implemented with idempotency and durable constraints, not inferred from a worker loop. Acknowledging before the side effect risks loss; acknowledging after it risks duplicate delivery after a crash. Pick and document the tradeoff, then make duplicates safe.

## Practical Example: A Bounded Worker

Bound the process by jobs and memory and expose a clear exit reason:

```php
<?php

declare(strict_types=1);

$maxJobs = 500;
$maxMemory = 256 * 1024 * 1024;
$processed = 0;
$stopRequested = false;

if (function_exists('pcntl_async_signals')) {
    pcntl_async_signals(true);
    pcntl_signal(SIGTERM, static function () use (&$stopRequested): void {
        $stopRequested = true;
    });
    pcntl_signal(SIGINT, static function () use (&$stopRequested): void {
        $stopRequested = true;
    });
}

while (!$stopRequested && $processed < $maxJobs) {
    $job = receiveJob(timeout: 5);

    if ($job === null) {
        continue;
    }

    try {
        processJob($job);
        acknowledge($job);
    } catch (Throwable $exception) {
        reportJobFailure($job, $exception);
        retryOrDeadLetter($job, $exception);
    } finally {
        unset($job);
    }

    $processed++;

    if (memory_get_usage(true) > $maxMemory) {
        report('worker_memory_budget_reached');
        break;
    }
}

closeConnections();
exit($stopRequested ? 0 : 0);
```

This is illustrative: a real implementation must ensure that its receive timeout, lease duration, acknowledgment, and shutdown deadline agree. The process should exit non-zero for unrecoverable initialization failure, while a graceful drain normally exits successfully after the current job. A supervisor policy should distinguish crash loops from planned recycling.

## Production Example: Deployment Drain

A safe deployment sequence for workers is:

```text
start new version
  → health/readiness check
  → stop old workers from receiving new jobs
  → allow in-flight jobs to finish until deadline
  → terminate stragglers according to retry contract
  → remove old version
```

If old and new code can see the same queue during a rolling deployment, messages must be compatible with both versions. Database migrations should preserve old readers until the drain is complete. A forced termination can redeliver an in-flight job; the handler must tolerate that outcome.

## Operational Failure Modes

### Poison messages

A malformed or permanently invalid message can fail instantly and be retried forever, consuming the worker and filling logs. Track attempts, classify permanent failures, move poison messages to a dead-letter/quarantine path, and alert with enough metadata to investigate without logging secrets.

### Retry storm

If a database outage causes every worker to retry immediately, the outage receives more traffic. Use bounded attempts, exponential backoff with jitter, circuit/bulkhead behavior where appropriate, and a queue policy that delays redelivery. Do not acknowledge work just to stop alerts unless the loss is an intentional business decision.

### Memory leak or fragmentation

A worker can process successfully while its RSS rises until the container is killed. Compare memory by job type and lifetime, use bounded caches, recycle after a measured threshold, and fix retaining references. A periodic restart reduces blast radius but may hide the triggering payload if metrics are missing.

### Stale connection or secret

The first job after an idle period may hit a closed connection or expired credential. Surface reconnect and authentication failures distinctly. A reconnecting client must not replay a non-idempotent write without knowing whether the server accepted it.

### Slow or stuck job

One job can block a single-threaded worker indefinitely. Set per-operation and overall job deadlines, make cancellation/retry semantics explicit, and use separate queues or worker classes for very different runtimes. A process-level timeout is a final guard, not a substitute for dependency timeouts.

### Shutdown during side effect

A graceful signal can arrive after a payment call and before acknowledgment. Redelivery is expected; the payment/provider operation needs an idempotency key and the local state machine needs reconciliation. `finally` can release local resources, but it cannot make a remote call atomic with acknowledgment.

## Security

Long-lived processes retain more than developers expect. Never retain raw credentials, request bodies, authorization decisions, or tenant-specific objects in process-global state. Clear logging context and per-job identity in `finally`. Run workers with least privilege and separate queues/credentials when trust domains differ.

Validate message payloads as untrusted input, even when a trusted producer created them. Protect dead-letter contents and logs. Do not put secrets in job IDs, process arguments, or metrics labels. Limit payload size and reject unexpected serialized object formats; unsafe deserialization can turn a queue into a code-execution boundary.

## Performance

Measure throughput, job latency percentiles, receive wait, acknowledgment time, retry rate, queue age, active workers, memory before/after each job, and downstream calls. Batch only when ordering, failure, and acknowledgment semantics permit it. Larger batches may improve database throughput but increase lock duration, redelivery work, and shutdown latency.

A single long-running process avoids repeated startup but can become a large blast radius. Multiple workers improve concurrency and isolation while consuming more memory and connections. Choose the number from queue arrival rate, job service time, memory, downstream limits, and required failure isolation.

## Testing

1. Unit-test decode, validation, retry classification, backoff, and idempotency.
2. Integration-test acknowledgment ordering, visibility timeout, redelivery, dead-lettering, and transactions.
3. Run a worker for many iterations and assert memory remains within a stated envelope.
4. Inject connection drops, timeouts, dependency failures, malformed jobs, and duplicate delivery.
5. Send graceful shutdown during idle, receive, and side-effect phases.
6. Test deploy drain with compatible old/new message and schema versions.
7. Use a subprocess and the actual supervisor/container limits for smoke tests.

## Exercises

1. Design the acknowledgment order for an email job and explain how a crash before and after acknowledgment behaves.
2. Add a bounded per-job context to a worker and create a test proving that tenant data cannot appear in the next job's logs.
3. Generate a poison message and design a dead-letter policy with an alert that avoids logging its secret fields.
4. Measure memory after every 100 jobs, then decide whether to fix a retaining reference, bound a cache, or recycle the process.
5. Draw a deployment drain with a 20-second shutdown deadline and a job whose maximum safe runtime is 60 seconds. State the required queue lease and retry behavior.

## Review Questions

1. What lifecycle boundary must a long-running process create explicitly?
2. Why does at-least-once delivery require idempotent handlers?
3. What can `gc_collect_cycles()` fix, and what can it not fix?
4. Why is a successful response/acknowledgment not atomic with a remote side effect?
5. Which signals and deployment events require a drain rather than an immediate kill?

## Summary

Long-running PHP processes trade repeated startup for persistent process state. They need explicit per-job cleanup, bounded memory, healthy connection handling, idempotency, retry/backoff, poison-message policy, signal-aware draining, and replacement during deployment. Treat planned recycling as a guardrail and measure the process over time. Chapter 66 develops signals and graceful shutdown in detail.

## References

- [PHP Manual: Process Control](https://www.php.net/manual/en/book.pcntl.php)
- [PHP Manual: `pcntl_signal`](https://www.php.net/manual/en/function.pcntl-signal.php)
- [PHP Manual: `pcntl_async_signals`](https://www.php.net/manual/en/function.pcntl-async-signals.php)
- [PHP Manual: Garbage Collection](https://www.php.net/manual/en/features.gc.php)
- [PHP Manual: `memory_get_usage`](https://www.php.net/manual/en/function.memory-get-usage.php)
- [PHP Manual: `fastcgi_finish_request`](https://www.php.net/manual/en/function.fastcgi-finish-request.php)
