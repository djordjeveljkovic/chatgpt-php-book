# AI Summary — Chapter 24 — Methods

- Status: complete
- Volume: Volume 3 — PHP OBJECT MODEL
- Last updated: 2026-09-14

## Written material

The complete chapter explains methods as behavioral contracts; `$this`; parameters, returns, `void`, `never`, and static methods; commands versus queries; side effects and retries; shopping-cart, interval, clock, result, and named-constructor examples; bad/better designs; edge cases; performance, security, and testing; mistakes; senior reasoning; exercises; review questions; and a summary.

## Concepts already explained

Instance methods, method contracts, commands, queries, `$this`, rebinding, `void`, `never`, static methods, named constructors, side effects, idempotency, injected clocks, result objects, and database atomicity.

## Terminology established

`Temperature`, `Reservation`, `ShoppingCart`, `TimeSlot`, `Clock`, `ConfirmReservation`, `ReservationResult`, `ReservationApplication`, and `EmailAddress`.

## Examples used

Reservation state transitions, cart accumulation, half-open interval overlap, clock injection, atomic reservation result, and validated email factory.

## Cross-references

The chapter outline is defined in [SKELETON.md](../../../SKELETON.md).

## Open threads

Chapter 25 should explain visibility as an API/ownership boundary, including public/protected/private, constants, inheritance implications, and PHP 8.4 asymmetric property visibility.

## Exact next section

Chapter 25 — Visibility: the Why This Matters section.

## Technical verification notes

Method contracts, object parameters, static methods, and named constructors follow PHP language behavior. The chapter intentionally treats exception and retry policy as practical contracts because PHP signatures do not declare thrown exceptions.

## Writing notes

Keep this summary short and update it after every writing session.
