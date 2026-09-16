---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 113
title: Aggregation
slug: aggregation
status: complete
summary: ../../_ai/chapter-summaries/113-aggregation-summary.md
---

# Chapter 113 — Aggregation

## Why This Matters

A dashboard rarely needs every row. It needs orders per day, revenue per customer, or the number of failed jobs in a window. Aggregation turns many rows into a smaller result, and the database can scan, filter, group, and use indexes without sending raw data through PHP. The hard part is defining exactly what is being counted and at which grain.

Aggregation is also where an innocent join most often creates a believable but wrong number. Establish the unit of one output row before writing `SUM`, `COUNT`, or `AVG`.

## Mental Model

`GROUP BY` partitions input rows into groups. Each selected expression must either identify the group or be reduced by an aggregate function. `WHERE` filters input rows before grouping; `HAVING` filters groups after aggregation.

For example, this produces one row per customer with at least one paid order:

```sql
SELECT o.customer_id,
       COUNT(*) AS paid_order_count,
       SUM(o.total_amount) AS paid_amount
FROM orders AS o
WHERE o.status = 'paid'
GROUP BY o.customer_id
HAVING SUM(o.total_amount) > 1000;
```

`COUNT(*)` counts rows. `COUNT(column)` counts only non-null values. `COUNT(DISTINCT column)` counts distinct non-null values and may require additional memory or sorting. Make the choice explicit rather than assuming they are interchangeable.

## Grouping Correctly

A group should represent a business question: one customer, one day, one tenant and status, or one product. Use a half-open time interval for a day in a known timezone, and decide whether the stored timestamp is UTC before grouping it. Database-specific date functions can change index use; filtering by a range is often more index-friendly than applying a function to every timestamp.

```sql
SELECT CAST(o.created_at AS DATE) AS order_day,
       COUNT(*) AS order_count,
       SUM(o.total_amount) AS gross_amount
FROM orders AS o
WHERE o.created_at >= :from_utc
  AND o.created_at < :to_utc
  AND o.status = 'paid'
GROUP BY CAST(o.created_at AS DATE)
ORDER BY order_day;
```

If the report needs a local business day, convert the boundaries in application code or use a database timezone expression with tests for daylight-saving transitions. Do not silently mix server timezone, PHP timezone, and user timezone.

Conditional aggregation can produce several measures in one pass. PostgreSQL and several other databases support `FILTER`; `CASE` is more portable:

```sql
SELECT COUNT(*) AS all_orders,
       SUM(CASE WHEN status = 'paid' THEN 1 ELSE 0 END) AS paid_orders,
       SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled_orders
FROM orders
WHERE created_at >= :from_utc AND created_at < :to_utc;
```

`SUM` over no rows may be `NULL`, while `COUNT` returns zero. Use `COALESCE` when the API contract requires a numeric zero:

```sql
SELECT COALESCE(SUM(total_amount), 0) AS amount
FROM orders
WHERE customer_id = :customer_id AND status = 'paid';
```

## Joins and Fan-Out

Joining two one-to-many relations before aggregating can multiply facts. Suppose an order has three items and two shipments. Joining both tables creates six intermediate rows, so a naive sum of the order total counts it six times. Aggregate each independent child relation first, or aggregate the fact at its natural grain:

```sql
WITH item_totals AS (
    SELECT order_id, SUM(quantity * unit_price) AS item_amount
    FROM order_items
    GROUP BY order_id
), shipment_counts AS (
    SELECT order_id, COUNT(*) AS shipment_count
    FROM shipments
    GROUP BY order_id
)
SELECT o.id,
       COALESCE(i.item_amount, 0) AS item_amount,
       COALESCE(s.shipment_count, 0) AS shipment_count
FROM orders AS o
LEFT JOIN item_totals AS i ON i.order_id = o.id
LEFT JOIN shipment_counts AS s ON s.order_id = o.id;
```

`SUM(DISTINCT total_amount)` is not a general repair: two legitimate orders can have the same amount. Fix the relation or the grain instead.

## Aggregation in PHP

A database result is still untrusted input at the application boundary. Bind parameters and map numeric fields deliberately. PDO drivers may return decimal values as strings; that is useful for preserving precision, but PHP floating-point arithmetic is not appropriate for exact currency totals. Keep money as integer minor units where possible, or use a decimal library and document rounding rules.

```php
<?php
declare(strict_types=1);

$sql = <<<'SQL'
SELECT o.customer_id,
       COUNT(*) AS order_count,
       COALESCE(SUM(o.total_amount), 0) AS amount
FROM orders AS o
WHERE o.status = 'paid'
  AND o.created_at >= :from_utc
  AND o.created_at < :to_utc
GROUP BY o.customer_id
ORDER BY amount DESC
SQL;

$stmt = $pdo->prepare($sql);
$stmt->execute(['from_utc' => $fromUtc, 'to_utc' => $toUtc]);

foreach ($stmt as $row) {
    $customerId = (int) $row['customer_id'];
    $orderCount = (int) $row['order_count'];
    $amountText = (string) $row['amount']; // Keep DECIMAL exact at this boundary.
    // Serialize amountText according to the API's decimal contract.
}
```

Do not fetch millions of detail rows just to count them in PHP. PHP-side aggregation is reasonable for a small already-fetched collection or a non-relational transformation; it is usually the wrong boundary for a database-wide report.

## Window Functions Are Different

A grouped query returns one row per group. A window function computes across related rows while retaining the original rows:

```sql
SELECT o.id,
       o.customer_id,
       o.total_amount,
       SUM(o.total_amount) OVER (PARTITION BY o.customer_id) AS customer_total
FROM orders AS o
WHERE o.status = 'paid';
```

This is useful for percentages, ranks, and running totals. A window does not replace `GROUP BY` when the required output itself is one row per group.

## Performance, Security, and Failure

Filtering before grouping reduces the input. An index beginning with selective equality and range columns can reduce work, but a grouping operation may still need a sort or hash table. Large groups consume memory; some engines spill to temporary storage. Check `EXPLAIN` for actual row counts, sort/hash behavior, and whether the chosen index helps.

A report query can time out while the underlying transaction remains open if application code holds the connection and transaction carelessly. Set sensible statement timeouts where supported, keep read transactions short, and release cursors before returning a pooled connection. A retry of a read is safe only if the result is allowed to reflect a later snapshot.

Allowlisted report dimensions and sort columns are required because identifiers cannot be bound as values. Tenant filters must be applied before grouping so one tenant cannot influence another tenant's totals.

## Testing and Senior Engineer Thinking

Test empty input, null amounts, repeated values, duplicate keys, timezone boundaries, and a fixture that would expose join fan-out. Compare a grouped result with a hand-calculated fixture and assert decimal representations exactly. For a financial report, define whether refunds, taxes, currency conversion, and rounding happen before or after grouping.

A senior engineer asks: what is the fact grain, what is the grouping grain, which rows are intentionally excluded, and whether the total is additive? A metric that cannot answer those questions is not ready for production, regardless of how fast its query runs.

## Exercises

1. Return every day in a date range with paid-order count, including days with zero orders; explain what calendar table or series source your database uses.
2. Write a query for each customer’s first paid order without grouping away the order identifier.
3. Construct a fixture with two shipments and three items, demonstrate fan-out, and repair it with pre-aggregation.

## Review Questions

- What is the difference between `WHERE` and `HAVING`?
- When does `COUNT(column)` differ from `COUNT(*)`?
- Why is `SUM(DISTINCT amount)` unsafe as a fan-out fix?
- How does a window function differ from `GROUP BY`?

## Summary

Aggregation is a statement about grain. Filter input before grouping, distinguish null from zero, protect totals from one-to-many fan-out, and keep exact decimal or integer money representations. Use PHP to present and map database results, while the database performs relational reduction close to the data.

## References

- [PostgreSQL: Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html)
- [PostgreSQL: Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)
- [PHP manual: PDOStatement](https://www.php.net/manual/en/class.pdostatement.php)
