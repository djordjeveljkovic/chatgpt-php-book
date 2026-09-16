---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 210
title: Laravel Overview
slug: laravel-overview
status: complete
summary: ../../_ai/chapter-summaries/210-laravel-overview-summary.md
---

# Chapter 210 — Laravel Overview

## Why This Matters

Laravel is a PHP application framework that combines routing, middleware, a service container, configuration, validation, authentication, database and queue integrations, views, and testing tools. It offers conventions that let a team build a complete application quickly, while still allowing the underlying PHP and HTTP behavior to remain visible.

Use Laravel as a set of boundaries and tools. A controller is not the domain, an Eloquent model is not automatically a public API, and a facade call does not eliminate dependency or failure design.

## Application Shape

A Laravel application typically has a public entry point, bootstrap and configuration, application code, routes, database migrations, resources, and tests. The exact directory contents and bootstrap APIs vary by Laravel version and project starter, so use the installed version's generated application as the authority.

A useful conceptual flow is:

~~~text
route
  -> middleware
  -> controller or invokable action
  -> request validation and authorization
  -> application/domain service
  -> Eloquent/query/repository boundary
  -> resource/response
~~~

The framework can make each step convenient; the application must still define ownership, authorization, transaction boundaries, and error contracts.

## Routes, Controllers, and Requests

Routes map an HTTP method and URI to a handler. Route model binding can turn an identifier into a model, but visibility and tenant authorization still belong to the operation. Form Request classes can combine validation and authorization checks; keep domain invariants in a domain or application operation so jobs and commands enforce them too.

~~~php
<?php

use Illuminate\Support\Facades\Route;

Route::post('/invoices', [InvoiceController::class, 'store'])
    ->middleware(['auth', 'throttle:invoices']);

final class InvoiceController
{
    public function store(StoreInvoiceRequest $request, CreateInvoice $create): JsonResponse
    {
        $this->authorize('create', Invoice::class);

        $invoice = $create->handle(
            actor: $request->user(),
            amountCents: $request->integer('amount_cents'),
        );

        return response()->json(
            ['id' => $invoice->id],
            201,
        );
    }
}
~~~

The classes and middleware names are illustrative; import and registration details depend on the application. The important boundary is that the request is authenticated and validated, the actor is server-derived, the operation performs authorization and domain checks, and the response exposes an intentional representation.

## Service Providers and the Container

Service providers are a central Laravel extension point for registering bindings and bootstrapping application services. Keep registration in register and runtime work that depends on other bindings in boot, following the framework's documented lifecycle. Do not open a database connection or make a remote call merely because a provider is loaded.

Use bindings for contracts with a meaningful implementation boundary:

~~~php
<?php

use Illuminate\Support\ServiceProvider;

final class BillingServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(PaymentGateway::class, function ($app): PaymentGateway {
            return new StripePaymentGateway(
                http: $app->make(HttpClient::class),
                clock: $app->make(Clock::class),
            );
        });
    }

    public function boot(): void
    {
        // Register policies or integration hooks that need bootstrapped services.
    }
}
~~~

Keep provider code small. A binding should not hide a missing configuration value, an unbounded client, or a cross-module dependency. See [Chapter 212](212-laravel-container.md) for resolution, lifetimes, contextual bindings, and testing.

## Eloquent, Queries, and Transactions

Eloquent provides an active-record style model and relationship/query APIs. It is useful when its conventions match the domain, but lazy relationships can create N+1 queries and model attributes can accidentally become a response contract. Use explicit eager loading, resource classes, selected columns, and authorization scopes.

Place transaction boundaries around a use case rather than around an arbitrary model method:

~~~php
<?php

use Illuminate\Support\Facades\DB;

$invoice = DB::transaction(function () use ($createInvoice, $command): Invoice {
    $invoice = $createInvoice->handle($command);
    DB::afterCommit(fn () => event(new InvoiceCreated($invoice->id)));

    return $invoice;
});
~~~

The exact after-commit event configuration depends on the Laravel version and queue setup. An event listener that sends email must still be idempotent and observable. Do not hold a transaction open around a remote payment call.

## Queues, Jobs, and Workers

Jobs move work out of the request, but they can run late, twice, or on a different code version. Make payloads small and versioned, re-check authorization and current state, set retries and backoff deliberately, and make side effects idempotent. A queue job needs timeout, visibility, failure, and dead-letter policies.

Long-running workers have different memory and state behavior from short PHP-FPM requests. Reset per-job state, avoid static request data, and measure memory and queue age.

## Testing and Operations

Use Laravel's feature testing tools for routes, middleware, validation, authentication, database effects, and response contracts. Use unit tests for framework-independent rules and integration tests for Eloquent, queues, mail, storage, and providers. Test query count or eager-loading behavior where performance matters.

Cache configuration/routes only according to the deployment process, and never cache secrets or environment values in a way that outlives their rotation policy. Monitor request latency, database queries, queue age, failed jobs, exception classes, and provider calls. Redact tokens, cookies, passwords, and personal data.

## Failure and Threat Analysis

* **Mass assignment:** request fields set protected model attributes. Use validated allow-lists and explicit mapping.
* **IDOR:** route model binding finds another tenant's record. Apply scoped queries and policy checks.
* **N+1:** lazy relationships issue queries in loops. Eager-load and measure.
* **Job replay:** a worker runs a command twice. Use idempotency and durable state checks.
* **Provider side effect in boot:** a service provider makes remote calls during startup. Keep boot deterministic.
* **Global facade dependence:** tests and workers share hidden state. Inject contracts where policy matters.
* **Stale config:** cached configuration ignores rotated secrets. Define deploy and rotation procedures.

## Exercises

1. Trace a Laravel route through middleware, Form Request, policy, controller, service, model, and response.
2. Refactor a controller that passes an Eloquent model directly to JSON into an explicit response resource.
3. Design a queued invoice job with retry, backoff, idempotency, authorization, and failure recording.
4. Measure and fix an N+1 relationship query in a feature test.

## Review Questions

1. What does Laravel provide beyond PHP and an HTTP server?
2. Why is an Eloquent model not automatically a public API representation?
3. What belongs in a service provider's register method?
4. Why must queued jobs be safe to run twice?
5. Where should a Laravel transaction boundary live?
6. Which framework metrics help diagnose application failures?

## Summary

Laravel provides conventions and integrations for routing, middleware, containers, providers, validation, persistence, queues, and testing. Keep domain and application decisions explicit, treat framework models and helpers as boundaries, make jobs and events idempotent, scope authorization, measure hidden queries and worker behavior, and follow the installed version's documentation for bootstrap details.

## References

- [Laravel Documentation](https://laravel.com/docs)
- [Laravel Directory Structure](https://laravel.com/docs/structure)
- [Laravel Service Providers](https://laravel.com/docs/providers)
- [Laravel Database Transactions](https://laravel.com/docs/database#database-transactions)
- [Laravel Queues](https://laravel.com/docs/queues)

