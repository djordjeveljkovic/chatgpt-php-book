---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 224
title: Measuring Performance
slug: measuring-performance
status: complete
summary: ../../_ai/chapter-summaries/224-measuring-performance-summary.md
---

# Chapter 224 — Measuring Performance

## Why This Matters

Performance decisions need evidence. A developer's local request, a single timer, or an average from a synthetic benchmark can miss database waits, queueing, cold starts, cache misses, and tail latency. Measuring performance means choosing a workload, defining a metric, collecting it consistently, and interpreting it with enough context to guide a safe change.

Measure before and after a change with the same assumptions. Preserve a baseline and record runtime, PHP version, dependencies, data volume, concurrency, cache state, and database configuration.

## Metrics and Dimensions

Useful metrics include:

* latency distributions such as p50, p95, and p99;
* throughput and completed work;
* error, timeout, and retry rate;
* CPU time, wall time, memory peak, and worker lifetime;
* database query count, duration, rows, locks, and pool wait;
* cache hit ratio, eviction, freshness, and stampede rate;
* queue depth, oldest message age, throughput, and handler duration.

Dimensions help locate a cause: route, operation, status, dependency, deployment version, tenant class, or payload size. Keep dimensions bounded. Never label a metric with raw user IDs, email addresses, tokens, URLs containing secrets, or unbounded exception messages.

Distinguish wall time from CPU time. Wall time includes I/O and scheduling; CPU time helps identify computation. A database span can consume wall time while PHP uses little CPU. Record both where the measurement system supports it.

## Instrument Stable Boundaries

Instrument at request, use-case, dependency, and worker boundaries. A timer should include a clear start and end and record outcome even when an exception occurs:

~~~php
<?php

declare(strict_types=1);

function timeOperation(string $operation, callable $work): mixed
{
    $started = hrtime(true);

    try {
        return $work();
    } catch (Throwable $exception) {
        recordCounter($operation . '.errors', 1);

        throw $exception;
    } finally {
        recordHistogram(
            $operation . '.duration_ms',
            (hrtime(true) - $started) / 1_000_000,
        );
    }
}
~~~

The names recordCounter and recordHistogram represent an application's metrics adapter. Keep instrumentation failures from breaking the business operation unless the metric is itself a required audit or compliance record. Avoid measuring only successful requests; failures and timeouts often have the greatest operational cost.

## Traces and Correlation

Metrics show shape; traces show causal paths. Propagate a correlation or trace context through HTTP calls, database spans, queue messages, and logs. A trace can show that p99 latency comes from a lock wait, a provider call, or a serialization step.

Do not put sensitive payloads into spans. Record route templates instead of full URLs, selected status and sizes, and safe identifiers. Sampling should retain errors and slow traces, while head or tail sampling policy depends on the tracing system and traffic volume.

## Load and Workload Shape

A performance result is meaningful only for a defined workload:

* request mix and payload sizes;
* data cardinality and distribution;
* concurrency and arrival pattern;
* cache warmness and expiration;
* database engine, indexes, and isolation;
* provider latency and failure rates;
* worker count, PHP-FPM limits, and CPU/memory capacity.

A steady load test can miss burst queueing. A benchmark with one row can miss a query plan that degrades at a million rows. Run baseline, steady, burst, and failure scenarios where the risk warrants it. Use production traffic copies only after scrubbing sensitive data and obtaining the required approval.

## Experiment Design

Change one major factor at a time, keep the environment comparable, and repeat enough runs to separate signal from noise. Randomize or alternate variants when a shared environment changes over time. Report distribution and confidence or variation, not only the fastest run.

Warm-up matters for OPcache, connection pools, caches, JIT settings, and branch or filesystem state. Measure cold and warm behavior separately if users experience both. A benchmark that includes setup in one variant and excludes it in another is not a fair comparison.

## Budgets and Regression Detection

Turn service objectives into checks: p95 route latency under a workload, maximum queue age, memory per worker, or query count per feature. Alert on sustained budget violations, error correlation, and saturation. A single slow request is a debugging signal; a trend is an operational problem.

Performance regression tests should use stable fixtures and tolerances wide enough for normal variance. Do not fail every build because a shared CI host fluctuated by one percent. Compare trends, flag large changes, and investigate with profiling or traces.

## Tooling and Privacy

Application metrics, database statistics, web-server logs, PHP-FPM status, profilers, and distributed tracing answer different questions. Combine them instead of asking one tool to explain all behavior. Keep retention and access controls appropriate for telemetry, and redact cookies, authorization headers, personal data, and request bodies.

A measurement pipeline can become a performance dependency. Batch or sample telemetry, bound queue sizes, and define behavior when the metrics backend is unavailable. Instrumentation must not add unbounded synchronous work to every request.

## Failure and Threat Analysis

* **Unrepresentative workload:** one small fixture hides scale behavior. Model cardinality and request mix.
* **Average-only reporting:** tail latency and timeouts disappear. Use distributions and slow traces.
* **High-cardinality labels:** metrics become expensive and unusable. Use templates and bounded categories.
* **Instrumentation failure:** telemetry breaks requests. Use a bounded non-blocking path.
* **Warm/cold confusion:** cache or OPcache state changes results. Separate and document scenarios.
* **Data leakage:** traces capture secrets or personal data. Redact at the instrumentation boundary.
* **Benchmark drift:** runtime, database, or dependency changes invalidate comparisons. Record environment metadata.

## Testing Measurement

Test metric naming, outcome recording, redaction, sampling, and correlation propagation at the adapter boundary. Integration-test trace and metric exporters where their protocol matters. Use a fake clock or deterministic exporter in unit tests and verify a failed exporter cannot grant access or corrupt a transaction.

## Exercises

1. Define a metric and trace schema for a checkout request with database and payment spans.
2. Create a workload matrix varying payload size, concurrency, cache state, and data cardinality.
3. Add a regression check for p95 latency or query count with a tolerance and documented environment.
4. Review telemetry for high-cardinality or sensitive labels and replace them with bounded dimensions.

## Review Questions

1. What context makes a performance measurement meaningful?
2. Why should wall and CPU time be distinguished?
3. What do metrics reveal that traces do not, and vice versa?
4. Which workload variables commonly change results?
5. Why must telemetry be bounded and redacted?
6. How should normal CI variance affect regression thresholds?

## Summary

Measure defined workloads with distributions, bounded dimensions, stable baselines, and recorded environment context. Instrument requests, dependencies, workers, metrics, traces, and queueing while preserving privacy and availability. Use budgets and regression trends to decide when profiling or code changes are justified.

## References

- [PHP Manual: hrtime](https://www.php.net/manual/en/function.hrtime.php)
- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)
- [OpenTelemetry PHP](https://opentelemetry.io/docs/languages/php/)
- [Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/)

