---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 207
title: Microservices
slug: microservices
status: complete
summary: ../../_ai/chapter-summaries/207-microservices-summary.md
---

# Chapter 207 — Microservices

## Why This Matters

A microservice is an independently deployable process that owns a bounded capability and communicates over an explicit network contract. Its value comes from independent change, deployment, scaling, and failure isolation. It also introduces latency, partial failure, operational overhead, data consistency problems, and a larger security surface.

Splitting a PHP monolith into services does not automatically improve architecture. A service boundary should follow ownership and change, not a table or a team diagram. The system must be able to explain who owns each invariant, how a request recovers after a timeout, and how operators observe a business operation across processes.

## Service Boundaries and Ownership

A service should own a capability, its business rules, and the data required to enforce its invariants. Other services use an API or event contract rather than reading its tables. Ownership includes schema migrations, authorization policy, retention, alerts, and on-call responsibility.

A service boundary is stronger when it has:

* a coherent domain and change reason;
* an API or event schema with an owner;
* independent deployment or scaling pressure;
* data that does not require every write to be in another service's transaction;
* a failure and recovery policy.

Do not split a single CRUD workflow into services merely because it has several tables. A modular monolith can preserve a transaction and provide boundaries before network calls become necessary.

## Synchronous APIs and Events

Use a synchronous API when the caller needs an immediate answer and the dependency's latency and failure can fit the request budget. Use an event or command when work can be delayed, retried, or processed independently. Document method, authentication, timeout, status and error shape, idempotency, rate limits, schema version, and compatibility.

A PHP client should make network behavior explicit:

~~~php
<?php

declare(strict_types=1);

interface InventoryClient
{
    public function reserve(
        int $orderId,
        int $customerId,
        string $idempotencyKey,
    ): ReservationResult;
}

final class OrderWorkflow
{
    public function __construct(private InventoryClient $inventory)
    {
    }

    public function place(
        int $orderId,
        int $customerId,
        string $requestKey,
    ): ReservationResult {
        if ($requestKey === '') {
            throw new InvalidArgumentException('Request key is required');
        }

        return $this->inventory->reserve(
            $orderId,
            $customerId,
            $requestKey,
        );
    }
}
~~~

The client interface describes a business capability, while an HTTP adapter handles URL construction, TLS, serialization, timeouts, retries, and problem responses. A timeout after the inventory service commits is ambiguous; propagate the idempotency key and reconcile by operation ID before retrying.

## Data and Consistency

Each service should be the authority for its own writes. Duplicated read models are acceptable when their staleness and rebuild process are explicit. A service must not update another service's database to achieve apparent consistency; that creates hidden coupling and makes independent deployment unsafe.

Distributed transactions need a workflow or saga: reserve inventory, authorize payment, create the order, and compensate or retry when a step fails. A saga is not magic rollback. Compensation may fail, may be visible to users, and may require manual resolution. Persist state transitions and correlation IDs so the workflow can resume after a process crash.

Use an outbox in the service that commits state and publishes an event. Consumers need durable event IDs, idempotent handlers, dead-letter handling, and replay policy. An event's schema is a public contract; version it additively where possible and retain consumers during migration.

## Reliability and Failure Isolation

Every network call needs a finite connect and total timeout. Retries should be limited, use backoff and jitter, and apply only to classified transient failures. Retry only when the operation is safe or carries an idempotency key. Circuit breakers, bulkheads, queues, and rate limits can prevent one dependency from exhausting all PHP-FPM workers.

Set a request budget across the call graph. If the edge allows two seconds, three serial services cannot each wait two seconds. Propagate a deadline or remaining timeout where possible. Do not hold a database transaction open across a remote call unless the system has a bounded and justified policy.

Graceful degradation is a product decision. A recommendation service may be optional; payment authorization is not. Return a clear pending or unavailable state instead of silently inventing success.

## Deployment and Operations

Services need independent build, configuration, migration, rollback, and ownership workflows. A deployment must be backward compatible with in-flight requests and queued messages. Use expand-and-contract schema and API changes, health and readiness checks, bounded startup, and safe rollback.

Observe business operations across services with a correlation ID and trace context. Measure request rate, latency percentiles, status and error classes, dependency time, queue age, retries, circuit state, and saga state. Logs must redact tokens, cookies, personal data, and payment details. Alert on customer impact and recovery signals, not only process health.

## Security

A private network is not an authorization policy. Authenticate service calls with credentials appropriate to the deployment, authorize the operation and tenant, validate message origin and schema, and rotate secrets. Use least-privilege service accounts and separate read/write permissions where possible.

Treat events and queue payloads as untrusted input. Re-check current authorization for commands, avoid forwarding user claims as trusted service identity, and prevent confused-deputy behavior. Apply network, body-size, rate, and resource limits at every exposed service.

## Migration from a Monolith

Extract a capability only after its invariants and ownership are understood. Characterize existing behavior, define an API or event contract, make the monolith call the new boundary behind a feature flag, and compare outcomes. Use a strangler sequence with a rollback path and metrics.

A service extraction may require an anti-corruption adapter to translate the monolith's data model. Dual writes can drift; prefer an authoritative owner and an outbox, or reconcile with explicit checks. Do not leave both systems silently authoritative.

## Failure and Threat Analysis

* **Distributed monolith:** services require coordinated deploys and synchronous calls. Reduce shared releases and define asynchronous boundaries where appropriate.
* **Shared database:** schema changes and permissions couple services. Assign ownership or remain a modular monolith.
* **Retry storm:** timeouts trigger unbounded retries. Set budgets, backoff, jitter, and idempotency.
* **Split-brain data:** duplicated state diverges. Name the source of truth and rebuild projections.
* **Saga failure:** compensation does not complete. Persist state, retry safely, and provide manual resolution.
* **Trace gap:** operators cannot connect a request to a queue job. Propagate correlation and trace context.
* **Service impersonation:** one service trusts user-supplied claims. Authenticate the caller and authorize the action separately.
* **Secret sprawl:** credentials appear in images, logs, or configuration. Use scoped secret distribution and rotation.

## Testing Microservices

Unit-test policies and workflow decisions. Integration-test each adapter against its database, queue, and provider boundary. Contract-test consumer expectations against provider schemas and error semantics. Feature or end-to-end tests should cover a small number of critical cross-service paths.

Test timeouts, duplicate requests, out-of-order and duplicate events, incompatible versions, partial success, dead letters, retries, and recovery after process termination. Run failure injection in a safe environment. A green local suite cannot prove network behavior, deployment compatibility, or operator recovery.

## Exercises

1. Define service ownership for Orders, Inventory, and Billing. List each invariant, data store, API, event, and on-call owner.
2. Design an inventory reservation API with timeout, idempotency, authorization, and status semantics.
3. Model an order saga with reservation, payment, confirmation, compensation, and manual resolution states.
4. Write a contract-test matrix for an event consumer across schema versions and duplicate delivery.
5. Create a migration plan that extracts one capability from a PHP monolith with feature flags and rollback.

## Review Questions

1. What makes a service boundary useful?
2. Why should a service own the data enforcing its invariants?
3. When should a call be synchronous versus event-driven?
4. Why is a saga not the same as a database rollback?
5. How do timeouts and retries interact with PHP-FPM capacity?
6. What must be observed across a multi-service business operation?
7. Why is a private network insufficient for authorization?

## Summary

Microservices provide independent deployment, scaling, and failure boundaries at the cost of distributed-systems complexity. Define service ownership around capabilities and invariants, communicate through versioned contracts, keep data authoritative per service, use outboxes and idempotent workflows, bound every network call, propagate identity and observability context, and test partial failure and recovery before extracting a boundary.

## References

- [Martin Fowler: Microservices](https://martinfowler.com/articles/microservices.html)
- [Martin Fowler: MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- [Martin Fowler: Saga](https://martinfowler.com/articles/saga.html)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)

