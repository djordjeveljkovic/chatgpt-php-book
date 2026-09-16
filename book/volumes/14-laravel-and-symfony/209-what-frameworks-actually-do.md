---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 209
title: What Frameworks Actually Do
slug: what-frameworks-actually-do
status: complete
summary: ../../_ai/chapter-summaries/209-what-frameworks-actually-do-summary.md
---

# Chapter 209 — What Frameworks Actually Do

## Why This Matters

A PHP framework is a collection of reusable runtime decisions around an application. It receives requests, boots configuration, resolves dependencies, dispatches routes, runs middleware, maps errors, and sends responses. It also supplies conventions and integrations for persistence, queues, validation, sessions, templates, and testing.

A framework does not remove the request-response model or the need to design domain rules. It automates plumbing and provides extension points. Understanding that plumbing lets you debug a failed middleware chain, avoid hidden database work, and keep framework details at the edge of the application.

## The Common Runtime Shape

A typical web framework performs work like this:

~~~text
web server
  -> PHP entry point and Composer autoloader
  -> application bootstrap and configuration
  -> dependency/container setup
  -> middleware pipeline
  -> route matching and parameter binding
  -> controller or handler
  -> response middleware and error mapping
  -> HTTP response
~~~

The names and classes differ between Laravel, Symfony, and versions, but the responsibilities are stable. CLI commands, queue workers, scheduled tasks, and HTTP requests may share bootstrap and services while having different lifecycles.

## What the Framework Owns

Framework code commonly owns:

* request and response objects;
* routing and middleware ordering;
* service discovery and dependency resolution;
* configuration loading and environment integration;
* error and exception handling;
* session, cookie, CSRF, and authentication adapters;
* database, queue, cache, filesystem, and mail integrations;
* command-line and worker bootstrapping;
* test helpers and application diagnostics.

Your application still owns requirements, authorization decisions, domain invariants, transaction choices, data classification, timeout policy, and recovery. A framework validator can parse a field; it cannot decide whether a refund is legal in the current state unless you put that rule in the correct application or domain boundary.

## Framework Versus Application Code

Keep framework-specific types at the outer edge when a rule does not need them. A controller can turn a request into a command:

~~~php
<?php

declare(strict_types=1);

final readonly class CreateInvoiceCommand
{
    public function __construct(
        public int $accountId,
        public int $amountCents,
    ) {
    }
}

final class InvoiceController
{
    public function __construct(private CreateInvoiceHandler $handler)
    {
    }

    public function store(Request $request): Response
    {
        $command = new CreateInvoiceCommand(
            accountId: (int) $request->input('account_id'),
            amountCents: (int) $request->input('amount_cents'),
        );

        $id = $this->handler->handle($command);

        return response()->json(['id' => $id], 201);
    }
}
~~~

The Request, Response, and response helper are framework-specific. The command and handler can remain usable from a queue or CLI entry point. Real code must validate types and authorize the account before creating the command; the example shows the boundary, not a complete endpoint.

## Conventions and Extension Points

Conventions reduce decisions: directory discovery, route files, provider registration, configuration names, and test bootstrapping. Extension points allow application code to add routes, bindings, middleware, events, commands, and policies.

Use an extension point when the framework guarantees its lifecycle. Do not patch vendor code or rely on undocumented call order. When a framework upgrade changes a bootstrap hook, the documented extension point and an integration test should reveal the impact.

Framework magic is usually a combination of an entry point, a container, reflection, attributes or configuration, registries, and middleware. Read the generated code and trace a request when behavior is unclear. A facade or helper may be convenient, but know whether it resolves a scoped service, opens a connection, caches a value, or performs I/O.

## Costs and Failure Modes

Frameworks add dependencies, startup work, conventions, upgrade constraints, and an abstraction layer between source and runtime. They can also hide:

* N+1 database queries behind property access;
* network calls inside a convenient client;
* transaction boundaries in request helpers;
* global state in static facades;
* middleware that changes authentication or input;
* serialization rules that expose fields unintentionally.

Measure query count, memory, boot time, queue behavior, and worker reuse. Configure finite timeouts and log safe correlation data. A framework default is a starting policy, not proof that it matches the application's threat model.

## Testing Framework Boundaries

Unit-test framework-independent domain and application code. Integration-test repositories, providers, container bindings, and serialization. Feature-test routes, middleware, authentication, validation, and response contracts. Test framework upgrades against representative flows and inspect deprecation notices before they become outages.

Do not replace every framework integration with mocks. A mocked router cannot prove route middleware order, and a mocked ORM cannot prove SQL constraints. Test each behavior at the boundary where the framework participates.

## Exercises

1. Trace one Laravel or Symfony request from the public entry point to the response. Record each bootstrap, middleware, and adapter boundary.
2. Extract a domain command from a controller and invoke it from both HTTP and a CLI entry point.
3. List three framework conveniences that could hide I/O. Add a test or metric that makes the cost visible.
4. Identify a framework default that is a security or availability policy and decide whether to override it.

## Review Questions

1. Which responsibilities do frameworks commonly automate?
2. Which decisions remain application or domain responsibilities?
3. Why should framework Request and Response types stay near the edge?
4. What can conventions simplify, and what can they hide?
5. Which test level should verify middleware and route behavior?
6. Why should a framework default be treated as a starting policy?

## Summary

Frameworks automate request handling, bootstrapping, dependency resolution, middleware, integrations, and testing conventions. They do not replace domain rules, authorization, transactions, failure recovery, or observability. Keep stable application behavior behind framework boundaries, trace hidden work, and test the real extension points.

## References

- [Laravel Documentation](https://laravel.com/docs)
- [Symfony Documentation](https://symfony.com/doc/current/index.html)
- [PHP Manual: Runtime Configuration](https://www.php.net/manual/en/configuration.php)
- [PHP-FIG PSR-15: HTTP Server Request Handlers](https://www.php-fig.org/psr/psr-15/)

