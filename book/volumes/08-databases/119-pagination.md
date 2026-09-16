---
book: The Complete Modern PHP Engineering Book
volume: 8
volume_title: DATABASES
chapter: 119
title: Pagination
slug: pagination
status: complete
summary: ../../_ai/chapter-summaries/119-pagination-summary.md
---

# Chapter 119 — Pagination

## Why This Matters

An endpoint that returns every matching row works on a small fixture and fails when the table grows. It consumes database, PHP, network, and client memory in proportion to the total result set. Pagination bounds one response, but the pagination strategy also determines query cost, ordering stability, duplicates, and what happens while rows are inserted or deleted.

Pagination is a contract between the database query and the API. Define the order, page size limits, cursor shape, consistency expectations, and behavior when a row disappears before implementing the controller.

## Offset Pagination

Offset pagination is easy to expose:

```sql
SELECT id, created_at, title
FROM articles
WHERE status = 'published'
ORDER BY created_at DESC, id DESC
LIMIT :limit OFFSET :offset;
```

The client requests page number and size, and the server computes the offset. The `id` tie-breaker is essential: ordering only by `created_at` does not define a deterministic order when timestamps are equal.

Offset pagination is understandable and useful for small, mostly static result sets. Its cost often grows with the offset because the database must locate or scan past preceding rows. Pages can also shift while rows are inserted or deleted: an item may appear twice or be skipped between requests.

Never accept an unlimited client-provided size. Clamp it to a server policy and validate numeric input before binding it. Parameter binding syntax for `LIMIT` and `OFFSET` varies by PDO driver, so use validated integers and the driver's documented behavior; never concatenate untrusted text into SQL.

## Keyset or Cursor Pagination

Keyset pagination starts after the last ordered key rather than counting preceding rows. With the order `(created_at DESC, id DESC)`, the next page condition is:

```sql
SELECT id, created_at, title
FROM articles
WHERE status = 'published'
  AND (created_at, id) < (:after_created_at, :after_id)
ORDER BY created_at DESC, id DESC
LIMIT :limit;
```

Row-value comparison syntax is supported by PostgreSQL and modern MySQL, but the equivalent disjunction is clearer and more portable:

```sql
AND (
    created_at < :after_created_at
    OR (created_at = :after_created_at AND id < :after_id)
)
```

The cursor contains the complete position in the ordering, not merely a timestamp. Encode it as an opaque, authenticated token at the API boundary. A client should not be able to alter a cursor to read a different tenant or bypass a filter. The server validates the cursor version, filter scope, direction, and key types before using it.

## A Typed Cursor Boundary

Keep cursor decoding separate from SQL construction:

```php
<?php

declare(strict_types=1);

final readonly class ArticleCursor
{
    public function __construct(
        public string $createdAt,
        public int $id,
    ) {
    }
}

function decodeCursor(string $token, string $secret): ArticleCursor
{
    $json = base64_decode($token, true);
    if ($json === false) {
        throw new InvalidArgumentException('Malformed cursor');
    }

    $payload = json_decode($json, true, flags: JSON_THROW_ON_ERROR);
    if (!is_array($payload) || !isset($payload['created_at'], $payload['id'])) {
        throw new InvalidArgumentException('Invalid cursor payload');
    }

    // Production tokens should include and verify an HMAC over the payload.
    if (!hash_equals(hash_hmac('sha256', $json, $secret), (string) ($payload['signature'] ?? ''))) {
        throw new InvalidArgumentException('Invalid cursor signature');
    }

    return new ArticleCursor((string) $payload['created_at'], (int) $payload['id']);
}
```

The example shows the boundary, not a complete cursor format. In a real implementation, sign the exact canonical payload and keep the signature outside the signed data or verify a documented envelope. Include the filter and sort direction in the signed payload when a token is reusable across requests. Do not expose internal database details that the API cannot support long term.

## Stable Ordering and Indexes

The `WHERE`, `ORDER BY`, and cursor predicate must agree. An index such as `(status, created_at DESC, id DESC)` can help the database find the first matching row and continue from a cursor. The useful index depends on selectivity, engine, and other query patterns; confirm it with the plan rather than adding every possible ordering.

If the ordering key can change while a client paginates, an item may move behind or ahead of the cursor. Prefer immutable keys such as creation sequence for feeds, or define the feed as a snapshot with a cutoff value. If the result set must be a consistent report, use a snapshot or materialized export rather than pretending cursor pagination provides a transaction spanning multiple HTTP requests.

## Page Size, Counts, and the Last Page

A `COUNT(*)` for every page can be more expensive than the page query, especially for complex filters. Decide whether the API needs an exact total, an estimate, or only `has_more`. Fetch `limit + 1` rows, return the first `limit`, and derive `has_more` from the extra row. This avoids a separate count in many feeds.

For administrative screens that require page numbers, an exact count may be appropriate. Cache or precompute it only when staleness is acceptable. State whether totals are approximate and whether they share the same consistency boundary as the rows.

## API Contract and Failure Modes

Document whether the first request omits a cursor, how invalid or expired cursors are reported, the maximum page size, and whether a deleted item can cause a shorter page. A cursor for a different tenant, filter, or sort must be rejected rather than silently interpreted.

For backward navigation, either support a reverse query and reverse the returned rows in application code or issue a separate cursor format. Do not fetch a large prefix merely to simulate “previous page.” For exports, a streaming job or snapshot may be a better contract than HTTP pagination.

## Common Mistakes

- Ordering by a non-unique column without a tie-breaker.
- Assuming offset cost remains constant for deep pages.
- Returning an unbounded page size.
- Treating an opaque cursor as trusted input without authenticating and validating it.
- Using a cursor that omits one of the order keys.
- Promising a stable multi-request snapshot without a snapshot design.
- Running an exact count on every request without measuring its cost.
- Concatenating page parameters into SQL.

## Testing

Test the first page, an empty result, the final short page, an invalid cursor, a cursor with the wrong filter, duplicate sort keys, and a row inserted between page requests. Generate rows with equal timestamps and verify that the tie-breaker prevents duplicates or gaps under the chosen consistency contract. Use query-plan tests or integration checks for the important index-backed paths, not only controller unit tests.

## Senior Engineer Thinking

Pagination is a query and consistency problem disguised as an API parameter. Select offset for simple, shallow, mostly static navigation; select keyset for large ordered feeds and predictable deep-page cost; select a snapshot or export when the consumer needs a stable report. State the trade-off instead of presenting one method as universal.

## Exercises

1. Convert an offset query ordered by `created_at` into a keyset query with an `id` tie-breaker.
2. Design and sign a cursor that includes filter scope and sort direction.
3. Compare `COUNT(*)` plus a page query with `LIMIT + 1` for a feed endpoint.
4. Create a test that inserts a row between two page requests and explain the result under offset and keyset pagination.

## Review Questions

1. Why must a pagination order be total and deterministic?
2. What causes deep offset pages to become expensive?
3. What keys belong in a cursor?
4. When is an exact total worth its extra query cost?
5. Why does keyset pagination not automatically provide a consistent snapshot?

## Summary

Pagination bounds response size, but its ordering and consistency contract determine correctness. Offset pagination is simple but can scan deeply and shift as rows change. Keyset pagination uses a complete ordered cursor for predictable continuation, provided the query, index, and cursor validation agree. Cap page sizes, avoid unsafe SQL concatenation, and test inserts, deletes, ties, and invalid cursors.

## References

- [PostgreSQL: `LIMIT` and `OFFSET`](https://www.postgresql.org/docs/current/queries-limit.html)
- [MySQL: `LIMIT` optimization](https://dev.mysql.com/doc/refman/8.4/en/limit-optimization.html)
- [PHP Manual: PDO prepared statements](https://www.php.net/manual/en/pdo.prepared-statements.php)
