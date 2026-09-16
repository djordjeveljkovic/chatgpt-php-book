---
book: The Complete Modern PHP Engineering Book
volume: 19
volume_title: SMALL ENGINEERING PROJECTS
chapter: 286
title: Cache-Backed Service
slug: cache-backed-service
status: complete
summary: ../../_ai/chapter-summaries/286-cache-backed-service-summary.md
---

# Chapter 286 — Cache-Backed Service

## Why This Matters

A cache can make a PHP service faster and cheaper, but it cannot make an unclear ownership model correct. If a product is updated in the database while an old object remains in Redis, which value may a customer see? If ten FPM workers miss the same key, how many database queries should run? If the cache is unavailable, should the request use the database, return a degraded response, or fail closed?

This project builds a tenant-scoped product lookup for the catalog importer from Chapter 283 and the URL service from Chapter 282. The database remains authoritative. The cache accelerates reads, uses bounded freshness, prevents tenant leakage, and has explicit invalidation, stampede, outage, and rollout behavior.

## State the Source-of-Truth Contract

Write the contract before adding a cache:

~~~text
authority: catalog database
cache role: performance and bounded read availability
write path: commit the database first; invalidate or publish only after commit
read path: catalog reads may be stale for at most 30 seconds
key: environment + tenant + resource + identifier + schema version
serialization: versioned JSON shape; no PHP object deserialization
miss: load authority, validate result, populate cache
negative result: cache a distinct not-found result briefly, with tenant-scoped key
invalidation: update transaction emits an invalidation event
outage: bounded database fallback for reads; never treat cache as write authority
privacy: tenant and access policy apply to cache reads and fills
~~~

“Cached” is not a consistency model. The contract must name the maximum intended staleness, the behavior after invalidation failure, and whether a caller may require read-your-writes. A cache hit is useful evidence that a value was stored; it is not evidence that the value is current.

## Choose the Boundary

Cache the result at the narrowest stable boundary that has a clear owner. A repository can cache a product projection; a controller should not cache a half-rendered response containing authorization and locale decisions. Do not cache a decision whose inputs are omitted from the key.

Common patterns include:

* cache-aside: application reads cache, loads authority on miss, then stores the result;
* read-through: a cache library owns the miss loader;
* write-through: writes update cache and authority together through one abstraction;
* write-behind: cache accepts writes before durable storage, which creates a much larger recovery contract.

Use cache-aside for this project. The database transaction owns product state, while invalidation or versioning handles derived cache state. Write-behind would make a performance component responsible for durability and is outside the first project’s scope.

## Design Keys and Values

A key should be deterministic, bounded, and complete:

~~~text
catalog:v3:{environment}:{tenant_id}:product:{sku}
~~~

Do not accept arbitrary query strings, raw serialized objects, or unbounded user input as key components. For this project, normalize each component to 1–128 ASCII letters, digits, dot, underscore, or hyphen, and reject control characters and separators. Keep tenant scope explicit. A missing tenant segment can turn a cache into a cross-tenant data leak even when the database query is properly authorized.

Store a versioned value with the fields the read contract needs, including schema version and the source record version used for freshness comparisons. Prefer JSON or a deliberately defined scalar/array shape. Never unserialize untrusted PHP objects from a shared cache. Bound value bytes, nested depth, collection size, and TTL. If the source value contains a secret or personal data, apply encryption/access control and retention policy to the cache, or do not cache it.

## Implement a Narrow Cache Port

Keep the cache provider replaceable and make misses explicit:

~~~php
<?php

declare(strict_types=1);

final readonly class CachedProduct
{
    public function __construct(
        public string $sku,
        public string $name,
        public int $priceCents,
        public int $expiresAt,
        public int $schemaVersion,
        public int $sourceVersion,
    ) {
        if ($sku === '' || $name === '' || $priceCents < 0 || $expiresAt < 0 || $schemaVersion < 1 || $sourceVersion < 0) {
            throw new InvalidArgumentException('Cached product is invalid');
        }
    }
}

enum CacheLookupKind: string
{
    case Hit = 'hit';
    case Miss = 'miss';
    case Negative = 'negative';
}

final readonly class CacheLookup
{
    public function __construct(
        public CacheLookupKind $kind,
        public ?CachedProduct $product,
    ) {
        if ($kind === CacheLookupKind::Hit && $product === null) {
            throw new InvalidArgumentException('A cache hit needs a product');
        }

        if ($kind !== CacheLookupKind::Hit && $product !== null) {
            throw new InvalidArgumentException('Only a cache hit can contain a product');
        }
    }
}

interface ProductCache
{
    public function get(string $key): CacheLookup;

    public function put(string $key, CachedProduct $product, int $ttlSeconds): void;

    public function delete(string $key): void;
}

function cacheKey(string $environment, string $tenantId, string $sku): string
{
    foreach ([$environment, $tenantId, $sku] as $part) {
        if (preg_match('/\A[A-Za-z0-9._-]{1,128}\z/D', $part) !== 1) {
            throw new InvalidArgumentException('Cache key component is invalid');
        }
    }

    return "catalog:v3:$environment:$tenantId:product:$sku";
}
~~~

The port does not expose a generic `mixed` value, which makes accidental cross-feature reuse less likely. A production cache adapter must also handle serialization failure, provider timeouts, eviction, and malformed values. A malformed cache entry is a miss plus telemetry, not a reason to pass unvalidated data to the caller.

## Handle Freshness and Expiration

TTL is a deletion hint, not a complete freshness guarantee. The authority may change immediately after a cache fill. Use a TTL shorter than the permitted staleness and include an expiry or source version when the read policy needs to make a decision.

For read-your-writes, route the caller to the authority for a bounded window, attach a version to the write, or invalidate synchronously before acknowledging the write. Do not promise read-your-writes merely because the application called `delete()`; invalidation can fail or race with an in-flight fill.

Negative caching can protect the database from repeated misses, but a newly created product may remain invisible until the negative TTL expires or an invalidation is issued. Use a short negative TTL for mutable namespaces and never let a not-found value bypass authorization.

## Invalidation Races

The classic race is:

~~~text
T1 cache miss → reads old database value
T2 writes new value → invalidates cache
T1 stores old value after invalidation
~~~

Possible protections include versioned values, write timestamps, compare-and-set, delayed invalidation, or accepting a bounded stale window explicitly. The right choice depends on the freshness contract. A blind delete does not prove that an earlier fill cannot repopulate the key.

An invalidation event should carry resource identity and source version, not an entire sensitive object. Consumers ignore an invalidation older than the version they already hold. If events are delivered at least once, invalidation handlers must be idempotent.

## Prevent a Stampede

When a hot key expires, many workers may miss and query the authority simultaneously. This dogpile can overload the database precisely when the cache is least helpful. Options include:

* request coalescing within one process;
* a short distributed fill lease with fencing or ownership checks;
* stale-while-revalidate with a bounded stale allowance;
* randomized TTL jitter;
* prewarming predictable hot keys;
* admission control and a bounded fallback.

A cache lock is not automatically safe. Set an owner token, expiry, and release condition; never delete a lock owned by a newer fill. If a fill takes longer than its lease, duplicate fills may occur, so the database and cache version policy must tolerate them.

## Failure Behavior

Decide what each failure means:

| Failure | Read behavior |
| --- | --- |
| cache miss | load and validate database result |
| malformed cached value | discard as miss; alert if frequent |
| cache timeout | catalog display may use a database fallback capped at 20 concurrent fills and a 100ms origin deadline |
| database timeout after miss | temporary failure; do not invent a value |
| invalidation outage | retain explicit stale-risk signal and bounded TTL |
| cache memory pressure | accept eviction; protect authority capacity |
| cache partition | use the same capped fallback for catalog display; return a temporary failure after the budget is exhausted |

The fallback needs its own concurrency and timeout budget. For this catalog display, the service permits at most 20 concurrent origin fills and a 100ms database deadline; after that it returns a temporary failure rather than adding unlimited load. A payment, authorization, or revocation path would use a different policy and would not serve stale catalog data. Chapter 281’s rate limiter and Chapter 247’s circuit-breaker guidance can protect this path.

Never use a stale cached authorization, entitlement, payment, or revocation decision without a contract that accepts the risk. A product display may tolerate thirty seconds of staleness; access control usually needs a much tighter boundary or an authoritative check.

## Test Cache Correctness

Test the provider and service boundary:

* key includes environment, tenant, resource, and version;
* tenant A cannot read tenant B’s cached value;
* valid hit, miss, negative hit, expiry, malformed value, and eviction;
* serialization and size limits reject unsafe values;
* cache outage falls back or fails according to capability;
* database write and invalidation ordering meet the freshness contract;
* stale fill cannot overwrite a newer version;
* concurrent hot-key misses are bounded or explicitly approximate;
* invalidation replay is idempotent;
* read-your-writes behavior is tested at its promised boundary;
* mixed-version workers interpret keys and values safely;
* cache data is removed or expires according to retention policy.

Use a real cache integration for TTL, atomic primitives, eviction, and concurrent fill behavior. A fake cache can test the service’s branch logic but cannot prove network timeout, serialization, replication, or eviction behavior. Load-test hot keys, cold-key cardinality, cache outage, authority latency, and invalidation backlog.

## Observe the Cache

Measure hit, miss, negative-hit, malformed-value, fill, eviction, timeout, invalidation, stale-fallback, and stampede-lease outcomes. Track cache latency, value size, key count, memory, authority fallback rate, fill duration, invalidation lag, and source-version drift.

Use bounded dimensions such as resource class, tenant class, schema version, outcome, and policy. Do not label metrics with raw tenant IDs, SKUs, destinations, serialized values, or arbitrary key text. Logs should contain a redacted key reference and correlation ID only when retention permits.

A high hit ratio can hide incorrect keys or stale values. Pair performance metrics with freshness violations, tenant-isolation alerts, authority load, and sampled source-version comparisons.

## Rollout and Recovery

Deploy the cache as an additive acceleration path. Start in observe-only or shadow mode using isolated cache keys; shadow reads must not double authority load without a budget. Warm a small tenant cohort, compare hit/miss/freshness behavior, then enable production reads with a conservative TTL and a tested bypass switch.

Version keys and values when serialization or authorization inputs change. During mixed deployment, old workers must not read a value whose meaning they cannot validate. If cache state is suspect, flushing derived keys is safer than restoring an old interpretation, but the resulting authority load must be capacity-tested. Never restore the cache as the source of truth after losing the database.

## Common Mistakes

* Treating a cache as authoritative because it is faster.
* Omitting tenant, environment, policy, or schema version from a key.
* Caching authorization or revocation decisions without an explicit freshness contract.
* Using PHP object deserialization for shared or untrusted cache values.
* Assuming a delete invalidation cannot race with an in-flight fill.
* Letting every worker fall back to the database during a cache outage.
* Locking a hot key without owner tokens, expiry, or fencing.
* Treating TTL as proof of freshness or read-your-writes.
* Caching unbounded query input, raw personal data, or large responses.
* Measuring hit ratio without correctness, staleness, and authority-load signals.
* Deploying new value semantics while old workers still read the same keys.

## Senior Engineer Thinking

The senior question is not “how do we add Redis?” It is “what is authoritative, how stale may this value be, which inputs define identity, what happens during invalidation and stampede races, and can the authority survive the cache’s failure mode?”

A cache is a derived-data system with a performance purpose and a consistency cost. Give it a narrow owner, complete keys, versioned values, bounded TTLs, explicit invalidation, safe fallbacks, realistic tests, and metrics that measure correctness as well as speed.

## Exercises

1. Design a cache key and value for a tenant-scoped product lookup. List every input that affects authorization or representation.
2. Draw the stale-fill invalidation race and compare delete, version, compare-and-set, and stale-while-revalidate strategies.
3. Define cache-outage behavior for a product display, entitlement check, and payment mutation.
4. Load-test a hot key expiry with ten, one hundred, and one thousand workers. Measure authority load and fill-lease behavior.
5. Design a mixed-version cache-key migration and a safe bypass/flush procedure.

## Review Questions

* Why must the authority remain explicit?
* Which fields belong in a cache key?
* How do TTL, invalidation, and versioning differ?
* Why can a negative cache hide a newly created resource?
* How does a stale fill repopulate a key after invalidation?
* Why is a cache lock not automatically a correctness lock?
* What fallback protects the database during cache outage?
* Which data should not be cached or logged?
* Why are real cache integrations needed beyond a fake?
* What evidence would show that a cache is fast but wrong?

## Summary

A cache-backed service is a source-of-truth and derived-data design. Keep the database authoritative, define staleness and read-your-writes behavior, choose a narrow cache-aside boundary, build complete tenant/versioned keys, serialize bounded data safely, handle negative caching and invalidation races, prevent hot-key stampedes, bound outage fallbacks, test real TTL/eviction/concurrency behavior, observe freshness and authority load, and roll out value semantics with versioned keys and bypass recovery.

## References

- [Chapter 233 — Caching](../15-performance/233-caching.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 247 — Circuit Breakers](../16-distributed-systems/247-circuit-breakers.md)
- [Chapter 248 — Bulkheads](../16-distributed-systems/248-bulkheads.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 281 — Rate Limiter](./281-rate-limiter.md)
- [Chapter 283 — File Importer](./283-file-importer.md)
- [Chapter 285 — Notification Dispatcher](./285-notification-dispatcher.md)
