# AI Summary — Chapter 221 — Symfony Messenger

- Status: complete
- Volume: Volume 14 — LARAVEL AND SYMFONY
- Last updated: 2026-09-16

## Written material

Covers Messenger messages and handlers, buses and routing, synchronous versus asynchronous dispatch, transactions and outbox timing, retries, failure transports, middleware and stamps, workers, serialization compatibility, observability, security, and testing.

## Concepts already explained

A transport is a distributed boundary with at-least-once delivery, serialization, retry, ordering, and worker lifecycle concerns. Keep messages small and versioned, make handlers idempotent, coordinate dispatch with commits, and re-check authorization at handling time.

## Terminology established

Message, handler, bus, Envelope, stamp, transport, failure transport, at-least-once delivery, outbox, operation ID, dispatch-after-commit, worker lifecycle.

## Examples used

GenerateInvoicePdf message and handler, InvoiceApplicationService bus dispatch, middleware/stamp policy, failure classification, and worker observability.

## Cross-references

- [Chapter 215 — Laravel Queues](../../volumes/14-laravel-and-symfony/215-laravel-queues.md)
- [Chapter 206 — Distributed Systems](../../volumes/13-architecture/206-distributed-systems.md)

## Exact next section

Chapter 222 — Laravel vs Symfony: the Why This Matters section.

## Technical verification notes

PHP examples linted; Symfony Messenger, Serializer, and microservices.io references are linked using version-neutral documentation URLs.
