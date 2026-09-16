---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 86
title: Intervals
slug: intervals
status: complete
summary: ../../_ai/chapter-summaries/086-intervals-summary.md
---

# Chapter 86 — Intervals

## Why This Matters

Reservations, maintenance windows, subscription periods, rate-limit windows, and log-retention ranges all describe spans between two points. Bugs often come from treating the endpoints as an implementation detail: does a reservation ending at 10:00 conflict with one starting at 10:00? Is an event exactly at the end included? Does a zero-length period occupy anything?

Once the interval convention is explicit, many problems reduce to simple comparisons. Sorting intervals makes it possible to merge overlapping coverage or detect conflicts with a scan. A sweep-line algorithm handles many simultaneous ranges, and a heap can track which active range ends first. The important work is choosing the model before choosing the algorithm.

## Mental Model

An interval is a contiguous range on an ordered domain, described by two boundaries and a rule about whether each boundary belongs to the range:

```text
ordered domain + start + end + endpoint convention → covered points
```

The domain might be integer positions, timestamps, prices, or another totally ordered value. The algorithm needs to compare values in that domain consistently. A timestamp interval also needs a time-zone and precision policy; the interval algorithm cannot repair ambiguous input.

For application time ranges, this chapter uses **half-open intervals** written **[start, end)**: the start is included and the end is excluded. This convention makes adjacent ranges fit without overlap and makes duration `end - start` when the domain supports subtraction.

## Core Concept

For half-open intervals `A = [aStart, aEnd)` and `B = [bStart, bEnd)`, assuming both have `start < end`, they overlap exactly when:

```text
aStart < bEnd AND bStart < aEnd
```

Each interval must begin before the other one ends. If `A` ends exactly where `B` begins, one strict comparison is false, so they do not overlap:

```text
A = [09:00, 10:00)
B = [10:00, 11:00)
```

The point 10:00 belongs to `B`, not `A`. This avoids assigning a shared boundary to both adjacent periods. Containment is also direct: `outer` contains `inner` when `outer.start <= inner.start` and `inner.end <= outer.end`. With non-empty half-open intervals, equality at either side is allowed.

Closed intervals such as `[start, end]` include both endpoints. Two closed intervals that share an endpoint overlap at that point. Open intervals exclude endpoints. These conventions are all valid, but their overlap and adjacency tests differ. Never copy a comparison formula without confirming its endpoint policy.

An interval's notation is also distinct from the half-open array-index range used in [Chapter 81 — Binary Search](081-binary-search.md). The notation is familiar, but here it describes a domain's covered values, not positions being searched.

## How It Works

### Validate the interval model first

For a reservation, define whether `start === end` is valid. Mathematically, `[t, t)` is empty and has no overlap with any interval. A booking system may reject it as a meaningless reservation. A measurement API may accept a zero-duration event, but should model it as a point event rather than covered time in `[t, t)`. Reversed endpoints (`start > end`) usually indicate invalid input; silently swapping them can hide a caller bug.

The following sections assume **non-empty** intervals with comparable endpoints. The example uses integer Unix timestamps so the comparison policy is clear. If the application accepts `DateTimeImmutable` values, normalize the source time zone and precision at the boundary and compare instants; do not compare unvalidated local wall-clock strings. Daylight-saving transitions can make a local time ambiguous or nonexistent, and a calendar day is not always a fixed number of seconds.

Use a named structure or a documented shape to keep endpoint meaning visible. In an array, `start` and `end` are easier to review than unexplained numeric offsets. If the interval is a persistent domain concept with invariants, a value object can validate `start < end` once at construction.

### Check overlap and containment

Here are pure helpers for integer boundaries:

```php
<?php

declare(strict_types=1);

function assertNonEmptyIntegerInterval(mixed $interval): void
{
    if (!is_array($interval)
        || !isset($interval['start'], $interval['end'])
        || !is_int($interval['start'])
        || !is_int($interval['end'])
        || $interval['start'] >= $interval['end']) {
        throw new InvalidArgumentException(
            'An interval must have integer start < end.',
        );
    }
}

/** @param array{start: int, end: int} $left
 *  @param array{start: int, end: int} $right
 */
function overlapsHalfOpen(array $left, array $right): bool
{
    assertNonEmptyIntegerInterval($left);
    assertNonEmptyIntegerInterval($right);

    return $left['start'] < $right['end']
        && $right['start'] < $left['end'];
}

/** @param array{start: int, end: int} $outer
 *  @param array{start: int, end: int} $inner
 */
function containsHalfOpen(array $outer, array $inner): bool
{
    assertNonEmptyIntegerInterval($outer);
    assertNonEmptyIntegerInterval($inner);

    return $outer['start'] <= $inner['start']
        && $inner['end'] <= $outer['end'];
}
```

The helpers reject empty and reversed intervals rather than guessing whether they should mean “no coverage” or malformed input. A different API can choose a different policy, but should do so consistently. If intervals are built through a validating value object, the helpers can rely on that invariant instead of repeating validation.

### Sort and merge coverage

Suppose a calendar has many blocked ranges and needs a compact representation of their union. Sort by ascending start, then ascending end. Scan from left to right while keeping the current merged range. The next range may start after the current range, overlap and extend it, or be fully contained within it.

For set-union output, merging touching intervals is often useful: `[1, 4)` and `[4, 7)` cover the same set as `[1, 7)`. The code below merges both overlap and adjacency by treating `next.start <= current.end` as mergeable:

```php
<?php

declare(strict_types=1);

/**
 * Merge the union of non-empty half-open integer intervals.
 *
 * Adjacent ranges are coalesced because they represent continuous coverage.
 *
 * @param list<array{start: int, end: int}> $intervals
 * @return list<array{start: int, end: int}>
 */
function mergeHalfOpenIntervals(array $intervals): array
{
    foreach ($intervals as $interval) {
        assertNonEmptyIntegerInterval($interval);
    }

    usort(
        $intervals,
        static fn (array $left, array $right): int =>
            ($left['start'] <=> $right['start'])
            ?: ($left['end'] <=> $right['end']),
    );

    $merged = [];

    foreach ($intervals as $next) {
        $lastIndex = count($merged) - 1;

        if ($lastIndex < 0 || $next['start'] > $merged[$lastIndex]['end']) {
            $merged[] = $next;
            continue;
        }

        if ($next['end'] > $merged[$lastIndex]['end']) {
            $merged[$lastIndex]['end'] = $next['end'];
        }
    }

    return $merged;
}

$busy = mergeHalfOpenIntervals([
    ['start' => 9, 'end' => 12],
    ['start' => 11, 'end' => 14],
    ['start' => 16, 'end' => 17],
    ['start' => 14, 'end' => 15],
]);

// [9, 15) and [16, 17)
```

The function validates before sorting, so its comparator only sees the expected integer shape. Sorting gives `O(n log n)` comparison work; the single merge pass is `O(n)`. The output uses `O(n)` worst-case space, and sorting can require implementation-dependent temporary memory. PHP's `usort()` reindexes the array, which is appropriate because the intervals are a list. See [Chapter 79 — Sorting](079-sorting.md) for comparator and key behavior.

If touching intervals must remain separate in the output, the merge condition changes: begin a new range when `next.start >= current.end`, rather than only when it is greater. That representation choice does not change the half-open overlap rule. Two adjacent reservations do not conflict even if a report later coalesces their continuous coverage.

### Detect conflicts in sorted data

For one candidate interval and a small unsorted list, checking every existing interval is often the clearest option: `O(n)` comparisons and `O(1)` extra space. For a large static set, sort once and keep the non-overlapping intervals ordered. To check a candidate, binary-search the insertion boundary by start time and inspect the neighboring interval(s). If the stored intervals are guaranteed non-overlapping and sorted, only the predecessor and successor can conflict.

The sorted representation has a maintenance cost. Searching the position is `O(log n)`, but inserting into a PHP array list at that position shifts later values and is `O(n)`. Re-sorting after each insertion costs more. For a read-heavy snapshot, the trade-off may be useful; for frequently updated availability, an interval tree or database range index may fit better. A binary search is only as correct as the sorted-input and comparator invariants from Chapter 81.

For one-time conflict detection among many intervals, sorting by start and checking each interval against the furthest-reaching prior end finds whether any overlap exists in `O(n log n)` total time. Comparing only adjacent intervals is insufficient when intervals may nest. For example, `[1, 100)`, `[2, 3)`, `[4, 5)` has a conflict even though the last two do not overlap each other. Track the maximum end seen so far.

### Sweep-line processing

A sweep-line algorithm turns interval endpoints into ordered events. Sort all starts and ends, then move from left to right while tracking active intervals or a count. Applications include peak concurrent usage, finding time ranges with any coverage, and reporting which intervals are active at a point.

Tie ordering at the same coordinate encodes endpoint semantics. For half-open intervals, an interval ending at `t` is no longer active at `t`, while one starting at `t` is active. Therefore, if maintaining a running active count, process end events before start events at the same timestamp. That rule is why adjacent intervals do not briefly appear to overlap. For closed intervals, the tie policy differs. Store a unique interval ID with events if the algorithm must report pairs rather than only a count.

Sorting `2n` endpoint events costs `O(n log n)` and the scan is `O(n)`. Reporting all overlapping pairs can take `O(n²)` when every interval overlaps every other interval; no algorithm can write a quadratic-size answer in subquadratic output time. This is an output-size bound, not a failure of the sweep technique.

### When a heap helps

If intervals arrive ordered by start and the task is to reuse the resource that frees earliest, a min-heap of end points can track active assignments. Before handling a new `[start, end)` interval, repeatedly free entries whose end is `<= start`; half-open adjacency means they are available at that exact start. Then assign the interval to an available resource or add a resource if none is free. If only the minimum number of resources is needed, end points alone are enough; to return room IDs, store `(end, roomId)` in the heap and put freed IDs in a separate available pool. A heap makes the next ending interval available in `O(1)` peek and removes it in `O(log n)`; each insertion also costs `O(log n)`. Chapter 83 explains heaps and Chapter 84 discusses priority queues. This is the familiar room-allocation or machine-scheduling problem.

A heap answers “which active interval ends first?” It does not answer arbitrary interval-overlap queries by itself, because its remaining values are not fully ordered. Use the representation that matches the needed operation.

## What PHP Does

The sample functions use PHP arrays as lists of small associative arrays. PHP arrays are ordered maps rather than compact numeric vectors, so each interval has more representation overhead than two packed integers in a native buffer. That is usually a reasonable choice for modest application-level collections, but memory can dominate when millions of PHP array entries are retained; see [Chapter 74 — Memory Complexity](074-memory-complexity.md).

`usort()` sorts values in place and assigns consecutive integer keys. Its comparator should return a negative, zero, or positive integer and satisfy a consistent order. The example compares typed integers with `<=>` instead of subtracting them. PHP 8.0 and later preserve the prior order of values that compare equal, but the comparator here also compares end points, so equal start/end intervals are equivalent regardless of their input order. The sorting API does not promise a particular sorting algorithm or portable complexity guarantee; `O(n log n)` here describes the usual comparison-sorting model. See the official [PHP Manual: `usort()`](https://www.php.net/manual/en/function.usort.php).

For timestamps, PHP's `DateTimeImmutable` offers explicit date-time values, but parsing, time zones, and daylight-saving policy are part of input normalization. Converting to integer epoch seconds is appropriate only when second precision is sufficient and the domain's calendar semantics have already been resolved. Do not convert local business dates to seconds by assuming every day is exactly 86,400 seconds.

## Practical Example: Coalescing Maintenance Windows

A deployment tool may receive maintenance windows from several sources and need a concise list of periods during which service is unavailable. The merge function above treats each window as half-open and coalesces touching ranges. Its output can drive a report or a later availability check.

For a candidate range `[candidateStart, candidateEnd)`, compare it against merged coverage. If the sorted ranges are non-overlapping, a linear scan may be simplest for a small number of windows. For repeated checks on a static list, binary search can find the first range whose end is greater than `candidateStart`; the candidate conflicts if that range starts before `candidateEnd`. State whether touching endpoints count as a conflict—the strict comparisons in this test encode half-open semantics.

If windows are stored in a database, do not fetch every row and merge it in PHP by default. Ask the database to filter by the relevant time range, and inspect its index and plan. Some databases have native range types or exclusion constraints; their syntax and guarantees are database-specific. Application-side checks alone are vulnerable to two concurrent requests both observing a free slot and then inserting overlapping reservations. Enforce the invariant transactionally or with a database constraint/lock appropriate to the database. The reservation case study in [Chapter 280](../../volumes/19-small-engineering-projects/280-tennis-reservation-service.md) develops this boundary further.

## Edge Cases

- **Empty input:** merging an empty list returns an empty list.
- **A singleton:** it is already merged, provided it is valid and non-empty.
- **Reversed or empty range:** reject, ignore, or represent explicitly according to the domain; do not let inconsistent policies drift between functions.
- **Equal endpoints between ranges:** adjacent half-open intervals do not conflict. A union operation may still coalesce them.
- **Duplicate ranges:** they collapse to one range in a union; a reservation list may need to preserve separate records instead.
- **Nested ranges:** the furthest end matters. Comparing only neighboring sorted starts can miss overlaps.
- **Equal start times:** sort by end as a deterministic secondary key, then apply the same merge rule.
- **Precision:** rounding milliseconds to seconds can create or remove a gap. Pick one precision and normalize before comparison.
- **Overflow:** subtracting very large integer boundaries can overflow or produce a float. Overlap and containment checks need only comparisons; calculate durations only after validating that the domain and range fit the chosen numeric representation.
- **Time zones and calendar rules:** compare instants for elapsed-time overlap, or use calendar operations for rules such as “the whole local business day.” Those are different requirements.

## Performance

Let `n` be the number of intervals and `m` the number of reported overlaps or output pairs.

| Operation | Time | Additional / retained space |
| --- | --- | --- |
| Test one pair for overlap | O(1) | O(1) |
| Scan for a conflict against n intervals | O(n) | O(1) |
| Sort then merge n intervals | O(n log n) | O(n) output; sort workspace is implementation-dependent |
| Sweep endpoint events | O(n log n) sort + O(n) scan | O(n) events; active-set space depends on the problem |
| Report every overlapping pair | O(n log n + m) for a suitable sweep | O(n + m), including the result |
| Allocate resources with an end-time heap | O(n log n) | O(n) worst-case heap |

The input and output both count toward peak live memory. In PHP, copying a list, decorating it with events, and returning a second merged list can keep several representations alive at once. For large data, filter in SQL, stream already ordered rows where possible, or process bounded chunks with a clearly defined boundary policy. Chunking arbitrary intervals independently is not correct unless intervals crossing chunk boundaries are reconciled.

Asymptotic bounds do not decide whether the work belongs in PHP. If a database owns persistent reservations, database range queries, constraints, and transactions may be the right tools. If a worker has already loaded a bounded schedule snapshot and needs a report, a PHP scan or merge may be clearer. Measure realistic data sizes and include transfer and allocation costs.

## Concurrency and Database Interaction

An interval overlap query can identify a conflict at one instant, but a read-then-write sequence is not an exclusive reservation:

```text
request A: sees no overlap
request B: sees no overlap
request A: inserts [10, 11)
request B: inserts [10, 11)
```

Each PHP request may execute sequentially within its own process while still racing with another PHP-FPM worker. Protect the invariant where concurrent writes meet: use a suitable database range/exclusion constraint, lock a stable resource row and check within a transaction, or use another database-specific atomic mechanism. A unique constraint on `(resource_id, start_time)` alone does not prevent intervals with different start times from overlapping.

The exact SQL/index solution depends on the database engine and isolation level. Treat a PHP-side availability check as advisory unless the later write is protected. If a request can be retried after an uncertain response, combine the reservation operation with an idempotency policy so a retry does not create a second logical booking. See later chapters on transactions, isolation, locking, and concurrency.

## Testing

Test the interval contract before testing performance:

1. Assert overlapping ranges in both argument orders.
2. Assert adjacency `[1, 4)` and `[4, 8)` is not a conflict.
3. Assert containment, equal ranges, and partial overlap.
4. Test empty input, a singleton, duplicates, nested ranges, equal starts, and a chain of touching intervals.
5. Verify that merge output is sorted, every output interval is non-empty, no two output intervals can be merged under the chosen adjacency policy, and its covered points equal the input union.
6. Assert the chosen policy for zero-length and reversed intervals (the example throws for both).
7. Test large timestamps and the actual precision/time-zone normalization at the input boundary.
8. For a database-backed booking flow, run concurrent requests and prove the database permits at most one conflicting reservation.

For bounded integer domains, property-based tests can compare the merge result with a simple reference implementation that enumerates covered integer points. The reference algorithm may be too slow for production input, but is easy to inspect and useful for generated small cases. For time ranges, test boundaries explicitly instead of relying only on random timestamps.

## Common Mistakes

- Using `<=` in the half-open overlap formula and treating adjacent ranges as conflicts.
- Merging touching ranges without explaining that union coalescing is a representation choice.
- Sorting only by start and then comparing each interval only with its immediate predecessor, which misses nested ranges.
- Sorting one representation and searching it with a different comparison rule.
- Treating strings such as `"2026-03-29 02:30"` as unambiguous instants without a zone and DST policy.
- Assuming a binary search makes insertion into a PHP list logarithmic.
- Loading all persistent reservations into PHP when the database can perform the range filter.
- Assuming a pre-insert availability check prevents simultaneous conflicting writes.
- Calculating durations by subtracting unvalidated or differently normalized endpoints.

## Senior Engineer Thinking

Start with the domain invariant. Does a boundary point belong to the interval? Are zero-length spans valid? Is the value an elapsed-time instant, a local calendar period, or an integer slot? These decisions determine every comparison, index, and test that follows.

Then identify the operation the system actually needs. Pairwise conflict, union/merge, repeated point queries, all overlapping pairs, peak concurrency, and resource allocation are related but distinct problems. A scan, sorted list, sweep-line event set, heap, interval tree, or database range index serves different workloads. State whether the collection is static or changes often, how many intervals fit in one process, and who owns the persistent invariant.

Finally, account for concurrent writers. An algorithm can be mathematically correct and still fail as a reservation system when two processes race. The database transaction or constraint is part of the design, as are idempotent retries and observability for rejected conflicts.

## Exercises

1. Write overlap and containment helpers for closed integer intervals `[start, end]`. Show how adjacency and point intervals differ from the half-open policy.
2. Change `mergeHalfOpenIntervals()` so touching ranges remain separate while overlapping ranges still merge. Add a test showing why this is different from overlap detection.
3. Given intervals sorted by start, write a scan that reports whether any pair overlaps. Include `[1, 100)`, `[2, 3)`, `[4, 5)` to catch the adjacent-comparison mistake.
4. Design a sweep-line algorithm that returns the maximum number of active half-open intervals. Specify event ordering for equal timestamps and test adjacent intervals.
5. Adapt the heap-based room-allocation algorithm to return the resource number assigned to each interval. Define deterministic behavior for equal start times.
6. Design a database-backed reservation operation for one court. Identify the transaction boundary, the race, the database guarantee that closes it, and retry behavior after a lost response.

## Review Questions

1. Under the half-open convention, what exact comparisons determine whether two non-empty intervals overlap?
2. Why do `[a, b)` and `[b, c)` not overlap, and why might a union algorithm still merge them?
3. Why can checking only adjacent start-sorted intervals miss an overlap?
4. Which event should be processed first at a shared timestamp in a half-open sweep-line count, and why?
5. What invariant allows a heap to help with room allocation, and what does that heap not support efficiently?
6. Why is finding an insertion point with binary search not enough to make PHP list insertion O(log n)?
7. Why is an application-side availability check insufficient when concurrent PHP requests can create reservations?
8. Which time-zone and precision policies must be decided before timestamps can be safely compared?

## Summary

Intervals become reliable once the domain, endpoint convention, and invalid-range policy are explicit. For non-empty half-open ranges `[start, end)`, overlap is `a.start < b.end && b.start < a.end`; containment uses inclusive boundary comparisons. Sorting by start followed by a scan merges coverage in O(n log n), while a sweep line or an end-time heap fits multi-interval and resource-allocation problems. Adjacency is not overlap, even when a union representation chooses to coalesce touching ranges. Choose data structures according to the operation and update pattern, account for PHP memory and database ownership, and enforce reservation invariants transactionally under concurrent requests.

## References

- [PHP Manual: `usort()`](https://www.php.net/manual/en/function.usort.php)
- [PHP Manual: `DateTimeImmutable`](https://www.php.net/manual/en/class.datetimeimmutable.php)
- [Chapter 74 — Memory Complexity](074-memory-complexity.md)
- [Chapter 79 — Sorting](079-sorting.md)
- [Chapter 81 — Binary Search](081-binary-search.md)
- [Chapter 83 — Heaps](083-heaps.md)
- [Chapter 84 — Priority Queues](084-priority-queues.md)
- [Chapter 85 — Graphs](085-graphs.md)
- [Chapter 87 — Sliding Windows](087-sliding-windows.md)
- [Chapter 108 — Indexes](../../volumes/08-databases/108-indexes.md)
- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)
- [Chapter 280 — Tennis Reservation Service](../../volumes/19-small-engineering-projects/280-tennis-reservation-service.md)
