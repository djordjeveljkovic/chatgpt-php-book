# AI Summary — Chapter 220 — Symfony HttpKernel

- Status: complete
- Volume: Volume 14 — LARAVEL AND SYMFONY
- Last updated: 2026-09-16

## Written material

Explains HttpKernel request-to-response flow, kernel contract, request/controller/response/exception/termination events, subscribers, listener priorities, main versus sub-requests, failure mapping, testing, and operations.

## Concepts already explained

HttpKernel coordinates lifecycle stages; listeners should implement focused boundary policies rather than hide domain workflows. Required durability cannot depend on termination work, and event registration and ordering require kernel integration tests.

## Terminology established

HttpKernel, kernel event, main request, sub-request, listener priority, response subscriber, exception mapping, termination work.

## Examples used

A kernel dispatch function, locale request subscriber, response/exception subscriber guidance, and a migration from termination email to an outbox-backed queue.

## Cross-references

- [Chapter 217 — Symfony Overview](../../volumes/14-laravel-and-symfony/217-symfony-overview.md)
- [Chapter 218 — Symfony Components](../../volumes/14-laravel-and-symfony/218-symfony-components.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 221 — Symfony Messenger: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XIV proofread. HttpKernel guidance links to official Symfony documentation.
