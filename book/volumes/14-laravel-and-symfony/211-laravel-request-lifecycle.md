---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 211
title: Laravel Request Lifecycle
slug: laravel-request-lifecycle
status: complete
summary: ../../_ai/chapter-summaries/211-laravel-request-lifecycle-summary.md
---

# Chapter 211 — Laravel Request Lifecycle

## Why This Matters

When a Laravel request behaves unexpectedly, the useful question is often “which lifecycle stage changed this value or decision?” The application is bootstrapped, service providers register bindings, middleware wraps the request, routing selects a handler, the handler returns a response, and middleware runs on the way out.

Understanding this path explains why a binding is unavailable during provider boot, why middleware can short-circuit a request, why a queued job does not share request state, and why a response may be modified after a controller returns.

## Entry and Bootstrap

The web server points the document root at the application's public entry point. Composer's autoloader is loaded, and the application bootstrap creates the framework application/container. Bootstrap code reads configuration, prepares exception handling and logging, and registers service providers and middleware according to the installed Laravel version.

Do not assume every request rebuilds every expensive service in the same way. PHP-FPM requests have process reuse and OPcache, while long-running workers retain process state between jobs. Keep boot deterministic, avoid remote calls in bootstrap, and make configuration validation failures visible.

## Providers and Registration

Service providers are a central place to register bindings, event listeners, routes, and integration configuration. The register phase should bind services; boot runs after providers are registered and can use bindings that other providers supplied. Exact provider discovery and registration files vary across Laravel releases and application skeletons.

~~~php
<?php

use Illuminate\Support\ServiceProvider;

final class SearchServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(SearchClient::class, function ($app): SearchClient {
            return new SearchClient(
                baseUrl: $app['config']->get('search.url'),
                timeoutSeconds: 2.0,
            );
        });
    }

    public function boot(): void
    {
        // Register policies or publish integration metadata.
    }
}
~~~

A provider should not query the database, call a remote service, or inspect the current user while the application is booting. Those operations belong to a request, command, or job where timeouts and identity are defined.

## Middleware Pipeline

Middleware wraps the request in an ordered pipeline. A middleware may inspect headers, start a session, authenticate, enforce CSRF, apply rate limits, add a correlation ID, or return a response without invoking the next handler. After the handler returns, it can modify the outgoing response.

~~~php
<?php

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class CorrelationId
{
    public function handle(Request $request, Closure $next): Response
    {
        $id = $request->header('X-Request-Id') ?: bin2hex(random_bytes(16));

        $response = $next($request);

        return $response->header('X-Request-Id', $id);
    }
}
~~~

Treat inbound correlation IDs as untrusted input: validate length and character policy, or generate a safe one. Do not log authorization headers or cookies with the correlation ID. Middleware order matters; authentication must occur before a policy requiring an actor, and a body-size limit should run before expensive parsing where possible.

## Routing and Handler Resolution

The router matches method and URI, applies route middleware, resolves route parameters, and invokes a controller, closure, or invokable action. Dependency injection resolves handler arguments through the container. Route model binding can simplify lookup but must be scoped for tenant visibility and followed by policy authorization.

A controller should translate the framework request to a command or query and map a domain result to a response. It should not rely on a route parameter or hidden form field as proof of ownership. A queue job or CLI command may invoke the same application service without passing through HTTP middleware, so authorization belongs below the transport boundary too.

## Response and Exception Flow

A handler can return a response, a view, a redirect, or a value Laravel converts into a response according to its conventions. Exceptions travel through the framework's exception handling and reporting configuration. A public response should use safe status and problem data; detailed traces belong in protected logs and error reporting.

Outbound middleware runs after the handler in reverse nesting order. It can add headers, save sessions, or transform a response. Be careful with response caching: a shared cache must not store a personalized response without explicit isolation and authorization policy.

## Jobs, CLI, and Long-Running Workers

A queue worker and Artisan command use application bootstrap but do not have an HTTP request, browser cookies, or a request-scoped user. Pass explicit actor or service identity, correlation ID, tenant, and command data. Do not read a stale global request object from a job.

Long-running workers retain objects and static state. Release large references, reset per-job context, and measure memory. A container binding scoped to a request or job lifecycle should not be treated as a permanent singleton. Test jobs in a worker-like process when state leakage matters.

## Debugging the Lifecycle

When debugging, inspect:

1. public entry point and application bootstrap;
2. configuration cache and environment source;
3. provider registration and binding lifetime;
4. middleware order and short-circuit responses;
5. route match and parameter binding;
6. handler dependencies and transaction boundary;
7. exception mapping and outbound middleware;
8. queue or worker boundaries after the response.

Use framework route and container inspection commands in a non-production environment. Add a safe correlation ID, query timing, and middleware diagnostics rather than dumping credentials or full request bodies.

## Failure and Threat Analysis

* **Provider side effect:** boot performs a remote call or database query. Keep boot deterministic and bounded.
* **Middleware order:** a rate limit or authorization check runs too late. Test the pipeline and route groups.
* **Trusting headers:** an arbitrary request ID or forwarded address controls security. Configure trusted proxies and validate headers.
* **Route binding leak:** a model is resolved outside tenant scope. Use scoped binding and policy checks.
* **Exception disclosure:** production response includes stack traces or SQL. Map safe errors and protect logs.
* **Worker state leak:** request identity or static caches persist into another job. Reset context and test reuse.
* **Cache leakage:** a personalized response enters a shared cache. Set explicit cache directives and keys.

## Testing the Lifecycle

Feature tests should send requests through the real router and middleware, asserting authentication, CSRF, headers, status, response shape, and short-circuit behavior. Integration tests should verify provider bindings, configuration, database and queue adapters. A unit test for a controller method cannot prove provider order or middleware behavior.

Add a smoke test for production-like bootstrap, a readiness check for dependencies, and a worker test for one process handling multiple jobs. Test exception mapping with debug disabled and ensure logs contain correlation data without secrets.

## Exercises

1. Trace one request and draw the inbound and outbound middleware order.
2. Add a correlation-ID middleware with length validation and a generated fallback.
3. Find a provider boot method that performs I/O and move it to an explicit command, request, or job.
4. Write a feature test proving a wrong-tenant route binding returns the documented visibility response.
5. Run two queue jobs in one worker process and detect leaked request or tenant state.

## Review Questions

1. What happens before a Laravel route handler runs?
2. Why should provider registration avoid I/O?
3. How can middleware change both inbound and outbound behavior?
4. Why is route model binding not authorization?
5. Which state is absent from a queue job?
6. What lifecycle bugs appear only in long-running workers?

## Summary

Laravel boots an application and container, registers providers, passes requests through ordered middleware, resolves routes and handlers, maps responses and exceptions, and reuses related bootstrap in jobs and CLI commands. Keep boot deterministic, validate trust-boundary inputs, authorize below HTTP, make middleware order explicit, and test both short requests and long-running worker behavior.

## References

- [Laravel Request Lifecycle](https://laravel.com/docs/lifecycle)
- [Laravel Middleware](https://laravel.com/docs/middleware)
- [Laravel Service Providers](https://laravel.com/docs/providers)
- [Laravel Queue Workers](https://laravel.com/docs/queues#running-the-queue-worker)
- [PHP Manual: Sessions](https://www.php.net/manual/en/book.session.php)

