# AI Summary — Chapter 173 — Property-Based Testing

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Property-based testing checks invariants over generated inputs. The chapter covers properties and independent oracles, deterministic generators, shrinking, domain boundaries, metamorphic and stateful tests, side effects, and regression counterexamples.

## Concepts already explained

Property, generator, oracle, shrinking, seed, metamorphic property, model-based test, and bounded generation.

## Examples used

A deterministic PHPUnit property for date ranges and guidance for generated stateful commands.

## Cross-references

- [Chapter 170 — Test Design](../../volumes/11-testing/170-test-design.md)
- [Chapter 172 — Mutation Testing](../../volumes/11-testing/172-mutation-testing.md)
- [Chapter 176 — Database Testing](../../volumes/11-testing/176-database-testing.md)

## Open threads

Continue with Chapter 174 on diagnosing and repairing flaky tests.

## Exact next section

Chapter 174 — Flaky Tests: the Why This Matters section.

## Technical verification notes

PHP fences and local links were checked after writing; property runs and Eris integration were not executed.
