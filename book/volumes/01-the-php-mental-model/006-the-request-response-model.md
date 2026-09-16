---
book: The Complete Modern PHP Engineering Book
volume: 1
volume_title: THE PHP MENTAL MODEL
chapter: 6
title: The Request/Response Model
slug: the-request-response-model
status: complete
summary: ../../_ai/chapter-summaries/006-the-request-response-model-summary.md
---

# Chapter 6 — The Request/Response Model

## Why This Matters

A web application is often described as if a user calls a PHP function and receives its return value. That description is convenient, but incomplete. Between the user and the function are several boundaries: a client creates a request, a web server accepts and routes it, a PHP entry point turns it into runtime data, application code performs work, and a response is assembled for a client that may disconnect before seeing it.

If those boundaries remain invisible, common incidents look mysterious. A form value appears as a string even when it represents a number. A controller returns an error after a database write has already committed. A client retries a timed-out request and creates a duplicate order. A response says “success” even though a warning or accidental output corrupted its body.

The request/response model gives us a better unit of reasoning. It lets us ask:

- What entered the process?
- Which parts were parsed, normalized, and validated?
- What status, headers, and body leave the process?
- Which side effects happened before that response?
- What can fail at each boundary?
- How can the behavior be tested without starting a browser?

This chapter establishes that model. Detailed HTTP semantics, caching, content negotiation, security headers, and protocol-level behavior belong in Volume IX.

## Mental Model

Treat one web interaction as a bounded exchange:

```text
Client
  │ request: method, target, headers, body
  ▼
Web server / reverse proxy
  │ routing, limits, TLS termination, forwarding
  ▼
PHP entry point / SAPI
  │ parsing and environment setup
  ▼
Application code
  │ validation, decisions, side effects
  ▼
Response: status, headers, body
  │
  ▼
Client
```

The request is input to this invocation. The response is output from it. A normal PHP web request should not be modeled as a permanent conversation in which local variables naturally survive until the next request. The process may be reused by PHP-FPM, but request data and application state still need explicit ownership and lifecycle rules.

There are two important boundaries here:

1. The transport boundary: bytes and metadata cross between the client, server, and PHP entry point.
2. The application boundary: parsed input becomes application values, and application results become a response.

Keeping these boundaries visible prevents transport details from spreading through business logic. A domain service should not need to know whether a value arrived from a JSON body, an HTML form, a CLI option, or a queue message.

## Core Concept

An HTTP request has three useful conceptual parts:

- The method and target describe the requested operation and resource location.
- Headers carry metadata such as content type, credentials, correlation identifiers, and client preferences.
- The body carries optional content, such as form fields, JSON, or an uploaded file.

The response has a matching structure:

- The status communicates the broad outcome to the client.
- Headers carry metadata about the response and how it should be interpreted.
- The body contains the representation or error details.

Status, headers, and body are separate channels of meaning. A JSON body containing `{"error":"not found"}` does not by itself make the response a not-found response. The status must also communicate that outcome. Conversely, a successful status does not prove that every intended side effect occurred; the application must define when success is reported.

## How It Works

### From bytes to PHP input

The client sends bytes. The web server or reverse proxy may terminate TLS, enforce size limits, select a route, and pass request information to the PHP SAPI. PHP then exposes parts of that information through the runtime environment and input facilities.

Typical sources include:

```php
<?php

declare(strict_types=1);

$method = $_SERVER['REQUEST_METHOD'] ?? 'GET';
$query = $_GET;
$form = $_POST;
$cookies = $_COOKIE;
$files = $_FILES;
$rawBody = file_get_contents('php://input');
```

These are transport-shaped values. Query parameters and form fields commonly arrive as strings. Arrays may be nested. A missing key is different from an empty string. An uploaded file has metadata and a temporary location, not simply a trusted pathname supplied by the client.

Parsing is not validation. If the body is JSON, decoding it answers “can these bytes be represented as a PHP value?” It does not answer “is this value allowed for this operation?” A robust boundary usually has these stages:

```text
receive
  → parse
  → validate shape and constraints
  → normalize into application values
  → perform application work
  → translate result into response
```

Each stage should have an explicit failure result. Malformed JSON, for example, is an input problem; it is not the same as a database outage.

### From PHP output to a response

At the other side, PHP code can set a status and headers and write a body:

```php
<?php

declare(strict_types=1);

http_response_code(201);
header('Content-Type: application/json; charset=utf-8');

echo json_encode(
    ['id' => 42],
    JSON_THROW_ON_ERROR,
);
```

The SAPI and web server turn these operations into the outgoing response. Headers must be sent before the response body is committed. Accidental output—such as a notice, debugging statement, or whitespace emitted too early—can therefore change or prevent headers. Output buffering can delay physical output, but it does not make a database transaction or the whole request atomic.

Framework response objects make this boundary more explicit. They usually collect status, headers, and body as data and emit them near the outer edge of the application. That design is easier to test because application code can return a response value instead of writing directly to a global output stream.

## What PHP Does

PHP provides the primitives; it does not decide what a request means. It exposes input, executes the selected script, allows code to produce output, and reports errors according to configuration and the active SAPI.

The same PHP function can be called from several entry points. A web controller might extract a request value, but the operation that calculates a price should accept a typed price and quantity. Converting transport input at the edge keeps the core code independent of `$_POST`, `$_GET`, and framework request objects.

PHP also does not roll back arbitrary effects when an exception is thrown. An exception can stop later application code, but it cannot undo an email already sent, a file already written, or a database commit unless the relevant dependency provides a transaction or compensating operation.

## Minimal Example

This handler separates parsing, validation, and response construction. It is intentionally small:

```php
<?php

declare(strict_types=1);

/** @return array{status: int, headers: array<string, string>, body: string} */
function createGreeting(string $rawBody): array
{
    try {
        /** @var mixed $decoded */
        $decoded = json_decode($rawBody, true, 512, JSON_THROW_ON_ERROR);
    } catch (JsonException) {
        return jsonResponse(400, ['error' => 'Malformed JSON']);
    }

    if (!is_array($decoded) || !isset($decoded['name']) || !is_string($decoded['name'])) {
        return jsonResponse(422, ['error' => 'name must be a string']);
    }

    $name = trim($decoded['name']);
    if ($name === '') {
        return jsonResponse(422, ['error' => 'name must not be empty']);
    }

    return jsonResponse(200, ['message' => "Hello, {$name}"]);
}

/** @param array<string, mixed> $payload */
function jsonResponse(int $status, array $payload): array
{
    return [
        'status' => $status,
        'headers' => ['Content-Type' => 'application/json; charset=utf-8'],
        'body' => json_encode($payload, JSON_THROW_ON_ERROR),
    ];
}
```

The return type is an application-level response description. A thin outer adapter can apply its fields to `http_response_code()`, `header()`, and the output stream. The greeting function can be tested without a web server and without mutating global request state.

## Practical Example: Side Effects and Boundaries

Suppose an endpoint creates an account. A naïve sequence might be:

```text
parse input
insert account
send welcome email
return 201 Created
```

Each step has a different failure surface. Parsing can fail before any side effect. The insert can fail because of a constraint or unavailable database. Email delivery can time out after the account has been committed. The client can disconnect after the insert and before receiving the response.

The response is not a transaction boundary. A `201` response means the application has chosen to report creation, but it does not make the email synchronous operation reliable. A more deliberate design might commit the account and record an outbox event in the same database transaction, then let a worker deliver the email. That moves the delivery concern to another boundary while preserving a durable record of intent.

The correct design depends on the requirement. If the account must exist before the client proceeds, the account transaction belongs on the request path. If the welcome email must be delivered eventually, an outbox and retryable worker may be appropriate. If an operation cannot safely be repeated, the boundary may need an idempotency key and a stored result.

## Bad Example

This handler mixes transport access, business rules, output, and a side effect:

```php
<?php

if (!isset($_POST['amount'])) {
    echo 'missing amount';
    return;
}

$amount = (int) $_POST['amount'];

chargeCard($amount);
echo json_encode(['ok' => true]);
```

Problems include:

- the input source is hard-coded into the business operation;
- casting silently turns some invalid values into an unintended integer;
- the response content type is not declared;
- a failed or repeated charge has no explicit policy;
- output is written before the caller can consistently choose status and headers;
- “JSON was produced” is treated as proof that charging succeeded.

The danger is not that this code is short. The danger is that important decisions are implicit and difficult to test.

## Better Example

Keep the edge adapter thin and pass a validated value inward:

```php
<?php

declare(strict_types=1);

function handleCreatePayment(string $rawBody, PaymentGateway $gateway): array
{
    $input = parsePaymentInput($rawBody);
    if ($input instanceof InputError) {
        return jsonResponse($input->status, ['error' => $input->message]);
    }

    try {
        $receipt = $gateway->charge($input->amountCents, $input->idempotencyKey);
    } catch (PaymentDeclined) {
        return jsonResponse(402, ['error' => 'Payment declined']);
    } catch (PaymentGatewayUnavailable) {
        return jsonResponse(503, ['error' => 'Payment service unavailable']);
    }

    return jsonResponse(201, ['receiptId' => $receipt->id]);
}
```

The example still needs a real parser and domain policy, but its boundary is visible. Input errors, business outcomes, and dependency failures can be represented separately. The gateway contract also makes the retry question explicit: the idempotency key must cause a repeated request to return or reuse the same logical charge rather than create another one.

## Failures and Retries

A client observes only what crosses the response boundary. These observations are ambiguous:

- A clear `4xx` response usually tells the client that changing the request may help.
- A `5xx` response usually tells the client that the server could not complete the operation, but not whether a side effect already happened.
- A timeout or connection reset tells the client even less: the server may have done nothing, may still be working, or may have completed the operation just before the connection failed.

Therefore, retry policy must be based on operation semantics, not merely on the absence of a response. Reads are often easier to retry than creates. A create operation may need an idempotency key, a unique business key, or a durable operation record. Database constraints remain important because an application-level “check then insert” can race with another request.

Errors should be translated at the outer boundary. Internal exception messages may contain SQL, paths, credentials, or implementation details and should not be returned to an untrusted client. Logs and traces should preserve diagnostic context without exposing secrets.

## Edge Cases

- Missing input is not always the same as `null`, an empty string, or an empty array.
- A request body can be malformed, too large, truncated, or encoded differently from what the parser expects.
- A client can disconnect while PHP is still performing work.
- A response can fail after a side effect has succeeded.
- Headers can already be committed when later code discovers an error.
- An exception handler can itself fail while trying to format an error response.
- A PHP process can be terminated before cleanup code runs.
- A response body may be valid JSON but still have the wrong status or content type.

These cases are reasons to define contracts at the boundary, not reasons to put every concern in one controller.

## Performance

Request cost includes more than PHP execution time. It can include proxy queuing, network transfer, request-body parsing, authentication, database work, serialization, and response transfer. Measure the stages that matter rather than optimizing the controller in isolation.

Parsing and encoding large bodies consume memory. Streaming can reduce peak memory for suitable formats and workflows, but it changes the design: validation, error reporting, and retry behavior must work with partial input. Do not load an entire upload into memory merely because a small JSON request was convenient to decode that way.

Response buffering, compression, and connection reuse are deployment concerns as well as application concerns. They can change when bytes reach the client, but they do not change the application’s logical status or undo a completed side effect.

## Security

Treat every request value as untrusted, including headers, cookies, query parameters, form fields, JSON, and upload metadata. Authentication identifies a caller; authorization decides whether that caller may perform this operation. Parsing a valid JSON document proves neither.

Avoid reflecting raw input into HTML, headers, logs, or SQL. Use context-appropriate output encoding, parameterized database operations, and controlled log fields. Limit body and upload sizes before expensive processing where possible. Do not leak stack traces or dependency errors in response bodies.

CSRF defenses, cookie attributes, CORS, content security policy, and other detailed web security controls depend on the deployment and protocol use case. They are covered in the security and HTTP volumes; the mental model here is that the request boundary is an attack boundary.

## Testing

Test the boundary at several levels:

1. Unit-test parsing and validation with malformed, missing, empty, and unexpected values.
2. Unit-test application behavior with a fake or in-memory dependency, including declared failure outcomes.
3. Integration-test the adapter that maps a request to application input and application output to status, headers, and body.
4. Exercise the real web stack when routing, server configuration, uploads, authentication middleware, or response behavior is part of the risk.

For the minimal example, useful assertions include:

```php
<?php

declare(strict_types=1);

$response = createGreeting('{"name":"Ada"}');

assert($response['status'] === 200);
assert($response['headers']['Content-Type'] === 'application/json; charset=utf-8');
assert(json_decode($response['body'], true, 512, JSON_THROW_ON_ERROR) === [
    'message' => 'Hello, Ada',
]);

$badResponse = createGreeting('{"name":');
assert($badResponse['status'] === 400);
```

The assertions inspect status, headers, and decoded body separately. In a real project, use a test framework's assertions rather than relying on PHP's `assert()` configuration.

Test failure paths, not just the happy path. In particular, verify that a dependency failure produces the intended response and that a retry does not duplicate a side effect. If the client can time out, test the operation record or idempotency behavior that lets a later request discover what happened.

## Common Mistakes

- Treating a web request as a direct function call from the browser to PHP.
- Assuming parsing performs validation or authorization.
- Returning an error response after an already-committed side effect without defining recovery.
- Retrying every timeout without asking whether the operation is idempotent.
- Mixing `$_POST`, headers, business rules, database calls, and output in one function.
- Assuming output buffering makes the request atomic.
- Testing only a controller's successful response and ignoring malformed input, disconnects, and dependency failures.
- Returning internal exception text to the client.

## Senior Engineer Thinking

When reviewing an endpoint, draw the boundaries before reviewing its syntax:

```text
untrusted request
  → parsed input
  → validated application command
  → side effects and durable state
  → response description
  → transport output
```

Then ask where each guarantee comes from. Does the parser guarantee a shape? Does validation enforce the business invariant? Does the database enforce uniqueness under concurrent requests? Does the external service support idempotency? Can the client distinguish “rejected” from “unknown outcome”? Does the test observe status, headers, body, and side effects separately?

Experienced engineers do not equate a response with the entire history of the request. They reason about what happened before the response, what may still be happening after a timeout, and what evidence allows the next request or operator to recover safely.

## Exercises

1. Add validation that rejects a greeting name longer than one hundred characters. Decide whether the limit belongs to parsing, validation, or a domain policy, and test the boundary value and the first invalid value.
2. Design the request and response contract for an endpoint that creates a reservation. List the side effects, the possible partial failures, and the key that would make a retry safe.
3. Refactor a controller that reads `$_POST` and writes directly to output. Introduce a request-to-command adapter and a response description, then test both independently.

## Review Questions

- What are the three conceptual parts of a request and of a response?
- Why is parsing different from validation?
- Why can a successful response coexist with an unsuccessful email delivery?
- What does a timeout tell the client, and what does it not tell the client?
- Which concerns belong at the transport edge, and which should be independent of HTTP?
- What must be tested besides the response body?

## Summary

A PHP web program participates in a bounded request/response exchange. Input crosses a transport boundary, is parsed into PHP values, validated and normalized into application values, and then drives decisions and side effects. The application translates its outcome into a response made of status, headers, and body.

The response is not a transaction and not a complete record of what happened. A dependency can fail after a database commit, a client can disconnect after work succeeds, and a retry can repeat a side effect unless the operation is designed for safe repetition. Reliable endpoints make parsing, validation, side effects, failure handling, and response construction explicit—and test each boundary separately.

## Sources and Further Reading

- [PHP Manual: Variables From External Sources](https://www.php.net/manual/en/language.variables.external.php)
- [PHP Manual: Reserved Variables](https://www.php.net/manual/en/reserved.variables.php)
- [PHP Manual: `header()`](https://www.php.net/manual/en/function.header.php)
- [PHP Manual: `http_response_code()`](https://www.php.net/manual/en/function.http-response-code.php)
- [PHP Manual: `php://input`](https://www.php.net/manual/en/wrappers.php.php)
