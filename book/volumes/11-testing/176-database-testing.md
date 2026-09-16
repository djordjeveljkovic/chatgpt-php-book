---
book: The Complete Modern PHP Engineering Book
volume: 11
volume_title: TESTING
chapter: 176
title: Database Testing
slug: database-testing
status: complete
summary: ../../_ai/chapter-summaries/176-database-testing-summary.md
---

# Chapter 176 — Database Testing

A database test verifies behavior at the boundary where PHP, SQL, a driver, and a real database meet. It can catch wrong SQL, missing constraints, transaction mistakes, collation differences, and migration errors that a mock or an in-memory array cannot see.

A good test suite uses several levels. Unit tests cover domain decisions quickly. Repository and migration tests use the production database engine in an isolated schema or container. A smaller number of end-to-end tests prove the application wiring. The database tests should be fast enough to run frequently and realistic enough to protect the invariants that matter.

## Why this matters

A repository can pass every mock-based test while selecting the wrong column, returning duplicate rows, relying on SQLite behavior that MySQL does not share, or committing half of a transaction. Constraints and isolation are database behavior, so they need a database to test.

Treat the database as part of the executable contract:

```text
migration → schema and constraints → SQL adapter → transaction boundary
    ↓              ↓                    ↓                ↓
  tested       tested against engine  tested with rows  tested under failure
```

A test should state which property it protects. “The query runs” is weaker than “a user cannot read another tenant's invoice” or “a duplicate idempotency key returns the original result.”

## Test against the production engine

SQLite is useful for a small, self-contained test, but it may differ from PostgreSQL or MySQL in types, locking, JSON functions, null ordering, collation, foreign-key defaults, and query plans. Do not use SQLite as a substitute for the production engine when those differences affect behavior.

Run repository tests against the same database family and a compatible major version as production. A disposable container, isolated CI service, or dedicated test database can provide this boundary. The test database must never contain production credentials or data.

Keep the connection factory explicit:

```php
<?php

declare(strict_types=1);

function testPdo(): PDO
{
    $dsn = getenv('TEST_DATABASE_DSN');
    $user = getenv('TEST_DATABASE_USER') ?: null;
    $password = getenv('TEST_DATABASE_PASSWORD') ?: null;

    if ($dsn === false || $dsn === '') {
        throw new RuntimeException('TEST_DATABASE_DSN is required');
    }

    return new PDO($dsn, $user, $password, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        PDO::ATTR_EMULATE_PREPARES => false,
    ]);
}
```

Failing when the test DSN is absent is safer than accidentally connecting to a developer or production database. In CI, provision a unique database or schema and destroy it after the job.

## Migrations are test input

Apply the same migrations used in deployment. A hand-written test schema can drift and allow queries that production cannot run. Test the migration from an empty database, and test upgrades from representative previous versions when backward compatibility matters.

A migration should create the constraints that enforce domain invariants:

```sql
CREATE TABLE idempotency_keys (
    tenant_id BIGINT NOT NULL,
    key_value VARCHAR(200) NOT NULL,
    result_json TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (tenant_id, key_value)
);
```

The composite key is part of the behavior. A test should prove that two requests with the same tenant and key cannot both create a record, while equal keys in different tenants are independent. Test foreign keys, unique constraints, check constraints, indexes used by critical queries, and nullable columns deliberately. A migration that runs successfully but omits a constraint is a failed migration for the domain.

## Repository integration tests

Use a repository implementation and real rows rather than mocking PDO. Keep each test focused on a behavior and create only the records it needs:

```php
final class IdempotencyRepository
{
    public function __construct(private PDO $pdo)
    {
    }

    public function save(int $tenantId, string $key, string $result, string $createdAt): void
    {
        $statement = $this->pdo->prepare(
            'INSERT INTO idempotency_keys
             (tenant_id, key_value, result_json, created_at)
             VALUES (:tenant_id, :key_value, :result_json, :created_at)'
        );
        $statement->execute([
            'tenant_id' => $tenantId,
            'key_value' => $key,
            'result_json' => $result,
            'created_at' => $createdAt,
        ]);
    }

    public function find(int $tenantId, string $key): ?array
    {
        $statement = $this->pdo->prepare(
            'SELECT tenant_id, key_value, result_json
             FROM idempotency_keys
             WHERE tenant_id = :tenant_id AND key_value = :key_value'
        );
        $statement->execute(['tenant_id' => $tenantId, 'key_value' => $key]);

        $row = $statement->fetch();

        return $row === false ? null : $row;
    }
}
```

The integration test can assert both returned data and persistence:

```php
public function testItStoresAndReadsAnIdempotencyResult(): void
{
    $pdo = testPdo();
    $repository = new IdempotencyRepository($pdo);

    $repository->save(7, 'request-1', '{"status":"accepted"}', '2026-09-16 12:00:00');

    self::assertSame(
        [
            'tenant_id' => 7,
            'key_value' => 'request-1',
            'result_json' => '{"status":"accepted"}',
        ],
        $repository->find(7, 'request-1'),
    );
}
```

The exact timestamp and JSON types depend on the database driver. Normalize values at the adapter boundary rather than making every test depend on driver-specific representations.

## Fixtures and isolation

Fixtures should be explicit, minimal, and owned by the test. A fixture builder can provide valid defaults while allowing a test to state the value it cares about:

```php
final class UserBuilder
{
    /** @return array{email: string, tenant_id: int, active: int} */
    public static function make(array $overrides = []): array
    {
        return array_replace([
            'email' => 'user-' . bin2hex(random_bytes(4)) . '@example.test',
            'tenant_id' => 7,
            'active' => 1,
        ], $overrides);
    }
}
```

Random identifiers avoid collisions, but they do not make assertions random. Keep values available to the test and use a deterministic clock when timestamps are part of the contract. Do not use a global “load every fixture” script; it hides relationships and makes tests order-dependent.

A transaction around each test can provide fast cleanup when the code under test uses the same connection:

```php
protected function setUp(): void
{
    parent::setUp();
    $this->pdo = testPdo();
    $this->pdo->beginTransaction();
}

protected function tearDown(): void
{
    if ($this->pdo->inTransaction()) {
        $this->pdo->rollBack();
    }
    parent::tearDown();
}
```

This pattern fails when the application opens another connection, dispatches a worker, uses a separate transaction, or commits through a connection the test does not control. In those cases, truncate or recreate an isolated schema, and test the actual transaction behavior explicitly. Never depend on cleanup that silently leaves rows for the next test.

## Transactions, constraints, and concurrency

Test rollback when a later operation fails. Assert that no partial row remains and that a retry has a defined result. Test uniqueness and foreign keys through the database, not only through pre-checks in PHP; two concurrent requests can pass the same pre-check.

Concurrency tests need independent connections and deliberate synchronization. A simple two-connection test can coordinate with a barrier or database lock, then assert the invariant after both operations finish. Use bounded timeouts so a deadlock does not hang the test suite. The test should reflect the production isolation level and retry policy; see [Chapter 114 — Transactions](../08-databases/114-transactions.md), [Chapter 115 — Isolation](../08-databases/115-isolation.md), and [Chapter 117 — Deadlocks](../08-databases/117-deadlocks.md).

Do not make every test a concurrency test. Select invariants that can actually race: inventory, uniqueness, idempotency, balance updates, and lease ownership. Record the database and isolation version used by CI because lock behavior can change across versions.

## Query assertions and test boundaries

Assert results and invariants before asserting implementation details. A query-count assertion can be useful for an N+1 regression, but it should not replace correctness. Snapshotting a whole row can make harmless columns break a test and can accidentally include secrets.

Keep SQL tests close to the adapter. Test HTTP authorization separately at the application boundary, then add at least one end-to-end test proving that a request cannot cross a tenant boundary. A repository test can prove that the tenant predicate exists; an endpoint test proves the authenticated subject is the value supplied to that predicate.

Use database logs or a query listener for diagnostics in CI, but do not print credentials or full sensitive parameters. Explain failures with the migration version, database engine, and test identifier.

## Testing and operations

A reliable CI database job should provision a clean engine, apply migrations, run schema and repository tests, run selected concurrency tests, and destroy the environment. Parallel workers need separate schemas or databases; otherwise one worker can truncate another worker's rows. Seed data must be versioned with the test code.

Monitor test duration and deadlocks. A slow suite may indicate missing indexes, unbounded fixture data, or leaked connections. Run backups and restore tests in the operational environment separately; a green repository test does not prove that a production backup can be restored.

## Exercises

1. Write a migration test that proves the composite idempotency key rejects duplicates within one tenant and permits the same key in another tenant.
2. Convert a repository test using an in-memory array into a PDO integration test against the production database engine.
3. Design a two-connection test for concurrent inventory decrements. State the invariant, isolation level, timeout, and retry assertion.
4. Add a failure after the first statement in a transaction and prove that the database contains no partial result.

## Review questions

- Why can a database mock not prove a SQL query or constraint works?
- When is SQLite an insufficient replacement for the production engine?
- What assumptions make transaction-per-test cleanup unsafe?
- Which invariants belong in database constraints as well as PHP tests?
- How should parallel database tests isolate schemas and connections?
- What does a repository test prove that an endpoint test still needs to prove?

## Summary

Use real database tests for SQL, migrations, constraints, transactions, isolation, and query behavior. Provision the production engine in isolation, build minimal deterministic fixtures, clean up without hidden shared state, test concurrency where invariants can race, and combine repository tests with endpoint and operational checks.

## References

- [PHP manual: PDO](https://www.php.net/manual/en/book.pdo.php)
- [PHP manual: PDO transactions](https://www.php.net/manual/en/pdo.transactions.php)
- [PHPUnit documentation: Writing tests](https://docs.phpunit.de/en/11.5/writing-tests-for-phpunit.html)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [PostgreSQL documentation: Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [MySQL documentation: InnoDB transaction model](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-model.html)
