---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 305
title: Database Decision Guide
slug: database-decision-guide
status: complete
summary: ../../_ai/chapter-summaries/305-database-decision-guide-summary.md
---

# Chapter 305 — Database Decision Guide

## Why This Matters

The database is not merely a storage detail. It owns durable constraints, concurrency behavior, query execution, and often the most reliable source of truth. Decide deliberately which work belongs in PHP, SQL, an index, a transaction, a queue, or a projection.

## Start With the Invariant

State what must never happen: duplicate reservation, cross-tenant read, invalid state transition, lost update, or export of uncommitted data. Then identify the concurrent actors, authority, consistency target, and recovery path. Put critical durable invariants at the boundary that can enforce them under races.

## Where the Work Belongs

| Work | Prefer | Check |
| --- | --- | --- |
| filtering and joining durable data | SQL/index | plan, selectivity, tenant scope |
| complex presentation policy | PHP/domain layer | transferred rows and consistency |
| uniqueness/conflict rule | database constraint/transaction | isolation, conflict response |
| slow noninteractive effect | queue/workflow | status, idempotency, reconciliation |
| repeated read shape | cache/projection | authority, freshness, rebuild |
| analytical aggregation | database or dedicated read model | workload, lock/write impact |

This is a decision guide, not an “always use SQL” rule. The input volume, data distribution, network transfer, and ownership decide.

## Queries and Indexes

Inspect generated SQL and its plan. Ask whether predicates are selective, the composite index matches filtering and ordering, joins preserve cardinality, statistics are current, and sorting or fetching dominates. Measure representative parameters; one fast plan does not prove tail behavior.

An index has read, write, storage, and maintenance costs. Add it for a measured access pattern, verify the changed plan and write workload, and remove it only with evidence that no supported path depends on it.

## Constraints and Transactions

Application validation improves feedback but cannot prevent a concurrent race by itself. Use uniqueness, foreign keys, checks, exclusion/conflict mechanisms where supported, and transactions with an explicit isolation assumption. Handle constraint conflicts as expected domain outcomes when they can occur under normal concurrency.

A transaction can make database changes atomic according to the database’s guarantees. It does not make an email, payment, webhook, cache invalidation, or external API call part of the same atomic unit. Use an outbox, idempotency, durable status, and reconciliation for cross-boundary effects.

## Locking and Concurrency

Keep transactions short, access resources in a consistent order, inspect deadlock reports, and retry only safe/idempotent work. Choose pessimistic locking when waiting or conflict prevention is appropriate; choose optimistic version checks when conflicts are less frequent and clients can handle them. Neither choice removes the need to define user-visible conflict behavior.

Read-after-write requirements may require the primary or a consistency token. A read replica can be healthy and still stale. Document whether stale data is acceptable for each capability.

## Pagination and Large Data

Use explicit ordering and a unique tie-breaker. Offset pagination can become expensive and shift under writes; keyset pagination can bound work but requires a validated cursor and compatible filters/collation. For backfills and exports, stream or batch with checkpoints, bounded memory, transaction scope, and restart behavior.

## Tenancy and Security

Tenant scope belongs in every relevant query, join, cache key, projection, export, and repair path. A database connection or ORM repository is not automatically tenant-safe. Test wrong-tenant reads and writes, including list/search paths and error behavior. Apply least privilege to application roles and separate migration or support access.

## Replicas, Caches, and Projections

Use replicas for appropriate read load only when freshness and failover behavior are explicit. Use caches for bounded latency or load reduction, with versioned/scoped keys, stale policy, invalidation or expiry, and rebuild behavior. Use projections for a deliberate read model with source authority, update lag, replay, and rebuild evidence.

## Migration and Reconciliation

Expand before contract: add compatible schema, deploy readers/writers that tolerate both forms, backfill in bounded chunks, verify counts and invariants, switch authority deliberately, then remove old paths. Plan for old PHP workers and queued messages. Reconciliation should compare authoritative state and derived state, classify drift, and repair without creating a second inconsistency.

## Database Decision Worksheet

| Question | Evidence |
| --- | --- |
| Which invariant is protected? | constraint, transaction, policy, or test |
| What is the source of truth? | owner and mutation path |
| What is the workload and data distribution? | plans, counts, percentiles |
| What consistency is required? | read-after-write and staleness budget |
| What happens under conflict or outage? | retry, reject, queue, reconcile |
| What are migration and rollback limits? | compatibility and forward recovery |
| How is tenant scope proven? | negative tests and audit evidence |

## Common Mistakes

- relying on check-then-insert;
- adding indexes without plan and write-cost evidence;
- hiding SQL behind an ORM;
- assuming a transaction includes external effects;
- ignoring isolation and deadlocks;
- treating replicas as current;
- calling a cache authoritative;
- omitting tenant predicates or cursor semantics;
- migrating schema without old-worker compatibility.

## Exercises

1. Place five operations in PHP, SQL, an index, a transaction, a queue, or a projection.
2. Design database enforcement for reservation overlap and explain conflict handling.
3. Compare offset and keyset pagination for a changing tenant dataset.
4. Plan a migration with old workers, backfill checkpoints, and reconciliation.
5. Inspect an ORM query and list the plan and tenant-scope evidence you need.

## Review Questions

- Which invariant belongs in the database?
- When is PHP the better boundary?
- Why does a transaction not cover a provider?
- What does a replica health check fail to prove?
- How do keyset cursors constrain query design?
- What makes a migration forward-recoverable?

## Summary

Database decisions follow invariants, authority, data shape, selectivity, consistency, concurrency, capacity, security, and recovery. Use constraints and transactions for durable rules, inspect real plans, make replica/cache/projection limits explicit, paginate and backfill with bounded work, and preserve tenant and migration safety.

## Chapter 306 Handoff

Database choices need performance evidence. Chapter 306 turns that evidence into a layered performance checklist covering workloads, tail latency, PHP-FPM, SQL, caches, queues, and downstream capacity.

## References

- [Chapter 106 — Prepared Statements](../08-databases/106-prepared-statements.md)
- [Chapter 110 — Query Plans](../08-databases/110-query-plans.md)
- [Chapter 114 — Transactions](../08-databases/114-transactions.md)
- [Chapter 117 — Deadlocks](../08-databases/117-deadlocks.md)
- [Chapter 119 — Pagination](../08-databases/119-pagination.md)
- [Chapter 123 — Database versus PHP Responsibilities](../08-databases/123-database-vs-php-responsibilities.md)
- [Chapter 277 — Database Migration](../18-legacy-php/277-database-migration.md)
- [Chapter 287 — Search/Filtering Service](../19-small-engineering-projects/287-search-filtering-service.md)
