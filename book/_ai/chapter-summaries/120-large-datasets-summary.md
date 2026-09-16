# AI Summary — Chapter 120 — Large Datasets

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains memory and driver buffering, generator-based streaming, keyset batch checkpoints, bounded write transactions, bulk loading, consistency while rows change, backpressure, idempotent retries, and operational observability.

## Concepts already explained

- Large-data work should target `O(batch size)` working memory.
- Streaming does not remove driver buffering, transaction retention, or downstream backpressure.
- Keyset checkpoints make resumable work predictable; checkpoints advance only with committed idempotent side effects.
- Set-based SQL reduces data movement but can increase locks, logs, and replication pressure.

## Terminology established

Working set, forward-only stream, keyset batch, checkpoint, backfill, cutoff, backpressure, idempotent retry, durable export.

## Examples used

- PDO generator for users and streamed CSV output.
- Keyset batch query and bounded update transaction.

## Cross-references

- [Chapter 110 — Query Plans](../../volumes/08-databases/110-query-plans.md)
- [Chapter 119 — Pagination](../../volumes/08-databases/119-pagination.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 121 — ORMs: the Why This Matters section.

## Technical verification notes

PHP examples use modern syntax and PDO APIs; driver buffering, `COPY`, and bulk-loading semantics are explicitly qualified as engine/driver-specific.
