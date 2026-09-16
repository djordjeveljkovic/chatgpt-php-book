---
book: The Complete Modern PHP Engineering Book
volume: 15
volume_title: PERFORMANCE
chapter: 233
title: Caching
slug: caching
status: complete
summary: ../../_ai/chapter-summaries/233-caching-summary.md
---

# Chapter 233 — Caching

A cache stores a reusable result closer to the reader or producer so future work is cheaper or faster. It trades computation, I/O, or latency for memory, staleness, invalidation complexity, and operational cost.

A cache is a performance mechanism with correctness semantics. Decide what may be stale, how keys represent authorization and version, what happens on a miss or outage, and how invalidation occurs before choosing a product.

## Why this matters

Caching a public product description may reduce database load. Caching an authorization decision under a key that omits tenant or actor identity can disclose data. Caching a failed provider response for too long can make an outage persistent. Every cache needs an owner, a freshness policy, and a failure policy.

Common layers include browser and CDN caches, reverse proxies, application memory, distributed object caches, and database pages. A hit at one layer may still leave expensive work at another. Measure hit rate, latency, size, evictions, stale age, and origin load.

## Cache-aside

Cache-aside loads from the cache first and fills it after an origin read:

```php
<?php

declare(strict_types=1);

interface Cache
{
    public function get(string $key): mixed;
    public function set(string $key, mixed $value, int $ttlSeconds): void;
}

function productFor(Cache $cache, ProductRepository $products, int $tenantId, int $id): Product
{
    $key = "tenant:{$tenantId}:product:{$id}:v1";
    $cached = $cache->get($key);
    if ($cached instanceof Product) {
        return $cached;
    }

    $product = $products->ownedBy($tenantId, $id);
    if ($product === null) {
        throw new DomainException('Product unavailable');
    }

    $cache->set($key, $product, 300);

    return $product;
}
```

The key includes tenant scope and a schema/version marker. A real cache often serializes values, so validate types after decoding and avoid unsafe object deserialization. Cache only data the caller is authorized to receive.

Cache-aside has a race when several requests miss at once. A short lock, request coalescing, stale-while-revalidate, or a bounded duplicate load can prevent a stampede. Locks need expiry and ownership; a lock that never expires is an outage.

## TTL and invalidation

TTL expresses maximum staleness or retention, not correctness by itself. Choose it from the domain's freshness requirement and origin cost. For data that must change immediately, invalidate or version it after a committed write, and define what happens if invalidation fails.

Versioned keys can make broad invalidation cheap: change `v1` to `v2` after a schema or policy change. They leave old values until expiry, so bound memory and avoid retaining sensitive data. An event-driven invalidation path needs delivery and duplicate handling like any other message.

Negative caching can protect an origin from repeated misses but can hide a newly created record during its TTL. Cache errors only with a deliberate short policy. Never cache authorization failures under a key that can be reused by another actor.

## Stampedes and hot keys

A popular key can overload one cache node or one origin record. Use jittered TTLs, bounded refresh work, request coalescing, replicas, or precomputation according to the workload. A cache with a 100% hit rate can still fail if one hot key consumes memory or a node disappears.

Estimate memory from key size, serialized value, metadata, replication, and eviction policy. Monitor evictions and fragmentation. A cache eviction is not necessarily an error; an unexpected eviction rate may make the origin unhealthy.

## HTTP and application caching

HTTP cache headers are a public protocol contract. `Cache-Control: private` and `Vary` must reflect authorization and request headers; a shared cache must not serve one user's response to another. ETags can avoid sending an unchanged representation but do not replace authorization or a database conditional update.

Application caches should separate public, tenant, and actor-scoped keys. Include locale, feature version, and representation format when they affect output. Keep secret-bearing responses out of shared caches unless the cache policy explicitly protects them.

## Outages and invalid data

A cache is usually an optimization, so decide whether an outage should bypass it, serve bounded stale data, or fail. Do not make a cache a hidden source of truth unless the system has persistence and recovery for it. If a cache contains security or financial state, treat it as a data store and design durability and authorization accordingly.

Protect cache clients with connection, operation, and total request timeouts. Bound value size and serialization work. Cache keys and values may contain personal data; apply access control, encryption, retention, and logging redaction where required.

## Testing and operations

Test hit, miss, expiry, stale, invalidation failure, cache outage, stampede protection, key scope, serialization mismatch, and version migration. Use a real cache adapter in integration tests; an array fake cannot reveal network timeouts, eviction, consistency, or size limits.

Observe hit ratio by key family, origin latency, cache latency, evictions, memory, stale age, invalidation lag, lock contention, and bypass rate. Alert on origin overload and stale policy violations rather than only cache process health.

## Exercises

1. Design a cache key for a tenant-scoped localized product response and list every dimension that affects its value.
2. Add stampede protection to a cache-aside path and define lock expiry and failure behavior.
3. Choose TTL and invalidation rules for public catalog data, account balances, and authorization policy. Explain why they differ.

## Review questions

- What correctness decisions must accompany a cache?
- Why are tenant and actor dimensions part of key design?
- How do TTL and invalidation solve different problems?
- Why can a cache outage expose hidden origin capacity limits?
- Which values should never enter a shared cache without an explicit policy?

## Summary

Caching exchanges origin work for staleness, memory, invalidation, and failure complexity. Use cache-aside or another explicit pattern, design scoped and versioned keys, bound stampede and value size, choose TTL and invalidation from correctness requirements, and observe both cache and origin behavior.

## References

- [PHP-FIG PSR-16: Simple Cache](https://www.php-fig.org/psr/psr-16/)
- [PHP-FIG PSR-6: Caching Interface](https://www.php-fig.org/psr/psr-6/)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [Martin Fowler: Cache-Aside](https://martinfowler.com/bliki/CacheAside.html)
