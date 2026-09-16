---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 185
title: Dependency Injection
slug: dependency-injection
status: complete
summary: ../../_ai/chapter-summaries/185-dependency-injection-summary.md
---

# Chapter 185 — Dependency Injection

Dependency injection means an object receives the collaborators it needs instead of constructing or locating them internally. The object declares a dependency; a composition root decides which implementation to provide.

The technique is simple, but its value is architectural. It makes data flow visible, keeps policy separate from wiring, and lets tests replace a clock, repository, provider, or transport without modifying production code.

## Why this matters

A class that calls `new PDO()`, reads a global configuration array, and reaches a singleton mailer has hidden inputs. Tests cannot control those inputs cleanly, and a change in deployment configuration can affect unrelated code. The class also knows how to build infrastructure when its real responsibility is business policy.

This design hides dependencies:

```php
final class InvoiceService
{
    public function send(int $invoiceId): void
    {
        $pdo = new PDO(getenv('DATABASE_DSN'));
        $mailer = Mailer::instance();
        // Query and send using hidden collaborators.
    }
}
```

Constructor injection makes the dependency graph explicit:

```php
<?php

declare(strict_types=1);

interface InvoiceRepository
{
    public function find(int $id): ?Invoice;
}

interface InvoiceMailer
{
    public function send(Invoice $invoice): void;
}

final readonly class Invoice
{
    public function __construct(
        public int $id,
        public string $recipient,
        public int $totalCents,
    ) {
    }
}

final class InvoiceService
{
    public function __construct(
        private InvoiceRepository $invoices,
        private InvoiceMailer $mailer,
    ) {
    }

    public function send(int $invoiceId): void
    {
        $invoice = $this->invoices->find($invoiceId);
        if ($invoice === null) {
            throw new DomainException('Invoice unavailable');
        }

        $this->mailer->send($invoice);
    }
}
```

The service owns the decision that an invoice must exist before sending. The repository and mailer own their integration details.

## Composition roots

A composition root is the narrow part of the application where concrete objects are assembled. In a PHP web application it may be a bootstrap file, a framework container configuration, or a command entry point. Keep it near the application boundary:

```php
$pdo = new PDO($dsn, $user, $password, [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
]);
$invoiceRepository = new PdoInvoiceRepository($pdo);
$invoiceMailer = new SmtpInvoiceMailer($smtpClient);
$invoiceService = new InvoiceService($invoiceRepository, $invoiceMailer);
```

The service does not need to know whether the repository uses PDO, an HTTP API, or a read replica. Each entry point should compose the graph it needs, including queue workers and CLI commands. A worker should not borrow a web request's mutable singleton state.

## Constructor injection and PHP types

Constructor injection is usually the default because an object cannot be valid until required dependencies exist. Use interfaces when the boundary has multiple meaningful implementations or when the implementation is infrastructure. Use concrete value objects when the dependency is stable and has no useful alternate behavior.

Avoid nullable dependencies that mean “sometimes this feature is configured.” Make the mode explicit with separate implementations or a null object:

```php
final class NullInvoiceMailer implements InvoiceMailer
{
    public function send(Invoice $invoice): void
    {
        // Deliberately discard mail in a documented local/test mode.
    }
}
```

A null object is safe only when discarding the effect is an explicit policy. It must not silently disable production notifications.

## Containers and service locators

A dependency-injection container can construct an object graph, manage shared lifetimes, and apply configuration. It should remain a composition tool. Code that asks the container for arbitrary services is using a service locator and has hidden dependencies again:

```php
// Hidden dependency: the method can request anything at runtime.
$service = $container->get(InvoiceService::class);
```

Prefer resolving the service at the boundary and passing it into the handler. Container autowiring is convenient, but inspect the generated graph, fail at startup for missing bindings, and avoid magic for values that require validation or security policy. A container cannot decide whether a tenant, credential, or retry policy is correct.

## Lifetimes and mutable state

Request-scoped objects can hold request data. Shared objects should be immutable or explicitly safe for reuse. A long-running worker must not retain a user token, transaction, or growing collection in a shared service. Choose lifetimes deliberately for PDO connections, caches, clients, and rate limiters.

Factories can inject runtime values without turning constructors into service locators. If a handler needs a new idempotency key or a per-request child object, inject a factory with a narrow method rather than a whole container.

## Testing and operations

Injection makes unit tests straightforward:

```php
final class InMemoryInvoices implements InvoiceRepository
{
    public function __construct(private ?Invoice $invoice)
    {
    }

    public function find(int $id): ?Invoice
    {
        return $this->invoice?->id === $id ? $this->invoice : null;
    }
}

final class RecordingMailer implements InvoiceMailer
{
    /** @var list<int> */
    public array $sent = [];

    public function send(Invoice $invoice): void
    {
        $this->sent[] = $invoice->id;
    }
}
```

A test can assemble these collaborators directly. Integration tests still need to exercise `PdoInvoiceRepository` and the real mail adapter or a contract environment. Dependency injection improves substitution; it does not make an adapter correct.

Review the composition root as part of deployment. A missing binding, wrong credential, or accidentally shared client should fail loudly at startup with a safe diagnostic. Instrument dependency failures at the adapter boundary, not by leaking infrastructure details through every domain class.

## Exercises

1. Refactor a class that constructs a PDO connection and mailer internally into constructor-injected interfaces.
2. Draw the composition root for a web request and a queue worker. Identify which objects are request-scoped and which are shared.
3. Replace a container lookup inside a domain service with an explicit dependency and add a focused test.

## Review questions

- What problem does dependency injection solve beyond making tests easier?
- Why is a composition root different from a service locator?
- When is an interface useful, and when is it needless indirection?
- Which objects should not be shared across requests or worker jobs?
- What must integration tests still verify after a dependency has been injected?

## Summary

Inject required collaborators, assemble them at explicit composition roots, use interfaces at meaningful boundaries, and keep containers out of business logic. Choose lifetimes deliberately, make optional behavior explicit, and combine substitution-friendly unit tests with real adapter and deployment checks.

## References

- [PHP manual: Constructor property promotion](https://www.php.net/manual/en/language.oop5.basic.php)
- [PHP-FIG PSR-11: Container Interface](https://www.php-fig.org/psr/psr-11/)
- [Martin Fowler: Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html)
