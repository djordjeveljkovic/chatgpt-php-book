# AI Summary — Chapter 160 — Integration Tests

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Covers real boundary verification for databases, filesystems, queues, serialization, and providers; fixture isolation, migrations, transactions, concurrency, external services, diagnostics, readiness, and CI execution.

## Concepts already explained

Integration evidence depends on production-like engines and connection topology. In-memory substitutes and outer transactions can hide real database behavior.

## Terminology established

Integration boundary, fixture isolation, schema snapshot, transaction strategy, provider fake, readiness deadline, environment drift.

## Examples used

A PDO repository, database fixture strategy, and provider failure mapping.

## Cross-references

- [Chapter 159 — Unit Tests](../../volumes/11-testing/159-unit-tests.md)
- [Chapter 176 — Database Testing](../../volumes/11-testing/176-database-testing.md)

## Exact next section

Chapter 161 — Feature Tests: the Why This Matters section.

## Technical verification notes

PHP/PDO examples linted; references include PHPUnit, PDO, Martin Fowler, and OWASP guidance.
