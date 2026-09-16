---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 274
title: Strangler Pattern
slug: strangler-pattern
status: complete
summary: ../../_ai/chapter-summaries/274-strangler-pattern-summary.md
---

# Chapter 274 — Strangler Pattern

## Why This Matters

Replacing a legacy PHP application all at once creates a large period in which nobody can tell whether a failure came from the new code, the old code, the runtime, the database, or the migration itself. The Strangler Pattern reduces that exposure by moving one capability or route at a time while the old system continues to serve the rest.

The pattern is not “put a proxy in front of a rewrite.” It is a sequence of ownership transitions. For each slice, the team must decide which path is authoritative, which data it owns, how old and new requests coexist, how effects are identified, what evidence proves parity, and how to return traffic safely.

## Mental Model

The legacy system remains behind a boundary while new capabilities grow around it:

~~~text
request
  ↓
capability router
  ├─ migrated slice ──→ new implementation
  └─ remaining paths ─→ legacy implementation
                         ↓
                 shared or transitioning state
~~~

Each migrated slice needs a contract, an authority, an observation plan, and a retirement condition. The router may be an HTTP proxy, front controller, feature decision, CLI dispatcher, queue consumer, or scheduled-job selector. Routing is only one part of the transition.

## Choose a Slice

Choose a capability with a bounded entry point and a comprehensible data and effect boundary:

* a read-only report with stable authorization;
* one admin workflow with isolated tables;
* a search endpoint whose result contract is already characterized;
* a queue message version with one consumer;
* a file import path with an explicit completion record.

Avoid slicing by a directory or framework namespace. A route that reads shared tables, writes an audit row, and triggers a cron repair job is not isolated merely because its controller has a unique filename.

Create a slice charter:

~~~text
slice: invoice receipt lookup
entry points: GET /invoices/{id}/receipt, admin preview
authority: new reader; legacy remains write authority
data: same invoice and authorization tables
effects: none in read path
parity evidence: status, headers, body shape, authorization, query count
rollback: route flag to legacy reader
retirement: old reader removed after traffic and consumer review
~~~

The charter makes an incomplete migration visible. If the new path cannot state its authority or rollback, it is not ready for traffic.

## Establish the Boundary

The strangler boundary should be as close as practical to the entry point, but it must preserve security and request semantics. Check:

* authentication, tenant context, authorization, and CSRF behavior;
* path, method, query, body, headers, cookies, and content negotiation;
* session and cache behavior;
* timeout, retry, redirect, and error classification;
* correlation identifiers and audit evidence;
* upload, streaming, and output-buffering behavior.

An HTTP hop does not automatically create a safe boundary. If the new service trusts headers that the old front controller used to derive, the migration may create an authorization defect. If both paths share a session format or cache key without a compatibility contract, traffic can fail in surprising ways.

## Read Migration

Read-only slices are often easier because they can compare old and new results without immediately creating two writers. Use a controlled strategy:

~~~text
request → new reader → response
           │
           └─ optional shadow read → normalized comparison
                                  ↓
                              bounded evidence
~~~

The shadow path must not change the user-visible answer or trigger writes. Compare authorization, missing-versus-null values, ordering, precision, pagination, cache headers, and error states—not only rendered text.

A read migration still has a data contract. The new path must understand old status values, timestamps, encodings, soft deletes, triggers, and replica freshness. Chapter 272 covers behavior capture; Chapter 277 covers deeper database transitions.

## Write Authority

For a write slice, choose exactly one authoritative writer during each phase. Common phases are:

| Phase | Old path | New path | Main risk |
| --- | --- | --- | --- |
| observe | authoritative | shadow/read-only | hidden new side effects |
| route | bypassed for slice | authoritative | rollback compatibility |
| drain | no new writes | authoritative | old retries or jobs |
| retire | removed or isolated | authoritative | unowned repair paths |

Dual writes are not a neutral transition. They require operation identity, idempotency, conflict handling, ordering, and reconciliation. If the old and new writers use different transaction boundaries, a successful response can leave divergent state.

For external effects, route one authority at a time. A payment, email, webhook, or file publication should not be emitted by both paths because a traffic experiment needs more data.

## Data Ownership During Coexistence

A shared database can support a strangler transition, but only with explicit ownership:

* identify every writer, including cron, reports, admin scripts, and triggers;
* preserve schema and message compatibility while both versions run;
* define which version owns each invariant and status transition;
* fence old workers before changing their interpretation of rows;
* record migrations and backfills separately from application rollout;
* reconcile old and new observations before changing authority.

Do not let the new application silently create a second interpretation of the same status column. If it needs a different state model, add a translation or versioned field and document the transition.

## Route Selection

Select traffic with a deterministic, observable policy:

~~~php
<?php

declare(strict_types=1);

enum RouteTarget: string
{
    case Legacy = 'legacy';
    case New = 'new';
}

final readonly class RouteDecision
{
    public function __construct(
        public RouteTarget $target,
        public string $reason,
        public string $policyVersion,
    ) {
        if ($reason === '' || $policyVersion === '') {
            throw new InvalidArgumentException('Route decision is incomplete');
        }
    }
}

function chooseTarget(bool $enabled, bool $compatible): RouteDecision
{
    if (!$enabled) {
        return new RouteDecision(RouteTarget::Legacy, 'flag-disabled', 'v1');
    }

    if (!$compatible) {
        return new RouteDecision(RouteTarget::Legacy, 'compatibility-failed', 'v1');
    }

    return new RouteDecision(RouteTarget::New, 'slice-enabled', 'v1');
}
~~~

The decision records why traffic was routed and which policy version made the choice. In production, the policy also needs a safe source, bounded dimensions, and a fail-safe default. Never route based on an unvalidated client-controlled header or an unstable random choice that cannot be reproduced during investigation.

## Rollout Stages

Use stages that increase exposure only after evidence:

1. deploy the new path dark and verify startup, configuration, and health;
2. run read-only or shadow checks with controlled fixtures;
3. route internal or synthetic traffic;
4. route a small, clearly selected cohort;
5. expand by measured capability and failure domain;
6. drain legacy workers and old retries;
7. remove the old path after consumers and recovery procedures are updated.

Each stage has entry and exit criteria. Track error classification, authorization differences, latency, resource use, database load, effect count, queue age, and rollback readiness. Chapter 264 covers rollout evidence and Chapter 265 covers rollback boundaries.

## Rollback Is a Data Decision

Routing traffic back is easy only when the new path has not changed irreversible state. Before enabling a slice, answer:

* Can old code read rows written by new code?
* Can old workers process messages emitted by new code?
* Are caches and sessions compatible?
* Which writes or external effects happened after the cutover?
* How will divergent observations be reconciled?
* Who has authority to stop traffic and repair state?

If the answer is no, use forward recovery, a compatibility translator, or a capability-specific degraded mode. A code rollback cannot undo a payment, email, schema deletion, consumed message, or provider callback. Chapter 265 develops this distinction.

## Observability and Operations

Give every request and job a route decision, slice version, operation identity, and outcome classification. Keep old and new metrics distinguishable without creating unbounded tenant or URL dimensions. Compare:

* traffic volume and authorization outcomes;
* response status and error classes;
* database reads/writes and lock timing;
* cache hit behavior and session failures;
* external effect attempts and unknown completion;
* queue processing and retry behavior;
* latency, memory, worker age, and capacity.

A strangler transition is an operational change. Update runbooks, alerts, dashboards, on-call ownership, and incident rollback instructions before broadening traffic.

## Retire the Legacy Slice

Retirement is a separate change. Before removing the old path:

1. prove that no supported route, job, consumer, admin tool, or recovery script still uses it;
2. verify old messages and in-flight work are drained or translated;
3. remove compatibility code only after the expiry condition is met;
4. preserve audit and reconciliation evidence;
5. rehearse the recovery path for the new owner;
6. remove routing flags and dead dashboards after observation retention expires.

Leaving a flag, adapter, or duplicate route indefinitely increases the number of possible architectures. Retirement is part of completing the migration, not optional cleanup.

## Common Mistakes

* Slicing by folder instead of capability and ownership.
* Putting a proxy in front of the application without preserving auth and request semantics.
* Running old and new writers without an authority or reconciliation policy.
* Shadowing a path that still sends mail, publishes messages, or mutates caches.
* Routing by client-controlled or unobservable criteria.
* Treating shared tables as private to the new path.
* Rolling back code after irreversible effects without a recovery plan.
* Ignoring cron, queue, admin, report, and repair consumers.
* Measuring only HTTP success while database load or authorization drifts.
* Keeping the old route forever because retirement was not part of the charter.

## Senior Engineer Thinking

The senior question is not “how can we get the new application in front of traffic?” It is “which capability can change authority independently, what compatibility must hold during coexistence, and what evidence allows us to retire the old path?”

The Strangler Pattern succeeds when each slice has a bounded contract, one clear authority, controlled effects, observable routing, and a recovery story. It is a method for reducing blast radius and making ownership transitions explicit, not a guarantee that a rewrite is simpler.

## Exercises

1. Write a slice charter for one legacy endpoint, including all web, CLI, cron, queue, and admin entry points.
2. Design a read-only shadow comparison. List every field and side effect that must be compared or prohibited.
3. Build a write-authority timeline for an order workflow. Include retries, old workers, schema compatibility, and external effects.
4. Use `RouteDecision` to define a deterministic rollout policy with a fail-safe default and observable policy version.
5. Create retirement criteria for a legacy route and identify evidence that proves no recovery or repair path still depends on it.

## Review Questions

* What does the Strangler Pattern move: code, traffic, or ownership?
* Why is a route or directory not automatically an isolated capability?
* Which contracts must a migration boundary preserve besides the response body?
* Why are read shadows safer than dual writes?
* What does one authoritative writer mean during coexistence?
* How can a new path read old data safely?
* Why is routing a data and rollback decision?
* What observations should distinguish old and new paths?
* What makes retirement part of the migration rather than cleanup?
* When should a team choose forward recovery instead of code rollback?

## Summary

The Strangler Pattern migrates a legacy system through bounded capability and ownership transitions. Choose a slice by behavior and data boundary, preserve request and security semantics, compare read paths safely, keep one authoritative writer, define compatibility for shared state and messages, route deterministically, stage exposure with evidence, treat rollback as a data decision, and retire the old path deliberately. A proxy alone is not a migration plan; each slice needs authority, observability, reconciliation, and recovery.

## References

- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 249 — Distributed Locks](../16-distributed-systems/249-distributed-locks.md)
- [Chapter 264 — Deployment](../17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 269 — Incident Response](../17-production-engineering/269-incident-response.md)
- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](./271-legacy-architecture.md)
- [Chapter 272 — Characterization Tests](./272-characterization-tests.md)
- [Chapter 273 — Safe Refactoring](./273-safe-refactoring.md)
- [Chapter 275 — Branch by Abstraction](./275-branch-by-abstraction.md)
- [Chapter 277 — Database Migration](./277-database-migration.md)
