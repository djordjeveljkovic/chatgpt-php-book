---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 109
title: Composite Indexes
slug: composite-indexes
status: complete
summary: ../../_ai/chapter-summaries/109-composite-indexes-summary.md
---

# Chapter 109 — Composite Indexes

## Why This Matters

A real query usually filters on more than one fact: a tenant, a state, and a time range; or an account and a stable ordering key. Several single-column indexes may look helpful, yet still force the database to combine indexes, fetch many rows, and sort them. A composite index stores several key columns in one ordered structure so the engine can navigate the combination directly.

Column order is the central design decision. `(organization_id, state, created_at)` and `(state, organization_id, created_at)` are different access paths. A composite index should follow the application's common predicates and ordering, measured against actual data. [Chapter 108](108-indexes.md) introduced index maintenance and selectivity; this chapter focuses on multi-column order and the leftmost-prefix behavior of B-trees.

## Mental Model

For an index on `(tenant_id, status, created_at)`, keys are ordered as if sorted by a dictionary:

```text
(10, 'open',   2026-01-02)
(10, 'open',   2026-01-07)
(10, 'closed', 2026-01-03)
(11, 'open',   2026-01-01)
```

All rows for one `tenant_id` are adjacent. Within that tenant, statuses are grouped; within a status, dates are ordered. The first column establishes the broadest navigation boundary. A query that supplies the first column can usually use the index more effectively than one that only supplies a later column.

The approximate seek cost is still logarithmic in the index size, followed by the entries in the matching range. But the number of entries and table rows fetched depends on the prefix's selectivity. A composite index is not a promise that every listed predicate becomes a constant-time lookup.

## The Leftmost Prefix

For a B-tree index `(a, b, c)`, a typical engine can use these leading portions:

| Query shape | Useful prefix |
| --- | --- |
| `a = ?` | `a` |
| `a = ? AND b = ?` | `a, b` |
| `a = ? AND b = ? AND c >= ?` | `a, b, c` with a range ending the seek |
| `b = ?` | Usually no seek on the index's beginning |
| `a = ? AND c = ?` | `a`; `c` may be a residual filter |

The exact optimizer can use skip scans, index intersection, or other strategies, but these are engine-specific and should be verified. Design for common, stable access patterns rather than relying on an optional optimization.

Equality predicates generally narrow a prefix before a range predicate. Once a range such as `created_at >= :since` begins scanning, later columns may not further narrow the ordered range in the same way. This is why `(tenant_id, status, created_at)` is often a useful shape for “one tenant, one status, recent rows.” The ideal order can change when the query has different equality fields, sort requirements, or data distributions.

## Minimal Example

An event feed commonly needs the newest events for one account:

```sql
CREATE INDEX events_account_kind_time_idx
    ON events (account_id, kind, occurred_at DESC);

SELECT id, payload, occurred_at
FROM events
WHERE account_id = :account_id
  AND kind = :kind
  AND occurred_at >= :since
ORDER BY occurred_at DESC
LIMIT 50;
```

The index narrows to an account and kind, then scans the time range in index order. The engine may still choose another plan when the range is broad or the table is small. `DESC` support and whether a scan can satisfy the requested direction depend on the database version and index implementation.

The PHP boundary is unchanged:

```php
<?php

declare(strict_types=1);

function recentEvents(PDO $db, int $accountId, string $kind, string $since): array
{
    $statement = $db->prepare(
        <<<'SQL'
        SELECT id, payload, occurred_at
        FROM events
        WHERE account_id = :account_id
          AND kind = :kind
          AND occurred_at >= :since
        ORDER BY occurred_at DESC
        LIMIT 50
        SQL
    );
    $statement->execute([
        'account_id' => $accountId,
        'kind' => $kind,
        'since' => $since,
    ]);

    return $statement->fetchAll(PDO::FETCH_ASSOC);
}
```

Do not concatenate the values to make an index “work.” An index changes access cost; it does not change SQL injection rules.

## Choosing Column Order

Start from the query contract, then compare candidate orders. Suppose a `jobs` table serves these requests:

```sql
-- A: pending jobs for one queue, oldest first
SELECT id, payload
FROM jobs
WHERE queue = :queue AND status = 'pending'
ORDER BY available_at, id
LIMIT 100;

-- B: all jobs for one owner, newest first
SELECT id, status, available_at
FROM jobs
WHERE owner_id = :owner_id
ORDER BY available_at DESC, id DESC
LIMIT 100;
```

One index cannot necessarily optimize both shapes. `(queue, status, available_at, id)` is designed for A. `(owner_id, available_at, id)` is designed for B. Combining all columns into one wide index can make writes and storage expensive while still failing to provide a useful leading prefix for either workload. Measure query frequency and latency before adding both.

A common heuristic is equality columns first, followed by range and ordering columns. It is a starting point, not a theorem: a highly selective range column, a required sort, or a tenant boundary can change the best order. Also consider the size of index entries. Large text keys increase index pages and cache pressure; a narrow stable key often performs better.

## Covering Indexes

An index is covering when it contains all values needed to answer a query, so the engine can avoid fetching table rows. Some engines support included, non-key columns; others require adding columns to the key, which changes ordering and size. For example, PostgreSQL supports `INCLUDE` columns:

```sql
CREATE INDEX tickets_org_state_created_cover_idx
    ON tickets (organization_id, state, created_at DESC)
    INCLUDE (id, subject);
```

The key columns provide navigation; included columns help return the projection but do not become ordering keys. This syntax is PostgreSQL-specific. MySQL's covering-index behavior generally comes from storing selected columns in the index key, subject to its storage engine rules. Use the target engine's documentation and verify the plan.

Covering indexes can reduce heap or clustered-row lookups, but they increase storage and write work. They also do not remove visibility checks, filtering, locking, or other engine-specific costs. Keep the selected projection purposeful instead of adding every column “just in case.”

## Bad Example: Independent Indexes for a Combined Predicate

Suppose the only indexes are:

```sql
CREATE INDEX orders_customer_idx ON orders (customer_id);
CREATE INDEX orders_status_idx   ON orders (status);
```

For this query:

```sql
SELECT id, total_cents
FROM orders
WHERE customer_id = :customer_id
  AND status = 'paid'
ORDER BY created_at DESC
LIMIT 50;
```

The optimizer may use one index and filter the other condition, combine indexes if supported, or scan the table. It may also sort the matching rows because neither index supplies `created_at` order. These are valid choices, not evidence that the optimizer is broken. The workload may justify a composite `(customer_id, status, created_at)` index, but confirm with an actual plan and write-cost measurement.

## Better Example: Stable Ordering

Pagination needs a deterministic order. Ordering only by `created_at` leaves ties whose relative order may vary. Add a unique tie-breaker and align the index:

```sql
CREATE INDEX messages_conversation_time_id_idx
    ON messages (conversation_id, sent_at DESC, id DESC);

SELECT id, body, sent_at
FROM messages
WHERE conversation_id = :conversation_id
ORDER BY sent_at DESC, id DESC
LIMIT 50;
```

For keyset pagination, the next page can use the last pair of values:

```sql
SELECT id, body, sent_at
FROM messages
WHERE conversation_id = :conversation_id
  AND (sent_at, id) < (:last_sent_at, :last_id)
ORDER BY sent_at DESC, id DESC
LIMIT 50;
```

Row-value comparison support and null semantics vary by engine. The equivalent disjunction is:

```sql
WHERE conversation_id = :conversation_id
  AND (
      sent_at < :last_sent_at
      OR (sent_at = :last_sent_at AND id < :last_id)
  )
```

The index follows the same prefix and ordering. [Chapter 119](119-pagination.md) develops pagination trade-offs further.

## Edge Cases and Engine Differences

- **Later-column predicates:** A predicate on `c` in `(a, b, c)` may be evaluated after scanning the `a` range rather than used to seek directly. Read the plan rather than assuming it is ignored or fully used.
- **Mixed directions:** A query ordering by `a ASC, b DESC` may require an engine/version-specific index definition or an additional sort.
- **OR predicates:** `a = 1 OR b = 2` does not map cleanly to one `(a, b)` index. Rewriting as a union or adding separate access paths may help, but compare duplicate and sort costs.
- **Nullable keys:** Null ordering differs by engine and direction. A cursor or business rule that depends on null position must specify it explicitly.
- **Collations:** Composite text keys use each column's comparison rules. A query with a different collation can lose ordered-index use.
- **Partitioning:** A local index may be applied per partition after partition pruning. Index order does not replace a partition key or correct pruning predicate.
- **Expression and partial indexes:** PostgreSQL and other engines offer specialized forms with different syntax and restrictions. Treat them as vendor-specific schema decisions.

## Performance and Index Lifecycle

A wide composite index has more bytes per entry. That increases cache misses, build time, replication or backup volume, and write amplification. Estimate the index size from the engine's tooling and test insert/update throughput with representative concurrency. A small read improvement may not justify slowing every write on a high-volume table.

Look for redundancy. If `(tenant_id, status, created_at)` exists, a separate `(tenant_id)` index may be unnecessary for many workloads, although a narrower index can still be cheaper for some queries. Never drop an index solely because its prefix appears covered: check query plans, constraint dependencies, foreign-key behavior, and operational usage first.

When changing order or replacing an index, create the candidate safely, compare plans, observe write metrics, and remove the old one only after a usage window. The migration method matters: concurrent or online creation options differ among PostgreSQL, MySQL, and managed services.

## Testing

Use a fixture that includes skew: a few large tenants, many small tenants, common and rare statuses, duplicate timestamps, and enough rows to cross memory and page boundaries. Compare queries with and without each candidate index. Record estimated and actual rows, sort or temporary-file behavior, buffer reads, and wall-clock latency under a warmed and cold cache where your test environment can control it.

A repository test can verify stable ordering and keyset boundaries:

```php
$firstPage = $repository->list($conversationId, null);
$last = $firstPage[array_key_last($firstPage)];
$secondPage = $repository->list($conversationId, [
    'sent_at' => $last['sent_at'],
    'id' => $last['id'],
]);

self::assertSame([], array_intersect(
    array_column($firstPage, 'id'),
    array_column($secondPage, 'id'),
));
```

The database integration test should prove there are no gaps or duplicates when timestamps tie. A plan assertion should remain at the level your engine version can support; exact node text is often too brittle for ordinary unit tests.

## Common Mistakes

- Assuming column order is interchangeable.
- Putting a low-value column first because it appears first in the `WHERE` clause.
- Believing every predicate after a range is equally useful for index navigation.
- Adding all selected columns to a covering index without measuring its size.
- Maintaining several overlapping indexes with no usage review.
- Ordering pages by a non-unique timestamp.
- Relying on vendor-specific skip scans, row comparisons, or included columns without documenting the dependency.
- Comparing plans only on uniform toy data.

## Senior Engineer Thinking

Choose a composite index by asking which contiguous range of keys the common query can navigate. Separate “can filter eventually” from “can seek to a narrow range” and from “can deliver rows already ordered.” Those are different benefits with different costs.

An index design is also a compatibility decision. Record the database engine and version, collation, migration method, expected data distribution, and queries that justify the index. Revisit the choice when tenant sizes, status ratios, retention, or workload shape changes. A plan that was correct for one distribution may become expensive after growth.

## Exercises

1. For `(tenant_id, state, created_at)`, write five queries and predict which leading prefix each can use. Verify the predictions with your database's plan tool.
2. Given the two `jobs` queries in this chapter, propose the smallest index set that meets a stated latency target. Include the write cost of each candidate.
3. Populate duplicate timestamps and implement keyset pagination with `(sent_at, id)`. Prove that pages contain no duplicates or gaps.
4. Compare a PostgreSQL `INCLUDE` index with a key-column covering index in your target engine. Measure size, write cost, and plan behavior.

## Review Questions

1. What does “leftmost prefix” mean for a B-tree composite index?
2. Why do equality columns commonly precede a range column?
3. Why can `(a, b)` help a query on `a` but not usually one on only `b`?
4. How does a covering index reduce work, and what does it cost?
5. Why does a unique tie-breaker matter for pagination?
6. When might two single-column indexes be preferable to one composite index?
7. Which composite-index behaviors require checking the target database's documentation?

## Summary

A composite B-tree index orders a tuple of columns, so its usefulness depends on the leftmost prefix, predicate shapes, ranges, ordering, and data distribution. Start with equality and tenant boundaries, then account for ranges and stable ordering; validate the result with plans and representative data. Covering indexes can avoid table lookups but increase storage and write work. Keep the index set small enough to operate, and document engine-specific assumptions.

## References

- [PostgreSQL documentation: multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [PostgreSQL documentation: index-only scans and covering indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)
- [MySQL 8.4 Reference Manual: multiple-column indexes](https://dev.mysql.com/doc/refman/8.4/en/multiple-column-indexes.html)
- [MySQL 8.4 Reference Manual: covering indexes](https://dev.mysql.com/doc/refman/8.4/en/ix01.html)
- [PHP Manual: PDO::prepare](https://www.php.net/manual/en/pdo.prepare.php)
