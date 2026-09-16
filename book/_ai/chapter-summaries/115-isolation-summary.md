# AI Summary — Chapter 115 — Isolation

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains dirty/non-repeatable/phantom/lost-update/write-skew anomalies, SQL isolation levels, PostgreSQL versus MySQL differences, invariant-driven design, conditional updates, PDO `SET TRANSACTION`, retries, snapshots, pooled connections, performance, concurrency tests, exercises, and review questions.

## Concepts already explained

Anomaly taxonomy; statement versus transaction snapshots; isolation as a correctness contract; constraint/atomic-statement/lock/isolation choices; serialization retry.

## Terminology established

Dirty read, non-repeatable read, phantom, lost update, write skew, statement snapshot, transaction snapshot, serializable failure.

## Examples used

Overlapping reservation count race; inventory conditional update; PDO serializable transaction; two-connection concurrency test design.

## Cross-references

Chapter 114 transactions; Chapter 116 locks; Chapter 117 deadlocks; Chapter 118 concurrency.

## Open threads

Explicit lock modes, deadlock handling, and broader concurrency patterns continue in Chapters 116–118.

## Exact next section

Chapter complete; next chapter is 116 Locks.

## Technical verification notes

PHP example sets PostgreSQL transaction isolation before the first data statement and rolls back active transactions. PostgreSQL and MySQL isolation behavior is qualified and linked to official documentation. Concurrency claims avoid treating SQLite as a production-engine substitute.
