# AI Summary — Chapter 21 — Classes

- Status: complete
- Volume: Volume 3 — PHP OBJECT MODEL
- Last updated: 2026-09-14

## Written material

The complete chapter now covers classes as named types and responsibility boundaries; namespaces and autoloading; declarations versus method execution; minimal, practical, and production reservation examples; bad/better boundaries; edge cases; performance, security, and concurrency implications; testing; mistakes; senior design reasoning; exercises; review questions; and a summary.

## Concepts already explained

Classes, objects, instances, types, namespaces, `$this`, invariants, autoloading, domain/application/infrastructure boundaries, final classes, static state, hidden I/O, and database-enforced concurrency.

## Terminology established

`Court`, `ReservationRequest`, `ReservationRepository`, `ReservationService`, `CreateReservation`, and `ConfirmationSender`.

## Examples used

`Court`, `ReservationRequest`, a repository/service split, and a reservation-confirmation orchestration example.

## Cross-references

The chapter outline is defined in [SKELETON.md](../../../SKELETON.md).

## Open threads

Chapter 22 should build on the distinction between a class definition and an object instance, then explain identity, assignment, passing, returning, comparison, and state ownership.

## Exact next section

Chapter 22 — Objects: the Why This Matters section.

## Technical verification notes

Claims about class declarations, `$this`, object identity/assignment, and object lifecycle were checked against the PHP Manual classes-and-objects and objects-and-references pages. Constructor promotion and readonly syntax are used as previews and are deferred to Chapters 23 and 26.

## Writing notes

Keep this summary short and update it after every writing session.
