# AI Summary — Chapter 57 — Garbage Collection

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Explains reference counting, unreachable cycles, the cycle collector's conceptual candidate/root traversal, `gc_collect_cycles()`, `gc_status()`, `gc_disable()`/`gc_enable()`, the distinction from allocator cache cleanup, strong owners, weak references, `WeakMap`, destructors, worker retention, recycle policies, performance, security, testing, exercises, and review questions.

## Concepts already explained

Prompt refcount reclamation, cyclic object graphs, reachability, roots, strong ownership, weak association, collector checkpoints, long-running worker retention, destructor boundaries, process recycling.

## Terminology established

Reference counting, cycle collector, candidate cycle, root, reachable, unreachable, strong reference, weak reference, retention, recycle policy.

## Examples used

Array and object cycles; listener registry ownership; `WeakMap`; a queue worker; destructor guidance; an unbounded static job list; bounded recent-job IDs.

## Cross-references

Links to Chapters 47, 51, 52, and 53 for refcounted values, references, copy-on-write, and object representation. Builds on Chapter 56's distinction between logical memory and RSS.

## Open threads

Later worker and runtime chapters should use ownership graphs and measured recycle policies when explaining process lifetime.

## Exact next section

Chapter 58 — Extensions: the Why This Matters section.

## Technical verification notes

Uses official PHP GC, `gc_collect_cycles()`, `gc_status()`, `WeakReference`, and `WeakMap` documentation plus `zend_gc.c`. GC counters, triggers, and internal data structures are identified as version-sensitive.
