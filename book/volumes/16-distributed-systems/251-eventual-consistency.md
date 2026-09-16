---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 251
title: Eventual Consistency
slug: eventual-consistency
status: complete
summary: ../../_ai/chapter-summaries/251-eventual-consistency-summary.md
---

# Chapter 251 — Eventual Consistency

## Why This Matters

An eventually consistent system allows a reader to observe an older value for a bounded or unbounded interval, while promising that replicas or derived views converge if updates stop and the system continues operating. This can improve availability and throughput, but it changes what users and developers may assume immediately after a write.

Eventual consistency is not “inconsistent data is acceptable.” It is a contract about delay, ordering, conflict resolution, and recovery. A product may tolerate a search result appearing seconds after creation; it may not tolerate a revoked permission granting access during the same interval.

## Mental Model

```text
authoritative write
        ↓
   event / log
        ↓ delayed delivery
  replica or read model
```

The consistency window begins when the authority commits and ends when the reader has incorporated that version. During the window, readers need a documented behavior: show the old value, show pending, route to the authority, or reject the operation.

## Where the Delay Appears

Delay can occur between:

* primary database and read replica;
* database commit and cache invalidation;
* outbox and broker publication;
* broker delivery and consumer processing;
* source model and search index;
* one region and another;
* configuration authority and application workers.

Measure each leg. A fast consumer cannot compensate for a delayed outbox publisher, and a fresh read model cannot compensate for a stale CDN response.

## Design the User Contract

State what the caller sees after each transition:

```text
create profile → profile lookup is immediately available from owner
create profile → search result may appear within 30 seconds
revoke session  → authorization checks use the owner or a short freshness bound
```

A pending state is often better than a false assertion. Include a version, last-updated time, or freshness marker when clients need to explain why a view is not current. Do not expose internal queue names as the user contract.

## Monotonic and Read-Your-Writes Behavior

Eventual consistency becomes easier to use when the system adds session guarantees. Read-your-writes routes a caller to a source that has incorporated its write. Monotonic reads prevent a caller from seeing version 8 and later version 7. A session token, version number, or replica position can carry the requirement.

These guarantees consume routing and storage complexity. Do not promise them without a way to detect whether a replica is sufficiently current. A timestamp alone can be unreliable when clocks differ; an authority-issued sequence or log position is a stronger signal when the system supports it.

## Events and Replay

Derived consumers should process events by identity and version:

~~~php
<?php

declare(strict_types=1);

final readonly class ProjectedRecord
{
    public function __construct(
        public int $version,
        public string $name,
    ) {
    }
}

function applyProjection(
    ProjectedRecord $current,
    int $incomingVersion,
    string $incomingName,
): ProjectedRecord {
    if ($incomingVersion <= $current->version) {
        return $current;
    }

    return new ProjectedRecord($incomingVersion, $incomingName);
}
~~~

This protects one projection from a late older event. It assumes versions are comparable for the same record. It does not merge concurrent changes or validate the domain invariant. A projection should be rebuildable from an authoritative history or snapshot when bugs or missed messages occur.

## Conflict Resolution

When replicas accept writes independently, concurrent changes need a policy:

* one authority wins;
* last write wins using an authority-issued ordering;
* merge fields under domain rules;
* preserve both versions for human resolution;
* reject the conflict and ask the caller to retry from a new version.

Generic last-write-wins is attractive because it is simple, but it can silently discard business actions. A shopping-cart quantity, account balance, or permission grant usually needs a command or transaction that revalidates the invariant.

## Caches and Invalidation

Cache invalidation is an eventual-consistency workflow. A write can commit while invalidation is delayed or lost. Use versioned keys, bounded TTLs, an outbox event, or an owner lookup according to the freshness requirement. A cache miss that reloads old replica data can reintroduce stale content.

Do not use arbitrary sleeps after invalidation. Wait for an acknowledgment or version, or state that the value may be stale. See [Chapter 233 — Caching](../../volumes/15-performance/233-caching.md).

## Recovery and Reconciliation

Consumers can fall behind, restart, or miss a message. Track offsets, event age, projection version, and failure count. Reconciliation compares derived state with the owner and repairs it using idempotent operations. It should be safe to run more than once and should not overwrite newer state with an old snapshot.

During a rebuild, expose stale or pending status rather than silently presenting an empty result as authoritative. Protect rebuild endpoints and bound their load; a full reindex can compete with interactive traffic.

## PHP and Framework Boundaries

PHP code sees the data source selected by its repository, client, ORM, cache, or framework configuration. A local object does not reveal whether its value came from a primary, replica, cache, or read model. Make source and freshness policy visible at the application boundary when it affects correctness.

Long-running workers can retain stale configuration or cached policy. Refresh process-local state according to its contract and recycle workers when a deployment or key rotation requires it.

## Testing

Test delayed delivery, duplicate and out-of-order events, lost invalidation, replica lag, stale cache, projection rebuild, concurrent writes, conflict policy, read-your-writes, and revocation. Use fake positions or clocks for deterministic unit tests and real broker/database integration for delivery and isolation semantics.

Assert allowed observations and convergence. A test that waits for a fixed sleep can pass on one machine and fail under load; a test that waits for an owner-issued version tests the actual contract.

## Security

Set a stricter consistency window for authorization, session revocation, password reset, secrets, and fraud decisions. A stale read can grant access or reuse a revoked credential. If eventual propagation is unavoidable, route sensitive decisions to the authority or use short-lived capabilities whose maximum exposure is explicit.

## Common Mistakes

* Calling eventual consistency a correctness policy without a freshness bound.
* Adding sleeps instead of waiting for a version or acknowledgment.
* Letting an older event overwrite a newer projection.
* Using last-write-wins for non-mergeable financial state.
* Rebuilding a read model without showing stale or pending status.
* Caching authorization or revocation longer than the security contract allows.
* Assuming a fresh PHP object came from an authoritative source.

## Senior Engineer Thinking

Ask which facts may be stale, for whom, for how long, and what happens when updates conflict or messages are lost. Eventual consistency is useful when the product can name the window and the system can observe and repair it. It is dangerous when it is used as a vague excuse for an unspecified race.

## Exercises

1. Define freshness and read-your-writes requirements for search, catalog pages, permissions, and payment status.
2. Design a versioned projection that handles duplicate and out-of-order events.
3. Simulate lost cache invalidation and build a safe reconciliation path.
4. Specify a security policy for revocation when the read model can lag by five seconds.

## Review Questions

* What is the consistency window?
* Which signals are stronger than a wall-clock timestamp for replica freshness?
* Why should projections use identity and version?
* When is last-write-wins unsafe?
* Which state usually needs stricter freshness than catalog content?
* Why are fixed sleeps poor consistency tests?

## Summary

Eventual consistency trades immediate visibility for availability, throughput, or simpler independent consumers. Define the consistency window, source authority, ordering and conflict policy, user-visible pending behavior, and reconciliation path. Use versions and idempotent projection updates, measure lag, and apply stricter freshness rules to authorization, revocation, payment, and other security-sensitive facts.

## References

- [Martin Kleppmann: Designing Data-Intensive Applications](https://dataintensive.net/)
- [PostgreSQL documentation: Streaming replication monitoring](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [Chapter 250 — Consistency](./250-consistency.md)
- [Chapter 233 — Caching](../../volumes/15-performance/233-caching.md)
