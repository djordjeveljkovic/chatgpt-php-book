---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 83
title: Heaps
slug: heaps
status: complete
summary: ../../_ai/chapter-summaries/083-heaps-summary.md
---

# Chapter 83 — Heaps

## Why This Matters

A worker may need the next retry due, a report may need the largest few values in a stream, or a scheduler may need to choose the earliest available task repeatedly. Sorting every item gives a complete order, but often the application only needs to know which single item comes next. A heap keeps that answer at its root and restores it efficiently after each insertion or removal.

Heaps are useful when the collection changes while the application repeatedly asks for its smallest or largest element. They do not make arbitrary search fast, and they do not keep all elements in sorted order. The requirement is usually narrower: maintain an extremum as values arrive and leave.

## Mental Model

Think of a min-heap as a waiting room where the smallest key is always visible at the front, while the other people are only partially ordered:

~~~text
insert + compare → restore heap invariant → read/remove minimum
~~~

Every parent is no greater than either child. That local rule is enough to make the root the global minimum, because every path from the root to any descendant is nondecreasing. It says nothing about the order between siblings or across separate branches.

## Core Concept

A **binary heap** is a complete binary tree that obeys a heap-order invariant. A complete tree fills each level from left to right, with only the final level allowed to be incomplete. In a **min-heap**, each parent is less than or equal to its children; in a **max-heap**, each parent is greater than or equal to its children.

One valid min-heap containing the values `2, 5, 3, 9, 7, 8` is:

~~~text
           2
         /   \
        5     3
       / \   /
      9   7 8
~~~

The root is the minimum. The left and right subtrees satisfy the same rule. But `5` and `3` are not in a defined left-to-right ordering, and reading the tree in-order would not produce a sorted list. A heap is not a binary search tree, nor is its shape an ordering of all values. Its invariant guarantees only that the root is the extremum.

This distinction matters when selecting an implementation. A sorted array supports binary search and ordered iteration; a heap supports efficient insertion and removal of the current extremum. Chapter 81 covers searching a sorted sequence, and Chapter 79 covers sorting all values.

## How It Works

### Represent a complete tree with positions

Completeness lets the usual binary heap store its nodes in a contiguous zero-indexed array, without parent or child pointers. For a node at index `i`:

~~~text
parent(i) = intdiv(i - 1, 2)     when i > 0
left(i)   = 2 * i + 1
right(i)  = 2 * i + 2
~~~

The example heap can be represented as:

~~~text
index:   0  1  2  3  4  5
value:  [2, 5, 3, 9, 7, 8]
~~~

Index `0` is the root. For index `1`, the children are at `3` and `4`; for index `2`, its left child is at `5`, and its right child would be at `6`, outside the array. The array's shape is complete by construction. The heap-order property still has to be maintained whenever elements change.

This is the conventional binary-heap representation and a useful model for understanding index arithmetic. It is not a promise about the internal storage layout of every library heap. PHP documents the public behavior of SPL heaps, not a stable internal representation contract.

### Insert by sifting up

To add a value, append it at the next array position. It is now a leaf, so only the relation with its parent can be wrong. Compare it with its parent and swap upward while the heap rule is violated. This is often called **sift up**, **bubble up**, or **percolate up**.

Starting with `[2, 5, 3, 9, 7, 8]`, insert `1`:

~~~text
append:        [2, 5, 3, 9, 7, 8, 1]
swap with 3:   [2, 5, 1, 9, 7, 8, 3]
swap with 2:   [1, 5, 2, 9, 7, 8, 3]
~~~

The new value travels only along one root-to-leaf path. A complete binary tree with `n` nodes has height `O(log n)`, so insertion takes `O(log n)` comparisons in the worst case. Reading the root takes `O(1)`.

### Remove the root by sifting down

The root is the item to remove. Move the final array item into the root position, remove the last slot, and compare the replacement with its children. For a min-heap, swap with the smaller child if the parent is larger, and continue until the rule holds or there are no children. This is **sift down**.

Only one path changes, so removing the root is `O(log n)`. An empty heap has no root, so callers must define their empty-case behavior. A general search for an arbitrary value may still inspect all `n` values: the heap invariant does not let us rule out either subtree based on an unrelated target.

### Build and update costs

Inserting `n` values one at a time costs `O(n log n)` in the worst case. If all values are already available, bottom-up heap construction (heapify) can build a heap in `O(n)`: most nodes are close to the leaves and need little or no sifting. An API may or may not expose an efficient bulk-build operation; check that API rather than assuming one.

A binary heap also does not naturally offer efficient arbitrary deletion or priority updates unless it tracks each element's current index and repairs the invariant afterward. If an item's priority changes in place, the comparisons that positioned it may no longer be valid. Use an indexed heap, insert a new version and discard stale entries when popped, or choose a different structure when arbitrary updates dominate.

## PHP Implementations

PHP's Standard PHP Library provides `SplHeap`, an abstract heap base class, and the concrete `SplMinHeap` and `SplMaxHeap`. `SplMinHeap` keeps the minimum at the top; `SplMaxHeap` keeps the maximum there. The classes accept `mixed` values, so the application must choose values whose comparison is meaningful and consistent. The [PHP Manual's `SplHeap` reference](https://www.php.net/manual/en/class.splheap.php) lists the public operations, while the [min-heap](https://www.php.net/manual/en/class.splminheap.php) and [max-heap](https://www.php.net/manual/en/class.splmaxheap.php) references specify which extremum is kept at the top.

For integers, the basic use is direct:

~~~php
<?php

declare(strict_types=1);

$heap = new SplMinHeap();
$heap->insert(8);
$heap->insert(3);
$heap->insert(5);

$nextSmallest = $heap->top();    // 3; does not remove it
$removedSmallest = $heap->extract(); // 3; removes it
~~~

`top()` and `extract()` throw `RuntimeException` when the heap is empty, so guard with `isEmpty()` when emptiness is an expected state. `extract()` removes the extremum. A less obvious API detail is that `SplHeap` implements `Iterator`, and advancing it with `next()` extracts and deletes the current top. A `foreach` loop therefore drains the heap. This is useful for consuming values in priority order, but it surprises code that expects an observational traversal. The manual documents these behaviors under [`top()`](https://www.php.net/manual/en/splheap.top.php), [`extract()`](https://www.php.net/manual/en/splheap.extract.php), and [`next()`](https://www.php.net/manual/en/splheap.next.php).

The SPL class is a concrete implementation. A heap is a data structure; a **priority queue** is an API contract for inserting items with priority and retrieving the next item according to that priority. PHP's [`SplPriorityQueue`](https://www.php.net/manual/en/class.splpriorityqueue.php) is implemented using a max-heap, and values with equal priority have no defined relative order. Chapter 84 covers its queue operations. A heap can implement a priority queue, but the terms do not mean the same thing.

### Comparator correctness and ties

Custom heap subclasses define `compare(mixed $value1, mixed $value2): int`, but do not assume one sign convention across SPL classes. `SplMaxHeap::compare()` returns a positive value when the first value is greater; `SplMinHeap::compare()` returns a positive value when the first value is lower. The concrete class's [`SplMaxHeap::compare()`](https://www.php.net/manual/en/splmaxheap.compare.php) or [`SplMinHeap::compare()`](https://www.php.net/manual/en/splminheap.compare.php) documentation defines the direction. For example, a natural ascending comparator in a `SplMinHeap` subclass returns the maximum first; reverse the comparison to keep the minimum at the top.

The comparison must be consistent and transitive: if `a` is less than `b`, and `b` is less than `c`, then `a` must be less than `c`. It should be deterministic and should not depend on a clock, database query, random value, or mutable external setting. Avoid subtraction such as `$a - $b` as an integer comparator: large values can overflow into a float. The spaceship operator (`<=>`) returns the comparison sign without that subtraction.

Ties need a domain decision. The PHP Manual warns that equal elements can end up in an arbitrary relative position; the heap is not stable. If equal priorities must have a deterministic order, add a unique sequence or other tie-breaker to the comparison key. If ties are truly equivalent, do not write application logic that depends on which one appears first.

An exception from `SplHeap::compare()` can leave the heap corrupted and block further actions. `isCorrupted()` reports the state, and `recoverFromCorruption()` unblocks it, but recovery does not guarantee that all values again satisfy the heap property. Prefer validating inputs before insertion and keeping the comparator total and non-throwing. If a comparator failure occurs, reconstruct a new heap from a trusted source of values rather than assuming recovery repaired the order. See the official [`compare()` documentation](https://www.php.net/manual/en/splheap.compare.php) for its return convention, tie caveat, and corruption warning.

### Top-k values from a stream

Suppose an import stream may contain millions of scores, but a report needs only the ten largest. Sorting all records uses `O(n log n)` time and `O(n)` retained space. A min-heap of at most `k` scores keeps the smallest retained score at the root. For each incoming score, fill the heap until it contains `k` values; afterward, replace the root only when the new score is greater.

~~~php
<?php

declare(strict_types=1);

/** @param iterable<int> $scores @return list<int> descending */
function largestScores(iterable $scores, int $k): array
{
    if ($k < 0) {
        throw new InvalidArgumentException('$k must not be negative.');
    }

    if ($k === 0) {
        return [];
    }

    $heap = new SplMinHeap();

    foreach ($scores as $score) {
        if ($heap->count() < $k) {
            $heap->insert($score);
        } elseif ($score > $heap->top()) {
            $heap->extract();
            $heap->insert($score);
        }
    }

    $largestFirst = [];

    while (!$heap->isEmpty()) {
        $largestFirst[] = $heap->extract();
    }

    rsort($largestFirst, SORT_NUMERIC);

    return $largestFirst;
}
~~~

This example returns the largest values in descending order. It treats duplicate values as separate observations; when equal scores fill the boundary, the numeric output is the same regardless of which equal occurrence remains. If records have IDs and the product requires deterministic tie selection, compare a composite key such as `(score, id)` and document which IDs win. The final `rsort()` costs `O(k log k)`; omit it if the caller only needs the set of retained values and does not need them ordered.

For `n` input values and `k > 0`, the heap uses `O(k)` retained memory and takes `O(n log(k + 1))` worst-case heap work, plus `O(k log k)` to order the result. When `k` is zero, this function returns immediately. If `k` is close to `n`, a full sort may be simpler or faster in practice. The right choice depends on the actual workload and the overhead of the PHP representation, not only Big O.

### Deterministic local scheduling

A min-heap can select the earliest task among tasks held by one process. The comparison below orders by scheduled time and then by a unique insertion sequence. `readonly` properties keep the fields that determine ordering from changing while the object is stored in the heap.

~~~php
<?php

declare(strict_types=1);

final class ScheduledTask
{
    public function __construct(
        public readonly int $runAt,
        public readonly int $sequence,
        public readonly string $name,
    ) {
    }
}

final class TaskHeap extends SplMinHeap
{
    protected function compare(mixed $a, mixed $b): int
    {
        // SplMinHeap expects a positive result when $a is the smaller value.
        $timeOrder = $b->runAt <=> $a->runAt;

        if ($timeOrder !== 0) {
            return $timeOrder;
        }

        return $b->sequence <=> $a->sequence;
    }
}

$tasks = new TaskHeap();
$tasks->insert(new ScheduledTask(1_800_000_000, 1, 'send receipt'));
$tasks->insert(new ScheduledTask(1_800_000_000, 2, 'refresh cache'));
$tasks->insert(new ScheduledTask(1_799_999_900, 3, 'expire token'));

while (!$tasks->isEmpty()) {
    $task = $tasks->extract();
    // Hand the task to the local worker when its policy permits.
}
~~~

The sequence makes distinct equal-time entries deterministic, but it must actually be unique within the heap's comparison domain. Inputs must all be `ScheduledTask` objects with integer timestamps and sequence numbers. Validate them at the boundary before calling `insert()`; an exception or invalid comparison during a heap operation can damage the structure. In production, compare the current time with the next task's `runAt`, sleep or wait only according to the worker design, and re-check after waking because the process may be delayed.

This heap is process-local and volatile. It does not coordinate PHP-FPM workers, survive a restart, guarantee a task runs once, or replace a durable database table or message broker. If tasks are persisted, let the database select a due batch using an indexed query and transaction/locking strategy appropriate to the engine; the heap can still be useful for a bounded in-memory scheduling window. Chapters on databases and queues develop those failure and concurrency guarantees.

## Complexity and Memory

For a binary heap holding `n` items:

| Operation | Typical/worst-case time | Result |
|---|---:|---|
| Read extremum (`peek`) | `O(1)` | Leaves the heap unchanged |
| Insert one item | `O(log n)` | Restores order along one path |
| Remove extremum | `O(log n)` | Replaces the root and sifts down |
| Search for an arbitrary value | `O(n)` | Heap order cannot direct the search |
| Build by repeated insertion | `O(n log n)` | Inserts all values separately |
| Bottom-up heapify | `O(n)` | Available only if the chosen API provides it |

The structure stores `O(n)` values. A streaming top-k algorithm stores `O(k)` candidates rather than all `n` inputs, though its output also needs `O(k)` memory. In PHP, values may be zvals, arrays, strings, or object references, and an SPL object's internal bytes are an implementation detail. Do not infer exact byte usage from the abstract array formula or assume SPL is always more memory-efficient than a PHP array. Measure representative data with the application's PHP version and allocator; Chapter 74 discusses peak live memory and Chapter 56 covers PHP-managed allocation.

## PHP, the Database, and Workload Boundaries

Use an in-process heap when PHP owns a bounded set of candidates and repeatedly needs the current extremum: a top-k calculation, a simulation, or local coordination inside one worker. It is a poor replacement for an indexed database query when the source of truth is already persisted. For example, selecting the next 100 due jobs should usually use a database predicate and an index that matches the query, with a transaction or locking rule so concurrent workers do not claim the same rows. Pulling every job into PHP and heapifying it wastes network, memory, and freshness.

Likewise, a heap cannot promise a globally earliest task across multiple independent process heaps. It only orders the values currently present in its own instance. A database, shared queue, or broker must own coordination when multiple workers can produce or consume the work. Chapter 78 introduces FIFO queues and backpressure; Chapter 84 covers priority queues and Chapter 91 compares structures by workload.

## Testing and Failure

Test the invariant through externally visible behavior rather than asserting a particular internal array layout:

1. Insert values in ascending, descending, and mixed order; repeatedly extract and verify a nondecreasing sequence from a min-heap or nonincreasing sequence from a max-heap.
2. Test one value, duplicate values, negative values, and the empty state. Confirm that callers guard `top()` and `extract()` where an empty heap is allowed.
3. For a custom comparator, test the sign in both argument orders, equality, and transitivity over representative values. Check that tie-breakers create the documented order.
4. Test top-k for `k = 0`, `k = 1`, `k > n`, duplicates, negative inputs, and values equal to the current cutoff. Compare the result to a full sort on small generated data.
5. Test mutable-ordering hazards: once an item is inserted, do not mutate fields used by the comparator. Replace or rebuild items when priority changes.
6. If comparator exceptions are possible, test the recovery policy explicitly. Do not treat `recoverFromCorruption()` as proof that the heap invariant has been restored.

Property-based tests can generate short value lists and assert that extracting every element yields the same multiset as the input in sorted order. This catches mistakes in both the invariant repair and boundary cases without pinning the test to one valid heap shape.

## Common Mistakes

- Assuming every parent-child heap is globally sorted or using a heap where ordered traversal or binary search is required.
- Searching for arbitrary elements in `O(log n)`; the heap has no search-order invariant.
- Confusing `SplMinHeap` with `SplMaxHeap` or forgetting that `SplPriorityQueue` uses max-heap priority semantics.
- Assuming equal-priority values leave in insertion order. Heap ties are not stable unless a tie-breaker is part of the comparison key.
- Depending on a mutable priority field after insertion and thereby invalidating the heap's ordering.
- Using a comparator that is inconsistent, stateful, or throws during sift operations.
- Expecting `foreach ($heap ...)` to preserve the heap; iteration advances by removing the top item.
- Treating an in-memory heap as durable, shared, or concurrency-safe scheduling infrastructure.
- Choosing a heap for top-k when `k` is almost the full input size without comparing it to a straightforward sort.

## Senior Engineer Thinking

Start from the operation the application needs. If it must repeatedly take the smallest or largest item while values are added, a heap is a strong candidate. If it needs every item in order, needs arbitrary range queries, or frequently updates arbitrary priorities, another representation may fit better.

Then write down the comparison contract, including type normalization, direction, and ties. Keep fields used for ordering immutable while stored. Decide what an empty heap means, who owns the candidates, and whether the data must survive process failure or be coordinated across workers. Those are application guarantees; the heap only maintains an extremum among the values inserted into one instance.

Finally, compare the real costs. A heap reduces retained memory for top-k and limits each adjustment to a path, but PHP object allocation, input hydration, sorting constants, and database indexes may dominate. Measure realistic data and keep persistence and concurrency invariants at the system boundary that can enforce them.

## Exercises

1. For the min-heap `[2, 5, 3, 9, 7, 8]`, calculate the parent and child indexes for each valid position. Show the array after inserting `1` and after removing the root.
2. Give an example of a complete binary tree that is not a valid min-heap. Then give a heap whose values are not in sorted array order.
3. A stream contains product records with `score` and `id`. Define which record wins when scores tie, then adapt the top-k comparison key and state its time and memory bounds.
4. A scheduled task's `runAt` can change after insertion. Describe two ways to update the schedule while preserving the invariant, including the stale-entry strategy.
5. A PHP worker reads due jobs from a database and also keeps a local min-heap. Which system owns durability and cross-worker claims? What can the local heap safely optimize?

## Review Questions

1. What two properties define a binary heap, and what does completeness buy the array representation?
2. Why does the root contain the extremum while siblings and separate branches need not be ordered?
3. What are the parent and child index formulas for a zero-based binary heap?
4. Why are insertion and root removal `O(log n)`, but arbitrary-value search `O(n)`?
5. How do `SplMinHeap` and `SplMaxHeap` differ? What happens when `SplHeap::next()` advances an iterator?
6. Why should a custom comparator be deterministic and non-throwing? What does `recoverFromCorruption()` not promise?
7. Why does top-k use a min-heap when retaining the largest `k` values?
8. Which guarantees must a database or message broker provide when scheduling spans processes or restarts?

## Summary

A binary heap is a complete binary tree with a local parent-child ordering rule. A min-heap exposes its minimum at the root; a max-heap exposes its maximum. A contiguous zero-indexed array can represent the shape using arithmetic indexes, and sift-up/sift-down operations restore the invariant in logarithmic time. A heap does not keep all elements sorted and does not make arbitrary searches fast. PHP's `SplMinHeap`, `SplMaxHeap`, and `SplHeap` expose these semantics, with important behavior around empty access, destructive iteration, comparator ties, and corruption. Heaps suit repeated-extremum and bounded top-k workloads; durability, cross-worker coordination, and database selection remain responsibilities of the system that owns the data.

## References

- [PHP Manual: `SplHeap`](https://www.php.net/manual/en/class.splheap.php)
- [PHP Manual: `SplMinHeap`](https://www.php.net/manual/en/class.splminheap.php)
- [PHP Manual: `SplMaxHeap`](https://www.php.net/manual/en/class.splmaxheap.php)
- [PHP Manual: `SplPriorityQueue`](https://www.php.net/manual/en/class.splpriorityqueue.php)
- [PHP Manual: `SplHeap::compare()`](https://www.php.net/manual/en/splheap.compare.php)
- [PHP Manual: `SplMinHeap::compare()`](https://www.php.net/manual/en/splminheap.compare.php)
- [PHP Manual: `SplMaxHeap::compare()`](https://www.php.net/manual/en/splmaxheap.compare.php)
- [PHP Manual: `SplHeap::top()`](https://www.php.net/manual/en/splheap.top.php)
- [PHP Manual: `SplHeap::extract()`](https://www.php.net/manual/en/splheap.extract.php)
- [PHP Manual: `SplHeap::next()`](https://www.php.net/manual/en/splheap.next.php)
- [PHP Manual: `SplHeap::recoverFromCorruption()`](https://www.php.net/manual/en/splheap.recoverfromcorruption.php)
- [Chapter 74 — Memory Complexity](./074-memory-complexity.md)
- [Chapter 77 — Stacks](./077-stacks.md)
- [Chapter 78 — Queues](./078-queues.md)
- [Chapter 79 — Sorting](./079-sorting.md)
- [Chapter 81 — Binary Search](./081-binary-search.md)
- [Chapter 84 — Priority Queues](./084-priority-queues.md)
- [Chapter 91 — Choosing the Right Data Structure](./091-choosing-the-right-data-structure.md)
