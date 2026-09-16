---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 91
title: Choosing the Right Data Structure
slug: choosing-the-right-data-structure
status: complete
summary: ../../_ai/chapter-summaries/091-choosing-the-right-data-structure-summary.md
---

# Chapter 91 — Choosing the Right Data Structure

## Why This Matters

PHP gives developers a flexible array that can be used as a list, map, set, stack, queue, grouping table, and more. That flexibility is convenient, but it can hide the fact that these roles have different invariants and costs. A list is convenient for ordered traversal. A map supports lookup by key. A queue preserves a removal order. A heap exposes an extremum. Choosing the wrong structure can make an otherwise simple operation slow, incorrect, wasteful, or impossible to make reliable across requests.

There is no universally best data structure. The useful question is which operations must be correct and fast, how often they occur, how the collection grows and changes, and which process or service owns the data. The chapters in this volume have introduced the individual structures and algorithms; this chapter turns them into a repeatable choice.

## Mental Model

Choose a representation from the workload contract, not from familiarity or a class name:

```text
required operations + identity/order rules + size/update pattern
                   + memory and lifetime + durability boundary
                              ↓
                       candidate structure
                              ↓
                  measured total cost and tests
```

A good structure makes the important operation easy while preserving required behavior. It does not remove the cost of building or refreshing itself, defining duplicate policy, keeping data fresh, or crossing a database or network boundary.

## Core Concept

Before choosing a representation, write down the operations and their expected frequency. A useful cost model is:

```text
build cost + query count × query cost + update count × update cost + memory/failure cost
```

For example, scanning a list once for a value can be simpler and cheaper than building a map. If thousands of later operations look up values from that same stable collection, paying to build an index may be worthwhile. If the collection changes after every lookup, the map also needs an update or invalidation policy. Chapters 73 and 80 develop this scan-versus-index comparison.

Then state the correctness contract:

- What makes two items the same: scalar value, canonical key, database ID, or object identity?
- Are duplicates rejected, preserved, counted, grouped, or replaced?
- Must results preserve input order, use a total order, or expose only a minimum/maximum?
- Which operations are frequent: lookup, membership, insertion, removal, iteration, range query, aggregation, or traversal?
- Is the collection static, updated occasionally, or continuously changing?
- How many items and how many bytes can be live at once?
- Does the data need to survive a request, process restart, deployment, or machine failure?

If the answer to those questions changes, the best choice may change too. The data structure is part of the behavior contract, not an optimization detached from it.

## A Decision Method

### 1. Start with behavior and identity

Write a small, direct version of the operation and define its result. “Find the record” is incomplete: is the caller asking for any matching value, the first matching key, all duplicates, or the value under an exact ID? A map that silently overwrites duplicate keys changes behavior compared with a scan that can return every matching row.

Choose identity before choosing a key. PHP array keys coerce some scalar values, so normalize or encode identifiers deliberately. A set of permission strings, a frequency map, and a set of object instances each have different equality rules. Chapters 75 and 76 discuss PHP map keys, presence, sets, and object identity.

### 2. Rank the operations

List the operations the application performs and estimate their counts. A workload of one search over a list of twenty records often favors a simple scan. Repeated exact-key lookup against a stable list may favor a map. Need the entire result ordered once? Sort. Need the next smallest item repeatedly while values arrive? Heap. Need first-in/first-out removal? Queue.

Do not optimize the operation that sounds impressive if it is rare. If a structure makes reads fast but makes every common write costly, the total workload may get worse. Include index construction, sorting, heap updates, cleanup, and output production in the comparison.

### 3. Account for growth and update shape

Ask whether the collection is bounded. A request-local set of a few hundred IDs is different from a process-lifetime cache that grows on every job. Ask whether updates are append-only, arbitrary, or batched. A sorted list can serve many binary searches well when changes are rare, while inserting into the middle of a PHP list still shifts later entries. A heap works well for repeated insertion and removal of the current extremum, but arbitrary update-by-ID needs extra machinery.

Estimate the common and worst-case sizes, not only the average. Consider an attacker-controlled key count, unusually large payloads, a dense graph, or many intervals that all overlap. Set a limit or select an external representation when the in-memory upper bound is unsafe.

### 4. Make order explicit

There are several distinct order requirements:

- **Insertion order:** use a list or a map whose iteration order is part of the contract.
- **FIFO or LIFO:** use a queue or stack abstraction, even if a PHP array implements it.
- **Full sorted output:** sort the complete collection under a comparator with explicit tie-breakers.
- **Repeated ordered search:** keep a sorted representation and use binary search, while accounting for updates.
- **Repeated next-smallest/largest:** use a min-heap or max-heap/priority queue.
- **No meaningful order:** avoid maintaining an order that callers do not need.

Stability, deterministic output, and priority are separate concepts. A stable sort preserves previous ties; a complete comparator defines deterministic ties; a priority queue selects by rank and may not be fair. See Chapters 79 and 84 rather than inferring those guarantees from a structure's name.

### 5. Count retained state and process lifetime

PHP arrays are ordered maps with meaningful per-entry overhead. A design with a values list, an ID map, a sorted copy, and a result array may retain several representations at once. Copy-on-write can defer some array copying, but mutation may separate shared storage; it does not make parallel indexes free. Measure peak live memory, including source data, payloads, temporary maps, sorted outputs, serialization, and in-flight requests. Chapter 74 covers those memory boundaries.

A generator can avoid materializing a source, but it cannot make an accumulated result bounded. A bounded batch or streaming algorithm can cap retained input, but still needs a downstream failure and checkpoint policy. If state persists in a long-running worker, define eviction, freshness, and maximum lifetime. A local PHP array is discarded at process exit and is not shared between PHP-FPM workers.

### 6. Identify the owner and required guarantees

If the database owns millions of persistent rows, use its indexes, range queries, constraints, and ordering rather than loading everything into PHP to search or sort. If the state must be coordinated across requests or hosts, an in-process structure is not the consistency boundary. A database, cache, or message broker may be needed, each with different durability and failure semantics.

The database is not automatically the right place for every transformation. PHP can be appropriate for a bounded domain-specific computation over records already loaded. Conversely, a fast PHP map does not provide a uniqueness constraint against concurrent inserts. Put the invariant at the boundary that can enforce it. Chapters 104–123 build out database ownership and query costs; distributed delivery concerns appear later in Chapter 244.

## Comparison Table

These are starting candidates under the stated conditions, not universal complexity promises. PHP Manual APIs describe behavior; Big O figures are the usual abstract model and still need workload measurement.

| Required operation or shape | Starting candidate | Why it fits | Main trade-off or boundary |
| --- | --- | --- | --- |
| Append and traverse in order | PHP list (`array`) | Direct positional and sequential access | Middle insertion/removal shifts values; PHP arrays use more memory than compact buffers |
| Repeated exact-key lookup or membership | Associative array map/set | Expected O(1) average lookup after build | O(n) retained keys and values; key coercion, duplicate policy, and freshness matter |
| Count occurrences by key | Frequency map | One counter per distinct value | Equality and normalization must be defined; cardinality may grow with input |
| Last-in, first-out work | Array stack or `SplStack` | Push/pop at the top | Does not support fast arbitrary removal or search |
| First-in, first-out work | Head-index array or `SplQueue` | Efficient endpoint consumption | Capacity, persistence, and retries are separate concerns |
| Search once in a small unsorted collection | Linear scan | No index build or additional storage | O(n) for a miss or late match |
| Search repeatedly in stable data | Map or sorted list + binary search | Amortizes index/sort construction over queries | Updates can stale or make the representation expensive |
| Return every value in a defined order | Sort a list/map values | Straightforward complete result | O(n log n) comparison model and temporary memory; DB may own ordering |
| Repeatedly select only current minimum/maximum | Heap / priority queue | Extremum at the root; efficient insert/extract | Arbitrary ID updates, fairness, and durable jobs need additional design |
| Model one-parent hierarchy | Tree-shaped data or parent-ID index | Represents parent/child relationships and traversal | Validate cycles, sharing, depth, and update rules |
| Model general relationships and paths | Graph, often adjacency list | Efficient traversal of sparse edges | Identity, density, graph size, and snapshot consistency matter |
| Merge ranges or check time conflicts | Sorted intervals / sweep / database range feature | Uses endpoint order and interval invariants | Endpoint policy and concurrent-write enforcement must be explicit |
| Maintain a recent window aggregate | Queue/deque plus running state | Removes expired values and avoids recomputing whole windows | Input ordering, endpoint inclusion, and retained output matter |
| Read data too large for memory | Generator/stream plus bounded processing | Keeps work proportional to current chunk | Does not by itself give resumability or bound sink latency |
| Reuse repeated deterministic subproblems | Local memo table | Avoids recomputing the same state | Complete keys, invalidation, and bounded lifetime are required |
| Survive process exit or coordinate writers | Database/broker/cache as appropriate | Owns shared or durable state | Network cost, transactions, availability, and delivery guarantees apply |

The table names operations because a single application may need several representations. The goal is not to force every record into one structure. Add a second index only when the repeated operation justifies its memory and update cost, and make the rule for keeping both views consistent explicit.

## Practical Example: A Small Event Snapshot

Suppose one request loads a bounded snapshot of event records and must do two things: look up an event by exact string ID many times and render the snapshot in timestamp order once. A list scan repeated for every lookup would cost O(qn). A map costs O(n) to build, gives expected O(1) average lookup, and can be sorted by value while retaining its ID keys for later lookup.

```php
<?php

declare(strict_types=1);

/**
 * Build an ID-keyed index and order it by timestamp then ID.
 *
 * @param iterable<array{id: string, occurred_at: int, type: string}> $events
 * @return array<string, array{id: string, occurred_at: int, type: string}>
 */
function indexAndOrderEvents(iterable $events): array
{
    $byId = [];

    foreach ($events as $event) {
        if (!is_array($event)
            || !is_string($event['id'] ?? null)
            || $event['id'] === ''
            || !is_int($event['occurred_at'] ?? null)
            || !is_string($event['type'] ?? null)) {
            throw new InvalidArgumentException('Malformed event record.');
        }

        // Prefixing preserves exact string IDs, including numeric-looking IDs.
        $key = 'event:' . $event['id'];
        if (array_key_exists($key, $byId)) {
            throw new InvalidArgumentException('Duplicate event ID.');
        }

        $byId[$key] = $event;
    }

    uasort(
        $byId,
        static fn (array $left, array $right): int =>
            ($left['occurred_at'] <=> $right['occurred_at'])
            ?: strcmp($left['id'], $right['id']),
    );

    return $byId;
}

$eventsById = indexAndOrderEvents([
    ['id' => 'evt-21', 'occurred_at' => 1_800_000_010, 'type' => 'payment'],
    ['id' => 'evt-20', 'occurred_at' => 1_800_000_005, 'type' => 'login'],
]);

$event = $eventsById['event:evt-21'] ?? null; // Expected lookup remains available.

foreach ($eventsById as $orderedEvent) {
    // Render in deterministic chronological order.
}
```

The complete comparator uses the unique string ID to break equal timestamps. `uasort()` preserves the map keys while ordering iteration, so the same representation supports both operations. Building takes O(n) expected map work and sorting takes O(n log n); each later key lookup is expected O(1) average. The PHP array retains O(n) entries. The prefix is a local encoding choice; the domain must still define whether IDs are case-sensitive or normalized.

This design fits one bounded, request-local snapshot. If input is millions of persistent rows, query the database with an appropriate `WHERE`, `ORDER BY occurred_at, id`, and an index rather than building a PHP copy. If events arrive continuously and the application only needs the next due event, a priority queue may fit better. If arbitrary updates by ID are frequent while chronological order is also needed, the map-plus-sort snapshot may be too expensive to rebuild; pick a representation and update strategy for that workload instead.

## Changing Workloads and Composite Choices

One data structure rarely serves every operation equally well. A map can support ID lookup while a sorted list supports range traversal, but keeping both updated on every write adds memory and consistency work. A heap can select the earliest deadline while a separate map tracks current task versions. A graph can use an adjacency list for traversal and a database index for persistent lookup. These combinations are justified when they serve recurring operations and have a clear update boundary.

Avoid casually layering structures to improve every theoretical operation. Each duplicate index creates another invariant: what happens when insertion partially succeeds, a process crashes between updates, or one copy is stale? Prefer one source of truth and rebuildable derived indexes when the workload allows it. If both structures must change atomically, use a transactional boundary or design reconciliation.

## Testing the Choice

Test the representation's contract and the operation distribution it was chosen for:

1. Assert duplicate, identity, and ordering behavior explicitly, including numeric-looking string IDs and ties.
2. Compare optimized results against a small, simple reference implementation such as a scan or full sort.
3. Include empty, singleton, duplicate-heavy, miss-heavy, and worst-shape inputs.
4. Test index freshness after updates, deletes, and source-version changes.
5. Measure build plus query cost at realistic sizes and query counts; do not time only the final lookup.
6. Measure peak memory when the original, index, sorted view, and output coexist.
7. For persistent state, test the actual SQL query plan, uniqueness constraint, transaction, and concurrent write behavior.
8. For streams or workers, test bounded memory, partial failure, restart, and cleanup.

Property-based tests are useful when an invariant has many possible inputs. For example, a sorted result should contain each input record once and satisfy the comparator for every adjacent pair. A set index should agree with a strict reference scan under the chosen canonicalization. The data structure can be asymptotically sound and still be incorrect for the application's equality, stale-data, or boundary rules.

## Operational and Database Boundaries

In a short-lived request, local structures often have a naturally bounded lifetime. A long-running worker does not: maps, memo tables, and accumulated result arrays can keep growing across jobs unless reset or capped. Track item count, oldest retained state, memory, and cache hit/update behavior where they affect operations. Rotate or rebuild derived state under a defined policy, not only after the process approaches its memory limit.

For persistent datasets, transfer and query cost often dominate in-process lookup. Filter and order at the database when that system owns the data; inspect plans and use indexes that match the predicate and ordering. Keep domain transformation in PHP when it is bounded and clearer there. Use database constraints for invariants that must hold across concurrent writers. A PHP map can reject duplicates seen in one request, but it cannot serialize independent workers.

For state that must survive restart or coordinate multiple consumers, choose a durable service with semantics matching the requirement. A process-local queue, heap, set, or memo table disappears when that process exits. Durability, acknowledgement, retries, leases, and idempotency are separate guarantees, not properties gained by choosing a different in-memory collection.

## Common Mistakes

- Choosing a data structure by name before stating required operations and identity.
- Comparing one lookup cost while ignoring index construction and update frequency.
- Using a map when duplicates must be retained, or a set when multiplicity matters.
- Assuming PHP's ordered-map iteration gives a complete business sort.
- Using binary search on data that changes without maintaining the sorted invariant.
- Using a heap for arbitrary priority updates or search without an auxiliary index.
- Keeping several redundant PHP arrays without counting their peak memory or synchronization rules.
- Assuming a generator makes the entire pipeline bounded when the consumer accumulates output.
- Treating a process-local structure as durable or shared across PHP workers.
- Moving work out of SQL without accounting for network transfer, PHP memory, and concurrent writes.
- Optimizing for Big O alone without measuring the real collection size, payload, and call boundary.

## Senior Engineer Thinking

Start with the operation table, then challenge every assumption. How many queries happen between updates? Are IDs canonical and unique? Does the caller need order or just membership? Is the data a snapshot or a live view? How much memory remains live while the response is serialized? What happens if the process ends halfway through an update? Where is the authoritative copy?

Choose the simplest candidate that meets the contract under expected and worst-case load. Include build, update, memory, and boundary costs. Keep a straightforward reference implementation for tests, benchmark realistic data, and revisit the choice only when the workload or measured bottleneck changes. Good structure selection is a constraint-driven engineering decision, not a contest to use the most advanced container.

## Exercises

1. For a list of 40 users and one membership check, compare a scan and a set. Then estimate the break-even when the same list serves repeated checks. State assumptions rather than inventing a universal threshold.
2. A process receives tasks with IDs, mutable priorities, and frequent cancellation. Compare a plain array scan, `SplPriorityQueue`, an ID map plus lazy invalidation, and a durable broker. State which operation each option makes expensive.
3. A report must return the globally earliest 100 database rows from a table containing 20 million records. Decide which work belongs in SQL and what PHP should retain.
4. Model a permission check where object identity, tenant scope, and role changes matter. Choose the membership representation and specify invalidation behavior.
5. An import reads 5 GB of newline-delimited input, rejects duplicate external IDs, and writes rows in groups. Design a memory bound, duplicate scope, batch transaction boundary, and restart policy.
6. A category hierarchy unexpectedly contains a shared child and a cycle. Explain why a tree traversal may now be wrong and which graph rules and visited state are needed.
7. Pick one structure from the comparison table and build a benchmark that includes construction, updates, successful and unsuccessful operations, and peak memory. Explain when the result would change your decision.

## Review Questions

1. Why is the frequency of operations part of data structure selection?
2. What costs should be counted when comparing a scan with a prebuilt index?
3. Which identity and duplicate policies should be decided before using a PHP array as a map or set?
4. When is a sorted list preferable to a heap, and when is a heap preferable to a sorted list?
5. Why can an ordered PHP map still fail to provide deterministic business ordering?
6. What memory cost can appear when an index and sorted view coexist?
7. Why do generators and batching not automatically make a sink durable or restartable?
8. Which problems should usually be handled where the authoritative data is stored?
9. What guarantees must be added when state must be shared across PHP processes or survive restart?
10. How should measurement influence a choice made from Big O analysis?

## Summary

Choose a data structure by specifying required operations, their frequency, identity and ordering rules, update shape, maximum retained state, and durability boundary. Include construction, maintenance, memory, output, and external calls in the cost model. PHP arrays can serve as lists, maps, and sets, while stacks, queues, heaps, graphs, intervals, windows, batches, and memo tables fit more specific invariants. Sorting or indexing is worthwhile only when its repeated benefit pays for build and update costs. Persistent data and concurrent invariants often belong in the database or a durable service. Test correctness against a simple reference, measure realistic workloads, and prefer the simplest structure that satisfies the full contract.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: `array_key_exists()`](https://www.php.net/manual/en/function.array-key-exists.php)
- [PHP Manual: `uasort()`](https://www.php.net/manual/en/function.uasort.php)
- [Chapter 72 — Why Algorithms Matter in PHP](072-why-algorithms-matter-in-php.md)
- [Chapter 73 — Big O](073-big-o.md)
- [Chapter 74 — Memory Complexity](074-memory-complexity.md)
- [Chapter 75 — Arrays and Hash Maps](075-arrays-and-hash-maps.md)
- [Chapter 76 — Sets](076-sets.md)
- [Chapter 77 — Stacks](077-stacks.md)
- [Chapter 78 — Queues](078-queues.md)
- [Chapter 79 — Sorting](079-sorting.md)
- [Chapter 80 — Searching](080-searching.md)
- [Chapter 81 — Binary Search](081-binary-search.md)
- [Chapter 82 — Trees](082-trees.md)
- [Chapter 83 — Heaps](083-heaps.md)
- [Chapter 84 — Priority Queues](084-priority-queues.md)
- [Chapter 85 — Graphs](085-graphs.md)
- [Chapter 86 — Intervals](086-intervals.md)
- [Chapter 87 — Sliding Windows](087-sliding-windows.md)
- [Chapter 88 — Batching](088-batching.md)
- [Chapter 89 — Memoization](089-memoization.md)
- [Chapter 90 — Streaming Algorithms](090-streaming-algorithms.md)
- [Chapter 244 — Queues](../../volumes/16-distributed-systems/244-queues.md)
