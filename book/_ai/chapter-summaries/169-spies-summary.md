# AI Summary — Chapter 169 — Spies

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Explains spies as post-execution interaction recorders, comparison with mocks/stubs/fakes, safe recording, failure behavior, asynchronous effects, ordering, testing, exercises, and review questions.

## Concepts already explained

A spy records calls for later assertions; it does not prove external delivery. Assert stable event fields and idempotency properties, avoid capturing secrets, and use integration tests for real transports.

## Terminology established

Spy, recorded call, post-condition evidence, stable field, publisher boundary, delivery semantics.

## Examples used

A notifier spy, registration event assertions, audit-log spy, and failing notifier for failure policy tests.

## Cross-references

- [Chapter 166 — Mocks](../../volumes/11-testing/166-mocks.md)
- [Chapter 164 — Contract Tests](../../volumes/11-testing/164-contract-tests.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 170 — Test Design: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XI proofread. Spy guidance links to PHPUnit, PHP, and Fowler.
