---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 63
title: Request Lifecycle
slug: request-lifecycle
status: complete
summary: ../../_ai/chapter-summaries/063-request-lifecycle-summary.md
---

# Chapter 63 — Request Lifecycle

## Why This Matters

“The request is over” is a runtime event, not merely the moment a controller returns. PHP has to initialize request data, load and execute the entry point, emit a response, run shutdown work, close or release resources, and return a worker to the pool. A failure at any point can produce a different operational outcome.

The lifecycle explains why a static variable is not a safe cross-request cache, why a database write can survive a failed response, why a shutdown function is not a finally block for a killed process, and why work performed after `fastcgi_finish_request()` still occupies an FPM worker.

```text
web server / FastCGI
        ↓
request startup and SAPI input
        ↓
bootstrap → route → application work
        ↓
response status/headers/body
        ↓
shutdown callbacks and destructors
        ↓
request teardown / worker reuse
```

## Mental Model

Separate four scopes:

| Scope | Examples | Safe assumption |
| --- | --- | --- |
| Request | `$_GET`, authenticated user, local variables | Discarded when the request ends |
| Worker process | Loaded code, some extension state, OS resources | May outlive one request; do not treat as application storage |
| Shared service | Database, Redis, filesystem, queue | Durable/shared according to its own contract |
| Deployment | Source release, configuration, OPcache state | Changes independently of a request |

PHP-FPM reuses workers for efficiency, but reuse is not permission to retain user or tenant data. If a value must survive a request, put it in an explicit shared store with a defined consistency and security policy.

## Core Concept

### Startup establishes the request context

The SAPI receives a request and PHP creates request-scoped state. Depending on configuration and SAPI, PHP populates superglobals, parses cookies and form data, exposes headers/server metadata, selects the current working/request environment, and applies request-level ini behavior. The exact ordering of framework bootstrap and middleware is application-specific; do not confuse it with engine guarantees.

The entry script then begins execution. A typical front controller looks like:

```php
<?php

declare(strict_types=1);

require dirname(__DIR__) . '/vendor/autoload.php';

$request = Request::fromGlobals();
$container = Bootstrap::create($request);
$response = $container->get(Application::class)->handle($request);

$response->send();
```

Every line can fail. Autoloading can find an incompatible class; configuration can be absent; authentication can reject; a database can time out; response serialization can fail. The lifecycle is useful precisely because it makes those failure positions visible.

### User code crosses boundaries

One request may cross several boundaries:

```text
request bytes
  → parsed input
  → validated application command
  → database transaction / external calls
  → domain result
  → serialized response bytes
```

A return from a function is not a commit. A database commit is not a response delivery guarantee. A response status is not proof that a client received it. Define the business point of success and make retries safe around it.

### Shutdown is best-effort cleanup

PHP supports `register_shutdown_function()` for code to run during normal script shutdown and many script-termination paths:

```php
<?php

declare(strict_types=1);

$startedAt = hrtime(true);

register_shutdown_function(static function () use ($startedAt): void {
    $elapsedMs = (hrtime(true) - $startedAt) / 1_000_000;
    error_log(sprintf('request_shutdown elapsed_ms=%.1f', $elapsedMs));
});
```

Use shutdown work for bounded telemetry or cleanup that is genuinely safe to run late. It is not a durable job queue, a transaction replacement, or a guarantee after an operating-system kill, a fatal out-of-memory condition, or a machine failure. A shutdown callback also runs in the same worker and can extend its occupancy.

Destructors and shutdown callbacks have ordering and error-handling subtleties. Keep them small, avoid starting new business workflows there, and test the actual failure modes you rely on.

## How It Works

Conceptually, a request proceeds through these phases:

1. **Accept and parse:** the web server and SAPI accept the request and expose metadata/body input.
2. **Initialize:** PHP and the application establish request state, error handling, logging context, and dependencies.
3. **Compile/load:** the entry file and reached dependencies are loaded; OPcache may change how compilation work is reused.
4. **Execute:** routing, validation, authorization, application logic, and side effects run.
5. **Emit:** status, headers, and body cross the response boundary.
6. **Terminate:** shutdown callbacks, destructors, output-buffer handling, and request cleanup run.
7. **Reuse or exit:** an FPM worker accepts another request or is retired by its manager.

This is a reasoning model. Framework middleware and SAPI implementation can add phases, and exact engine cleanup details are version/build-sensitive. Chapters 40–59 cover source, opcodes, memory, extensions, and OPcache; Chapters 61–62 cover the preceding transport and FPM pool.

### Response commitment and client disconnects

Once response bytes are committed, the application may be unable to change status or headers. A client can disconnect before, during, or after the application performs a side effect. The server may stop sending bytes while PHP continues until its own abort/timeout policy takes effect.

```text
validate → charge card → database commit → client disconnect
                                      ↓
                              retry may duplicate work
```

Use idempotency keys, durable state transitions, and provider-specific idempotency mechanisms for operations that can be retried. “The browser did not receive 200” is not evidence that the charge or commit did not happen.

### Finishing the response is not finishing the process

On FPM, `fastcgi_finish_request()` can flush the response and finish the request from the web client's point of view, allowing PHP to continue work. That work still uses the worker, consumes CPU/memory, holds resources, and can be terminated by process or platform limits. It is not a substitute for a queue:

```php
$response->send();

if (function_exists('fastcgi_finish_request')) {
    fastcgi_finish_request();
}

// Still running inside the FPM worker:
recordNonCriticalMetric();
```

Use a queue for work that needs independent retries, durable delivery, backpressure, or a different time budget. If a small bounded metric is performed after the response, measure its effect on worker occupancy.

## Practical Example: Explicit Request Scope

Keep request data in a scope object and pass it to code that needs it:

```php
<?php

declare(strict_types=1);

final readonly class RequestContext
{
    public function __construct(
        public string $requestId,
        public string $userId,
    ) {
    }
}

function handle(RequestContext $context, ReservationCommand $command): Response
{
    $reservation = createReservation($context->userId, $command);

    return Response::json(
        ['id' => $reservation->id],
        status: 201,
        headers: ['X-Request-Id' => $context->requestId],
    );
}
```

An explicit context is easier to test than code that reads globals everywhere. It is still request data: do not place it in a process-global singleton or static cache. A correlation ID may be logged and returned, but it must not be accepted as authorization.

## Failure Modes

### Parse or bootstrap failure

If the entry file cannot parse, normal application error handling cannot run inside that file. A missing extension, invalid configuration, autoload failure, or incompatible class can fail during bootstrap. Keep a health check that exercises the actual release and runtime, not only a syntax linter.

### Error after a side effect

An exception after a database commit but before response serialization produces an ambiguous client result. Record a durable operation ID and make a retry query the operation's state. Do not wrap unrelated remote side effects in a local transaction and call the whole sequence atomic.

### Headers already sent

Whitespace, debugging output, a notice, or an early template can commit output before an error handler sets status or headers. Use response objects/output discipline, keep production diagnostics out of the body, and test the full entry path.

### Timeouts and stuck resources

A slow query, lock, DNS call, socket, or unbounded loop can hold a worker. A PHP timeout may not interrupt every kind of system call in the same way across platforms and configurations. Put timeouts at the dependency API, application budget, FPM/web-server, and client layers, then log which one fired.

### Fatal termination

Out-of-memory, process termination, host failure, and forced container kills can skip cleanup. Durable state must not depend on a shutdown callback. Use leases, transaction rollback from the durable system, reapers, and idempotent retries for work that must recover.

## Performance

Profile the phases separately: queue time before PHP, bootstrap, application CPU, database/external latency, serialization, response transfer, and cleanup. A low controller time can coexist with high user latency if the pool is queued. A short response can coexist with a long worker occupancy if post-response work runs.

Request memory should return near a stable baseline after a worker handles similar traffic. Increasing high-water memory, retained arrays, growing static caches, or unclosed resources are signals to investigate. Avoid optimizing startup while a request spends most of its time waiting on a downstream service.

## Security

Request scope is an authorization and data-isolation boundary. Clear or replace logging/user context for each request, never reuse an authenticated identity from process state, and prevent sensitive request data from leaking into logs, error pages, or response headers. Validate input before side effects and authorize against current durable state.

Do not use `register_shutdown_function()` to “finish” security-critical actions after telling the client success. Keep secrets out of correlation IDs and request URLs. Treat client disconnect, retry, and duplicate delivery as normal untrusted behavior.

## Testing

1. Unit-test parsing, validation, response mapping, and idempotency decisions.
2. Integration-test request startup against the real SAPI and environment mapping.
3. Test failures before a side effect, during a transaction, after commit, and during response serialization.
4. Use a subprocess or integration harness to test shutdown, timeout, and fatal behavior where practical.
5. Test that request context, authentication, and mutable caches do not leak between sequential requests in one worker.
6. Measure that post-response work does not exceed the worker and deployment deadline.

## Exercises

1. Draw the lifecycle for a request that commits a reservation and then loses its client connection. Mark which facts are durable and which are ambiguous.
2. Build a pair of sequential integration requests with different users and prove that no request context leaks through a static property or singleton.
3. Add a shutdown timer to a test endpoint. Compare normal return, thrown exception, timeout, and forced process termination.
4. Decide whether an email, audit event, and metrics update belong before response, after response, or in a queue. State the reliability guarantee for each.

## Review Questions

1. Which state is request-scoped, worker-scoped, and shared-service state?
2. Why is a shutdown callback not a durable job runner?
3. What does `fastcgi_finish_request()` change, and what does it not change?
4. How can a client retry after a successful database commit?
5. Which lifecycle observations distinguish a slow request from a queued request?

## Summary

A PHP request has startup, application, response, shutdown, teardown, and worker-reuse phases. Request state must be isolated from process state; response delivery is separate from durable side effects; and cleanup is best-effort. Model failures and retries at each boundary. Chapter 64 broadens the view from one request to the worker processes that provide concurrency.

## References

- [PHP Manual: `register_shutdown_function`](https://www.php.net/manual/en/function.register-shutdown-function.php)
- [PHP Manual: `fastcgi_finish_request`](https://www.php.net/manual/en/function.fastcgi-finish-request.php)
- [PHP Manual: PHP error handling](https://www.php.net/manual/en/book.errorfunc.php)
- [PHP Manual: Output control](https://www.php.net/manual/en/book.outcontrol.php)
- [PHP Manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
