# AI Summary — Chapter 112 — Joins

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains join output grain, inner/left/right/full/cross/self joins, `ON` versus `WHERE`, `EXISTS`/`NOT EXISTS`, foreign-key cardinality, PDO flat-result fetching, N+1 avoidance, nulls, plans, tenant scope, failures, tests, exercises, and review questions.

## Concepts already explained

Join cardinality and row grain; outer-join preservation; semijoin/anti-join; key-based relationships; fan-out awareness; database-side relational work.

## Terminology established

Input/output grain, preserved side, null-extended row, semijoin, anti-join, fan-out, N+1.

## Examples used

Customers, orders, and order_items schema; paid-order inner/left joins; `EXISTS`; PDO order-item report; N+1 discussion.

## Cross-references

[Chapter 111 — EXPLAIN](../../volumes/08-databases/111-explain.md); Chapter 113 aggregation.

## Open threads

Locks and concurrency coordination continue in Chapters 116–118.

## Exact next section

Chapter complete; next chapter is 113 Aggregation.

## Technical verification notes

PHP snippets use strict types, PDO exception mode, prepared statements, and valid named parameters. SQL examples use standard join syntax with PostgreSQL support noted for full outer joins. Links point to PostgreSQL and PHP manuals.
