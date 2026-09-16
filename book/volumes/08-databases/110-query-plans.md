---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 110
title: Query Plans
slug: query-plans
status: complete
summary: ../../_ai/chapter-summaries/110-query-plans-summary.md
---

# Chapter 110 — Query Plans

## Why This Matters

A SQL statement describes the result, not the procedure. The database optimizer chooses a procedure: which table to read first, which index to use, how to join rows, whether to sort, and when to stop. Two equivalent statements can receive different plans, and the same statement can change plans as data, statistics, indexes, or engine versions change.

Knowing how to read a plan turns “the query is slow” into a testable claim. You can ask whether the engine scans too many rows, estimates the cardinality incorrectly, sorts a large intermediate result, or chooses a join order that multiplies work. [Chapter 109](109-composite-indexes.md) explains access paths; [Chapter 111](111-explain.md) provides the commands and measurements for examining them.

## Mental Model

A plan is a tree of relational operations. Leaf operations read rows; parent operations filter, join, aggregate, sort, or limit them:

```text
Limit 50
└── Sort by created_at DESC
    └── Filter status = 'open'
        └── Index or table scan on tickets
```

The tree's shape describes dependencies. In a nested-loop join, the outer operation produces rows and the inner operation runs once per outer row. If the outer side produces 100,000 rows and the inner side performs a costly scan each time, the total work can be enormous. A hash or merge join may be better when both inputs are large, while a nested loop can be excellent when the outer side is small and the inner side has a selective index.

Most optimizers compare estimated costs rather than timing every alternative. Estimates depend on table and index statistics, predicate selectivity, available indexes, memory settings, and engine-specific cost models. A plan with a lower estimated cost is the optimizer's best model, not a proof of lower wall-clock latency.

## What a Plan Contains

Plan output differs by database, but these fields are common:

| Field | Question it answers |
| --- | --- |
| Access method | Is this a sequential/table scan, index lookup, range scan, or bitmap operation? |
| Estimated rows | How many rows does the optimizer expect at this node? |
| Actual rows | How many rows did execution produce, when runtime analysis is enabled? |
| Startup and total cost | How expensive does the optimizer expect the node to be? |
| Loops | How many times did a child execute, often under a nested loop? |
| Filter | Which rows were discarded after being read? |
| Sort or materialization | Is an intermediate result being ordered or stored? |
| Join type/order | Which inputs are joined first and by what algorithm? |

An estimate that is off by a factor of 100 can propagate to parent nodes and produce a poor join order. Compare estimates with actuals at the first large divergence. Fixing that selectivity problem—through statistics, a better predicate, data modeling, or an index—may improve the whole plan more than tuning a later node.

## Core Access Methods

A sequential or table scan reads the table pages and tests predicates. It is often correct for a small table or a query returning a large fraction of rows. An index lookup navigates an index to a key or range, then may fetch table rows. It is attractive when the range is narrow and the random fetch cost is lower than scanning everything.

A bitmap or index-merge operation can combine candidate row locations from one or more indexes before fetching rows. The names and implementations vary. A covering or index-only scan can return values without visiting the base table when visibility and projection conditions allow it. A sort node may be avoidable when an index supplies the required order, but a sort can be cheaper than an awkward index path for a small result.

Do not label a sequential scan “bad” by itself. Label it bad when its input size, repeated execution, or latency violates the workload's requirement. A full scan of a 20-row lookup table is normally healthier than an index designed solely to avoid it.

## Join Plans

For two inputs, a nested-loop join repeatedly probes the inner side. With an index on the inner join key and a small outer result, the cost is approximately `O(O * log I)` for `O` outer rows and `I` inner rows, plus fetched rows. Without a useful inner access path, it can approach `O(O * I)` work.

A hash join builds a hash table from one input and probes it with the other. Its typical work is approximately linear in both inputs, `O(O + I)`, with memory and spill costs. It generally supports equality joins. A merge join walks sorted inputs, often around `O(O + I)` after sorting or when indexes already provide order. Exact algorithms and eligibility are engine-specific.

This SQL does not dictate the join order:

```sql
SELECT o.id, c.name
FROM orders AS o
JOIN customers AS c ON c.id = o.customer_id
WHERE o.organization_id = :organization_id
  AND o.created_at >= :since;
```

The optimizer may filter `orders` first and probe `customers`, scan customers and join orders, or choose another strategy. The useful index might be on `(organization_id, created_at, customer_id)` for the filter, on `orders(customer_id)` for a different join shape, or no extra index if the selected result is large. Read the plan and the data distribution before deciding.

## A PHP Workflow

Application code should keep normal query execution separate from diagnostics. A safe plan collector accepts only a known query or a trusted query identifier; it should not accept arbitrary SQL text from an HTTP request.

```php
<?php

declare(strict_types=1);

final class QueryDiagnostics
{
    public function __construct(private PDO $db)
    {
    }

    /** @return list<array<string, mixed>> */
    public function explainTickets(int $organizationId, string $since): array
    {
        $sql = <<<'SQL'
            EXPLAIN
            SELECT id, subject, created_at
            FROM tickets
            WHERE organization_id = :organization_id
              AND state = 'open'
              AND created_at >= :since
            ORDER BY created_at DESC
            LIMIT 100
            SQL;

        $statement = $this->db->prepare($sql);
        $statement->execute([
            'organization_id' => $organizationId,
            'since' => $since,
        ]);

        return $statement->fetchAll(PDO::FETCH_ASSOC);
    }
}
```

The exact result columns depend on the database driver. Configure PDO's exception mode and use a diagnostic connection with the same schema, statistics, role permissions, and relevant settings as the target workload. A plan from a different engine or empty database answers a different question.

## Cardinality and Statistics

The optimizer needs a model of how values are distributed. A uniform estimate can be badly wrong for a skewed tenant, a status where almost every row is `active`, or correlated columns such as `country` and `timezone`. Histograms, extended statistics, sampled statistics, and engine-specific analysis features can improve the model, but their names and limits vary.

After bulk loads, deletes, or a major change in value distribution, check whether statistics are current. A stale statistic can make the engine reject a good index or choose a nested loop for a large result. Updating statistics can change plans without changing application code, so record the maintenance operation and observe the result.

Prepared statements add another consideration. Some engines choose a plan using a representative or generic parameter assumption; others can choose custom plans for values. A query that is fast for a rare tenant may be slow for the tenant holding most rows. Do not conclude that the plan is universally good from one parameter value. Test common, rare, empty, and boundary cases.

## Bad Example: Fixating on One Node

A team sees a sort node and adds an index to eliminate it. The index then causes millions of random table fetches, while the original plan sorted a small, already-filtered result in memory. The new plan is slower even though the sort disappeared.

The correct question is total work: rows read, rows discarded, memory, disk spill, loops, and end-to-end latency. A plan node is not an isolated defect. Compare the complete plan with representative parameters and data.

## Better Example: Follow the First Estimate Error

Suppose the plan expects 10 rows from `organization_id = :id` but receives 500,000. The optimizer may pick a nested loop and repeatedly probe an inner table because it believes the outer side is tiny. Adding a random index to the inner side might help somewhat, but first investigate:

1. Is the statistics sample or refresh out of date?
2. Is the value highly skewed compared with the model?
3. Are the column types or collations causing a cast?
4. Are correlated predicates missing from the statistics model?
5. Does the application need a different tenant or time boundary?

After correcting the cause, re-run the plan with the same parameters. A good fix should be evaluated against the whole workload, not only the one slow request.

## Performance and Plan Stability

Plans can change after data growth, index creation, statistics refreshes, configuration changes, failover to another server, or an engine upgrade. Record plan shape and query metrics for important statements. A plan hash or normalized digest can help group executions, but it is not a semantic guarantee: two plans with similar labels can have different row counts and costs.

Set a latency objective and alert on measured behavior, not merely on a plan text change. A changed plan that reduces p95 latency is not a regression. Conversely, a plan can retain the same shape while becoming slower because the data no longer fits in memory.

Plan forcing or hints can be useful for a documented emergency, but they trade optimizer freedom for a manual assumption. Use a hint only when the engine supports it, the reason is understood, and the dependency is monitored. Prefer correcting statistics, schema, query shape, or data distribution where possible.

## Security and Operations

Plan output can contain SQL text, relation names, parameter values, row estimates, and environment details. Restrict diagnostic endpoints and redact sensitive literals before sending plans to logs or external tooling. `EXPLAIN ANALYZE` executes the statement on engines that implement it; never run it against an untrusted or mutating query without understanding side effects. [Chapter 111](111-explain.md) details this distinction.

A plan inspection command should use a read-only database role where the engine supports it, a bounded query, and a timeout. Do not run diagnostics inside every production request. Sample slow queries through the database's supported telemetry or an application profiler, then reproduce and inspect them deliberately.

## Testing

Keep a small set of plan regression cases for high-value queries. Seed enough data to exercise the intended access path and include skewed values. Run them against the supported database version in CI or a staging environment. Assert broad properties—such as the absence of a full scan on a table that is expected to be large—only when that property is part of an operational requirement. Allow expected plan changes after engine upgrades through an explicit review.

Measure execution separately from planning where the engine exposes both. Warm and cold cache results answer different questions. Use the same bind types and session settings as the application; a literal query and a prepared query may be planned differently in some systems.

## Common Mistakes

- Treating the cheapest estimated plan as proof of the fastest runtime.
- Calling every sequential scan a bug.
- Ignoring loops and multiplying a child cost by the number of outer rows.
- Looking only at total cost instead of the first large estimate error.
- Testing one parameter value and declaring a plan stable.
- Reading a plan from a different engine, schema, statistics state, or data volume.
- Adding hints before understanding cardinality, casts, or stale statistics.
- Running an execution plan with analysis on a mutating statement without considering side effects.
- Exposing raw plans and SQL parameters through a public diagnostic endpoint.

## Senior Engineer Thinking

A plan is an evidence-backed hypothesis about cost. Read it as a tree, compare estimated and actual cardinalities, and connect the largest divergence to a data, schema, or query assumption. Then measure the fix across the parameter and workload distribution that matters.

Good plan work also includes operational context: database version, statistics age, cache state, bind behavior, concurrency, lock waits, and deployment changes. The target is predictable service behavior, not a visually pleasing plan or one fast benchmark run.

## Exercises

1. Choose a two-table query and draw its plan as a tree. For each node, record estimated rows, actual rows, loops, and whether the operation can spill to disk.
2. Create skewed tenant data and compare plans for a small tenant and the largest tenant. Explain any join-order or access-method change.
3. Make a query's statistics stale, capture the plan, refresh statistics, and compare the result. Document what changed and what did not.
4. Build a PHP diagnostic command that accepts a fixed query name and bind values, returns a redacted plan, and cannot execute arbitrary SQL.

## Review Questions

1. What does a query plan describe that SQL text leaves open?
2. Why can a nested-loop join be excellent for one cardinality and disastrous for another?
3. What is the practical value of comparing estimated and actual rows?
4. Why is a sequential scan not automatically a defect?
5. Which changes can cause a previously good plan to change?
6. Why should prepared statements be tested with multiple parameter distributions?
7. When is a plan hint a risky substitute for fixing the underlying model?

## Summary

The optimizer turns SQL into a plan tree of scans, index access, joins, filters, sorts, and limits. Its choices depend on statistics, data distribution, indexes, settings, parameter behavior, and engine version. Read plans as cost hypotheses: find the first major cardinality error, account for loops and spills, and measure the complete workload. Keep diagnostics controlled and treat execution plans as operational evidence that must be revisited as the system changes.

## References

- [PostgreSQL documentation: using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL documentation: planner statistics](https://www.postgresql.org/docs/current/planner-stats.html)
- [MySQL 8.4 Reference Manual: optimizing queries with EXPLAIN](https://dev.mysql.com/doc/refman/8.4/en/using-explain.html)
- [MySQL 8.4 Reference Manual: optimizer statistics](https://dev.mysql.com/doc/refman/8.4/en/optimizer-statistics.html)
- [PHP Manual: PDO error handling](https://www.php.net/manual/en/pdo.error-handling.php)
