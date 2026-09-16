---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 120
title: Large Datasets
slug: large-datasets
status: complete
summary: ../../_ai/chapter-summaries/120-large-datasets-summary.md
---

# Chapter 120 — Large Datasets

## Why This Matters

A query that is correct for 500 rows can take down a worker when it returns 50 million. The database must find the rows, the driver must transfer them, PHP must represent them, and the consumer must process them. Each layer can become the bottleneck, and PHP's request memory limit does not protect the database or the network.

Large-data work is therefore a boundary-design problem. Decide whether the operation is an interactive page, a bounded report, a background export, a backfill, or a continuous stream. Each has a different latency, memory, retry, and consistency contract. [Chapter 119](119-pagination.md) covers serving a bounded slice to a client; this chapter focuses on processing a large result or changing many rows safely.

## The Memory Model

This familiar code materializes the entire result set twice: once in the driver and once in the PHP array.

```php
<?php

declare(strict_types=1);

$cutoff = '2025-01-01T00:00:00+00:00';
$statement = $pdo->prepare(
    'SELECT id, email FROM users WHERE created_at < :cutoff'
);
$statement->execute(['cutoff' => $cutoff]);
$rows = $statement->fetchAll(PDO::FETCH_ASSOC);

foreach ($rows as $row) {
    sendToArchive($row);
}
```

`fetchAll()` is the visible allocation, but buffering behavior also depends on the PDO driver and its configuration. A generator avoids retaining already-processed PHP rows, although it cannot make an unbounded database query cheap by itself. Check the driver documentation and measure peak memory with the actual database connection.

The working-set goal is usually `O(batch size)` rather than `O(total rows)`. A streaming loop can still fail if each item creates a large object graph, if a transaction remains open for hours, or if the downstream service is slower than the database. Memory, lock duration, and throughput all belong in the design.

## Streaming Rows

For a driver that supports unbuffered or forward-only reads, a generator gives the caller a small interface and releases each row after processing:

```php
<?php

declare(strict_types=1);

/** @return Generator<int, array{id: int, email: string}> */
function usersBefore(PDO $pdo, string $cutoff): Generator
{
    $statement = $pdo->prepare(
        'SELECT id, email
         FROM users
         WHERE created_at < :cutoff
         ORDER BY id'
    );
    $statement->execute(['cutoff' => $cutoff]);

    while ($row = $statement->fetch(PDO::FETCH_ASSOC)) {
        yield [
            'id' => (int) $row['id'],
            'email' => (string) $row['email'],
        ];
    }
}

foreach (usersBefore($pdo, '2025-01-01T00:00:00+00:00') as $user) {
    archiveUser($user);
}
```

The generator's memory behavior is useful only while the statement and connection remain usable. Do not issue another query on a connection when the driver forbids it during an unbuffered result. A long stream also holds a snapshot or cursor open in some engines, which can delay cleanup of old row versions. Keep the query narrow, consume it promptly, and use a dedicated connection when the driver requires it.

For a report written to a file, stream output as well:

```php
$output = fopen('php://output', 'wb');
fputcsv($output, ['id', 'email']);

foreach (usersBefore($pdo, $cutoff) as $user) {
    fputcsv($output, $user);
}
```

An HTTP request is usually the wrong place for an hours-long export. Queue a job, write to durable storage, record progress, and let the client download a completed artifact. A CLI worker has a clearer timeout and retry contract, but it still needs a memory budget and a plan for interruptions.

## Keyset Batches and Checkpoints

Offset is a poor work cursor for a changing table: it becomes slower at deep offsets and rows can move between batches. Use an ordered, unique key and remember the last successful value:

```sql
SELECT id, email
FROM users
WHERE id > :last_id
ORDER BY id
LIMIT :batch_size;
```

The PHP worker processes one batch, commits the durable side effect, and stores the greatest `id` only after the batch succeeds. If the worker stops before the checkpoint, it repeats a batch. That is safe only when the operation is idempotent, for example an upsert keyed by `user_id` or an archive write with a unique source key.

For a composite order, checkpoint every key in the predicate, just as a cursor does:

```sql
WHERE (created_at, id) > (:created_at, :id)
ORDER BY created_at, id
```

If new rows must be excluded from a finite migration, capture a cutoff at the start and include it in every batch. If rows can be deleted, a numeric checkpoint can safely skip no existing row, but it cannot discover a row inserted with an older key; the migration contract must state whether that is acceptable.

## Batching Writes and Transactions

One transaction per row creates too much round-trip and commit overhead. One transaction for ten million rows creates a huge rollback surface, holds locks for too long, and can produce a large transaction log. A bounded batch is usually a better compromise:

```php
<?php

declare(strict_types=1);

$update = $pdo->prepare(
    'UPDATE users
     SET email_normalized = :normalized
     WHERE id = :id'
);

foreach (usersBefore($pdo, $cutoff) as $user) {
    $pdo->beginTransaction();
    try {
        $update->execute([
            'normalized' => strtolower($user['email']),
            'id' => $user['id'],
        ]);
        $pdo->commit();
    } catch (Throwable $exception) {
        if ($pdo->inTransaction()) {
            $pdo->rollBack();
        }
        throw $exception;
    }
}
```

This example intentionally shows the transaction boundary, but a production backfill should usually fetch and update explicit batches, then commit once per batch. Keep the batch small enough that a retry is affordable and large enough to amortize round trips. Set statement and lock timeouts where the database supports them, and record the checkpoint only in the same transaction as the work it acknowledges.

Bulk SQL can be much faster than a PHP loop when the transformation is expressible in the database:

```sql
UPDATE users
SET email_normalized = lower(email)
WHERE id > :last_id AND id <= :batch_end;
```

Measure both options. A set-based update reduces data movement, but it may hold locks on many rows and can make replication lag or vacuum/undo pressure worse. A PHP loop offers per-row branching and an external API call, but it pays for round trips and must handle partial progress.

## Loading and Exporting Efficiently

Database-native bulk loaders and exporters often outperform individual `INSERT` statements. PostgreSQL's `COPY` and MySQL's bulk-load facilities have different security, file, and transaction semantics; use the engine's documented interface and do not assume that a local file is visible to the server process. Validate delimiters, encoding, line endings, and null representation before loading external data.

For application-side inserts, prepare once, bind the right types, and insert in bounded transactions. Multi-row `INSERT` reduces round trips but has a parameter limit and a larger statement failure surface. If a batch can be retried, use a natural idempotency key or an upsert policy rather than relying on “the process probably stopped before the commit.”

Avoid fetching columns that the operation does not need. A covering index or narrow projection may reduce I/O, but adding an index also increases write cost and storage. Start from the access pattern and confirm with an execution plan as described in [Chapter 110](110-query-plans.md).

## Consistency While Data Changes

An export made from many independent queries is not automatically a snapshot. Rows can be inserted, updated, or deleted between batches. Decide whether the consumer needs a point-in-time view, a repeatable feed, or an eventually complete result.

A database snapshot or repeatable-read transaction can provide a consistent view, but a transaction held across a large export may retain old versions and block maintenance. A cutoff column, a change-data-capture stream, or a versioned export can provide a more operationally manageable contract. For a backfill, repeatable work plus an idempotent checkpoint is often more useful than a multi-hour transaction.

## Backpressure, Retries, and Observability

The producer must not outrun the consumer. Bound the number of in-flight rows, flush output incrementally, and pause or queue work when a downstream system slows. A generator limits retained rows; it does not limit external concurrency if each iteration dispatches asynchronous work without waiting.

Make retries explicit. A process can die after the side effect succeeds but before the checkpoint commits. Design for at-least-once execution: use idempotent writes, a unique operation key, or a reconciliation query. Record batch start and end keys, row counts, duration, retry count, error class, and last successful checkpoint. Alert on stalled progress and replication lag, not only on process death.

## Common Mistakes

- Calling `fetchAll()` for a result whose size is controlled by production data.
- Assuming a generator disables every form of driver buffering.
- Holding one transaction open for an entire migration.
- Updating a checkpoint before the corresponding side effect commits.
- Retrying a batch whose external effect is not idempotent.
- Using offset batches for a mutable, deep table scan.
- Running an export in a request with no timeout or resumable artifact.
- Measuring only PHP memory while ignoring database snapshots, logs, locks, and network throughput.

## Senior Engineer Thinking

Large-data work has a unit of progress. Name that unit, its ordering key, its consistency boundary, its retry behavior, and the evidence that proves progress. Choose streaming when a sequential read is enough, keyset batches when work must resume, set-based SQL when the database can transform data safely, and a durable job when the operation outlives an HTTP request.

## Exercises

1. Rewrite a `fetchAll()` export as a forward-only generator and identify the driver assumptions that must be tested.
2. Design a resumable backfill with a keyset checkpoint, a batch transaction, and an idempotent retry rule.
3. Compare a set-based `UPDATE` with a PHP transformation for a table of 100 million rows. Include locks, logging, retries, and observability.
4. Define a consistency contract for an export while source rows continue to change.

## Review Questions

1. Why does a generator not by itself guarantee constant memory?
2. What must be true before a checkpoint can advance?
3. Why can one transaction for a massive job be operationally harmful?
4. When is set-based SQL preferable to a PHP loop, and what new pressure can it create?
5. How should a worker behave if the side effect succeeds but the process dies before checkpointing?

## Summary

Large datasets require an explicit memory, progress, consistency, and retry contract. Stream narrow results when the driver permits it, use keyset checkpoints for resumable batches, bound transaction size, and make repeated work idempotent. Choose set-based SQL, PHP processing, or a durable export job according to transformation complexity, lock pressure, and consumer needs.

## References

- [PHP Manual: Generators](https://www.php.net/manual/en/language.generators.php)
- [PHP Manual: PDOStatement::fetch](https://www.php.net/manual/en/pdostatement.fetch.php)
- [PostgreSQL: `COPY`](https://www.postgresql.org/docs/current/sql-copy.html)
- [MySQL: Loading data](https://dev.mysql.com/doc/refman/8.4/en/load-data.html)
