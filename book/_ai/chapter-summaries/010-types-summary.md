# AI Summary — Chapter 10 — Types

- Status: complete
- Volume: Volume 2 — PHP LANGUAGE FUNDAMENTALS
- Last updated: 2026-09-14

## Written material

Complete chapter covering runtime types versus domain invariants, built-in types, scalar precision, arrays as lists/maps/records, object and interface identity, nullability, union/intersection/DNF types, `mixed`, `void`, `never`, `iterable`, `callable`, enums, typed properties/constants, Zend type checks, reservation parsing, money modeling, database serialization, security, concurrency, testing, common mistakes, senior reasoning, exercises, review questions, and summary.

## Concepts already explained

Types as runtime categories and declaration gates; semantic validation as a separate responsibility; PHP’s built-in values; PHPDoc array shapes; object/interface contracts; enum conversion; nullable results; unions, intersections, and DNF; uninitialized typed properties; type-check failures; native type limits.

## Terminology established

Runtime type, domain invariant, type declaration, nullable type, union type, intersection type, DNF type, backed enum, typed property, uninitialized property, `mixed`, `void`, `never`, `iterable`, `callable`, value object.

## Examples used

Type inspection; `Clock` interface; backed `Surface` enum; union/intersection/DNF declarations; `CourtId` and `ReservationInput` value objects; repository nullable result; weak payment strings; `Money`/`Currency` improvement; prepared database serialization.

## Cross-references

Builds on Chapter 7's contracts and boundaries, Chapter 8's syntax, and Chapter 9's variable state. Prepares Chapter 11 on type juggling, Chapter 12 on strict types, Chapter 17 on arrays, and Volume IV on zvals and runtime type representation.

## Open threads

Scalar coercion and comparison behavior belong to Chapter 11; caller-side strict typing belongs to Chapter 12; array structure and complexity belong to Chapter 17; detailed Zend value internals belong to Volume IV.

## Exact next section

Chapter 11 — Type Juggling: the Why This Matters section.

## Technical verification notes

Claims and examples were checked against current PHP 8.x manual material: built-in types, `get_debug_type()`, type declarations, nullable/union/intersection/DNF rules, `mixed`, `iterable`, `never`, typed properties, and enums. Version-specific features should still be checked against the project’s minimum PHP version.

## Writing notes

Status is complete. Preserve the distinction between native type shape and semantic/domain validation in later chapters.
