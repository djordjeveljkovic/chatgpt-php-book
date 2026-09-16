---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 144
title: SQL Injection
slug: sql-injection
status: complete
summary: ../../_ai/chapter-summaries/144-sql-injection-summary.md
---

# Chapter 144 — SQL Injection

SQL injection occurs when data supplied by one party is interpreted as part of a SQL program by another. The database is doing exactly what the application asked; the defect is that the application allowed input to alter the statement's structure.

The durable fix is to keep SQL structure and values separate. Prepared statements are the main mechanism for values, while allowlists and fixed query shapes are needed for identifiers and SQL keywords. Escaping is not a substitute for either.

## Why this matters

An injected predicate can disclose rows belonging to another user, change data, bypass a login check, or alter a schema when the database account permits it. The impact is determined by the database privileges and by what the application exposes, so “the database is behind a firewall” is not a sufficient boundary.

The useful question is not whether a particular string looks dangerous. It is whether any untrusted value can cross the database boundary as executable syntax. A search term, cookie, JSON property, import file, and internal message can all be untrusted at that boundary.

## The vulnerable boundary

This code builds one SQL program by concatenating a value into it:

```php
<?php
declare(strict_types=1);

$email = $_POST['email'] ?? '';
$pdo = new PDO($dsn, $user, $password);

$sql = "SELECT id, password_hash FROM users WHERE email = '$email'";
$user = $pdo->query($sql)->fetch(PDO::FETCH_ASSOC);
```

If the input contains a quote followed by SQL syntax, the original string literal ends and the database parses the remainder as a predicate. A login query is especially attractive because it may turn a check into a condition that is true for more than one row. Error messages can also reveal table names, column names, and database details.

The same failure appears in less obvious forms:

```php
$sort = $_GET['sort'] ?? 'created_at';
$sql = "SELECT id, title FROM posts ORDER BY $sort";

$ids = implode(',', $_GET['ids'] ?? []);
$sql = "SELECT * FROM posts WHERE id IN ($ids)";
```

Parameters bind values, not SQL identifiers, sort directions, or a variable-length list of syntax tokens. Treating those as a reason to concatenate arbitrary input simply moves the vulnerability to another clause.

## Prepared statements for values

Use a prepared statement and bind every value:

```php
<?php
declare(strict_types=1);

$pdo = new PDO($dsn, $user, $password, [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_EMULATE_PREPARES => false,
]);

$statement = $pdo->prepare(
    'SELECT id, password_hash FROM users WHERE email = :email'
);
$statement->execute(['email' => $email]);
$row = $statement->fetch(PDO::FETCH_ASSOC);
```

The SQL text is parsed as a statement and `:email` is a value placeholder. The value does not become SQL syntax, even when it contains quotes or comment characters. Native prepares are a useful default where the driver supports them; test the actual driver and database combination because placeholder rules and supported statement forms are driver-specific.

Explicit types are useful when input has a domain constraint:

```php
$statement = $pdo->prepare(
    'SELECT id, total_cents FROM invoices
     WHERE account_id = :account_id AND id = :id'
);
$statement->bindValue(':account_id', $accountId, PDO::PARAM_INT);
$statement->bindValue(':id', $invoiceId, PDO::PARAM_INT);
$statement->execute();
```

Binding an integer does not validate authorization. The query still needs the account predicate, and the application still needs to decide whether a missing row should be a 404 or a forbidden response. Injection prevention and access control solve different problems.

## Dynamic SQL needs a fixed vocabulary

For a sort option, map a small external vocabulary to fixed SQL fragments:

```php
$sortExpressions = [
    'newest' => 'created_at DESC',
    'oldest' => 'created_at ASC',
    'title' => 'title ASC',
];

$sort = $_GET['sort'] ?? 'newest';
$orderBy = $sortExpressions[$sort] ?? $sortExpressions['newest'];

$statement = $pdo->prepare(
    "SELECT id, title, created_at FROM posts ORDER BY $orderBy LIMIT :limit"
);
$statement->bindValue(':limit', min(max((int) ($_GET['limit'] ?? 20), 1), 100), PDO::PARAM_INT);
$statement->execute();
```

The interpolated fragment is selected from code-owned constants, not copied from the request. The same method applies to a table selected from a finite set and to an `ASC`/`DESC` choice. Do not use a regular expression as a substitute for a finite allowlist when the accepted vocabulary is small.

For an `IN` list, create one placeholder per validated value:

```php
$rawIds = $_GET['ids'] ?? [];
if (!is_array($rawIds)) {
    throw new InvalidArgumentException('ids must be an array');
}

$ids = [];
foreach ($rawIds as $rawId) {
    if (!is_string($rawId) && !is_int($rawId)) {
        throw new InvalidArgumentException('Invalid identifier');
    }
    $id = filter_var($rawId, FILTER_VALIDATE_INT);
    if ($id === false || $id < 1) {
        throw new InvalidArgumentException('Invalid identifier');
    }
    $ids[] = $id;
}
if (count($ids) > 100) {
    throw new InvalidArgumentException('Too many identifiers');
}

if ($ids === []) {
    return [];
}

$placeholders = implode(', ', array_fill(0, count($ids), '?'));
$statement = $pdo->prepare("SELECT id, title FROM posts WHERE id IN ($placeholders)");
$statement->execute($ids);
```

This preserves the fixed grammar while letting the number of values vary. Put a maximum on list length so an attacker cannot turn the endpoint into an expensive query.

## Validation, escaping, and second-order injection

Validation describes what the application accepts: an email has an email policy, an identifier is a positive integer, and a page size has a bounded range. It improves correctness and resource use. It does not make string concatenation safe. A value that passes validation may still contain valid SQL syntax.

SQL escaping functions are easy to use incorrectly because their correctness depends on the connection, character set, quoting context, and SQL dialect. Even correctly escaped strings do not solve injection in identifiers or other syntax positions. Use parameters and fixed query fragments instead.

Second-order injection happens when an apparently harmless value is stored, then later concatenated into a different query by a batch job or administrative tool. Review every database write and every later read-to-query path. “It came from our database” does not mean it is trusted; data may have entered through an older endpoint, an import, or another service.

## Errors, privileges, and transactions

Log a correlation ID, operation name, and safe database error category for operators, but do not return SQL text, credentials, query parameters containing secrets, or stack traces to a client. Keep detailed logs access-controlled and redact values.

The application database role should have only the permissions required by that application. Separate migration privileges from runtime privileges where practical, and use separate read-only roles for read paths. Least privilege reduces impact; it does not replace parameterization.

Prepared statements do not make a group of operations atomic. If a transfer updates two accounts, use a transaction and enforce balance invariants at the database boundary:

```php
$pdo->beginTransaction();

try {
    $debit = $pdo->prepare(
        'UPDATE accounts SET balance_cents = balance_cents - :debit_amount
         WHERE id = :id AND balance_cents >= :minimum_amount'
    );
    $debit->execute([
        'debit_amount' => $amountCents,
        'minimum_amount' => $amountCents,
        'id' => $fromId,
    ]);

    if ($debit->rowCount() !== 1) {
        throw new DomainException('Insufficient funds or unknown account');
    }

    $credit = $pdo->prepare(
        'UPDATE accounts SET balance_cents = balance_cents + :amount WHERE id = :id'
    );
    $credit->execute(['amount' => $amountCents, 'id' => $toId]);
    $pdo->commit();
} catch (Throwable $exception) {
    if ($pdo->inTransaction()) {
        $pdo->rollBack();
    }
    throw $exception;
}
```

The exact isolation and locking requirements depend on the database and invariant; see [Chapter 115 — Isolation](../08-databases/115-isolation.md) and [Chapter 116 — Locks](../08-databases/116-locks.md).

## Testing the boundary

Tests should prove both behavior and the boundary itself:

1. Insert values containing quotes, Unicode, comment markers, and SQL-looking text, then assert they are treated as data.
2. Test an empty and an oversized `IN` list.
3. Test every accepted sort token and an unknown token; assert the unknown token selects the safe default.
4. Assert that a request for another account cannot return its rows.
5. Run static analysis and review every dynamic query fragment.
6. In an isolated test environment, use a proxy or database audit log to inspect statement shape without storing production secrets.

Do not make a scanner's payload the only test. A scanner can find a reflected error or a known pattern, but it cannot prove authorization, least privilege, or safe query construction across asynchronous jobs. A useful review rule is: every concatenated SQL fragment must be a code-owned constant selected from an allowlist, and every external value must be bound or represented by a validated placeholder list.

## Exercises

1. Rewrite a search endpoint that accepts `q`, `sort`, and `page_size` so values are bound, sort expressions are allowlisted, and limits are bounded.
2. Add a tenant predicate to a prepared query and write a test proving a valid invoice ID cannot cross tenant boundaries.
3. Inspect a batch import that stores a user-provided report name. Find the later query that uses it and remove the second-order injection path.

## Review questions

- Why can a prepared statement bind a value but not a column name?
- When does an allowlist make a dynamic fragment safe?
- Why is escaping fragile compared with parameterization?
- How are injection prevention, authorization, and least privilege different controls?
- What evidence would show that a query is safe when the first request only stores data?

## References

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [PHP manual: PDO prepared statements](https://www.php.net/manual/en/pdo.prepared-statements.php)
- [PHP manual: PDO::prepare](https://www.php.net/manual/en/pdo.prepare.php)
- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
