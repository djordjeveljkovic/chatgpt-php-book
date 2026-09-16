---
book: The Complete Modern PHP Engineering Book
volume: 18
volume_title: LEGACY PHP
chapter: 276
title: Framework Migration
slug: framework-migration
status: complete
summary: ../../_ai/chapter-summaries/276-framework-migration-summary.md
---

# Chapter 276 — Framework Migration

## Why This Matters

A framework migration changes the application’s lifecycle as well as its APIs. A legacy PHP page may depend on include order, superglobals, output buffering, a global database handle, and a particular exit path. A framework adds a front controller, request object, middleware, dependency container, response object, event hooks, and a different boot and termination sequence.

The risk is not that a framework is unfamiliar. The risk is that behavior moves across lifecycle boundaries without being named. Authentication may run later. A transaction may commit earlier. A session may use a different cookie or lock. A CLI command may load different configuration. A queue worker may retain objects that were request-local before.

Migrate responsibilities and contracts, not merely controllers. The goal is a stable boundary where legacy and framework paths can coexist, be observed, and be rolled back safely.

## Mental Model

Compare the old and new execution paths:

~~~text
legacy request:
server → index.php → includes → globals → page script → output/exit

framework request:
server → front controller → bootstrap → middleware → handler
         → application operation → response → termination hooks
~~~

The framework does not eliminate the old dependencies. It changes where they must be adapted. Map:

* entry and termination behavior;
* configuration and environment loading;
* authentication, authorization, and tenant context;
* database connection and transaction ownership;
* templates, escaping, redirects, and headers;
* session, cache, filesystem, and upload state;
* CLI, cron, queue, and worker lifecycles;
* logging, metrics, tracing, and error handling.

Chapter 209 explains what frameworks actually do; this chapter applies that model to legacy migration.

## Choose the Migration Shape

Possible shapes include:

| Shape | Legacy role | Framework role | Main risk |
| --- | --- | --- | --- |
| framework around legacy | existing scripts remain handlers | owns front controller and adapters | hidden bootstrap assumptions |
| legacy route beside framework | each path has one owner | new routes use framework lifecycle | inconsistent auth/session behavior |
| framework operation behind legacy page | old entry point remains | new operation owns one capability | transaction/effect drift |
| framework shell with legacy adapters | old infrastructure is wrapped | new application owns decisions | adapters become permanent |
| separate capability | old system serves remaining routes | framework app owns a slice | shared data and rollback |

Select by capability and ownership, not by how easy a directory move appears. Chapter 274 covers capability migration and Chapter 275 covers abstraction-based coexistence.

## Write a Lifecycle Charter

Before moving one path, record:

~~~text
capability: admin invoice lookup
old entry: /admin/invoice.php with bootstrap.php
new entry: framework route and controller
auth owner: existing authorization policy
data owner: legacy invoice reader
state: session and cache keys remain compatible
effects: none; read-only
parity evidence: status, headers, body, query count, authorization
rollback: route selector returns to legacy entry
~~~

The charter prevents a framework migration from quietly changing several contracts at once. For a write path, include transaction scope, operation identity, outbox behavior, and recovery owner.

## Front Controller and Routing

The front controller is a useful seam only when it preserves the request boundary. During coexistence, route selection should be explicit:

* authenticate and establish tenant context before the handler;
* preserve method, path, query, body, headers, cookies, and content negotiation;
* map legacy redirects and status codes deliberately;
* prevent a route from reaching both old and new writers;
* record path, implementation, policy version, and outcome;
* keep a safe default and a tested rollback switch.

Avoid routing solely by controller directory. A legacy URL may be called by a webhook, form, bookmark, CLI script, or integration with a non-browser client. Preserve its actual contract or publish an explicit compatibility change.

## Bootstrap and Configuration

Legacy bootstrap files often do several unrelated jobs: load environment variables, define constants, include helpers, open a database connection, configure error reporting, start a session, and set timezone or locale. A framework’s bootstrap may perform the same jobs in a different order and scope.

Inventory each side effect and classify it:

| Concern | Legacy mechanism | Migration decision |
| --- | --- | --- |
| configuration | constants, includes, server variables | typed framework configuration adapter |
| database | global handle and helper functions | one connection/transaction owner |
| errors | warnings, display, custom handlers | mapped exception and logging policy |
| sessions | direct `$_SESSION` access | compatible session/cookie boundary |
| helpers | globally included functions | explicit adapter or injected service |
| output | echo, buffering, `exit` | response object and termination contract |

Do not load both bootstraps indiscriminately. Double initialization can start sessions twice, register duplicate handlers, redefine constants, change error levels, or open competing connections. Compose a narrow compatibility bootstrap instead.

## Middleware and Authorization

Middleware changes order and scope. A legacy page may authenticate inside the script after parsing input; a framework may run authentication and authorization before the controller. That can be an improvement, but it is not behavior-neutral until the contract is checked.

For each middleware or hook, identify:

* what it reads and writes;
* whether it can short-circuit;
* which status, headers, or redirect it produces;
* whether it runs for web, CLI, and queue paths;
* which tenant, actor, and capability context it establishes;
* whether it changes session, cache, or transaction state.

Never assume that reaching a framework controller means authorization succeeded. Keep object-level authorization close to the capability and preserve fail-closed behavior. Chapter 154 covers authorization boundaries.

## A Framework Adapter Boundary

Keep framework request details out of the application operation:

~~~php
<?php

declare(strict_types=1);

final readonly class InvoiceLookupInput
{
    public function __construct(
        public string $invoiceId,
        public string $actorId,
        public string $tenantId,
    ) {
        if ($invoiceId === '' || $actorId === '' || $tenantId === '') {
            throw new InvalidArgumentException('Lookup input is incomplete');
        }
    }
}

interface InvoiceLookup
{
    public function find(InvoiceLookupInput $input): ?array;
}

final class FrameworkInvoiceController
{
    public function __construct(private InvoiceLookup $lookup)
    {
    }

    /** @param array<string, string> $request */
    public function __invoke(array $request, string $actorId, string $tenantId): array
    {
        $input = new InvoiceLookupInput(
            invoiceId: $request['invoice_id'] ?? '',
            actorId: $actorId,
            tenantId: $tenantId,
        );

        $invoice = $this->lookup->find($input);

        return $invoice === null
            ? ['status' => 404, 'body' => ['error' => 'not_found']]
            : ['status' => 200, 'body' => $invoice];
    }
}
~~~

The controller translates framework request data into an application input and translates the result into a response shape. It does not open a database transaction, read a global, or decide whether a tenant can see an invoice. The legacy adapter can implement `InvoiceLookup` while the framework lifecycle is introduced.

The example uses arrays at the HTTP boundary intentionally. A production framework response object can wrap the returned status and body; the important contract is that HTTP details stop at the controller.

## Templates and Responses

Legacy output may mix HTML, SQL results, warnings, redirects, and partial fragments. A framework template engine can alter escaping, whitespace, missing-variable behavior, layout inheritance, and response buffering.

Characterize:

* content type and charset;
* escaping context and raw HTML exceptions;
* status and redirect codes;
* headers, cookies, and cache directives;
* empty, missing, null, and invalid values;
* partial output when an error occurs.

Do not globally disable framework escaping to preserve one unsafe template. Adapt the template data and fix security defects as explicit changes. Keep an unsafe observed behavior visible rather than blessing it as a framework contract.

## Database, ORM, and Transactions

An ORM changes query generation, hydration, identity maps, lazy loading, and transaction conventions. It may also change result ordering, null handling, timestamps, relation loading, and query count.

Before using an ORM for a legacy path, characterize:

* exact query and result grain;
* authorization predicates and tenant filters;
* transaction owner and lock duration;
* lazy/eager loading and query count;
* default scopes, soft deletes, casts, and events;
* trigger, audit, outbox, and cache effects.

Keep the first framework path behind a repository or application port. Do not allow an ORM model to become the new shared module before data ownership and migration authority are explicit. Chapter 277 owns schema and data migration mechanics.

## Sessions, Files, and Caches

Framework services can use different session serialization, cookie names, filesystem roots, cache prefixes, and locking policies. During coexistence, define compatibility or isolate namespaces.

Never share a cache key between old and new code if the value shape or authorization scope differs. Never let a new worker reuse a legacy session or serialized payload without a documented parser and security review. Temporary files need owner, permissions, naming, cleanup, and concurrent-job behavior.

## CLI, Cron, and Queues

Do not migrate only the browser path. Framework commands and workers have their own bootstrap and lifecycle:

| Process | Migration check |
| --- | --- |
| CLI | binary, `php.ini`, cwd, user, exit code, signals |
| cron | schedule, overlap, lock, environment, retries |
| queue | message version, acknowledgment, visibility, worker reset |
| admin script | authorization, audit, input parsing, recovery |

A framework worker may be long-lived where a legacy command was short-lived. Reset process-local state, recycle workers after a defined bound, and drain old consumers before changing message or database contracts. Chapters 215 and 221 cover framework queue and Messenger behavior.

## Observability and Rollback

Record implementation, route, framework version, policy version, correlation ID, operation ID, outcome, latency, query count, and effect state. Keep fields bounded and redact request bodies, credentials, and personal data.

Rollback can mean route rollback, adapter rollback, framework artifact rollback, or data/effect recovery. Restoring the legacy route cannot undo a committed write, sent email, consumed message, or changed session. Before a write cutover, verify old readers, workers, repair scripts, and recovery tools can interpret new state.

## Rollout Sequence

~~~text
characterize legacy entry point
          ↓
add framework adapter with old authority
          ↓
route read-only or internal traffic
          ↓
compare lifecycle and output evidence
          ↓
move one owner or effect deliberately
          ↓
drain old workers and callers
          ↓
remove compatibility bootstrap
~~~

Stop on authorization differences, changed effect count, incompatible session/message state, unexpected query load, or unknown completion. A framework migration is complete only when the old entry points, bootstraps, workers, and runbooks have an explicit retirement decision.

## Common Mistakes

* Migrating controllers while leaving cron, queue, and admin paths on old assumptions.
* Loading legacy and framework bootstraps together without ordering or ownership.
* Treating middleware order as an implementation detail.
* Assuming a framework controller already has authorization and tenant context.
* Using an ORM without characterizing query shape, locks, nulls, and side effects.
* Sharing sessions or caches across incompatible serializers or value shapes.
* Globally disabling escaping to preserve one legacy template.
* Letting a framework worker retain request state across jobs.
* Rolling back framework code after irreversible data or provider effects.
* Installing a permanent compatibility bootstrap with no owner or expiry.

## Senior Engineer Thinking

The senior question is not “which framework should replace the old code?” It is “which lifecycle contract is changing, which boundary can translate it, and how will web, CLI, workers, data, and effects remain observable during coexistence?”

A framework is a set of lifecycle and infrastructure decisions. Use it to make boundaries explicit, not to hide legacy behavior behind conventions. Migrate one capability, preserve security and data authority, and make every bootstrap, middleware, worker, and response difference reviewable.

## Exercises

1. Map one legacy page and its framework equivalent from entry to termination. Mark each lifecycle, auth, transaction, output, and effect difference.
2. Write a lifecycle charter for a read-only route, then add a write version with transaction, operation identity, and rollback fields.
3. Design a compatibility bootstrap that loads configuration and a legacy adapter without double-starting sessions or handlers.
4. Compare a legacy database helper with an ORM repository. List query, lock, result-shape, event, and cache behaviors to characterize.
5. Create a rollout and retirement plan covering HTTP, CLI, cron, queue, admin, sessions, caches, and recovery tools.

## Review Questions

* Why is a framework migration a lifecycle migration?
* Which legacy bootstrap side effects need explicit ownership?
* What must a front controller preserve during coexistence?
* Why can middleware order change authorization behavior?
* Which ORM behaviors require characterization beyond returned rows?
* How do session and cache compatibility affect a framework migration?
* Why must CLI, cron, and queue paths be migrated separately?
* What does a framework adapter own, and what should remain in the application operation?
* Which rollback actions cannot reverse data or external effects?
* What evidence allows a compatibility bootstrap to be retired?

## Summary

Framework migration changes request, bootstrap, middleware, response, database, session, worker, and termination lifecycles. Choose a capability boundary, write a lifecycle charter, preserve authentication and request semantics, adapt legacy bootstrap side effects deliberately, keep application operations independent of framework objects, characterize templates and ORM behavior, isolate sessions and caches, migrate CLI/cron/queue paths, observe rollout, and treat rollback as a data/effect decision. A framework helps only when it makes contracts and ownership clearer.

## References

- [Chapter 154 — Authorization](../10-security/154-authorization.md)
- [Chapter 209 — What Frameworks Actually Do](../14-laravel-and-symfony/209-what-frameworks-actually-do.md)
- [Chapter 210 — Laravel Overview](../14-laravel-and-symfony/210-laravel-overview.md)
- [Chapter 215 — Laravel Queues](../14-laravel-and-symfony/215-laravel-queues.md)
- [Chapter 219 — Symfony Dependency Injection](../14-laravel-and-symfony/219-symfony-dependency-injection.md)
- [Chapter 221 — Symfony Messenger](../14-laravel-and-symfony/221-symfony-messenger.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 264 — Deployment](../17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 270 — PHP 5 Codebases](./270-php-5-codebases.md)
- [Chapter 271 — Legacy Architecture](./271-legacy-architecture.md)
- [Chapter 272 — Characterization Tests](./272-characterization-tests.md)
- [Chapter 274 — Strangler Pattern](./274-strangler-pattern.md)
- [Chapter 275 — Branch by Abstraction](./275-branch-by-abstraction.md)
- [Chapter 277 — Database Migration](./277-database-migration.md)
