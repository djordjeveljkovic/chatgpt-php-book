# AI Summary — Chapter 245 — Dead-Letter Queues

- Status: complete
- Volume: Volume 16 — DISTRIBUTED SYSTEMS
- Last updated: 2026-09-16

## Written material

Explains dead-letter queues as controlled failure paths rather than disposal. Covers failure classes, bounded failure envelopes, retry-to-DLQ flow, authorized replay, poison messages, deployment bugs, retention and privacy, PHP worker behavior, testing, and security. Includes a typed DeadLetter record.

## Concepts already explained

Dead-letter queue, poison message, quarantine, failure envelope, replay authorization, repair, discard, retention, and replay drill.

## Terminology established

Failure category, retry exhaustion, safe payload reference, replay rate, stop condition, failure-store capacity, and separate replay permission.

## Examples used

Failure classification, main-queue-to-DLQ flow, typed FailureCategory and DeadLetter, replay procedure, retention policy, and replay drills.

## Cross-references

Chapters 243 and 244, plus queue-performance guidance in Chapter 234.

## Open threads

Continue with backpressure and overload control in Chapter 246.

## Exact next section

Chapter 246 — Backpressure: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP example. Local Markdown links resolved and git diff --check passed. No live broker, replay, or retention test was run.

## Writing notes

Treats replay as an authorized, rate-limited write operation that preserves original identities.
