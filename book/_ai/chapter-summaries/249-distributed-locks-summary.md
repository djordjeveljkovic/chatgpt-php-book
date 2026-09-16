# AI Summary — Chapter 249 — Distributed Locks

- Status: complete
- Volume: Volume 16 — DISTRIBUTED SYSTEMS
- Last updated: 2026-09-16

## Written material

Covers lock safety and liveness, leases, owner tokens, fencing tokens, granularity, lock ordering, deadlocks, remote effects, PHP process pauses, testing, and security. Includes typed LockLease and conditional-release port examples.

## Concepts already explained

Distributed lock, lease, owner token, fencing token, stale owner, safety, liveness, lock granularity, canonical lock order, and protected resource.

## Terminology established

Lock authority, renewal, expiry, late write, per-resource lock, global lock, conditional release, and stale-effect rejection.

## Examples used

Lease lifecycle, fencing sequence, typed LockLease and LockStore, lock-scope examples, two-lock deadlock, and paused-owner tests.

## Cross-references

Chapters 117, 118, 242, and 247.

## Open threads

Continue with consistency guarantees and version visibility in Chapter 250.

## Exact next section

Chapter 250 — Consistency: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP example. Local Markdown links resolved and git diff --check passed. No distributed lock or fencing integration was run.

## Writing notes

Emphasizes that the protected resource, not merely the lock service, must reject stale owners.
