# AI Summary — Chapter 78 — Queues

- Status: complete
- Volume: Volume 6 — ALGORITHMS AND DATA STRUCTURES
- Last updated: 2026-09-15

## Written material

Defines FIFO semantics and queue invariants; compares PHP arrays using `array_shift()`, a compacting head-index queue, and `SplQueue`; implements the custom queue; applies FIFO to breadth-first search and an in-process work buffer; distinguishes in-memory structures from durable queues; discusses duplicate delivery, retries, backpressure, capacity, testing, memory, and security.

## Concepts already explained

FIFO, producer, consumer, worker, queue capacity, backpressure, idempotency, bounded retry, poison message, dead-letter queue, visibility timeout, duplicate delivery, and breadth-first search.

## Terminology established

Enqueue at the rear; dequeue from the front; FIFO preserves removal order on a single queue. Queue structure does not imply durability, process sharing, retry policy, or completion order under concurrency.

## Examples used

`array_shift()` FIFO snippet; `ArrayQueue` with head/tail indices, empty checks, `null` payload support, reset, and compaction; `SplQueue`; graph BFS; process-local email-job buffer.

## Cross-references

Chapters 75–77 explain arrays, sets, and stacks; Chapter 84 covers priority queues; Chapter 244 covers durable/distributed queues.

## Open threads

Chapter 79 — Sorting follows. Preserve the distinction between FIFO data-structure semantics and broker delivery guarantees.

## Exact next section

Chapter complete; Chapter 79 — Sorting: the Why This Matters section.

## Technical verification notes

Checked official PHP Manual pages for `array_shift()` (numeric key reindexing; `null` on empty input), `SplQueue` (doubly linked list/FIFO mode), `SplQueue::enqueue()`, `SplQueue::dequeue()`, and SPL endpoint complexity. The Manual does not promise `array_shift()` complexity; the chapter states that distinction explicitly.

## Writing notes

Maintain the distinction between FIFO data-structure semantics and broker delivery guarantees.
