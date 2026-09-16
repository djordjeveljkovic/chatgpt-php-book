---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 212
title: Laravel Container
slug: laravel-container
status: complete
summary: ../../_ai/chapter-summaries/212-laravel-container-summary.md
---

# Chapter 212 — Laravel Container

## Why This Matters

Laravel's service container resolves classes and bindings, injects dependencies, manages lifetimes, and provides the composition mechanism used by much of the framework. Understanding it turns “Laravel magic” into a set of explicit construction rules.

The container is a wiring tool. It should not become a hidden service locator used throughout domain code. Type-hint stable contracts, bind volatile implementations at the edge, and make lifetimes and configuration visible.

## Automatic Resolution

Concrete classes with resolvable constructors can often be built through reflection without an explicit binding. Interfaces and primitives need instructions:

~~~php
<?php

declare(strict_types=1);

final class InvoiceService
{
    public function __construct(
        private InvoiceRepository $invoices,
        private Clock $clock,
    ) {
    }
}
~~~

Laravel can resolve InvoiceService when the container knows InvoiceRepository and Clock. If a constructor requires a scalar or an unbound interface, add a binding, configuration value, contextual rule, or factory. Prefer typed configuration objects over reading environment variables in service methods.

## Bindings and Lifetimes

A provider's register method is the normal place to bind services. The exact API is documented by the installed Laravel version, but the concepts are stable:

~~~php
<?php

use Illuminate\Support\ServiceProvider;

final class ApplicationServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(
            PaymentGateway::class,
            StripePaymentGateway::class,
        );

        $this->app->singleton(
            FeatureFlags::class,
            fn ($app): FeatureFlags => new FeatureFlags(
                $app['config']->get('features'),
            ),
        );
    }
}
~~~

A transient binding creates a new instance when resolved according to container rules. A singleton reuses one instance for the application/container lifetime. Laravel also provides scoped lifetimes for a request or job lifecycle in supported versions. Choose the lifetime from state and resource ownership, not convenience.

A singleton must not retain request user, tenant, mutable command, or correlation data. Long-running workers make accidental process lifetime more dangerous than short PHP-FPM requests.

## Interfaces and Contextual Bindings

Bind an interface to an implementation when the application depends on a capability:

~~~php
<?php

$this->app->bind(
    PaymentGateway::class,
    fn ($app): PaymentGateway => new StripePaymentGateway(
        http: $app->make(HttpClient::class),
        clock: $app->make(Clock::class),
    ),
);
~~~

Contextual binding selects a different implementation for a specific consumer. Use it when the difference is a real policy, such as read versus write storage or a provider selected by a bounded context. Too many contextual rules make the object graph difficult to discover; a named factory or explicit constructor can be clearer.

Do not bind every concrete class merely to make the container visible. Automatic resolution is often enough for stable application classes.

## Tags, Factories, and Decoration

A tag can group multiple implementations for a consumer such as a notification dispatcher. A factory can create a value based on validated input. Decoration can add metrics, tracing, authorization, caching, or retry policy, but ordering and idempotency must be explicit.

Keep container configuration declarative and cheap. A binding closure should construct an object, not perform a network call, migrate a database, or inspect a current request. Defer expensive work until the operation that needs it and give that work a timeout.

## Container and Application Boundaries

Controllers, jobs, listeners, and commands are reasonable container entry points. Resolve an application service there, then pass typed commands and actors. Domain objects should receive their dependencies through constructors or methods, not call the global container.

A facade or helper can be convenient for framework services, but it hides the dependency from a class's constructor. Use explicit injection when a service's behavior depends on the collaborator or when unit testing needs a controlled substitute. Keep global access at composition or transport edges.

## Testing the Container

Test that important bindings resolve with valid configuration, interfaces map to the intended implementation, contextual rules choose the correct policy, and scoped services do not leak state across jobs. Use integration tests with the real application container for wiring; unit-test application rules with direct constructors and fakes.

A container test should not assert every internal binding. Assert the contracts that deployment and behavior depend on. A smoke bootstrap test catches missing providers, configuration, and autoloading before a request reaches production.

## Failure and Security

A missing binding is an application startup or request failure; do not catch it and silently choose an insecure default. Validate required configuration early and fail clearly. Do not bind a permissive fake, debug logger, or development credential in production through an ambiguous environment value.

Secrets should be supplied through configuration and scoped credentials, not captured in publicly inspectable service objects or logged by resolution diagnostics. Binding a privileged implementation is an authorization decision; review providers and module ownership accordingly.

## Failure and Threat Analysis

* **Service locator drift:** application classes call the container everywhere. Use constructor injection and explicit commands.
* **Wrong lifetime:** a singleton retains tenant or request state. Use transient/scoped lifetime and reset worker context.
* **Unbound primitive:** a default timeout or URL is silently wrong. Validate typed configuration.
* **Context mismatch:** one implementation is used for all modules. Use an explicit policy or contextual binding.
* **Boot I/O:** binding performs network or database work. Defer it to a bounded operation.
* **Insecure fallback:** a missing provider selects an allow-all implementation. Fail closed.
* **Configuration cache:** stale values outlive secret rotation. Define deployment and rotation policy.

## Exercises

1. Bind a PaymentGateway interface and write a container integration test for its production implementation.
2. Compare singleton, scoped, and transient lifetimes for a tenant-aware service in PHP-FPM and queue workers.
3. Replace a service-locator call in a domain service with constructor injection.
4. Add a smoke bootstrap test that fails when a required configuration value or provider binding is missing.

## Review Questions

1. What can Laravel resolve automatically?
2. When should an interface be bound?
3. How do singleton and scoped lifetimes differ in long-running workers?
4. Why should binding closures avoid I/O?
5. Which classes are reasonable container entry points?
6. How can container configuration become a security boundary?

## Summary

Laravel's container is a composition and lifetime mechanism. Use automatic resolution for simple concrete classes, bind interfaces at providers, choose lifetimes from ownership, inject dependencies into application and domain code, keep binding closures cheap, validate configuration, and test important wiring with the real container.

## References

- [Laravel Service Container](https://laravel.com/docs/container)
- [Laravel Service Providers](https://laravel.com/docs/providers)
- [Laravel Configuration](https://laravel.com/docs/configuration)
- [PHP Manual: Anonymous Functions](https://www.php.net/manual/en/functions.anonymous.php)
- [PHP-FIG PSR-11: Container Interface](https://www.php-fig.org/psr/psr-11/)

