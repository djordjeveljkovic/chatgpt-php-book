---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 104
title: SQL for PHP Developers
slug: sql-for-php-developers
status: complete
summary: ../../_ai/chapter-summaries/104-sql-for-php-developers-summary.md
---

# Chapter 104 — SQL for PHP Developers

## Why This Matters

Most PHP applications spend more time waiting for a database than executing PHP instructions. A controller can be perfectly typed and still be slow because it asks for rows one at a time, sorts a large result in memory, or leaves an important invariant to a race-prone application check. SQL is therefore part of the application's algorithm, data model, and correctness boundary.

The useful question is not “how do I turn this array into a query?” It is “which rows does the requirement describe, which constraints must always hold, and which work should the database perform?” This chapter builds that model before [Chapter 105](105-pdo.md) crosses the PHP/database boundary.

## Mental Model

A relational database stores relations: sets of rows with named attributes. A table is a durable relation with a schema. A query describes a new relation derived from existing relations. An execution engine chooses a physical strategy—an index lookup, a scan, a join algorithm, a sort, or an aggregate—to produce that result.

Think in three layers:

~~~text
business requirement
        ↓
relational expression: rows, predicates, grouping, ordering
        ↓
physical work: scans, indexes, joins, sorting, memory, locks
~~~

SQL is declarative: the application states the result it needs, while the database chooses a plan. This does not mean cost is invisible. A query that returns ten rows may still read millions of rows before it can know which ten qualify. [Chapter 110](110-query-plans.md) and [Chapter 111](111-explain.md) examine that choice in detail.

## Tables, Keys, and Constraints

Model facts separately from relationships. A small order schema might look like this (the exact identity and timestamp types vary by engine):

~~~sql
CREATE TABLE customers (
    id INTEGER PRIMARY KEY,
    email VARCHAR(320) NOT NULL UNIQUE,
    display_name VARCHAR(200) NOT NULL,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    status VARCHAR(20) NOT NULL,
    total_minor INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL,
    CONSTRAINT orders_customer_fk
        FOREIGN KEY (customer_id) REFERENCES customers (id),
    CONSTRAINT orders_total_non_negative
        CHECK (total_minor >= 0)
);
~~~

The primary key identifies one row. A foreign key says that a referenced customer must exist, subject to the engine's foreign-key settings and transaction behavior. NOT NULL says that the fact is required. UNIQUE protects a key such as an email address. A check constraint protects a local predicate. These are executable documentation and concurrency protection: two requests cannot both “pass” an application-only uniqueness check and commit the same unique value.

A surrogate integer key is convenient, but it does not make a domain key unnecessary. customers.id identifies the row internally; customers.email may still be a business identity and needs a unique constraint if duplicates are invalid. Decide whether email comparison is case-sensitive and how normalization works before adding that constraint. Collation and database-specific text comparison can change the answer.

## Data Manipulation Is Set-Based

SQL statements operate on sets of rows. A statement can affect zero, one, or many rows, and the affected-row count is part of its result. Prefer one statement that expresses the set operation over a PHP loop that performs one statement per row.

~~~sql
UPDATE orders
SET status = 'expired'
WHERE status = 'pending'
  AND created_at < :cutoff;
~~~

The predicate is a contract. It makes the operation safe to retry if setting pending rows to expired is idempotent. The application should inspect the affected count when the requirement expects a particular row to exist. “No rows changed” might mean a missing order, a stale state, or an already-completed operation; the code should distinguish these cases when they matter.

The common query pipeline is easiest to reason about in this logical order:

~~~text
FROM / JOIN     choose source rows
WHERE           filter individual rows
GROUP BY        form groups
HAVING          filter groups
SELECT          project columns and expressions
ORDER BY        order the result
LIMIT/OFFSET    restrict the returned window
~~~

The written SQL conventionally places SELECT first, but WHERE cannot use an aggregate result that is created later. This is why a group predicate belongs in HAVING, while a row predicate belongs in WHERE.

## NULL Is a Third State

NULL represents an absent or unknown value; it is not the empty string and it is not zero. Comparisons involving NULL produce an unknown truth value, so this query does not find missing timestamps:

~~~sql
SELECT id
FROM orders
WHERE shipped_at = NULL;
~~~

Use IS NULL or IS NOT NULL:

~~~sql
SELECT id
FROM orders
WHERE shipped_at IS NULL;
~~~

WHERE keeps only rows for which its predicate is true. Both false and unknown are excluded. COALESCE() can provide a display fallback, but using it to hide a modeling ambiguity can make reporting incorrect. Decide whether “not shipped,” “not applicable,” and “unknown due to import failure” are one state or three states, then model that decision explicitly.

## A Small PHP-to-SQL Boundary

Represent query parameters as values, not as a concatenated SQL string. The mechanics belong to [Chapter 106](106-prepared-statements.md); this example shows the shape of a set-based read:

~~~php
<?php

declare(strict_types=1);

/** @return list<array{id: int, status: string, total_minor: int}> */
function findRecentOrders(PDO $db, int $customerId, DateTimeImmutable $since): array
{
    $statement = $db->prepare(
        <<<'SQL'
        SELECT id, status, total_minor
        FROM orders
        WHERE customer_id = :customer_id
          AND created_at >= :since
        ORDER BY created_at DESC, id DESC
        SQL
    );

    $statement->execute([
        'customer_id' => $customerId,
        'since' => $since->format('Y-m-d H:i:s'),
    ]);

    /** @var list<array{id: int, status: string, total_minor: int}> */
    return $statement->fetchAll(PDO::FETCH_ASSOC);
}
~~~

The second ordering key makes equal timestamps deterministic. Without it, the database is free to return tied rows in different orders across executions or plans. The PHPDoc describes the intended shape; a production boundary should also account for driver-dependent scalar conversions and validate external data where the type matters.

## A Practical Reporting Query

Suppose the product asks for each customer’s number and total value of completed orders in the last 30 days. The requirement contains a time cutoff, a status predicate, grouping, and an output order:

~~~sql
SELECT
    c.id,
    c.email,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total_minor), 0) AS total_minor
FROM customers AS c
LEFT JOIN orders AS o
    ON o.customer_id = c.id
   AND o.status = 'paid'
   AND o.created_at >= :since
GROUP BY c.id, c.email
ORDER BY total_minor DESC, c.id ASC;
~~~

The LEFT JOIN preserves customers with no matching orders. Putting the order predicates in the ON clause preserves that meaning. Moving them to WHERE would reject the NULL-extended rows and effectively turn the result into an inner join. Join behavior is developed further in [Chapter 112](112-joins.md).

COUNT(o.id) counts matched orders; COUNT(*) counts the preserved customer row even when no order matched. SUM over no matched values is NULL, so COALESCE gives the report a numeric zero. A report query should state its unit: total_minor avoids floating-point money arithmetic, but the currency still needs a domain rule if one customer can have multiple currencies.

## Transactions and Invariants

A transaction groups statements into one unit of atomic work. The exact isolation and locking guarantees depend on the engine and configuration, but the application should define which facts must hold at commit.

~~~sql
BEGIN;

INSERT INTO orders (customer_id, status, total_minor, created_at)
VALUES (:customer_id, 'pending', :total_minor, :created_at);

UPDATE inventory
SET available = available - :quantity
WHERE sku = :sku
  AND available >= :quantity;

-- The application checks that exactly one inventory row changed.
COMMIT;
~~~

If the inventory update affects zero rows, the application must roll back the transaction. Checking availability in PHP first and updating later creates a race: two requests can observe the same stock. A conditional update, a database constraint, or an appropriate lock puts the invariant close to the write. [Chapter 114](114-transactions.md) covers transaction boundaries, and [Chapter 115](115-isolation.md) explains what concurrent transactions can observe.

## Bad Example: Treating SQL as String Formatting

This code mixes untrusted data with program syntax:

~~~php
$email = $_POST['email'] ?? '';
$sql = "SELECT id FROM customers WHERE email = '$email'";
$row = $db->query($sql)->fetch();
~~~

Quotes are not an input validation strategy. An apostrophe can break the literal, and crafted input can change the statement. The fix is a prepared statement, but it also requires choosing a sensible input policy and handling the case where no row exists. [Chapter 144](../../volumes/10-security/144-sql-injection.md) treats injection as a security boundary; the next two chapters show the database API.

## Performance: Rows, Work, and Data Movement

Query cost is not just the number of returned rows. A query can read a large table, evaluate a predicate, sort candidate rows, build a hash table for a join, and then return a small result. Measure and reason about:

- rows examined and rows returned;
- index selectivity and whether a predicate is usable by an index;
- join cardinality and intermediate result size;
- sort and aggregate memory;
- network bytes crossing from the database to PHP;
- PHP memory used by fetchAll() and hydrated domain objects.

Selecting only required columns reduces transfer and hydration cost. A streaming or batched fetch can bound PHP memory for a large result, but it does not automatically make the database query cheap. An index can reduce search work, yet every index adds write and storage cost. [Chapter 108](108-indexes.md) develops that trade-off.

## Security and Correctness

Use least-privilege database credentials. A read-only report process should not receive schema-altering permissions. Keep secrets out of SQL logs and exception messages. Treat database error text as diagnostic data, not as a response to expose to an end user.

Validate values at the domain boundary, then bind them as values. A prepared parameter protects SQL syntax; it does not prove that an amount is non-negative, that a date range is sensible, or that a caller may access a customer. Authorization predicates must be part of the query or be enforced by a trusted data boundary; loading a row and checking ownership in a later, unrelated step can create a time-of-check/time-of-use gap.

## Testing

Test constraints and query semantics against the same database engine used in production where possible. SQLite is useful for fast unit-level examples, but differences in type coercion, locking, collations, JSON, date functions, and ALTER TABLE behavior can make it an unsafe substitute for every integration test.

A focused test for a customer report should include a customer with no orders, a customer with one qualifying order, a non-qualifying status, equal timestamps, and multiple currencies if the schema permits them. Assert rows and columns rather than the database’s incidental physical order; when order is part of the contract, include an explicit tie-breaker and assert it.

## Common Mistakes

- treating NULL as an ordinary value;
- omitting a deterministic tie-breaker from paged or user-visible ordering;
- using COUNT(*) when an outer join requires counting matches;
- doing one query per item in a PHP loop;
- trusting a UNIQUE-looking application check without a database constraint;
- selecting every column and hydrating objects when a small projection is enough;
- using a database-specific feature without documenting the portability boundary;
- assuming a query that returns few rows did little work.

## Senior Engineer Thinking

Translate a feature request into facts, identities, constraints, and read models before writing PHP. Ask which invariant must hold after a crash or duplicate request, which rows may be absent, and which order is meaningful. Then inspect the physical cost: cardinality, indexes, sorting, network transfer, and memory. SQL is a language for expressing the requirement, while the schema and transaction boundary determine which incorrect states are impossible.

## Exercises

1. Design tables for a library checkout system. Identify primary keys, business keys, foreign keys, and at least three constraints.
2. Rewrite a PHP loop that loads every customer and then counts its orders as one grouped query. Explain how the outer-join case changes COUNT(*) versus COUNT(order_id).
3. Create rows containing NULL, an empty string, and zero. Write predicates that distinguish the three states and explain unknown truth values.
4. For an inventory decrement, list the race in a read-then-write implementation and propose a conditional SQL update whose affected-row count indicates success.
5. For a query returning the ten newest events, specify a deterministic ordering and explain what happens when timestamps tie.

## Review Questions

1. What does a relational query describe, and who chooses the physical execution strategy?
2. Why are primary keys and business uniqueness constraints separate concepts?
3. Why does WHERE value = NULL match no missing values?
4. When is HAVING appropriate instead of WHERE?
5. Why can moving an outer-join predicate from ON to WHERE change the result set?
6. Why is an application-only uniqueness check unsafe under concurrency?
7. What costs can exist between reading a table and returning ten rows?
8. Why is a deterministic tie-breaker important for pagination and tests?

## Summary

SQL is the executable expression of a relational model. Keys and constraints protect identity and invariants; set-based statements express work over many rows; NULL requires three-valued logic; and grouping, joins, ordering, and aggregates have precise semantics. A PHP application should send values through a safe database boundary, keep correctness constraints near the data, and measure physical work rather than judging a query by its result size. PDO and prepared statements provide that boundary in the next chapters.

## References

- [PHP Manual: PDO](https://www.php.net/manual/en/book.pdo.php)
- [PostgreSQL documentation: SQL language](https://www.postgresql.org/docs/current/sql.html)
- [PostgreSQL documentation: constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [SQLite documentation: SQL language](https://www.sqlite.org/lang.html)
