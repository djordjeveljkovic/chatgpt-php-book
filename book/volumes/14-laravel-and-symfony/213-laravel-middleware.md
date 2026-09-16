---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 213
title: Laravel Middleware
slug: laravel-middleware
status: complete
summary: ../../_ai/chapter-summaries/213-laravel-middleware-summary.md
---

# Chapter 213 — Laravel Middleware

## Why This Matters

Laravel middleware wraps HTTP request handling with reusable policy. It can authenticate a request, apply rate limits, establish tenant context, add security headers, transform a request, or record timing around the next layer. The pipeline is powerful because the same concern can apply to many routes, but ordering and scope determine the actual behavior.

Middleware is transport-bound infrastructure. Business invariants belong in application and domain services, while middleware should decide whether a request may proceed and how the HTTP boundary is observed. Names and registration APIs vary across Laravel versions; consult the documentation for the target major version while keeping the pipeline responsibilities stable.

## The Pipeline Shape

A middleware receives a request and a next callable. It may reject the request, modify it before calling the next layer, modify the response afterward, or perform cleanup in a termination hook where the runtime supports one.

~~~php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class AddCorrelationId
{
    public function handle(Request $request, Closure $next): Response
    {
        $incoming = $request->header('X-Correlation-Id');
        $correlationId = is_string($incoming) && preg_match(
            '/^[A-Za-z0-9._-]{1,100}$/D',
            $incoming,
        ) === 1
            ? $incoming
            : bin2hex(random_bytes(16));

        $request->attributes->set('correlation_id', $correlationId);
        $response = $next($request);
        $response->headers->set('X-Correlation-Id', $correlationId);

        return $response;
    }
}
~~~

A correlation ID is for tracing, not authentication. Validate its size and character set, avoid trusting it as a user identity, and do not reflect an arbitrary header into logs or responses. The application logger and downstream clients should carry the same safe identifier.

## Scope and Ordering

Laravel applications typically provide global middleware, named groups, and route-specific middleware. Use the narrowest scope that matches the policy:

- global middleware for concerns required by nearly every HTTP request;
- a group for browser or API defaults;
- route middleware for a particular capability or resource;
- controller or application policy checks when authorization depends on loaded domain state.

Ordering is behavior. A body-size limit should run before expensive parsing; authentication must establish identity before an authorization check; tenant resolution must precede tenant-scoped queries; exception handling and tracing should surround the work they observe. Middleware order can differ by framework version and application bootstrap style, so verify it with a route test.

Avoid putting a database query in global middleware when only a few routes need it. It increases latency and can make health checks depend on application data. A middleware that loads the current user does not prove that a controller authorizes the specific order or tenant.

## Authentication and Authorization

Authentication middleware establishes a principal. Authorization middleware or policies decide whether that principal may perform an action on a particular resource. Keep object-level checks close to the loaded model or application service:

~~~php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class RequireJson
{
    public function handle(Request $request, Closure $next): Response
    {
        if (!$request->expectsJson() && !$request->isJson()) {
            return response()->json([
                'type' => 'https://example.test/problems/content-type',
                'title' => 'JSON is required',
                'status' => 415,
            ], 415);
        }

        return $next($request);
    }
}
~~~

Do not use a content-type middleware as a substitute for schema validation. Reject an unsupported representation early, then validate fields in the request object or application boundary. For cookie-authenticated browser mutations, CSRF middleware and SameSite policy remain relevant; API middleware does not automatically eliminate replay or authorization concerns.

## Request and Response Mutation

A middleware may add attributes or headers, but the contract should be explicit. Avoid silently changing user input in ways a controller cannot see. If you normalize an identifier, use a typed request object or value object and document the rule. Never place credentials, raw request bodies, or personal data in correlation headers.

Response headers can add cache policy, security policy, or tracing metadata. Apply cache directives according to authenticated and public representations. A middleware that adds a public cache directive to an authenticated response can leak tenant data through shared caches.

## Rate Limits, Maintenance, and Short-Circuiting

Rate limiting, maintenance mode, feature gates, and request size checks are natural middleware concerns because they can reject before expensive work. Their state must be shared correctly across workers and must distinguish identity dimensions. A static counter in one PHP process is not a distributed limiter.

Short-circuit responses need stable status codes and error representations. A 429 response should include a useful retry policy when known; maintenance responses should not accidentally expose debug information. Ensure health and readiness routes have an intentional middleware policy rather than bypassing every security control.

## Exceptions, Transactions, and Long Work

Exception rendering belongs near the HTTP boundary. Middleware can add correlation context and map known failures, but it should not turn all exceptions into success responses. Preserve safe diagnostics and log the original failure once with a correlation ID.

Do not wrap a long remote call in a middleware-created database transaction. A transaction belongs to the application use case and should cover the local invariant. Queue slow work rather than holding an HTTP worker open. Middleware can set request timeouts or observe duration, but it cannot make an unbounded downstream call safe by itself.

## Testing and Operations

Test each middleware in isolation for accepted and rejected requests, then test representative routes with the actual stack and order. Assert status, headers, authentication context, tenant scope, CSRF behavior, cache directives, and correlation IDs. A unit test of handle does not prove registration or ordering.

Monitor request duration by middleware or route group, rejection rates, rate-limit store latency, authentication failures, and correlation coverage. Avoid logging full headers and bodies. If middleware depends on a store or identity provider, define fail-open or fail-closed behavior; security and tenant middleware generally fail closed.

## Common Mistakes

- Treating middleware as a place for all business rules.
- Registering expensive middleware globally.
- Assuming authentication proves object-level authorization.
- Ordering tenant, auth, validation, and rate limiting accidentally.
- Mutating input without documenting the transformed value.
- Adding public cache headers to private responses.
- Wrapping remote work in a long database transaction.
- Testing a middleware class without testing its route registration.

## Senior Engineer Thinking

Middleware is a pipeline of HTTP boundary policy. Keep scope and ordering intentional, short-circuit unsafe or expensive requests early, keep domain decisions in the domain boundary, and test the composed stack. Treat headers, identity, tenant context, errors, and failure policy as client-visible contracts.

## Exercises

1. Draw the middleware order for an authenticated, tenant-scoped JSON route and justify every step.
2. Add a correlation-ID middleware that accepts only safe incoming IDs and test generated IDs.
3. Design separate middleware groups for browser and API requests, including CSRF, content type, and cache policy.
4. Identify a middleware that should move into an application service because its decision requires domain state.

## Review Questions

1. What can middleware do before and after the next callable?
2. Why does ordering affect security and performance?
3. Why is authentication not object-level authorization?
4. Which concerns belong in middleware rather than domain services?
5. How should middleware failures be observed without leaking credentials?

## Summary

Laravel middleware wraps HTTP processing with reusable authentication, rate-limit, tenant, validation, tracing, and response policy. Use the narrowest scope, verify ordering, keep domain invariants outside the pipeline, handle private caching and failure deliberately, and test both middleware behavior and actual route composition. Registration names vary by Laravel version, but the boundary and reasoning remain stable.

## References

- [Laravel documentation: Middleware](https://laravel.com/docs/middleware)
- [Laravel documentation: Request lifecycle](https://laravel.com/docs/lifecycle)
- [Laravel documentation: CSRF protection](https://laravel.com/docs/csrf)
- [Symfony HttpFoundation Response](https://symfony.com/doc/current/components/http_foundation.html)
- [OWASP HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)

