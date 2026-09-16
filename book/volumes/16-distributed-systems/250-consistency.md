---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 250
title: Consistency
slug: consistency
status: complete
summary: ../../_ai/chapter-summaries/250-consistency-summary.md
---

# Chapter 250 — Consistency

## Why This Matters

Consistency describes what values readers may observe when several processes read and write shared state. It is a contract about visibility and ordering, not a synonym for “the data is correct.” A system can be strongly consistent and enforce the wrong business rule, or eventually consistent and correctly model a product that tolerates delay.

Choose the weakest model that satisfies the invariant and user experience, then make the delay and conflict behavior explicit. “The cache is eventually consistent” is not enough when the cached value is an authorization decision.

## A Vocabulary of Guarantees

Useful guarantees include:

* **read-after-write:** a reader sees its successful write or a newer value;
* **monotonic reads:** one reader does not move backward to an older version;
* **consistent prefix:** observations expose a prefix of an ordered history without gaps or reordering within that prefix;
* **causal consistency:** causally related writes are observed in order;
* **linearizability:** each operation appears to take effect at one point between invocation and response;
* **serializability:** a transaction result is equivalent to some serial order of transactions.

These are different dimensions. A database transaction may be serializable while a read replica is stale. A message consumer may preserve causal order for one aggregate while unrelated aggregates remain independent.

## Consistency Is Scoped

State has an authority and a scope:

```text
payment status → payment owner, operation identity, version
product page   → catalog owner, representation version, freshness
permission     → authorization owner, actor, tenant, policy version
```

Ask who can write, who can read, which versions are acceptable, and how long stale data may be used. A single global consistency setting rarely expresses all domain requirements.

## Primary, Replica, and Cache

A write to a primary followed by a read from a lagging replica may return the old value. A cache may be older still. Options include:

* route the read to the primary for a bounded session;
* wait until a replica reaches a known position;
* return a pending or stale response when permitted;
* invalidate or version the cache after commit;
* make the read an operation lookup against the owner.

Do not solve a consistency requirement by adding arbitrary sleeps. Sleep does not prove replication or invalidation completed. Use a protocol signal, version, or bounded fallback.

## Versioned State

Versions prevent an older update from overwriting a newer one:

~~~php
<?php

declare(strict_types=1);

final readonly class VersionedValue
{
    public function __construct(
        public int $version,
        public string $value,
    ) {
        if ($version < 1) {
            throw new InvalidArgumentException('Version must be positive');
        }
    }
}

function applyIfNewer(
    VersionedValue $current,
    VersionedValue $incoming,
): VersionedValue {
    return $incoming->version > $current->version ? $incoming : $current;
}
~~~

This helper gives last-version-wins behavior for one ordered version source. It does not resolve concurrent updates with equal or incomparable versions, and it does not enforce an invariant such as “balance cannot become negative.” Use the owning database's conditional update or a domain conflict policy for those cases.

## Conflicts

Concurrent writers can produce conflicts:

```text
read balance 100       read balance 100
subtract 30            subtract 80
write 70               write 20
```

The lost update is not fixed by a cache or a replica. Use an atomic update, optimistic version check, serializable transaction, lock, command serialization, or domain merge. A last-write-wins register may be correct for a profile nickname and incorrect for money.

Conflict resolution needs a semantic owner. Merging two JSON documents field by field can violate relationships between fields. A domain command that revalidates the aggregate invariant is often safer than generic document merging.

## Consistency and Events

An event says what the producer committed at a point in its authority. Delivery can be delayed, duplicated, or out of order. Consumers should track event identity and version, tolerate replay, and expose lag. Derived read models are consistent with their source only after processing the relevant event.

```text
source commit → event/outbox → broker → consumer → read model
       t0             t1          t2        t3
```

The interval from t0 to t3 is a consistency window. Define whether users may read the old projection during it and how the UI or API communicates pending state.

## Transactions and Boundaries

A local database transaction can make multiple changes atomic within its database and isolation contract. It does not include a remote provider, another database, a cache, or an already delivered message. Coordinate cross-boundary work with an outbox, idempotency, compensation, or a workflow state machine.

Do not claim global consistency because one service uses a transaction. State the authority, boundary, isolation, and propagation path.

## What PHP Does

PHP's local variable assignment does not synchronize replicas, invalidate caches, or publish events. A request may read one version and a later request may reach a different worker, connection, replica, or cache. Framework abstractions can hide those choices; the consistency contract remains an application and storage concern.

In a long-running worker, stale configuration, cached policy, or retained model state can outlive the intended freshness window. Reset or refresh process-local state deliberately.

## Testing

Test read-after-write, replica lag, cache staleness, out-of-order events, duplicate events, lost invalidation, concurrent writers, version conflicts, and workflow recovery. Use controllable clocks and fake replication positions for deterministic unit tests, then run integration tests against the real database and message semantics.

Assert the allowed observation set. A test that waits 100 ms and happens to see the new value proves little; a test that waits for a version or owner acknowledgment proves a contract.

## Security

Stale authorization, tenant membership, revocation, or secret data can be a security issue. Define a stricter consistency policy for security-sensitive state than for public catalog text. Never use an eventually consistent cache to grant permission unless the security contract explicitly tolerates its maximum staleness and revocation behavior.

## Common Mistakes

* Treating consistency as a single global switch.
* Adding sleeps instead of waiting for a version or acknowledgment.
* Reading from replicas immediately after a write without a contract.
* Using last-write-wins for non-mergeable financial or inventory state.
* Assuming an outbox makes every consumer immediately consistent.
* Merging documents without revalidating domain invariants.
* Caching authorization longer than revocation can tolerate.
* Believing a PHP local variable is shared across workers.

## Senior Engineer Thinking

Ask “which observations are allowed, for whom, for how long, and which invariant rejects an invalid write?” Consistency is a product and domain decision expressed through storage, messages, caches, and APIs. Choose a model per fact, expose the consistency window, and test the stale and conflicting states users can actually encounter.

## Exercises

1. Define consistency requirements for product descriptions, payment status, permissions, and inventory.
2. Design a read-after-write policy with a primary, replica, and cache. Specify version signals and fallback.
3. Reproduce a lost update and fix it with a conditional version update.
4. Measure an event-driven read model's consistency window and design its pending response.

## Review Questions

* What is the difference between consistency and correctness?
* Which guarantees are useful for a reader or transaction?
* Why does a local transaction not cover a remote provider?
* What problem do versions solve, and what do they not solve?
* Why are stale authorization decisions risky?
* How should a derived read model communicate lag?

## Summary

Consistency defines which versions and orderings readers may observe across shared state. Scope the guarantee to an owner and fact, distinguish primary, replica, cache, and projection behavior, use versions or atomic domain transitions for conflicts, coordinate cross-boundary changes with explicit workflows, and set stricter freshness rules for security-sensitive state. Test stale, delayed, duplicated, and concurrent observations directly.

## References

- [Martin Kleppmann: Designing Data-Intensive Applications, consistency discussion](https://dataintensive.net/)
- [PostgreSQL documentation: Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Chapter 115 — Isolation](../../volumes/08-databases/115-isolation.md)
- [Chapter 251 — Eventual Consistency](./251-eventual-consistency.md)
