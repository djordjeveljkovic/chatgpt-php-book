---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 135
title: REST
slug: rest
status: complete
summary: ../../_ai/chapter-summaries/135-rest-summary.md
---

# Chapter 135 — REST

## Why This Matters

REST is often reduced to “use nouns in URLs and JSON responses.” The useful idea is broader: model resources and their representations over a uniform interface, use HTTP method and status semantics deliberately, and keep interactions stateless enough that requests can be routed and retried according to an explicit contract.

REST is a style, not a framework or a requirement that every operation be forced into CRUD. A command endpoint can be clearer than an artificial resource when the domain has an explicit action, but its HTTP semantics still need to be defined.

## Resource Representations

A resource is a conceptual piece of state; a representation is a transfer format for it. `/reservations/42` can return JSON today and a different representation under content negotiation without changing the resource identity. Do not expose a database table as the API model by accident.

```http
GET /reservations/42 HTTP/1.1
Accept: application/json
```

```json
{
  "id": "42",
  "court_id": "7",
  "starts_at": "2026-09-16T18:00:00Z",
  "ends_at": "2026-09-16T19:00:00Z",
  "status": "confirmed"
}
```

Use stable public identifiers and explicit timestamps. If a field is mutable or sensitive, decide whether it belongs in the representation and whether its absence differs from `null`.

## Method Semantics

`GET` retrieves a representation and should be safe; `HEAD` retrieves metadata; `POST` submits a subordinate resource or command; `PUT` replaces a resource at a known URI and is intended to be idempotent; `PATCH` applies a partial modification whose semantics must be documented; `DELETE` removes or transitions a resource. “Idempotent” means repeating the same request has the same intended effect, not that every response body is identical.

Use conditional requests for concurrent edits:

```http
PUT /documents/9 HTTP/1.1
If-Match: "version-12"
Content-Type: application/json

{"title":"Updated"}
```

If the representation changed, return `412 Precondition Failed` or the conflict policy documented by the API. This is the HTTP layer of an optimistic version check; the database must still enforce the update atomically.

## Collections and Relationships

Collection endpoints need explicit filtering, ordering, pagination, maximum page sizes, and link or cursor policy. Do not return an unbounded array. A nested URL can communicate scope, but deep nesting can make relationships difficult to evolve; `/users/7/reservations` and `/reservations?user_id=7` may both be reasonable depending on the contract.

Represent relationships with stable identifiers or links, and avoid silently embedding an unbounded graph. Clients should be able to request the fields or expansions they need under a documented cost limit.

## Caching and Hypermedia

HTTP caching uses method semantics and response headers. `ETag`, `Last-Modified`, `Cache-Control`, and conditional requests can reduce transfer and support conflict detection. A private authenticated representation must not be stored in a shared cache unless the directives and cache key guarantee isolation.

The original REST constraint includes hypermedia as the engine of application state, but many JSON APIs use only a subset of REST constraints. Be precise about the style being adopted. If clients receive action links, document their relation names and authorization-dependent presence; do not require clients to guess URL templates.

## Errors and Actions

Return a consistent problem representation for parse errors, validation failures, missing resources, conflicts, and rate limits. An action such as cancellation can be represented as `POST /reservations/42/cancel` when it has its own authorization, idempotency, and transition rules. The endpoint should return the resulting resource or an operation status, not a success string that leaves the client to infer state.

## Security and Concurrency

REST's stateless request model does not remove authentication, authorization, CSRF concerns for browser credentials, or replay risk. Authenticate every request according to the API's credential scheme, authorize the specific resource, validate content types and sizes, and use idempotency or conditional requests for retries and concurrent edits.

Do not treat a URI as a secret. Apply tenant scoping and object-level authorization at the service or repository boundary. Rate limit expensive collection and action endpoints, and set timeouts for downstream calls.

## Testing

Test method safety and idempotency, conditional requests, cache validators, content negotiation, collection pagination, action transitions, authorization by object, malformed representations, and problem details. Contract tests should assert semantics clients rely on, not private controller method names. Add property or state-transition tests for resources with many legal and illegal transitions.

## Common Mistakes

- Treating REST as URL naming alone.
- Using `POST` for every mutation without defining replay behavior.
- Calling a database row a public resource representation without a compatibility policy.
- Omitting conditional requests from concurrent edits.
- Returning unbounded collections.
- Assuming stateless requests are automatically secure or idempotent.

## Senior Engineer Thinking

Choose the REST constraints that solve the client's problem and document the subset. Preserve HTTP semantics where they help intermediaries and clients, model domain actions explicitly, and put concurrency and authorization guarantees below the transport layer as well. A good REST API makes state transitions and recovery behavior observable to an independent client.

## Exercises

1. Design a reservation resource with collection filtering, pagination, and a cancellation action.
2. Add `ETag`/`If-Match` handling to an update flow and map it to a database version column.
3. Write a method/status matrix for one API and identify which operations are safe or idempotent.

## Review Questions

1. What is the difference between a resource and its representation?
2. Why is `PUT` intended to be idempotent while `POST` often is not?
3. How do conditional HTTP requests interact with database optimistic locking?
4. Which REST constraints does your API actually adopt?

## Summary

REST is a set of architectural constraints around resources, representations, uniform HTTP semantics, stateless interactions, and cacheability. Use those semantics deliberately, define collection and action contracts, add conditional requests for concurrent edits, and keep authorization, idempotency, and domain invariants explicit below the HTTP layer.

## References

- [Roy Fielding: Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
