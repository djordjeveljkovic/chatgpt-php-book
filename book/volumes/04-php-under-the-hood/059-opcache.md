---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 59
title: OPcache
slug: opcache
status: complete
summary: ../../_ai/chapter-summaries/059-opcache-summary.md
---

# Chapter 59 — OPcache

## Why This Matters

PHP source is not normally parsed and compiled from scratch for every request. OPcache stores compiled script data in shared memory so later requests can execute cached opcodes. This removes repeated lexing, parsing, and compilation work, but it introduces a cache lifecycle into deployment: source changes must become visible to every worker, cached memory must have capacity, and configuration must match the SAPI that serves traffic.

OPcache is therefore both a performance feature and an operational dependency. A deployment that updates files while workers execute stale cached code can produce mixed behavior. A deployment that resets a shared cache at the wrong time can cause a compile stampede. A deployment with too little shared memory can waste the feature or fail to cache important scripts.

## Mental Model

```text
PHP source file
    → tokenize / parse / compile
    → opcodes and associated immutable script data
    → OPcache shared memory
    → worker executes cached code
```

The conceptual cache is shared by PHP processes on the same host when OPcache is configured for shared memory. Workers still have process-local runtime state, request data, stacks, and mutable values. OPcache does not share ordinary PHP variables between requests and does not replace an application cache such as Redis.

The exact representation, optimizer passes, hash tables, and memory layout are implementation details. The stable operational contract is that OPcache caches compiled scripts and decides whether a cached script is valid according to configuration and deployment actions.

## Core Concept: Compilation and Invalidation

On a cache miss, the engine reads a script, compiles it, and OPcache may store the resulting representation. On a hit, the worker can use the cached compiled form. A simplified decision is:

```text
cached key exists?
    no  → compile → store if there is capacity
    yes → is cached code valid under policy?
              yes → execute cached form
              no  → recompile / replace according to policy
```

`opcache.validate_timestamps` controls whether OPcache checks script timestamps. When timestamp validation is disabled, changed files are not automatically noticed; a controlled reset or process restart is required after deployment. When validation is enabled, `opcache.revalidate_freq` influences how often checks occur. Defaults and distribution configuration vary, so inspect `opcache_get_configuration()` on the target runtime rather than assuming a manual default is your production setting.

A cache key is affected by path and configuration such as `opcache.use_cwd` and `opcache.revalidate_path`. Releases that use different physical paths can therefore create distinct cache entries. An atomic release strategy plus a deliberate worker/cache lifecycle is easier to reason about than editing files in place.

## What PHP Does

Inspect OPcache from the same SAPI used by the workload:

```php
<?php

declare(strict_types=1);

if (function_exists('opcache_get_status')) {
    $status = opcache_get_status(false);
    $configuration = opcache_get_configuration();

    var_dump([
        'enabled' => $status['opcache_enabled'] ?? null,
        'cache_full' => $status['cache_full'] ?? null,
        'memory_usage' => $status['memory_usage'] ?? null,
        'configuration' => $configuration['directives'] ?? [],
    ]);
}
```

These functions are commonly disabled or restricted in production-facing contexts. A diagnostic endpoint must be authenticated and non-public because paths, directives, and script information can disclose deployment details. Prefer CLI/admin tooling or a metrics collector with controlled access.

Useful commands are:

```text
php --ini
php --ri opcache
php -i | grep -i '^opcache\.'
```

The CLI may use a separate configuration and often has a different `opcache.enable_cli` value from FPM. Verify both when a CLI worker or migration process matters.

## Minimal Example: A Cache-Aware Deployment

A safe conceptual sequence is:

```text
build immutable release directory
    → run syntax, integration, and smoke tests
    → make the new release visible atomically
    → reload/restart workers or perform a controlled OPcache reset
    → warm critical scripts if justified
    → verify version marker and cache status
```

The version marker can be a committed build identifier exposed through an authenticated health or diagnostics path:

```php
final class BuildInfo
{
    public function __construct(private readonly string $commit) {}

    public function commit(): string
    {
        return $this->commit;
    }
}
```

A response header or log field containing the build identifier helps detect mixed workers. Do not use a writable web endpoint to call `opcache_reset()` as a routine deployment mechanism; it expands the attack surface and can create a synchronized compile storm.

## Practical Example: Configuration Choices

Common settings represent different trade-offs:

| Setting | Operational question |
| --- | --- |
| `opcache.enable` | Should the SAPI use OPcache? |
| `opcache.enable_cli` | Should CLI workers/scripts use it? |
| `opcache.memory_consumption` | How much shared memory is available for cached data? |
| `opcache.interned_strings_buffer` | How much shared memory is reserved for interned strings? |
| `opcache.max_accelerated_files` | How many script entries can be indexed? |
| `opcache.validate_timestamps` | Should source changes be discovered automatically? |
| `opcache.revalidate_freq` | How often should timestamp checks occur when enabled? |
| `opcache.preload` | Should a preload script load persistent code at process startup? |
| `opcache.jit` / `opcache.jit_buffer_size` | Is JIT configured, and is it justified by workload evidence? |

The values interact with application size, deployment style, PHP version, OS shared-memory limits, and worker topology. Increasing `memory_consumption` does not fix a process RSS leak in application code, and enabling JIT does not make network or database latency disappear.

## Preloading and JIT

Preloading was introduced in PHP 7.4. A configured preload script runs when the PHP process starts and can make selected classes and functions persist in the process. It is powerful but couples deployment to process restart and requires code to be suitable for the preload model. It is not a general replacement for OPcache invalidation, and a preload script must be revised and reloaded deliberately when code changes.

JIT became part of OPcache in PHP 8.0. It can compile selected hot paths to native instructions, but typical web applications are often dominated by database, network, templating, and framework work. Benchmark a representative CPU-bound workload before enabling it. Treat JIT settings as version- and CPU-sensitive; record them with benchmark results and verify the production SAPI actually uses the intended configuration.

These features are distinct:

```text
OPcache: cache compiled PHP representations
Preloading: make selected code available at process startup
JIT: optionally compile hot VM paths further
```

## Production Example: Immutable Releases

Suppose release A is active and release B is built. An in-place copy can expose a half-updated tree: one request may read a new class while another reads an old included file. OPcache validation timing adds another variable. A safer layout is:

```text
/srv/app/releases/2026-09-14-1200/
/srv/app/releases/2026-09-14-1300/
/srv/app/current → /srv/app/releases/2026-09-14-1300
```

Build and test the immutable directory, switch `current` atomically, and reload or gracefully restart workers according to the deployment policy. Verify that new workers resolve the new physical path and that old workers drain. A rollback points the link back and repeats the worker/cache verification; it is not enough to change the symlink if old workers keep executing old compiled code.

The exact symlink and cache-key behavior depends on configuration. The operational rule is to make the release path, invalidation action, and worker lifecycle an explicit tested sequence.

## Bad Example

```ini
; Production
opcache.validate_timestamps=0
```

This can be a valid production choice for immutable deployments, but it is unsafe when paired with “copy files into the live directory and hope workers notice.” The problem is the deployment contract, not the directive in isolation.

## Better Example

Document the mode with its required action:

```text
Production OPcache contract
- release directories are immutable
- source is never edited in the active release
- workers are gracefully reloaded after activation
- a build marker is checked by smoke tests
- rollback reloads workers as well
- cache status and shared-memory capacity are monitored
```

For a development environment, timestamp validation can make iteration convenient. For production, disabling timestamp checks can avoid filesystem checks and make the deployed artifact deterministic, provided restarts/reloads are reliable. There is no universal setting independent of deployment design.

## Performance

OPcache primarily saves repeated compilation work. Its benefit depends on script count, request rate, code size, cache hit behavior, filesystem latency, and the cost of the rest of the request. Shared-memory lookups and optimization also have costs, and an undersized cache can cause churn or leave scripts uncached.

Monitor cache hits, misses, wasted memory, free memory, number of cached scripts, restarts, and request latency. Use `opcache_get_status()` for diagnostics where permitted, and correlate with deployment events. Warmup can reduce first-request latency after a restart, but warming every possible route can consume capacity and hide invalidation mistakes.

Do not benchmark a cold CLI invocation against a warm FPM pool and call the difference “the application.” Run cold, warm, and post-deployment scenarios separately.

## Security and Reliability

OPcache does not make source code secret. It does not isolate tenants, encrypt data, or replace filesystem permissions. A compromised process can still access application state and possibly diagnostics. Restrict status/reset functions, protect configuration files, and keep release directories readable only as needed.

A cache inconsistency can become a correctness or security incident when authorization code, route definitions, or configuration changes are mixed across workers. Make deployment verification include a version marker, critical endpoint behavior, and worker drain status. Never use an uncoordinated cache reset from an attacker-reachable request.

## Testing and Verification

Test OPcache as an environment matrix:

1. Confirm the module is loaded in FPM and, separately, CLI if relevant.
2. Execute a script twice and observe warm behavior in a controlled environment.
3. Change a script under timestamp validation and verify the documented revalidation behavior.
4. With validation disabled, change a script and verify that a restart/reset is required.
5. Deploy two immutable releases and assert that every worker reports the intended build marker.
6. Fill a test cache near capacity and observe status, warnings, misses, and latency.
7. If using preloading or JIT, test startup, reload, rollback, and representative performance.

Do not assert exact cache counters across PHP versions or distributions. Record PHP version, SAPI, OPcache configuration, filesystem layout, and worker lifecycle with test results. Ensure tests cannot reset a shared production cache.

## Common Mistakes

- Assuming OPcache shares ordinary PHP variables.
- Assuming CLI and FPM have the same OPcache configuration.
- Disabling timestamp validation without a restart/invalidation deployment step.
- Editing files in place during a live release.
- Treating `opcache_reset()` as harmless request code.
- Increasing cache memory without monitoring shared-memory capacity and script count.
- Enabling JIT or preloading without a representative benchmark and restart policy.
- Measuring only cold starts or only warm requests.

## Senior Engineer Thinking

OPcache is a cache with a lifecycle, not a switch. Design its invalidation protocol at the same time as the deployment protocol. Define what “new code is live” means, how old workers drain, how rollback works, and what evidence proves all workers agree. Then measure whether the saved compilation cost matters for the workload.

## Exercises

1. Compare cold and warm execution of a small application under CLI and FPM. Record the configuration for each.
2. Implement a build-marker endpoint and write a smoke test that detects mixed releases across workers.
3. Model a deployment with `validate_timestamps=0`. List every event required for activation and rollback.
4. Fill OPcache in a disposable environment with many scripts. Interpret the status output and choose a capacity change.
5. Benchmark a CPU-bound PHP loop with and without JIT on the target PHP build, then explain why the result may not predict web latency.

## Review Questions

1. What does OPcache cache, and what does it not share?
2. How do timestamp validation and explicit invalidation differ operationally?
3. Why must PHP CLI and FPM be inspected separately?
4. What deployment properties make disabled timestamp validation safe?
5. How do preloading and JIT differ from ordinary opcode caching?
6. Which signals indicate an undersized or churning OPcache?

## Summary

OPcache stores compiled PHP representations in shared memory and can remove repeated compilation work across workers. It introduces cache capacity, configuration, invalidation, preload, JIT, and deployment-lifecycle concerns. Treat OPcache as part of the release protocol: use immutable artifacts, deliberate worker reloads, protected diagnostics, environment-specific verification, and measurements that distinguish cold from warm behavior.

## Official References

- [PHP OPcache manual](https://www.php.net/manual/en/book.opcache.php)
- [OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)
- [OPcache status and configuration functions](https://www.php.net/manual/en/ref.opcache.php)
- [OPcache preloading](https://www.php.net/manual/en/opcache.preloading.php)
- [OPcache JIT](https://www.php.net/manual/en/opcache.jit.php)
- [PHP source: OPcache](https://github.com/php/php-src/tree/master/ext/opcache)
