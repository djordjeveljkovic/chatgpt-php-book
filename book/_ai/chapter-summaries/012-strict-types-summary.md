# AI Summary — Chapter 12 — Strict Types

- Status: complete
- Volume: Volume 2 — PHP LANGUAGE FUNDAMENTALS
- Last updated: 2026-09-14

## Written material

Complete chapter explaining `declare(strict_types=1)`, caller-file behavior, weak versus strict scalar calls, return checks, conversion boundaries, value objects, database hydration, concurrency limits, security, testing, production rollout, exercises, review questions, and official references.

## Concepts already explained

Per-file strictness; scalar parameter and return checks; the int-to-float exception; `TypeError` as a contract failure; transport-to-domain conversion; strict types versus semantic validation, authorization, database constraints, and concurrency.

## Terminology established

Strict typing, weak/coercive typing, caller boundary, scalar declaration, type contract, conversion boundary, value object, return contract, contract failure.

## Examples used

Strict and weak callers of an integer function; positive integer and money parsers; `CourtId`; reservation command conversion; integer percentage calculation; prepared database hydration; strict-mode testing fixtures.

## Cross-references

Builds on Chapters 9–10 and Chapter 11’s coercion material; prepares operators and later database, security, testing, and runtime discussions; points to Volume IV for value and call-frame internals.

## Open threads

Gradual strictness migration in legacy code, static-analysis configuration, and broader type-system details belong in later volumes.

## Exact next section

Chapter 13 — Operators: the Why This Matters section.

## Technical verification notes

Version-sensitive claims checked against the PHP Manual type declarations and `declare` documentation and the PHP 7.0 scalar-type migration material. The chapter identifies scalar strictness as per-file and records the modern feature versions for typed properties, unions, intersections, and typed class constants.

## Writing notes

Status is complete. Preserve the distinction between strict representation checks and semantic/domain validation.
