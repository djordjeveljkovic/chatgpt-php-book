# AI Summary — Chapter 65 — Long-Running PHP Processes

- Status: complete
- Volume: Volume 5 — PHP RUNTIME
- Last updated: 2026-09-14

## Written material

Explains the process/job nested lifecycle for queue consumers, daemons, and streaming commands. Covers per-job scope, cumulative memory, garbage collection, stale connections, transactions, stale configuration/code, acknowledgments, at-least-once delivery, idempotency, retry/backoff, poison messages, shutdown/draining, deployment compatibility, security, performance, and testing.

## Concepts already explained

Long-running process, job boundary, bounded worker, process recycling, redelivery, acknowledgment ordering, durable idempotency, visibility/lease assumptions, poison message, drain, and stale dependency.

## Terminology established

Nested lifecycle, process-lifetime state, job-scoped state, retryable/permanent failure, graceful drain, and crash-safe redelivery.

## Examples used

Queue loop with classification/cleanup, signal-aware bounded worker, memory/job limits, deployment drain, poison-message and retry-storm scenarios, and connection/transaction failure cases.

## Cross-references

Builds on CLI Chapter 60, request lifecycle Chapter 63, worker processes Chapter 64, garbage collection Chapter 57, and points to signals in Chapter 66. References PCNTL, garbage collection, memory, and FastCGI manuals.

## Open threads

Signal delivery, async signal handling, and detailed graceful shutdown behavior are expanded in Chapter 66.

## Exact next section

None — chapter complete.

## Technical verification notes

PCNTL function availability is guarded in examples and linked to the PHP Manual. Queue delivery semantics are deliberately presented as application/broker contracts, not PHP guarantees.

## Writing notes

Treat process restart and duplicate delivery as normal recovery paths; never rely on `finally` or shutdown callbacks after forced termination.
