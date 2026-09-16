---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 111
title: EXPLAIN
slug: explain
status: complete
summary: ../../_ai/chapter-summaries/111-explain-summary.md
---

# Chapter 111 — EXPLAIN

## Why This Matters

Indexes and query rewrites are hypotheses until the database shows what it plans to do. `EXPLAIN` exposes that plan: scans, index choices, join order, estimates, filters, sorts, and limits. Runtime variants add actual row counts and timing. Reading this output is how you distinguish “the index is missing” from “the index exists but the query returns half the table” or “the estimates are wrong.”

`EXPLAIN` is vendor-specific in syntax and output. Treat its output as diagnostic evidence tied to a database engine, version, schema, statistics state, and parameter set. [Chapter 110](110-query-plans.md) develops the plan model; this chapter gives a disciplined inspection workflow and PHP examples.

## Mental Model

There are two questions to separate:

1. **What plan would the optimizer choose?** Plain `EXPLAIN` answers this without normally running the statement.
2. **What happened during execution?** A runtime form such as PostgreSQL `EXPLAIN (ANALYZE)` executes the statement and reports actual rows, loops, and timing.

An estimated plan is safe to inspect for a read query but can still reveal schema and SQL details. An analyzed plan is an execution. On a `SELECT`, that consumes resources and may acquire locks. On an `INSERT`, `UPDATE`, or `DELETE`, it can change data. Never add an execution option casually to a mutating statement.

## Minimal Examples

PostgreSQL's text form is:

```sql
EXPLAIN
SELECT id, subject, created_at
FROM tickets
WHERE organization_id = 42
  AND state = 'open'
ORDER BY created_at DESC
LIMIT 100;
```

For runtime evidence, PostgreSQL supports options such as:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT id, subject, created_at
FROM tickets
WHERE organization_id = 42
  AND state = 'open'
ORDER BY created_at DESC
LIMIT 100;
```

`ANALYZE` runs the query. `BUFFERS` adds buffer-use information, and `FORMAT JSON` makes the plan easier for tools to parse. These are PostgreSQL options; use the target engine's grammar rather than copying them to MySQL.

MySQL's estimated forms include:

```sql
EXPLAIN
SELECT id, subject, created_at
FROM tickets
WHERE organization_id = 42
  AND state = 'open'
ORDER BY created_at DESC
LIMIT 100;
```

MySQL can also return structured output:

```sql
EXPLAIN FORMAT=JSON
SELECT id, subject, created_at
FROM tickets
WHERE organization_id = 42
  AND state = 'open'
ORDER BY created_at DESC
LIMIT 100;
```

On supported MySQL 8.0 releases, `EXPLAIN ANALYZE` reports an executed tree with actual timing and row information:

```sql
EXPLAIN ANALYZE
SELECT id, subject, created_at
FROM tickets
WHERE organization_id = 42
  AND state = 'open'
ORDER BY created_at DESC
LIMIT 100;
```

Check the version-specific manual before putting an option into migration or monitoring tooling. SQLite has its own `EXPLAIN QUERY PLAN` output and should be learned from the SQLite documentation rather than treated as a smaller PostgreSQL or MySQL plan.

## Reading a Plan

Read from the leaves upward. First identify which relation is accessed and how: sequential/table scan, index lookup, range access, bitmap operation, or an engine-specific method. Then inspect the rows entering and leaving each node, filters, loops, and sorts.

In PostgreSQL text output, a simplified node might look like:

```text
Limit  (cost=0.42..18.10 rows=50 width=48)
  ->  Index Scan Backward using tickets_org_state_created_idx on tickets
        (cost=0.42..900.00 rows=2500 width=48)
        Index Cond: (organization_id = 42)
        Filter: (state = 'open')
```

The `cost` values are planner units, not milliseconds. `rows` is an estimate for one execution of the node, and `width` is an estimated row size. An actual plan might add:

```text
(actual time=0.030..0.210 rows=50 loops=1)
```

Multiply child work by loops. A node taking 2 ms but executing 50,000 times is not a 2 ms operation in the request. Also distinguish an index condition that narrows navigation from a filter applied after rows are fetched. A large gap between rows read and rows returned suggests wasted work or a cardinality problem.

MySQL's tabular output uses columns such as `type`, `possible_keys`, `key`, `key_len`, `rows`, `filtered`, and `Extra`; JSON and tree output provide more structure. `type` is an access-method category, not a universal quality score. `Using temporary` or `Using filesort` can be relevant, but neither phrase alone proves a defect. Assess row counts, query limits, memory, and measured latency together.

## A Parameterized PHP Diagnostic

Use a fixed statement and bind values. The `EXPLAIN` keyword changes the statement being prepared, while the predicates remain parameterized:

```php
<?php

declare(strict_types=1);

final class TicketPlanReader
{
    public function __construct(private PDO $db)
    {
    }

    /** @return list<array<string, mixed>> */
    public function estimatedPlan(int $organizationId, string $since): array
    {
        $statement = $this->db->prepare(
            <<<'SQL'
            EXPLAIN
            SELECT id, subject, created_at
            FROM tickets
            WHERE organization_id = :organization_id
              AND state = 'open'
              AND created_at >= :since
            ORDER BY created_at DESC
            LIMIT 100
            SQL
        );
        $statement->execute([
            'organization_id' => $organizationId,
            'since' => $since,
        ]);

        return $statement->fetchAll(PDO::FETCH_ASSOC);
    }
}
```

The result shape varies by driver. PostgreSQL can use `FORMAT JSON`, after which one row contains a JSON document; MySQL's `FORMAT=JSON` has a different document shape. Keep engine-specific readers behind an adapter rather than writing one parser that assumes every plan is alike.

Do not accept arbitrary SQL or an index name from a web request. Identifiers cannot be bound as values, and concatenating them creates an injection boundary. Use a server-side allowlist of query definitions:

```php
/** @var array<string, callable(PDO, array<string, scalar>): array> $queries */
$queries = [
    'tickets.open' => static function (PDO $db, array $params): array {
        $reader = new TicketPlanReader($db);
        return $reader->estimatedPlan(
            (int) $params['organization_id'],
            (string) $params['since'],
        );
    },
];
```

A production diagnostic command should also authenticate operators, redact values from logs, use a timeout, and run against a controlled replica or staging copy where the plan is representative.

## Estimates Versus Actuals

The most useful comparison is often the first node where estimated and actual row counts diverge. Suppose the optimizer expects 20 rows but sees 200,000. A downstream nested loop may now execute thousands of times more work than planned. Investigate stale statistics, skew, correlated predicates, implicit casts, or a parameter-sensitive workload.

A runtime plan's timing includes the current cache and system load. A cold cache, a warm cache, lock waits, parallel workers, and concurrent writes can produce different results. Use repeated samples and report the environment with the plan. Do not compare a local laptop plan with a production latency number as if they were the same measurement.

`EXPLAIN ANALYZE` can also alter behavior through triggers, volatile functions, locks, or writes. Even for a `SELECT`, functions invoked by the query can have side effects in a poorly designed system. Use a read-only transaction or a safe replica when the engine and workload support it, and understand that a replica's statistics and data distribution may differ.

## Comparing Index Candidates

Suppose a query lists open tickets for one organization. Capture a baseline estimated and runtime plan, then add one candidate index in a test environment:

```sql
CREATE INDEX tickets_org_state_created_idx
    ON tickets (organization_id, state, created_at DESC);
```

Run the same bind values and compare:

- rows read versus rows returned;
- whether a sort or temporary structure remains;
- loops and actual time at the expensive nodes;
- buffer or page reads;
- memory use and disk spills;
- planning time, execution time, and write impact on the table.

A plan that changes from a table scan to an index scan is not automatically better. If the query returns most tickets, a scan may remain cheaper. Conversely, an index scan that returns only 100 rows from a huge table can be a significant improvement even if its estimated cost numbers look unfamiliar.

Drop or retain the candidate through a documented migration process. Do not create production indexes ad hoc from an application request. [Chapter 108](108-indexes.md) and [Chapter 109](109-composite-indexes.md) cover index lifecycle and column order.

## Bad Example: Logging Plans in Requests

This pattern is dangerous:

```php
// Do not do this with request input.
$sql = 'EXPLAIN ' . $_GET['sql'];
$plan = $db->query($sql)->fetchAll();
```

It grants SQL execution privileges to an untrusted caller, may expose sensitive data, and can consume enough database resources to become a denial of service. It also makes plan output impossible to correlate with a known application query.

## Better Example: Fixed Query Definitions

Register a query identifier and a fixed SQL template in a console-only diagnostic command. Values are still bound:

```php
final class PlanCommand
{
    public function __construct(private TicketPlanReader $reader)
    {
    }

    public function run(int $organizationId, string $since): void
    {
        foreach ($this->reader->estimatedPlan($organizationId, $since) as $row) {
            print_r($row);
        }
    }
}
```

Keep the output in an access-controlled artifact with the database engine and version, schema migration revision, statistics timestamp, bind-shape category (for example, small or large tenant), and timestamp. Redact literal values if the output leaves the protected environment.

## Edge Cases

- **Prepared statements:** The database may choose a generic or parameter-specific plan, depending on engine and driver settings. Reproduce the application's prepare mode and bind types.
- **Views and CTEs:** A plan may inline, materialize, or otherwise transform them. Read the actual nodes instead of assuming a view is a stored result.
- **Partitioning:** Look for partition pruning. A plan that scans every partition can dominate cost even if each local index is good.
- **Parallelism:** Worker counts, startup cost, and coordination affect whether a parallel plan helps. A plan from a different server setting is not directly comparable.
- **Locks and waits:** A plan may show efficient row access while the request is slow waiting for a lock. Pair plans with wait and transaction metrics.
- **Cache and JIT:** Buffer cache state and optional compilation change timing. Record relevant settings when benchmarking.
- **Row-level security:** The database may add hidden predicates. Inspect permissions and policies when a plan seems to filter more than the application query says.

## Performance and Operations

Keep an estimated plan close to the query-review workflow and use runtime plans deliberately. Production telemetry should identify normalized query text, latency percentiles, rows returned, errors, and waits. A plan snapshot helps explain a regression but should not be the only metric.

For a plan regression, compare the last known-good deployment, migration, statistics refresh, engine setting, and data distribution. Roll back a harmful schema change only with a migration plan and knowledge of active dependencies. A forced plan can stabilize an incident, but document its expiry and verify that it does not hide a model problem.

Plan output can be large. Store a compact summary plus a link to the full protected artifact, and avoid putting raw bind values into general application logs. Make diagnostic commands observable and rate-limited.

## Testing

Plan tests are integration tests. Run them against the same database family and a representative schema. Seed enough rows to make the intended choice meaningful, and include both common and selective parameter values. Test query results as well as plan properties; an optimization must preserve semantics.

Keep assertions resilient. An exact textual plan can change after a patch release or statistics update. Prefer a structured output format where available and assert a requirement such as partition pruning, an index condition, or a bounded row estimate. Review intentional plan changes during engine upgrades.

For the PHP diagnostic code, test that only registered query identifiers are accepted, parameters are bound, plan output is redacted when required, and database exceptions are reported without leaking credentials. Use a real database for the SQL behavior and a unit test for command authorization and allowlisting.

## Common Mistakes

- Confusing estimated cost units with milliseconds.
- Running `EXPLAIN ANALYZE` on a write statement without understanding that it executes.
- Comparing output from different engines, versions, schemas, or statistics states.
- Looking at a single plan node without accounting for loops and parent work.
- Assuming a sort or table scan is always a problem.
- Binding the wrong type and then diagnosing a plan for a different expression.
- Exposing arbitrary SQL, schema details, or bind values through a public endpoint.
- Treating one parameter value as representative of a skewed workload.
- Asserting exact plan text in brittle tests.

## Senior Engineer Thinking

Use `EXPLAIN` to connect an application symptom to a database operation. Establish a baseline, keep the query and bind values fixed, compare the first cardinality divergence, and measure the complete workload after one controlled change. A plan is useful only with its environment: engine version, schema, statistics, data distribution, cache state, settings, and concurrency.

Make diagnostics safe enough to run under pressure. Fixed query definitions, authorization, bounded execution, redaction, and read-only defaults let a team investigate without turning an incident tool into a new attack surface.

## Exercises

1. Capture an estimated plan and a runtime plan for one read query in PostgreSQL or MySQL. Annotate each node with access method, estimated rows, actual rows, loops, and sorting.
2. Change one index or predicate, then compare plans for a small and a large tenant. Explain why the same query can need different access paths.
3. Write a console-only PHP command that exposes two allowlisted diagnostic queries and rejects arbitrary SQL. Add tests for binding and authorization.
4. Run a plan regression after a statistics refresh or engine upgrade. Decide which plan changes are harmless and which require action.

## Review Questions

1. What is the difference between an estimated plan and a runtime plan?
2. Why can `EXPLAIN ANALYZE` be unsafe for a mutating statement?
3. What does a large estimated-versus-actual row difference suggest?
4. Why must loops be included when judging a plan node's cost?
5. Which parts of a plan comparison must be kept constant?
6. Why should a PHP plan tool allowlist query definitions?
7. What makes a plan regression test resilient across engine updates?

## Summary

`EXPLAIN` reveals how a database intends to execute a query; runtime variants reveal what happened. Read the plan from its leaf scans through joins, filters, sorts, and limits, and look for the first meaningful cardinality error. Compare candidate indexes and rewrites using the same engine, schema, statistics, bind shapes, and representative data. Keep PHP diagnostics fixed, authorized, parameterized, redacted, and aware that analyzed plans execute work.

## References

- [PostgreSQL documentation: using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL documentation: EXPLAIN command reference](https://www.postgresql.org/docs/current/sql-explain.html)
- [MySQL 8.4 Reference Manual: EXPLAIN statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [MySQL 8.4 Reference Manual: obtaining information with EXPLAIN ANALYZE](https://dev.mysql.com/doc/refman/8.4/en/explain.html#explain-analyze)
- [SQLite documentation: EXPLAIN QUERY PLAN](https://sqlite.org/eqp.html)
- [PHP Manual: PDO::prepare](https://www.php.net/manual/en/pdo.prepare.php)
