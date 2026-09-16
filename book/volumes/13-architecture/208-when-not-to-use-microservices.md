---
book: The Complete Modern PHP Engineering Book
volume: 13
volume_title: ARCHITECTURE
chapter: 208
title: When Not to Use Microservices
slug: when-not-to-use-microservices
status: complete
summary: ../../_ai/chapter-summaries/208-when-not-to-use-microservices-summary.md
---

# Chapter 208 — When Not to Use Microservices

## Why This Matters

Microservices can give teams independent deployment, process isolation, and ownership of separate data and capabilities. They also create a distributed system: network calls, deployment coordination, observability, schema evolution, retries, authentication, incident response, and more infrastructure. The boundary is valuable only when its benefits outweigh those costs.

A modular monolith is often the better starting point. It can keep domain boundaries, explicit ports, separate modules, and independent tests while using one process and one deployment. A team can extract a service later when evidence shows that separate scaling, ownership, fault isolation, or release cadence is worth the operational cost.

## The Decision Is About Constraints

Do not choose an architecture from company fashion or a diagram. Measure the constraints:

- How many teams can own and operate services on call?
- Which components need different scaling or availability?
- Which data must be consistent in one transaction?
- How often do parts need independent deployment?
- What are the network, security, compliance, and latency boundaries?
- Can the organization support tracing, schema compatibility, incident response, and local development?
- What is the cost of a service failure and a partial rollout?

If the answers do not show a meaningful boundary, a process split adds failure modes without solving a problem.

## When a Modular Monolith Fits

A modular monolith keeps one deployable process while enforcing internal boundaries. Each module owns its application services, domain objects, persistence access, and public interface. Other modules call that interface rather than importing tables, mutable entities, or internal classes.

~~~php
<?php

declare(strict_types=1);

interface BillingPort
{
    public function authorize(string $accountId, int $amountMinor, string $requestKey): Authorization;
}

final class CheckoutService
{
    public function __construct(private BillingPort $billing)
    {
    }

    public function place(string $accountId, int $amountMinor, string $requestKey): OrderReceipt
    {
        $authorization = $this->billing->authorize(
            $accountId,
            $amountMinor,
            $requestKey,
        );

        return OrderReceipt::fromAuthorization($authorization);
    }
}
~~~

The initial BillingPort can be implemented by an in-process module. It still defines an ownership and idempotency contract. If Billing later becomes a remote service, an HTTP adapter can implement the same port while the application keeps its policy boundary.

A modular monolith is not a pile of shared code. Enforce boundaries with namespaces, dependency rules, separate module tests, review policy, and database access conventions. One process does not remove the need for tenant isolation, authorization, transaction design, or observability.

## Cost of a Service Split

A remote call changes ordinary control flow:

- latency and timeouts replace a function call;
- authentication and authorization cross a network boundary;
- responses can be delayed, duplicated, reordered, or lost;
- retries can duplicate a business effect;
- deployments can run incompatible versions;
- local development needs service discovery and representative dependencies;
- a database transaction no longer spans both owners;
- logs and traces must cross processes;
- an outage can affect only some workflows.

The team also needs build pipelines, runtime environments, secrets, certificates, dashboards, alerts, ownership, on-call coverage, and a safe rollback or reconciliation plan. Count these as product design costs, not infrastructure trivia.

## Strong Consistency and Shared Data

If two parts must update the same business invariant atomically, a process split requires a distributed transaction, a workflow with compensation, or a redesign of the invariant. Each has costs and failure states. A foreign key and one database transaction are often simpler and safer when the data has one owner and the workload fits one deployment.

Do not create a service per table and then share the database. Shared tables couple deployments while adding network failure. If a service owns data, other components should use an explicit API or event and accept the documented consistency. If immediate consistency matters, keep the operation within the owner or reconsider the boundary.

## Team and Operational Readiness

A service boundary should match ownership. If one small team must coordinate and operate twelve services, independent deployment may become a queue of operational work. A service that no one can debug or page on is not autonomous.

Before splitting, establish:

- service-level objectives and a clear on-call owner;
- centralized logs, metrics, traces, and correlation IDs;
- automated builds, migrations, security scans, and rollback;
- contract tests and compatibility policy;
- local and staging environments that reproduce failure modes;
- bounded timeout, retry, rate-limit, and resource policies;
- incident runbooks and reconciliation tools.

A platform can provide defaults, but the service team still owns its behavior and data.

## Extraction Signals

Extraction is more likely to pay off when evidence shows one capability:

- scales or has availability requirements that differ materially;
- has a separate team and release cadence;
- requires a technology or runtime incompatible with the main application;
- needs stronger isolation for security or compliance;
- creates deployment contention or a measurable fault-isolation benefit;
- has a stable contract and a data boundary that can be owned.

A high CPU endpoint alone may need an index, cache, queue, or worker before it needs a service. A slow query is not an architectural boundary. Measure the bottleneck and compare a local optimization, modular separation, and process extraction.

## A Safe Extraction Path

Use a staged migration:

1. Name the capability and its owner.
2. Define the module API, invariants, authorization, and failure contract.
3. Remove direct cross-module table access.
4. Add contract and integration tests.
5. Add metrics for calls, latency, failures, and usage.
6. Introduce an adapter so the caller can use in-process or remote implementations.
7. Move data ownership and migration responsibility deliberately.
8. Deploy the remote implementation behind a controlled route or percentage.
9. Reconcile differences and retain a rollback plan.
10. Remove the old path only after evidence and a support-window decision.

The adapter boundary is useful only if the two implementations obey the same meaningful contract. A remote implementation may need asynchronous outcomes, operation IDs, and a different consistency model; do not hide those facts behind a synchronous method that lies.

## Alternatives

Consider simpler responses before a microservice:

- split modules inside the monolith;
- move slow work to a queue worker;
- add a read model or cache;
- partition a database table or use a read replica;
- isolate a high-risk operation behind a narrow process;
- use a managed provider for a commodity capability;
- improve deployment and ownership without changing runtime topology.

A queue worker is still a distributed boundary if it uses another process, but it may fit the workload better than a request-facing service. A managed provider shifts operations and dependency risk rather than removing them; evaluate its contract, exit path, data retention, and failure behavior.

## Testing and Operations

Test module contracts inside the monolith before extraction. After extraction, add contract tests, timeout and retry tests, duplicate and ambiguous-outcome tests, schema compatibility tests, and deployment compatibility tests. End-to-end tests should cover a small number of workflows, not every branch.

Measure request latency, dependency saturation, queue age, error budget, contract failures, and reconciliation volume. Track the cost of local development and on-call incidents. If a service boundary repeatedly requires synchronized releases or direct database access, treat that as evidence the boundary is not autonomous.

## Common Mistakes

- Splitting a monolith because microservices are fashionable.
- Creating a service per table while sharing the database.
- Assuming a network adapter preserves local transaction semantics.
- Ignoring team ownership and on-call capacity.
- Using retries without idempotency and a timeout budget.
- Extracting a slow query before measuring simpler fixes.
- Hiding remote latency and partial failure behind a misleading synchronous interface.
- Calling a modular monolith an architectural failure.

## Senior Engineer Thinking

Microservices are one response to scale, ownership, and isolation constraints. Start with explicit modules and contracts, measure the pressure, and extract only when independent deployment or runtime isolation earns the distributed-system cost. A smaller topology with clear boundaries can be more reliable, faster to change, and easier to operate.

## Exercises

1. Compare a modular monolith and two services for a checkout and billing workflow. List consistency, deployment, and incident trade-offs.
2. Define extraction signals for one hot path and identify measurements needed before splitting it.
3. Design an in-process BillingPort and a remote adapter with idempotency, timeout, and ambiguous-outcome behavior.
4. Draw the minimum operational platform a team needs before owning five production services.

## Review Questions

1. What benefits must justify the cost of a service boundary?
2. Why is shared-database microservices coupling dangerous?
3. Which constraints make a modular monolith a strong choice?
4. Why can a queue worker be a better solution than a request-facing service?
5. What evidence should trigger extraction?
6. Which contracts cannot be hidden behind the same synchronous interface after extraction?

## Summary

Do not adopt microservices without a concrete need for independent scaling, ownership, deployment, technology, or fault isolation. A modular monolith can enforce domain boundaries while preserving local transactions and simpler operations. Measure bottlenecks, define contracts, establish operational readiness, and extract gradually only when the distributed-system costs are justified.

## References

- [Martin Fowler: MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- [Martin Fowler: Microservices](https://martinfowler.com/articles/microservices.html)
- [Sam Newman: Monolith to Microservices](https://samnewman.io/books/monolith-to-microservices/)
- [microservices.io: Decompose by Business Capability](https://microservices.io/patterns/decomposition/decompose-by-business-capability.html)
- [Google SRE: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)

