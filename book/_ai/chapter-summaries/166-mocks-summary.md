# AI Summary — Chapter 166 — Mocks

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Explains mocks as predeclared interaction verifiers, expectations and cardinality, argument matching, failures, asynchronous boundaries, overspecification, testing, exercises, and review questions.

## Concepts already explained

Mocks protect meaningful collaborator protocols; they do not prove adapter, transport, or asynchronous delivery behavior. Mock only stable side-effect boundaries and avoid asserting incidental implementation details.

## Terminology established

Mock, expectation, cardinality, argument matcher, interaction contract, overspecified test.

## Examples used

PHPUnit payment-gateway mock with argument and idempotency-key expectations, plus a provider-failure test.

## Cross-references

- [Chapter 165 — Test Doubles](../../volumes/11-testing/165-test-doubles.md)
- [Chapter 167 — Stubs](../../volumes/11-testing/167-stubs.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 167 — Stubs: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XI proofread. PHPUnit guidance links to official documentation and Fowler's test-double explanation.
