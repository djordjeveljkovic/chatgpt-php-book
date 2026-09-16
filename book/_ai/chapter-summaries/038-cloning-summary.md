# AI Summary — Chapter 38 — Cloning

- Status: complete
- Volume: Volume 3 — PHP OBJECT MODEL
- Last updated: 2026-09-14

## Written material

Complete chapter on object identity, assignment versus clone, shallow copy, `__clone()`, ownership, deep-copy limits, readonly clone behavior, database identity, concurrency, performance, security, testing, exercises, and review questions.

## Concepts already explained

Object alias, outer identity, shallow copy, owned subgraph, shared immutable value, identity reset, copy-with-changes.

## Terminology established

Clone policy, ownership boundary, external identity, PHP snapshot versus database snapshot.

## Examples used

Draft, ReservationDraft, LineItems, ReservationRequest, ServiceState, and clone ownership tests.

## Cross-references

Builds on Chapters 22, 23, 26, 34, and 37; connects to Chapter 39 serialization and later persistence/concurrency chapters.

## Open threads

Connect object cloning to object handles, references, and memory behavior in Volume IV.

## Exact next section

Chapter complete; next chapter is Chapter 39 — Serialization.

## Technical verification notes

Official PHP Manual references included. Version note covers readonly property reinitialization during `__clone()` from PHP 8.3.
