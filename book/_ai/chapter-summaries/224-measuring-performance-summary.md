# AI Summary — Chapter 224 — Measuring Performance

- Status: complete
- Volume: Volume 15 — PERFORMANCE
- Last updated: 2026-09-16

## Written material

Covers performance metrics and dimensions, wall versus CPU time, instrumentation, traces, workload shape, experiment design, budgets, regression detection, privacy, and telemetry failure.

## Concepts already explained

Measurements need workload, environment, data, cache, concurrency, and runtime context. Use bounded dimensions and redacted telemetry; metrics and traces answer different questions.

## Terminology established

Latency distribution, p95, p99, wall time, CPU time, high-cardinality label, trace context, workload matrix, telemetry budget.

## Examples used

PHP operation timer, metric/trace schema, workload matrix, and regression checks.

## Cross-references

- [Chapter 223 — Performance Mental Model](../../volumes/15-performance/223-performance-mental-model.md)
- [Chapter 225 — Benchmarking](../../volumes/15-performance/225-benchmarking.md)

## Exact next section

Chapter 225 — Benchmarking: the Why This Matters section.

## Technical verification notes

PHP timing example linted; OpenTelemetry and Prometheus references are linked.
