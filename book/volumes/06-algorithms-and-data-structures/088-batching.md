---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 88
title: Batching
slug: batching
status: complete
summary: ../../_ai/chapter-summaries/088-batching-summary.md
---

# Chapter 88 — Batching

## Why This Matters

A PHP importer that sends one database statement per row may spend more time waiting on round trips and transaction setup than transforming the records. Sending every row in one enormous request can replace that overhead with a memory spike, a long lock, a timeout, or a request that exceeds a database or service limit. Batching chooses a bounded middle ground: collect a group of logical items, perform one operation for the group, then continue.

The same trade-off appears when writing files, calling a bulk HTTP endpoint, publishing events, or processing records from a stream. A useful batch is large enough to amortize fixed costs and small enough to bound memory, latency, and recovery work. The batch boundary is an engineering decision, not just a loop optimization.

## Mental Model

Batching groups individual items into a single unit at a boundary:

```text
items → bounded buffer → one bulk operation → next bounded buffer
```

The buffer should flush when a stated condition is reached. Common limits are:

- **Item count:** flush after at most a configured number of records.
- **Bytes:** flush before the encoded request or retained payload exceeds a budget.
- **Age:** flush when the oldest buffered item has waited long enough.
- **Input boundary:** flush at end of file, end of a page, or before graceful shutdown.

Usually the policy combines these: flush when *any* maximum is reached. Count alone does not protect against one unusually large record; bytes alone may allow too many tiny records and bind parameters; time alone does not cap memory. A batch is a local grouping decision. It does not imply FIFO delivery, persistence, retry, or shared state between workers; [Chapter 78 — Queues](078-queues.md) covers those separate concerns.

## Core Concept

Suppose each individual database write costs `r` seconds of fixed request/round-trip overhead plus `c` seconds of per-record work. Writing `n` records one by one pays roughly `n × (r + c)`. With batches of `B` records, a simplified model is:

```text
ceil(n / B) × r + n × c
```

Batching can reduce repeated fixed cost. It cannot remove the per-record work, and the formula leaves out contention, transaction logging, serialization, network payload size, and server-side limits. Bigger batches can improve throughput until another cost dominates; then they can make latency and failure recovery worse.

The batch size therefore has several meanings. A count limit bounds the number of values and often SQL placeholders. A byte limit approximates transfer or memory cost. A time limit bounds how long sparse traffic waits for its batch. Each is a different resource budget. The smallest applicable limit triggers the flush.

## How It Works

### Buffer by more than record count

A first implementation often appends rows until it has `B` items. That is easy to reason about, but it assumes rows are similar in size and that the input arrives quickly enough. A production ingestion path should also define a maximum item size and a maximum age.

For example, an event writer might use illustrative limits of 200 records, an estimated 256 KiB of record data, and 250 ms from the first item in the buffer. These are starting values, not universal recommendations. If a single item exceeds the byte limit, reject it, split it according to the payload format, or route it through a separate path. Do not let one oversize record silently bypass the bound.

There is a subtlety with time-based flushing. A synchronous `foreach` can check age when it receives the next item, but it cannot flush while blocked waiting for a quiet source. If the system must flush even when no new item arrives, the read must yield periodically or an event loop/timer must wake the writer. A time threshold is only as reliable as the mechanism that gets execution back to the flush check.

### Keep transformation and delivery separate

An input iterator can be large or unbounded. The program should transform each item, add it to a bounded batch, submit completed groups, then release references before reading more. This is different from collecting every input into an array and later splitting it: that still retains `O(n)` input memory.

The bulk-operation callback is also an explicit boundary. A callback that loops and makes one network request per item has grouped the PHP values but has not reduced network round trips. To amortize remote overhead, the database driver or API must actually accept a multi-row or bulk request.

### Flush only after the sink succeeds

The buffer can be cleared after the sink confirms success. If submission throws, stop and propagate the error or let a caller-owned recovery policy decide what to do. Do not silently drop the batch and continue as though it was written.

Success can still be ambiguous: a database or service may commit the operation and the connection may fail before PHP receives the response. Retrying can duplicate effects. Stable per-record keys, an idempotent upsert, a durable import cursor, or a staging-and-merge design can make replay safe. Batching does not create exactly-once processing.

## What PHP Does

A batch held in a PHP array keeps references to all its items live until the callback finishes and the array is cleared. If the records are large nested arrays, memory includes those payloads and any temporary serialization or parameter arrays created by the sink. Passing an array to a function does not immediately duplicate it in every case because PHP uses copy-on-write, but modifying or serializing it can require additional storage. Budget peak live memory, not just the visible batch count; [Chapter 74 — Memory Complexity](074-memory-complexity.md) discusses that distinction.

Use a monotonic clock for elapsed-time limits. PHP's `hrtime()` returns a monotonic high-resolution timestamp counted from an arbitrary point, so it is suitable for measuring age; it is not a calendar timestamp to persist. The sample allows an injected clock for deterministic tests. See the [PHP Manual: `hrtime()`](https://www.php.net/manual/en/function.hrtime.php).

The database, HTTP client, or broker determines what “one batch call” means on the wire and whether that operation is atomic. PHP's array shape only groups values in one process. Verify the driver's parameter limit, payload limit, transaction behavior, and timeout separately.

## Practical Example: Bounded Event Inserts

This helper accepts any iterable, so the producer need not materialize all rows. It flushes on count, estimated bytes, elapsed age observed while the loop is running, or end of input. If the callback throws, the exception propagates and the buffer is not treated as successful.

```php
<?php

declare(strict_types=1);

/**
 * Write an iterable in bounded batches.
 *
 * @template T
 * @param iterable<T> $records
 * @param callable(list<T>): void $writeBatch
 * @param callable(T): int $measureBytes Estimated retained or encoded bytes.
 * @param null|callable(): int|float $clockNanoseconds Monotonic test clock.
 */
function processInBatches(
    iterable $records,
    callable $writeBatch,
    callable $measureBytes,
    int $maxItems = 200,
    int $maxBytes = 262_144,
    int $maxAgeMilliseconds = 250,
    ?callable $clockNanoseconds = null,
): void {
    if ($maxItems < 1 || $maxBytes < 1 || $maxAgeMilliseconds < 1) {
        throw new InvalidArgumentException('Batch limits must be positive.');
    }

    $clockNanoseconds ??= static function (): int|float {
        $now = hrtime(true);
        if ($now === false) {
            throw new RuntimeException('Monotonic clock is unavailable.');
        }

        return $now;
    };

    $maxAgeNanoseconds = $maxAgeMilliseconds * 1_000_000;
    $batch = [];
    $batchBytes = 0;
    $firstItemAt = null;

    $flush = static function () use (
        &$batch,
        &$batchBytes,
        &$firstItemAt,
        $writeBatch,
    ): void {
        if ($batch === []) {
            return;
        }

        $writeBatch($batch);
        $batch = [];
        $batchBytes = 0;
        $firstItemAt = null;
    };

    foreach ($records as $record) {
        $recordBytes = $measureBytes($record);
        if (!is_int($recordBytes) || $recordBytes < 0) {
            $flush();
            throw new InvalidArgumentException('Byte measurement must be a non-negative integer.');
        }
        if ($recordBytes > $maxBytes) {
            $flush();
            throw new LengthException('One record exceeds the configured batch byte limit.');
        }

        $now = $clockNanoseconds();
        $ageExpired = $batch !== []
            && $now - $firstItemAt >= $maxAgeNanoseconds;
        $countFull = count($batch) >= $maxItems;
        $bytesWouldOverflow = $recordBytes > $maxBytes - $batchBytes;

        if ($ageExpired || $countFull || $bytesWouldOverflow) {
            $flush();
        }

        if ($batch === []) {
            $firstItemAt = $now;
        }
        $batch[] = $record;
        $batchBytes += $recordBytes;

        if (count($batch) >= $maxItems || $batchBytes >= $maxBytes) {
            $flush();
        }
    }

    $flush();
}
```

The callback can send one multi-row SQL statement per call. Here is the shape for a PDO driver that supports multi-row `VALUES`; the example assumes an `event_log(event_id, payload_json)` table and two bind parameters per row:

```php
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

$writeBatch = static function (array $events) use ($pdo): void {
    if ($events === []) {
        return;
    }

    $values = implode(', ', array_fill(0, count($events), '(?, ?)'));
    $statement = $pdo->prepare(
        'INSERT INTO event_log (event_id, payload_json) VALUES ' . $values,
    );
    if ($statement === false) {
        throw new RuntimeException('Could not prepare event batch.');
    }

    $parameters = [];
    foreach ($events as $event) {
        $parameters[] = $event['event_id'];
        $parameters[] = $event['payload_json'];
    }

    if (!$pdo->beginTransaction()) {
        throw new RuntimeException('Could not begin event batch transaction.');
    }

    try {
        $statement->execute($parameters);
        if (!$pdo->commit()) {
            throw new RuntimeException('Could not commit event batch.');
        }
    } catch (Throwable $exception) {
        if ($pdo->inTransaction()) {
            $pdo->rollBack();
        }
        throw $exception;
    }
};

/** @var iterable<array{event_id: string, payload_json: string}> $events */
$events = readValidatedEvents();
$measureBytes = static fn (array $event): int =>
    strlen($event['event_id']) + strlen($event['payload_json']) + 32;

processInBatches(
    $events,
    $writeBatch,
    $measureBytes,
    maxItems: 200,
    maxBytes: 262_144,
    maxAgeMilliseconds: 250,
);
```

The byte measurement is a conservative application estimate, not an exact wire-size guarantee; leave headroom for SQL/protocol overhead and choose the limit for the actual driver. The `200`-row example uses 400 placeholders, but each database has its own bind-parameter and statement-size limits. A stable unique `event_id` can help make retries safe, but a plain duplicate-key error is not automatically a successful idempotent retry. Use the database's upsert semantics or a durable checkpoint designed for the import.

The sample commits each batch independently. That bounds transaction duration and the amount of work rolled back with one failure, but a later exception does not undo earlier commits. If the entire input must be atomic, a single transaction may be appropriate for a small bounded dataset; it may be disastrous for a large import. Staging and a final merge can provide another boundary. Pick the guarantee before choosing batch size.

## Backpressure and Failure Boundaries

If `writeBatch()` is synchronous, its duration naturally pauses input consumption. That is basic backpressure: the producer advances only after the sink finishes. If the pipeline starts asynchronous writes and queues them without a bound, the in-memory backlog can grow without limit even though every individual batch is small. Limit in-flight batches and define whether the producer waits, rejects, or spills to durable storage when the sink is slower.

Larger batches increase the amount of data that can be affected by one failed request and the number of records that may need replay. A multi-row statement inside a transaction may be all-or-nothing for that transaction, but a remote API can accept some elements and reject others. Read the sink's contract. If it reports per-item outcomes, checkpoint successes independently; if it only reports one outcome, assume the whole batch may need replay after an ambiguous timeout and make each item idempotent.

Validate and normalize records before they enter a batch when possible. One invalid row can reject an entire database statement. For bulk APIs with partial success, record which IDs succeeded, failed permanently, or may be retried. Do not retry a complete batch after a partial side effect unless the repeated effects are safe.

Batching also affects shutdown. A command or worker should flush its final partial batch before reporting success, but obey a shutdown deadline. If the sink is unavailable at shutdown, the process should exit unsuccessfully or persist the uncommitted records somewhere durable; silently discarding them is data loss. [Chapter 65 — Long-Running PHP Processes](../05-php-runtime/065-long-running-php-processes.md) covers worker drain and restart behavior.

## Batch Size and Performance

Let `n` be the number of records and `B` the maximum number per batch. Ignoring partial failures, the pipeline invokes its sink `ceil(n / B)` times and visits each record once, for `O(n)` application work plus the cost of the sink operations. A byte limit or timer may create smaller batches, so count the observed batch distribution rather than assuming every group is full.

The buffer retains `O(B)` record references, but a more useful bound is the configured byte budget plus any largest in-flight sink payload. Actual peak memory can include:

- the source's current record and parser buffers;
- the batch array and nested payloads;
- a serialized request or database parameter array;
- copies made by transformations or client libraries;
- batches still in flight when writes are concurrent.

Start with limits derived from the database's parameter and packet caps, the external service's request limit, acceptable item latency, and the worker's memory budget. Then measure throughput, latency percentiles, bytes per batch, records per batch, flush reason, sink duration, retry count, and peak memory. Increase the limit only while throughput improves and latency, errors, memory, lock duration, and recovery cost remain acceptable.

`maxAgeMilliseconds` in the sample is measured from the first item in each buffer. Under sparse traffic, waiting for a full batch can add unacceptable latency, so time flushing matters. Under heavy traffic, count or byte limits will usually flush first. A single huge record must have its own explicit policy. These constraints resemble independent sides of a queue capacity problem; the queue itself is covered in Chapter 78.

## Database and Network Boundaries

For a database import, batching can reduce round trips and commit overhead, but only if the persistence adapter performs a bulk operation. Repeated prepared-statement executions inside a transaction may amortize commit and lock setup while retaining one execution per row. A multi-row insert can reduce statement calls further, but can hit placeholder limits or large query plans. Measure the actual driver and database.

Do not replace database filtering with a batch loop over every row unless the application needs to inspect or transform those rows. Use an index and a bounded keyset cursor for large scans where possible. Offset pagination can become expensive and shifting data may cause rows to be skipped or repeated; the read strategy and write batch size are separate decisions. See later database chapters on indexes, transactions, and large datasets.

For HTTP APIs, honor documented per-request item and byte limits, timeouts, and per-item responses. A timeout does not reveal whether the server applied the batch. Include stable item identifiers so retries can be deduplicated, and avoid retrying indefinitely on permanent validation failures. [Chapter 67 — Streams](../05-php-runtime/067-streams.md) covers bounded data transfer and timeout semantics.

## Security

Batch inputs still need validation and authorization. A bulk endpoint must not skip per-record tenant or permission checks just because it handles many records at once. Validate each item's size and shape before buffering, cap the number and total bytes accepted, and redact sensitive fields from batch-level logs.

For SQL, bind every value. Dynamic placeholder count can depend on the validated batch size, but table and column names should remain fixed or selected from an allowlist. Do not concatenate user-controlled values into the SQL statement. Large batches can turn an otherwise modest request into a denial-of-service vector through CPU, memory, query size, or lock retention.

## Testing

Test both the grouping policy and the sink boundary:

1. Empty input makes no sink call; fewer than the count limit flushes once at end of input.
2. Exactly `maxItems` values flush once, and `maxItems + 1` values flush with no item lost or duplicated.
3. Byte limits flush before the next record would exceed the budget; one oversized record follows the documented rejection policy.
4. A fake monotonic clock verifies age-triggered flushing deterministically without sleeping.
5. The callback receives only non-empty batches and sees the same order and contents as the input.
6. If the callback throws, assert that the error propagates and the caller does not checkpoint that batch as complete.
7. Inject database failure halfway through an import, rerun it, and verify the idempotency/checkpoint contract.
8. Test signal or cancellation during input and during a write; verify final flush or explicit failure behavior.
9. For a bulk HTTP sink, test a response with mixed per-item results and a timeout after the remote side may have accepted the request.

Avoid relying on real sleeps for time-trigger tests; the injected clock makes boundary cases repeatable. Integration tests should use the target database or service to verify placeholder limits, transaction rollback, duplicate-key handling, and actual request size.

## Common Mistakes

- Setting a count limit while ignoring that record sizes vary widely.
- Accumulating the full source in memory and calling that batching.
- Building a PHP batch but still sending one network request per record.
- Treating a batch as automatically atomic across database, broker, or HTTP systems.
- Retrying after a timeout without a stable idempotency key or checkpoint.
- Committing per batch while assuming earlier batches roll back when a later batch fails.
- Using a time threshold in a blocking loop and assuming it fires while the source is idle.
- Increasing batch size to compensate for a sink whose sustained throughput is below the producer rate.
- Running several asynchronous batches without bounding in-flight bytes.

## Senior Engineer Thinking

Write down the guarantees at each boundary: when a source record is considered read, when a batch is complete, what a sink success means, and when progress is durable. Then choose limits from payload shape, database/service constraints, memory, throughput, and latency. A batch size that is safe for one endpoint or database driver may be unsafe for another.

Next ask what happens after partial progress. Which records may already have committed? Can a batch be replayed safely? Is the checkpoint advanced per item or per batch? Will a shutdown flush finish before its deadline? These decisions determine whether batching is a performance technique or a source of data loss and duplicate effects. Measure fill ratio, latency, errors, and memory in production so that later tuning follows workload evidence rather than guesswork.

## Exercises

1. Change the example so it flushes when the next record would exceed the byte limit, even when the count limit is not reached. Test a batch whose total is exactly the limit.
2. Add a deterministic fake clock and test a partial batch that crosses the maximum age only when another source item arrives. Explain how an event-loop timer would change the idle case.
3. Calculate how many placeholders a multi-row insert needs for `B` records with `C` bound columns. Pick a safe `B` for a driver-specific limit and leave room for fixed parameters.
4. Design a restartable file import with a unique source ID, per-batch transaction, durable checkpoint, and a failure after commit but before checkpoint update. Show how rerun avoids loss and duplication.
5. Compare one transaction for the whole file, one transaction per batch, and one transaction per row. Discuss lock duration, rollback work, throughput, and partial progress.
6. Design a bounded asynchronous writer. State the maximum number of in-flight batches, backpressure behavior, and shutdown policy when one request is still pending.

## Review Questions

1. Which fixed costs can batching amortize, and which per-record costs remain?
2. Why combine item-count, byte, and age limits?
3. Why does a synchronous consumer naturally apply backpressure, and how can async buffering remove that bound?
4. What can PHP know after a network timeout during batch submission?
5. Why does committing each batch not make an entire import atomic?
6. How does a monotonic clock help with a maximum batch age?
7. Why can a time threshold fail to flush while a blocking source is idle?
8. What additional memory may coexist with the PHP batch array?
9. When does a bulk operation reduce round trips, and when has the code only grouped values locally?

## Summary

Batching groups records to amortize fixed call, transaction, or network costs while bounding memory, latency, and recovery work. Combine maximum count and bytes with an age trigger and an end-of-input flush; a blocking source needs an event-loop or timeout wakeup to flush while idle. The sink must perform a real bulk operation for round trips to fall. Batch commits create partial progress, and ambiguous failures require idempotent records or durable checkpoints. Choose limits from driver and service caps, payload sizes, memory and latency budgets, then measure throughput, batch fill, failures, retries, and peak live memory.

## References

- [PHP Manual: `hrtime()`](https://www.php.net/manual/en/function.hrtime.php)
- [PHP Manual: PDO Transactions](https://www.php.net/manual/en/pdo.transactions.php)
- [Chapter 60 — PHP CLI](../05-php-runtime/060-php-cli.md)
- [Chapter 65 — Long-Running PHP Processes](../05-php-runtime/065-long-running-php-processes.md)
- [Chapter 67 — Streams](../05-php-runtime/067-streams.md)
- [Chapter 74 — Memory Complexity](074-memory-complexity.md)
- [Chapter 78 — Queues](078-queues.md)
- [Chapter 86 — Intervals](086-intervals.md)
- [Chapter 90 — Streaming Algorithms](090-streaming-algorithms.md)
- [Chapter 104 — SQL for PHP Developers](../../volumes/08-databases/104-sql-for-php-developers.md)
- [Chapter 108 — Indexes](../../volumes/08-databases/108-indexes.md)
- [Chapter 114 — Transactions](../../volumes/08-databases/114-transactions.md)
- [Chapter 120 — Large Datasets](../../volumes/08-databases/120-large-datasets.md)
- [Chapter 244 — Queues](../../volumes/16-distributed-systems/244-queues.md)
