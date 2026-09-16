---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 302
title: Algorithm Guide
slug: algorithm-guide
status: complete
summary: ../../_ai/chapter-summaries/302-algorithm-guide-summary.md
---

# Chapter 302 — Algorithm Guide

## Why This Matters

An algorithm is a contract between an operation, its inputs, its resource bounds, and its correctness rule. Choose it by workload and failure model, then verify it with representative evidence. Big-O notation is useful, but it does not include database round trips, allocations, lock waits, or provider quotas.

## The Decision Path

1. Define the result and edge cases.
2. State input size, distribution, ordering, mutation, and resource limits.
3. Choose the authoritative boundary: PHP, SQL, an index, a queue, or a read model.
4. Select a structure and algorithm together.
5. Estimate work, memory, I/O, and contention.
6. Test properties and failure paths.
7. Measure a representative workload and keep a regression guard.

## Searching, Grouping, and Deduplication

Linear search is appropriate for a small or single-pass collection. For repeated membership checks, normalize the key and use a map/set with explicit equality semantics. For grouped results, decide whether group order and item order are contractual before selecting a map of lists.

Sorting can make later lookup or merge work cheaper, but it costs time and memory. If the data is durable and filtered, let an indexed database query reduce rows before PHP receives them. Do not sort a complete table in application memory merely because the code is familiar.

## Pagination and Determinism

Offset pagination is simple but can scan and shift as rows are inserted. Keyset pagination uses a cursor containing the ordered values and a unique tie-breaker; it is usually better for large, changing datasets. The cursor must be validated, scoped to the same tenant and filter, and interpreted with the same direction and collation.

Stable output is part of correctness when clients resume, compare, cache, or export results. “The database usually returns this order” is not a contract.

## Intervals and State Transitions

For reservations or schedules, define whether endpoints are half-open, whether touching intervals overlap, and which timezone or precision applies. Sort or query candidates efficiently, but enforce the conflict invariant at the concurrency boundary. A single-process overlap algorithm cannot prevent two transactions from accepting the same slot.

For state machines, make allowed transitions explicit and idempotent. Reject stale versions or record the conflict; do not let a convenient assignment skip an audit or authorization rule.

## Batching and Streaming

Batching bounds transaction size, memory, and retry scope. Choose a batch size from row width, lock duration, downstream capacity, and recovery cost. Record a checkpoint that is safe to resume; avoid relying on array offsets when rows can change.

Streaming with an iterator or generator lowers peak memory only when the consumer is incremental. Close cursors, release resources, and define what partial output means when an item fails. A stream is not automatically backpressure: the downstream boundary must be able to slow production.

## Retries and Backoff

Retry classification is an algorithmic decision. Retry only transient failures, with an operation identity, bounded attempts or deadline, exponential backoff, jitter, and a clear owner of the retry budget. Unknown completion requires querying evidence or reconciling before another external effect.

Backoff does not make a non-idempotent operation safe. A queue’s retry algorithm also needs poison-message handling, visibility/acknowledgement rules, and a capacity limit.

## Graph and Traversal Work

Use a canonical node identity and a visited set for graph traversal. Choose breadth-first search when shortest edge count matters; choose depth-first search for exploration or dependency ordering, while bounding depth and handling cycles. Validate untrusted graphs before expensive traversal.

## Complexity Worksheet

| Quantity | Record |
| --- | --- |
| result and invariant | exact output and what must never change |
| input distribution | typical, tail, empty, duplicate, adversarial |
| operation count | comparisons, queries, calls, allocations |
| memory | peak PHP, database, worker, and response memory |
| ordering | explicit order and tie-breaker |
| failure | checkpoint, retry, replay, or reconciliation |
| evidence | property, integration, load, and production signals |

## Common Mistakes

- quoting complexity while ignoring I/O;
- choosing an algorithm before defining equality or ordering;
- using offset pagination for an unbounded changing dataset;
- changing business semantics during deduplication or sorting;
- retrying without idempotency;
- streaming while retaining every result in a second array;
- fixing a race with a faster single-process algorithm;
- omitting malformed, empty, huge, and adversarial inputs.

## Exercises

1. Design stable keyset pagination for a tenant-scoped search.
2. Compare PHP sorting with an indexed SQL query using measured row counts.
3. Define a resumable batch algorithm with a failure checkpoint.
4. Prove which reservation endpoints overlap under half-open semantics.
5. Review a retry loop for classification, deadline, jitter, and unknown completion.
6. Implement graph traversal rules that reject cycles or handle them safely.

## Review Questions

- What belongs in an algorithm’s contract besides its output?
- Why are tie-breakers necessary?
- When should SQL or an index own the work?
- How does batching change failure recovery?
- Why does backoff not create idempotency?
- Which evidence would disprove your complexity model?

## Summary

Algorithm selection begins with the result, invariant, input distribution, resource limits, authoritative boundary, and failure model. Deterministic ordering, bounded batching, streaming consumers, concurrency-safe invariants, retry identity, and measured I/O matter as much as asymptotic complexity.

## Chapter 303 Handoff

Algorithms are only useful when their claims are supported by the right evidence. Chapter 303 provides a testing decision guide that maps behavior and risk to unit, integration, contract, security, load, and recovery tests.

## References

- [Chapter 72 — Why Algorithms Matter in PHP](../06-algorithms-and-data-structures/072-why-algorithms-matter-in-php.md)
- [Chapter 90 — Streaming Algorithms](../06-algorithms-and-data-structures/090-streaming-algorithms.md)
- [Chapter 280 — Tennis Reservation Service](../19-small-engineering-projects/280-tennis-reservation-service.md)
- [Chapter 281 — Rate Limiter](../19-small-engineering-projects/281-rate-limiter.md)
- [Chapter 287 — Search/Filtering Service](../19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 302 — Data Structure Guide](301-data-structure-guide.md)
