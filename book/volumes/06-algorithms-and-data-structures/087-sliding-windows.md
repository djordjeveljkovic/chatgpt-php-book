---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 87
title: Sliding Windows
slug: sliding-windows
status: complete
summary: ../../_ai/chapter-summaries/087-sliding-windows-summary.md
---

# Chapter 87 — Sliding Windows

## Why This Matters

A reporting service may need the sum of every 15 readings in a sensor stream. An API may need the maximum value seen in each 100-request slice. A rate limiter may count events from the last minute. These tasks all examine a region that moves through ordered data, but the meaning of “window” differs: it may contain a fixed number of records, a variable number selected by a condition, or every event whose timestamp falls within a duration.

A naive implementation rebuilds each window from scratch. For a sequence of `n` values and a fixed width `k`, that can take O(nk) time. A sliding-window algorithm carries forward just enough state from one position to the next, often reducing the work to O(n). The central design question is which facts can be updated when the left and right boundaries move.

A sliding window is an algorithm over an ordered sequence or event stream. It is different from the interval operations in [Chapter 86 — Intervals](086-intervals.md), which compare or combine arbitrary ranges. Before choosing a queue, deque, or accumulator, define how the window boundaries are measured and which events belong at an exact boundary.

## Mental Model

For a fixed number of items, the window is a contiguous slice that moves one position at a time:

```text
values:  [ 4,  8,  2,  7,  1,  9 ]
width 3: [ 4,  8,  2 ]
             [ 8,  2,  7 ]
                 [ 2,  7,  1 ]
                     [ 7,  1,  9 ]
```

The next window retains most of the previous one. It removes the outgoing item at the left edge and adds the incoming item at the right edge. With time-based data, the right edge may be the timestamp of the current event and multiple old events may expire at once.

Every example needs an ordering. Arrays have iteration order; event streams may be ordered by event timestamp or by arrival time. A window over the last 20 records is not the same as a window over the last 20 seconds when events arrive at an irregular rate.

## Core Concept

Sliding-window problems commonly fall into three groups:

- **Fixed-count window:** exactly `k` consecutive samples or records, such as the mean of every five measurements.
- **Variable-count window:** boundaries move to satisfy a condition, such as the longest contiguous range whose nonnegative values sum to at most a budget.
- **Time-based window:** keep events whose timestamps fall within a duration relative to a moving time boundary, such as recent requests in the last minute.

A fixed window over an indexed list has simple boundaries: for a window ending at index `right`, its start is `right - k + 1`. A time window must define whether the start and end are inclusive. [Chapter 86](086-intervals.md) establishes half-open intervals as one useful convention, but event counting sometimes deliberately includes the current event at both the cutoff and current timestamp. Either convention can work if expiration and tests implement the same rule.

The key invariant is:

> The maintained state describes exactly the values that belong to the current window, or enough information to derive the requested answer for that window.

For a sum, retain the running total and update it by subtracting and adding. For a maximum, keep a monotonic deque of candidate indices; a single scalar is not enough because the current maximum may leave the window. For an event count, retain timestamps until they expire. The algorithm and its state follow from the requested aggregate.

## Fixed-Count Windows: Running Sums

Recomputing each sum costs O(k) per output. For a fixed window of nonnegative integer measurements, retain a running sum and a FIFO of the current `k` values. The generator below yields a result only after a complete window exists. Its index counts consumed values from zero, regardless of an input array's keys.

```php
<?php

declare(strict_types=1);

/**
 * Yield the sum and ending zero-based position for each complete window.
 *
 * @param iterable<int> $values Nonnegative integer measurements.
 * @return Generator<int, array{endIndex: int, sum: int}>
 */
function fixedWindowSums(iterable $values, int $width): Generator
{
    if ($width < 1) {
        throw new InvalidArgumentException('Window width must be at least one.');
    }

    $window = new SplQueue();
    $windowSize = 0;
    $sum = 0;
    $index = 0;

    foreach ($values as $value) {
        if (!is_int($value) || $value < 0) {
            throw new InvalidArgumentException(
                'Values must be nonnegative integers.',
            );
        }

        if ($windowSize === $width) {
            $outgoing = $window->dequeue();
            $sum -= $outgoing;
            --$windowSize;
        }

        // Since values and the maintained sum are nonnegative, this check
        // prevents PHP integer overflow before adding the incoming value.
        if ($sum > PHP_INT_MAX - $value) {
            throw new OverflowException('Window sum exceeds the integer range.');
        }

        $sum += $value;
        $window->enqueue($value);
        ++$windowSize;

        if ($windowSize === $width) {
            yield ['endIndex' => $index, 'sum' => $sum];
        }

        ++$index;
    }
}
```

For `[4, 8, 2, 7, 1, 9]` and width `3`, the sums are `14`, `17`, `10`, and `17`. Each value enters once and leaves at most once, so processing `n` values takes O(n) time and the queue retains at most O(k) values. The generator emits lazily; if the caller appends every result to an array, the caller still uses O(n) output memory. If sums may be negative, or need exact decimal arithmetic, choose a representation and overflow policy that fits the domain rather than silently changing to floating-point behavior.

A sum alone does not answer every fixed-window query. An average is the sum divided by the fixed width. A moving count can use an integer accumulator. For a minimum or maximum, adding the incoming value and removing the outgoing value does not tell us the next extremum if the prior one leaves; retain candidates in a monotonic deque instead.

## Fixed-Count Windows: Sliding Maximum

A **monotonic deque** stores candidate indices in decreasing value order. When a new value arrives, any smaller or equal value at the back can never be the maximum of a future window: the newer value is at least as large and expires later. Remove those dominated candidates, add the new index, then expire indices that fall left of the current window. The front is the maximum candidate.

This implementation uses a fixed-capacity ring buffer for the deque, so its live state is O(k), even though indices move through an input of length `n`. It expects a list of integers; `array_is_list()` is available from PHP 8.1 onward.

```php
/**
 * Return the maximum value for every complete fixed-width window.
 *
 * @param list<int> $values
 * @return list<int>
 */
function slidingMaximum(array $values, int $width): array
{
    if ($width < 1) {
        throw new InvalidArgumentException('Window width must be at least one.');
    }
    if (!array_is_list($values)) {
        throw new InvalidArgumentException('Values must be a zero-indexed list.');
    }

    $count = count($values);
    if ($width > $count) {
        return [];
    }

    // The ring stores input indices, not copied values.
    $deque = array_fill(0, $width, 0);
    $head = 0;
    $size = 0;
    $maxima = [];

    foreach ($values as $index => $value) {
        if (!is_int($value)) {
            throw new InvalidArgumentException('Values must be integers.');
        }

        $windowStart = $index - $width + 1;

        // Expire indices that are no longer in the current window.
        while ($size > 0 && $deque[$head] < $windowStart) {
            $head = ($head + 1) % $width;
            --$size;
        }

        // Drop values dominated by the newer value. Keeping the newer index
        // also avoids retaining an equal value that would expire sooner.
        while ($size > 0) {
            $tail = ($head + $size - 1) % $width;
            if ($values[$deque[$tail]] > $value) {
                break;
            }
            --$size;
        }

        $tail = ($head + $size) % $width;
        $deque[$tail] = $index;
        ++$size;

        if ($index >= $width - 1) {
            $maxima[] = $values[$deque[$head]];
        }
    }

    return $maxima;
}
```

For `[4, 8, 2, 7, 1, 9]` with width `3`, the output is `[8, 8, 7, 9]`. Each index is inserted once and removed at most once from either end, so the total time is O(n), not O(nk). The ring buffer uses O(k) storage; the returned list uses O(n - k + 1) output space. If results should be streamed rather than retained, yield each maximum as it is found and let the caller decide whether to store it.

Equal values are handled by removing the older one (`<=` in value terms) because the newer equal value remains in future windows longer. If the application needs the identity or index of the earliest maximum, change that tie policy to preserve earlier equal candidates and test the resulting behavior.

## Variable-Count Windows and Two Pointers

Some problems ask for the longest contiguous range that satisfies a condition, rather than a range of exactly `k` items. When the condition is monotone as a boundary moves, two pointers can avoid checking every possible range. In the next example all values are nonnegative, and the condition is “sum is at most the limit.” Extending the right boundary can only increase the sum; moving the left boundary right can only decrease it.

```php
/**
 * Find the longest nonempty range of nonnegative values whose sum is <= limit.
 * Ties keep the earliest range. The returned end index is exclusive.
 *
 * @param list<int> $values
 * @return array{start: int, end: int, sum: int}|null
 */
function longestNonnegativeRangeAtMost(array $values, int $limit): ?array
{
    if ($limit < 0) {
        throw new InvalidArgumentException('Limit must be nonnegative.');
    }
    if (!array_is_list($values)) {
        throw new InvalidArgumentException('Values must be a zero-indexed list.');
    }

    $left = 0;
    $sum = 0;
    $best = null;

    foreach ($values as $right => $value) {
        if (!is_int($value) || $value < 0) {
            throw new InvalidArgumentException(
                'Values must be nonnegative integers.',
            );
        }

        if ($value > $limit) {
            // No valid range can include this item or cross it.
            $left = $right + 1;
            $sum = 0;
            continue;
        }

        while ($sum > $limit - $value) {
            $sum -= $values[$left];
            ++$left;
        }
        $sum += $value; // Invariant: 0 <= $sum <= $limit.

        $length = $right - $left + 1;
        if ($best === null || $length > $best['end'] - $best['start']) {
            $best = [
                'start' => $left,
                'end' => $right + 1,
                'sum' => $sum,
            ];
        }
    }

    return $best;
}
```

The range is represented as `[start, end)`, matching the list-index convention from Chapter 81. Each pointer only moves right, so the algorithm takes O(n) time and O(1) extra space. Returning `null` means no nonempty valid range exists; for an empty input or values each larger than the limit, that is the result.

The nonnegative-value precondition is essential. Negative numbers break the monotonic relationship: expanding a window might decrease its sum, so shrinking the left edge whenever the sum is too large can discard a range that later becomes valid. For general signed values, use a different algorithm based on prefix sums and an ordered data structure, or brute force for small input. A two-pointer template is correct only when its invariant matches the data domain.

## Time-Based Windows and Event Time

A **count window** holds a fixed number of records. A **time window** holds records whose timestamps fall within a duration. If events arrive every millisecond, a one-minute window may contain millions of values; if they arrive once a day, it may contain one. The width in seconds is not a memory bound.

There are two relevant notions of time:

- **Processing time** is when this PHP process receives or handles an event.
- **Event time** is the timestamp attached to when the event occurred in the source domain.

They can differ because of network delay, retries, offline clients, batching, or clock skew. A local process-time window can answer “how many did this worker handle recently?” It cannot answer “how many occurred in the last minute?” unless arrival and occurrence time are equivalent by contract.

The function below expects events sorted by nondecreasing event timestamp in integer milliseconds since the Unix epoch. It counts the current event in the closed trailing window `[t - width, t]`, keeping an event whose timestamp equals the cutoff. It uses `SplQueue` to remove expired events from the oldest end. `bottom()` peeks at the first queue node; the explicit `dequeue()` removes it.

```php
/**
 * Yield the count in each event's closed trailing time window [t - width, t].
 *
 * @param iterable<array{occurredAtMs: int, id: string}> $events
 * @return Generator<int, array{occurredAtMs: int, count: int}>
 */
function trailingEventCounts(iterable $events, int $widthMs): Generator
{
    if ($widthMs < 1) {
        throw new InvalidArgumentException('Window duration must be positive.');
    }

    $window = new SplQueue();
    $previousTimestamp = null;

    foreach ($events as $event) {
        if (!is_array($event)
            || !isset($event['occurredAtMs'], $event['id'])
            || !is_int($event['occurredAtMs'])
            || !is_string($event['id'])
            || $event['occurredAtMs'] < 0) {
            throw new InvalidArgumentException('Invalid event shape or timestamp.');
        }

        $timestamp = $event['occurredAtMs'];
        if ($previousTimestamp !== null && $timestamp < $previousTimestamp) {
            throw new InvalidArgumentException(
                'Events must be ordered by nondecreasing event time.',
            );
        }

        $cutoff = $timestamp - $widthMs;
        while (!$window->isEmpty()
            && $window->bottom()['occurredAtMs'] < $cutoff) {
            $window->dequeue();
        }

        $window->enqueue($event);
        $previousTimestamp = $timestamp;
        yield ['occurredAtMs' => $timestamp, 'count' => count($window)];
    }
}
```

Suppose width is 60,000 milliseconds and the current event is at timestamp 180,000. Events at 120,000 through 180,000 are included; events earlier than 120,000 have expired. Events with equal timestamps are accepted and counted in input order, so the emitted count for successive events at the same timestamp grows. If they represent one simultaneous batch, group them before emitting a result.

This algorithm assumes ordered input and handles no late event after its event-time position has passed. A streaming system that accepts out-of-order events needs an explicit policy: buffer and reorder within a bounded lateness allowance, drop late events, or revise prior results when late data arrives. Watermarks and corrections are stream-processing protocols, not properties provided by `SplQueue`. The code also stores event payloads to count them; if only counts matter and exact per-event output is unnecessary, time buckets or aggregate counters may use less memory.

The queue stores O(r) events, where `r` is the number of events whose timestamps remain in the duration. If the event rate is unbounded, a duration alone does not cap `r`; enforce an event-count limit or aggregate records. Processing is O(n) for `n` ordered events because every event is enqueued once and dequeued at most once.

## Edge Cases and Boundaries

The window policy should define these cases before code ships:

- **Width zero or negative:** reject it or define an explicit empty-window result. The examples reject it.
- **Input shorter than the width:** emit no complete fixed-count windows. Partial leading windows require a separate contract.
- **Empty input:** return no fixed-width output and no event-time output.
- **Non-list PHP arrays:** the indexed examples reject them. `array_is_list()` is available in PHP 8.1 and later; validate once at the boundary if supporting older PHP 8 releases.
- **Duplicate values or timestamps:** these are distinct samples/events unless the domain defines deduplication.
- **Exact time boundary:** the event-time example includes an event exactly at `t - width`. If the required model is half-open `[t - width, t)`, exclude the right-edge event and define whether the current event is added before or after the query.
- **Missing, malformed, or out-of-order timestamps:** validate at ingestion; do not silently assume arrival order is event-time order.
- **Integer range:** the sum example checks overflow for nonnegative integers. Timestamp units and subtraction must also stay in a supported integer range.
- **Result retention:** a lazy generator only saves output memory if the consumer processes results without accumulating them.

Window boundaries do not give a system concurrency guarantees. Two PHP workers each maintaining a local “requests in the last minute” queue do not share a global count. A distributed rate limiter needs shared atomic state and a clock/consistency policy, which is an application and storage design concern.

## Performance and Memory

Let `n` be the number of input items, `k` the fixed count width, and `r` the maximum number of records simultaneously retained in a time-based window.

| Operation | Time | Auxiliary state | Output |
| --- | --- | --- | --- |
| Recompute each fixed-width sum | O(nk) | O(1) | O(n) if retained |
| Running fixed-width sum | O(n) | O(k) queue | O(1) lazily or O(n) retained |
| Sliding maximum with monotonic deque | O(n) | O(k) ring buffer | O(n - k + 1) in the example |
| Variable nonnegative range with two pointers | O(n) | O(1) | One range |
| Ordered time-window count | O(n) | O(r) queued events | O(1) lazily or O(n) retained |

The abstract bounds count stored values, not PHP bytes. A `SplQueue` node has metadata beyond its payload; an array ring buffer uses PHP array entries; event payloads may dominate both. If each event includes a large decoded body, queue a compact timestamp and identifier or retain only the aggregate needed for the result.

When the source is a database, ask whether it can filter and aggregate by the requested time range more efficiently than loading all events into PHP. Indexes, grouping, retention, and query plans belong to the database's workload. When the data is already a bounded stream in a command or worker, a generator plus a small queue can make memory use predictable. For large result sets, avoid materializing both all inputs and all outputs unless that peak fits the worker's budget.

## Testing

Test the window definition and its invariant, not only a typical sample:

1. For fixed sums, test width `1`, width equal to the input length, width larger than the input, an empty iterable, duplicate values, and a generator source.
2. Verify that each sum equals a simple recomputation over that exact slice. Test the integer overflow guard and reject negative/non-integer values according to the example contract.
3. For sliding maximum, compare each result with `max()` over a brute-force slice on generated short lists. Include increasing, decreasing, all-equal, negative, and duplicate-maximum sequences so ring wrap and expiry are exercised.
4. For the variable range, compare with all subranges on small generated nonnegative lists. Include zeroes, an item equal to the limit, an item larger than the limit, and no valid item.
5. Add a signed-value counterexample to prove the two-pointer precondition matters; do not use the nonnegative implementation for it.
6. For event-time counts, test an event exactly on the cutoff, just before it, duplicate timestamps, an empty stream, an out-of-order event, and several events expiring on one step.
7. Test that consuming a generator lazily does not build a second result list, and separately measure peak memory for realistic payloads.
8. For a distributed rate limiter, test concurrent requests against the shared atomic operation; a unit test of one PHP queue cannot prove cross-process behavior.

A brute-force reference may be O(nk), but is simple enough to compare against optimized implementations for many small random inputs. Keep performance measurement separate from correctness tests.

## Common Mistakes

- Recomputing every aggregate from scratch when one outgoing and one incoming item suffice.
- Treating a fixed number of records as a fixed duration when event frequency varies.
- Expiring events with the wrong strictness at the cutoff boundary.
- Assuming a time window has bounded memory without bounding the event rate or retained payloads.
- Using BFS/queue or a rolling sum as if it also tracked the current minimum or maximum.
- Updating a maximum by subtraction when the old maximum leaves, without retaining other candidates.
- Applying the two-pointer sum pattern to negative values, where the condition is not monotone.
- Treating event arrival order as event-time order without validating it or defining late-event policy.
- Assuming a process-local window enforces a global limit across FPM or worker processes.
- Returning a generator but then collecting every yielded result and expecting bounded output memory.

## Senior Engineer Thinking

Translate “last window” into a precise rule: count or duration, sequence order or event time, complete or partial, and inclusive or exclusive boundaries. Then write the invariant that the maintained state must satisfy after each item. The invariant often reveals both the right data structure and the proof that each item enters and leaves only a bounded number of times.

Next separate the algorithm from the source and deployment model. A moving average over an in-memory sensor batch is a local computation. A rate limit shared by many servers needs an atomic shared operation. An event-time report that accepts late data needs buffering and correction semantics. A linear-time algorithm can still exceed memory, overflow its accumulator, or answer the wrong question if its window definition is vague.

## Exercises

1. Implement a fixed-width average using `fixedWindowSums()`. Define how an incomplete final window is handled and how floating-point precision affects the result.
2. Modify `slidingMaximum()` to return the index of the chosen maximum as well as its value. Specify whether the earliest or latest equal maximum wins.
3. Given nonnegative page-view counts, implement the shortest contiguous window whose total is at least a target. Explain why shrinking the left side is safe and what changes if counts can be negative.
4. Change `trailingEventCounts()` to use a half-open event-time window and write tests for events exactly at both boundaries.
5. Extend the event-time implementation to buffer slightly out-of-order events. Define the maximum allowed lateness, memory cap, output correction policy, and behavior for events later than the allowance.
6. Design a distributed “at most N requests per minute per API key” limiter. State the time-window boundary, atomic storage operation, clock assumption, memory/retention bound, and retry behavior.

## Review Questions

1. What distinguishes a fixed-count window from a time-based window?
2. How does carrying a running sum reduce fixed-window work from O(nk) to O(n)?
3. Why does a monotonic deque preserve enough information to report each sliding maximum?
4. Why must the maximum implementation preserve indices rather than only values?
5. Which property of nonnegative values makes the two-pointer range algorithm correct?
6. What is the difference between processing time and event time?
7. Why does a duration limit fail to impose a memory limit when event volume is unbounded?
8. What exactly does the event-time example do with an event at the cutoff and multiple events with the same timestamp?
9. Why does a generator not guarantee bounded total memory when its consumer retains all outputs?
10. Which guarantees must a distributed rate limiter add beyond an in-process queue?

## Summary

A sliding window moves over an ordered sequence or event stream while maintaining only the state needed for the current range. Running sums update in constant work per item; monotonic deques maintain extrema; two pointers find variable ranges when their condition is monotone; and a queue expires old events from an ordered time stream. These approaches can reduce repeated work to O(n), but their correctness depends on window width, value assumptions, ordering, and boundary policy.

Count windows, processing-time windows, and event-time windows answer different questions. A generator can stream results, but retained outputs, PHP array/node overhead, event density, and payload size still determine peak memory. Define cutoffs, late-event handling, overflow policy, and concurrency ownership explicitly. A process-local data structure does not enforce a limit across PHP workers or servers.

## References

- [PHP Manual: `array_is_list()`](https://www.php.net/manual/en/function.array-is-list.php)
- [PHP Manual: Generators](https://www.php.net/manual/en/language.generators.php)
- [PHP Manual: `SplQueue`](https://www.php.net/manual/en/class.splqueue.php)
- [PHP Manual: `SplQueue::enqueue()`](https://www.php.net/manual/en/splqueue.enqueue.php)
- [PHP Manual: `SplQueue::dequeue()`](https://www.php.net/manual/en/splqueue.dequeue.php)
- [PHP Manual: `SplDoublyLinkedList::bottom()`](https://www.php.net/manual/en/spldoublylinkedlist.bottom.php)
- [Chapter 74 — Memory Complexity](074-memory-complexity.md)
- [Chapter 78 — Queues](078-queues.md)
- [Chapter 81 — Binary Search](081-binary-search.md)
- [Chapter 86 — Intervals](086-intervals.md)
