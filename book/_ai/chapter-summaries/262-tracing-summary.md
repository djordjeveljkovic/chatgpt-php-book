# AI Summary — Chapter 262 — Tracing

- Status: complete
- Volume: Volume 17 — PRODUCTION ENGINEERING
- Last updated: 2026-09-16

## Written material

Explains distributed tracing as causal evidence across PHP-FPM, application code, databases, providers, queues, and workers. Covers traces, spans, kinds, parent/child relationships, links, trace context propagation, request/trace/operation/message identity, PHP request and worker lifetimes, framework-neutral span lifecycle helpers, asynchronous workflows, retries, partial failure, sampling, bounded attributes and events, correlation with logs and metrics, performance, security/privacy, database and messaging boundaries, concurrency, testing, common mistakes, and senior operational reasoning.

## Concepts already explained

Trace, span, trace ID, span ID, span kind, parent span, span link, trace context, carrier, propagation, baggage boundary, server/client/producer/consumer/internal span, span status, span event, logical operation span, attempt span, sampling, head sampling, parent-based sampling, tail sampling, current context, exporter failure, clock skew, and trace completeness.

## Terminology established

Causal path, meaningful span boundary, context carrier, propagation boundary, diagnostic workflow, representative trace, sampled absence, logical-versus-physical work, bounded span attribute, trace retention boundary, and observability evidence boundary.

## Examples used

Trace tree, HTTP and queue propagation diagrams, typed Span and Tracer interfaces, exception-safe inSpan helper, nested checkout/billing spans, request-to-queue trace, retry attempt spans, sampling policies, safe span attributes, signal comparison table, and worker cleanup tests.

## Cross-references

Chapters 224, 238, 260, 261, and 263.

## Open threads

Continue Volume XVII with Chapter 263 on health checks, carrying forward explicit readiness contracts, process state, dependency health, metrics, traces, logs, and failure isolation.

## Exact next section

Chapter 263 — Health Checks: the Why This Matters section.

## Technical verification notes

PHP examples were linted with PHP 8.5.10. Local Markdown links resolved and git diff --check passed. Tracing, propagation, sampling, and HTTP semantic-convention claims were checked against current official OpenTelemetry documentation on 2026-09-16. No live collector, exporter, queue, database, PHP-FPM, or framework integration was run.

## Writing notes

Separates trace context from authentication, cancellation, request identity, operation identity, and durable business evidence. Treats sampling, exporter behavior, worker context, attribute cardinality, clock skew, and privacy as production constraints.
