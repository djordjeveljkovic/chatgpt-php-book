---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 277
title: Database Migration
slug: database-migration
status: complete
summary: ../../_ai/chapter-summaries/277-database-migration-summary.md
---

# Chapter 277 — Database Migration

## Why This Matters

A database migration changes a durable contract shared by code, jobs, reports, administrators, integrations, backups, and recovery procedures. A column rename that looks local can break an old PHP worker, a trigger, a CSV export, or a manually maintained query.

The safest migration is a sequence of compatible states. Add what the next version needs, teach readers and writers to coexist, backfill with evidence, switch authority, and remove old structures only after every consumer and recovery path has moved. This is slower than a single destructive change, but it keeps the intermediate system understandable and recoverable.

## Mental Model

Model a database migration as a contract transition:

~~~text
old application ──┐
                   ├─ compatible schema states ─→ new application
old workers ───────┘             │
                         backfill/reconciliation
                                   ↓
                             retirement
~~~

The migration has separate dimensions:

* schema shape: tables, columns, indexes, constraints, triggers, views;
* data meaning: nullability, status values, units, precision, and ownership;
* access behavior: query plans, locks, isolation, transactions, and timeouts;
* process compatibility: web, CLI, cron, workers, reports, and repair jobs;
* recovery: backups, restore order, rollback, reconciliation, and audit evidence.

A migration is complete only when these dimensions agree. “The migration script succeeded” is one observation, not a compatibility proof.

## Inventory Before Changing

For one target table or capability, inventory:

| Surface | Questions |
| --- | --- |
| writers | Which applications, jobs, triggers, and administrators write it? |
| readers | Which paths depend on columns, ordering, nulls, or status values? |
| constraints | Which uniqueness, foreign-key, check, and default rules hold? |
| indexes | Which queries depend on index order, selectivity, or covering behavior? |
| transactions | Who starts, commits, retries, or locks work? |
| derived state | Which caches, search indexes, files, or messages depend on rows? |
| recovery | Can backups, restore tools, and repair scripts interpret both states? |
| operations | What lock time, table size, replication lag, and capacity are safe? |

Use source search, schema inspection, query logs, deployment artifacts, job schedules, and characterization tests. Mark unknown writers explicitly. A table’s directory, ORM model, or most recent migration file does not prove ownership.

## Define the Invariant

Write the invariant before writing the migration:

~~~text
invoice amount is an integer number of cents
every invoice has exactly one tenant
paid invoices cannot return to pending
an invoice operation has one durable operation_id
the receipt event is emitted only after the invoice mutation commits
~~~

Distinguish structural constraints from application conventions. A database `NOT NULL`, unique index, foreign key, or check constraint can protect an invariant across every writer when the target engine enforces it and every writer uses that database boundary. An invariant enforced only in one PHP controller is vulnerable to cron jobs and administrative scripts.

Do not add a constraint before measuring existing violations. First inventory and quarantine invalid data, then repair or explicitly choose a compatibility rule.

## Expand and Contract

The general sequence is:

1. expand the schema additively;
2. deploy readers that tolerate old and new states;
3. deploy writers that can populate both representations when necessary;
4. backfill in bounded, resumable batches;
5. verify counts, checksums, invariants, and lag;
6. switch reads or write authority deliberately;
7. observe old consumers and repair paths;
8. contract only after the compatibility window closes.

Expansion is not automatically safe. An index can consume capacity or hold locks. A new column can change row size and plans. A trigger can duplicate effects. Measure the database and replication behavior in a production-shaped environment.

## Example: Rename a Status Field

Suppose old code uses `state` and the new application wants `status`. A compatible plan is:

~~~text
state only
   ↓
add nullable status
   ↓
old readers + new readers
   ↓
writers populate state and status
   ↓
backfill fills only NULL status from a current state read; never overwrite a concurrent value
   ↓
new readers prefer status, fallback only with evidence
   ↓
stop old writers and verify consumers
   ↓
remove state later
~~~

Do not rename the column in place while old code still runs. A compatibility reader must define conflicts: if `state = paid` and `status = pending`, which value is authoritative, who repairs it, and which operation is blocked?

## A Typed Migration Policy

Use modern PHP to make migration authority and evidence explicit:

~~~php
<?php

declare(strict_types=1);

enum MigrationPhase: string
{
    case Expanded = 'expanded';
    case Backfilling = 'backfilling';
    case Switched = 'switched';
    case Contracted = 'contracted';
}

final readonly class MigrationObservation
{
    public function __construct(
        public MigrationPhase $phase,
        public int $rowsScanned,
        public int $rowsChanged,
        public int $violations,
        public int $lastKey,
    ) {
        if ($rowsScanned < 0 || $rowsChanged < 0 || $violations < 0 || $lastKey < 0) {
            throw new InvalidArgumentException('Migration counts cannot be negative');
        }

        if ($rowsChanged > $rowsScanned) {
            throw new InvalidArgumentException('Changed rows exceed scanned rows');
        }
    }
}

// Illustrative gate; production policy needs the additional evidence described below.
function mayContract(MigrationObservation $observation, int $remainingRows): bool
{
    return $observation->phase === MigrationPhase::Switched
        && $observation->violations === 0
        && $remainingRows === 0;
}
~~~

The policy is not a migration runner. It prevents a release checklist from treating “backfill finished” as enough evidence. A real policy also needs schema version, database identity, batch cursor, verification timestamp, owner, and recovery reference.

## Dual Reads and Writes

Dual reads can help during a transition, but fallback can hide corruption. Record which field answered, whether both values matched, and whether a repair was scheduled. Do not silently prefer the new value when a conflict may represent a lost write.

Dual writes are harder. They can fail between writes, diverge under retries, or observe different transactions. If both columns must be populated:

* give the logical operation an idempotency key;
* write in one transaction when the database and invariant permit it;
* define behavior if only one representation is changed;
* make repair deterministic and bounded;
* compare counts and values continuously;
* keep one field authoritative at each phase;
* block contraction while violations or unknowns remain.

An asynchronous synchronizer needs its own delivery, ordering, retry, and dead-letter contract. It is not equivalent to an atomic dual write.

## Backfills

A backfill is production work, not a one-line script. Design it with:

* a stable, indexed traversal key;
* bounded batch size and transaction duration;
* resumable cursor and durable progress record;
* rate limit and pause control;
* retry classification and idempotency;
* protection from concurrent application updates;
* verification query and violation handling;
* metrics for age, throughput, errors, locks, and replication lag;
* a stop and rollback/repair decision.

Do not use offset pagination on a changing table for a correctness-critical backfill. Rows can move between pages as updates occur. Use a monotonic key or a carefully defined snapshot, and state what happens to rows inserted or changed during the run.

## Constraints and Indexes

Adding a constraint or index can fail on existing data, block writers, consume disk, change query plans, or increase replication lag. Before applying it:

1. measure violations and duplicate keys;
2. test the DDL and lock behavior with the target database/version;
3. estimate disk, memory, and replication capacity;
4. choose an online or maintenance strategy appropriate to the vendor;
5. verify the resulting constraint or index;
6. observe query plans and write latency after deployment.

Vendor behavior is not interchangeable. PostgreSQL, MySQL, and their versions differ in DDL locking, online index operations, constraint validation, replication, and implicit commits. Follow the target vendor’s documentation and test on the exact production family.

## Transactions and Concurrency

A migration can race with application writes. Define the concurrency model:

* Can old and new writers update the same row?
* Which lock protects the invariant?
* Can a backfill overwrite a newer application value?
* Is the conversion idempotent?
* What does a retry do after an ambiguous commit?
* How are deadlocks and lock timeouts classified?
* Which reads may observe an intermediate state?

Keep transactions short, preserve a canonical lock order, and avoid holding row locks while calling providers. If a data transformation needs multiple steps, store progress and make each step restartable.

## Reconciliation

Reconciliation compares the old and new representations and repairs authorized differences. A useful record includes:

~~~text
entity: invoice-123
operation: status-sync
old value: paid
new value: pending
authority at observation: old
decision: repair new value
reason: old writer still active
actor and time: migration-worker / timestamp
verification: next read matched
~~~

Do not turn reconciliation into an unbounded “make tables equal” job. Define which differences are expected, which are violations, which side is authoritative, and when a repair must stop for human review. Preserve an audit trail without copying sensitive row data into logs.

## Rollback and Recovery

Schema rollback is often less safe than forward repair. Dropping a column destroys information. A backfill may have changed values. A new constraint may prevent old code from writing. A new application may have emitted messages based on new state.

Before migration, decide:

* which schema states each artifact can read and write;
* whether the old artifact can run after expansion or partial backfill;
* how to freeze or fence writers;
* how to restore a backup and replay durable messages;
* how to deduplicate and reconcile durable messages before replay;
* how to reconcile rows and external effects;
* who decides between route rollback, code rollback, and forward recovery.

Chapter 267 covers backups and Chapter 265 covers rollback. This chapter owns the compatibility and data decisions specific to a schema transition.

## Capacity and Operations

Migration load competes with application load. Measure table size, batch duration, rows per second, CPU, I/O, lock waits, replication lag, cache churn, connection use, and queue age. A backfill that is fast in staging may saturate production storage or cause replicas to become too stale for reads.

Use a control loop: observe, pause or slow, verify, and resume. Make progress visible by capability, table, batch cursor, and schema version. Do not report “100% complete” if violations, unknown writers, or unverified replicas remain.

## Common Mistakes

* Renaming or dropping a column while old code still runs.
* Assuming the migration file documents every reader and writer.
* Adding constraints before measuring invalid data.
* Using offset pagination for a changing backfill.
* Running an unbounded transaction over a large table.
* Treating dual writes as atomic without proving the transaction boundary.
* Falling back between conflicting fields without recording the conflict.
* Calling asynchronous synchronization equivalent to an atomic write.
* Ignoring triggers, reports, admin scripts, repair jobs, and backups.
* Testing DDL on a different database engine or version.
* Rolling back code after data or external effects have changed.
* Letting migration load starve application queries and replicas.
* Contracting because the happy-path count matched once.
* Logging raw rows or sensitive data in reconciliation records.

## Senior Engineer Thinking

The senior question is not “how quickly can we change the schema?” It is “which invariant is moving, which consumers must coexist, which state is authoritative at each phase, and what evidence permits the next irreversible step?”

Database migration is compatibility engineering for durable state. Add before removing, make backfills resumable, keep authority explicit, treat constraints and indexes as capacity events, reconcile rather than guess, and make forward recovery a first-class option. The database is part of the application’s architecture and recovery boundary.

## Exercises

1. Write a writer/reader inventory for renaming a legacy status column. Include triggers, workers, reports, backups, and repair scripts.
2. Design an expand-and-contract sequence for splitting a full name into first and last names. State null, formatting, and ambiguous-data rules.
3. Specify a resumable backfill with a cursor, batch limit, transaction scope, pause condition, progress record, and verification query.
4. Build a reconciliation policy for conflicting old/new status values. Identify authoritative phases and human-review conditions.
5. Draw rollback and forward-recovery paths for a migration that has already emitted queue messages and changed a unique constraint.

## Review Questions

* Why is a database migration a contract transition?
* Which surfaces must be inventoried beyond application source?
* What is the difference between schema shape and data meaning?
* Why does expand-and-contract preserve safer intermediate states?
* When can dual reads hide corruption?
* What makes a backfill resumable and safe under concurrent updates?
* Why do constraints and indexes create operational risk?
* Which authority and lock questions must be answered before concurrent writes?
* Why is forward repair often safer than schema rollback?
* What evidence allows the old representation to be contracted?

## Summary

Database migration is compatibility engineering for durable state shared by applications, jobs, reports, integrations, and recovery tools. Inventory all readers and writers, define invariants, expand additively, make old and new code coexist, backfill in bounded resumable batches, control dual reads/writes, measure constraints and index capacity, preserve transaction and lock semantics, reconcile explicit differences, and treat rollback as a data/effect decision. Contract only after authority, consumers, recovery, and violations are understood.

## References

- [Chapter 104 — SQL for PHP Developers](../08-databases/104-sql-for-php-developers.md)
- [Chapter 114 — Transactions](../08-databases/114-transactions.md)
- [Chapter 116 — Locks](../08-databases/116-locks.md)
- [Chapter 120 — Large Datasets](../08-databases/120-large-datasets.md)
- [Chapter 123 — Database vs PHP Responsibilities](../08-databases/123-database-vs-php-responsibilities.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 242 — Idempotency](../16-distributed-systems/242-idempotency.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 267 — Backups](../17-production-engineering/267-backups.md)
- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](./271-legacy-architecture.md)
- [Chapter 274 — Strangler Pattern](./274-strangler-pattern.md)
- [Chapter 276 — Framework Migration](./276-framework-migration.md)
- [Chapter 278 — PHP Version Migration](./278-php-version-migration.md)
