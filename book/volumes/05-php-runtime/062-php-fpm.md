---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 62
title: PHP-FPM
slug: php-fpm
status: complete
summary: ../../_ai/chapter-summaries/062-php-fpm-summary.md
---

# Chapter 62 — PHP-FPM

## Why This Matters

PHP-FPM (FastCGI Process Manager) is the process manager most commonly placed behind a production web server for PHP. It turns the FastCGI protocol into an operational system: a master process manages pools, workers accept requests, configuration controls process counts and recycling, and status/slow-request facilities expose some of the runtime's health.

FPM is not “PHP running in the background” in an unlimited sense. It is a finite resource pool. Every worker consumes memory, can wait on a database or upstream, and can be unavailable while serving one slow request. Correct configuration starts with a workload and memory budget, not with copying a sample `pm.max_children` value.

```text
systemd / container supervisor
              ↓ start, stop, reload, restart
FPM master process
       ├─ pool: www → worker 1
       ├─ pool: www → worker 2
       ├─ pool: admin → worker 1
       └─ sockets / TCP listeners
              ↓ FastCGI requests
web server / reverse proxy
```

## Mental Model

An FPM master owns process-management decisions. A worker handles a request and, in the usual non-threaded PHP-FPM model, does not handle another request concurrently. When the request ends, FPM can reuse the worker for the next request; request-scoped state is cleaned up, while process-level resources and caches may remain according to the runtime and extension behavior.

```text
master starts
  → reads global and pool configuration
  → creates/listens on endpoint
  → starts or spawns workers according to pm mode
  → worker accepts request
  → PHP request lifecycle
  → worker returns to idle pool
  → master monitors, spawns, retires, or stops workers
```

FPM settings are documented in the [PHP Manual FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php). Configuration syntax and available directives can vary with PHP version and build; inspect the deployed binary and validate configuration during deployment.

## Core Concept

### Pools are isolation and capacity boundaries

A pool can have its own listener, user/group, environment policy, PHP settings, and process limits. Separate pools can protect an administrative application from a public application or provide different memory and timeout budgets. They do not create a magical security boundary: the operating-system user, filesystem permissions, shared database, network access, and application credentials still determine actual isolation.

Use separate pools when the operational requirements differ materially. Too many pools fragment capacity and make incident diagnosis harder. Start with a clear ownership model: which pool serves which release, which user owns its files, and which metrics identify saturation.

### Process-manager modes

The principal `pm` modes have different startup and capacity behavior:

| Mode | Behavior | Tradeoff |
| --- | --- | --- |
| `static` | Keep `pm.max_children` workers | Predictable capacity and memory; idle workers remain allocated |
| `dynamic` | Maintain configured start, spare, and maximum counts | Adapts to demand; more tuning and spawn churn |
| `ondemand` | Spawn when work arrives and retire idle workers | Low idle footprint; cold-start latency and churn |

`pm.max_children` is a concurrency ceiling for the pool, not a throughput guarantee. If all workers are busy, requests queue at the web-server/FastCGI boundary or are rejected/time out. Raising it can improve throughput only while CPU, memory, database connections, and downstream services have headroom.

### Recycling is controlled damage

`pm.max_requests` can retire a worker after it has served a number of requests. Recycling can bound the impact of a leak or fragmentation in application code or an extension, but it is not a cure. It also creates respawn work and can hide a regression if used without a memory alert. Record worker RSS/peak memory over time and investigate the retaining object or extension when practical.

### Timeouts are layered

A request can encounter client, proxy, web-server, FastCGI, FPM, PHP, database, and external-service timeouts. A longer outer timeout does not make a shorter inner operation reliable. A worker blocked in a downstream call still consumes a slot until the call returns or the process is terminated.

```text
client deadline
  > proxy/web-server deadline
  > FastCGI read deadline
  > application budget
  > database/external-call deadlines
```

The inequality is illustrative, not a universal set of values. Design a budget with margin for response transfer and cleanup, and make the smallest deadline produce a diagnosable failure.

## Practical Example: Budgeting a Pool

Suppose a host has 2 GiB available to PHP, while the operating system, web server, and sidecars require 600 MiB. If a representative worker uses 80 MiB at its high-water mark, a conservative first estimate is:

```text
(2048 MiB - 600 MiB) / 80 MiB = 18 workers
```

That is a starting ceiling, not a final answer. Leave headroom for allocator behavior, deployment spikes, OPcache/shared memory, non-PHP processes, and measurement error. Then validate under representative concurrency and database load. A lower `pm.max_children` with a fast, bounded queue can be healthier than swapping the host into failure.

The operational loop is:

```text
measure worker memory and request time
  → set a safe ceiling
  → load test with downstream limits
  → observe queueing and errors
  → adjust one constraint at a time
```

## Production Example: Pool Configuration

The exact file location is distribution-dependent, but a pool commonly contains settings shaped like these:

```ini
[app]
user = app
group = app
listen = /run/php/app.sock
listen.owner = web
listen.group = web
listen.mode = 0660

pm = dynamic
pm.max_children = 18
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 6
pm.max_requests = 1000

request_slowlog_timeout = 3s
slowlog = /var/log/php-fpm/app-slow.log
request_terminate_timeout = 30s
pm.status_path = /fpm-status
ping.path = /fpm-ping
```

These values are illustrative. Protect status and ping locations at the web-server boundary; do not publish them as unauthenticated public endpoints. Confirm directive availability and unit syntax with the [FPM configuration reference](https://www.php.net/manual/en/install.fpm.configuration.php), then run the deployed FPM binary's configuration test before reload.

### Graceful reload and deployment

A reload should apply configuration and code changes without dropping healthy in-flight work when the service manager and FPM build support the intended behavior. A restart is more disruptive and may be necessary for some changes. Deployment should use a complete immutable release, point the web server and/or pool at the intended path, validate configuration, reload or restart according to the change, and monitor old workers draining and new workers serving.

Do not assume OPcache invalidation, symlink changes, and FPM reload have identical timing. Test the exact deployment procedure, including a request during the transition. The source-to-execution and cache boundaries were introduced in Chapter 40 and Chapter 59.

## Operational Failure Modes

### Pool exhaustion

Symptoms include rising upstream queue time, 502/503 responses, and all workers showing active. Causes include slow SQL, external calls, lock contention, CPU saturation, or a request that never returns. Check active-worker counts, request duration percentiles, slow logs, downstream latency, and the web-server upstream queue. Increasing the pool can worsen the root cause by increasing concurrent database work.

### Memory pressure and OOM kills

If the product of worker count and high-water memory exceeds host/container limits, the kernel or platform may kill processes. The resulting error can look like a random request failure. Alert on resident memory, container limits, worker count, restart rate, and swap/OOM events. Set `pm.max_children` from measured memory, and use recycling only as a bounded mitigation.

### Stale or inconsistent release

Some workers can serve old code while others serve a new release during a transition. Incompatible schema/code changes can then fail only for some requests. Use backwards-compatible migrations, immutable release directories, and a tested reload/drain strategy. A pool restart cannot repair an incompatible database migration.

### Socket and permission errors

A missing socket, wrong owner/group/mode, different chroot/path, or service-start ordering can prevent the web server from connecting. Check the listener from both service identities and include endpoint ownership in deployment diagnostics. Avoid making a socket world-writable to “fix” a permission issue.

### Slowlog is not tracing

FPM slow logs can show a stack trace for requests exceeding the configured threshold, but they do not explain every queue or downstream event. Combine them with application logs, access logs, database telemetry, and request IDs. Ensure log paths are writable and rotated; an unwritable diagnostic path is itself a failure.

## Security

Run pools with least privilege and keep application files non-writable by the serving user where deployments permit. Restrict socket permissions and TCP listeners. Review pool environment handling, including whether inherited environment variables are cleared, and inject only the configuration the application needs. Never expose FPM status, ping, or a diagnostic endpoint without an access policy.

Do not solve isolation by setting `open_basedir` or a similar option and assuming the process is sandboxed. Application, OS, container, filesystem, network, and secret-store controls have different guarantees. Disable production display of errors and ensure FPM/web-server logs do not expose credentials or request bodies.

## Performance

FPM performance is queueing plus work. Track request rate, active/idle workers, max active workers, queue time, service time, memory, CPU, and downstream latency. A useful capacity approximation is:

```text
concurrency ≈ arrival rate × average service time
```

This is Little's Law applied to a stable system; tail latency and bursts still matter. If average service time doubles, the same arrival rate needs twice the concurrency to avoid a growing queue—until another resource saturates.

Avoid measuring only requests per second at a low concurrency. Load-test cold starts, warm workers, slow dependencies, large responses, and deployment reloads. Compare `static`, `dynamic`, and `ondemand` only against the actual traffic pattern and memory envelope.

## Testing

1. Run the FPM configuration test in CI/deployment using the same PHP build.
2. Smoke-test socket ownership, web-server connectivity, a healthy request, and a controlled error.
3. Load-test until the intended worker ceiling and verify queue/timeout behavior.
4. Inject a slow database or upstream and confirm deadlines, slow logging, and recovery.
5. Test graceful reload during in-flight requests and during a release change.
6. Verify that status/ping/diagnostic endpoints are inaccessible to unauthorized clients.

Keep a capacity worksheet with measured worker memory, downstream connection limits, and chosen headroom. Re-run it after PHP, extension, framework, or workload changes.

## Exercises

1. Measure the high-water memory of a representative application worker and derive a conservative `pm.max_children` from a stated host budget.
2. Draw a timeout budget across browser, proxy, web server, FPM, application, and database. Explain which layer should fail first.
3. Simulate all workers waiting on a slow upstream. Identify the first metric that rises and the change that would reduce—not merely move—the queue.
4. Design a two-release deployment for a schema change that must work while old and new workers coexist.

## Review Questions

1. What does `pm.max_children` limit, and why is it not a throughput setting by itself?
2. When is worker recycling useful, and why should it not replace leak investigation?
3. What is the difference between pool exhaustion and an unavailable FPM socket?
4. Why can increasing PHP workers overload the database?
5. Which FPM operational endpoints require protection, and what should they be used for?

## Summary

PHP-FPM manages finite FastCGI worker pools. Pool mode, process count, memory, request deadlines, recycling, sockets, reloads, and observability are production controls with tradeoffs. Size the pool from measured memory and downstream capacity, then test overload and deployment transitions. Chapter 63 follows one request through the lifecycle that each FPM worker hosts.

## References

- [PHP Manual: Installation of FPM](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: FPM status page](https://www.php.net/manual/en/fpm.status.php)
- [PHP Manual: FPM pool options](https://www.php.net/manual/en/install.fpm.configuration.php#listen)
- [PHP Manual: PHP configuration directives](https://www.php.net/manual/en/ini.list.php)
