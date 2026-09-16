---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 154
title: Authorization
slug: authorization
status: complete
summary: ../../_ai/chapter-summaries/154-authorization-summary.md
---

# Chapter 154 — Authorization

## Why This Matters

Authentication establishes an identity. Authorization decides whether that subject may perform a particular action on a particular resource in a particular context. A valid login must not become permission to read every record, cross a tenant boundary, change ownership, or approve a state transition.

Authorization failures are ordinary application failures with security consequences. They arise from an unscoped query, a hidden form field treated as trusted, a cached role that outlives revocation, a queue message lacking tenant context, or a check that races the write. Make the policy explicit and enforce it at every operation boundary.

## A Decision Has Four Inputs

Model a decision as subject, action, resource, and context. Context can include tenant membership, resource state, purpose, time, device assurance, or a support ticket. Deny by default when a required input is absent.

~~~php
<?php

declare(strict_types=1);

final readonly class Actor
{
    public function __construct(
        public int $id,
        public int $tenantId,
        public string $role,
    ) {
    }
}

final readonly class Document
{
    public function __construct(
        public int $id,
        public int $tenantId,
        public int $ownerId,
        public string $state,
    ) {
    }
}

function mayEdit(Actor $actor, Document $document): bool
{
    if ($actor->tenantId !== $document->tenantId) {
        return false;
    }

    return $actor->id === $document->ownerId
        || in_array($actor->role, ['editor', 'admin'], true);
}
~~~

The example is deliberately small. A production policy may require an active membership, a sensitivity check, an approval state, or an audit event. Keep identity and membership server-side; a request field named role or tenant_id is input, not proof.

## RBAC, ABAC, and Relationships

Role-based access control maps roles to capabilities such as documents.read or billing.refund. Attribute-based access control evaluates attributes such as department, region, classification, or time. Relationship-based access control asks whether the subject is an owner, editor, member, or delegate of the resource.

Systems commonly combine them. A role may grant the ability to edit while a relationship and tenant scope decide which documents are visible. Avoid an is_admin flag when the real policy has independent dimensions. Name permissions around actions and resources, keep exceptional grants narrow and expiring, and audit administrative changes.

## Query Scoping and Object Checks

An insecure direct object reference occurs when changing an identifier from one resource to another returns a record without checking visibility. Scope queries in the data layer and check the concrete object before a protected action:

~~~php
<?php

function findVisibleDocument(
    Actor $actor,
    int $documentId,
    DocumentRepository $documents,
): Document
{
    $document = $documents->findInTenant($documentId, $actor->tenantId);

    if ($document === null || !mayEdit($actor, $document)) {
        throw new RuntimeException('Resource is unavailable');
    }

    return $document;
}
~~~

The repository scope reduces accidental leakage through lists, counts, timing, and exports. The policy still matters for action-specific rules. Do not load every tenant's records and filter in PHP; it increases cost and creates more paths where a forgotten filter can leak data.

Map writable fields explicitly. A generic hydrator that accepts owner_id, role, state, or tenant_id creates mass-assignment risk. Keep commands separate from persistence entities so an API request cannot silently set fields intended only for a domain transition.

## State, Concurrency, and Transactions

Permission to view an object does not imply permission to delete, refund, publish, or approve it. A policy should consider the current state and the actor's action. The state check and mutation must be atomic:

~~~php
<?php

function approveDocument(PDO $db, Actor $actor, int $documentId): void
{
    $statement = $db->prepare(
        'UPDATE documents
         SET state = :approved
         WHERE id = :id
           AND tenant_id = :tenant
           AND state = :pending
           AND reviewer_id = :reviewer',
    );

    $statement->execute([
        'approved' => 'approved',
        'id' => $documentId,
        'tenant' => $actor->tenantId,
        'pending' => 'pending',
        'reviewer' => $actor->id,
    ]);

    if ($statement->rowCount() !== 1) {
        throw new RuntimeException('Approval is unavailable');
    }
}
~~~

In practice, determine reviewer eligibility before the update and include the resulting policy constraint in the transaction or conditional statement. A row lock, optimistic version, or conditional update prevents two requests from both consuming a one-time transition. Database constraints remain necessary for invariants such as unique membership.

## Tenants, Caches, and Background Work

A tenant is a security boundary. Derive active tenant context from server-side membership and a trusted host or route mapping. Include tenant scope in every query, unique constraint, cache key, object-store path, export, event, and job. A support tool should display tenant context, require a reason, and produce an audit event.

Authorization caches must include every decision input or use a policy version. Invalidate or version them when memberships, roles, resource ownership, or sensitivity changes. Never cache a decision without considering subject, action, resource, tenant, and relevant state.

Queue consumers must authorize the command they receive. A job created by an authorized request can become dangerous when replayed after membership is revoked or when a producer bug omits a tenant. Carry the subject and intended operation, then re-check current policy before side effects.

## Failure and Threat Analysis

* **IDOR:** an unscoped identifier exposes another tenant's object. Scope the repository and check the object.
* **Mass assignment:** request binding sets ownership or approval state. Map an allow-list.
* **TOCTOU:** state changes between a check and write. Use a transaction, lock, version, or conditional update.
* **Stale cache:** a revoked role still grants access. Version or invalidate policy data.
* **Fail-open service:** an unavailable policy dependency grants access. Fail closed and provide a retry path.
* **Confused deputy:** a privileged worker follows an unscoped user instruction. Carry subject, tenant, and action.
* **Information leakage:** status codes, counts, exports, or timing reveal hidden resources. Define the visibility policy for every endpoint.

Record safe subject, action, resource class and identifier, tenant, outcome, policy version, and correlation ID. Do not record credentials or protected document contents.

## Testing Authorization

Build a decision matrix for subjects, actions, resources, states, tenants, and expected outcomes. Test anonymous, disabled, owner, peer, manager, support, wrong-tenant, missing, and stale-membership cases. Integration tests must cover list, get, update, delete, export, queue, CLI, and administrative paths.

Add regression tests for every authorization defect. Include concurrency tests for one-time transitions, cache invalidation tests for role changes, and property or state-transition tests for resources with many legal and illegal states. Use the real database engine for scope and constraint tests.

## Exercises

1. Implement a policy for a multi-tenant document with owner, editor, viewer, and support relationships. Write the decision table first.
2. Refactor a document lookup to enforce tenant scope and add a regression test for an IDOR attempt.
3. Model an approval transition using a conditional database update and test two concurrent approvals.
4. Design a support-access audit event that records reason and scope without storing document contents.

## Review Questions

1. How do subject, action, resource, and context form an authorization decision?
2. When is RBAC insufficient by itself?
3. Why should list queries be scoped in the database?
4. What is the risk of mass assignment?
5. How can a time-of-check/time-of-use race bypass a policy?
6. Which inputs must be included in an authorization-cache key?
7. Why must a queue consumer re-check authorization?

## Summary

Authorization is a deny-by-default decision about a subject, action, resource, and context. Combine roles with relationships and attributes where needed, scope data queries, allow-list writable fields, and enforce state changes atomically. Treat tenants, caches, queues, exports, and support tools as authorization surfaces, and test every entry point and concurrency boundary.

## References

- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP Insecure Direct Object Reference Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- [OWASP Multi-Tenant Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Application_Security_Cheat_Sheet.html)
- [OWASP Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

