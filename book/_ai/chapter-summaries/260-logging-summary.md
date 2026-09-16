# AI Summary — Chapter 260 — Logging

- Status: complete
- Volume: Volume 17 — PRODUCTION ENGINEERING
- Last updated: 2026-09-16

## Written material

Explains structured logging, levels, correlation, request and business events, exception handling, volume and cost, collection and delivery, PHP workers, privacy, security, testing, and retention. Includes a typed Logger boundary and redaction helper.

## Concepts already explained

Structured event, correlation ID, operation ID, audit record, log sampling, bounded field, redaction, retention, and log delivery failure.

## Terminology established

Stable event name, upstream timing, diagnostic reference, log cardinality, failure category, and audit boundary.

## Examples used

Structured JSON event, typed Logger, redact and logProviderTimeout functions, asynchronous correlation, sampling, retention, and secret-absence tests.

## Cross-references

Chapters 254 and 262.

## Open threads

Continue with metrics and quantitative operational signals in Chapter 261.

## Exact next section

Chapter 261 — Metrics: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP examples. Local Markdown links resolved and git diff --check passed. No live collector or log backend was run.

## Writing notes

Separates operational logs from metrics, traces, audit records, and durable business state.
