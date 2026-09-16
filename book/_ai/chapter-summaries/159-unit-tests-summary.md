# AI Summary — Chapter 159 — Unit Tests

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Covers isolated behavior, PHPUnit structure, arrange-act-assert, strict assertions, injected seams, controlled clocks, boundary cases, builders, test doubles, over-mocking, and fast suite organization.

## Concepts already explained

Unit tests should preserve meaningful invariants while isolating slow or nondeterministic boundaries. Stubs, fakes, mocks, and integration tests serve different contracts.

## Terminology established

Unit, seam, arrange-act-assert, strict assertion, fake clock, test data builder, over-mocking, mutation testing.

## Examples used

Percentage value object, PHPUnit assertions, and an injectable Clock/ExpiryPolicy.

## Cross-references

- [Chapter 158 — Why Tests Exist](../../volumes/11-testing/158-why-tests-exist.md)
- [Chapter 165 — Test Doubles](../../volumes/11-testing/165-test-doubles.md)

## Exact next section

Chapter 160 — Integration Tests: the Why This Matters section.

## Technical verification notes

PHP and PHPUnit examples linted; framework behavior is identified as PHPUnit-specific where applicable.
