# AI Summary — Chapter 291 — Debugging Production

- Status: complete
- Volume: Volume 20 — SENIOR ENGINEERING
- Last updated: 2026-09-16

## Written material

The chapter presents production debugging as controlled learning under pressure: stabilize harm, preserve evidence, define scope, separate symptoms from causes, build timelines, narrow hypotheses, correlate metrics/logs/traces, inspect PHP-FPM/CLI/worker and shared-resource limits, handle database/queue/cache/provider failures, treat completion as unknown, run bounded experiments, recover compatibly, communicate uncertainty, and verify service and data recovery.

## Concepts already explained

Production incident, stabilization, containment, evidence packet, impact scope, event time, ingestion time, hypothesis loop, evidence preservation, unknown completion, bounded experiment, failure domain, mixed-version diagnosis, forward recovery, recovery verification, and investigation record.

## Terminology established

Incident timeline, symptom/cause chain, evidence packet, hypothesis loop, observability map, boundary diagnostic table, safe-experiment record, recovery checklist, and handoff note.

## Examples used

Tenant-scoped catalog/search or reservation incident, PHP-FPM saturation, queue backlog, database locks, cache behavior, provider timeout, unknown external outcome, and mixed-version deployment.

## Cross-references

The chapter links to [Chapter 256 — PHP-FPM](../../volumes/17-production-engineering/256-php-fpm.md), [Chapter 260 — Logging](../../volumes/17-production-engineering/260-logging.md), [Chapter 261 — Metrics](../../volumes/17-production-engineering/261-metrics.md), [Chapter 262 — Tracing](../../volumes/17-production-engineering/262-tracing.md), [Chapter 263 — Health Checks](../../volumes/17-production-engineering/263-health-checks.md), [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md), [Chapter 269 — Incident Response](../../volumes/17-production-engineering/269-incident-response.md), [Chapter 280 — Tennis Reservation Service](../../volumes/19-small-engineering-projects/280-tennis-reservation-service.md), [Chapter 284 — Queue Worker](../../volumes/19-small-engineering-projects/284-queue-worker.md), [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md), [Chapter 288 — Code Review](../../volumes/20-senior-engineering/288-code-review.md), [Chapter 289 — Architecture Review](../../volumes/20-senior-engineering/289-architecture-review.md), and [Chapter 290 — Technical Debt](../../volumes/20-senior-engineering/290-technical-debt.md).

## Open threads

Continue with Chapter 292 — Performance Investigation, beginning with workload, bottleneck, and measurement discipline.

## Exact next section

Chapter 292 — Performance Investigation: the Why This Matters section.

## Technical verification notes

The chapter contains one illustrative PHP memory-observation function. PHP 8.5.10 linted the example, local Markdown links and required handoff files resolved, and `git diff --check` passed. The chapter was independently proofread for stabilization, evidence preservation, hypothesis testing, PHP/runtime limits, unknown completion, recovery, communication, and the Chapter 292 handoff.

## Writing notes

Keep this summary short and update it after every writing session.
