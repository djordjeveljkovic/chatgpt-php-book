---
book: The Complete Modern PHP Engineering Book
volume: 19
volume_title: SMALL ENGINEERING PROJECTS
chapter: 280
title: Tennis Reservation Service
slug: tennis-reservation-service
status: complete
summary: ../../_ai/chapter-summaries/280-tennis-reservation-service-summary.md
---

# Chapter 280 — Tennis Reservation Service

## Why This Matters

A tennis reservation service sounds like a small CRUD application: choose a court, choose a time, save a booking. The difficult part is the boundary between two requests that arrive close together. Both may see an apparently free court, both may try to reserve it, and one must lose without corrupting the schedule or charging a customer twice.

This project is useful because it concentrates several engineering habits in a small domain. We need a precise time model, explicit invariants, validation at trust boundaries, a transaction and uniqueness strategy, an idempotent command, authorization, and an operational view of contention. The feature is small enough to hold in your head and serious enough to punish vague contracts.

## Define the First Contract

Start with a bounded product contract rather than a table named `reservations`:

~~~text
facility: Northstar Tennis Club
courts: named courts with an active/inactive state
opening hours: 07:00–22:00 in the club's IANA time zone
slot rule: reservations are fixed 60-minute half-open intervals
lead time: booking must start at least 30 minutes from now
horizon: booking may start no more than 14 local days ahead
ownership: a member may cancel only their own future reservation
conflict: one court cannot have overlapping active reservations; active means held or confirmed
command identity: client retry uses an idempotency key
effect: confirmation notification is sent only after commit
~~~

The contract deliberately leaves policy visible. Is a 30-minute booking allowed? Can a reservation cross midnight? What happens when daylight-saving time changes the length of a local day? These are domain decisions, not formatting details.

## Model Time Precisely

Store instants in UTC and retain the club time zone for presentation and local policy. Parse a member’s local input in the facility zone, validate the local date and opening hours, then convert the resulting instants to UTC. Do not compare formatted strings or server-local timestamps.

Represent a slot as a half-open interval `[start, end)`. Two reservations overlap exactly when:

~~~text
existing.start < requested.end
and requested.start < existing.end
~~~

Thus `[10:00, 11:00)` and `[11:00, 12:00)` are adjacent, not conflicting. Inclusive-end comparisons incorrectly reject valid back-to-back bookings.

~~~php
<?php

declare(strict_types=1);

final readonly class TimeSlot
{
    public function __construct(
        public DateTimeImmutable $start,
        public DateTimeImmutable $end,
    ) {
        if ($end <= $start) {
            throw new InvalidArgumentException('A slot must end after it starts');
        }
    }

    public function overlaps(self $other): bool
    {
        return $this->start < $other->end
            && $other->start < $this->end;
    }
}

function parseClubSlot(
    string $date,
    string $startTime,
    DateTimeZone $clubZone,
    DateTimeImmutable $now,
): TimeSlot {
    // Incomplete by design: the caller must reject or resolve DST gaps and folds first.
    $start = DateTimeImmutable::createFromFormat(
        '!Y-m-d H:i',
        "$date $startTime",
        $clubZone,
    );

    $errors = DateTimeImmutable::getLastErrors();
    if ($start === false || ($errors !== false && ($errors['warning_count'] > 0 || $errors['error_count'] > 0))) {
        throw new InvalidArgumentException('Invalid local date or time');
    }

    return new TimeSlot($start->setTimezone(new DateTimeZone('UTC')), $start->modify('+60 minutes')->setTimezone(new DateTimeZone('UTC')));
}
~~~

The example establishes representation, not the whole policy. It still needs opening-hour, lead-time, horizon, and daylight-saving checks. A local time that occurs twice or does not occur on a clock change must have an explicit product policy; silently accepting PHP’s normalization can book a different instant than the member selected.

## Separate Policy from Persistence

Keep the reservation rule independent of HTTP and SQL. A policy can answer whether a requested slot is eligible; a repository can find existing reservations; an application service can coordinate authorization, idempotency, transaction, and notification.

~~~php
<?php

declare(strict_types=1);

final readonly class ReservationRequest
{
    public function __construct(
        public string $memberId,
        public string $courtId,
        public TimeSlot $slot,
        public string $idempotencyKey,
    ) {
        if ($memberId === '' || $courtId === '' || $idempotencyKey === '') {
            throw new InvalidArgumentException('Reservation request is incomplete');
        }
    }

    public function fingerprint(): string
    {
        return hash('sha256', implode('|', [
            $this->memberId,
            $this->courtId,
            $this->slot->start->format(DateTimeInterface::ATOM),
            $this->slot->end->format(DateTimeInterface::ATOM),
        ]));
    }
}

interface ReservationStore
{
    public function findByIdempotency(string $memberId, string $key): ?Reservation;

    public function hasOverlap(string $courtId, TimeSlot $slot): bool;

    public function insert(Reservation $reservation): void;
}

final readonly class ReservationService
{
    public function __construct(
        private ReservationStore $store,
        private ReservationPolicy $policy,
    ) {
    }

    public function reserve(ReservationRequest $request, DateTimeImmutable $now): Reservation
    {
        $existing = $this->store->findByIdempotency($request->memberId, $request->idempotencyKey);
        if ($existing !== null) {
            if ($existing->requestFingerprint() !== $request->fingerprint()) {
                throw new IdempotencyConflict('Key was reused with different input');
            }

            return $existing;
        }

        $this->policy->assertAllowed($request, $now);
        if ($this->store->hasOverlap($request->courtId, $request->slot)) {
            throw new ReservationConflict('Court is already reserved');
        }

        $reservation = Reservation::new($request);
        $this->store->insert($reservation);

        return $reservation;
    }
}
~~~

The service reads clearly, but the check-then-insert sequence is not safe under concurrency by itself. The repository contract must be implemented inside a transaction with a database strategy that serializes conflicting reservations. Application-level `hasOverlap()` is useful for an early error and a test seam; it is not the final invariant. The stored reservation exposes the canonical request fingerprint used above; it is not recomputed from mutable display fields.

## Enforce the Conflict Invariant

The database strategy depends on the engine and schema. Options include a range-exclusion constraint where the database supports it, a per-court lock row, or a serializable transaction with a carefully designed conflict query. A plain `SELECT` followed by `INSERT` under a default isolation level can allow two writers through. Define the conflict predicate over every active reservation state—here, `held` and `confirmed`—and make cancellation or expiration remove a row from that predicate only through an authorized state transition.

A portable lock-row approach gives each court a stable serialization point:

1. begin a short transaction;
2. lock the court row in a canonical order;
3. check active reservations for the half-open overlap predicate;
4. insert the reservation if no conflict exists;
5. insert an outbox notification record;
6. commit;
7. publish or deliver the notification from the outbox worker.

The lock must cover every writer, including an administrative tool. If another path inserts directly into the reservation table without taking the same lock, the application has two incompatible concurrency contracts. Chapter 116 covers locks; Chapter 277 covers schema transitions and hidden writers.

Add uniqueness for idempotency, such as `(member_id, idempotency_key)`, and store enough request identity to decide whether a retry is the same command. A reused key with a different court or slot should be rejected as a conflict, not return the first result accidentally. The key needs a retention policy long enough to cover the client retry window and any delayed response.

## Validate at the Boundary

An HTTP handler should parse untrusted values and then call a domain operation. It should not decide overlap rules by string comparison or trust a member ID supplied by the browser.

Validate:

* authenticated member identity and permission to reserve;
* a known active court;
* strict date and time syntax;
* facility time-zone interpretation;
* opening hours, slot duration, lead time, and booking horizon;
* idempotency-key length, character set, and ownership scope;
* request size and rate limits;
* cancellation identity and state.

Return a stable error category such as `invalid_request`, `court_unavailable`, `duplicate_request`, or `temporarily_unavailable`. Do not expose SQL errors, lock details, or whether another member holds a private reservation unless the product intentionally permits that information.

## Notifications Are a Separate Boundary

Do not send email or push notifications inside the transaction before the reservation commits. If the provider succeeds and the database rolls back, the member receives a false confirmation. If the database commits and the process dies before sending, the member receives no confirmation.

Use a durable outbox record committed with the reservation. The worker claims records with a lease, sends using a stable message or notification identity, records the provider result, and retries only according to the provider’s contract. An ambiguous timeout is unknown completion. The worker should reconcile or use a provider idempotency key before sending again. Chapters 241–243 cover partial failure, idempotency, and message delivery.

## Test the Edges First

The highest-value tests are not only “a reservation appears.” Include:

| Case | Expected result |
| --- | --- |
| 10:00–11:00 then 11:00–12:00 | both succeed |
| 10:00–11:00 then 10:59–11:59 | second conflicts |
| different courts, same interval | both succeed |
| malformed or nonexistent local time | invalid request or explicit policy result |
| before opening or after closing | invalid request |
| same idempotency key, same request | original result |
| same key, changed slot | conflict |
| two concurrent requests, same court/slot | exactly one reservation |
| unauthorized cancellation | forbidden, no state change |
| worker retry after ambiguous notification | no duplicate effect |

Use a fixed clock and facility zone in unit tests. Use a real database integration test for locking, uniqueness, transaction boundaries, and rollback. A mock repository cannot prove that two real transactions serialize correctly. Test the HTTP contract separately from the domain policy, and test the outbox worker with a provider fake that can return accepted, rejected, and unknown outcomes.

## Observe Contention and Capacity

Record reservation attempt outcome, court identifier, duration bucket, correlation ID, transaction retry count, lock wait, and outbox age. Avoid logging member names or unrestricted request values. Court ID is bounded only if the facility set is bounded; otherwise apply the same cardinality discipline used in Chapter 261.

Watch:

* conflict ratio by court and time window;
* transaction and lock-wait latency;
* database deadlocks and serialization retries;
* idempotency replay rate;
* outbox age, delivery failures, and unknown provider outcomes;
* reservation capacity and API rate-limit responses.

A sudden increase in conflicts may indicate a popular court, a client retry storm, or a bug that rounds every request into one slot. Metrics should help distinguish these cases without turning member input into labels.

## Rollout and Change Safety

For an existing club application, add the reservation API beside the old booking page. Read old reservations through an adapter, characterize state and cancellation behavior, and choose one authority for writes. If a new slot representation is needed, expand the schema first and keep old readers tolerant of the new columns.

Deploy the conflict invariant before enabling traffic that relies on it. During a mixed-version window, old writers must take the same court lock or be fenced from the affected courts. A feature flag that routes requests to the new handler does not control an unobserved admin script.

If a release fails, route new requests to the known-good implementation only when both versions can interpret the same reservations and idempotency records. Do not restore an old binary that treats a committed reservation as absent or reuses an operation key with different meaning. Drain workers, reconcile outbox records, and choose forward repair when a notification or reservation has already committed.

## Common Mistakes

* Storing local wall-clock strings without a facility time zone.
* Treating an end time as inclusive and rejecting adjacent reservations.
* Letting PHP’s date normalization silently choose a nonexistent or ambiguous local time.
* Relying on `hasOverlap()` without a transaction and database invariant.
* Locking courts in inconsistent order when a request can reserve more than one resource.
* Accepting a reused idempotency key for a changed request.
* Sending confirmation before the reservation commits.
* Retrying an unknown provider result as if the first attempt definitely failed.
* Testing only the happy path or only a mocked repository.
* Logging member identity or raw request data in high-cardinality metrics.
* Enabling the new route while an untracked admin or repair tool remains an old writer.
* Assuming code rollback reverses a committed reservation or notification.

## Senior Engineer Thinking

The senior question is not “can we insert a reservation?” It is “what exactly is a time slot, which writers share the conflict invariant, how is concurrency serialized, what does a client retry mean, and what evidence remains when the notification result is unknown?”

Small projects are valuable because they expose the complete path from input to durable state to external effect. Keep time, authority, idempotency, transaction scope, and recovery explicit even when the codebase is only a few classes. The same habits scale to scheduling, inventory, payments, and distributed workflows.

## Exercises

1. Define the facility’s daylight-saving policy for a nonexistent and an ambiguous local time. Write example inputs and expected instants.
2. Implement overlap checks for adjacent, containing, and equal intervals, then compare them with a brute-force oracle over minute slots.
3. Design a database-backed `ReservationStore` using a court lock row. State the transaction, index, isolation, retry, and timeout policy.
4. Add idempotency records that reject the same key with a different request fingerprint. Specify retention and cleanup.
5. Simulate a committed reservation followed by an outbox-worker timeout. Describe the reconciliation and retry evidence.

## Review Questions

* Why is a tennis reservation service more than CRUD?
* Why should instants be stored in UTC while local policy uses the facility time zone?
* What does half-open interval notation change at adjacent boundaries?
* Why is an application overlap check insufficient under concurrency?
* Which writers must share the court serialization strategy?
* What should happen when an idempotency key is reused with changed input?
* Why does the notification belong in an outbox flow?
* Which tests require a real database rather than a mock?
* How can metrics distinguish contention from a retry storm safely?
* When is code rollback unsafe after a reservation or notification commits?

## Summary

The tennis reservation service is a compact exercise in precise domain and systems design. Model local times and UTC instants deliberately, use half-open intervals, separate policy from transport and persistence, enforce overlap and idempotency invariants in a shared transaction strategy, validate authorization and input at the boundary, deliver notifications through an outbox, test concurrency and unknown outcomes, observe contention with bounded dimensions, and roll out only when old and new writers share compatible contracts.

## References

- [PHP Manual: DateTimeImmutable](https://www.php.net/manual/en/class.datetimeimmutable.php)
- [PHP Manual: Date and Time Formats](https://www.php.net/manual/en/datetime.formats.php)
- [Chapter 114 — Transactions](../08-databases/114-transactions.md)
- [Chapter 116 — Locks](../08-databases/116-locks.md)
- [Chapter 120 — Large Datasets](../08-databases/120-large-datasets.md)
- [Chapter 136 — Webhooks](../09-http-and-application-development/136-webhooks.md)
- [Chapter 141 — Idempotency](../09-http-and-application-development/141-idempotency.md)
- [Chapter 158 — Why Tests Exist](../11-testing/158-why-tests-exist.md)
- [Chapter 223 — Performance Mental Model](../15-performance/223-performance-mental-model.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 242 — Idempotency](../16-distributed-systems/242-idempotency.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 277 — Database Migration](../18-legacy-php/277-database-migration.md)
- [Chapter 279 — Legacy Case Study](../18-legacy-php/279-legacy-case-study.md)
