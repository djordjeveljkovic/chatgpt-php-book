# AI Summary — Chapter 113 — Aggregation

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains grouping grain, `WHERE`/`HAVING`, count/null behavior, time ranges and timezones, conditional aggregation, join fan-out and pre-aggregation, exact money handling, PHP mapping, window-function distinction, performance, security, testing, exercises, and review questions.

## Concepts already explained

Group partitions; `COUNT(*)` vs `COUNT(column)` vs distinct; null versus zero; conditional metrics; fact/grouping grain; pre-aggregation; window functions; decimal money boundaries.

## Terminology established

Grouping grain, fact grain, fan-out, conditional aggregation, half-open interval, window function.

## Examples used

Daily paid-order report; customer totals; item/shipment pre-aggregation; PDO decimal mapping; customer partition window total.

## Cross-references

Chapter 112 joins; Chapter 111 EXPLAIN; Chapters 104–107 for SQL/PDO/query design.

## Open threads

Transaction and isolation effects on report snapshots continue in Chapters 114–115.

## Exact next section

Chapter complete; next chapter is 114 Transactions.

## Technical verification notes

PHP snippet is syntactically valid and uses PDO prepared statements. SQL distinguishes portable `CASE` conditional sums from engine-specific date behavior. References use PostgreSQL aggregate/window docs and PHP PDO docs.
