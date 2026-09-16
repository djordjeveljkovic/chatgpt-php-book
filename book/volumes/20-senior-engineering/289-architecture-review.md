---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 289
title: Architecture Review
slug: architecture-review
status: complete
summary: ../../_ai/chapter-summaries/289-architecture-review-summary.md
---

# Chapter 289 — Architecture Review

## Why This Matters

Code review asks whether a concrete change is safe. Architecture review asks whether the proposed system shape makes safety, ownership, change, failure, and operation manageable. A system can contain individually correct classes and still have the wrong boundaries, duplicated sources of truth, unsafe effects, or an impossible recovery plan.

Architecture is not only a diagram. It is visible in dependency direction, Composer packages, database ownership, queues, caches, deployment units, process limits, and the people who are expected to operate the result. An architectural choice becomes expensive when implementation, data, traffic, and team responsibility have already formed around it.

## Mental Model: Architecture Allocates Responsibility

Review architecture as a set of connected maps:

| Map | Question |
| --- | --- |
| capability and actor | which users, jobs, and teams need this capability? |
| component and dependency | which component calls, imports, or deploys with which other component? |
| data ownership | who may create, change, validate, and repair each durable fact? |
| effects and timing | which work is synchronous, asynchronous, retried, or externally visible? |
| deployment and operations | which artifact, process, alert, and recovery action belongs to whom? |

The central question is: who owns each decision, state transition, failure, metric, and recovery action? If an answer is “everyone,” the boundary is probably not explicit enough.

## Define the Review Scope

Start an architecture review with a brief:

~~~text
goal: add tenant-scoped product search without changing product ownership
users: catalog operators and customer-facing applications
systems: PHP-FPM API, catalog database, cache, search projection, queue worker
non-goals: replacing the catalog database or splitting the whole application
qualities: tenant isolation, stable ordering, bounded latency, rebuildability
constraints: mixed PHP releases and one database owner during migration
decision horizon: operate the design for at least three years
~~~

Record affected users and tenants, systems and teams, expected lifetime, blast radius, non-goals, irreversible decisions, and constraints. Keep line-level style and naming out of the architecture review unless they reveal an ownership or dependency problem. A review is proportional when it spends the most time on the decisions that are expensive to reverse or dangerous to operate.

## Architecture Review and Code Review

The two activities overlap, but they answer different questions:

| Code review | Architecture review |
| --- | --- |
| concrete diff and its callers | system structure and evolution |
| local behavior and implementation evidence | boundaries, dependencies, ownership, and quality attributes |
| merge safety | fitness of a proposed shape |
| tests for changed behavior | failure, migration, capacity, and recovery evidence |
| approve, request changes, or accept risk | approve, revise, prototype, defer, or reject a structural decision |

An architecture review may result in a small implementation. A code review may expose an architectural defect. The distinction prevents a reviewer from demanding a rewrite merely because a design is unfamiliar, while still giving structural risks the attention they deserve.

## Build an Architecture Review Packet

Ask the author to provide:

* current-state and proposed-state context maps;
* a data-flow or sequence diagram for normal and failure paths;
* invariants and quality attributes in measurable language;
* dependency, data-owner, and external-effect inventories;
* runtime, capacity, deployment, and rollback assumptions;
* migration stages and mixed-version behavior;
* evidence already available and questions still unresolved;
* an owner and review date for accepted risks.

A diagram is useful when it explains ownership, failure, or change. Boxes and arrows that do not identify sources of truth, trust boundaries, synchronous calls, queues, or deployment units are decoration rather than review evidence.

## Define Architectural Fitness

Replace vague claims such as “clean,” “decoupled,” or “scalable” with qualities and signals:

| Quality | Example architectural claim | Possible fitness signal |
| --- | --- | --- |
| correctness | one owner enforces reservation overlap | concurrent conflict test and constraint inspection |
| isolation | search never crosses tenant scope | cross-tenant query, cache, and export tests |
| availability | catalog reads survive an optional cache outage | dependency failure test and error-budget policy |
| performance | p95 search latency stays within the API budget | query plan and workload test |
| operability | operators can identify and stop a bad rollout | bounded metrics, alert, flag, and runbook |
| evolvability | old and new workers can coexist during migration | compatibility matrix and mixed-version rehearsal |

These checks are sometimes called fitness functions: repeatable observations that constrain architectural drift. They are not a substitute for judgment. A passing latency check under the wrong workload or a passing isolation test that omits exports can create false confidence.

## Review Boundaries and Dependency Direction

Choose boundaries around capabilities and ownership, not around the number of classes or the popularity of a pattern. In a modular monolith, a `Catalog` module may expose an application service while preventing a `Reservation` module from querying catalog tables directly. A Composer package boundary can make that rule visible, but namespaces alone do not enforce it.

Review for:

* domain code importing framework, database, or provider details in the wrong direction;
* modules sharing tables without a named data owner;
* global configuration, static registries, and hidden process-local state;
* synchronous calls that create latency and failure coupling;
* queues introduced without delivery, schema, retry, and ownership contracts;
* cycles between modules or packages;
* a service split that merely renames a class and adds a network boundary.

Use adapters when a capability must depend on a legacy representation or external provider. The adapter should translate a contract and expose the remaining failure semantics; it should not conceal that a remote call can time out or that a database is still shared.

## Review Ownership

Make ownership explicit:

| Concern | Required owner |
| --- | --- |
| authorization policy | one authoritative policy boundary |
| durable product fact | source-of-truth component |
| schema and migration | data owner |
| retry and idempotency identity | effect or workflow owner |
| cache invalidation | projection/cache owner |
| queue replay and repair | workflow/operator owner |
| alerts and service objectives | operational owner |
| rollback and forward recovery | deployment/data owner |

“Shared ownership” is often a polite description of an unsafe gap. Two components may collaborate, but one must decide what is valid and one must be accountable for repair. Ownership should appear in interfaces, runbooks, dashboards, permissions, and deployment rules—not only in a meeting note.

## Review Data and Consistency

For every important fact, ask:

1. What is the source of truth?
2. Who may write it, and where is that authorization enforced?
3. What must be atomic?
4. What may be eventually consistent, and for how long?
5. How are caches, projections, and search indexes rebuilt?
6. Which schema versions can old and new readers understand?
7. What happens when a backfill, projection, or reconciliation job stops halfway?

Chapter 287’s search results are derived state. Product rows remain authoritative; the search projection needs an explicit lag signal, rebuild path, tenant scope, and behavior when it is stale or unavailable. Chapter 280’s reservation overlap is a durable invariant; it belongs at a concurrency boundary that can serialize the decision, not in two independently deployed copies of a pre-check.

Do not let a cache become an accidental source of truth. Do not let a search index become a second writer because it is convenient to update there. If strong consistency is unnecessary, state the accepted stale window and the user-visible behavior. If it is required, identify the cost and evidence rather than calling the system “real time.”

## Review Effects and Workflow Boundaries

Trace every externally visible effect:

* payments and refunds;
* email, SMS, and notifications;
* webhooks and partner calls;
* queue messages and scheduled jobs;
* file writes and exports;
* audit records and privacy-sensitive events.

For each effect, identify transaction ownership, timeout ambiguity, retry ownership, idempotency identity, outbox/inbox behavior, duplicate handling, and repair. A payment call inside a database transaction couples provider latency to database locks and still does not make the external effect atomic with the local commit. An outbox can close the publish gap, but it creates a delivery and reconciliation responsibility that must be operated.

## Security Architecture

Review trust boundaries, not merely individual validation calls. Verify:

* authenticated identity and tenant scope flow through HTTP, CLI, jobs, exports, caches, and repair tools;
* authorization has one clear policy owner and fails safely;
* secrets are delivered, rotated, and revoked at the right process boundary;
* service-to-service identity cannot be confused with end-user authorization;
* administrative and replay paths have least privilege and audit records;
* logs, metrics, traces, queues, and backups do not widen the sensitive-data boundary;
* rate limits and resource quotas protect shared dependencies from noisy tenants.

If authorization is applied after a cross-tenant query or after a shared cache lookup, the architecture is already wrong. Moving the check into a controller does not repair a source-of-truth or cache-key boundary.

## Runtime, Capacity, and Failure Domains

Make the proposed structure concrete in PHP environments:

| Runtime | Architectural question |
| --- | --- |
| PHP-FPM | what is the per-request memory, connection, and timeout budget? |
| CLI/cron | how are leases, overlap, retries, and operator invocation controlled? |
| long-running worker | how are stale configuration, memory growth, and graceful drain handled? |
| queue/broker | who owns backlog, poison messages, replay, and schema compatibility? |
| database/cache | which connection, lock, hot-key, and failure budgets are shared? |

Estimate request and queue rates, payload sizes, database connections, fan-out, memory per worker, retry amplification, backlog growth, and noisy tenants. A service boundary can isolate a failure while creating a new bottleneck. A cache can lower average latency while concentrating expiry load into a stampede. A worker split can improve throughput while exhausting the database connection pool.

Walk failure scenarios before approving the shape:

| Failure | Architectural question |
| --- | --- |
| database unavailable | does the request fail safely and visibly? |
| provider timeout | is a transaction held open, and is completion unknown? |
| duplicate queue delivery | where does idempotency converge the effect? |
| stale worker after deploy | are message and configuration versions compatible? |
| cache outage | is the cache optional, bounded, and observable? |
| partial migration | can old and new code coexist and can repair resume? |

The architecture is incomplete if nobody can observe, drain, stop, roll forward, restore, replay, or repair it.

## Make Trade-Offs Explicit

Compare alternatives against the qualities that matter:

| Decision | Questions to compare |
| --- | --- |
| modular monolith vs service split | ownership, latency, deployment, failure isolation, team capacity |
| synchronous vs asynchronous work | user latency, unknown completion, delivery, replay, consistency |
| query vs cache/search projection | freshness, rebuildability, cost, isolation, invalidation |
| strong vs eventual consistency | invariant, stale window, conflict handling, availability, recovery |
| shared vs owned database | transaction needs, migration authority, coupling, backup and restore |

Every option has a cost. “Move it to a service” is not a quality attribute, and “keep it simple” is not an analysis. State what improves, what worsens, which uncertainty remains, and what would cause the decision to be revisited.

## Migration and Evolution

Prefer an expand-and-contract sequence when versions must overlap:

1. add a compatible schema, interface, message field, or adapter;
2. deploy readers and writers that tolerate both representations;
3. backfill or migrate gradually with checkpoints and reconciliation;
4. switch traffic or ownership using observable stop conditions;
5. remove compatibility code only after old consumers and recovery paths are gone.

The same shape applies to replacing a Composer package, migrating a framework boundary, changing a database column, or extracting a modular-monolith capability. Reversibility is not the same as a binary Git revert: messages emitted, rows transformed, caches filled, and external effects may require forward recovery or repair.

## Architecture Decision Records

Record why the structure was selected:

~~~text
title:
status: proposed | accepted | superseded
context and constraints:
decision:
alternatives considered:
quality attributes and fitness signals:
data and effect ownership:
failure modes:
migration and rollback/recovery:
observability and stop conditions:
accepted risks:
owner and review date:
~~~

An ADR is not a list of classes. It preserves the reasoning, consequences, and boundaries that would otherwise disappear when the original authors leave or the implementation becomes familiar. Accepted risks need an owner, a measurable consequence, and a review condition.

## A PHP Case Study

Suppose a tenant-scoped catalog search is proposed as a new service. The request enters PHP-FPM, authenticates the tenant, calls a search application service, reads a database or projection, fills a cache, and may enqueue a reindex job. A naive split creates two product writers, copies authorization into three services, and lets a notification provider run inside a transaction.

A better architecture keeps product truth in the catalog owner, exposes a narrow search contract, treats the index and cache as rebuildable derived state, carries tenant scope through every key and job, and uses an outbox for asynchronous reindex or notification effects. It may remain a modular monolith until separate deployment, scaling, security, or failure isolation justifies a process boundary.

The review should demand evidence for stable ordering and cursor compatibility, tenant isolation, projection lag, query cost, queue delivery, cache failure, mixed-version workers, and recovery. The question is not whether the service diagram looks modern. It is whether the structure preserves the required qualities at a cost the team can operate.

## Architecture Review Workflow

Use a repeatable sequence:

1. intake the goal, constraints, and risk;
2. inspect current state before proposed state;
3. map boundaries, owners, invariants, and effects;
4. walk normal, timeout, duplicate, partial, and recovery paths;
5. review security, data, runtime, capacity, operations, and migration;
6. identify evidence gaps and the smallest useful experiment;
7. compare alternatives and state consequences;
8. decide to approve, revise, prototype, defer, or reject;
9. record accepted risks, owners, deadlines, and stop conditions;
10. revisit the decision after implementation and rollout.

## Common Mistakes

* confusing a diagram with an architecture;
* treating microservices as automatic decoupling;
* ignoring shared database tables and hidden transaction ownership;
* duplicating authorization or durable writes across boundaries;
* putting provider calls inside database transactions;
* adding queues without idempotency, schema compatibility, or replay ownership;
* measuring average latency while ignoring tail latency, memory, locks, and backlog;
* omitting migration, rollback, repair, and mixed-version behavior;
* recording decisions without alternatives, consequences, owners, or review dates;
* requesting a rewrite when a focused boundary, adapter, or fitness check would reduce the risk.

## Senior Engineer Thinking

Architecture review is responsibility allocation under uncertainty. Boundaries determine who owns truth, timing, failure, security, capacity, and recovery. A senior engineer does not choose a pattern first and search for justification afterward. They identify the qualities that matter, make failure modes visible, compare costs, ask what evidence could falsify the safety claim, and leave a decision that another team can operate.

Every accepted architectural compromise is a candidate debt item. Chapter 290 will examine how to classify and manage the obligations that remain after a reasonable decision.

## Exercises

1. Draw current and proposed boundary maps for the Chapter 287 search service, including tenant scope, database, cache, projection, queue, and deployment units.
2. Assign one owner to each product row, cache value, search document, queue message, retry, alert, rollback action, and repair procedure.
3. Compare a modular monolith and a service split using correctness, latency, consistency, operations, migration cost, and team ownership.
4. Identify the consistency requirements and concurrency boundary for the Chapter 280 reservation service.
5. Design a failure matrix for a payment-plus-notification workflow, including unknown completion and replay.
6. Estimate PHP-FPM, database, cache, and queue capacity for a stated workload and identify the first shared bottleneck.
7. Write an ADR for replacing a database query with a rebuildable search projection.
8. Turn “the system should scale” and “the service should be decoupled” into measurable fitness signals.

## Review Questions

* How does architecture review differ from code review?
* Which maps reveal responsibility allocation and hidden coupling?
* Why are data ownership and authorization boundaries inseparable?
* What makes a search index or cache derived state rather than a source of truth?
* Which questions expose unsafe external effects and unknown completion?
* How do PHP-FPM, CLI, and long-running workers change architectural analysis?
* Why should failure-mode analysis precede a service split or queue introduction?
* What evidence fits a capacity, security, migration, or recovery claim?
* What must an ADR preserve beyond the selected class structure?
* When is a modular monolith the safer architecture?

## Summary

Architecture review evaluates whether responsibilities, boundaries, data ownership, effects, quality attributes, failure behavior, runtime limits, migration, and operations form a system the team can keep safe. Map current and proposed structure, name one owner for each important decision and fact, make consistency and external effects explicit, quantify capacity and fitness, test failure and mixed-version paths, compare trade-offs, record accepted risks, and revisit the architecture after rollout. A good boundary makes correctness, change, and recovery easier to own.

## References

- [Chapter 240 — Backoff](../../volumes/16-distributed-systems/240-backoff.md)
- [Chapter 241 — Partial Failure](../../volumes/16-distributed-systems/241-partial-failure.md)
- [Chapter 242 — Idempotency](../../volumes/16-distributed-systems/242-idempotency.md)
- [Chapter 251 — Eventual Consistency](../../volumes/16-distributed-systems/251-eventual-consistency.md)
- [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md)
- [Chapter 275 — Branch by Abstraction](../../volumes/18-legacy-php/275-branch-by-abstraction.md)
- [Chapter 277 — Database Migration](../../volumes/18-legacy-php/277-database-migration.md)
- [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 288 — Code Review](288-code-review.md)
