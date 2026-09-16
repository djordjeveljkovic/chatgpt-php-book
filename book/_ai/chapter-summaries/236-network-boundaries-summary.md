# AI Summary — Chapter 236 — Network Boundaries

- Status: complete
- Volume: Volume 16 — DISTRIBUTED SYSTEMS
- Last updated: 2026-09-16

## Written material

Explains network calls as process, ownership, timing, and failure boundaries. Covers synchronous/asynchronous work, boundary contracts, serialization and absent/null/value distinctions, narrow PHP ports and adapters, remote ambiguity, compatibility, ownership, security, and testing.

## Concepts already explained

Network boundary, transport representation, synchronous interaction, asynchronous interaction, ambiguous completion, contract version, and boundary adapter.

## Terminology established

Request identity, response schema, trust boundary, compatibility window, operation lookup, and response representation.

## Examples used

A network-path diagram, a typed invoice request and gateway, a vague generic client contrasted with a named billing client, and compatibility and failure test cases.

## Cross-references

Chapter 39 — Serialization, Chapter 151 — Deserialization, and Chapter 141 — Idempotency.

## Open threads

Continue with latency and time budgets in Chapter 237.

## Exact next section

Chapter 237 — Latency: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP examples. Local Markdown links resolved and git diff --check passed. Live network integration was not run.

## Writing notes

Preserves the book's distinction between PHP method calls and remote operations whose completion may be unknown.
