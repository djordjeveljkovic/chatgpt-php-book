---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 107
title: Query Design
slug: query-design
status: complete
summary: ../../_ai/chapter-summaries/107-query-design-summary.md
---

# Chapter 107 — Query Design

## Why This Matters

A query is an executable part of an application, not a string hidden inside a repository method. Its predicates define which facts are considered, its joins define cardinality, its ordering defines user-visible behavior, and its selected columns determine transfer and hydration cost. Poor query design can make a correct PHP application slow, memory-heavy, or subtly wrong.

Start with the required result and its invariant. Then choose the relational operations that express it, inspect the expected cardinality and indexes, and measure the plan with realistic data. [Chapter 104](104-sql-for-php-developers.md) introduces SQL; later chapters examine indexes, joins, aggregation, and plans in depth.

## Start With Grain

The grain is what one output row represents. A query for one row per customer must not accidentally return one row per order item. Write the grain before writing SQL:

```text
required result: one row per customer
filter: customers active on the report date
optional relation: zero or more orders
output: customer plus aggregate order facts
```

If the query joins two one-to-many relations before aggregating, it can multiply rows. Aggregate each child relation to the parent grain or use a correlated `EXISTS`/subquery when only presence is needed. [Chapter 112](112-joins.md) covers join cardinality and fan-out.

## Predicates and Nulls

Use explicit predicates and understand SQL's three-valued logic. `NULL = NULL` is not true; it is unknown. Use `IS NULL` or `IS NOT NULL` for null checks. Keep tenant, authorization, and soft-delete predicates visible in the query boundary rather than relying on a caller to remember them.

Half-open time ranges avoid double-counting adjacent windows:

```sql
WHERE happened_at >= :from_time
  AND happened_at < :to_time
```

Bind values rather than interpolating them. Validate that `from_time` precedes `to_time`, and define the time zone and precision at the API boundary.

## Select Only What the Boundary Needs

`SELECT *` couples a repository to schema growth and transfers columns the caller may not need. Choose a projection that matches the DTO or view model. Fewer bytes reduce database I/O, network transfer, PHP zvals, and hydration work. Do not select a large text or JSON column merely to check whether a row exists.

Use `EXISTS` when the result is boolean:

```sql
SELECT EXISTS (
    SELECT 1
    FROM reservations
    WHERE court_id = :court_id
      AND starts_at < :requested_end
      AND ends_at > :requested_start
) AS has_conflict;
```

The database may stop searching after the first match. The exact plan is engine-specific, but the query communicates the required cardinality better than loading all conflicts into PHP.

## Ordering Is Part of the Contract

Rows have no guaranteed order without `ORDER BY`. If the ordering column is not unique, add a deterministic tie-breaker:

```sql
ORDER BY created_at DESC, id DESC
```

The order should align with pagination, indexes, and the API's meaning. Avoid sorting in PHP after loading a large result when the database can use an index or sort closer to the data. Conversely, a domain-specific comparator involving external API data may belong in PHP after the database has narrowed the candidate set.

## Avoid N+1 Queries

An N+1 pattern loads a parent list and then queries children once per parent. The number of round trips grows with the result size. Replace it with a join, a grouped child query using `WHERE parent_id IN (...)`, or an eager-loading mechanism that generates bounded queries. Measure the result shape: one join can create duplication and memory pressure when each parent has many children.

The best query count is not always one. Two bounded queries with clear grains can be cheaper and easier to paginate than one enormous join. Compare rows scanned, rows returned, network bytes, and PHP memory rather than optimizing a round-trip count in isolation.

## Query Shape and Index Use

A useful predicate can still be expensive when it applies a function to an indexed column, casts incompatible types, or uses a leading wildcard:

```sql
-- Often prevents ordinary index use on email:
WHERE LOWER(email) = :email

-- Store a normalized value or use an engine-supported functional index.
WHERE email_normalized = :email_normalized
```

Do not infer index use from SQL appearance. [Chapters 108–111](108-indexes.md) cover index design and plan inspection. Keep data types consistent across parameters and columns, and verify with `EXPLAIN` on representative data.

## PHP Boundary and Repository Design

A repository method should expose a domain-shaped result and make query policy explicit:

```php
<?php

declare(strict_types=1);

/** @return list<array{id: int, title: string}> */
function findPublishedArticles(PDO $db, int $limit): array
{
    $limit = max(1, min($limit, 100));
    $statement = $db->prepare(
        'SELECT id, title
         FROM articles
         WHERE status = :status
         ORDER BY created_at DESC, id DESC
         LIMIT ' . $limit
    );
    $statement->execute(['status' => 'published']);

    return $statement->fetchAll(PDO::FETCH_ASSOC);
}
```

The validated integer is embedded because `LIMIT` parameter behavior varies by PDO driver; a fixed upper bound prevents arbitrary SQL text. The status remains a bound value. In a production repository, prefer a driver-specific parameter approach if supported and test it against the configured driver.

Do not return raw database rows by default when the schema contains sensitive or unstable columns. Map to a DTO or explicit array shape at the boundary, and keep hydration cost proportional to the endpoint.

## Failure and Operational Behavior

A query can fail because of a timeout, deadlock, unavailable database, canceled statement, constraint, or exhausted connection pool. Set operation-appropriate timeouts and translate expected failures into domain outcomes. Do not retry every query: a read may be safe to retry while a write may need an idempotency key and a fresh transaction.

Log query fingerprints, duration, row counts, and operation identifiers without logging secrets. Track slow-query rates and plan changes after schema or data-distribution changes. A query that was fast at 10,000 rows can fail at 100 million.

## Common Mistakes

- Writing SQL before naming the output grain.
- Relying on implicit row order.
- Selecting every column for a small DTO.
- Loading all matches into PHP to test existence or count.
- Hiding tenant or authorization predicates in a distant caller.
- Treating one query as automatically better than two clear bounded queries.
- Concatenating unvalidated structure or values into SQL.
- Assuming a query plan remains stable as data and statistics change.

## Testing

Test empty results, null values, equal sort keys, boundary timestamps, duplicate parent relations, authorization filters, and maximum page sizes. Use integration tests against the database engine for SQL semantics and query shape. Add plan or performance checks for critical paths, but avoid asserting every low-level plan detail if harmless engine upgrades can change it.

## Senior Engineer Thinking

Good query design makes the result grain, correctness boundary, and cost visible. Push filtering, aggregation, uniqueness, and ordering to the database when it has the data and indexes to perform them efficiently. Keep orchestration and domain rules in PHP when they depend on application behavior or external systems. Measure the boundary instead of following slogans about “one query” or “thin repositories.”

## Exercises

1. State the grain of a customer report and rewrite a fan-out query so each customer appears once.
2. Replace a “load all rows then check” existence operation with `EXISTS`.
3. Add a deterministic tie-breaker to a feed query and explain how it affects pagination.
4. Find an N+1 path in a repository and compare a join with a bounded second query.

## Review Questions

1. What does “query grain” mean?
2. Why is `ORDER BY` required for a stable API result?
3. When can two bounded queries be better than one join?
4. Why can a function on an indexed column change performance?
5. Which failures are safe to retry, and what does a write need before retry?

## Summary

Design SQL from the required result grain, predicates, cardinality, ordering, and cost. Select only needed columns, use `EXISTS` for presence, avoid N+1 round trips, bind values, allow-list dynamic structure, and inspect plans on realistic data. A repository boundary should expose a clear result and an explicit failure policy.

## References

- [PostgreSQL: Queries](https://www.postgresql.org/docs/current/queries.html)
- [MySQL: Optimization](https://dev.mysql.com/doc/refman/8.4/en/optimization.html)
- [PHP Manual: PDO prepared statements](https://www.php.net/manual/en/pdo.prepared-statements.php)
