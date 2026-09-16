---
book: The Complete Modern PHP Engineering Book
volume: 16
volume_title: DISTRIBUTED SYSTEMS
chapter: 253
title: Service Boundaries
slug: service-boundaries
status: complete
summary: ../../_ai/chapter-summaries/253-service-boundaries-summary.md
---

# Chapter 253 — Service Boundaries

## Why This Matters

A service boundary separates ownership, deployment, runtime failure, and usually data. It is useful when a team or domain needs independent change, scaling, security, or availability. It is expensive because every boundary adds a network contract, latency, observability, deployment compatibility, and partial-failure path.

A service is not justified by a class count or an organization chart. A modular monolith can provide clear ownership and dependency direction without turning every method call into a remote operation. Split a service when the benefits outweigh the distributed-systems cost.

## Find the Boundary

A useful boundary gives one area a coherent reason to change and a clear authority for its invariants:

```text
catalog owns product description and publication
orders owns order lifecycle and totals
payments owns charge state and provider identity
```

Each owner exposes commands, queries, or events rather than its tables and internal classes. A boundary should make illegal dependencies difficult, not merely document them in a diagram.

Signals for a possible split include:

* independent data ownership and invariants;
* different scaling or latency profiles;
* different security or compliance boundary;
* independent deployment and failure tolerance;
* a team able to operate the service continuously;
* a stable contract that does not require constant synchronous chatter.

Signals against a split include shared transactions, frequent cross-domain joins, a single team and release cadence, no independent scaling need, and a workflow that requires many synchronous calls to complete one request.

## Synchronous Chatter

This shape creates a distributed monolith:

```text
checkout → orders → inventory → pricing → promotions → customer
```

Every dependency adds latency and a failure mode. If the services must all be available for every request, independent deployment has not produced independent availability. Prefer local composition, coarse-grained APIs, cached or asynchronous data, and an explicit pending workflow where the product permits it.

## Data Ownership

One service should be the authority for a fact. Other services may hold derived projections, but they must treat them as views with freshness and reconciliation rules.

```text
payments database: charge state, provider ID
orders database:   order state, payment reference
analytics store:   derived events
```

Do not share a database table as an integration API. Direct reads couple schemas, migrations, indexes, and retention. If a split is temporary, record the migration plan and the point at which direct access will end.

## Contracts

A service contract includes:

* operation and event schema;
* authentication, authorization, and tenant scope;
* timeout, retry, idempotency, and rate-limit behavior;
* consistency and freshness guarantees;
* error and pending states;
* version and deprecation policy;
* ownership and support expectations.

Contract tests verify the consumer's assumptions against a provider fixture or test service. They do not prove provider capacity, network behavior, or cross-service business correctness; those need integration and failure tests.

## A Domain Port

Keep application code dependent on a semantic port:

~~~php
<?php

declare(strict_types=1);

final readonly class PaymentReference
{
    public function __construct(public string $operationId)
    {
        if ($operationId === '') {
            throw new InvalidArgumentException('Missing payment operation');
        }
    }
}

interface PaymentAuthorizer
{
    public function authorize(
        int $customerId,
        int $amountCents,
        string $idempotencyKey,
        int $deadlineNs,
    ): PaymentReference;
}

function authorizeOrder(
    PaymentAuthorizer $payments,
    int $customerId,
    int $amountCents,
): PaymentReference {
    return $payments->authorize(
        $customerId,
        $amountCents,
        idempotencyKey: 'order-' . bin2hex(random_bytes(16)),
        deadlineNs: hrtime(true) + 500_000_000,
    );
}
~~~

The example shows the boundary's semantic inputs and operation identity. In production, the idempotency key should be derived from the durable order operation, not generated afresh every retry. The adapter can be local in a modular monolith or remote in a service deployment.

## Modular Monolith First

A modular monolith can enforce boundaries with namespaces, module APIs, dependency rules, separate repositories, and architecture tests. Calls remain local, transactions can remain local where appropriate, and deployment is simpler. It still requires ownership discipline; a monolith with unrestricted table access is not modular.

Extract when the module has a stable contract, an owner, an independently useful deployment or scaling reason, and a plan for data migration and operational support. Extraction should preserve the semantic boundary, not merely move classes and add HTTP.

## Workflows Across Services

Cross-service workflows need an explicit coordinator or event-driven state machine:

```text
order created → payment pending → payment authorized → order confirmed
       ↘ timeout/rejection → order cancelled or manual review
```

Persist state and operation identities. Use outbox events, idempotent consumers, compensation, reconciliation, and bounded retries. Do not hold a database transaction while waiting for a remote service.

## Deployment and Versioning

Old and new versions overlap during rolling deployments. Add fields before requiring them, accept old messages for their maximum lifetime, and keep provider and consumer changes compatible. Version semantic changes rather than exposing database migrations as an API.

Observe contract usage so removal has evidence. A service with unknown consumers cannot safely delete a field or route.

## Operations

The service owner needs dashboards, alerts, runbooks, capacity limits, dependency maps, and an on-call path. A boundary without operational ownership is a new outage domain with no recovery plan.

Measure request and event latency, error and retry rate, queue age, dependency saturation, consistency lag, deployment version, and tenant impact. Propagate trace context while keeping labels and payloads bounded.

## Testing

Test module rules in a monolith, contract schemas, authentication and tenant scope, timeout and retry behavior, duplicate effects, out-of-order events, mixed-version deployment, dependency outage, data migration, replay, and reconciliation. Test the business workflow across real boundaries in a smaller integration suite.

## Security

A service boundary is not automatically a trust boundary. Authenticate calls, authorize each operation, validate tenant scope, protect secrets, and avoid trusting fields merely because they arrived from an internal network. Restrict administrative replay, migration, and direct-data access.

## Common Mistakes

* Splitting by table or noun without an invariant owner.
* Sharing a database and calling the services independent.
* Creating many synchronous calls for one user request.
* Generating a new operation identity on every retry.
* Extracting a module before its contract and ownership are stable.
* Ignoring deployment overlap and old queued messages.
* Creating a service without observability or on-call ownership.
* Treating an internal network as authorization.

## Senior Engineer Thinking

Ask what independent capability the boundary buys, who owns the fact, what happens when the dependency is slow or unavailable, and whether the team can operate it. Service decomposition is a commitment to contracts and failure handling; a modular monolith is often the best place to prove the boundary before accepting distributed cost.

## Exercises

1. Identify ownership boundaries for catalog, orders, payments, and analytics in a modular monolith.
2. Draw a checkout workflow with one synchronous payment call and one asynchronous fulfillment event.
3. Define the contract and migration plan for extracting payments from the monolith.
4. List the operational responsibilities required before launching the extracted service.

## Review Questions

* What makes a service boundary valuable?
* Why is a shared database usually shared ownership rather than integration?
* How does synchronous chatter undermine independent availability?
* What should happen during mixed-version deployment?
* Why can a modular monolith be a better first step?
* Which operational duties accompany a new service?

## Summary

Service boundaries create independent ownership and deployment at the cost of network latency, partial failure, consistency windows, and operational work. Choose boundaries around coherent invariants, keep data ownership explicit, define durable contracts and workflows, prefer modular boundaries before remote extraction, and require security, observability, migration, and on-call readiness before splitting a service.

## References

- [Martin Fowler: MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- [Sam Newman: Monoliths and Microservices](https://samnewman.io/books/monolith-to-microservices/)
- [Chapter 194 — Modular Monolith](../../volumes/13-architecture/194-modular-monolith.md)
- [Chapter 208 — When Not to Use Microservices](../../volumes/13-architecture/208-when-not-to-use-microservices.md)
