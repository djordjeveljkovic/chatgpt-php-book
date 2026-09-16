---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 220
title: Symfony HttpKernel
slug: symfony-httpkernel
status: complete
summary: ../../_ai/chapter-summaries/220-symfony-httpkernel-summary.md
---

# Chapter 220 — Symfony HttpKernel

HttpKernel is the Symfony component that turns a `Request` into a `Response` through a kernel contract and a sequence of events. It coordinates routing, controller resolution, argument handling, response creation, and exception processing while allowing application and framework listeners to participate.

Understanding this lifecycle helps diagnose middleware-like behavior, authentication order, response headers, exception mapping, and work that runs after the response. It also prevents listeners from becoming hidden business workflows.

## Why this matters

A controller is only one stage in an HTTP request. A listener may reject a request before controller execution, alter a response after the controller returns, or convert an exception into a safe error response. The order and scope of those stages affect security and correctness.

The conceptual flow is:

```text
Request
  → kernel.request listeners
  → routing/controller resolution
  → controller arguments and invocation
  → kernel.view when no Response is returned
  → kernel.response listeners
  → Response
  → kernel.terminate work
```

Exceptions can interrupt the flow and enter `kernel.exception` listeners. The exact internal implementation is version-specific; the event contracts and documented lifecycle are the useful application boundary.

## The kernel contract

At the component level, the kernel handles a request:

```php
<?php

declare(strict_types=1);

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\HttpKernelInterface;

function dispatch(HttpKernelInterface $kernel, Request $request): Response
{
    return $kernel->handle($request, HttpKernelInterface::MAIN_REQUEST);
}
```

The kernel may be embedded in a front controller, a sub-request, a test client, or a worker. Main-request and sub-request behavior must be distinguished when a listener should run only once. In current Symfony terminology, the request type constants and event classes should be used according to the supported version.

## Request and controller events

A `kernel.request` listener can add request attributes, establish locale, or reject an unauthenticated request. It should not trust a route parameter as an authorization decision. Routing and controller resolution then select the operation; the controller or application service authorizes the loaded resource.

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class LocaleSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::REQUEST => ['onRequest', 20]];
    }

    public function onRequest(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;
        }

        $locale = $event->getRequest()->headers->get('Accept-Language');
        if (is_string($locale) && $locale !== '') {
            $event->getRequest()->setLocale(strtok($locale, ',') ?: 'en');
        }
    }
}
```

Parse and allow-list locale values in production; this example demonstrates the lifecycle, not a complete locale policy. Listener priorities are part of behavior and should be documented when security or request mutation depends on them.

## Response and exception events

A response listener can add headers, correlation IDs, or cache policy. An exception listener can map known domain exceptions to stable HTTP responses. Do not return stack traces or provider payloads to clients, and do not let a generic listener turn authorization failures into successful responses.

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class ProblemSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::EXCEPTION => 'onException'];
    }

    public function onException(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();
        if (!$exception instanceof DomainException) {
            return;
        }

        $event->setResponse(new JsonResponse(
            ['error' => 'domain_conflict'],
            409,
        ));
    }
}
```

A production exception policy should distinguish validation, forbidden, not found, conflict, and unavailable states, and should log a correlation ID with a redacted diagnostic.

## Termination and long-running processes

After the response is sent, `kernel.terminate` listeners may perform work. Do not assume the client waits for that work or that it is durable; use a queue when delivery matters. In PHP-FPM, termination behavior depends on server configuration and process lifecycle. In long-running workers, reset request-scoped services and avoid retaining request data.

A slow terminate listener can consume workers and create hidden latency or shutdown behavior. Measure it and keep it bounded. Never place a required database commit only in termination work.

## Sub-requests and recursion

Fragments, error handling, and embedded controllers may create sub-requests. A listener that mutates every request can run more than once. Check `isMainRequest()` when policy is main-request-only, and make shared listeners idempotent. Avoid recursively forwarding requests as a substitute for a clear application service call.

## Testing and operations

Test listeners with event objects for focused behavior, then test a kernel boot for registration, priority, and configuration. HTTP integration tests should verify authentication order, response headers, exception status, content negotiation, and safe error bodies. Test both main and sub-requests where a listener claims to distinguish them.

Observe route, controller, event, exception, request type, response status, and duration. Propagate correlation IDs to queue messages and external calls. Do not log cookies, authorization headers, or raw request bodies by default.

## Exercises

1. Add a response subscriber that sets a security header and test it on success and error responses.
2. Define listener priorities for authentication, locale, and routing-dependent policy. Explain why each order is required.
3. Move a required post-response email from termination work to an outbox-backed queue and test failure recovery.

## Review questions

- Where can a `kernel.request` listener run before controller authorization?
- Why should main-request and sub-request behavior be explicit?
- What belongs in an exception listener versus an application service?
- Why is termination work unsuitable for required durability?
- Which event ordering and registration behaviors need kernel integration tests?

## Summary

HttpKernel turns requests into responses through a documented lifecycle of request, controller, view, response, exception, and termination events. Use listeners for focused boundary policies, keep domain workflows explicit, distinguish main and sub-requests, map failures safely, and test registration and event ordering through the real kernel.

## References

- [Symfony HttpKernel Component](https://symfony.com/doc/current/components/http_kernel.html)
- [Symfony Kernel Events](https://symfony.com/doc/current/reference/events.html)
- [Symfony EventDispatcher](https://symfony.com/doc/current/components/event_dispatcher.html)
- [Symfony HttpFoundation](https://symfony.com/doc/current/components/http_foundation.html)
