# AI Summary — Chapter 25 — Visibility

- Status: complete
- Volume: Volume 3 — PHP OBJECT MODEL
- Last updated: 2026-09-14

## Written material

The complete chapter covers public/protected/private visibility for properties and methods, class constants, encapsulation, inheritance scope, protected extension seams, PHP 8.4 asymmetric property visibility, property-hook cautions, authorization distinction, bad/better user and notification examples, edge cases, performance, security, testing, mistakes, senior reasoning, exercises, review questions, and a summary.

## Concepts already explained

Visibility, API surface, encapsulation, public/protected/private, class constants, protected extension seam, private declaration scope, asymmetric property visibility, `private(set)`, property hooks, authorization, representation, and inheritance coupling.

## Terminology established

`Account`, `Reservation`, `Base`/`Child`, `ReservationStatus`, `Wallet`, `Notification`, `ImportReport`, `User`, and `Actor`.

## Examples used

Controlled reservation status, wallet balance, notification template hook, PHP 8.4 import counter, and role-promotion authorization.

## Cross-references

The chapter outline is defined in [SKELETON.md](../../../SKELETON.md).

## Open threads

Chapter 26 should explain constructors as initialization and invariant boundaries, constructor promotion, dependency injection, validation/failure, inheritance caveats, factories, and testability.

## Exact next section

Chapter 26 — Constructors: the Why This Matters section.

## Technical verification notes

Visibility behavior and PHP 8.4 asymmetric property visibility were checked against the official PHP Manual. The chapter distinguishes visibility from authorization and treats property hooks as a preview rather than a substitute for domain methods.

## Writing notes

Keep this summary short and update it after every writing session.
