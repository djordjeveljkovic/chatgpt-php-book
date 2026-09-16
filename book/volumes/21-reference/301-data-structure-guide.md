---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 301
title: Data Structure Guide
slug: data-structure-guide
status: complete
summary: ../../_ai/chapter-summaries/301-data-structure-guide-summary.md
---

# Chapter 301 — Data Structure Guide

## Why This Matters

Choose a structure from the operation and constraints, not from habit. In PHP, an array is an ordered hash map with meaningful memory overhead; it is not a compact vector, universal set, or database index.

## Start With the Operations

Write the dominant operations first: lookup by key, membership, insertion, deletion, ordered traversal, priority selection, range query, grouping, or streaming. Then state expected size, mutation, ordering, key semantics, memory budget, concurrency, and whether the database already owns the data.

## Selection Guide

| Need | Candidate | Important qualification |
| --- | --- | --- |
| keyed lookup | associative array/map | average-case lookup; key coercion and memory matter |
| ordered sequence | list/array | insertion and deletion position may be linear |
| membership set | map or dedicated set abstraction | define equality and normalization |
| FIFO work | queue/deque | bound capacity and acknowledgement separately |
| highest/lowest next | heap/priority queue | comparator direction and tie-breaking matter |
| sorted range | database index or sorted structure | choose the authority and mutation cost |
| hierarchy | tree or adjacency map | define parent validity and traversal limits |
| relationships | adjacency list/graph | visited-state and cycle handling are required |
| huge input | iterator/generator/chunks | downstream backpressure and failure scope remain |
| durable search | database index/projection | tenant scope, freshness, and rebuildability matter |

## PHP Arrays and Maps

Use an associative array when a bounded in-memory map or ordered collection is the actual requirement. For deduplication while preserving first-seen order, keep a membership map and an output sequence. Avoid repeated `in_array()` scans when the workload is large and keys can express the equality rule.

State whether keys are integers, strings, or normalized identifiers. Do not assume a string that looks numeric has the same key behavior as an opaque identifier. Test missing keys separately from values that happen to be `null`.

## Queues, Stacks, and Heaps

A queue models arrival order; a priority queue models selection by priority. Neither guarantees durable delivery, fairness, or retry safety. Use SPL structures or a library when they express the operation, but document comparator direction, tie-breaking, capacity, and ownership.

For a bounded worker, the queue size is a resource policy. If it grows without bound, the structure is hiding backpressure rather than solving it.

## Trees and Graphs

Trees are useful for hierarchical ownership, but input may be malformed or cyclic. Validate parent relationships and bound depth. Graph traversal requires a canonical node identity and a visited set; otherwise cycles or repeated edges can cause unbounded work.

## Eager Versus Lazy Data

An array materializes all values; an iterator or generator produces them incrementally. Laziness reduces peak memory only when consumers also process incrementally. It does not remove total I/O, CPU, provider effects, or the need to close cursors and handle partial failure.

For imports and exports, choose chunk size from memory, transaction scope, retry scope, and downstream capacity. A generator inside a transaction can accidentally keep locks or connections open for the entire iteration.

## In-Memory Versus Database Structures

Use a database index when data is durable, shared, large, or queried by multiple processes. Use PHP structures for bounded local work or a deliberate projection. Do not download a whole table to sort or deduplicate if SQL can reduce the work and preserve the same semantics.

Indexes also have write, storage, and maintenance costs. A structure is not “faster” without naming the operation, data distribution, and measurement.

## Complexity and Memory

Complexity is a model, not a performance guarantee. State average-case assumptions, constants, allocations, hash behavior, I/O, and cache locality. A nominally linear PHP array operation can still exceed a worker memory limit; a database query with more logical work can win by avoiding network transfer.

## Decision Worksheet

| Question | Decision evidence |
| --- | --- |
| What operation dominates? | measured call/query counts and workload |
| What is the size distribution? | median, tail, and adversarial sizes |
| Is ordering contractual? | API, pagination, or business rule |
| Who owns the data? | process, service, or database |
| What is the memory bound? | worker/container budget and peak estimate |
| What happens on partial failure? | checkpoint, retry, replay, or discard |
| How is correctness tested? | invariant, property, integration, and load evidence |

## Common Mistakes

- claiming exact O(1) without assumptions;
- treating arrays as compact lists;
- using repeated scans for large membership work;
- relying on incidental order or unstable tie-breakers;
- assuming generators eliminate total work;
- ignoring key coercion and equality semantics;
- reimplementing database indexes in application memory;
- forgetting capacity and failure behavior.

## Exercises

1. Choose structures for tenant lookup, stable pagination, top-k selection, and streaming export.
2. Replace repeated membership scans with an order-preserving structure and state its memory cost.
3. Design a bounded graph traversal that handles cycles and malformed input.
4. Compare a PHP sort with an indexed SQL query using a representative workload.
5. Select chunk size for a worker and explain its retry and transaction boundaries.

## Review Questions

- Why is a PHP array not a compact vector?
- When is a database index the right data structure?
- What does a generator improve, and what does it not improve?
- Which properties must a priority queue specify?
- Why do stable tie-breakers matter?
- How do memory and ownership change a structure choice?

## Summary

Data-structure choice follows dominant operations, data size, memory, ordering, equality, mutation, ownership, durability, and failure behavior. PHP arrays are powerful but costly maps; queues and heaps need capacity and tie rules; generators need incremental consumers; and databases often provide the right durable structure.

## Chapter 302 Handoff

Chapter 302 turns structure selection into algorithm selection, including complexity, deterministic output, pagination, batching, streaming, retries, and measurement.

## References

- [Chapter 17 — PHP Arrays](../02-php-language-fundamentals/017-arrays.md)
- [Chapter 72 — Why Algorithms Matter in PHP](../06-algorithms-and-data-structures/072-why-algorithms-matter-in-php.md)
- [Chapter 90 — Streaming Algorithms](../06-algorithms-and-data-structures/090-streaming-algorithms.md)
- [Chapter 283 — File Importer](../19-small-engineering-projects/283-file-importer.md)
- [Chapter 287 — Search/Filtering Service](../19-small-engineering-projects/287-search-filtering-service.md)
