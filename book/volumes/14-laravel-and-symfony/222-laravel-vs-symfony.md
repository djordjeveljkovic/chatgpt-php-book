---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 222
title: Laravel vs Symfony
slug: laravel-vs-symfony
status: complete
summary: ../../_ai/chapter-summaries/222-laravel-vs-symfony-summary.md
---

# Chapter 222 — Laravel vs Symfony

## Why This Matters

Laravel and Symfony are mature PHP ecosystems that can both serve production HTTP applications, APIs, workers, and command-line tools. The choice should follow the team's requirements, existing code, hosting model, release policy, and operational skills. A framework comparison based only on syntax or popularity hides the cost of migration and the quality of the application's boundaries.

Laravel emphasizes an integrated application experience with conventions, expressive helpers, Eloquent, queues, notifications, and first-party integrations. Symfony emphasizes reusable components, explicit service configuration, event and kernel contracts, and a broad component ecosystem. Both can be used selectively, and both still require deliberate authorization, transactions, testing, security, and operations.

## Compare Boundaries, Not Marketing

Evaluate the same concern in each framework:

| Concern | Laravel concepts | Symfony concepts | Decision question |
| --- | --- | --- | --- |
| HTTP entry point | routing, middleware, controllers | HttpKernel, routing, event subscribers, controllers | Which lifecycle and extension points fit the team? |
| Object graph | service container and bindings | DependencyInjection container and aliases | How explicit must wiring and compilation be? |
| Persistence | Eloquent and query builder | Doctrine integration or another adapter | Which model and migration boundaries are needed? |
| Async work | queues, jobs, workers | Messenger buses, transports, handlers | What delivery, retry, and observability policy is required? |
| Validation and security | requests, gates, policies, guards | Validator, Security, voters, firewalls | Where do identity and object authorization live? |
| Testing | feature helpers, factories, fakes, browser tools | WebTestCase, kernel tests, PHPUnit integrations | Which boundaries need a booted application? |
| Components | framework conventions and packages | independently installable components | Can a library avoid a full framework? |

Names and defaults change across major versions. Treat this as a map for questions, then read the supported release documentation and upgrade guides before implementing a production decision.

## A Stable Application Port

Keep domain and application code behind ports so a framework choice remains an entry-point and adapter concern:

~~~php
<?php

declare(strict_types=1);

interface PlaceOrder
{
    public function handle(PlaceOrderCommand $command): OrderReceipt;
}

final readonly class PlaceOrderCommand
{
    /** @param list<array{sku: string, quantity: int}> $lines */
    public function __construct(
        public int $customerId,
        public array $lines,
        public string $requestKey,
    ) {
    }
}

final readonly class OrderReceipt
{
    public function __construct(
        public string $orderId,
        public string $status,
    ) {
    }
}
~~~

A Laravel controller can validate a request, authenticate the actor, construct the command, invoke PlaceOrder, and map the receipt to JSON. A Symfony controller can perform the same boundary work using its request and validation mechanisms. The domain service should not receive an Illuminate request, a Symfony request, or a framework response.

This port does not erase differences. Transaction helpers, dependency bindings, queue dispatch, exception mapping, and ORM behavior remain framework adapters. Keep those differences at the edge and document their failure and lifecycle semantics.

## Laravel's Integrated Path

Laravel's conventions can reduce setup for a team building a conventional application. Routes, middleware, controllers, request validation, Eloquent models, migrations, jobs, notifications, configuration, and testing helpers fit a cohesive workflow. First-party packages can reduce integration work when their contracts match the product.

The convenience is valuable when the team accepts the framework's conventions and keeps model, query, and application boundaries clear. Eloquent calls still have query cost; a job can still run twice; a middleware order can still be security-critical. A concise helper is not a business guarantee.

Choose Laravel when its integrated defaults, team familiarity, package ecosystem, and delivery speed fit the constraints. Verify long-term support, package compatibility, queue and database drivers, and the team's ability to inspect framework behavior during incidents.

## Symfony's Component Path

Symfony can provide a full-stack framework or focused components such as HttpFoundation, HttpKernel, Routing, DependencyInjection, EventDispatcher, Console, Validator, Serializer, and Messenger. Explicit configuration and contracts can help a large application or library define boundaries and replace individual mechanisms.

The additional explicitness has a cost in configuration, concepts, and integration choices. Autowiring, bundles, event subscribers, Doctrine, and Messenger still need ownership and tests. A component-only design can reduce framework coupling, but the application must assemble the missing lifecycle and policies deliberately.

Choose Symfony when component reuse, explicit service graph, configurable lifecycle, long support horizons, or an existing Symfony ecosystem matter. Verify bundle compatibility, component versions, configuration compilation, and operational defaults.

## Persistence and Domain Boundaries

Laravel Eloquent and Symfony's common Doctrine integration offer different object and persistence models. Do not decide from whether one model class feels more convenient. Ask:

- Is the domain model an ORM model or mapped separately?
- Who owns transactions, locks, optimistic versions, and outbox writes?
- How are lazy loading, identity maps, and unit-of-work behavior controlled?
- Can tenant scope and authorization be enforced in every query?
- How are migrations, mixed-version deployments, and read models operated?
- Which database engine semantics must integration tests prove?

Either framework can use a repository or data-mapper boundary when the domain requires it. Either can accidentally leak persistence models into API responses. Keep public representations explicit and test query shape and cost.

## HTTP and Middleware Models

Laravel middleware and Symfony kernel events express similar extension needs with different lifecycles and configuration. In either framework, authentication must precede object authorization, tenant context must precede scoped queries, and rate limits or body-size checks should happen before expensive work.

Test the composed stack. A unit test of one middleware or event subscriber does not prove registration, order, sub-request behavior, exception mapping, or private response caching. Keep required database commits and external effects in application services and outbox boundaries rather than post-response hooks.

## Queues and Messaging

Laravel jobs and Symfony Messenger can both run synchronous or asynchronous work depending on configuration. The transport determines visibility, acknowledgment, retry, ordering, and dead-letter behavior. A job handler or message handler should carry small identifiers, re-check current authorization where needed, and make effects idempotent.

Do not migrate queue code by translating method names only. Preserve operation IDs, after-commit dispatch, payload privacy, retry classification, worker memory limits, and observability. Run an integration test against the chosen transport and use a framework fake only for dispatch intent.

## Testing and Team Workflow

Both ecosystems integrate with PHPUnit and can provide factories, HTTP clients, container bindings, database resets, fakes, and browser tools. Keep the testing questions independent of helper names:

- Does the route and middleware stack enforce the public contract?
- Does the real database enforce constraints and transaction behavior?
- Does a worker retry and deduplicate correctly?
- Does an external adapter satisfy its contract?
- Does a browser user complete a critical journey?

Framework-specific helpers are productive when their setup is understood. A test that globally disables middleware or uses an impossible factory state can give false confidence in either ecosystem.

## Migration and Coexistence

A framework migration is a product and operational project. Estimate routes, authentication, persistence mappings, jobs, scheduled commands, templates, configuration, assets, tests, deployment, and team training. Preserve behavior with API, contract, and end-to-end tests before moving a boundary.

A strangler migration can route one module or endpoint through the new framework while the old path remains supported. Shared sessions, database transactions, event schemas, and cache keys require an explicit compatibility plan. Avoid running two frameworks against mutable data without ownership and conflict rules.

A company can also use both ecosystems: a Symfony component inside a Laravel application, or a Laravel service alongside Symfony services. Keep shared domain contracts and event schemas independent of framework classes, and standardize logging, tracing, authentication, and deployment policy.

## Decision Record

Record the decision and revisit date:

1. State product, team, performance, compliance, and hosting constraints.
2. Compare the current and alternative framework on those constraints.
3. Prototype one representative route, persistence operation, queue job, and test.
4. Measure development time, query behavior, startup, memory, and failure diagnostics.
5. List migration and long-term maintenance costs.
6. Choose a framework and document rejected alternatives and assumptions.
7. Revisit only when constraints or evidence change.

A framework's benchmark or feature list is input, not an architecture decision. The right choice is the one the team can build, secure, debug, upgrade, and operate for the expected lifetime.

## Common Mistakes

- Choosing from syntax, hype, or a single benchmark.
- Treating Eloquent or Doctrine as the domain model without examining boundaries.
- Assuming middleware and kernel events have identical ordering or scope.
- Translating queue APIs while losing idempotency and retry policy.
- Comparing testing helper names instead of test questions.
- Migrating framework and database ownership simultaneously without a compatibility plan.
- Sharing framework-specific classes across domain and integration boundaries.
- Ignoring team training, support, upgrades, and incident response.

## Senior Engineer Thinking

Laravel and Symfony can both support reliable systems. Compare their boundaries, defaults, ecosystem, operational cost, and team fit against a concrete workload. Keep domain contracts independent, prototype representative paths, preserve delivery and data semantics during migration, and choose the framework the team can operate responsibly.

## Exercises

1. Build a comparison matrix for one application covering HTTP, persistence, queues, security, testing, support, and team skills.
2. Implement one application port with a Laravel adapter and a Symfony adapter. Identify behavior that cannot be hidden.
3. Plan a strangler migration for one module, including sessions, transactions, queues, cache keys, and rollback.
4. Prototype a route, database operation, queue job, and test in both ecosystems and record measured differences.

## Review Questions

1. Which constraints should drive a Laravel versus Symfony decision?
2. What does a framework-independent application port protect?
3. How do Eloquent and Doctrine-style persistence boundaries differ in design concerns?
4. Why must queue semantics survive a framework migration?
5. What does a feature or kernel unit test fail to prove?
6. When can using both ecosystems be reasonable?
7. Which evidence belongs in a framework decision record?

## Summary

Laravel and Symfony are capable PHP ecosystems with different defaults, extension models, and component boundaries. Compare them against workload, team, support, persistence, HTTP, queue, security, testing, and operational constraints. Keep application contracts framework-independent, prototype representative paths, preserve transaction and delivery semantics during migration, and choose the system the team can operate and evolve.

## References

- [Laravel documentation](https://laravel.com/docs)
- [Symfony documentation](https://symfony.com/doc/current/index.html)
- [Laravel release notes](https://laravel.com/docs/releases)
- [Symfony releases and maintenance](https://symfony.com/releases)
- [PHP-FIG PSR standards](https://www.php-fig.org/psr/)
- [Martin Fowler: Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)

