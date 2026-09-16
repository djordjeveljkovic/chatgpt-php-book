---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 84
title: Priority Queues
slug: priority-queues
status: complete
summary: ../../_ai/chapter-summaries/084-priority-queues-summary.md
---

# Chapter 84 — Priority Queues

## Why This Matters

A FIFO queue answers “which item arrived first?” A priority queue answers “which item should be handled next according to this ordering rule?” A scheduler may need the most urgent task, a timer may need the nearest deadline, and a search algorithm may need the most promising candidate. In each case, the next item can change whenever new work arrives. Sorting the entire collection after every insertion performs unnecessary work when the application only needs its current best item.

Priority queues are useful inside one PHP process for scheduling, simulation, best-first search, and retaining a small top-ranked subset from a large stream. They do not provide persistence, locking, worker coordination, retries, or delivery guarantees. Like the FIFO queue in [Chapter 78](./078-queues.md), an SPL priority queue is an in-memory data structure. A durable job system requires a broker or database-backed design, covered later in [Chapter 244](../../volumes/16-distributed-systems/244-queues.md).

## The Priority-Queue Contract

A priority queue stores entries and exposes the entry that ranks first under a comparison policy. Its basic operations are:

- **Insert:** add a value with a priority.
- **Peek:** inspect the next value without removing it.
- **Extract:** return and remove the next value.
- **Count or empty check:** inspect how much work is waiting.

The contract says which entry wins; it does not require a particular internal representation. A binary heap is a common implementation because it keeps the top entry available without keeping every entry fully sorted. [Chapter 83](./083-heaps.md) explains heap shape and its invariant. Here the focus is the queue behavior built on top of that structure.

Make the ordering rule explicit before choosing a PHP class. “Priority 10 beats priority 2” is one convention; “earliest timestamp wins” is another. Domain names such as `urgent`, `normal`, and `background` need a defined mapping to comparable values. Avoid relying on accidental PHP comparisons between unrelated types. Validate priorities at the boundary and compare values under one consistent policy.

Priority is also separate from fairness. If one high-priority task continually arrives, lower-priority work may never run. A scheduler may need aging, quotas, per-customer limits, or separate queues to prevent starvation. A priority queue applies its ordering policy; it does not decide whether that policy is fair.

## Ordering, Ties, and Changes

The comparison policy determines which entry comes first and what happens when priorities match. A max-priority queue treats larger priorities as more urgent. A min-priority queue treats smaller values—often timestamps or costs—as more urgent. The application should state this direction in names and tests rather than leave it implicit.

When equal priorities are possible, decide whether their relative order matters. If it does not, any equal-priority item may be extracted first. If it does, include a deterministic tie-breaker, usually an insertion sequence or domain key. A sequence number can make equal-priority entries FIFO among themselves, but it is only meaningful in the lifetime and scope where it is assigned. It does not establish fairness across processes or restarts.

PHP's `SplPriorityQueue` accepts repeated priorities and repeated values as separate entries. It does not replace an existing entry when the same value is inserted again. The PHP Manual explicitly says that equal-priority elements have undefined order and can leave in an order different from insertion order. Do not infer stability from one run.

Most heap-based priority queues do not provide efficient arbitrary lookup, removal, or priority update by job ID. Updating a queued task is commonly handled with **lazy invalidation**: insert a new entry carrying a version, record the newest version by ID, and ignore older entries when they reach the top. Cancellation can use the same version check. This preserves efficient insertion and extraction but may retain stale entries until they are popped; long-lived schedulers need a bound or a rebuild policy.

## Choosing Between a Scan, Sort, and Heap

The right structure depends on how often entries arrive, how many “next” items are requested, and whether all results must be ordered:

| Workload | Reasonable choice | Typical cost |
| --- | --- | --- |
| Find one best entry in an existing small array | Linear scan | O(n) time, O(1) extra space |
| Produce the complete ordered result once | Sort | O(n log n) time, O(n) retained input |
| Repeatedly insert and extract only the next entry | Priority queue / heap | O(log n) insert and extract; O(1) peek |
| Keep the best k values from a stream | Heap bounded to k entries | O(n log(k + 1)) time for k >= 1, O(k) structure space |

Repeatedly scanning an array for the best item costs O(n) per extraction, or O(n²) to drain all n items. Re-sorting after every insertion also repeats work. A heap is valuable when ordering changes as entries arrive and the program needs only a small prefix or the next item at each step.

If every entry must be returned in order and the complete input already fits in memory, a sort is often simpler and can be competitive in practice. A priority queue does not magically make full ordering linear: draining n entries still costs O(n log n). For streamed top-k selection, though, a bounded heap can avoid retaining all n inputs. [Chapter 79](./079-sorting.md) discusses sorting choices; [Chapter 74](./074-memory-complexity.md) explains why retained and peak memory matter.

These are algorithmic costs. The PHP Manual documents SPL operations and behavior, not end-to-end latency or exact bytes per entry. Payload size, comparison code, allocation overhead, and runtime version affect measurements.

## Using `SplPriorityQueue`

`SplPriorityQueue` is implemented as a max-heap. Larger priorities are extracted first. The value and priority are separate arguments to `insert()`:

```php
<?php
declare(strict_types=1);

$tasks = new SplPriorityQueue();
$tasks->setExtractFlags(SplPriorityQueue::EXTR_BOTH);

$tasks->insert(['id' => 'email-42', 'recipient' => 'ada@example.test'], 30);
$tasks->insert(['id' => 'payment-19', 'account' => 'acct-7'], 90);
$tasks->insert(['id' => 'report-8', 'format' => 'csv'], 10);

$next = $tasks->top(); // Inspect without removing.
printf("Next: %s at priority %d\n", $next['data']['id'], $next['priority']);

while (!$tasks->isEmpty()) {
    $entry = $tasks->extract(); // Return and remove the current top.
    printf("Handling %s\n", $entry['data']['id']);
}
```

With the default ordering, `payment-19` comes before `email-42`, which comes before `report-8`. The chosen values are examples only: a larger number means higher priority here because this class is a max-heap. `top()` peeks; `extract()` removes. Calling `extract()` on an empty queue is an error, so guard it with `isEmpty()` or `count()`.

Extraction flags control the value returned by `current()`, `top()`, and `extract()`:

- `SplPriorityQueue::EXTR_DATA` returns the inserted value. This is the default.
- `SplPriorityQueue::EXTR_PRIORITY` returns the priority.
- `SplPriorityQueue::EXTR_BOTH` returns an array with `data` and `priority` keys.

These flags change the returned representation, not the ordering policy or stored entries. Set the flag explicitly when code depends on a particular shape. `key()` is an iterator-node index; it is not the inserted data or priority and should not be treated as a stable job identifier.

The iterator interface needs special care. `SplPriorityQueue::next()` extracts the top node, so `foreach` consumes the queue as it advances. If the original must remain available, clone the queue and drain the clone:

```php
$ordered = clone $tasks;
$ordered->setExtractFlags(SplPriorityQueue::EXTR_BOTH);

while (!$ordered->isEmpty()) {
    $entry = $ordered->extract();
    // Inspect entries in priority order; $tasks remains populated.
}
```

On PHP 8.5, a runtime check confirmed that extracting from a clone decreases only the clone's count. PHP object cloning is shallow: the queue container is copied, while object payloads inside it should still be treated as shared object instances unless separately cloned. Do not mutate a payload while assuming the queue clone made an independent copy of that object.

## Deterministic Equal-Priority Order

If equal priority must preserve insertion order, store a sequence alongside the business priority and compare both fields. The following class accepts integer priorities and uses a monotonically increasing sequence to give earlier entries precedence on ties:

```php
<?php
declare(strict_types=1);

final class StablePriorityQueue extends SplPriorityQueue
{
    private int $sequence = 0;

    public function insert(mixed $value, mixed $priority): true
    {
        if (!is_int($priority)) {
            throw new InvalidArgumentException('Priority must be an integer.');
        }

        return parent::insert($value, [
            'level' => $priority,
            'sequence' => $this->sequence++,
        ]);
    }

    public function compare(mixed $priority1, mixed $priority2): int
    {
        $byLevel = $priority1['level'] <=> $priority2['level'];
        if ($byLevel !== 0) {
            return $byLevel;
        }

        // On a max-heap, an earlier (smaller) sequence must compare greater.
        return $priority2['sequence'] <=> $priority1['sequence'];
    }
}
```

The comparison method must be deterministic and transitive. Validate inputs before they reach the heap; the PHP Manual warns that throwing from `SplHeap::compare()` can corrupt the heap and leave it blocked. This subclass limits its own priorities to integers in `insert()`, while the internal composite value keeps comparisons predictable. If insertion order is not a requirement, the built-in class is simpler.

These subclass examples declare the literal `true` return type and therefore require PHP 8.2 or later. On earlier PHP versions, use the return type declared by that version's SPL method (for example, `bool` where applicable).

## Example: A Process-Local Scheduler

Suppose one command must process a batch of candidate jobs from highest urgency down. A priority queue exposes the next job through extraction; if more jobs arrive while work proceeds, it can also accept them without re-sorting the existing set:

```php
$pending = new SplPriorityQueue();
$pending->setExtractFlags(SplPriorityQueue::EXTR_BOTH);

foreach ($incomingJobs as $job) {
    $pending->insert($job, $job['urgency']);
}

while (!$pending->isEmpty()) {
    $entry = $pending->extract();
    handleJob($entry['data']);
}
```

This structure only schedules the work in this process. It does not reserve the job for one worker, keep a retry after a crash, or record completion. If `handleJob()` fails after extraction, this simple loop has lost its in-memory entry. Persist the job and its state in a broker or database when it must outlive the command. For durable work, make handlers safe to retry and design acknowledgement, leases, and idempotency at the system boundary; see [Chapter 244](../../volumes/16-distributed-systems/244-queues.md).

Urgency is a policy with consequences. A constant flow of urgent jobs can starve background work. A system may increase a task's effective priority as it waits, cap urgent work per interval, or allocate a minimum share to each class. If deadline order matters more than user-assigned urgency, compare due timestamps and break ties deterministically. Monitor oldest-item age as well as queue depth: a stable depth can still hide starvation of one class.

## Example: Retaining the Largest k Values

For a stream with n scores, sorting all values retains O(n) values even when only the best k are needed. Maintain a min-priority queue of at most k scores: its top is the smallest score currently in the candidate set, so a better new score can displace it.

```php
<?php
declare(strict_types=1);

final class MinIntPriorityQueue extends SplPriorityQueue
{
    public function insert(mixed $value, mixed $priority): true
    {
        if (!is_int($priority)) {
            throw new InvalidArgumentException('Priority must be an integer.');
        }

        return parent::insert($value, $priority);
    }

    public function compare(mixed $priority1, mixed $priority2): int
    {
        return $priority2 <=> $priority1;
    }
}

function largestScores(iterable $scores, int $k): array
{
    if ($k < 0) {
        throw new InvalidArgumentException('k must be zero or greater.');
    }

    $best = new MinIntPriorityQueue();
    $best->setExtractFlags(SplPriorityQueue::EXTR_DATA);

    foreach ($scores as $score) {
        if (!is_int($score)) {
            throw new InvalidArgumentException('Every score must be an integer.');
        }
        if ($k === 0) {
            continue;
        }

        $best->insert($score, $score);
        if ($best->count() > $k) {
            $best->extract(); // Discard the smallest retained score.
        }
    }

    $result = [];
    while (!$best->isEmpty()) {
        $result[] = $best->extract(); // Ascending among the retained values.
    }

    return array_reverse($result); // Return largest first.
}
```

The queue stores at most k scores, so it takes O(k) structure space and O(n log(k + 1)) time for k >= 1. When k is zero, it reads and validates each input but stores nothing, so that case takes O(n) time; a caller that does not need validation could return early instead. Equal scores can appear more than once because this example treats the input as a multiset. If the requirement is the top k distinct scores, define and test that duplicate policy explicitly. For record objects, compare a numeric score but store the whole record as the value.

## Complexity and Memory

For a heap-backed priority queue with n entries, the usual bounds are:

- Peek at the top: O(1).
- Insert one entry: O(log n).
- Extract the top: O(log n).
- Find an arbitrary value or update/remove by ID: O(n) without a separate index or handle.
- Store n entries: O(n) space, in addition to payload memory.

Inserting n entries one at a time costs O(n log n). Extracting them all also costs O(n log n). A separate sorted array can be more suitable when all entries must be traversed in order and no more values will arrive. `SplPriorityQueue` exposes insertion, top access, and extraction, but not a bulk-build or arbitrary-update API; do not assume it can reuse an existing array in linear time.

Space includes the priority, value, and heap bookkeeping for every live entry. A top-k algorithm bounds queue entries, not the size of each payload. Store compact identifiers instead of large hydrated objects when possible, and fetch details when selected. A scheduler with lazy-invalidated revisions may have more heap entries than live jobs until stale entries are discarded. Measure representative payloads and worker lifetimes rather than extrapolating bytes from the asymptotic bound.

## Operational Boundaries

An in-memory priority queue is lost when its request, command, or worker process exits. It is also private to that process; multiple PHP-FPM workers do not share one mutable SPL object. For a durable job scheduler, persist the authoritative state and use a transactional claim, lease, or broker operation so two consumers do not independently run the same item. A local heap can still be a useful cache of upcoming deadlines, but it must be rebuilt or reconciled against that source of truth after restart.

If priority derives from user-controlled input, validate its range and type. Unbounded priority values can bypass intended classes, and an unbounded number of entries can exhaust memory. Define maximum pending work, backpressure, cancellation, and starvation policy. Record queue depth, the age of the oldest task in each class, extraction rate, stale-entry count, and handler failures. These measurements describe system health; the data structure alone cannot.

## Testing

Test the ordering contract and lifecycle behavior explicitly:

1. Insert distinct integer priorities and assert that the default queue extracts highest first.
2. Set each extraction flag and assert the exact result shape for `top()`, `current()`, and `extract()`; include `EXTR_BOTH` keys.
3. Verify that `top()` leaves the count unchanged and `extract()` decreases it.
4. Insert equal-priority entries and assert only the documented behavior: their order is unspecified unless a tie-break policy is implemented.
5. For a stable subclass, enqueue several equal-priority entries and verify insertion order, then verify that higher priority still wins.
6. Clone a populated queue, drain one copy, and confirm that the other retains its entries. Include an object payload if callers might mutate payload objects.
7. Verify that iterator advancement consumes entries, and avoid relying on `foreach` as a read-only view.
8. Compare top-k results against a sorted reference for empty input, k equal to zero, k larger than the input, duplicate scores, and randomized short arrays.
9. For lazy invalidation, verify that stale versions and cancelled IDs are skipped and that the current version is handled once.

Avoid timing assertions in unit tests. Benchmarks should use realistic queue sizes, payloads, insertion/extraction ratios, and PHP versions. Comparator tests should include equality and ordering transitivity; a broken comparator can invalidate the structure rather than merely return an inconvenient order.

## Common Mistakes

- Assuming default `SplPriorityQueue` extracts the smallest priority; it is a max-heap.
- Assuming equal priorities preserve insertion order.
- Treating insertion of the same job ID as an update or replacement.
- Calling `foreach` or `next()` and expecting the queue to remain intact.
- Confusing the inserted value, its priority, and the iterator's integer node index.
- Throwing from a comparison method after insertion has begun.
- Using a priority queue when one linear scan or one full sort is simpler.
- Claiming FIFO fairness among different priorities, or ignoring starvation.
- Treating an in-process queue as durable, shared across workers, or safe after a crash.
- Bounding the number of entries while allowing each queued payload to grow without limit.

## Senior Engineer Thinking

Write down the ordering policy in domain terms first: which task wins, how ties behave, and whether waiting changes rank. Then identify the needed operations. If the system often searches by job ID, changes priorities, or cancels arbitrary tasks, a plain heap may be the wrong abstraction unless you also maintain an ID-to-entry index or use lazy invalidation with bounded cleanup.

Next separate scheduling from execution guarantees. A local priority queue chooses a next candidate. A transaction, broker, lease, idempotent handler, and retry policy determine whether work survives crashes and concurrent consumers. Finally, measure the actual workload. A full sort is often clearer for a fixed batch; a heap is useful when arrivals continue and the application repeatedly asks for only the next item or a bounded best subset.

## Exercises

1. Define numeric priorities for a notification sender with critical, normal, and bulk messages. Choose and justify a tie-break policy.
2. Implement an earliest-deadline-first min-priority queue and test equal deadlines. State which direction the comparator uses.
3. Adapt `largestScores()` to return the top k records by score and then by a stable record ID. Specify whether duplicate scores are retained.
4. Add version-based lazy invalidation to a scheduler. Insert two revisions of the same task and prove that only the newest revision is handled.
5. Compare a single scan, sorting the entire batch, and a heap bounded to k entries for n = 10,000,000 and k = 100. State the time and retained-space trade-offs without assuming constant factors.
6. Design a durable scheduled-job workflow around a database or broker. Identify which component selects due work, prevents two workers from claiming it at once, records retries, and handles a crash after a side effect.

## Review Questions

1. What is the difference between a FIFO queue and a priority queue?
2. Why does the default `SplPriorityQueue` return larger numeric priorities first?
3. What does the PHP Manual say about ordering entries with identical priorities?
4. Which extraction flag returns both the inserted value and its priority? What keys does the result contain?
5. Why can a clone be useful before draining a priority queue, and what remains shared by shallow object cloning?
6. How can a bounded min-heap retain the largest k values from a stream?
7. Why are arbitrary lookup and priority updates expensive without an auxiliary index?
8. Which operational guarantees are missing from a process-local SPL queue?
9. How can strict priority cause starvation, and what policies can reduce it?

## Summary

A priority queue exposes the next entry under an explicit ordering policy. It fits workloads that receive changing entries and repeatedly need the next best candidate, or that must retain only a bounded top-k subset. `SplPriorityQueue` is a max-heap: larger priorities come first, equal-priority order is undefined, and extraction flags determine whether callers receive the value, priority, or both. Its iterator consumes entries, so clone before draining when the original must remain available. A heap does not efficiently find or update arbitrary entries, guarantee fairness, or provide durable job delivery. Choose the policy, tie-breaker, update behavior, memory bound, and process boundary as part of the design.

## References

- [PHP Manual: `SplPriorityQueue`](https://www.php.net/manual/en/class.splpriorityqueue.php)
- [PHP Manual: `SplPriorityQueue::insert()`](https://www.php.net/manual/en/splpriorityqueue.insert.php)
- [PHP Manual: `SplPriorityQueue::compare()`](https://www.php.net/manual/en/splpriorityqueue.compare.php)
- [PHP Manual: `SplPriorityQueue::setExtractFlags()`](https://www.php.net/manual/en/splpriorityqueue.setextractflags.php)
- [PHP Manual: `SplPriorityQueue::top()`](https://www.php.net/manual/en/splpriorityqueue.top.php)
- [PHP Manual: `SplPriorityQueue::extract()`](https://www.php.net/manual/en/splpriorityqueue.extract.php)
- [PHP Manual: `SplPriorityQueue::next()`](https://www.php.net/manual/en/splpriorityqueue.next.php)
- [PHP Manual: `SplPriorityQueue::key()`](https://www.php.net/manual/en/splpriorityqueue.key.php)
- [PHP Manual: `SplHeap::compare()`](https://www.php.net/manual/en/splheap.compare.php)
- [PHP Manual: Cloning Objects](https://www.php.net/manual/en/language.oop5.cloning.php)
- [Chapter 74 — Memory Complexity](./074-memory-complexity.md)
- [Chapter 78 — Queues](./078-queues.md)
- [Chapter 79 — Sorting](./079-sorting.md)
- [Chapter 83 — Heaps](./083-heaps.md)
- [Chapter 91 — Choosing the Right Data Structure](./091-choosing-the-right-data-structure.md)
- [Chapter 244 — Queues](../../volumes/16-distributed-systems/244-queues.md)
