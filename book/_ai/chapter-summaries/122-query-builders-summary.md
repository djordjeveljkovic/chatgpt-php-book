# AI Summary — Chapter 122 — Query Builders

- Status: complete
- Volume: Volume 8 — DATABASES
- Last updated: 2026-09-16

## Written material

Explains parameterized values versus SQL identifiers, allowlisted sorting, composable filters, list parameters, Doctrine DBAL builders, raw SQL, mutable builder state, pagination, type/null semantics, authorization scope, and query observability.

## Concepts already explained

- Query builders improve composition but do not secure identifiers or prove performance.
- Values stay bound; structural fragments come from finite trusted maps.
- Builders should be fresh or explicitly immutable, and generated SQL still needs plan inspection.
- Empty lists, omitted filters, and SQL NULL require distinct application semantics.

## Terminology established

Query builder, structural fragment, allowlist, dialect, projection, builder state, query fingerprint.

## Examples used

- Safe sort map and deterministic ordering.
- Typed filter builder and Doctrine DBAL query builder.

## Cross-references

- [Chapter 112 — Joins](../../volumes/08-databases/112-joins.md)
- [Chapter 119 — Pagination](../../volumes/08-databases/119-pagination.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 123 — Database vs PHP Responsibilities: the Why This Matters section.

## Technical verification notes

PHP snippets use current syntax; PostgreSQL-specific `ILIKE` is labeled; list binding and DBAL APIs are explicitly qualified by driver/library behavior.
