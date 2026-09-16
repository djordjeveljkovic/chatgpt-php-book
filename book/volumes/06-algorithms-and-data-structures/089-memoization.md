---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 89
title: Memoization
slug: memoization
status: complete
summary: ../../_ai/chapter-summaries/089-memoization-summary.md
---

# Chapter 89 — Memoization

## Why This Matters

Some algorithms reach the same subproblem by several paths. A dependency graph may ask for the remaining duration of a shared prerequisite from multiple parents. A recursive parser may revisit the same position and grammar rule. Recomputing the same deterministic answer wastes work, and in a branching recursion the repeated work can grow exponentially.

Memoization stores the result of a computation under a key for its inputs. When the same inputs occur again, the program returns the stored result instead of performing the computation again. This can reduce work dramatically, but only when the cache key identifies equivalent inputs and the result is still valid for the data being queried.

Memoization is a technique for avoiding repeated computation. It does not inherently provide a time-to-live, cross-process sharing, durability, refresh, or eviction. Those are concerns of a cache system. This chapter focuses on bounded in-process memoization and the assumptions that make reuse correct.

## Mental Model

```text
complete, canonical inputs ──► deterministic computation ──► result
          │                                               │
          └──────────── cache key ─── memo table ◄────────┘
                                         │
                            same key: return prior result
```

The memo table is an index from a computation key to its result. A hit is correct only if the key represents every input that can change the answer. If the data changes while the key remains the same, the entry is stale. If two different inputs accidentally produce the same key, one result can be returned for the wrong problem.

## Core Concept

Memoization is most straightforward for a deterministic computation whose result depends only on explicit inputs. For a fixed input, the computation returns the same value and does not perform a side effect that must happen once per call. A pure mathematical function is an easy example. A function that reads the current time, mutable global state, an unversioned database row, or a random value has hidden inputs; memoizing it by only its declared arguments can return an answer from the wrong state.

Let **S** be the number of distinct subproblems actually reached and **T** the total work without reuse. Memoization performs the computation once per distinct key, then pays lookup and key-construction cost for later calls. If each state examines outgoing transitions, total work is often O(S + transitions) rather than expanding the same subtree repeatedly. The memo table uses O(S) entries. If there are no repeated subproblems, lookup and memory overhead may make the result slower and larger than the direct computation.

Memoization is often called **top-down dynamic programming**: start from the requested result, recurse into smaller subproblems, and remember answers. **Tabulation** or **bottom-up dynamic programming** computes those answers in an order where prerequisites are already available, usually with iteration. Both approaches reuse subproblem results; the difference is how the evaluation order is produced and whether recursion is used.

## Choose and Canonicalize the Key

Before adding a cache, list every input that can change the result. Depending on the calculation, the key may need:

- all function arguments that affect the result;
- an algorithm or schema version when result meaning changes across deployments;
- a tenant, locale, feature configuration, permissions version, or currency;
- a version or snapshot identifier for mutable source data.

Then define equality. Are `"17"` and `17` the same identifier? Are email addresses case-insensitive? Do two arrays with different key insertion order represent the same set of options? Normalize values according to the domain before building a key. Do not use PHP's loose comparison rules as an accidental identity policy.

PHP arrays accept integer and string keys, and integer-looking decimal strings can be converted to integer keys. A memo table keyed directly by external string IDs can therefore blur representations such as a canonical numeric string and an integer. Prefix string IDs or encode their type explicitly. Chapter 85 uses the same prefixed-key approach for graph vertices.

For several string inputs, concatenate them with an unambiguous encoding. A separator alone is unsafe when the values may contain that separator; length-prefixing each field avoids that ambiguity:

```php
<?php
declare(strict_types=1);

function scheduleMemoKey(string $snapshotId, string $taskId): string
{
    return 'critical-path:v1:'
        . strlen($snapshotId) . ':' . $snapshotId
        . ':' . strlen($taskId) . ':' . $taskId;
}
```

The fixed prefix keeps the PHP array key a string. Lengths make the component boundaries unambiguous, and the versioned prefix gives the key a namespace. A key is not correct merely because it is unique: it must also include every dependency of the result. In this example, `snapshotId` must change whenever task durations or dependency edges change. If no trustworthy version exists, keep the memo table local to one immutable snapshot and discard it when that snapshot changes.

For structured input, first decide its semantic equality and canonical form. A list may be sorted if order is irrelevant; an ordered sequence must keep its order. Encoding arbitrary objects or unnormalized arrays directly can be expensive and can preserve irrelevant representation differences. Hashing a canonical encoding can make the resulting key shorter, but hashing does not fix a missing input, inconsistent normalization, or an unsafe cache lifetime.

## Top-Down Example: Critical Path in a Dependency DAG

Suppose a release plan is a directed acyclic graph of tasks. Each task has an estimated duration, and outgoing edges point to tasks that follow it. The remaining critical-path duration for a task is its duration plus the longest remaining duration among its successors. If several branches share a later task, a naive recursion calculates that shared task repeatedly.

The schedule below stores exact string IDs behind prefixed array keys. The memo key includes the snapshot version and task ID. An active-recursion set detects a cycle, because a cycle would make this recurrence recurse without reaching a base case.

```php
<?php
declare(strict_types=1);

function taskArrayKey(string $taskId): string
{
    // Prevent integer-like string IDs from becoming PHP integer array keys.
    return 'task:' . $taskId;
}

/**
 * @param array<string, int> $durationByKey
 * @param array<string, list<string>> $successorsByKey
 * @param array<string, int> $memo
 * @param array<string, true> $active
 */
function remainingCriticalPath(
    string $taskId,
    string $snapshotId,
    array $durationByKey,
    array $successorsByKey,
    array &$memo,
    array &$active,
): int {
    $memoKey = scheduleMemoKey($snapshotId, $taskId);
    if (array_key_exists($memoKey, $memo)) {
        return $memo[$memoKey];
    }

    if (isset($active[$memoKey])) {
        throw new LogicException('The task dependencies contain a cycle.');
    }

    $taskKey = taskArrayKey($taskId);
    if (!array_key_exists($taskKey, $durationByKey)) {
        throw new InvalidArgumentException("Unknown task: {$taskId}");
    }
    $duration = $durationByKey[$taskKey];
    if (!is_int($duration) || $duration < 0) {
        throw new InvalidArgumentException('Task durations must be non-negative integers.');
    }

    $active[$memoKey] = true;
    try {
        $longestSuccessor = 0;
        foreach ($successorsByKey[$taskKey] ?? [] as $successorId) {
            $longestSuccessor = max(
                $longestSuccessor,
                remainingCriticalPath(
                    $successorId,
                    $snapshotId,
                    $durationByKey,
                    $successorsByKey,
                    $memo,
                    $active,
                ),
            );
        }

        if ($duration > PHP_INT_MAX - $longestSuccessor) {
            throw new OverflowException('Critical-path duration exceeds the integer range.');
        }

        return $memo[$memoKey] = $duration + $longestSuccessor;
    } finally {
        unset($active[$memoKey]);
    }
}

$snapshotId = 'release-42';
$durationByKey = [
    taskArrayKey('schema') => 2,
    taskArrayKey('api') => 5,
    taskArrayKey('worker') => 3,
    taskArrayKey('release') => 2,
];
$successorsByKey = [
    taskArrayKey('schema') => ['api', 'worker'],
    taskArrayKey('api') => ['release'],
    taskArrayKey('worker') => ['release'],
    taskArrayKey('release') => [],
];

$memo = [];
$active = [];
$days = remainingCriticalPath(
    'schema',
    $snapshotId,
    $durationByKey,
    $successorsByKey,
    $memo,
    $active,
);
// $days is 9. The shared release task is computed once and then reused.
```

`array_key_exists()` is deliberate: a valid cached result may be `0` or `null`, so a truthiness check or `isset()` is not a general cache-hit test. This function returns integers, but the rule matters when memoizing nullable results; Chapter 80 explains the distinction between a missing key and a present key with `null`.

The arrays are treated as one immutable graph snapshot for the duration of calls using this memo table. Reusing `$memo` for another snapshot requires a different `$snapshotId`. If code mutates graph data without changing the version, the key still matches and the old answer is returned. A cycle triggers an exception before the active node is memoized; callers should treat the calculation as failed rather than using a partial result as a valid schedule.

This is a scheduling estimate, not a resource-constrained project plan. It assumes successor relationships capture all precedence constraints and that task durations compose along a path. It throws if a summed duration exceeds PHP's integer range. Machine capacity, calendars, retries, and resource contention require additional state and policy.

## Iterative Alternative: Bottom-Up Evaluation

Top-down recursion visits only states needed by the requested task, but recursion depth follows the longest dependency chain. A chain with many tasks can use substantial call-stack space. If a topological order is available, process it in reverse so each successor's result is ready before its predecessor:

```php
<?php
declare(strict_types=1);

/**
 * The order must contain every task once in a valid source-to-sink topological order.
 * @param list<string> $topologicalOrder
 * @param array<string, int> $durationByKey
 * @param array<string, list<string>> $successorsByKey
 * @return array<string, int> task array key => remaining critical-path duration
 */
function remainingCriticalPathsBottomUp(
    array $topologicalOrder,
    array $durationByKey,
    array $successorsByKey,
): array {
    if (!array_is_list($topologicalOrder)) {
        throw new InvalidArgumentException('Topological order must be a list.');
    }

    $remainingByKey = [];
    $seen = [];

    for ($index = count($topologicalOrder) - 1; $index >= 0; --$index) {
        $taskId = $topologicalOrder[$index];
        $taskKey = taskArrayKey($taskId);
        if (isset($seen[$taskKey])) {
            throw new LogicException('Each task must appear once in the topological order.');
        }
        $seen[$taskKey] = true;

        if (!array_key_exists($taskKey, $durationByKey)) {
            throw new InvalidArgumentException("Unknown task in order: {$taskId}");
        }
        $duration = $durationByKey[$taskKey];
        if (!is_int($duration) || $duration < 0) {
            throw new InvalidArgumentException('Task durations must be non-negative integers.');
        }

        $longestSuccessor = 0;
        foreach ($successorsByKey[$taskKey] ?? [] as $successorId) {
            $successorKey = taskArrayKey($successorId);
            if (!array_key_exists($successorKey, $remainingByKey)) {
                throw new LogicException('The supplied order is incomplete or not topological.');
            }
            $longestSuccessor = max($longestSuccessor, $remainingByKey[$successorKey]);
        }

        if ($duration > PHP_INT_MAX - $longestSuccessor) {
            throw new OverflowException('Critical-path duration exceeds the integer range.');
        }

        $remainingByKey[$taskKey] = $duration + $longestSuccessor;
    }

    if (count($remainingByKey) !== count($durationByKey)) {
        throw new LogicException('The topological order must contain every task exactly once.');
    }

    return $remainingByKey;
}

$topologicalOrder = ['schema', 'api', 'worker', 'release'];
$remainingByKey = remainingCriticalPathsBottomUp(
    $topologicalOrder,
    $durationByKey,
    $successorsByKey,
);
// $remainingByKey[taskArrayKey('schema')] is 9.
```

The iterative version avoids recursive call state, but computes every task in the supplied order, even if only one root is queried. It checks that each task appears once and that the order contains every task; a successor that has not yet been evaluated is rejected. [Chapter 85](./085-graphs.md) covers graph models and cycle detection. If the graph changes, rebuild the order and any memoized results from the new snapshot.

The example uses `array_is_list()`, available since PHP 8.1, to check its positional input contract once. On earlier PHP versions, validate consecutive integer keys with an equivalent check or rely on a documented and enforced caller contract.

## Memoization, Invalidation, and Memory

Memoization trades memory for less repeated computation. If there are S distinct keys, the memo table stores O(S) results, plus keys and PHP array overhead. If each result is a large object or nested array, the retained result graph may cost much more than one scalar per entry. Returning a large result can also keep data alive longer than the request would otherwise need it.

Choose a lifetime that matches the data:

- **One call or request:** local arrays are easy to discard and need no cross-request invalidation. Reuse them only while the underlying inputs remain the same.
- **One immutable snapshot:** scope the memo table to the object that owns that snapshot, or include an immutable snapshot/version ID in every key.
- **Mutable inputs:** clear affected entries, advance the version, or build a new memo table. Selective invalidation may require reverse dependencies so changed nodes can invalidate all results that depend on them.
- **Long-running worker:** bound entries or reset the table between jobs. A cache that grows with every unique input can eventually exhaust worker memory.

Versioned keys prevent old and new snapshot results from colliding, but they do not reclaim the old entries. If a worker retains multiple versions, memory grows across versions. Clear old versions or use an explicit maximum size and eviction policy. Eviction changes hit rate, not correctness, provided every miss recomputes from current inputs.

PHP arrays are convenient memo tables, but each entry has HashTable metadata in addition to its key and value. Use a scalar result or compact record when possible. Measure peak memory when the state space is large, and consider whether a database query, a streaming algorithm, or recomputation with less retained state better fits the workload. [Chapter 74](./074-memory-complexity.md) distinguishes live input, output, and auxiliary memory.

## Runtime and Operational Boundaries

A local memo table belongs to the PHP process that created it. In a normal PHP-FPM request, local variables are not a cache shared with the next request or another worker. A long-running CLI worker may retain object or static state across jobs, so the application must choose when to clear it. Process lifetime affects reuse and memory; it does not make an in-memory value durable.

Memoization also does not replace a shared cache when many processes need to reuse results. Redis, a database table, or another shared store introduces serialization, network failures, expiry, consistency, and eviction decisions. Those systems may be useful, but their behavior is broader than memoization. Keep the authoritative data in its proper source and treat cached answers as derived values with an explicit freshness policy.

Do not memoize a computation that performs required side effects. A cache hit skips the function body, so sending an email, charging a card, writing an audit event, or updating a counter inside a memoized function would silently suppress work on repeated calls. Separate deterministic calculation from effects, then memoize only the calculation if repeated inputs are common.

## Performance and Complexity

For the DAG example, let V be the number of reachable tasks and E the outgoing dependency edges they reference. With memoization, each task's duration is computed once and each edge is examined once, giving O(V + E) time under the usual array-lookup model. The memo and active-state maps use O(V) space. Recursive call state uses O(D), where D is the maximum dependency depth and can be O(V) for a chain.

The iterative version also takes O(V + E) time and stores O(V) results, but has no recursive call stack. It processes all tasks in the supplied topological order. A plain recursion without memoization can expand the same shared branch repeatedly; on layered DAGs with many converging paths, work may be exponential in the number of levels even though the graph itself is small.

These bounds count state and edge operations, not arbitrary PHP memory bytes or wall-clock time. Key construction, string length, hashing, result copying, PHP array overhead, input validation, and cache cleanup all have cost. If the same state is rarely requested twice, a direct traversal may be easier and faster. Benchmark representative input shapes, including a deep chain and a highly shared graph.

## Testing

Test the computation contract and the memo's scope:

1. Use a small DAG with a shared successor. Compare memoized output with a simple reference calculation and confirm the shared state appears only once in the memo.
2. Test a leaf task, a zero-duration task, an unknown task ID, an empty successor list, a cycle, and integer overflow in accumulated durations.
3. Use numeric-looking IDs such as `"17"` and `"017"`; verify that the canonical key scheme keeps them distinct.
4. Call the same state twice under one snapshot and assert the same result. Change a duration or edge and advance the snapshot version; verify that the updated result is computed.
5. Test nullable or false-like results in generic memo helpers and distinguish a cache miss from a cached `null`, `false`, or `0`.
6. Compare bottom-up and top-down results on generated small DAGs with valid topological orders.
7. Exercise a deep chain to determine whether recursion depth is appropriate for the application's input bound.
8. For long-running workers, test the configured entry limit or reset behavior and observe memory after many distinct keys.

Property-based checks can generate small DAGs by only adding edges from an earlier to a later vertex. This guarantees acyclicity and permits comparison against a brute-force path enumerator for small inputs. Do not use fragile wall-clock thresholds as correctness assertions; use a benchmark for performance and a deterministic count of computed states for reuse properties.

## Common Mistakes

- Memoizing a function whose result depends on hidden mutable state, time, randomness, or side effects.
- Omitting a meaningful argument, tenant, locale, configuration, or source version from the key.
- Using a delimiter-based key without escaping or length-prefixing fields.
- Treating PHP array keys as typed identities when numeric strings can be converted.
- Using `isset()` when `null` is a valid cached result.
- Assuming memoization persists between PHP requests or is shared across workers.
- Reusing results after their source snapshot changes without invalidation or versioning.
- Allowing an unbounded memo table in a long-running worker.
- Assuming recursion plus memoization removes recursion-depth risk.
- Memoizing when subproblems do not overlap enough to repay key and memory overhead.

## Senior Engineer Thinking

Start with the computation, not the cache: what is one subproblem, what inputs determine its answer, and which states overlap? Then choose a canonical key and a lifetime tied to the source data. Decide how changes invalidate old results and how many distinct states can be retained.

Finally, pick the evaluation order. Top-down recursion is concise when only a small portion of a DAG is needed and depth is bounded. Bottom-up iteration is predictable for deep inputs when a topological or other dependency order exists. If neither reuse nor a simple dependency order exists, adding a memo table may only add complexity. Treat performance as a measured property of the workload, not an automatic benefit of caching.

## Exercises

1. Add a counter to `remainingCriticalPath()` and show that the shared `release` node is evaluated once when reaching it from both `api` and `worker`.
2. Implement a memoized Fibonacci function and a bottom-up version. Compare their results and state the time, result-table space, and recursion-stack space for each.
3. Extend `scheduleMemoKey()` to include tenant ID and scheduling policy version. Specify how you normalize each field and show that different tuples cannot collide.
4. Change one task duration. Implement coarse invalidation by changing the snapshot ID, then describe how selective invalidation would find dependent ancestors.
5. Adapt the graph calculation to compute a boolean “can reach release” result. Test a cached `false` value to ensure it is not mistaken for a miss.
6. Measure a deeply nested dependency chain. Compare the recursive implementation with bottom-up evaluation and explain which bound determines the safe choice.

## Review Questions

1. What property of a computation makes memoization safe?
2. How does memoization differ from general caching?
3. Which inputs belong in a memo key, and what can happen if one is omitted?
4. Why does prefixing or encoding a string ID matter when using a PHP array as the memo table?
5. How does top-down memoization differ from bottom-up tabulation?
6. Why does memoization reduce repeated work but not necessarily reduce memory or recursion depth?
7. What invalidation options exist when input data changes?
8. Why is a process-local memo table neither shared nor durable?
9. When might memoization make a program slower or less reliable?

## Summary

Memoization stores deterministic computation results under canonical keys so repeated subproblems can reuse prior work. Correct reuse depends on including every meaningful input and tying the cache lifetime to the data snapshot. Top-down recursion visits only requested states but uses stack space proportional to dependency depth; bottom-up iteration avoids recursion when a dependency order is available. A memo table costs O(S) space for S distinct states, and PHP array overhead and retained result payloads matter. Memoization is not a guarantee of persistence, process sharing, freshness, or bounded growth; those properties require explicit lifetime, invalidation, and storage decisions.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: `array_key_exists()`](https://www.php.net/manual/en/function.array-key-exists.php)
- [PHP Manual: `array_is_list()`](https://www.php.net/manual/en/function.array-is-list.php)
- [Chapter 74 — Memory Complexity](./074-memory-complexity.md)
- [Chapter 80 — Searching](./080-searching.md)
- [Chapter 81 — Binary Search](./081-binary-search.md)
- [Chapter 85 — Graphs](./085-graphs.md)
- [Chapter 86 — Intervals](./086-intervals.md)
- [Chapter 90 — Streaming Algorithms](./090-streaming-algorithms.md)
- [Chapter 91 — Choosing the Right Data Structure](./091-choosing-the-right-data-structure.md)
