---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 122
title: Query Builders
slug: query-builders
status: complete
summary: ../../_ai/chapter-summaries/122-query-builders-summary.md
---

# Chapter 122 — Query Builders

## Why This Matters

Application filters are dynamic: a report may accept a status, date range, tenant, search term, sort order, and page cursor. Concatenating pieces of SQL by hand is difficult to review and easy to make unsafe. A query builder gives code a structured way to add clauses and parameters while preserving a composable API.

A builder is not a security boundary by itself. It can parameterize values, but identifiers, sort directions, function names, and raw expressions usually cannot be bound as values. A reliable design separates trusted SQL structure from untrusted data and makes the final query inspectable.

## Values and Identifiers Are Different

This is safe in principle because the value is a parameter:

```php
$where[] = 'status = :status';
$parameters['status'] = $status;
```

This is unsafe because a column name is SQL syntax, not a value:

```php
$orderBy = $_GET['sort'];
$sql .= ' ORDER BY ' . $orderBy;
```

Use a finite map for identifiers and directions:

```php
<?php

declare(strict_types=1);

$sortColumns = [
    'newest' => 'created_at',
    'name' => 'display_name',
];

$sortKey = (string) ($_GET['sort'] ?? 'newest');
$sortColumn = $sortColumns[$sortKey] ?? $sortColumns['newest'];
$direction = strtolower((string) ($_GET['direction'] ?? 'desc')) === 'asc'
    ? 'ASC'
    : 'DESC';

$sql = "SELECT id, display_name, created_at
        FROM users
        ORDER BY {$sortColumn} {$direction}, id {$direction}";
```

The interpolated fragments come only from constants selected by the map. The tie-breaker makes pagination deterministic; see [Chapter 119](119-pagination.md). A regular expression check is not a substitute for an allowlist when the valid set is small.

## A Composable Filter

A small application-level filter object can build SQL and parameters without coupling the controller to string assembly:

```php
<?php

declare(strict_types=1);

final readonly class UserFilter
{
    public function __construct(
        public ?string $status = null,
        public ?string $search = null,
        public ?string $createdAfter = null,
    ) {
    }
}

/** @return array{sql: string, parameters: array<string, mixed>} */
function buildUserQuery(UserFilter $filter): array
{
    $where = [];
    $parameters = [];

    if ($filter->status !== null) {
        $where[] = 'status = :status';
        $parameters['status'] = $filter->status;
    }

    if ($filter->search !== null) {
        $where[] = 'display_name ILIKE :search';
        $parameters['search'] = '%' . $filter->search . '%';
    }

    if ($filter->createdAfter !== null) {
        $where[] = 'created_at >= :created_after';
        $parameters['created_after'] = $filter->createdAfter;
    }

    $sql = 'SELECT id, display_name, created_at FROM users';
    if ($where !== []) {
        $sql .= ' WHERE ' . implode(' AND ', $where);
    }
    $sql .= ' ORDER BY created_at DESC, id DESC';

    return ['sql' => $sql, 'parameters' => $parameters];
}
```

`ILIKE` is PostgreSQL-specific. A portable builder should either expose a database dialect or use the engine's documented case-insensitive comparison. A placeholder cannot represent a list of arbitrary values in every driver:

```sql
WHERE id IN (:ids) -- Usually one scalar parameter, not a list.
```

Expand a validated list into named placeholders, use a driver-specific array parameter, or insert into a temporary/staging table according to the database and size of the input. Never serialize a list into one SQL string and concatenate it.

## Library Builders and Raw SQL

Libraries such as Doctrine DBAL expose a builder that handles clause composition and parameter binding:

```php
$query = $connection->createQueryBuilder();
$query
    ->select('u.id', 'u.display_name')
    ->from('users', 'u')
    ->andWhere('u.status = :status')
    ->setParameter('status', 'active')
    ->orderBy('u.created_at', 'DESC')
    ->addOrderBy('u.id', 'DESC')
    ->setMaxResults(50);

$rows = $query->executeQuery()->fetchAllAssociative();
```

The exact execution and parameter-type APIs vary by DBAL version. A builder still permits raw predicates and expressions, and a caller can still select an invalid table or construct a costly join. Review generated SQL and use the database's plan tools. A builder improves composition; it does not prove correctness or performance.

Raw SQL is appropriate for a window function, common table expression, vendor-specific lock, or bulk update. Put it behind a named method, bind values, and keep dialect-specific code explicit. Avoid wrapping every query in a generic abstraction that erases the distinction between a count, a page, and an aggregate.

## Builder State and Reuse

Treat a builder as mutable unless its library explicitly promises immutability. Reusing one instance across two queries can leak a `WHERE`, join, parameter, or limit from the first query into the second. Create a fresh builder per operation or encapsulate a base specification in a function that returns a new instance.

If a query is assembled by optional filters, test each combination that matters. The empty filter should produce valid SQL; optional joins should not multiply rows accidentally; a filter on a left-joined table belongs in the `ON` clause or `WHERE` clause according to the intended null behavior. [Chapter 112](112-joins.md) explains why this placement changes results.

## Search, Lists, and Pagination

A search endpoint often combines dynamic filters with a stable cursor:

```sql
SELECT id, display_name, created_at
FROM users
WHERE status = :status
  AND (created_at, id) < (:cursor_created_at, :cursor_id)
ORDER BY created_at DESC, id DESC
LIMIT :limit;
```

The builder should add the cursor predicate only when a cursor is present, validate the cursor before it reaches the builder, and cap the limit. Search patterns such as `'%term%'` may not use an ordinary B-tree index; choose full-text or specialized indexing according to the engine and language requirements. A builder cannot compensate for a query shape that has no suitable access path.

## Query Builders and Types

PHP values do not always carry the database type the query needs. A string `'0'`, a date object, a UUID, and a binary identifier may need explicit binding or conversion. Use the library's type system when available, and make timezone, precision, and null behavior explicit at the boundary.

For optional values, distinguish “no filter” from “filter for SQL NULL.” Omitting a clause and emitting `column IS NULL` are different operations. Likewise, an empty list may mean “return no rows” or “do not apply this filter”; define the application semantics before generating SQL.

## Security and Observability

Log a query fingerprint and timing, not raw secrets or unrestricted parameter values. Correlate slow query logs with the application operation and tenant or request class without leaking personal data. Keep a way to inspect final SQL and bound types in tests or local diagnostics.

Validate authorization before building a query, and include tenant or ownership predicates in the repository method that owns the data boundary. A query builder makes it easy to add a filter and equally easy for one call path to forget it. Prefer APIs that require a scope object or repository method rather than passing arbitrary conditions from a controller.

## Common Mistakes

- Assuming a query builder automatically prevents injection in identifiers or raw expressions.
- Reusing a mutable builder and leaking clauses between requests or queries.
- Treating one placeholder as a portable list parameter.
- Letting controllers concatenate arbitrary sort columns.
- Hiding vendor-specific SQL behind a false portability promise.
- Skipping plan inspection because the query was built by a library.
- Applying a filter on a left join in the wrong clause and changing its meaning.
- Treating an empty list, omitted filter, and `NULL` filter as the same case.

## Senior Engineer Thinking

Use a query builder for controlled variability: optional predicates, joins, projections, and limits. Keep the grammar of SQL under application control, keep values parameterized, and preserve a path to readable SQL when the query is complex. The right abstraction makes dangerous choices difficult while leaving performance-relevant choices visible.

## Exercises

1. Add safe sort selection and a deterministic tie-breaker to a filtered user query.
2. Implement expansion for an `IN` list and define the behavior for an empty list.
3. Reproduce a left-join bug caused by putting a related-table filter in `WHERE` instead of `ON`.
4. Compare a library builder and raw SQL for a reporting query with a common table expression. Record what remains database-specific.

## Review Questions

1. Why can values be bound but identifiers usually cannot?
2. What state can accidentally leak when a mutable builder is reused?
3. Why is a query builder not a guarantee of good performance?
4. How should a builder represent an empty list and an omitted filter?
5. When is raw SQL the clearer engineering choice?

## Summary

Query builders structure dynamic SQL, but they do not eliminate SQL semantics, injection risks in identifiers, or query-plan costs. Keep values parameterized, allowlist structural fragments, create builders with clear ownership, represent optional and null filters precisely, and inspect generated SQL. Use raw SQL openly when database-specific or set-oriented features make it the better boundary.

## References

- [Doctrine DBAL: Query Builder](https://www.doctrine-project.org/projects/doctrine-dbal/en/current/reference/query-builder.html)
- [PHP Manual: PDO prepared statements](https://www.php.net/manual/en/pdo.prepared-statements.php)
- [PostgreSQL: Value expressions](https://www.postgresql.org/docs/current/sql-expressions.html)
- [OWASP: SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
