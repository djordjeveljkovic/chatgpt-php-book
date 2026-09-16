# AI Summary — Chapter 20 — Exceptions

- Status: complete
- Volume: Volume 2 — PHP LANGUAGE FUNDAMENTALS
- Last updated: 2026-09-14

## Written material

The complete chapter explains exceptions as abrupt control flow for failed promises. It covers the Throwable hierarchy, propagation, finally, cause chaining, custom taxonomy, engine behavior, expected results versus exceptional failures, boundary translation, transactions and external effects, edge cases, complexity, security, database and concurrency concerns, testing, common mistakes, senior-engineer reasoning, exercises, and review questions.

## Concepts already explained

- Exception, Error, and Throwable; specific catch ordering
- Stack unwinding, cleanup, finally hazards, previous exceptions
- Domain outcomes versus dependency failures, retryability, idempotency
- Transaction ownership, concurrency constraints, worker reset, diagnostic redaction

## Terminology established

Throwable hierarchy, exception taxonomy, propagation, stack unwinding, previous exception, boundary translation, retryable failure, idempotency, transaction owner.

## Examples used

Reservation service classification; HTTP boundary translation; order/outbox side-effect analysis; unsafe default-value catch; typed price validation; exception and cleanup tests.

## Cross-references

Builds on Chapter 19 error channels and the request/response model; prepares for classes, databases, distributed systems, testing, and production engineering.

## Open threads

Later chapters can define richer domain error types, framework exception handlers, retry policies, and incident/observability workflows.

## Exact next section

Chapter complete; no next section.

## Technical verification notes

The chapter relies on PHP's documented Throwable hierarchy, propagation, catch matching, finally behavior, and lack of checked exceptions. Internal VM wording is intentionally high-level; transaction and retry claims are application-boundary guidance.
