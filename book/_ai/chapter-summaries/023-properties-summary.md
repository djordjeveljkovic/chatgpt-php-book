# AI Summary — Chapter 23 — Properties

- Status: complete
- Volume: Volume 3 — PHP OBJECT MODEL
- Last updated: 2026-09-14

## Written material

The complete chapter covers instance/static properties; types, defaults, nullability, and uninitialized state; lifecycle-driven properties; typed properties, readonly, dynamic-property migration, property hooks/asymmetric visibility previews; reservation and interval examples; bad/better designs; edge cases; performance, security, and testing; mistakes; senior reasoning; exercises; review questions; and a summary.

## Concepts already explained

Properties, instance state, static state, typed properties, uninitialized state, nullable types, defaults, readonly, dynamic properties, property hooks, asymmetric visibility, ownership, lifecycle, aliasing, and mass assignment.

## Terminology established

`Match`, `ReservationState`, `User`, `Point`, `TimeSlot`, `ReservationRecord`, `Reservation`, and `Money` exercise design.

## Examples used

Status transition, typed point, interval invariant, persistence lifecycle, bad public reservation, and typed/private reservation examples.

## Cross-references

The chapter outline is defined in [SKELETON.md](../../../SKELETON.md).

## Open threads

Chapter 24 should build on property ownership and explain methods as behavioral contracts, `$this`, parameters/returns, side effects, static methods, and testable service behavior.

## Exact next section

Chapter 24 — Methods: the Why This Matters section.

## Technical verification notes

Claims about typed properties (PHP 7.4), readonly properties (PHP 8.1), dynamic-property deprecation (PHP 8.2), and property hooks/asymmetric visibility (PHP 8.4) were checked against the current official PHP Manual. Detailed readonly treatment is deferred to Chapter 34.

## Writing notes

Keep this summary short and update it after every writing session.
