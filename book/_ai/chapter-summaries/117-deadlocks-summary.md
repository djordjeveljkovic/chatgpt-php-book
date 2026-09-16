# AI Summary — Chapter 117 — Deadlocks

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains waits-for cycles, canonical lock ordering, multi-table causes, fresh-transaction retries, idempotency, jittered backoff, database diagnostics, testing, exercises, and review questions.

## Concepts already explained

- A deadlock is a cycle of transactions waiting for resources held by one another; the database aborts a victim.
- Consistent ordering and short transactions reduce cycles.
- Recovery reruns the complete operation in a fresh transaction and requires bounded, replay-safe work.

## Terminology established

Deadlock, waits-for graph, deadlock victim, canonical lock order, deadlock classifier, bounded retry, jittered backoff.

## Examples used

- Canonically ordered account transfer with PDO.
- Generic bounded deadlock retry wrapper and two-connection integration test design.

## Cross-references

- [Chapter 116 — Locks](../../volumes/08-databases/116-locks.md)
- [Chapter 118 — Concurrency](../../volumes/08-databases/118-concurrency.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 118 — Concurrency: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated Volume VIII proofread. PostgreSQL and MySQL deadlock behavior is presented with engine-specific qualification and official references.
