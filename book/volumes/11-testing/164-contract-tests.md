---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 164
title: Contract Tests
slug: contract-tests
status: complete
summary: ../../_ai/chapter-summaries/164-contract-tests-summary.md
---

# Chapter 164 — Contract Tests

## Why This Matters

A contract test checks an agreement between independently deployed components. An API consumer depends on paths, methods, headers, status codes, fields, and behavior; an event consumer depends on event types, versions, ordering, and required data. A provider can pass its own tests while still breaking a client whose assumptions were never visible to the provider.

Contract tests make those assumptions executable. They complement API and end-to-end tests: an API test checks one application boundary, while a contract test checks compatibility between a producer and a named consumer.

## Consumer and Provider

The consumer owns the expectations it needs. It should express only fields and behavior it actually uses, rather than copying an entire provider schema. The provider verifies that it can satisfy every published consumer contract. This consumer-driven approach makes unused provider flexibility visible and avoids turning incidental implementation fields into promises.

A contract may cover:

- request method, path, query, headers, and body;
- response status, headers, and required body fields;
- authorization and error behavior;
- pagination, conditional requests, idempotency, and rate limits;
- event type, version, required fields, and delivery semantics;
- backward compatibility during a migration window.

A schema alone does not capture behavior. “Returns an order” is incomplete if the client relies on a 404, a stable money representation, or a cursor that remains valid for a defined period.

## A Focused Contract

Keep a contract small and explicit. This example describes a consumer that needs an order identifier and total, while allowing unrelated response fields:

~~~json
{
  "request": {
    "method": "GET",
    "path": "/v1/orders/order-42",
    "headers": {
      "Accept": "application/json"
    }
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "id": "order-42",
      "total": {
        "amount": 1999,
        "currency": "EUR"
      }
    }
  }
}
~~~

The matching rule matters. A JSON object with extra response fields can satisfy this contract if extras are intentionally allowed; a missing required field or changed type cannot. For arrays, specify ordering only when the consumer relies on it. For timestamps, define format, timezone, and precision instead of matching a string by accident.

A PHP contract assertion can encode the stable subset:

~~~php
<?php

declare(strict_types=1);

/** @param array<string, mixed> $body */
function assertOrderContract(array $body): void
{
    assert($body['id'] === 'order-42');
    assert(is_array($body['total'] ?? null));
    assert($body['total']['amount'] === 1999);
    assert($body['total']['currency'] === 'EUR');
}
~~~

A real suite should use an assertion library that reports field paths. The point is to avoid exact snapshots that fail whenever the provider adds an unrelated optional field.

## Provider Verification and States

The provider must set up a known state for each interaction. A “paid order exists” state should create an order with stable identifiers and permissions, or use a provider-state endpoint available only in the test environment. It should not depend on yesterday's shared database row.

Provider verification then runs the request through the real route, middleware, serialization, and persistence boundary. Verify authentication and authorization context as well as the response. Keep provider-state setup outside production code or protect it with test-environment controls; never expose a fixture endpoint on a production listener.

When a contract expects an error, verify the error status and stable problem fields. A provider that changes a 403 to a 404 may preserve an implementation detail while breaking a consumer's user flow or security policy.

## Events and Webhooks

For asynchronous contracts, specify event type, version, required envelope fields, payload shape, delivery ID, duplicate behavior, and ordering or replay expectations. Consumers should tolerate unknown optional fields and repeated event IDs when the protocol says delivery is at least once.

A provider can publish a fixture event to a consumer verifier, but the test must also cover signing, timestamp tolerance, and an invalid signature at the webhook boundary. Contract compatibility does not prove that the event was delivered, retried, or deduplicated correctly; those belong to integration and delivery tests.

## Compatibility and Versioning

Classify changes before merging:

- Adding an optional response field is usually compatible.
- Removing or renaming a required field is breaking.
- Adding an enum value can break clients that reject unknown values.
- Tightening validation or changing default pagination can be breaking.
- A new required request field is breaking unless the negotiated version changes.
- A changed authorization or retry policy can break behavior without changing JSON.

Run old consumer contracts against a new provider before deployment. Keep contracts for supported clients and event versions, and retire them only after the support window and traffic evidence allow it. Contract tests cannot replace a migration plan when both provider versions run concurrently.

## Tooling and CI

OpenAPI can define a broad HTTP schema and generate validation or client tooling. Pact and similar tools can record consumer expectations and publish provider verification results. Select a tool that matches the transport and ownership model; an oversized generated schema is not automatically a useful contract.

Store contracts with an owner, provider, consumer, version, and support window. In CI, publish the consumer contract, let the provider verify it against the candidate build, and block release when a supported contract fails. A matrix can become expensive, so group equivalent clients and use a compatibility policy for each version.

Do not let a contract broker become the only source of truth. Review meaningful contract changes, protect credentials, redact personal data in recorded examples, and expire contracts that have no active consumer.

## Failure Analysis

A contract failure can indicate an intentional breaking change, an unrecorded consumer assumption, wrong fixture state, authentication setup drift, or an actual provider regression. Report the interaction, consumer, provider build, request shape, response status, and field path. Do not weaken the matcher until the compatibility decision is understood.

If a provider must make a breaking change, publish a new version or migration path, run both contracts during the window, and monitor client adoption. If a consumer's expectation is wrong, change the contract with review and update the consumer behavior rather than hiding the mismatch.

## Common Mistakes

- Treating a generated schema snapshot as the complete behavioral contract.
- Making every provider field required and blocking compatible evolution.
- Using shared mutable provider fixtures.
- Verifying only successful responses and ignoring authorization or errors.
- Publishing secrets or personal data inside recorded interactions.
- Removing contracts without evidence that the consumer is gone.
- Running contracts only after deployment.

## Senior Engineer Thinking

A contract is a negotiated boundary with an owner and a lifetime. Keep it minimal, test the behavior clients use, verify provider state through real routes, and run compatibility checks before release. Separate schema compatibility from delivery guarantees, authorization, and operational behavior so each concern has a test that can observe it.

## Exercises

1. Write a consumer contract for a paginated order list. Decide which fields, ordering, cursor, and error cases are required.
2. Add a provider state for an unauthorized tenant and verify the expected status and problem shape.
3. Classify five proposed API changes as compatible or breaking, then choose an evolution strategy.
4. Design an event contract for at-least-once webhook delivery with duplicate IDs and a versioned payload.

## Review Questions

1. What does the consumer own in a consumer-driven contract?
2. Why should extra optional response fields usually be allowed?
3. Why are provider states necessary?
4. Which behavioral changes can break a client without changing the JSON schema?
5. What does an API contract fail to prove about delivery or external side effects?

## Summary

Contract tests make compatibility between an independent consumer and provider executable. Keep expectations minimal, verify real provider routes and states, include errors and authorization, version asynchronous events, classify changes before release, and protect recorded data. Use schema tools and consumer-driven tools as evidence within an owned compatibility policy.

## References

- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Pact documentation](https://docs.pact.io/)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [JSON Schema specification](https://json-schema.org/specification)
- [PHPUnit documentation](https://docs.phpunit.de/)

