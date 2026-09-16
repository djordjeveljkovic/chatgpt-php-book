# AI Summary — Chapter 280 — Tennis Reservation Service

- Status: complete
- Volume: Volume XIX — SMALL ENGINEERING PROJECTS
- Last updated: 2026-09-16

## Written material

Chapter 280 incrementally designs a tennis reservation service. It defines a product contract, models facility-local input and UTC instants, uses half-open intervals, separates policy from persistence and HTTP, explains why check-then-insert races, enforces conflicts through a shared transaction/locking strategy, scopes durable idempotency keys, validates authorization and input, delivers notifications through an outbox, tests concurrency and unknown provider outcomes, observes contention, and plans rollout and recovery.

## Concepts already explained

Facility time zone, UTC instant, half-open interval, local-time ambiguity, opening-hour policy, lead-time and booking horizon, overlap predicate, conflict invariant, court serialization point, lock-row strategy, active reservation state, durable idempotency key, request fingerprint, idempotency replay, outbox notification, unknown provider outcome, bounded contention metrics, and forward recovery after committed effects.

## Terminology established

Northstar Tennis Club, court, `TimeSlot`, `ReservationRequest`, `ReservationStore`, `ReservationService`, `ReservationPolicy`, `ExportOutcome` cross-domain comparison, `outbox`, and `court serialization point`.

## Examples used

- A fixed-duration club reservation contract with a facility time zone, opening hours, lead time, horizon, cancellation ownership, and post-commit confirmation.
- A typed `TimeSlot` with half-open overlap logic and local-to-UTC parsing.
- A typed reservation request, repository port, and application service with idempotency and early conflict checks.
- A court-row locking transaction sequence with overlap verification, reservation insertion, and outbox creation.
- Boundary, authorization, idempotency, concurrency, retry, and notification test cases.
- Bounded metrics for conflicts, lock waits, retries, idempotency replays, and outbox age.

## Cross-references

The chapter links to Chapters 114, 116, and 120 for transactions, locks, and data access; Chapters 136 and 141 for effects and HTTP idempotency; Chapter 158 for testing; Chapters 223 and 261 for performance and metrics; Chapters 241–243 for partial failure and message delivery; Chapter 265 for rollback; and Chapters 277–279 for migration and case-study continuity.

## Open threads

Continue Volume XIX with Chapter 281 — Rate Limiter, carrying forward explicit invariants, bounded state, concurrency, idempotency, observability, and recovery into admission control.

## Exact next section

Chapter 281 — Rate Limiter: the Why This Matters section.

## Technical verification notes

The PHP examples use `DateTimeImmutable`, `DateTimeZone`, enums/readonly-style typed domain objects, and should be linted with PHP 8.2 or newer. The interval predicate and date parsing require boundary tests, including adjacent slots and daylight-saving transitions. Database locking and uniqueness require a real integration test; mocked repositories cannot prove serialization.

## Writing notes

Keep time semantics explicit: local policy is resolved in the venue zone, durable instants are compared consistently, and ambiguous/nonexistent local times require a product decision. Distinguish early application checks from database-enforced invariants and provider unknowns from safe failures.
