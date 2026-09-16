---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 112
title: Joins
slug: joins
status: complete
summary: ../../_ai/chapter-summaries/112-joins-summary.md
---

# Chapter 112 — Joins

## Why This Matters

Useful data is usually split across tables. A customer is stored separately from orders, and an order is stored separately from its line items because those relationships have different lifecycles and cardinalities. A join reconstructs a useful view from those relations. A correct join preserves the intended meaning; an incorrect join silently duplicates revenue, drops customers, or turns an optional relationship into a required one.

The database should perform the relational work close to the indexed data. PHP should receive the smallest result that the application actually needs. This chapter uses a small schema throughout:

```sql
CREATE TABLE customers (
    id          BIGINT PRIMARY KEY,
    email       VARCHAR(320) NOT NULL UNIQUE,
    name        VARCHAR(200) NOT NULL
);

CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers (id),
    status      VARCHAR(20) NOT NULL,
    total_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
    created_at  TIMESTAMP NOT NULL
);

CREATE TABLE order_items (
    order_id   BIGINT NOT NULL REFERENCES orders (id),
    product_id BIGINT NOT NULL,
    quantity   INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(12, 2) NOT NULL CHECK (unit_price >= 0),
    PRIMARY KEY (order_id, product_id)
);
```

## Mental Model

A join combines rows according to a predicate. Start by asking two questions:

1. What does one output row represent?
2. How many output rows can one input row produce?

Joining `customers` to `orders` produces one row per order, so a customer with three orders appears three times. Joining that result to `order_items` produces one row per item. This multiplication is correct for an order-detail report, but wrong if you then count customers or sum an order total without controlling the grain.

`ON` describes how rows match. `WHERE` filters the result after the join. That distinction is especially important for outer joins.

## Core Join Types

An inner join keeps only rows with a match on both sides:

```sql
SELECT c.id, c.name, o.id AS order_id, o.created_at
FROM customers AS c
JOIN orders AS o ON o.customer_id = c.id
WHERE o.status = 'paid';
```

The result contains customers with paid orders. Customers with no paid order do not appear. `JOIN` means `INNER JOIN` unless a database-specific extension changes the context.

A left join keeps every row from its left input and fills right-side columns with `NULL` when no match exists:

```sql
SELECT c.id, c.name, o.id AS order_id
FROM customers AS c
LEFT JOIN orders AS o
  ON o.customer_id = c.id
 AND o.status = 'paid';
```

The status predicate belongs in `ON`: a customer remains in the result even when no paid order exists. Moving it to `WHERE` removes the null-extended rows and makes the query behave like an inner join:

```sql
-- This excludes customers without a paid order.
SELECT c.id, c.name, o.id AS order_id
FROM customers AS c
LEFT JOIN orders AS o ON o.customer_id = c.id
WHERE o.status = 'paid';
```

A right join is the same idea with the preserved table on the right; using a left join after choosing the driving table is usually easier to read. A full outer join preserves unmatched rows from both sides. PostgreSQL supports `FULL OUTER JOIN`; check your database before using it in portable code. A cross join forms every pair, so its row count is `|A| × |B|`; use it only when that Cartesian product is intentional.

A self-join joins a table to itself, such as finding pairs of customers with the same domain. Give each instance a different alias. For existence questions, `EXISTS` is often clearer and safer than joining:

```sql
SELECT c.id, c.email
FROM customers AS c
WHERE EXISTS (
    SELECT 1
    FROM orders AS o
    WHERE o.customer_id = c.id
      AND o.status = 'paid'
);
```

This is a semijoin: an order match decides whether the customer qualifies, but it does not duplicate the customer once per matching order. `NOT EXISTS` expresses an anti-join, such as customers with no paid order.

## Join Keys and Cardinality

Join on a key that represents the relationship. `orders.customer_id` references `customers.id`; it should not be matched on names or email addresses that can change. Foreign keys protect referential integrity, while indexes make the lookup practical. An index beginning with `orders.customer_id` supports the common customer-to-order lookup; a composite index such as `(customer_id, status)` can help when both columns are selective predicates.

Estimate the grain before writing the select list. If an order has five items, this query returns five rows for that order:

```sql
SELECT o.id, o.customer_id, i.product_id, i.quantity
FROM orders AS o
JOIN order_items AS i ON i.order_id = o.id;
```

Do not use `DISTINCT` as a reflexive repair for an unintended one-to-many join. It can hide a modeling error and force a sort or hash operation. Decide whether the report needs order rows, item rows, or an aggregate, and write that intention explicitly.

## A Practical PHP Example

PDO returns each joined row; it does not infer an object graph. Fetch a flat report when that is what the caller needs:

```php
<?php
declare(strict_types=1);

$pdo = new PDO($dsn, $username, $password, [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
]);

$sql = <<<'SQL'
SELECT o.id AS order_id,
       c.email,
       o.created_at,
       i.product_id,
       i.quantity,
       i.unit_price
FROM orders AS o
JOIN customers AS c ON c.id = o.customer_id
JOIN order_items AS i ON i.order_id = o.id
WHERE o.id = :order_id
ORDER BY i.product_id
SQL;

$stmt = $pdo->prepare($sql);
$stmt->execute(['order_id' => $orderId]);
$items = $stmt->fetchAll();
```

This query has one row per item and gives the application an unambiguous shape. If a response needs nested order data, group these rows in PHP only after the database has selected and filtered them. For a large result, iterate with `fetch()` rather than retaining every row in memory.

## Bad Example and Better Design

A common N+1 design loads customers and then queries orders once per customer. Its application-level query count grows with the number of customers and adds network round trips:

```php
foreach ($customers as $customer) {
    $stmt = $pdo->prepare('SELECT id FROM orders WHERE customer_id = :id');
    $stmt->execute(['id' => $customer['id']]);
    $customer['orders'] = $stmt->fetchAll();
}
```

One join or two deliberately batched queries can express the same operation. A join is appropriate when the result has a stable relational shape; separate queries may be better when two large collections would create a huge fan-out. Measure both with `EXPLAIN` and use the result grain as the deciding constraint.

## Edge Cases and Performance

`NULL` does not equal `NULL` in ordinary SQL comparisons. A nullable join key therefore does not match another null key; use `IS NULL` when testing for null explicitly. Beware of functions or casts applied to indexed join columns, because they may prevent an index-friendly access path. Matching different data types can also force a cast and produce poor plans or incorrect comparisons.

A join may be logically correct but expensive. The optimizer chooses nested-loop, hash, or merge strategies depending on statistics, indexes, and estimated cardinality. Keep statistics current, select only needed columns, and inspect a representative plan with `EXPLAIN` as described in [Chapter 111 — EXPLAIN](111-explain.md). A plan estimate that is far from the actual row count often indicates stale statistics, skew, or a missing correlation in the model.

## Security and Failure Modes

Prepared statements protect values but do not make identifiers safe. Table names, sort columns, and join direction cannot be bound as parameters; choose them from a server-side allowlist. A join can also expose data across tenants if the tenant predicate is omitted. Put tenant scope into every relevant relationship or enforce it through database design and tested query helpers.

A query timeout or lost connection can occur after the server has begun work. Treat the operation as failed and do not assume that a partially returned result is complete. For a read, retry only when the operation and connection policy allow it; keep retries bounded and observable.

## Testing and Review

Integration tests should cover a customer with no orders, one order with multiple items, a missing optional relation, null values, and duplicate-looking display names. Assert row count and meaning, not only that the query executes. Use a plan check for high-traffic queries, but avoid brittle tests that require one exact plan across database versions.

## Senior Engineer Thinking

Before adding a join, state its input and output grain. Then verify the relationship is represented by a key, that the expected cardinality is enforced by constraints, and that an index supports the access pattern. A senior review asks whether the query is answering an existence question, a detail question, or an aggregation question; `EXISTS`, a join, and a grouped query have different semantics even when their first result looks similar.

## Exercises

1. Write a query that returns every customer and the timestamp of their latest paid order, including customers with none.
2. Rewrite a query that joins orders to items and counts customers so that each customer is counted once.
3. Use `NOT EXISTS` to find customers who have never placed an order, then compare its plan with a left anti-join on your database.

## Review Questions

- What is the output grain of a join between orders and order_items?
- Why can placing a right-table predicate in `WHERE` change a left join into an inner join?
- When is `EXISTS` preferable to a join?
- Which constraints and indexes make a foreign-key join reliable and fast?

## Summary

Joins reconstruct related data, but every join changes row cardinality. Choose the preserved side deliberately, keep relationship predicates in `ON` for outer joins, use `EXISTS` for existence, and let keys, constraints, indexes, and plans support the intended grain. PDO can fetch the resulting rows safely; it cannot repair an ambiguous query design.

## References

- [PostgreSQL: Table Expressions and Joins](https://www.postgresql.org/docs/current/queries-table-expressions.html)
- [PostgreSQL: EXISTS and Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html)
- [PHP manual: PDOStatement::fetch](https://www.php.net/manual/en/pdostatement.fetch.php)
