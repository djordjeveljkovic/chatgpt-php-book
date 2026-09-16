---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 217
title: Symfony Overview
slug: symfony-overview
status: complete
summary: ../../_ai/chapter-summaries/217-symfony-overview-summary.md
---

# Chapter 217 — Symfony Overview

Symfony is both a full-stack PHP framework and a collection of reusable components. The framework supplies conventions and integration around HTTP, routing, configuration, dependency injection, console commands, security, templates, persistence, and messaging. The components can also be used independently.

The useful mental model is a set of explicit boundaries assembled into an application. Symfony does not remove HTTP semantics, database transactions, authorization, or distributed-system failure; it provides tested mechanisms for composing them.

## Why this matters

A framework request still moves through the same engineering chain:

```text
server request → Request object → routing → controller/service
              → domain/application work → Response → server output
```

The framework creates objects, dispatches events, resolves arguments, and formats failures around that chain. Your application still decides what a user may do, which data is authoritative, when a transaction commits, and what happens if a provider times out.

Framework knowledge is strongest when connected to the underlying PHP and HTTP model. A controller is an adapter; a service is a dependency-injected object; a response is an HTTP message; a Messenger handler is a worker entry point with retry semantics.

## Full-stack framework and components

A Symfony application commonly uses a kernel, service container, router, configuration system, bundles, and a set of components. A smaller application may use only `HttpFoundation`, `Routing`, and `DependencyInjection`. A library should avoid requiring the full framework when one component expresses its need.

Components have their own contracts and release policies. Keep application code behind the component interfaces you actually use, and treat framework configuration as code that deserves review, tests, and deployment compatibility.

## A request through Symfony

At a high level, the front controller creates a request and asks the kernel to handle it:

```php
<?php

declare(strict_types=1);

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\HttpKernelInterface;
use Symfony\Component\HttpKernel\TerminableInterface;

require dirname(__DIR__) . '/vendor/autoload.php';

$request = Request::createFromGlobals();
/** @var HttpKernelInterface&TerminableInterface $kernel */
$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response);
```

The exact bootstrap and kernel class vary by application and Symfony version. The important boundaries are stable: request parsing, routing, controller invocation, response creation, termination work, and error handling. In a long-running worker, request-scoped state must be reset between requests.

## Configuration and environments

Keep environment-specific values at the deployment boundary. Symfony configuration can define services, routes, framework behavior, and parameters; secrets should use the deployment's secret mechanism and least-privilege access. Do not put credentials in committed YAML or expose the entire container configuration in production diagnostics.

Separate configuration from runtime state. A cache warmup can compile a service graph, but a cache hit does not prove a provider is reachable or a database migration is applied. Deploy configuration and schema changes with an explicit compatibility order.

## Controllers and application boundaries

A controller should translate HTTP input and output. It can call an application service, return a response, and map a domain exception to an HTTP status through a deliberate policy. It should not become a database query bag or a place where every business invariant is reimplemented.

Symfony's dependency injection container can construct a controller and its services. Use constructor injection, narrow interfaces, and explicit ports. Keep authorization tied to the authenticated actor and loaded resource; route matching is not authorization.

## Events, console, and messaging

Symfony's EventDispatcher lets listeners react to framework or application events. Use it for cross-cutting policies and independent reactions, while keeping critical domain invariants in domain code. Symfony Console provides CLI boundaries; commands need input validation, exit semantics, and safe output. Messenger can route messages synchronously or through transports; queues add retries, duplicates, dead letters, and observability obligations.

A framework component does not define your delivery guarantee. Configure and test the actual transport and failure policy.

## Testing and operations

Use unit tests for domain and application policies, integration tests for the container, router, database, and component adapters, and a small number of HTTP tests for critical workflows. Test configuration compilation and cache warmup in CI. Avoid testing private framework internals; test your contracts at the boundary.

Observe request IDs, route and controller names, status classes, duration, queue message IDs, dependency latency, and exception categories. Redact authorization headers, cookies, tokens, and personal data. Keep production error responses safe while retaining protected diagnostic context.

## Exercises

1. Trace one Symfony request from the front controller to the response and identify where authentication, authorization, validation, and transaction policy belong.
2. Choose a component-only design for a small webhook receiver and list what the full framework would otherwise provide.
3. Add a console command that invokes an application service and define its validation, retry, and exit-code policy.

## Review questions

- What does Symfony provide around the PHP request/response model?
- When should an application use a component instead of the full framework?
- Why is a matched route not proof of authorization?
- Which responsibilities remain application or domain decisions?
- What additional failure modes appear when Messenger uses a queue transport?

## Summary

Symfony combines a full-stack framework with reusable components around explicit PHP and HTTP boundaries. Use its container, kernel, router, events, console, and messaging mechanisms deliberately; keep business rules and ownership in application code; and test configuration, adapters, and critical request paths at the right boundaries.

## References

- [Symfony documentation](https://symfony.com/doc/current/index.html)
- [Symfony Components](https://symfony.com/components)
- [Symfony HttpKernel](https://symfony.com/doc/current/components/http_kernel.html)
- [Symfony Best Practices](https://symfony.com/doc/current/best_practices.html)
