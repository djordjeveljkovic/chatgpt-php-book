---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 232
title: PHP-FPM
slug: php-fpm
status: complete
summary: ../../_ai/chapter-summaries/232-php-fpm-summary.md
---

# Chapter 232 — PHP-FPM

PHP-FPM is a FastCGI process manager for PHP. A web server accepts connections and forwards requests to a pool of PHP worker processes. The pool's process count, request limits, timeouts, and restart behavior determine how much concurrent work the application can handle.

FPM is a capacity boundary, not just a service to keep running. One slow database query or upstream call occupies a worker, and enough occupied workers produce queueing and timeouts even when CPU is low.

## Why this matters

A request can be healthy in isolation and still fail under concurrency because every worker is busy. Capacity planning must include worker memory, CPU, database connections, upstream limits, request duration, and traffic bursts.

```text
client → web server → FastCGI queue → PHP-FPM pool → PHP request
                                    ↘ database / HTTP / filesystem
```

Measure the whole path. A high FPM queue may be caused by a slow database, exhausted connection pool, CPU saturation, or a deliberately small worker pool.

## Pool modes and worker count

FPM supports pool process-management modes such as `static`, `dynamic`, and `ondemand`. Their settings control how many workers are kept ready and how idle workers are created or removed. The correct mode depends on traffic shape and startup cost; it is not a universal production default.

A starting capacity estimate is constrained by memory:

```text
max_children ≤ memory available for PHP / peak resident memory per worker
```

Leave room for the operating system, web server, OPcache shared memory, database clients, monitoring, and bursts. Benchmark peak rather than average worker memory. Increasing `pm.max_children` can increase throughput until another dependency saturates, then increase latency and failure.

Use `pm.max_requests` to recycle workers after a bounded number of requests when extensions or application code retain memory. Recycling masks a leak's impact; it does not find or fix the leak. Observe worker age and memory before choosing the value.

## Timeouts and queues

Set a web-server request timeout, FPM request termination policy, and finite timeouts for database and upstream calls. Their relationship should leave enough time for cleanup and an accurate error response. A request that waits on an upstream until the client disconnects still consumes a worker.

The FastCGI listen backlog and web-server connection limits affect bursts. A large backlog can absorb a short burst but can also hide overload and increase user-visible waiting. Apply backpressure and rate limits at the edge for expensive work.

Do not confuse `request_terminate_timeout` with an application-level timeout. Killing a PHP process may interrupt cleanup or leave an external operation ambiguous. Make external operations idempotent and reconcile them.

## Graceful reloads and deploys

A graceful reload starts new workers with new code while allowing current requests to finish within a bounded policy. Long-running requests and stuck dependencies can delay retirement, so monitor old-worker age and enforce finite request deadlines. Deploy code, configuration, and OPcache policy together; see [Chapter 231 — OPcache](./231-opcache.md).

Use readiness checks that reflect the pool's ability to accept work without making every check depend on a slow business query. A process can be alive but unable to serve because all children are occupied or the database is unavailable.

## Observability

Expose FPM status only through an authenticated, internal path. Useful signals include active and idle processes, max active processes, accepted connections, listen queue length, slow requests, terminated requests, request duration, and worker memory. Correlate FPM observations with web-server access logs and dependency metrics.

A slowlog can capture a PHP backtrace for requests that exceed a threshold. Protect its files because arguments and paths can contain sensitive data. Use sampling and retention appropriate to the environment.

## PHP request behavior

Traditional PHP-FPM workers handle one request at a time. Static globals, request-scoped services, and open transactions should not leak between requests; framework bootstraps and worker lifecycle still deserve tests. FPM process isolation limits some state sharing, but shared databases, caches, files, and external systems remain shared boundaries.

A worker crash loses in-memory request state. Durable work must be committed to a database or queue before returning success. A response sent to a client does not prove a background side effect completed.

## Testing and operations

Load-test representative request mixes with realistic response sizes and dependency latency. Test pool exhaustion, slow upstreams, database saturation, worker recycling, graceful reload, failed startup, and rollback. Record memory per worker and the point at which latency rises sharply.

Tune one constraint at a time. A bigger pool may only move queueing to the database. Set alerts for sustained listen queue growth, maxed workers, high termination counts, memory pressure, and latency percentiles. Include an operator runbook for draining, reloading, and restoring capacity.

## Exercises

1. Measure peak resident memory for a representative PHP worker and derive a conservative `pm.max_children` for a host with a fixed memory budget.
2. Simulate a slow upstream and observe FPM workers, queue length, web-server timeouts, and database connections.
3. Design a graceful deploy for a release that changes both PHP code and an OPcache policy.

## Review questions

- Why can a low CPU reading coexist with an exhausted FPM pool?
- How should worker memory constrain `pm.max_children`?
- What does `pm.max_requests` mitigate and what does it hide?
- Why must timeout budgets include database and upstream calls?
- Which signals distinguish a live process from a ready pool?

## Summary

PHP-FPM turns PHP execution into a bounded worker pool. Size it from peak memory and dependency capacity, choose process mode for traffic shape, set finite timeouts and recycling policy, deploy with graceful reloads, observe queue and worker state, and test overload and recovery rather than tuning from averages alone.

## References

- [PHP manual: PHP-FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP manual: FPM status page](https://www.php.net/manual/en/install.fpm.status.php)
- [PHP manual: FPM configuration directives](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP manual: FastCGI Process Manager](https://www.php.net/manual/en/install.fpm.php)
