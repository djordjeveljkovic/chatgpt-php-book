# AI Summary — Chapter 84 — Priority Queues

- Status: complete
- Volume: Volume 6 — ALGORITHMS AND DATA STRUCTURES
- Last updated: 2026-09-15

## Written material

Completed and proofread chapter covers the priority-queue contract, ordering and tie policies, update/cancellation via lazy invalidation, scan/sort/heap trade-offs, `SplPriorityQueue` flags and consuming iteration, clone behavior, deterministic tie-breaking, process-local scheduling, top-k selection, complexity, operational boundaries, tests, mistakes, exercises, review questions, and summary. The subclass examples use PHP 8.2+'s literal `true` return type.

## Concepts already explained

Priority ordering, max-heap and min-heap queue policies, tie-breaking, starvation, lazy invalidation, bounded top-k selection, and the distinction between an in-process priority queue and durable job scheduling.

## Terminology established

Peek observes the next entry without removing it; extract returns and removes it. `SplPriorityQueue` is a max-heap by default. Equal priorities have undefined extraction order unless the application adds a tie-breaker. `EXTR_DATA`, `EXTR_PRIORITY`, and `EXTR_BOTH` select returned data shape. Iterator advancement consumes entries.

## Examples used

`SplPriorityQueue` processing urgency-ranked jobs; a stable integer-priority subclass; version-based lazy invalidation guidance; a reversed-comparison integer min-queue retaining the largest k scores from an iterable; clone-before-drain pattern.

## Cross-references

Chapters 74 (memory complexity), 78 (queues), 79 (sorting), 83 (heaps), 91 (data-structure selection), and 244 (durable queues).

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

Checked official PHP Manual behavior for max-heap ordering, duplicate-priority instability, insert/compare/extraction flags, `top()`, `extract()`, `next()` consuming the top node, iterator `key()`, and comparison exceptions potentially corrupting a heap. All five PHP blocks linted on PHP 8.5.10. Runtime checks passed for extraction flags, max-heap ordering, stable ties, min-heap top-k selection, independent clone container state with shared object payloads, and consuming iteration; 13,500 generated top-k cases matched sorted references. The documented subclass return types require PHP 8.2 or later. All local Markdown links resolve.

## Writing notes

Keep heap implementation mechanics in Chapter 83. Preserve the distinction between queue ordering and job durability/fairness.
