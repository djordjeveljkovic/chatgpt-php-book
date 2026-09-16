---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 160
title: Integration Tests
slug: integration-tests
status: complete
summary: ../../_ai/chapter-summaries/160-integration-tests-summary.md
---

# Chapter 160 — Integration Tests

## Why This Matters

An integration test verifies that real components collaborate correctly across a boundary: PHP and a database, an application and a filesystem, a serializer and a message broker, or an HTTP client and a provider protocol. Unit tests can prove a query builder constructs a string; an integration test can prove the real database accepts it, enforces constraints, and returns the expected types.

Integration tests are slower and require more setup, so they should target important boundaries and failure modes. They are evidence about actual configuration and adapters, not a replacement for unit tests.

## Choose the Boundary

Name the integration contract before writing the test. Examples include:

* a repository persists and retrieves a domain object;
* a migration creates the indexes and constraints the application requires;
* a serializer round-trips a message schema;
* an HTTP adapter maps timeout and error responses correctly;
* a filesystem adapter rejects an unsafe path and stores bytes atomically.

Use the real component at the boundary. A fake database cannot prove SQL syntax, transaction isolation, collation, indexes, or foreign keys. A mocked HTTP client cannot prove TLS configuration, request encoding, timeout settings, or provider response parsing.

## Database Example

A repository test should provision a known schema and isolate each test's data. A transaction rollback can be effective when the code under test uses the same connection and does not commit independently; otherwise use a temporary schema, unique test identifiers, or cleanup with verified ownership.

~~~php
<?php

declare(strict_types=1);

final class UserRepository
{
    public function __construct(private PDO $db)
    {
    }

    public function create(string $email): int
    {
        $statement = $this->db->prepare(
            'INSERT INTO users (email) VALUES (:email)',
        );
        $statement->execute(['email' => $email]);

        return (int) $this->db->lastInsertId();
    }

    public function findByEmail(string $email): ?array
    {
        $statement = $this->db->prepare(
            'SELECT id, email FROM users WHERE email = :email',
        );
        $statement->execute(['email' => $email]);

        $row = $statement->fetch(PDO::FETCH_ASSOC);

        return $row === false ? null : $row;
    }
}
~~~

The test should use the same production database engine and relevant SQL mode. Assert the behavior the application depends on: uniqueness, null handling, transaction rollback, timezone representation, and type conversion. Do not assume an in-memory SQLite database reproduces PostgreSQL or MySQL semantics.

## Fixtures and Isolation

Use migrations or schema snapshots to create a reproducible database. Keep fixtures minimal and explicit. A factory can generate valid defaults, but each test should own its identifiers and avoid relying on global row order. Parallel workers need separate schemas, databases, prefixes, or transaction strategies.

Cleanup is part of the test design. A failed test must not leave data that changes the next run. Verify that cleanup targets only test-owned records; a broad delete against a shared environment is a production incident waiting to happen. Never point destructive integration setup at production credentials.

## Transactions and Concurrency

Transactions provide a useful isolation boundary, but the test must match the application's connection behavior. If a queue worker or second connection participates, the outer test transaction may hide committed data. Use two connections when testing visibility, locks, or deadlocks, coordinate them with barriers, and make timing bounded.

Test constraints under concurrent operations when a race matters. For example, two inserts for the same idempotency key should result in one successful claim and one handled uniqueness conflict. A single-threaded test cannot prove this property.

## External Services

For a third-party provider, decide whether the integration test calls a sandbox, a local emulator, or a contract-controlled fake. A sandbox can reveal protocol and environment issues but may be slow, rate-limited, or mutable. A local fake gives deterministic failures but cannot prove the provider's real behavior. Keep credentials and personal data out of test traffic.

Test timeouts, malformed responses, non-2xx status codes, invalid signatures, rate limits, retries, and partial success. Make the adapter's error taxonomy stable so the domain service can choose whether to retry, compensate, or fail.

## Observability and Diagnostics

Integration failures should preserve the useful context without leaking secrets: database engine and migration version, sanitized request metadata, correlation ID, fixture identifier, and provider status. Redact authorization headers, cookies, tokens, and personal data. Capture logs and traces on failure, not arbitrary production payloads.

A test that fails only after several minutes is expensive to diagnose. Add health checks, readiness waits with deadlines, and clear setup errors. A dependency that never becomes ready is a failed environment, not a reason to wait forever.

## Failure and Threat Analysis

* **Environment drift:** the test database differs from production. Use the production engine, migrations, and relevant configuration.
* **Leaked credentials:** test logs or fixtures contain secrets. Use dedicated short-lived credentials and redaction.
* **Shared state:** tests pass alone but fail in a suite. Isolate schemas, identifiers, files, and queues.
* **False transaction isolation:** an outer transaction hides another connection's behavior. Test with the same connection topology as production.
* **Unbounded waits:** a container or provider never becomes ready. Use deadlines and actionable diagnostics.
* **Destructive cleanup:** a broad reset targets the wrong database. Require an explicit test marker and verify ownership.
* **Brittle provider dependency:** a live sandbox changes behavior. Keep deterministic adapter tests and schedule sandbox checks separately.

## Running Integration Tests

Tag or group integration tests and run them in CI with provisioned dependencies. Cache immutable images or build artifacts, not mutable database state. Run migrations from a clean database, then run the tests and collect logs when setup or teardown fails.

Make the suite reproducible from a documented command. If parallel execution is enabled, allocate isolated resources and ensure random test order does not alter results. Track duration and failure rate; a slow suite that developers never run is weak feedback.

## Exercises

1. Write a repository integration test using a real database engine. Cover creation, lookup, unique email failure, and rollback.
2. Compare transaction rollback, temporary schema, and cleanup strategies for your application. Choose one for parallel CI.
3. Build an HTTP provider fake that can return timeout, malformed JSON, invalid signature, and rate-limit responses. Map each to a domain outcome.
4. Run a two-connection test for a uniqueness or locking invariant and identify which isolation behavior it proves.

## Review Questions

1. What evidence does an integration test provide that a unit test cannot?
2. Why can an in-memory database give false confidence?
3. When does an outer test transaction hide production behavior?
4. How should integration fixtures be isolated under parallel execution?
5. Which provider failures must an adapter test cover?
6. Why should readiness waits have deadlines?

## Summary

Integration tests prove real collaboration across database, filesystem, queue, serialization, and provider boundaries. Use production-like engines and configuration, isolate fixtures and resources, test constraints and failure mapping, bound setup waits, and redact diagnostics. Keep fast unit tests for local decisions and use integration tests where wiring and external behavior are the risk.

## References

- [PHPUnit Documentation](https://docs.phpunit.de/)
- [PHP PDO](https://www.php.net/manual/en/book.pdo.php)
- [PHP PDO Transactions](https://www.php.net/manual/en/pdo.transactions.php)
- [Martin Fowler: IntegrationTest](https://martinfowler.com/bliki/IntegrationTest.html)
- [OWASP Database Security Testing](https://owasp.org/www-project-web-security-testing-guide/)

