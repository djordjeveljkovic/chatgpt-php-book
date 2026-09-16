---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 249
title: Distributed Locks
slug: distributed-locks
status: complete
summary: ../../_ai/chapter-summaries/249-distributed-locks-summary.md
---

# Chapter 249 — Distributed Locks

## Why This Matters

A distributed lock coordinates processes that do not share memory. It can prevent duplicate cache refreshes or serialize a maintenance task, but it is easy to mistake “I acquired a lock” for “my operation is safe.” A crashed process, expired lease, network partition, delayed packet, or paused runtime can make an old holder continue after another process acquires the same name.

Use a lock only when the protected invariant and failure behavior are explicit. If a database constraint, atomic update, queue partition, or idempotency record can enforce the requirement, it is often a clearer authority than a general-purpose lock.

## Safety and Liveness

Two properties matter:

* **safety:** incompatible owners are not both allowed to commit a protected effect;
* **liveness:** a failed owner does not block progress forever.

A lease improves liveness by expiring, but expiration does not automatically revoke code that is still running. The protected resource must reject stale owners, or the operation must be harmless when repeated.

## Lease Lifecycle

```text
acquire → work while lease valid → release
    ↓             ↓
  denied      renew or expire
```

The owner needs a unique token, not just a lock key. Release must delete only the record owned by that token. A process that accidentally releases another owner's lock can create overlapping work.

Lease duration should exceed normal work with margin but remain bounded. Renewal must be authenticated and conditional on ownership. A long pause can make a lease appear valid locally while the authority has already expired it.

## Fencing Tokens

A fencing token is a monotonically increasing number issued on each successful acquisition. The protected resource stores or compares the token and rejects operations from an older token:

```text
owner A acquires token 41
owner A pauses
lease expires
owner B acquires token 42
owner B writes with 42 → accepted
owner A resumes and writes with 41 → rejected
```

Fencing moves safety into the resource that accepts the effect. A lock service alone cannot stop a paused client from sending a late write to a database, filesystem, or provider that ignores ownership.

## A Lock Port

Keep acquisition, token, and conditional release visible:

~~~php
<?php

declare(strict_types=1);

final readonly class LockLease
{
    public function __construct(
        public string $name,
        public string $ownerToken,
        public int $fencingToken,
        public int $expiresAtMs,
    ) {
        if ($name === '' || $ownerToken === '' || $fencingToken < 1) {
            throw new InvalidArgumentException('Invalid lock lease');
        }
    }
}

interface LockStore
{
    public function acquire(string $name, int $ttlMs): ?LockLease;
    public function release(LockLease $lease): void;
}

function withLock(LockStore $store, string $name, callable $work): mixed
{
    $lease = $store->acquire($name, 5_000);
    if ($lease === null) {
        throw new ResourceBusy('Lock unavailable');
    }

    try {
        return $work($lease);
    } finally {
        $store->release($lease);
    }
}
~~~

The storage implementation must perform acquisition and conditional release atomically. The work must check remaining lease time or renew, and the protected write should carry `fencingToken` where stale owners are possible. A `finally` block is necessary but cannot release a lease whose authority has already expired for another owner.

## Lock Granularity

A global lock is simple but destroys parallelism and creates a single hot point. A per-tenant or per-resource lock improves concurrency but can create many keys, deadlocks, and fairness problems. Choose granularity from the invariant:

```text
global report generation       → one lock may be sufficient
one customer's balance         → customer-scoped serialization
unique email address           → database unique constraint
cache refresh                  → short lock plus stale fallback
```

Do not lock a larger scope than the invariant requires. Do not lock a smaller scope than the invariant requires and assume application discipline will hold across every writer.

## Lock Ordering and Deadlocks

Acquiring two locks in inconsistent order creates a cycle:

```text
request A: lock customer → lock invoice
request B: lock invoice → lock customer
```

Use a canonical order, reduce lock duration, and define a bounded retry for an explicitly detected deadlock. Release all locks before backoff. See [Chapter 117 — Deadlocks](../../volumes/08-databases/117-deadlocks.md).

## What a Lock Does Not Do

A lock does not make a remote operation atomic with local state. It does not survive a lock-store outage unless the design defines the behavior. It does not prove that the holder is healthy, and it does not repair work after a crash.

For a payment, prefer provider idempotency and a durable operation record. For inventory, use the owning database's constraint and transaction. For a scheduled singleton task, a lease can be appropriate if duplicate execution is safe and a stale worker cannot commit destructive results.

## PHP and Process Pauses

PHP workers can be delayed by blocking I/O, CPU scheduling, garbage collection, process suspension, or host pressure. A local elapsed-time check is not enough to prove lease validity; the lock authority decides. Long work should renew conditionally or stop before expiry.

Do not hold a lock while making an unbounded provider call. Use a bounded deadline and a recovery workflow. A lock held across a network boundary increases contention and makes the failure window larger.

## Testing

Test concurrent acquisition, duplicate release, lease expiry, delayed old owner, renewal after expiry, store outage, process crash, lock ordering, fencing rejection, and recovery. Pause a simulated owner after acquisition, let another acquire the lock, then verify the first owner's write is rejected by the protected resource.

Run a failure drill for lock-store unavailability. Decide whether the operation fails closed, uses an idempotent fallback, or continues without the lock. Do not let the client silently assume ownership when the authority cannot answer.

## Security

Lock names and tokens may reveal tenant or business data. Scope and authorize administrative lock operations, protect transport credentials, and avoid accepting caller-controlled names without validation. A fencing token is a concurrency value, not a secret; the owner token should still be protected from logs and other tenants.

## Common Mistakes

* Treating a lease expiration as remote cancellation.
* Releasing a lock by name without checking the owner token.
* Using a lock where a database constraint is the actual invariant authority.
* Holding a lock across an unbounded network call.
* Omitting fencing for a resource that accepts delayed writes.
* Acquiring multiple locks in inconsistent order.
* Failing open when the lock service is unavailable without a safe policy.
* Assuming a process-local clock proves distributed lease validity.

## Senior Engineer Thinking

Ask which resource rejects stale work, what happens when the holder pauses, and whether the invariant can be enforced atomically somewhere simpler. The lock is only one participant; safety comes from the owner, lease authority, fencing or idempotency, protected resource, and recovery policy working together.

## Exercises

1. Decide whether a lock, unique constraint, atomic update, or idempotency record should protect four example invariants.
2. Model a lease expiry while the PHP process is paused. Add fencing and test the late write.
3. Find and fix a two-lock deadlock by defining canonical ordering.
4. Specify behavior when the lock store is unavailable during a payment, cache refresh, and scheduled report.

## Review Questions

* What are lock safety and liveness?
* Why is a unique owner token needed for release?
* What problem do fencing tokens solve?
* Why does a lease not cancel a paused process?
* When is a database constraint clearer than a distributed lock?
* Why should locks not span an unbounded network call?

## Summary

Distributed locks coordinate independent processes but introduce lease, pause, expiry, deadlock, and store-outage risks. Use them only for a clearly named invariant, acquire and release with unique ownership, bound and renew leases conditionally, use fencing or idempotency to reject stale effects, keep lock scope and order deliberate, and test paused owners and authority failures.

## References

- [Martin Kleppmann: How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Chapter 117 — Deadlocks](../../volumes/08-databases/117-deadlocks.md)
- [Chapter 242 — Idempotency](./242-idempotency.md)
- [Chapter 118 — Concurrency](../../volumes/08-databases/118-concurrency.md)
