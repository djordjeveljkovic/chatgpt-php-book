---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 231
title: OPcache
slug: opcache
status: complete
summary: ../../_ai/chapter-summaries/231-opcache-summary.md
---

# Chapter 231 — OPcache

OPcache stores compiled PHP script data in shared memory so workers can reuse it instead of parsing and compiling every request. It reduces CPU and startup work, but it does not make application code, database queries, or external calls faster by itself.

OPcache is part of the PHP runtime and deployment contract. The CLI and FPM SAPIs may use different `php.ini` files and different caches. A setting verified in a shell may say nothing about the web workers serving traffic.

## Why this matters

Without opcode caching, each PHP-FPM request may repeat lexical analysis, parsing, compilation, and opcode allocation. With OPcache, workers can execute cached compiled scripts until the cache is invalidated or the process restarts. The benefit is workload-dependent; measure request CPU, latency, hit rate, and memory rather than assuming a percentage.

The conceptual path is:

```text
PHP source → parse and compile → opcodes → shared OPcache memory → worker execution
```

OPcache does not cache a request's local variables, database rows, or response. A slow query remains slow.

## Core settings

A production configuration often enables OPcache and allocates shared memory:

```ini
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
opcache.revalidate_freq=0
```

The values depend on the codebase and host memory. `validate_timestamps=0` avoids checking source timestamps on each request, but it requires an explicit invalidation or restart during deployment. Never edit deployed source in place and assume workers will see it.

When timestamp validation is enabled, `revalidate_freq` controls how often changes are checked. It is convenient for development but can serve old code for a window. Development and production should have deliberate, documented policies.

## Deployment and invalidation

Build the release in a separate directory or image, run tests and cache warmup, switch the active release atomically, then reload or restart workers according to the deployment policy. If old workers can serve the old release while new workers serve the new one, database and API changes must remain backward-compatible during the overlap.

`opcache_reset()` and `opcache_invalidate()` have operational and SAPI limitations; do not expose them as a public endpoint. A process reload is often the clearer deployment boundary. Verify the active release path and PHP configuration from an authenticated diagnostic, without exposing source or environment secrets.

## Memory and file limits

OPcache shared memory is finite. If the codebase, generated proxies, or vendor tree exceeds the configured capacity, scripts may not remain cached. Monitor used memory, free memory, cached scripts, hit rate, and restart or invalidation events. Increasing memory without checking the host's total budget can cause worker or host pressure.

`max_accelerated_files` should cover the deployed script set with headroom. The setting does not reserve that entire amount as PHP object memory, but a table that is too small can reduce caching effectiveness. Preloading and interned strings also consume resources and should be measured.

## Preloading and JIT

Preloading can load selected classes or functions when the PHP process starts. It couples the process to the loaded code and makes deployment invalidation and mutable assumptions more difficult. Use it only for measured, stable code and verify behavior after every PHP/runtime change.

The JIT compiler is not a universal web-performance switch. Typical web workloads spend substantial time in database and network waits, and JIT adds memory and tuning complexity. Benchmark representative production-shaped work before enabling it and keep a rollback path.

## Inspecting OPcache

The status API can support a protected diagnostic:

```php
<?php

declare(strict_types=1);

$status = function_exists('opcache_get_status')
    ? opcache_get_status(false)
    : false;

if (!is_array($status)) {
    throw new RuntimeException('OPcache status unavailable');
}

$memory = $status['memory_usage'] ?? [];
$statistics = $status['opcache_statistics'] ?? [];

return [
    'used_bytes' => $memory['used_memory'] ?? null,
    'free_bytes' => $memory['free_memory'] ?? null,
    'hit_rate' => $statistics['opcache_hit_rate'] ?? null,
];
```

Do not expose cached script paths, configuration, or status to unauthenticated users. A status snapshot is evidence for diagnosis, not a request to reset the cache on every alert.

## Testing and operations

Test deployment compatibility with old and new workers, cache warmup, invalidation, read-only releases, and rollback. Compare FPM and CLI configuration explicitly. Measure CPU time, request latency, OPcache hit rate, memory use, worker restarts, and cache resets before and after changes.

A low hit rate may indicate timestamp policy, insufficient memory, too many generated paths, or a deployment problem. A high hit rate does not prove correctness; stale code can be cached perfectly. Alert on stale release identifiers and failed reloads as well as cache capacity.

## Exercises

1. Measure a representative PHP endpoint with OPcache enabled and disabled in an isolated environment. Record CPU, latency, memory, and hit rate.
2. Design a deployment with `validate_timestamps=0`, atomic release switching, worker reload, and rollback.
3. Inspect the FPM and CLI `php.ini` files and explain any OPcache differences.

## Review questions

- What does OPcache cache and what does it not cache?
- Why does disabling timestamp validation change deployment requirements?
- Why can CLI observations differ from FPM behavior?
- When might preloading or JIT add more risk than benefit?
- Which metrics show capacity or stale-code problems?

## Summary

OPcache reuses compiled PHP scripts in shared memory, reducing repeated compilation work. Configure it against measured code and memory needs, make invalidation part of deployment, distinguish FPM from CLI, treat preloading and JIT as benchmarked choices, and monitor hit rate, capacity, restarts, and release identity.

## References

- [PHP manual: OPcache](https://www.php.net/manual/en/book.opcache.php)
- [PHP manual: OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)
- [PHP manual: opcache_get_status](https://www.php.net/manual/en/function.opcache-get-status.php)
- [PHP manual: Preloading](https://www.php.net/manual/en/opcache.preloading.php)
