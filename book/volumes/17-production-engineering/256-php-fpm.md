---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 256
title: PHP-FPM
slug: php-fpm
status: complete
summary: ../../_ai/chapter-summaries/256-php-fpm-summary.md
---

# Chapter 256 — PHP-FPM

## Why This Matters

PHP-FPM is the process boundary that turns a PHP application into a service. It accepts FastCGI requests, assigns them to worker processes, and recycles or stops those workers according to pool policy. Its limits determine whether requests execute, queue, fail quickly, or consume resources until a timeout.

Chapter 232 introduced FPM as a performance and capacity model. This chapter treats it as a production operating boundary: pool ownership, deployment, signals, health, process limits, logs, permissions, and failure recovery.

## Pool Model

```text
Nginx → FPM master → pool → child worker → one request
                         ├─ database
                         ├─ cache
                         └─ provider
```

The master manages child processes. A traditional worker handles one request at a time, so a slow dependency occupies a child. The pool is healthy only when worker count, memory, CPU, database connections, provider quotas, and request deadlines fit together.

## Process Management

FPM supports process-management modes such as static, dynamic, and ondemand. The choice depends on traffic shape, startup cost, memory, and latency targets. The setting is not a universal “production value.”

`pm.max_children` is a concurrency ceiling for a pool. A safe value is constrained by more than CPU:

```text
pool children × peak resident memory
  + OPcache + web server + system + sidecars
  < host or container memory budget
```

The total database connections across all pools and replicas must also fit the database budget. A larger pool can increase queueing downstream and worsen latency.

`pm.max_requests` can recycle workers after a bounded number of requests. It is useful when an extension or application path retains memory, but recycling reduces impact rather than finding the cause. Track worker age and memory and investigate growth.

## Request Limits and Timeouts

Align the edge, FPM, application, database, and provider deadlines. A web-server timeout that fires first may leave an ambiguous remote effect; an FPM termination that fires first may skip useful application classification. Keep request termination as a last-resort boundary, not the only timeout policy.

Bound request body, execution time, upload size, and slow requests. A bound should protect the pool without rejecting valid work under normal payload and dependency latency. See [Chapter 238 — Timeouts](../16-distributed-systems/238-timeouts.md).

## Status and Slow Logs

FPM status information can reveal active processes, idle processes, max active processes, accepted requests, listen queue, and slow or terminated requests. Expose it only on an internal authenticated path or through an operator channel.

A slow log can capture a backtrace for a request that exceeds a threshold. Protect the output: paths, arguments, and request context can contain sensitive information. A slow trace identifies where a worker was observed, not necessarily why a dependency was slow.

Correlate FPM data with Nginx upstream timings, application traces, database pool wait, and provider metrics. A full FPM pool is a symptom; the root cause may be a lock, query, provider, CPU, or memory problem.

## Graceful Reload and Deployment

A graceful reload starts new workers with new configuration while old workers finish current requests. Long-running requests or stuck dependencies can delay retirement. Set finite deadlines, observe old-worker age, and define what happens when graceful completion exceeds the drain window.

Deploy code and configuration as a release. Verify the active release path, PHP binary, extensions, pool configuration, and OPcache policy. Keep database and queue contracts compatible while old and new workers overlap. See [Chapter 231 — OPcache](../15-performance/231-opcache.md) and [Chapter 264 — Deployment](./264-deployment.md).

## Permissions and Sockets

The FPM pool user and group determine access to application files, runtime directories, Unix sockets, temporary files, and logs. Use a narrow runtime directory and explicit ownership. A socket must be accessible to Nginx but not to unrelated users.

Do not make the entire application tree writable by the worker. Code and configuration should be immutable at runtime; uploads, cache files, sessions, and generated artifacts need separate paths and policies.

## Health and Readiness

Liveness asks whether the process is running. Readiness asks whether the pool can accept useful work. A health endpoint that performs a slow business query on every probe can consume the same workers it is supposed to monitor.

Use a cheap process and pool signal for liveness, and a bounded dependency-aware readiness policy. Do not report ready when all children are occupied or when required configuration is absent. Avoid making one transient optional provider failure remove all service capacity.

## PHP Worker State

Request state should end with the request. FPM normally tears down request-local userland statics, globals, transactions, locale, tracing context, and tenant identity between requests. Long-running workers, persistent connections, extension-managed state, open streams, and external resources need deliberate cleanup. FPM process isolation limits cross-request leakage but does not protect shared databases, files, caches, or external systems.

A worker crash loses local state. Return success only after required durable work is committed or accepted by its durable owner. A response sent by Nginx does not prove a background effect completed.

## Operations and Failure Drills

Test startup with invalid configuration, missing extensions, unavailable sockets, unwritable directories, full pool, slow database, slow provider, worker crash, memory growth, graceful drain, forced termination, and rollback. Test the exact service user and container limits used in deployment.

Monitor active and idle workers, queue length, max active processes, request duration, terminated requests, worker memory, process restarts, startup failures, and upstream errors. Alert on sustained customer-visible latency and queue growth, not every short-lived worker transition.

## Security

Run pools with least privilege, restrict status and slow-log access, protect socket permissions, and avoid exposing environment or source paths. Do not assume an internal FastCGI socket makes a request trusted; application authentication and authorization still apply.

## Common Mistakes

* Setting `pm.max_children` from CPU count alone.
* Counting one pool's connections without summing replicas and pools.
* Using `pm.max_requests` as a substitute for leak investigation.
* Allowing graceful reloads to wait forever on a stuck provider.
* Exposing FPM status or slow logs publicly.
* Sharing writable code and upload directories.
* Calling a process health check readiness.
* Treating an Nginx response as proof of durable business completion.

## Senior Engineer Thinking

Ask which finite resource the pool protects, what occupies a child, and what evidence distinguishes queueing from dependency saturation. FPM tuning is successful when the entire request path remains within its contract and can drain, deploy, fail, and recover predictably.

## Exercises

1. Derive a conservative pool size from memory, database connections, provider concurrency, and a host budget.
2. Design liveness and readiness checks that do not consume scarce request capacity.
3. Perform a graceful reload while one request is stuck behind a bounded provider timeout.
4. Build a dashboard joining FPM queue, worker memory, Nginx upstream time, PHP errors, and database wait.

## Review Questions

* What does `pm.max_children` actually bound?
* Why must memory and database connections be included in pool sizing?
* How is readiness different from liveness?
* What can delay a graceful reload?
* Which state should not survive between requests?
* Why is worker recycling not a leak fix?

## Summary

PHP-FPM is a production process and capacity boundary. Size pools from memory and all shared dependency budgets, align request and termination limits, protect sockets and writable paths, expose status safely, drain workers deliberately, reset request state, and test startup, saturation, reload, crash, and rollback behavior with the real service user and runtime limits.

## References

- [PHP manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP manual: FPM status page](https://www.php.net/manual/en/fpm.status.php)
- [Chapter 232 — PHP-FPM](../15-performance/232-php-fpm.md)
- [Chapter 238 — Timeouts](../16-distributed-systems/238-timeouts.md)
