---
book: The Complete Modern PHP Engineering Book
volume: 19
volume_title: SMALL ENGINEERING PROJECTS
chapter: 287
title: Search/Filtering Service
slug: search-filtering-service
status: complete
summary: ../../_ai/chapter-summaries/287-search-filtering-service-summary.md
---

# Chapter 287 — Search/Filtering Service

## Why This Matters

Search begins as a `WHERE` clause and becomes a resource-allocation problem. Users can combine filters, sort by several fields, request deep pages, and submit a query at the same time as an import changes the catalog. A permissive endpoint can turn arbitrary input into SQL injection, tenant leakage, expensive scans, unstable pagination, or a cache with unbounded keys.

This project builds a tenant-scoped product search for the catalog from Chapter 283. It treats query syntax, authorization scope, SQL shape, ordering, pagination, caching, limits, and observability as one contract. The service accepts a small language that can be validated and measured rather than exposing the database schema directly.

## Define the Query Contract

Start with a deliberately narrow API:

~~~text
scope: authenticated tenant, derived server-side
filters: optional exact sku, active flag, category, and name prefix
sorts: price ascending/descending or name ascending, then sku ascending
page: keyset cursor, maximum 50 rows
search term: maximum 80 validated UTF-8 bytes after NFC normalization; prefix only
filter semantics: omitted means no predicate; literal null, empty strings, and duplicate parameters are invalid; active=0 is false and active=1 is true
result: stable product projection, total count omitted initially
limits: bounded filter count, query time, response bytes, and rate
cache: optional for public catalog projections; key includes all semantics
~~~

Do not promise a total count unless the product needs it and the database capacity supports it. `COUNT(*)` over a large filtered relation can cost more than returning the page. A “search” that silently changes from prefix to substring matching is a contract change because index use and result meaning change.

## Parse into a Query Object

The HTTP controller should parse syntax and build a typed query object. It must not concatenate `sort`, `direction`, column names, or filter values into SQL. Values use bound parameters; identifiers come from a fixed server-side map.

~~~php
<?php

declare(strict_types=1);

enum ProductSort: string
{
    case Price = 'price';
    case Name = 'name';
}

final readonly class ProductSearch
{
    public function __construct(
        public string $tenantId,
        public ?string $sku,
        public ?bool $active,
        public ?string $category,
        public ?string $namePrefix,
        public ProductSort $sort,
        public bool $descending,
        public ?string $cursor,
        public int $limit,
    ) {
        if ($tenantId === '' || ($sku !== null && $sku === '') || $limit < 1 || $limit > 50) {
            throw new InvalidArgumentException('Search query is invalid');
        }

        if ($namePrefix !== null && ($namePrefix === '' || strlen($namePrefix) > 80)) {
            throw new InvalidArgumentException('Search prefix is invalid or too long');
        }

        if ($category !== null && ($category === '' || strlen($category) > 64)) {
            throw new InvalidArgumentException('Category is invalid or too long');
        }
    }
}

function sortSql(ProductSort $sort, bool $descending): string
{
    $column = match ($sort) {
        ProductSort::Price => 'price_cents',
        ProductSort::Name => 'name',
    };

    $direction = $descending ? ' DESC' : ' ASC';

    return $column . $direction . ', sku' . $direction;
}
~~~

The enum and `match` expression constrain identifiers to code-reviewed choices. This contract uses an 80-byte bound after the boundary validates UTF-8 and NFC-normalizes the search term; it does not promise 80 user-perceived characters. A cursor must be authenticated, integrity-protected, and tied to the query scope and sort; a raw base64 string is only an encoding.

## Preserve Authorization Scope

The tenant ID comes from trusted authentication and authorization context, not from a query parameter. Every repository query includes tenant scope. Management search may require actor capabilities in addition to tenant membership. A cache key and cursor must include the same scope dimensions.

Avoid different “not found” behavior that reveals whether a SKU belongs to another tenant. Search result counts, latency, and error details can also disclose information. Return only fields authorized for the search capability, and do not let a public catalog projection accidentally include internal cost, supplier, or moderation state.

## Build the SQL Shape Safely

The query builder can construct a fixed family of statements:

~~~text
SELECT sku, name, price_cents, active
FROM products
WHERE tenant_id = :tenant
  AND (:active is absent OR active = :active)
  AND (:category is absent OR category = :category)
  AND (:sku is absent OR sku = :sku)
  AND (:prefix is absent OR name >= :prefix_lower
                     AND name < :prefix_upper)
  AND (keyset predicate when cursor exists)
ORDER BY allowed_column [ASC|DESC], sku [ASC|DESC]
LIMIT :limit_plus_one
~~~

The query above is pseudocode: `is absent` and the optional keyset predicate represent conditional SQL assembly, not literal SQL syntax. In actual SQL, optional predicates should be assembled from validated pieces rather than relying on `:active IS NULL OR ...` when that harms the query plan. Bind values and use an allow-list for every optional identifier. A parameter can represent data; it cannot safely represent a table or column name.

Prefix bounds and collation are database-specific. This project normalizes valid UTF-8 input to NFC and uses a case-sensitive database collation; invalid UTF-8 is rejected, and the database adapter owns and tests the prefix-successor calculation. `name >= lower AND name < upper` is only correct under a matching collation and successor rule. If the database’s collation or Unicode behavior cannot support the contract, use a documented search strategy or a dedicated search index. Do not claim that a PHP string comparison and a database comparison have identical ordering.

## Define Stable Ordering

A page needs a total order. Sorting by `price_cents` alone is unstable when many products share a price. Add a unique tie-breaker such as `sku` and define its direction consistently:

~~~text
ORDER BY price_cents ASC, sku ASC
~~~

For descending order, decide whether both fields reverse or only the primary field does. The cursor predicate must match that exact lexicographic order. A query that orders one way and advances the cursor another way can skip or repeat records.

If product updates can change sort fields during pagination, a user may still see a moving result set. Options include accepting live semantics, using a snapshot/version boundary, or returning a search-session token with an explicit lifetime. Do not promise a stable snapshot without a database or index mechanism that provides one.

## Prefer Keyset Pagination

Offset pagination is easy to explain but expensive and unstable for deep pages. The database may scan and discard thousands of rows, while inserts or deletes shift later offsets. Keyset pagination uses the last ordered values:

~~~text
first page: ORDER BY price_cents, sku LIMIT 51
next page:  WHERE (price_cents, sku) > (:last_price, :last_sku)
            ORDER BY price_cents, sku LIMIT 51
~~~

The exact tuple syntax and mixed-direction predicate vary by database. The example shows price ascending; the name sort uses `(name, sku)` in the same direction, and descending queries use the corresponding reversed predicate. Sign or encrypt the cursor, include tenant, filter fingerprint, sort, direction, and schema version, and reject a cursor used with a different query. Return at most `limit` rows and use the extra row to decide whether `next_cursor` exists.

An opaque cursor prevents casual editing; integrity protection prevents tampering. It does not freeze changing data or authorize access to a different tenant.

## Index for the Workload

An index is a performance structure, not a correctness or authorization boundary. For a common query, measure a plan against the real database and data distribution. In this project, `sku` is unique within a tenant, and it is the final tie-breaker for every supported sort. Candidate indexes might include:

~~~text
(tenant_id, active, price_cents, sku)
(tenant_id, active, name, sku)
(tenant_id, sku)
~~~

Do not add every possible composite index. Each index consumes storage and write capacity, and its column order matters. A prefix search, equality filters, sort, and keyset predicate may need different designs. Use `EXPLAIN` on the exact database version, compare rows examined and returned, and observe plan changes after imports. Enforce the tenant-scoped SKU uniqueness with a database constraint; the tie-breaker is only stable if that invariant holds.

If the catalog grows beyond relational prefix search, introduce a dedicated search index as a derived projection. Then define indexing lag, source authority, tenant filtering, rebuild, stale results, and fallback behavior. A search index is not automatically more correct than the database.

## Cache Carefully

Search results have higher key cardinality than one-product lookups. A cache key must include tenant scope, normalized filters, sort/direction, cursor, representation version, policy version, and schema version:

~~~text
search:v2:{environment}:{tenant}:{policy_version}:{schema_version}:{filter_hash}:{sort}:{direction}:{cursor}:{representation}
~~~

Hashing keeps the key bounded; it does not repair an incomplete normalization. Canonicalize filter order, booleans, empty values, Unicode policy, and cursor encoding before hashing. Bound the number of filter combinations and cache only projections whose staleness is acceptable.

Invalidate or version results after catalog changes. A product import can make many query keys stale, so broad deletion may be expensive. Use short TTLs, generation versions, targeted invalidation, or accept a documented stale window. Do not cache authorization failures or a tenant’s result under a public key.

## Limit Query Cost

Admission controls should protect the database as well as the HTTP endpoint:

* cap result size, filter count, prefix length, and cursor age;
* reject unsupported combinations rather than silently running a table scan;
* apply separate rate and concurrency limits to interactive search and exports;
* bound database statement time and response bytes;
* cancel or stop work after the request deadline;
* protect search from bulk importer and analytics workloads.

Rate limiting counts requests; a query cost model may charge for prefix breadth, joins, or estimated rows. A short query can still be expensive. Chapter 281 covers limiter policy; Chapters 229 and 233 cover database and cache performance.

## Tests That Matter

Test syntax, semantics, and resource behavior:

* allow-listed filters and sorts reject unknown identifiers;
* values are bound and hostile input cannot change SQL shape;
* tenant scope is present in every query and cache key;
* null, empty, false, zero, and missing filters mean distinct documented things;
* prefix, collation, Unicode, and case behavior match the database contract;
* equal primary sort values use a stable tie-breaker;
* first and subsequent cursors neither skip nor duplicate rows;
* a cursor from another query, tenant, sort, or version is rejected;
* concurrent insert/update/delete behavior matches snapshot or live semantics;
* large offsets are rejected or bounded;
* cache normalization prevents equivalent queries from fragmenting keys;
* stale and invalidated results follow policy;
* database timeout, plan regression, and rate-limit paths are observable.

Use prepared statements or a query builder and a real database integration test for SQL shape, collation, indexes, plans, keyset boundaries, and tenant isolation. Property-test pagination by comparing every page to a sorted reference set under controlled mutations. A mocked repository cannot prove an index is used or that a cursor matches database ordering.

## Observe Search

Measure query count, result count, cache hit/miss, database latency, rows examined where available, timeout/cancellation, rejected cost, cursor errors, plan version, response bytes, and rate-limit decisions. Use bounded labels such as operation, filter class, sort class, result-size bucket, and tenant class. Do not label metrics with raw query strings, SKUs, cursors, tenant IDs, or arbitrary prefixes.

Alert on query timeouts, rows-examined spikes, cache-key cardinality, origin load, plan changes, tenant-isolation errors, and rising rejection rates. A high cache hit ratio can conceal stale or cross-scope results, so pair it with freshness and authorization checks.

## Rollout and Recovery

Introduce the typed parser and query builder behind the existing search path. Compare normalized results and query cost in a safe shadow or replay environment; do not execute a shadow query against production for every request without a budget. Add indexes concurrently or during a controlled window according to the database engine, then verify plans and write impact.

If a new search index or cache is wrong, fall back to the authoritative database with bounded capacity or disable the feature. Do not restore a previous query implementation that interprets a new cursor or representation incorrectly. Version cursors and cache keys, keep old readers during the compatibility window, and retire them only after evidence shows no active clients remain.

## Common Mistakes

* Concatenating filter values or sort identifiers into SQL.
* Trusting a client-supplied tenant ID in the query.
* Treating an index as an authorization or correctness guarantee.
* Comparing PHP string ordering with database collation without testing.
* Sorting by a non-unique field and using an incomplete cursor.
* Reusing a cursor with changed filters, tenant, or sort direction.
* Offering unbounded offsets, wildcard scans, counts, or filter combinations.
* Caching query results without tenant, policy, representation, or cursor dimensions.
* Shadow-querying production without an origin-load budget.
* Assuming a search index has no lag or rebuild failure.
* Logging raw queries, prefixes, cursors, or tenant identifiers in metrics.

## Senior Engineer Thinking

The senior question is not “how do we add filters to a listing?” It is “what query language do we promise, which identity and authorization scope is invariant, how is order made total, what does pagination mean under mutation, which index and cache costs are acceptable, and how do we fail when search becomes expensive or stale?”

A search service is a controlled language over a changing dataset. Parse it into typed policy, keep SQL shape allow-listed, preserve tenant scope, choose stable pagination, measure real plans, bound query cost, version cache and cursor semantics, and treat search indexes and caches as derived systems with explicit lag and recovery.

## Exercises

1. Define a query grammar for exact SKU, active state, category, and name prefix. List accepted, rejected, empty, null, and Unicode inputs.
2. Implement keyset pagination for `(price_cents, sku)` in both directions and compare every page with a sorted reference list.
3. Design a signed cursor payload containing tenant scope, filter fingerprint, sort, direction, version, and last key.
4. Use `EXPLAIN` to compare candidate tenant/filter/sort indexes under realistic data distributions.
5. Design cache keys and invalidation for a bulk catalog import that changes 100,000 products.
6. Choose rate and concurrency limits for interactive search, public lookup, and export, and explain the fallback when the database is saturated.

## Review Questions

* Why is a search endpoint a language and resource policy?
* Which query inputs may be bound as values, and which require allow-lists?
* Why must tenant scope exist in SQL, cursors, and cache keys?
* What makes ordering total and pagination stable?
* Why can offset pagination degrade with depth and mutation?
* What does an opaque signed cursor protect, and what does it not protect?
* Why do PHP and database collation rules need separate verification?
* How do indexes improve performance without enforcing correctness?
* What makes search cache cardinality harder than single-resource caching?
* Which signals reveal an expensive or stale search service?

## Summary

A search/filtering service is a constrained query language over durable, changing data. Define an explicit grammar and cost contract, parse into typed objects, derive tenant scope server-side, allow-list SQL identifiers and bind values, define collation and null semantics, make sort order total, use authenticated keyset cursors, measure workload-specific indexes, bound query and cache cardinality, protect the database with rate/concurrency limits, test real SQL and pagination under mutation, observe freshness and cost, and version query, cursor, cache, and search-index behavior through rollout.

## References

- [Chapter 104 — SQL for PHP Developers](../08-databases/104-sql-for-php-developers.md)
- [Chapter 109 — Composite Indexes](../08-databases/109-composite-indexes.md)
- [Chapter 110 — Query Plans](../08-databases/110-query-plans.md)
- [Chapter 119 — Pagination](../08-databases/119-pagination.md)
- [Chapter 120 — Large Datasets](../08-databases/120-large-datasets.md)
- [Chapter 123 — Database vs PHP Responsibilities](../08-databases/123-database-vs-php-responsibilities.md)
- [Chapter 229 — Database Performance](../15-performance/229-database-performance.md)
- [Chapter 233 — Caching](../15-performance/233-caching.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 281 — Rate Limiter](./281-rate-limiter.md)
- [Chapter 283 — File Importer](./283-file-importer.md)
- [Chapter 286 — Cache-Backed Service](./286-cache-backed-service.md)
