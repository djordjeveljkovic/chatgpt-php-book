---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 57
title: Garbage Collection
slug: garbage-collection
status: complete
summary: ../../_ai/chapter-summaries/057-garbage-collection-summary.md
---

# Chapter 57 — Garbage Collection

## Why This Matters

Most PHP requests are short-lived. At request shutdown, the engine can discard the request's remaining state, so developers may never notice how values become unreachable. Long-running workers, daemons, queue consumers, and server processes remove that convenient boundary. A reference cycle that would have disappeared at the end of a request can remain in memory until PHP's cycle collector finds it.

Garbage collection is not a synonym for “PHP frees everything immediately.” PHP primarily uses reference counting for prompt reclamation and supplements it with a cycle collector for certain cyclic structures. That distinction explains both ordinary behavior and the characteristic memory growth of a worker that retains cycles, listeners, closures, or caches.

## Mental Model

```text
value becomes unreachable
    ├─ refcount reaches zero → release immediately (normally)
    └─ cycle keeps refcounts non-zero → cycle collector must detect it
```

The stable language-level guarantee is about reachability and observable behavior. The exact root buffers, colors, thresholds, and traversal details are Zend implementation details. They have changed over PHP history and should be learned from the source for the target release, not treated as an application API.

Cycle collection also does not compensate for an application that still has a live path to an object. If a global registry, queue, cache, or closure can reach a value, that value is not garbage even if the programmer no longer intends to use it.

## Core Concept: Reference Counting and Cycles

Consider two ordinary values:

```php
$first = ['name' => 'Ada'];
$second = $first;

unset($first);  // the value remains reachable through $second
unset($second); // it can now be released
```

For refcounted values such as arrays, strings, and objects, the engine tracks references to the payload. The exact representation is covered in Chapters 47, 51, 52, and 53. A cycle defeats simple decrementing:

```php
$a = [];
$b = [];
$a['peer'] =& $b;
$b['peer'] =& $a;

unset($a, $b);
```

The local variables are gone, but the two structures can still refer to one another. A pure reference-counting scheme would see nonzero counts forever. PHP's cycle collector identifies candidate cycles and determines whether the values can be reclaimed.

The example uses references to make the cycle obvious. Object graphs create the same class of problem:

```php
final class Node
{
    public ?Node $parent = null;
    public array $children = [];
}

$parent = new Node();
$child = new Node();
$child->parent = $parent;
$parent->children[] = $child;

unset($parent, $child);
```

## How It Works

Conceptually, the collector does not scan every byte of every value on every assignment. Refcounting marks values as possible cycle candidates when relationships are changed. Candidate structures are placed into internal bookkeeping, and a later collection traverses relevant references to decide whether a path from a root keeps them alive.

This is a simplified model:

```text
candidate buffer fills
    → collector examines possible cyclic structures
    → references reachable from roots are preserved
    → unreachable cycles are destroyed
```

The collector's trigger policy and internal data structures are version-sensitive. `gc_status()` exposes counters useful for diagnosis, but its fields and meanings can evolve. Consult the manual for the PHP version deployed and avoid parsing undocumented output as a permanent contract.

## What PHP Does

The `gc_*` functions are diagnostic and control tools for advanced cases:

```php
<?php

declare(strict_types=1);

$before = gc_status();

for ($i = 0; $i < 10_000; $i++) {
    $a = new stdClass();
    $b = new stdClass();
    $a->peer = $b;
    $b->peer = $a;
    unset($a, $b);
}

$collected = gc_collect_cycles();
$after = gc_status();

var_dump([
    'collected' => $collected,
    'before' => $before,
    'after' => $after,
]);
```

`gc_collect_cycles()` requests a collection and returns the number of collected cycles. It is a useful experiment and sometimes a deliberate checkpoint in a long-running process, but it is not a substitute for removing an accidental owner. `gc_disable()` and `gc_enable()` affect automatic cycle collection for the current process; disabling collection around a carefully bounded operation can reduce interruptions, but leaving it disabled in a worker is dangerous.

`gc_mem_caches()` is related to allocator caches, not cycle detection. It may release memory held by the memory manager for reuse, but it is not a general “free all PHP memory” operation. These distinctions prevent misleading runbooks.

## Minimal Example: A Real Owner Versus a Cycle

This registry retains objects by design:

```php
final class ListenerRegistry
{
    /** @var list<Closure> */
    private array $listeners = [];

    public function add(Closure $listener): void
    {
        $this->listeners[] = $listener;
    }
}

$registry = new ListenerRegistry();
$service = new stdClass();
$registry->add(static function () use ($service): void {
    // The registry owns the closure, and the closure owns $service.
});
```

Calling `gc_collect_cycles()` cannot collect `$service` while `$registry` remains reachable. The fix is an unregister operation, scope-limited registry, or weak association—not a more frequent collection.

PHP 7.4 introduced `WeakReference`, and PHP 8.0 introduced `WeakMap`. These provide ways to associate metadata with objects without making the object strongly reachable. They are useful for caches and side tables, but weak references change semantics: the target can disappear at any time after its strong references are gone. Code must tolerate a missing target.

```php
<?php

$metadata = new WeakMap();
$object = new stdClass();
$metadata[$object] = ['seen' => time()];

unset($object);
// The corresponding WeakMap entry can disappear when the object is collected.
```

The `time()` value here is only illustrative; production code should use an injected clock when time affects behavior.

## Practical Example: A Queue Worker

A worker's memory after each job can be modeled as:

```text
baseline process
 + current job graph
 + intentionally retained cache
 + accidentally retained graph
 + allocator-reserved pages
```

If the baseline rises after every job, inspect what remains reachable. Common causes include static arrays, global service locators, logging context, event subscribers, closures capturing a container, and libraries with process-wide caches. If logical usage returns but RSS rises and stabilizes, allocator reuse or native-library behavior may explain it. If both rise without a plateau, investigate retention first.

An explicit worker boundary is often the safest mitigation:

```php
for (;;) {
    $job = $queue->receive();
    if ($job === null) {
        break;
    }

    try {
        handle($job);
    } finally {
        unset($job);
        gc_collect_cycles(); // A measured checkpoint, not a universal cure.
    }
}
```

A production worker should also have a graceful recycle policy based on job count, age, or measured memory. Recycling limits the blast radius of leaks in extensions and dependencies that the application cannot reclaim. It does not remove the need to find the leak.

## Destructors and Collection

Destructors make object graphs more operationally sensitive. Destruction can run user code, perform I/O, throw or trigger errors depending on the situation, and observe partially torn-down state. Do not use destructors as a transaction boundary or as the only place to release an external lock. Prefer explicit `close()`, `commit()`, or scope-managed abstractions whose behavior is tested.

The order in which a complex cyclic graph is destroyed is not a safe coordination mechanism. If two destructors depend on one another, the design is already fragile. Keep destructors small and idempotent where they are unavoidable.

## Bad Example

```php
final class Worker
{
    private static array $processed = [];

    public function run(Job $job): void
    {
        self::$processed[] = $job;
        handle($job);
    }
}
```

No collector can reclaim jobs still reachable from the static property. The array is a cache with no eviction policy, whether or not the author intended it to be.

## Better Example

Keep only the bounded information required for diagnostics:

```php
final class RecentJobIds
{
    /** @var list<string> */
    private array $ids = [];

    public function add(string $id, int $capacity = 1_000): void
    {
        $this->ids[] = $id;
        if (count($this->ids) > $capacity) {
            array_shift($this->ids);
        }
    }
}
```

This is bounded in cardinality, although `array_shift()` is O(n) for an ordinary PHP array and may be a poor hot-path choice. A ring buffer or `SplFixedArray` may change the trade-off; benchmark with the actual workload. The important correction is ownership and an eviction policy, not the particular container.

## Performance

Cycle detection has a cost, especially when an application creates many candidate graphs. For normal request/response work, optimize object lifetimes and data flow before changing GC settings. In a worker, measure job latency, collection duration, collected cycles, memory usage, RSS, and queue throughput together. A manual collection that reduces RSS but adds enough pause time to violate latency objectives is not automatically an improvement.

Avoid calling `gc_collect_cycles()` after every small object. That can turn amortized work into repeated scans. A measured batch checkpoint, a worker recycle, or eliminating the cycle is usually more robust.

## Security and Reliability

An attacker who can cause a worker to create many cyclic graphs or unbounded retained state can create memory pressure and reduce throughput. Enforce limits on batch size, nesting, uploaded data, and job payloads. Never rely on garbage collection to defend against an input that the application intentionally retains.

Weak references can prevent cache ownership from becoming a denial-of-service vector, but they can also create race-like “entry disappeared” behavior within a process. Handle absence explicitly and never use a weak map as the sole source of durable state.

## Testing and Verification

Write tests that distinguish a leak from a delayed collection:

1. Create a known cyclic graph and call `gc_collect_cycles()`; assert that the returned count is positive in an isolated experiment, without asserting a precise count for an unrelated graph.
2. Run many jobs in a subprocess and record memory after each batch.
3. Compare a version with a static owner, a version with explicit removal, and a version with a weak association.
4. Test that unregistering a listener removes its strong path to the service.
5. Test worker shutdown and restart behavior rather than assuming request teardown will happen in production.

Use `gc_status()` for counters and `memory_get_usage()` for PHP-managed measurements. Collect worker RSS from the process supervisor or operating system. Pin PHP versions when treating metrics as regression thresholds because GC status fields and allocator behavior are not universal.

## Common Mistakes

- Calling every memory problem a garbage-collection problem.
- Expecting a cycle collector to remove objects that a cache or registry still owns.
- Disabling GC in a long-running worker without a bounded plan.
- Treating `gc_mem_caches()` as cycle collection.
- Using destructors for critical business transactions.
- Assuming weak references keep an object alive long enough to use it.
- Measuring only end-of-request memory for a process that never ends.

## Senior Engineer Thinking

Draw the ownership graph. Mark strong edges, weak edges, globals, statics, caches, and external resources. Ask whether an object is reachable by design or merely reachable because cleanup was forgotten. Then choose among explicit removal, bounded retention, weak association, process recycling, and a code fix.

Garbage collection is a runtime safety net for a particular class of unreachable graph. It cannot decide that a live cache entry is “old,” that a job should be retried, or that an external connection is no longer needed. Those are application and operations policies.

## Exercises

1. Create a cyclic object graph and compare memory before and after `gc_collect_cycles()`.
2. Implement an event subscriber with `remove()`. Prove with a worker-style loop that removal changes retention.
3. Replace a metadata array keyed by object IDs with `WeakMap`. Test what happens after the last strong reference is removed.
4. Add a memory-based recycle threshold to a toy worker. Explain why it is a safety valve rather than proof that no leak exists.

## Review Questions

1. Why does reference counting reclaim many values promptly?
2. Why can a cycle defeat reference counting?
3. How is a live cache owner different from an unreachable cycle?
4. When is `gc_collect_cycles()` useful, and why is it not a universal fix?
5. What problem do `WeakReference` and `WeakMap` solve, and what semantic cost do they introduce?
6. Why should worker RSS and logical PHP memory both be measured?

## Summary

PHP combines prompt reference-count reclamation with cycle collection for unreachable cyclic graphs. A collector cannot reclaim values still owned by statics, caches, registries, or closures. Long-running processes need explicit ownership, bounded retention, weak associations where appropriate, measured collection checkpoints, and often a recycle policy. Diagnose reachability before tuning GC.

## Official References

- [PHP garbage collection](https://www.php.net/manual/en/features.gc.php)
- [`gc_collect_cycles()`](https://www.php.net/manual/en/function.gc-collect-cycles.php)
- [`gc_status()`](https://www.php.net/manual/en/function.gc-status.php)
- [`WeakReference`](https://www.php.net/manual/en/class.weakreference.php)
- [`WeakMap`](https://www.php.net/manual/en/class.weakmap.php)
- [PHP source: garbage collector](https://github.com/php/php-src/blob/master/Zend/zend_gc.c)
