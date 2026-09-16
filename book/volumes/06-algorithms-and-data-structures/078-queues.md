---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 78
title: Queues
slug: queues
status: complete
summary: ../../_ai/chapter-summaries/078-queues-summary.md
---

# Chapter 78 — Queues

## Why This Matters

A queue answers a simple question: when several items are waiting, which one should be handled next? The usual answer is the one that arrived first. That rule appears in breadth-first search, request buffering, event loops, and background work. It is also the foundation for thinking about durable job queues, although an in-memory queue and a message broker solve different problems.

In PHP, the first implementation many developers reach for is an array plus `array_shift()`. It is easy to understand and fine for a handful of elements. Repeatedly removing the first element from a large PHP array has a cost that a head-indexed array or `SplQueue` can avoid. Choosing the right representation matters when queues grow or are drained repeatedly.

## Mental Model

```text
dequeue ← [ first ][ next ][ next ][ last ] ← enqueue
             ↑
          dequeue
```

For a first-in, first-out (FIFO) queue, insertion happens at the rear and removal happens at the front. If `A`, then `B`, then `C` are enqueued, they are dequeued in that same order. A queue may also expose `peek` (inspect the next item without removing it), `isEmpty`, and `count`.

FIFO describes the order of operations on one queue. It does not automatically mean that several concurrent producers have a globally fair order, or that multiple consumers finish jobs in insertion order. Those are separate guarantees.

## Core Concept

A queue is useful when work should be processed in arrival order, or when an algorithm explores items in layers. Unlike a stack, which processes the newest item first (Chapter 77), a queue leaves older waiting work at the front.

The core invariant is:

> The next item removed is the earliest item that was inserted and has not already been removed.

The invariant says nothing about persistence, concurrency, retries, capacity, or priority. A queue data structure only organizes items in one process. A durable distributed queue needs storage and protocols for producer/consumer coordination; see Chapter 244.

## How It Works

### A queue backed by `array_shift()`

Appending to a list and taking its first value is easy to read:

```php
$queue = [];
$queue[] = 'first';
$queue[] = 'second';

if ($queue !== []) {
    $next = array_shift($queue); // 'first'
}
```

The empty check matters when `null` is a valid queued value. `array_shift()` also returns `null` for an empty array, so checking only the return value cannot distinguish an empty queue from a queue whose next value is `null`. The PHP Manual specifies that `array_shift()` removes the first value and reindexes numeric keys from zero; it does not promise a time complexity. In practice, treat repeated front removal as work that can scale with the remaining array, and avoid draining a large queue this way. Draining `n` items can then approach O(n²) total work.

This approach is still a reasonable choice for small, short-lived lists where clarity matters more than throughput. As Chapter 75 explains, a PHP array is an ordered map rather than a compact vector, so the native representation also has meaningful per-entry memory overhead.

### A head-index queue

Instead of removing the front element, keep it in place and advance a head index. Appending writes at the tail; dequeue reads and unsets the current head. Periodically compact the remaining entries so that consumed positions do not make the array grow without bound.

```php
<?php
declare(strict_types=1);

final class ArrayQueue implements Countable
{
    /** @var array<int, mixed> */
    private array $items = [];

    private int $head = 0;
    private int $tail = 0;

    public function enqueue(mixed $value): void
    {
        $this->items[$this->tail] = $value;
        ++$this->tail;
    }

    public function dequeue(): mixed
    {
        if ($this->isEmpty()) {
            throw new UnderflowException('Cannot dequeue from an empty queue.');
        }

        $value = $this->items[$this->head];
        unset($this->items[$this->head]);
        ++$this->head;

        if ($this->isEmpty()) {
            // Reset indices so an emptied queue can be reused indefinitely.
            $this->items = [];
            $this->head = 0;
            $this->tail = 0;
        } elseif ($this->head >= 1024 && $this->head * 2 >= $this->tail) {
            // Copy only live entries; occasional compaction prevents stale
            // indices and hash-table holes from accumulating forever.
            $this->items = array_values($this->items);
            $this->tail = count($this->items);
            $this->head = 0;
        }

        return $value;
    }

    public function peek(): mixed
    {
        if ($this->isEmpty()) {
            throw new UnderflowException('Cannot peek at an empty queue.');
        }

        return $this->items[$this->head];
    }

    public function isEmpty(): bool
    {
        return $this->head === $this->tail;
    }

    public function count(): int
    {
        return $this->tail - $this->head;
    }
}
```

The implementation accepts `null`, because emptiness is represented by the indices rather than by a special return value. It removes references to dequeued values with `unset()`. When the queue is drained, it resets its indices. If many items have been consumed but some remain, compaction copies live items into a fresh packed array. The specific threshold is a policy choice, not a PHP rule.

With ordinary PHP array append and indexed access, enqueue and dequeue are expected O(1) operations between compactions. A compaction is O(k) for `k` live values, but the threshold ensures it only happens after substantial progress; queue operations are amortized O(1). The live storage is O(n), with a temporary additional copy during compaction. This is a compact implementation for moderate in-process queues, but its exact memory and runtime costs should be measured for the workload.

### `SplQueue`

PHP’s Standard PHP Library provides `SplQueue`, a queue implemented with a doubly linked list. Its public queue operations express the intended behavior directly:

```php
$queue = new SplQueue();
$queue->enqueue('first');
$queue->enqueue('second');

while (!$queue->isEmpty()) {
    $next = $queue->dequeue();
    // Handle $next.
}
```

The PHP Manual documents `enqueue()` as adding at the end and `dequeue()` as removing from the front. The SPL data-structure documentation describes endpoint additions/removals on the doubly linked list as O(1). It provides O(n) storage with per-node overhead, which can exceed a packed or head-indexed array for the same number of values. `SplQueue` inherits other list operations such as `push()` and `pop()`; use `enqueue()` and `dequeue()` in queue code so that the FIFO intent stays clear.

### Choosing among the three

| Representation | Enqueue | Dequeue | Useful when | Main trade-off |
| --- | --- | --- | --- | --- |
| Array with `array_shift()` | Append is typically amortized O(1) | Front shift may be O(n) | Small, simple, bounded work | Repeated shifts can be expensive |
| Head-index array | Expected amortized O(1) | Expected amortized O(1) | Moderate in-process workload or custom API | More code; compaction and memory need care |
| `SplQueue` | O(1) endpoint operation per SPL docs | O(1) endpoint operation per SPL docs | Want an explicit standard FIFO structure | Linked-node overhead and inherited list API |

Complexity describes growth, not every elapsed time. An array can have better locality and lower per-item overhead than linked nodes for some workloads, while `SplQueue` avoids copying the remaining sequence on dequeue. Measure representative queue sizes before optimizing. For a large queue that must outlive a PHP process, none of these in-memory structures is the right persistence mechanism.

## What PHP Does

PHP arrays are ordered hash tables, discussed in Chapters 49–50 and summarized in Chapter 75. A head-index array is still a PHP array; the index prevents the program from asking PHP to remove and renumber the front entry on every dequeue. `unset()` removes the consumed value from the live map, and occasional `array_values()` creates a new dense array from the remaining entries.

The PHP Manual documents the visible behavior of `array_shift()`, including numeric key reindexing and the `null` result on empty input. It does not specify a complexity guarantee. Complexity tables in this chapter describe the usual data-structure cost model and common PHP implementation behavior, not a language-level promise for every runtime version.

## Practical Example: Breadth-First Search

Breadth-first search explores a graph in layers: first the start node, then its neighbors, then nodes two edges away. A FIFO queue is the structure that preserves that order. Mark nodes as visited when they are enqueued; otherwise, two parents can add the same node before it is processed.

```php
/**
 * @param array<string, list<string>> $graph Adjacency list of canonical IDs.
 * @return list<string>
 */
function breadthFirstOrder(array $graph, string $start): array
{
    $queue = new ArrayQueue();
    $queue->enqueue($start);
    $visited = [$start => true];
    $order = [];

    while (!$queue->isEmpty()) {
        $node = $queue->dequeue();
        $order[] = $node;

        foreach ($graph[$node] ?? [] as $neighbor) {
            if (isset($visited[$neighbor])) {
                continue;
            }

            $visited[$neighbor] = true;
            $queue->enqueue($neighbor);
        }
    }

    return $order;
}
```

For a graph with `V` visited vertices and `E` inspected edges, this traversal takes O(V + E) expected time and O(V) additional space for the queue and visited set. Each vertex is enqueued at most once. As in Chapter 76, define canonical identity for set keys; this example assumes node IDs are stable, canonical strings. If the graph is too large to keep in PHP memory, store adjacency and traversal state elsewhere or stream a suitable representation rather than assuming that a faster queue solves the memory bound.

## Practical Example: In-Process Job Buffer

A queue can also decouple production of work from its handling within one process:

```php
$jobs = new ArrayQueue();
$jobs->enqueue(['id' => 'email-104', 'recipient' => 'a@example.test']);
$jobs->enqueue(['id' => 'email-105', 'recipient' => 'b@example.test']);

while (!$jobs->isEmpty()) {
    $job = $jobs->dequeue();
    sendEmail($job['recipient']);
}
```

Here a producer is the code that enqueues work, a consumer handles each item, and a worker is the running process or task that consumes it. If `sendEmail()` throws, this simple loop has already removed the job; there is no retry or recovery. The example is intentionally process-local: it can smooth work within one request or command, but a PHP-FPM request ending or a CLI process crashing discards the queue. It offers no cross-process coordination, acknowledgement, persistence, visibility timeout, delayed retry, or delivery guarantee. See Chapter 244 for durable queues and their broker-level concerns.

## Production Concerns

An in-memory queue is not a production job system simply because it has FIFO behavior. Durable consumers need explicit failure policy. A process can perform a side effect and then crash before recording success; when work is retried, the side effect may happen twice. Make handlers idempotent where possible, for example by recording a stable job ID under a uniqueness constraint or by using a provider’s idempotency key. Retrying does not by itself make an operation safe.

Retries should be bounded and normally delayed with backoff, often with jitter. Re-enqueuing a failing job immediately can create a hot loop and waste consumer capacity on work that is not making progress. After a configured number of attempts, a poison message may need quarantine or a dead-letter queue for investigation. Those are delivery and operations policies layered above FIFO.

Capacity is also part of correctness. If items arrive faster than consumers finish them, queue depth and oldest-item age grow. An unbounded in-memory queue can exhaust a worker’s memory. A bounded system needs a backpressure policy: slow or reject producers, cap outstanding work, shed low-priority work, or add consumer capacity. Monitor depth, enqueue/dequeue rates, processing latency, retry counts, and failures. More consumers can improve throughput while changing completion order and increasing load on databases or APIs.

Even a durable broker that delivers messages FIFO may not provide global completion order with multiple consumers. Retries, visibility timeouts, partitions, and parallel processing affect what the application observes. If order matters per customer or aggregate, partition by that identity and enforce the rule at the processing boundary. Exactly-once effects generally require application-level idempotency and transactional design, not merely a queue setting.

## Edge Cases and Correctness

- **Empty removal:** Decide whether it returns a sentinel, returns an optional result, or throws. The head-index example throws `UnderflowException` and supports `null` as an item.
- **Reuse after draining:** Reset indices when empty so that a long-lived queue does not keep an ever-growing logical index.
- **Compaction:** Copy only live entries, reset both indices consistently, and test with values on both sides of the threshold.
- **Mutation while iterating:** Prefer an explicit `while (!isEmpty())` consume loop when newly enqueued work should also be processed. A `foreach` loop has different iteration semantics and may not include later additions.
- **Fairness and priority:** FIFO is not priority scheduling. Urgent work or deadlines require a different policy, such as the priority queue in Chapter 84.
- **Unbounded production:** If consumers cannot catch up, a queue only stores the overload; it does not remove it.
- **Multiple processes:** A PHP object in one process is not shared mutable state among PHP-FPM workers.

## Performance and Memory

If every queue operation is O(1), processing `n` entries takes O(n) total time. If each removal shifts all remaining entries, the total may instead be O(n²). That difference is often irrelevant for a few dozen items and material for a queue drained repeatedly at scale.

Space is O(n) for `n` live values, plus structure-specific overhead. A head-index queue can temporarily allocate another live-sized array during compaction. Unsetting values removes references from the queue, but actual process memory returned to the operating system depends on PHP’s allocator and other live references. `SplQueue` uses linked nodes with metadata; a PHP array stores hash-table metadata. Do not infer a universal memory winner without measuring the target workload.

Bound both item count and payload size when queue data comes from external input. For large payloads, store a small identifier and fetch the body when processing rather than copying large objects through every stage. If the data must survive process exit, use a durable store or broker rather than growing an in-memory structure.

## Security

Treat queued data as input at the point of consumption. The producer and consumer may run different code versions or have different permissions; validate the payload and authorize the action against current state. Avoid embedding secrets or unnecessary personal data in long-lived durable messages. For in-memory queues, unbounded user-controlled enqueueing is still a memory-exhaustion risk.

## Testing

Test behavior and invariants rather than private storage details:

1. Enqueue several values and verify FIFO dequeue order.
2. Verify that `peek()` does not remove the front value.
3. Verify empty behavior and confirm that `null` can be enqueued and dequeued distinctly from emptiness.
4. Drain a queue, reuse it, and verify indices reset correctly.
5. Enqueue and dequeue across the compaction threshold; ensure no values are lost, reordered, or returned twice.
6. For breadth-first search, test cycles, disconnected vertices, repeated edges, and a case where depth-first order would differ.
7. For a worker, test handler failure, bounded retries, idempotent effects, and behavior when the process stops before acknowledgement.

Useful properties include: every enqueued item appears at most once in the dequeue sequence; dequeue order matches enqueue order; count equals successful enqueues minus successful dequeues; and an empty queue stays empty after a failed dequeue.

## Common Mistakes

- Repeatedly calling `array_shift()` on a large queue without considering the cost of moving/reindexing remaining numeric entries.
- Testing `array_shift()`'s result against `null` to detect emptiness when `null` is a valid queued value.
- Forgetting to reset or compact a head-index queue in a long-running process.
- Re-enqueueing failed work immediately with no retry limit or delay.
- Assuming an in-process queue is durable, shared between FPM workers, or safe across restarts.
- Assuming FIFO enqueue order guarantees completion order under concurrency.
- Treating a queue as backpressure; it can hide overload while memory, latency, and age grow.
- Using a FIFO queue when the requirement is priority, deadline, or fairness scheduling.

## Senior Engineer Thinking

Start by asking what the queue promises: order, capacity, persistence, acknowledgement, retry, visibility, and failure recovery. The word “queue” names both a basic data structure and a family of production messaging systems, but those systems add guarantees beyond FIFO.

For an algorithm inside one PHP process, the choice between `array_shift()`, a head-index array, and `SplQueue` is mostly about workload scale, memory, and clarity. For jobs that must survive a request or host failure, select a durable system and design the handler for duplicate delivery, backpressure, and observability. Do not let the simple semantics of the data structure imply guarantees that the system does not provide.

## Exercises

1. Implement a small queue backed by a PHP array and `array_shift()`. Add an empty check that allows `null` payloads, then compare its behavior with `ArrayQueue`.
2. Modify `ArrayQueue` to add a non-throwing dequeue API that can represent both “no item” and a `null` item without ambiguity. A result object is one option. Explain your API choice.
3. Run breadth-first search on an adjacency list with a cycle. Return shortest unweighted distance from a start node and explain why marking at enqueue time avoids duplicate work.
4. Add a configurable maximum capacity to an in-memory queue and define whether enqueue blocks, rejects, or drops when full.
5. Design a durable email-job handler that may be delivered twice. Specify its idempotency key, retry schedule, maximum attempts, dead-letter behavior, and metrics.

## Review Questions

1. What invariant does FIFO preserve?
2. Why can repeated `array_shift()` calls be costly, and which part of that claim comes directly from the PHP Manual?
3. How does a head-index queue avoid shifting the remaining values? Why does it need compaction?
4. What does `SplQueue` use internally, and what are its memory trade-offs?
5. Why is a queue useful in breadth-first search?
6. Why does FIFO enqueue order not guarantee FIFO completion order across multiple workers?
7. What failures can cause duplicate job delivery, and how does idempotency help?
8. How can queue depth and oldest-item age reveal backpressure problems?
9. Which guarantees belong to an in-memory data structure, and which require a durable queue system?

## Summary

A queue preserves FIFO order: enqueue at the rear and dequeue from the front. `array_shift()` is simple but reindexes numeric keys and is a poor choice for repeated removal from large arrays. A head-index PHP array and `SplQueue` provide efficient endpoint operations with different memory and implementation trade-offs. Queues support practical algorithms such as breadth-first search and process-local work buffers, but they do not automatically provide persistence, concurrency safety, retries, backpressure, or exactly-once effects. Production job handling must define those guarantees explicitly.

## References

- [PHP Manual: `array_shift()`](https://www.php.net/manual/en/function.array-shift.php)
- [PHP Manual: `SplQueue`](https://www.php.net/manual/en/class.splqueue.php)
- [PHP Manual: `SplQueue::enqueue()`](https://www.php.net/manual/en/splqueue.enqueue.php)
- [PHP Manual: `SplQueue::dequeue()`](https://www.php.net/manual/en/splqueue.dequeue.php)
- [PHP Manual: SPL data structures](https://www.php.net/manual/en/spl.datastructures.php)
- [Chapter 75 — Arrays and Hash Maps](./075-arrays-and-hash-maps.md)
- [Chapter 76 — Sets](./076-sets.md)
- [Chapter 77 — Stacks](./077-stacks.md)
- [Chapter 84 — Priority Queues](./084-priority-queues.md)
- [Chapter 244 — Queues](../../volumes/16-distributed-systems/244-queues.md)
