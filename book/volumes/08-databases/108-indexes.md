---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 108
title: Indexes
slug: indexes
status: complete
summary: ../../_ai/chapter-summaries/108-indexes-summary.md
---

# Chapter 108 — Indexes

## Why This Matters

A query can be logically correct and still make an application unusable. A request that looks up one order may scan ten million rows, sort them, and hold a database connection while PHP waits. An index gives the database another access path: it can locate likely rows without examining every row in the table.

Indexes are not free. They consume disk and memory, slow inserts and updates, and add choices to the optimizer's work. Good indexing starts with a measured access pattern and a clear correctness requirement. [Chapter 107](107-query-design.md) discusses shaping queries; this chapter explains the data structure that can make a shaped query affordable. [Chapter 111](111-explain.md) shows how to inspect the resulting plan.

## Mental Model

Think of a table as a heap of records and an index as a separately maintained, ordered directory:

```text
table rows:       [row 1] [row 2] [row 3] ... [row N]
index on email:   "ada@example.test" -> row 42
                  "lee@example.test" -> row 981
```

The index stores key values and a locator or primary-key value. A database can search the index, then fetch the matching table rows. In a common B-tree index, finding a key takes approximately `O(log N)` comparisons and returning `k` matches costs roughly `O(log N + k)` index work, plus the cost of fetching rows. A full table scan is approximately `O(N)` row visits. These are useful models, not promises about a particular engine's page reads or cache state.

An index is useful when it removes enough work to repay its maintenance cost. A column with only two values may not be selective enough for an index on a large query, while a timestamp or tenant identifier may be useful because the query also limits the time range or partition of data. The optimizer decides whether the index is cheaper for this query and this distribution of values.

## Core Concept

The most common index is a B-tree. Its keys are kept in sorted order across pages, allowing equality predicates, range predicates, and ordered retrieval. A database may also support hash, bitmap, full-text, spatial, or specialized indexes. Their availability and behavior are engine-specific; do not infer that an index type in PostgreSQL has the same syntax or trade-offs in MySQL.

Consider an application that lists recent paid orders for one customer:

```sql
CREATE INDEX orders_customer_status_created_idx
    ON orders (customer_id, status, created_at);
```

This index can help a query that constrains the leading columns and then uses the date range:

```sql
SELECT id, total_cents, created_at
FROM orders
WHERE customer_id = :customer_id
  AND status = :status
  AND created_at >= :since
ORDER BY created_at DESC
LIMIT 50;
```

Whether it also avoids a sort depends on the engine, index direction support, statistics, and the exact query. The index definition is a candidate access path, not a guarantee that the optimizer will use it.

## How It Works

For a B-tree lookup, the engine descends from a root page through branch pages to a leaf page. Leaf entries point to table rows or contain enough data to return the result. A range scan follows adjacent leaf entries. Random table fetches can dominate the cost when a predicate matches many rows, which is why an optimizer may choose a sequential scan even when an index technically matches the predicate.

Indexes are maintained as rows change. An insert adds an entry; changing an indexed column usually removes one key and adds another; deleting a row removes the entry. An index can therefore increase write amplification and create page splits or bloat. A table with ten indexes does not receive ten times the read benefit automatically, but each write must maintain the relevant structures.

A unique index also enforces a constraint. Prefer a database `UNIQUE` constraint when uniqueness is part of the data model; the database can create the supporting unique index and communicate the invariant to schema tools. A plain index can speed a lookup but cannot prevent duplicate values.

Foreign-key enforcement and indexing are related but distinct. Some engines automatically create an index for a primary key or unique constraint. They do not all automatically index every foreign-key column. Index the referencing column when joins, parent deletes, or child lookups require it, and verify the behavior in the engine you deploy.

## Minimal Example

The PHP call remains parameterized. Indexing does not make interpolating input safe:

```php
<?php

declare(strict_types=1);

function findUserByEmail(PDO $db, string $email): ?array
{
    $statement = $db->prepare(
        'SELECT id, email, display_name
         FROM users
         WHERE email = :email'
    );
    $statement->execute(['email' => $email]);

    $user = $statement->fetch(PDO::FETCH_ASSOC);

    return $user === false ? null : $user;
}
```

If `email` must be unique, define that invariant in the schema:

```sql
ALTER TABLE users
    ADD CONSTRAINT users_email_unique UNIQUE (email);
```

The constraint supplies an access path in common relational engines and, more importantly, makes concurrent duplicate inserts fail atomically. Application code that first checks `SELECT` and then inserts still has a race; [Chapter 114](114-transactions.md) and [Chapter 118](118-concurrency.md) cover those boundaries.

## Practical Example: Choosing an Index

Suppose a support dashboard runs this query frequently:

```sql
SELECT id, subject, created_at
FROM tickets
WHERE organization_id = :organization_id
  AND state = 'open'
ORDER BY created_at DESC
LIMIT 100;
```

Start with evidence: query frequency, row counts per organization, the number of open tickets, latency targets, and the current plan. A plausible index is:

```sql
CREATE INDEX tickets_org_state_created_idx
    ON tickets (organization_id, state, created_at DESC);
```

The index is most valuable if it lets the engine find a small ordered slice. If one organization owns most tickets and almost all are open, `state` contributes little filtering power. If organizations are small but the table is huge, the leading `organization_id` may be enough to reduce the work. The right answer depends on cardinality and the query mix, so test with representative data rather than a ten-row fixture.

A PHP repository should keep the query stable and pass values separately:

```php
<?php

declare(strict_types=1);

final class TicketRepository
{
    public function __construct(private PDO $db)
    {
    }

    /** @return list<array{id: int, subject: string, created_at: string}> */
    public function openForOrganization(int $organizationId): array
    {
        $query = <<<'SQL'
            SELECT id, subject, created_at
            FROM tickets
            WHERE organization_id = :organization_id
              AND state = 'open'
            ORDER BY created_at DESC
            LIMIT 100
            SQL;

        $statement = $this->db->prepare($query);
        $statement->execute(['organization_id' => $organizationId]);

        /** @var list<array{id: int, subject: string, created_at: string}> */
        return $statement->fetchAll(PDO::FETCH_ASSOC);
    }
}
```

The index does not replace a limit, and a limit does not replace an index. Together they can bound the number of rows PHP receives while the database finds the first page efficiently.

## Bad Example

This predicate often prevents a normal index on `email` from being used efficiently:

```sql
SELECT id
FROM users
WHERE LOWER(email) = LOWER(:email);
```

The function is applied to the column, so a plain `email` index may not provide the required ordering. A leading wildcard has a similar problem:

```sql
SELECT id
FROM users
WHERE email LIKE '%@example.test';
```

The exact behavior is engine-specific, but the query cannot generally seek to one known beginning of the B-tree. Do not “fix” these queries by adding indexes blindly. Consider a normalized column maintained at write time, an expression or functional index supported by the chosen engine, or a full-text/search system when the requirement is search rather than equality.

## Better Example

If email matching is case-insensitive by domain rules, make that rule explicit. One portable approach is a normalized value:

```sql
ALTER TABLE users ADD COLUMN email_normalized VARCHAR(320) NOT NULL;
CREATE UNIQUE INDEX users_email_normalized_uq
    ON users (email_normalized);
```

PHP normalizes at the boundary according to the application's documented policy, then queries the indexed value:

```php
$normalized = strtolower(trim($email));
$statement = $db->prepare(
    'SELECT id, email FROM users WHERE email_normalized = :email'
);
$statement->execute(['email' => $normalized]);
```

Lowercasing every Unicode email address is a policy decision, not a universal law. If the database has a native case-insensitive type or collation, use its documented semantics and test them. Schema-level uniqueness must use the same semantics as lookup.

## Edge Cases

- **NULL:** `NULL` is not equal to another `NULL` under SQL's three-valued logic. Unique-index treatment of multiple nulls differs by engine and configuration; verify it before using nullable uniqueness as a business rule.
- **Low selectivity:** An index on a boolean may be ignored when a value matches a large fraction of the table. It can still help a rare value or a composite access path.
- **Ordering and collation:** Text ordering and comparisons follow the column's collation. An index created under one collation may not satisfy a query requiring another.
- **Implicit casts:** Comparing different data types can force conversion or prevent an efficient seek. Bind an integer as an integer and keep joined columns compatible.
- **Expressions:** A function, arithmetic expression, or cast around a column may require an expression index or a rewritten predicate. Check the target engine's rules.
- **Statistics:** Plans depend on statistics. After a large data change, the optimizer may need its normal statistics refresh or an engine-specific analyze operation.
- **Soft deletes:** If nearly every query adds `deleted_at IS NULL`, account for that predicate in an index strategy and test the distribution. A partial or filtered index is engine-specific.

## Performance and Operations

Measure both reads and writes. A useful index reduces rows examined, sorting, random page reads, or join work for an important query. Its costs include index storage, cache pressure, write latency, maintenance time, and potentially slower bulk loads. Remove an index only after checking all consumers, including ad hoc jobs and foreign-key operations.

Use production-shaped data. A planner can correctly choose a scan for a tiny development table even though a production table needs an index. Capture query latency and rows returned, but also inspect rows examined, buffer reads, temporary files, and lock waits where the engine exposes them. [Chapter 110](110-query-plans.md) explains the optimizer's choices; [Chapter 111](111-explain.md) gives the inspection workflow.

Index creation can lock or compete with application traffic depending on the engine and DDL mode. Read the vendor's online or concurrent-index documentation before applying a migration during peak load. Build large indexes in a controlled migration, monitor progress and disk space, and have a rollback or drop plan that does not remove a required constraint.

## Testing

Test the invariant and the access pattern separately. A database integration test should prove that duplicate normalized emails fail with the expected constraint error. A plan-oriented test can run `EXPLAIN` against a representative fixture, but avoid asserting an exact plan node across engine versions unless the plan is a supported contract. Assert a high-level property such as an intended index being considered, and keep performance tests on realistic volumes.

In application tests, verify the repository's parameter types, ordering, limit, and behavior for no rows. A unit test that mocks `PDO` can check the call shape, but only a real database can validate collation, null behavior, constraint races, and index access.

## Common Mistakes

- Adding an index for every column mentioned in a query without measuring cardinality or write cost.
- Treating an index as a guarantee that a query will use it.
- Using a plain index where the schema requires uniqueness.
- Assuming every foreign key receives an index automatically.
- Hiding the indexed value behind a function, cast, or incompatible collation.
- Testing plans on a tiny fixture and extrapolating to production.
- Creating overlapping indexes and never checking their storage and write cost.
- Adding an index migration without checking DDL locking and available disk space.
- Interpolating user input because the query has an index; parameterization remains required.

## Senior Engineer Thinking

An index is a materialized access path for a workload, not a decoration on a table. State the query, the expected result size, the data distribution, and the write rate before choosing one. Then validate the proposal with the engine's plan and production-like measurements.

The most durable design connects the database invariant to the index where possible: uniqueness belongs in a constraint, tenant isolation belongs in every relevant predicate, and a pagination order needs a stable tie-breaker. Keep application code parameterized and observable so a plan change can be traced to a schema, data, or query change.

## Exercises

1. Create a `users` table with 100,000 rows and compare an email lookup before and after a unique constraint. Record latency and the plan; explain why the result changes.
2. Design indexes for the support-ticket query in this chapter under two distributions: one organization owns 80% of tickets, and organizations are evenly sized. Explain which assumptions change your choice.
3. Test `NULL` and duplicate behavior for a nullable unique column in your target database. Write the business rule that the schema should enforce.
4. Find a query in a PHP application that applies a function to an indexed column. Rewrite it using a normalized or engine-supported expression index, then verify the plan on representative data.

## Review Questions

1. What work does a B-tree index avoid, and what work does it add to writes?
2. Why can an optimizer choose a table scan even when a matching index exists?
3. How does selectivity affect the value of an index?
4. Why is a unique constraint preferable to an application-side “check then insert” for uniqueness?
5. What problems can functions, casts, collations, and leading wildcards cause for an index lookup?
6. Which index costs should be measured besides query latency?
7. Why must a plan be tested with production-shaped data?

## Summary

Indexes provide maintained access paths that can change a query from scanning `N` rows to seeking through an ordered structure and reading the relevant matches. They also consume storage, memory, write work, and maintenance time. Choose them from measured query patterns, constraints, data distribution, and operational limits; then verify the optimizer's actual choice. Parameterized PHP, database-enforced invariants, representative data, and plan inspection turn an index proposal into an engineering decision.

## References

- [PostgreSQL documentation: indexes](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL documentation: multicolumn indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [MySQL 8.4 Reference Manual: optimization and indexes](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)
- [MySQL 8.4 Reference Manual: multiple-column indexes](https://dev.mysql.com/doc/refman/8.4/en/multiple-column-indexes.html)
- [PHP Manual: PDO prepared statements](https://www.php.net/manual/en/pdo.prepared-statements.php)
