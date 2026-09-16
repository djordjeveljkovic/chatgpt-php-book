# AI Summary — Chapter 226 — Profiling

- Status: complete
- Volume: Volume 15 — PERFORMANCE
- Last updated: 2026-09-16

## Written material

Covers sampling and instrumentation profilers, representative workloads, phase timing, Xdebug, Blackfire, traces, profile interpretation, memory retention, privacy, and optimization workflow.

## Concepts already explained

Profiles identify dominant CPU, allocation, call-count, and wait costs only in a defined workload. Protect profile data and revalidate latency, memory, errors, saturation, cost, and correctness after changes.

## Terminology established

Sampling profiler, instrumentation profiler, inclusive time, self time, call-count explosion, phase timer, profile baseline, worker retention.

## Examples used

PHP PhaseTimer, Xdebug/Blackfire workflow, database and trace correlation, and multi-job worker profiling.

## Cross-references

- [Chapter 225 — Benchmarking](../../volumes/15-performance/225-benchmarking.md)
- [Chapter 232 — PHP-FPM](../../volumes/15-performance/232-php-fpm.md)

## Exact next section

Chapter 227 — CPU: the Why This Matters section.

## Technical verification notes

PHP profiling timer linted; Xdebug, Blackfire, OpenTelemetry, and PHP-FPM references are linked.
