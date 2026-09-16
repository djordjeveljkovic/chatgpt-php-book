# AI Summary — Chapter 161 — Feature Tests

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Explains application-flow tests through routing, middleware, authentication, authorization, validation, persistence, serialization, sessions, asynchronous work, controlled providers, and deterministic timing.

## Concepts already explained

Feature tests should send real application requests and assert public contracts plus durable effects. They need isolated state, high-risk negative cases, and bounded asynchronous coordination.

## Terminology established

Feature test, application boundary, scenario, public contract, durable effect, controlled provider, bounded polling.

## Examples used

A framework-shaped create-order feature test and cases for authentication, validation, idempotency, logout, and provider failure.

## Cross-references

- [Chapter 158 — Why Tests Exist](../../volumes/11-testing/158-why-tests-exist.md)
- [Chapter 162 — API Tests](../../volumes/11-testing/162-api-tests.md)

## Exact next section

Chapter 162 — API Tests: the Why This Matters section.

## Technical verification notes

PHP example linted; framework helper methods are explicitly identified as illustrative pseudocode.
