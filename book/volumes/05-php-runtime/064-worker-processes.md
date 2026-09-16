---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 64
title: Worker Processes
slug: worker-processes
status: complete
summary: ../../_ai/chapter-summaries/064-worker-processes-summary.md
---

# Chapter 64 — Worker Processes

## Why This Matters

PHP applications handle concurrent traffic even when each individual worker executes ordinary userland code sequentially. Ten browser requests can arrive together; PHP-FPM gives them to different workers, and those workers contend for shared databases, caches, files, CPUs, and upstream services. “PHP runs one request at a time” describes one worker, not the application.

Worker processes are therefore a capacity and failure-isolation design. They determine how many requests can execute at once, how much memory the host needs, how a slow dependency spreads, and what happens when one request crashes a worker.

```text
                 web-server queue
              ┌───┴───┬───┬───┐
              ▼       ▼   ▼   ▼
           worker 1 worker 2 worker 3 ...
              │       │   │
              └───────┴───┴──────→ database / cache / APIs
```

## Mental Model

A worker is a process with a finite lifetime and resources. In a conventional FPM deployment it accepts one request, executes it, emits a response, performs request cleanup, and becomes available again. A process supervisor or FPM master starts, monitors, reloads, and retires workers.

```text
arrival
  → queue
  → worker assignment
  → request service
  → response
  → cleanup
  → idle worker or worker exit
```

The system has at least three kinds of concurrency:

1. **Request concurrency:** multiple requests are in flight in different PHP workers.
2. **Dependency concurrency:** those requests may simultaneously use the same database or API.
3. **Process-management concurrency:** the master, workers, web server, and supervisor change state independently.

The first does not guarantee the second is safe. A database transaction, unique constraint, lock, or idempotency key still has to protect shared invariants.

## Core Concept

### One worker is not one user

Workers are reused. Request data, authenticated identity, temporary files, logging context, and mutable service state must be reset or scoped per request. A process-global cache may contain shared immutable data, but using it for user-specific state can leak data across requests. Chapter 63 develops the lifecycle boundary.

Some resources intentionally survive requests: OPcache, persistent connections, extension state, and allocated process memory. Their lifetime and reset behavior vary by extension and configuration. Treat each persistent resource as a documented dependency, not as a reason to assume all PHP variables survive.

### Capacity is the smallest bottleneck

A simplified throughput model is:

```text
effective concurrency
  = min(PHP workers, web-server capacity, database capacity,
        external-service capacity, CPU/memory capacity)
```

If 20 workers all make a database query but the database safely supports only 8 concurrent expensive queries, setting `pm.max_children=20` can increase queueing inside the database and make every request slower. Worker count is a concurrency budget that must be coordinated with downstream budgets.

For a stable workload, Little's Law gives a useful check:

```text
in-flight work ≈ arrival rate × average service time
```

Tail latency, burstiness, retries, and queue limits make real systems less tidy. Still, the equation catches impossible plans: a service handling 100 requests per second at 500 ms average service time needs about 50 in-flight request slots before burst headroom.

### Queueing is a failure mode

When all workers are busy, new requests wait. Waiting increases latency without doing useful application work. A queue can absorb a short burst, but an arrival rate above sustainable service rate grows the queue until a timeout, rejection, or memory limit occurs.

```text
traffic burst → queue grows → deadlines expire → clients retry
                                      ↓
                              more traffic and work
```

This retry amplification is why a healthy system needs bounded queues, timeouts, backpressure, and clients that do not retry every failure blindly.

## Practical Example: Worker Sizing

Suppose a container has 1,024 MiB available, non-PHP services need 300 MiB, and a high-water worker uses 70 MiB. The arithmetic gives approximately ten workers:

```text
(1024 - 300) / 70 = 10.3
```

A prudent initial ceiling is lower than the rounded result to leave headroom for allocator behavior and bursts. Then load-test with realistic code, extensions, response sizes, and database calls. Measure resident memory rather than relying only on a development `memory_limit`, because the process and native allocations also count against the container/host.

Worker sizing is an iterative control loop:

```text
measure high-water memory and service time
  → set process and downstream budgets
  → load test at and beyond the limit
  → observe queue, tail latency, errors, CPU, memory
  → change one constraint and repeat
```

## Production Example: A Slow Dependency

Consider a handler that makes a 2-second upstream call. With 8 workers, at most 8 such requests can execute at once in that pool. The ninth waits or fails. Increasing to 32 workers may improve concurrency, but it can also create 32 simultaneous upstream calls, exhaust connection limits, and cause the upstream to slow further.

Better controls include:

- a short connect and response timeout;
- a concurrency limit appropriate to the upstream;
- a bounded queue for work that need not be synchronous;
- a cache for safe repeated reads;
- a fallback that is honest about stale or unavailable data;
- metrics for active workers, queue time, dependency latency, and timeout reason.

Do not turn a request worker into a hidden queue consumer by waiting indefinitely for a dependency.

## Failure Isolation and Lifecycle

Processes provide isolation from ordinary userland memory and fatal failures. If one worker exits, other workers may continue, though shared services and the user-visible request can still be affected. An extension crash, host OOM, filesystem failure, or shared database outage can defeat the apparent isolation.

FPM may retire workers after a configured number of requests, on graceful reload, or after a fatal event. Recycling bounds accumulated process damage; it does not guarantee that every request starts from a pristine operating-system process. Make request correctness independent of worker identity.

## Operational Failure Modes

### All workers active

Look first for long request time, downstream latency, lock waits, and retry storms. An active count without request duration is not enough. Compare arrival rate, completion rate, queue time, and timeout rate. If the pool is saturated by one endpoint, isolate its budget or move asynchronous work out of the request path.

### Memory grows across workers

A growing worker high-water mark can result from application retention, static caches, circular structures, native extension behavior, fragmentation, or unusually large requests. Compare a new worker to one after thousands of requests, use allocation/profiling tools where available, and inspect what workload triggers growth. `pm.max_requests` is a guardrail while the cause is investigated.

### Thundering herd after restart

A restart can remove warm workers and caches simultaneously. If all workers immediately open database connections or compile code, the dependency sees a spike. Use controlled rolling/graceful procedures, readiness checks, warm-up only when it is safe, and a deployment plan that preserves capacity.

### Duplicate side effects

Two workers can process the same request after a client timeout, message redelivery, or user double-click. Local PHP variables cannot coordinate them. Protect the durable operation with an idempotency key, a unique database constraint, an atomic state transition, or a queue delivery policy.

### Worker is “idle” but traffic is slow

The visible FPM pool may be idle while the web server, network, database, or client is slow. Measure each boundary. Conversely, a worker can remain active after the response was flushed if post-response code is still running.

## Security

Process separation is not tenant isolation. Workers in one pool may access the same files, environment, socket, database credentials, and network. Use least-privileged pool identities, filesystem permissions, separate secrets, and explicit authorization in application code. Never store an authenticated user or authorization decision in process-global mutable state.

Protect process and status endpoints. Avoid exposing command-line arguments or environment values containing secrets to diagnostics. Log worker IDs or process IDs only as operational correlation fields; they are not stable identities and should not be trusted by application authorization.

## Performance

Track:

- request arrival and completion rate;
- active, idle, and maximum-active workers;
- queue and service time percentiles;
- worker RSS/high-water memory and restart rate;
- CPU, file descriptors, and network connections;
- database pool usage and lock waits;
- external call latency, timeout, and retry rate.

Optimize the largest measured constraint. More workers help CPU-bound work only while CPU is available; they help I/O-bound work only while dependencies and memory can support the concurrency. A worker process is not free parallelism.

## Testing

1. Run sequential requests through one worker and assert no request-state leakage.
2. Run concurrent requests that contend for one invariant and verify database-level correctness.
3. Load-test at pool capacity and observe queue and timeout behavior.
4. Inject slow, failed, and retrying dependencies.
5. Kill or recycle workers during representative traffic and verify recovery.
6. Test deployment/reload while requests are in flight.

Use real databases for concurrency tests where the invariant depends on locking or uniqueness. A mock that returns “available” to every call cannot demonstrate that two workers cannot reserve the same slot.

## Exercises

1. Given a measured 65 MiB worker, 900 MiB available to PHP, and 250 MiB headroom for other processes, calculate an upper bound and choose a conservative pool size.
2. Model eight workers calling an upstream with a limit of four concurrent requests. Choose a control strategy and explain its failure behavior.
3. Write a test where two concurrent requests attempt the same state transition. Identify the race in a check-then-write implementation and move the invariant to the durable boundary.
4. Draw a retry storm caused by a saturated pool. Add bounded queues, deadlines, and client retry rules.

## Review Questions

1. Why does one-request-at-a-time execution inside a worker not remove application concurrency?
2. What is the smallest-bottleneck rule, and how does it affect `pm.max_children`?
3. Why can a queue make an outage worse rather than hide it?
4. What kinds of state may outlive a request, and why must user state not rely on them?
5. Which correctness guarantees require a database or another shared service rather than PHP memory?

## Summary

Worker processes provide PHP concurrency, finite capacity, and partial failure isolation. Pool size must fit memory, CPU, queues, databases, and upstream limits. Reuse makes state hygiene essential, while retries and concurrent workers make durable idempotency and constraints essential. Chapter 65 examines the stronger lifecycle change: a PHP process deliberately serving many jobs over a long period.

## References

- [PHP Manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: FPM status page](https://www.php.net/manual/en/fpm.status.php)
- [PHP Manual: Process control extensions](https://www.php.net/manual/en/book.pcntl.php)
- [PHP Manual: OPcache](https://www.php.net/manual/en/book.opcache.php)
- [PHP Manual: `memory_get_usage`](https://www.php.net/manual/en/function.memory-get-usage.php)
