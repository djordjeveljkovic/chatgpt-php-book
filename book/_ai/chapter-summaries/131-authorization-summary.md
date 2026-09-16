# AI Summary — Chapter 131 — Authorization

- Status: complete
- Volume: Volume 9 — HTTP AND APPLICATION DEVELOPMENT
- Last updated: 2026-09-16

## Written material

Explains deny-by-default policies, roles and permissions, ABAC/ReBAC, object-level checks, query scoping, state transitions, tenant boundaries, support access, caching, threats, testing, exercises, review questions, and references.

## Concepts already explained

Authorization is a subject/action/resource/context decision; authentication is separate; query scope and transactional state checks defend against IDOR and races.

## Terminology established

RBAC, ABAC, ReBAC, IDOR, mass assignment, TOCTOU, tenant boundary, authorization cache.

## Examples used

Typed invoice policy, visible-invoice repository, conditional state policy, multi-tenant analysis.

## Cross-references

[Chapter 130](../../volumes/09-http-and-application-development/130-authentication.md) for identity; Volume VIII for transactions and concurrency.

## Open threads

Later security chapters can deepen authorization and multi-tenant controls.

## Exact next section

Chapter complete.

## Technical verification notes

PHP examples are syntax-checked; references point to OWASP authorization, IDOR, multi-tenant, and mass-assignment guidance.
