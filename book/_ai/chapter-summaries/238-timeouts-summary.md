# AI Summary — Chapter 238 — Timeouts

- Status: complete
- Volume: Volume 16 — DISTRIBUTED SYSTEMS
- Last updated: 2026-09-16

## Written material

Explains connect, handshake, write, read, total, lease, and lock timeouts; monotonic deadlines; deadline propagation; timeout ambiguity and cancellation; choosing coherent values; timeout storms; PHP resource cleanup; security; and deterministic testing.

## Concepts already explained

Phase timeout, total deadline, remaining budget, unknown completion, cancellation boundary, timeout storm, lease timeout, and cleanup boundary.

## Terminology established

Monotonic deadline, parent deadline, response-progress timeout, queue visibility timeout, deadline margin, and ambiguous result.

## Examples used

A deadline timeline, typed PHP Deadline helper, timed-out payment sequence, coherent timeout relationships, and finally-based cleanup.

## Cross-references

Chapter 141 — Idempotency and Chapters 237 and 239 in this volume.

## Open threads

Continue with retry classification and budgets in Chapter 239.

## Exact next section

Chapter 239 — Retries: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP examples. Local Markdown links resolved and git diff --check passed. Live timeout and cancellation tests were not run.

## Writing notes

Emphasizes that a local timeout does not prove a remote side effect was cancelled.
