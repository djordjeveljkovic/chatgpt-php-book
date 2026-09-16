---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 226
title: Profiling
slug: profiling
status: complete
summary: ../../_ai/chapter-summaries/226-profiling-summary.md
---

# Chapter 226 — Profiling

## Why This Matters

Profiling identifies where a program spends time, allocates memory, waits, or repeats work. It turns “the endpoint is slow” into evidence about a function, query, lock, provider call, serialization step, or queue. Profiling is most useful after a reproducible slow case and a defined performance target exist.

A profiler is an observation tool. It changes overhead and may expose sensitive values, so profile representative workloads in a safe environment and protect the output.

## Sampling and Instrumentation

A sampling profiler periodically records the current call stack. It has lower overhead and is useful for CPU hotspots in a running process. An instrumenting profiler records function entry and exit, which can provide detailed call counts and durations at higher overhead. A trace or span profiler shows time across HTTP, database, queues, and providers.

Use sampling for broad production-like diagnosis when supported, instrumentation for a focused local case, and traces for distributed waits. No tool alone can distinguish every CPU, allocation, lock, and I/O cause.

## Profile the Real Path

Start with a slow request, job, or command and record:

* route or operation and a safe correlation ID;
* PHP version, extensions, OPcache/JIT state, and worker type;
* input size, data cardinality, cache state, and concurrency;
* database plan, query count, lock time, and provider spans;
* latency distribution, errors, memory, and queue age.

Reproduce the case with synthetic or scrubbed data. Do not copy production passwords, tokens, personal records, or payment details into a profiling environment. A profile of an empty database or a warm local cache can point to the wrong optimization.

## A Small In-Process Profile

For a bounded local experiment, mark phases and record elapsed time and memory:

~~~php
<?php

declare(strict_types=1);

final class PhaseTimer
{
    /** @var array<string, float> */
    private array $started = [];

    /** @var array<string, float> */
    private array $elapsedMs = [];

    public function start(string $phase): void
    {
        $this->started[$phase] = hrtime(true) / 1_000_000;
    }

    public function stop(string $phase): void
    {
        if (!isset($this->started[$phase])) {
            throw new LogicException('Unknown phase');
        }

        $this->elapsedMs[$phase] =
            (hrtime(true) / 1_000_000) - $this->started[$phase];
    }

    /** @return array<string, float> */
    public function report(): array
    {
        return $this->elapsedMs;
    }
}
~~~

This phase timer can show whether parsing, domain work, SQL, or serialization dominates, but it cannot show call-stack detail or concurrency. Keep it bounded, disable or sample it in normal traffic, and ensure a failed phase cannot leak input data into logs.

## Xdebug, Blackfire, and Traces

Xdebug provides development debugging and profiling features that can produce cachegrind-compatible output. Enable profiling intentionally; its overhead and output volume make it unsuitable as an always-on production setting.

Blackfire and similar profilers can combine call graphs, wall time, CPU, memory, and assertions for controlled environments. Distributed tracing systems and OpenTelemetry connect application spans to downstream services. Tool names and setup differ by PHP version and deployment; follow current vendor documentation and compare profiles from the same environment.

A database's EXPLAIN plan, slow-query log, PHP-FPM status, and queue metrics complement a PHP profile. If the top PHP frame is waiting on a driver, inspect the dependency rather than optimizing that frame.

## Reading a Profile

Look for:

* high self time: work inside one function;
* high inclusive time: the function or its children dominate;
* call-count explosions: loops, N+1 queries, or repeated parsing;
* allocation and retained memory: large arrays, copies, or leaks;
* blocking spans: SQL, locks, filesystem, network, or queue waits;
* cold-start work: autoloading, bootstrap, cache misses;
* tail-only paths: retries, contention, or large payloads.

Change one cause, then re-profile the same workload. A shorter function can move cost elsewhere. A cache can reduce median time while increasing stale-data or stampede risk. A parallel call can lower wall time while increasing provider load and failure paths.

## Memory and Long-Running Workers

A request profile may hide retention in a queue worker. Profile multiple jobs in one process and compare memory after each job. Large arrays, static caches, closures retaining graphs, and unclosed resources can cause worker growth. Set a restart policy and fix ownership rather than relying only on restarts.

PHP's memory reports are useful clues but do not represent every operating-system allocation. Compare process RSS, PHP memory, request count, and worker lifetime where possible.

## Security and Privacy

Profiles can contain function arguments, SQL, URLs, class names, file paths, and user data. Restrict access, encrypt storage where required, set retention, and scrub or sample sensitive values. Never enable a remote profiler against an untrusted endpoint without authentication and network controls.

Do not profile a production operation that triggers payment, email, deletion, or other irreversible side effects unless the experiment is explicitly safe. Use a fake or a read-only path.

## Failure and Threat Analysis

* **Wrong environment:** local profile omits queueing or I/O. Reproduce workload and inspect distributed traces.
* **Profiler overhead:** instrumentation changes timings. Compare sampling, tracing, and baseline.
* **Sensitive output:** arguments or SQL enter profile files. Scrub and restrict access.
* **Top-frame confusion:** a driver wait is blamed on PHP code. Inspect spans and database plans.
* **Call-count explosion:** repeated query or serialization cost grows with data. Profile representative cardinality.
* **Worker retention:** memory grows only after many jobs. Profile process reuse and reset state.
* **Optimization regression:** faster code increases load or weakens cache consistency. Re-measure capacity and correctness.

## Profiling Workflow

1. Define the symptom and target.
2. Capture a baseline metric and representative trace.
3. Profile the smallest reproducible path.
4. Identify the dominant cost or wait.
5. Make one bounded change.
6. Re-run correctness, load, and security checks.
7. Compare latency, memory, errors, saturation, and cost.
8. Keep or revert the change with evidence.

Do not optimize a random hot-looking function. Confirm that it contributes to the user-visible budget and that the proposed change preserves authorization, consistency, and failure behavior.

## Testing and Operations

Add regression tests for query count, payload size, algorithmic behavior, memory ceilings, and timeout classification where profiling found a defect. Use performance tests for the service-level budget and integration tests for database or provider behavior. Keep profiling output out of normal application logs and delete it according to retention policy.

## Exercises

1. Add phase timing to a request and identify whether parsing, database, or serialization dominates.
2. Profile a representative loop and compare call count and memory before and after an optimization.
3. Run multiple jobs in one PHP worker and graph memory after each job.
4. Design a safe production-like profiling experiment that cannot send email or charge a payment.

## Review Questions

1. How do sampling and instrumentation profiling differ?
2. Why should a profile include workload and environment metadata?
3. What does inclusive time reveal?
4. How can a profiler point to a driver while the real issue is a database lock?
5. Why must long-running workers be profiled across multiple jobs?
6. Which profiling data needs security and retention controls?

## Summary

Profiling finds dominant CPU, allocation, call-count, and wait costs in a defined workload. Use sampling, instrumentation, traces, database plans, and worker metrics together; protect profile data; re-profile after one bounded change; and validate latency, memory, errors, saturation, cost, and correctness before keeping an optimization.

## References

- [Xdebug Profiling](https://xdebug.org/docs/profiler)
- [Blackfire PHP Profiler](https://blackfire.io/docs/php/integrations)
- [OpenTelemetry PHP](https://opentelemetry.io/docs/languages/php/)
- [PHP Manual: hrtime](https://www.php.net/manual/en/function.hrtime.php)
- [PHP-FPM Status](https://www.php.net/manual/en/install.fpm.configuration.php)

