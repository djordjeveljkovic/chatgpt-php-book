---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 271
title: Legacy Architecture
slug: legacy-architecture
status: complete
summary: ../../_ai/chapter-summaries/271-legacy-architecture-summary.md
---

# Chapter 271 — Legacy Architecture

## Why This Matters

Legacy architecture is not architecture that looks old. It is architecture whose important boundaries are difficult to see, difficult to change, or owned by nobody. A PHP application can have namespaces, Composer, a framework, and automated deployment while still depending on a global include, a shared database table, a cron overlap assumption, or a particular order of side effects.

The danger is not age by itself. The danger is that a change crosses an invisible boundary. Replacing a template helper may alter escaping. Moving a query into a repository may change transaction ownership. Splitting a cron command may duplicate an email. Extracting a service may leave authorization in the old process while writes move to the new one.

The first architectural task is therefore to make the system legible. Identify what runs, who owns each decision and piece of data, which effects cross process boundaries, and where evidence can prove that a change preserved the contract. Only then can a team choose between stabilization, refactoring, extraction, or replacement.

## Mental Model

Model the legacy application as a graph of responsibilities and effects:

~~~text
entry points ──→ orchestration ──→ business decisions ──→ data changes
     │                  │                    │                  │
     ├─ web/FPM         ├─ globals            ├─ SQL             ├─ tables
     ├─ CLI/cron        ├─ includes           ├─ templates       ├─ files
     ├─ queue           ├─ framework hooks    ├─ providers       └─ messages
     └─ admin           └─ shared state        └─ authorization
~~~

The graph has at least four different kinds of edge:

* a call edge: one function or class invokes another;
* a data edge: one component reads or writes a value another component owns;
* a lifecycle edge: behavior depends on bootstrap order, request scope, or process lifetime;
* an effect edge: code sends a message, changes a file, charges a card, or publishes an event.

A call graph alone misses the most important legacy dependencies. A function can be textually independent while still depending on a session variable, a database trigger, an include side effect, or a queue worker’s current directory.

## Architecture as an Observed System

The intended architecture may be documented in a diagram, but the effective architecture is the set of dependencies that can change behavior. Compare three views:

| View | Question | Typical evidence |
| --- | --- | --- |
| Intended | What do the team and documents say should happen? | diagrams, ADRs, module names |
| Static | What relationships are present in source and configuration? | imports, includes, SQL, routes, scripts |
| Observed | What actually runs and produces effects? | traces, logs, queries, files, messages |

Disagreement between the views is an architectural finding, not an invitation to declare one view correct. An unused-looking class can be reached by a generated route. A table described as read-only can be written by a report script. A service boundary drawn in a diagram may be only an HTTP call with no independent data ownership.

Begin with a bounded capability such as invoice creation or password reset. List its entry points, decisions, data reads and writes, external effects, process owners, failure behavior, and recovery evidence. A small accurate map is more useful than a large speculative diagram.

## Common Legacy Shapes

### The Shared Bootstrap

Many requests enter through a front controller that includes configuration, opens a database connection, defines helper functions, starts a session, and then includes a page script. The page script may rely on every side effect having occurred in a particular order.

This is not automatically wrong. It becomes dangerous when the bootstrap is an untyped hidden dependency. A CLI command may include only part of it. A test may call a page function without it. A second entry point may define a slightly different constant or database mode.

### The Page That Owns Everything

A page controller may authenticate a user, query several tables, apply business rules, render HTML, write audit rows, and send mail. Its code is easy to reach but hard to reason about because presentation, policy, persistence, and effects share a transaction and error path.

Extracting a service class does not solve this by itself. First identify which decisions must remain together, which data is authoritative, and which effect can be delayed or retried. The new class should represent a real responsibility, not merely move lines to another file.

### The Database as a Shared Module

In a legacy system, the database often acts as an undocumented module boundary. Multiple applications may depend on column names, trigger side effects, status values, stored procedures, reporting views, and the timing of background jobs.

If two processes write the same table, they may share business invariants and necessarily share the table’s schema and constraint effects, whether or not they share a repository class. Data ownership must be discovered from writes and constraints, not inferred from the directory that contains a query.

### The Scheduled Workflow

A cron job can be the real coordinator of the product. It may select rows, change a status, create files, call a provider, and rely on the next run to repair an interrupted step. Its architecture includes schedule frequency, overlap behavior, host selection, lock policy, and replay behavior.

Treat the job as a process with a contract. Chapter 243 covers message delivery and Chapter 249 covers distributed locks; this chapter asks where the process belongs and who owns the workflow state.

## Find Hidden Coupling

Search and observation should be combined. Inspect these surfaces:

* includes, autoloaders, bootstrap files, constants, globals, and superglobals;
* database tables, triggers, views, stored procedures, and status columns;
* sessions, cache keys, file names, shared directories, and lock files;
* route names, form fields, serialized payloads, CLI arguments, and environment variables;
* cron entries, queue consumers, deployment hooks, and administrative scripts;
* provider calls, email templates, webhooks, and retry or reconciliation jobs;
* framework events, magic methods, filters, callbacks, and output buffering.

For each dependency, record whether it is explicit, inferred, observed, or unknown. Record the owner and the smallest safe experiment. A dependency map should answer “what can break if this changes?” rather than merely “what calls this?”

~~~text
unknown dependency
       ↓ inspect
static reference ──→ runtime observation ──→ contract evidence
       │                    │                       │
       └──── no path? ──────┴──── mark unknown ─────┘
~~~

Do not use a search result as permission to delete code. Prove reachability, prove non-use across entry points and deployments, or keep the behavior while adding instrumentation and an owner.

## Architectural Boundaries

### Responsibility Boundary

A responsibility boundary groups a decision and the data needed to make it. “Orders” is too broad if one module owns pricing, payment capture, fulfillment, and reporting. “Calculate an order total from authorized line items” is a decision that can have a clear input, output, and invariant.

### Data Boundary

The data owner is the component responsible for invariants, writes, migrations, retention, and recovery. A component that can read a table is not necessarily its owner. Shared writes create coordination cost and make extraction unsafe until the invariant and transition are explicit.

### Process Boundary

Web requests, CLI commands, cron jobs, queue workers, and deployment hooks have different lifetimes and failure modes. Process-local state, current working directory, loaded configuration, session locks, and open connections cannot be assumed to cross the boundary safely.

### Effect Boundary

External effects need an identity, an outcome record, and a recovery policy. “Send an email” is not just a function call if it can time out after the provider accepted it. Put the effect behind an explicit boundary and connect it to durable state or an outbox when the business invariant requires reconciliation.

### Ownership Boundary

Every important decision should have a named owner: a module, team, or process with authority to change it and evidence to verify it. Architecture without ownership becomes a collection of dependencies that everybody uses and nobody can safely alter.

## A Small Architecture Catalog

Use a typed catalog to turn a diagram into reviewable evidence:

~~~php
<?php

declare(strict_types=1);

enum ArchitectureEdge: string
{
    case Call = 'call';
    case Data = 'data';
    case Lifecycle = 'lifecycle';
    case Effect = 'effect';
}

final readonly class ArchitectureDependency
{
    public function __construct(
        public string $from,
        public string $to,
        public ArchitectureEdge $edge,
        public string $contract,
        public string $owner,
        public bool $observed,
    ) {
        if ($from === '' || $to === '' || $contract === '' || $owner === '') {
            throw new InvalidArgumentException('Architecture dependency is incomplete');
        }
    }
}

function isUnobserved(ArchitectureDependency $dependency): bool
{
    return !$dependency->observed;
}
~~~

The catalog does not pretend to discover architecture automatically. It makes the team state what relationship exists, what contract it carries, who owns it, and whether the relationship has been observed. `observed: false` is useful because it turns an assumption into a testable work item.

The catalog can be stored temporarily in a review document or represented by a small machine-readable file. Do not build a permanent architecture registry before proving that the fields help decisions. The useful unit is a dependency that changes the risk of a specific change.

## Dependency Direction

Legacy code often has a dependency direction that is accidental:

~~~text
HTTP page → global helper → database handle → business rule
   ↑             │              ↓              │
template ←───────┴──────────── shared state ←──┘
~~~

The goal of a refactoring seam is not to impose fashionable layers. It is to make direction visible and protect a decision from infrastructure details. A durable direction for a small capability might be:

~~~text
entry point → application operation → domain decision
                    │                     │
                    ├─ repository port   └─ pure policy
                    └─ effect port
                         ↓
                    adapters and infrastructure
~~~

The application operation coordinates. The domain decision owns a rule that does not require a database connection. A repository or effect port describes what the operation needs; an adapter handles SQL, HTTP, mail, or legacy globals. This direction is valuable only when the contract is explicit and the adapter can be replaced or observed.

Do not move every class into `Domain`, `Application`, and `Infrastructure` directories while preserving global state and hidden transactions. Directory names cannot create a boundary that behavior does not support.

## Transactions and Ownership

A common extraction failure is to move a write while leaving the transaction boundary behind. Suppose a legacy page inserts an order, updates stock, writes an audit row, and sends mail. If a new service owns only the insert, the old page may continue to own the stock update and audit row. A failure can now leave two owners with incompatible partial states.

Before extracting, document:

* the invariant that must hold;
* the database transaction scope;
* which process owns each write;
* whether external effects occur before or after commit;
* the operation identity used for retries;
* the evidence used to reconcile ambiguous outcomes.

A seam is safe when the old and new paths can coexist without violating these facts. If they cannot, first introduce a compatibility boundary or a single owner, then migrate callers.

## Runtime and Framework Boundaries

Legacy architecture is shaped by the runtime. A global connection may be harmless in a short-lived request and dangerous in a long-running worker. A static cache may improve one request and leak tenant data into the next job. A framework hook may run before authorization in one entry point and after it in another.

Compare entry points explicitly:

| Entry point | Lifetime | Configuration | State risk | Recovery question |
| --- | --- | --- | --- | --- |
| Web request | short-lived; handled by pooled FPM workers | web SAPI and FPM | session, globals, output | can the request be retried? |
| CLI command | one invocation | CLI `php.ini`, user, cwd | files, locks, exit codes | what happens after interruption? |
| Cron job | repeated process | scheduler environment | overlap, duplicate effects | who owns the schedule and lease? |
| Queue worker | long-lived | startup snapshot | stale memory, old code | how are messages and workers drained? |

Chapter 256 covers PHP-FPM process behavior, while Chapter 258 covers configuration. The architectural question is which assumptions cross those boundaries and where they must be removed or made explicit.

## Observability as an Architectural Seam

When behavior is not understood, instrumentation is often safer than immediate extraction. Add bounded evidence at a boundary:

* operation name and version;
* actor or tenant identifier where permitted;
* input shape or stable hash, not sensitive payloads;
* dependency and database operation names;
* outcome classification and duration;
* operation identity for retried effects;
* source path, worker identity, and configuration version.

Use the logging, metrics, and tracing guidance from Chapters 260–262. Avoid adding unbounded SQL text, personal data, or raw serialized payloads. Instrumentation is part of the architecture only when it has a defined owner, retention, and failure behavior.

## Incremental Architectural Change

Prefer transitions that leave a working, observable system after each step:

~~~text
map one capability
      ↓
record contracts and invariants
      ↓
instrument the old path
      ↓
introduce an adapter or seam
      ↓
move one decision or owner
      ↓
compare outcomes and reconcile
      ↓
remove the old path after evidence
~~~

Useful seams include a request-to-application-operation adapter, a repository around one query family, a command translator for old queue messages, a file-system port, or an effect recorder for provider calls. The seam must reduce the number of places that know a legacy detail. If it only adds another forwarding layer, it may increase complexity without reducing coupling.

For dual paths, define the authority. A shadow read may compare results without changing the answer. A dual write needs idempotency, reconciliation, and a rule for conflicts. A traffic split needs compatible schemas, messages, caches, and rollback. Chapters 264–265 cover rollout and rollback evidence; Chapters 274–275 develop specific migration patterns.

## Common Mistakes

* Drawing boxes around directories and calling them modules.
* Treating a database schema as owned by the application that has the newest code.
* Extracting a class without moving the invariant or transaction owner.
* Replacing a global with a service locator and calling the dependency explicit.
* Testing only HTTP requests while ignoring CLI, cron, queue, and deployment entry points.
* Sending both old and new paths to external providers without operation identity.
* Adding telemetry that leaks payloads or creates unbounded cardinality.
* Creating a large architecture catalog with no evidence or decision attached.
* Moving business rules into adapters because the legacy database is inconvenient.
* Splitting a process before defining data ownership and recovery behavior.
* Removing an include or hook because static analysis cannot find its caller.
* Keeping a compatibility seam after its owner, contract, or removal condition has disappeared.

## Senior Engineer Thinking

The senior question is not “which architecture diagram is most modern?” It is “which behavior is owned where, which edge is invisible, which invariant crosses a process, and what smallest experiment can make the next change safer?”

Legacy architecture rewards humility and precision. Preserve observed behavior while it is being understood, but do not mistake accidental coupling for a business requirement. Name the boundary, assign ownership, record the evidence, and choose a transition whose intermediate state is safe. A good architecture change reduces the amount of knowledge required to make the next change.

## Exercises

1. Choose one capability in a legacy application. Draw intended, static, and observed maps, marking every call, data, lifecycle, and effect edge.
2. Build ten `ArchitectureDependency` records. For each unobserved relationship, write the smallest safe experiment that could verify it.
3. Find a workflow that spans a web request and a cron or queue process. Identify its invariant, transaction scope, owner, retry identity, and recovery evidence.
4. Design a seam around one legacy global or database helper. State what it preserves, what it intentionally changes, and how a shadow or characterization test will detect drift.
5. Review a proposed extraction. Reject it if it does not define data ownership, effect identity, rollback behavior, or a safe coexistence period.

## Review Questions

* Why can a call graph miss important legacy dependencies?
* What is the difference between intended, static, and observed architecture?
* Why is a shared database table an ownership boundary even when code is separated?
* Which contracts must be defined before moving a transaction or external effect?
* Why do web, CLI, cron, and queue entry points need separate architectural analysis?
* What does an unobserved dependency record communicate?
* When does an adapter reduce coupling, and when does it merely add forwarding code?
* Why are dual writes harder than shadow reads?
* What makes observability a safe architectural seam?
* What evidence should allow a legacy path to be removed?

## Summary

Legacy architecture is the effective graph of responsibilities, data, lifecycles, effects, and ownership—not the directory tree or the intended diagram. Compare intended, static, and observed behavior; find hidden coupling; assign data and process ownership; make transaction and effect boundaries explicit; account for different PHP entry points; and use bounded evidence before extracting. Incremental seams are safe when old and new paths can coexist, preserve invariants, identify retries, expose outcomes, and have a removal condition. The goal is not fashionable layering; it is reducing invisible knowledge and making the next change verifiable.

## References

- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 249 — Distributed Locks](../16-distributed-systems/249-distributed-locks.md)
- [Chapter 256 — PHP-FPM](../17-production-engineering/256-php-fpm.md)
- [Chapter 258 — Configuration](../17-production-engineering/258-configuration.md)
- [Chapter 260 — Logging](../17-production-engineering/260-logging.md)
- [Chapter 261 — Metrics](../17-production-engineering/261-metrics.md)
- [Chapter 262 — Tracing](../17-production-engineering/262-tracing.md)
- [Chapter 264 — Deployment](../17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 272 — Characterization Tests](./272-characterization-tests.md)
- [Chapter 274 — Strangler Pattern](./274-strangler-pattern.md)
- [Chapter 275 — Branch by Abstraction](./275-branch-by-abstraction.md)
