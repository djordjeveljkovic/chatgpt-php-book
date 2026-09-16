---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 214
title: Laravel ORM
slug: laravel-orm
status: complete
summary: ../../_ai/chapter-summaries/214-laravel-orm-summary.md
---

# Chapter 214 — Laravel ORM

## Why This Matters

Laravel's Eloquent ORM maps PHP model objects to database rows and provides query builders, relationships, casts, scopes, transactions, and lifecycle hooks. It can make ordinary persistence productive, but an expressive model call can still execute an expensive query, load stale data, or bypass an authorization and invariant boundary.

Use Eloquent as a persistence tool. Keep domain rules and transaction decisions explicit, inspect generated SQL and query plans, and avoid returning ORM models as an accidental public API contract. Method names and configuration details can vary by Laravel version; verify examples against the target release.

## Models and Explicit Attributes

A model represents a table mapping and may carry casts, relationships, scopes, and persistence behavior. Treat request input as untrusted and allow-list fields for mass assignment. Prefer explicit command mapping over accepting every request key.

~~~php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

final class Reservation extends Model
{
    protected $table = 'reservations';

    protected $casts = [
        'starts_at' => 'immutable_datetime',
        'ends_at' => 'immutable_datetime',
        'amount_minor' => 'integer',
    ];

    protected $fillable = [
        'court_id',
        'starts_at',
        'ends_at',
    ];

    protected $guarded = [
        'status',
        'tenant_id',
        'user_id',
    ];
}
~~~

Do not rely on both fillable and guarded as a complete authorization system. Assign tenant, owner, status, and approval fields from server-side policy and domain commands. A model's casts transform values for PHP access; they do not necessarily enforce database constraints or API formats.

## Queries and Query Cost

Eloquent queries are lazy until executed by methods such as get, first, exists, count, or a write operation. A chain can look small while producing a table scan, a large result set, or a query per row. Use indexes that match filters and ordering, select only required columns for read paths, and inspect plans in the target database.

~~~php
<?php

use App\Models\Reservation;
use Illuminate\Support\Collection;

function openReservations(int $tenantId, int $customerId): Collection
{
    return Reservation::query()
        ->where('tenant_id', $tenantId)
        ->where('customer_id', $customerId)
        ->whereIn('status', ['held', 'confirmed'])
        ->orderBy('starts_at')
        ->limit(100)
        ->get(['id', 'court_id', 'starts_at', 'ends_at', 'status']);
}
~~~

The query is tenant-scoped and bounded, but the business policy may need a cursor rather than a fixed limit. Do not call get on an unbounded administrative or export query. Use chunking, lazy iteration, or a dedicated batch process while respecting consistency and mutation rules.

## Relationships and N plus 1

Relationships provide a readable way to navigate related data, but property access can trigger lazy queries. Loading reservations and then accessing customer or court for every row creates N plus 1 queries. Eager load only the relationships needed for the response and verify query count in tests.

~~~php
$orders = Order::query()
    ->with(['customer:id,name', 'lines:id,order_id,sku,quantity'])
    ->where('tenant_id', $tenantId)
    ->latest('id')
    ->paginate(50);
~~~

The exact selected columns must include keys needed to connect the relationship. Eager loading can also load too much data; choose a projection or query join for a report. Prevent lazy loading in development or test environments when the project supports that option, then fix the query rather than adding a blanket eager load.

## Scopes, Resources, and Boundaries

A local scope can name a reusable query condition such as open status or tenant filtering. A scope should not hide an authorization assumption that callers might forget. Tenant scope is safest when repository or policy boundaries make it difficult to omit.

Map models to API resources or response DTOs. Serializing a model can expose columns or relationships added later, trigger lazy queries, and couple clients to database names. Explicit resources also provide a place to redact internal IDs and apply authorization.

## Writes, Transactions, and Concurrency

A model save does not define the transaction for a multi-step use case. Wrap state changes that must commit together in an application or service transaction. Add database constraints and conditional updates for invariants that concurrent requests can violate.

~~~php
use Illuminate\Support\Facades\DB;

function confirmReservation(int $tenantId, int $reservationId): void
{
    DB::transaction(function () use ($tenantId, $reservationId): void {
        $reservation = Reservation::query()
            ->where('tenant_id', $tenantId)
            ->whereKey($reservationId)
            ->lockForUpdate()
            ->firstOrFail();

        if ($reservation->status !== 'held') {
            throw new DomainException('Reservation is not held');
        }

        $reservation->status = 'confirmed';
        $reservation->save();
    });
}
~~~

The example locks a row while the transaction is open. It still needs an application-level authorization check and any database constraint for court/time conflicts. Avoid calling an external provider inside this transaction; record an outbox effect and deliver after commit.

## Model Events, Observers, and Casts

Model events and observers can centralize persistence-adjacent behavior, but hidden side effects make writes difficult to reason about. Use them for narrow, consistent concerns such as cache invalidation or audit capture when their ordering and failure policy are documented. Keep critical business transitions in explicit application or domain commands.

Attribute casts should be deterministic and compatible with stored data. A cast is not validation of an incoming request, authorization, or a migration. Test nulls, old formats, timezone conversion, enum evolution, and serialization.

## Soft Deletes, Migrations, and Legacy Data

Soft deletes change query semantics: default queries may omit rows while administrative or uniqueness queries need deleted records. Define whether a deleted record can be recreated, restored, or referenced by an event. A soft delete is not a data-retention policy by itself.

Migrations must support mixed application versions during deployment. Add compatible columns, deploy readers and writers, backfill, verify, then remove old paths. Eloquent model code should tolerate the intermediate schema only for the planned window and should fail safely when required data is absent.

## Testing and Operations

Use unit tests for domain policy and request mapping, database tests for model casts, relationships, constraints, transactions, scopes, and query behavior, and API tests for response resources and authorization. Factories help create fixtures but should not bypass tenant or state invariants.

Detect N plus 1 queries and unbounded loads in development and CI. Monitor query count, latency, rows read, lock waits, connection saturation, and slow query samples. Redact SQL bindings that contain credentials or personal data. Use the real database engine for isolation and constraint behavior; SQLite is not always semantically equivalent to production.

## Common Mistakes

- Treating Eloquent models as the domain model and API schema simultaneously.
- Allowing mass assignment of tenant, owner, status, or approval fields.
- Ignoring lazy loading and N plus 1 queries.
- Loading unbounded collections into memory.
- Assuming save is an atomic business transaction.
- Calling external services inside an open database transaction.
- Using model events for hidden critical business effects.
- Treating casts or soft deletes as complete validation or retention policy.

## Senior Engineer Thinking

Eloquent is productive when its query, mapping, and lifecycle behavior remain visible. Scope data and writes by tenant and invariant, inspect SQL and plans, define transactions outside one-row saves, map public responses explicitly, and test the real database behavior that the ORM abstracts.

## Exercises

1. Refactor a controller that mass-assigns request data into a validated command and explicit model mapping.
2. Find an N plus 1 relationship query and replace it with a bounded projection or selective eager load.
3. Add a transaction and conditional conflict check to a reservation confirmation flow.
4. Design a mixed-version migration for adding a required model attribute.

## Review Questions

1. What does an ORM model abstraction hide, and what must remain visible?
2. Why can relationship property access create N plus 1 queries?
3. Which fields should never be mass-assigned from a request?
4. Why is a model save not necessarily a business transaction?
5. What are the limits of casts and soft deletes?
6. Why should API resources be separate from ORM models?

## Summary

Laravel's Eloquent ORM maps models and relationships to database operations, but it does not remove query cost, authorization, constraints, transactions, or migration concerns. Allow-list writes, scope tenants, bound and inspect queries, prevent N plus 1 behavior, map API responses explicitly, and test the real database semantics behind the ORM.

## References

- [Laravel documentation: Eloquent ORM](https://laravel.com/docs/eloquent)
- [Laravel documentation: Eloquent relationships](https://laravel.com/docs/eloquent-relationships)
- [Laravel documentation: Database transactions](https://laravel.com/docs/database)
- [Laravel documentation: Eloquent resources](https://laravel.com/docs/eloquent-resources)
- [OWASP Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)

