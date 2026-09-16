# AI Summary — Chapter 168 — Fakes

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Explains fakes as simplified working implementations, in-memory repositories and queues, contract tests, fake drift, state isolation, resource limits, testing, exercises, and review questions.

## Concepts already explained

Fakes provide behavior rather than one answer, so they can model workflows while diverging from database, broker, filesystem, or network semantics. Shared contract tests and focused integration tests keep them honest.

## Terminology established

Fake, in-memory repository, contract test, fake drift, behavior gap, test-state isolation.

## Examples used

An in-memory order repository, duplicate-ID policy, in-memory queue, and repository contract data provider.

## Cross-references

- [Chapter 167 — Stubs](../../volumes/11-testing/167-stubs.md)
- [Chapter 176 — Database Testing](../../volumes/11-testing/176-database-testing.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 169 — Spies: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XI proofread. Fake and contract-test guidance links to PHPUnit and Fowler.
