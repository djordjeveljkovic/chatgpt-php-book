# AI Summary — Chapter 83 — Heaps

- Status: complete
- Volume: Volume 6 — ALGORITHMS AND DATA STRUCTURES
- Last updated: 2026-09-15

## Written material

Completed and proofread chapter explains complete binary-tree shape, min/max heap invariants, zero-based array indexes, sift-up/sift-down, heapify, complexity, PHP SPL heaps, comparator laws, ties, corruption, top-k streaming, local scheduling, database/process boundaries, testing, failure cases, and selection trade-offs.

## Concepts already explained

Heap-order invariant, complete binary tree, extremum, sift up, sift down, heapify, bounded top-k, heap corruption and recovery, destructive iteration.

## Terminology established

Binary heap, min-heap, max-heap, parent, child, root, comparator, tie-breaker, stable order, priority queue, indexed heap, stale entry.

## Examples used

Min-heap array/index diagram; `SplMinHeap` peek and extraction; streaming largest-k integer scores; stable local task scheduler ordered by timestamp and sequence.

## Cross-references

Chapters 74 (memory), 77 (stacks), 78 (queues), 79 (sorting), 81 (binary search), 84 (priority queues), and 91 (data-structure selection). Official PHP Manual references cover `SplHeap`, `SplMinHeap`, `SplMaxHeap`, comparator behavior, empty access, iteration, and corruption recovery.

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

Exact SPL claims checked against the official PHP Manual on 2026-09-15, including min/max top behavior, the opposite sign conventions for custom `SplMinHeap::compare()` and `SplMaxHeap::compare()`, `RuntimeException` on empty `top()`/`extract()`, destructive `next()` iteration, arbitrary ordering for comparator ties, and possible invariant damage after comparator exceptions. All three PHP examples linted and behavior assertions passed on PHP 8.5.10. Additional checks confirmed timestamp-and-sequence task ordering and top-k results against sorted references for generated inputs and boundary values. Local links resolve.
