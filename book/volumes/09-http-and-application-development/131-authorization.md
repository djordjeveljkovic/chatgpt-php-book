---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 131
title: Authorization
slug: authorization
status: complete
summary: ../../_ai/chapter-summaries/131-authorization-summary.md
---

# Chapter 131 — Authorization

## Why This Matters

Authentication answers who is making a request. Authorization answers whether that subject may perform an action on a resource in a context. An authenticated user must not automatically be able to read every object, change every field, or cross every tenant boundary.

Authorization bugs are usually ordinary application bugs with security consequences: an unscoped lookup, a missing queue check, a cached decision reused for another user, or a state change that races the policy check. Make the subject, action, resource, and context explicit and enforce the rule at the operation boundary.

## Deny by Default

Start with a policy that denies access unless a specific rule grants it. Pass a trusted, typed subject and the actual resource into the policy:

```php
<?php

declare(strict_types=1);

final readonly class User
{
    public function __construct(public int $id, public string $role) {}
}

final readonly class Invoice
{
    public function __construct(
        public int $id,
        public int $accountId,
        public string $state,
    ) {}
}

function mayViewInvoice(User $user, Invoice $invoice): bool
{
    return $user->role === 'support' || $user->id === $invoice->accountId;
}
```

The support rule is deliberately small; a real support policy may require a tenant scope, a reason, a temporary elevation, and an audit event. Never infer a role from a request field or hidden form control. Load identity and memberships from trusted server-side state.

Return an unauthenticated response when no valid identity exists and a forbidden response when an identity lacks permission. Some applications return `404 Not Found` for an object outside the subject's visibility to reduce enumeration. Choose that policy consistently and do not confuse an unavailable database with a missing object.

## Roles, Permissions, and Relationships

Role-based access control (RBAC) maps users to roles and roles to permissions. It works well for stable capabilities such as `billing.read`. Attribute-based access control (ABAC) considers properties such as department, region, time, or sensitivity. Relationship-based access control (ReBAC) asks whether the subject is an owner, editor, or member of the resource.

Many systems combine them: a role permits an action while a relationship or tenant membership scopes the resource. Avoid a single `isAdmin` flag when the real policy has independent dimensions. Name permissions around actions and resources, and make exceptional grants auditable and expirable.

## Object Checks and Query Scoping

An insecure direct object reference (IDOR) occurs when a client changes `/invoices/41` to `/invoices/42` and the handler loads 42 without checking its account. Check the concrete object before every protected operation:

```php
$invoice = $invoices->find($request->integer('id'));

if ($invoice === null || !mayViewInvoice($currentUser, $invoice)) {
    throw new ForbiddenOrNotFound();
}
```

Scope list and update queries as well. Filtering thousands of rows after loading is slower and can leak counts or timing. Make the repository carry the boundary:

```php
/** @return list<Invoice> */
function findVisibleInvoices(User $user): array
{
    return $user->role === 'support'
        ? $db->fetchInvoicesForSupport()
        : $db->fetchInvoicesForAccount($user->id);
}
```

The query is defense in depth; policy code is still needed for transitions and side effects. Map request fields to an allow-list so a caller cannot mass-assign `ownerId`, `role`, or `state`.

## State Changes and Tenants

A user may view an invoice but not issue a refund. A manager may approve only while the request is pending. Encode action and current state, then enforce the precondition in the same transaction as the mutation:

```php
function mayApprove(User $user, Invoice $invoice): bool
{
    return $user->role === 'billing_manager'
        && $invoice->state === 'pending';
}
```

Without a transaction, two requests can both pass an in-memory check and both perform a one-time action. Use a row lock, conditional update, or version check where a concurrent change matters.

In a multi-tenant system, tenant ID is a security boundary. Derive the active tenant from server-side membership and an explicit host or route mapping; do not trust a client-supplied tenant ID alone. Preserve the boundary in every query, unique constraint, cache key, background job, export, and event. Support tools need visible tenant context, a reason, and an audit record.

## Failure and Threat Analysis

* **Missing check:** a new route or queue consumer bypasses controller policy. Put authorization in reusable application services and test every entry point.
* **IDOR:** an unscoped identifier exposes another user's object. Scope queries and check the resource.
* **Mass assignment:** request binding lets a user set ownership, role, or approval state. Map an allow-list.
* **Stale cache:** a role change remains effective after a decision is cached. Include all decision inputs and invalidate or version policy data.
* **TOCTOU:** a resource changes between check and write. Use a transaction, lock, conditional update, or version.
* **Fail-open dependency:** an unavailable policy service must not silently grant access.
* **Information leakage:** status codes, counts, exports, or timing reveal resources. Define visibility for the whole endpoint.

Record subject, action, safe resource identifier, tenant, outcome, policy version, and correlation ID according to privacy requirements. Do not log credentials or sensitive document contents.

## Testing Authorization

Test a matrix of subjects, actions, resources, states, tenants, and expected outcomes. Include anonymous users, owners, peers, managers, support users, disabled users, and unknown resources. Integration tests must exercise real route binding, repository queries, queue consumers, exports, and mutation transactions. Add a regression test for every authorization bug and a concurrency test for one-time transitions.

## Exercises

1. Implement a policy for a multi-tenant document with owner, editor, viewer, and support relationships. Write a decision table first.
2. Refactor an ID lookup so ordinary callers can fetch only resources in their tenant. Add an IDOR regression test.
3. Model an approval transition with a conditional database update. Test two concurrent approvals.
4. Design a support-access audit event that helps investigation without storing document contents or credentials.

## Review Questions

1. How does authorization differ from authentication?
2. Why is a client-supplied object ID not an authorization decision?
3. When is RBAC insufficient by itself?
4. Why should list queries be scoped instead of filtered after loading?
5. What is the authorization risk of mass assignment?
6. How can a race occur between a check and a state change?
7. Which inputs belong in an authorization-cache key?

## Summary

Authorization is a deny-by-default decision about a subject, action, resource, and context. Keep policies explicit, scope database queries, protect state transitions with the transaction that enforces them, and treat tenants, caches, queues, exports, and support tools as authorization surfaces. Test both the decision matrix and every entry point so new paths cannot silently bypass the rule.

## References

- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [OWASP Insecure Direct Object Reference Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- [OWASP Multi-Tenant Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Application_Security_Cheat_Sheet.html)
- [OWASP Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)
