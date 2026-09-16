---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 229
title: Database Performance
slug: database-performance
status: complete
summary: ../../_ai/chapter-summaries/229-database-performance-summary.md
---

# Chapter 229 — Database Performance

## Why This Matters

Database time is often a dominant part of application latency and capacity. A PHP method can look efficient while issuing hundreds of queries, scanning millions of rows, waiting on locks, transferring unused columns, or holding connections during slow work.

Performance work starts with a measured query and workload. Optimize access path, cardinality, transaction, and data shape together. A faster query that returns incorrect tenant data, weakens a lock invariant, or exhausts the connection pool is not an improvement.

## Query Shape and Indexes

Write the narrowest query that expresses the use case. Filter by selective columns, return only required fields, bound the result, and define ordering. Indexes support actual predicates and ordering; each index consumes storage and adds write cost.

~~~php
<?php

declare(strict_types=1);

function findOpenReservations(PDO $db, int $tenantId, int $customerId): array
{
    $statement = $db->prepare(
        'SELECT id, starts_at, ends_at, status
         FROM reservations
         WHERE tenant_id = :tenant
           AND customer_id = :customer
           AND status IN (:held, :confirmed)
         ORDER BY starts_at, id
         LIMIT 100',
    );
    $statement->execute([
        'tenant' => $tenantId,
        'customer' => $customerId,
        'held' => 'held',
        'confirmed' => 'confirmed',
    ]);

    return $statement->fetchAll(PDO::FETCH_ASSOC);
}
~~~

Named parameter reuse and list binding behavior can vary by PDO driver; an application may need separate placeholders for each status. The example emphasizes projection, tenant scope, deterministic ordering, and a bound result. Verify the generated SQL and plan on the target database.

A composite index such as tenant, customer, status, starts_at may help this query, but column order and data distribution determine whether it is useful. Do not add it from column names alone; inspect plans and the write workload.

## N plus 1 and Set-Based Work

Loading one row and then querying a related record for each result creates N plus 1 queries. Prefer a join, eager-loading strategy, or second bounded query keyed by collected IDs. Keep authorization and tenant predicates on every path.

Set-based SQL lets the database filter, aggregate, and join close to the data. PHP loops are appropriate for domain rules that need application state, but moving a large result into PHP can multiply memory, network, and CPU cost. Compare plan, rows read, transferred bytes, and application time.

## Plans and Cardinality

Use EXPLAIN or the database equivalent to inspect scans, index usage, join order, estimated and actual rows, sorting, and temporary structures. Estimates can be wrong when statistics are stale or predicates are correlated. Capture plans for representative parameter values, including selective and unselective cases.

A query can regress after data grows even when source code is unchanged. Track plan changes and slow-query samples after migrations, index changes, and major data distribution shifts. Do not force an index without measuring the broader workload; a plan that helps one parameter can harm another.

## Transactions and Locks

Transactions define consistency and can hold locks. Keep them as short as the invariant permits, perform validation before opening them where possible, and do not make a remote HTTP call while holding a database lock without an explicit design.

Use conditional updates or version columns for optimistic concurrency; use row locks when serialized access is required. Measure lock waits and deadlocks, and retry deadlock victims only when the entire transaction is safe to repeat.

~~~php
<?php

declare(strict_types=1);

function claimJob(PDO $db, int $jobId, string $workerId): void
{
    $db->beginTransaction();
    try {
        $select = $db->prepare(
            'SELECT id FROM jobs
             WHERE id = :id AND status = :pending
             FOR UPDATE',
        );
        $select->execute(['id' => $jobId, 'pending' => 'pending']);

        if ($select->fetchColumn() === false) {
            throw new DomainException('Job is unavailable');
        }

        $update = $db->prepare(
            'UPDATE jobs SET status = :running, worker_id = :worker
             WHERE id = :id AND status = :pending',
        );
        $update->execute([
            'running' => 'running',
            'worker' => $workerId,
            'id' => $jobId,
            'pending' => 'pending',
        ]);
        $db->commit();
    } catch (Throwable $exception) {
        $db->rollBack();
        throw $exception;
    }
}
~~~

The locking syntax and behavior are database-specific. A production queue may use a lease and visibility timeout rather than a long row lock. The claim needs an expiry or recovery path if the worker crashes.

## Pagination and Batch Work

Offset pagination becomes more expensive as the offset grows because the database may scan and discard earlier rows. Keyset pagination uses a stable ordered key:

~~~sql
SELECT id, starts_at
FROM reservations
WHERE tenant_id = :tenant
  AND (starts_at, id) > (:after_time, :after_id)
ORDER BY starts_at, id
LIMIT 100;
~~~

Tuple comparison syntax varies by database. An equivalent expanded predicate is starts_at greater than the cursor time, or starts_at equal to the time and id greater than the cursor ID. The index and ordering must match the cursor. Batch updates should be bounded and resumable, and should avoid changing the ordering key while scanning.

## Connections and Schema

Each PHP worker can hold a database connection while a request runs. Persistent connections, pools, replicas, and proxies have different lifetimes and failure behavior. A pool limit must account for web workers, queue workers, migrations, and administrative clients.

Release connections and cursors when a long operation no longer needs them. Set connect and query timeouts where supported, classify failures, and prevent one slow report from consuming every connection. A cache can reduce queries but adds freshness, invalidation, and tenant-key requirements.

Indexes, foreign keys, triggers, generated columns, and constraints affect writes. Add an index online or in a staged migration when table size and database capabilities require it. Backfill in bounded batches and monitor lock and replication impact.

## Testing and Operations

Performance tests need representative data volume and distribution. Test query count to catch N plus 1, plans for critical queries, pagination at deep cursors, lock conflicts, deadlock retry, and transaction rollback. Run database-specific integration tests; an in-memory fake cannot prove indexes, isolation, or planner behavior.

Monitor query latency and rows read, slow-query rate, connection usage, lock waits, deadlocks, buffer cache behavior, replication lag, and migration duration. Correlate query regressions with release, schema, and data growth. Redact sensitive parameters in traces and slow-query logs.

## Common Mistakes

- Adding indexes without inspecting query plans and write cost.
- Returning full rows or unbounded results.
- Creating N plus 1 queries through a loop.
- Holding transactions while calling external systems.
- Assuming offset pagination stays fast at large offsets.
- Treating database engines as interchangeable for locking and syntax.
- Retrying deadlocks or timeouts without a repeatable transaction.
- Measuring a query on a tiny dataset and extrapolating to production.

## Senior Engineer Thinking

Database performance is access-path and workload design. Shape queries narrowly, index from measured predicates and ordering, inspect plans and cardinality, keep transactions and locks bounded, paginate with stable keys, and monitor connections, waits, and data growth alongside application latency.

## Exercises

1. Measure a reservation query before and after a composite index using representative tenant distributions.
2. Replace an N plus 1 report with a join or bounded second query and compare rows and query count.
3. Design keyset pagination for a table with duplicate timestamps.
4. Simulate a deadlock and decide whether a transaction retry is safe.

## Review Questions

1. Which query and data characteristics determine whether an index helps?
2. Why does N plus 1 multiply database and application cost?
3. What does an execution plan reveal?
4. Why must transaction retry include the whole transaction?
5. When is keyset pagination preferable to offsets?
6. Which database semantics require engine-specific integration tests?

## Summary

Database performance depends on query shape, indexes, plans, cardinality, transaction and lock duration, pagination, connection capacity, and data growth. Measure representative workloads, prevent N plus 1 and unbounded reads, scope tenants, use stable cursors, stage schema changes, and monitor rows, waits, plans, and replication alongside latency.

## References

- [PHP PDO prepared statements](https://www.php.net/manual/en/pdo.prepare.php)
- [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [MySQL EXPLAIN](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [Martin Fowler: Optimistic Offline Lock](https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html)

