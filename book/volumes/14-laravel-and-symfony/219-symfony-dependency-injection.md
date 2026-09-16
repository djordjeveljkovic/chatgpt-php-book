---
book: The Complete Modern PHP Engineering Book
volume: 14
volume_title: LARAVEL AND SYMFONY
chapter: 219
title: Symfony Dependency Injection
slug: symfony-dependency-injection
status: complete
summary: ../../_ai/chapter-summaries/219-symfony-dependency-injection-summary.md
---

# Chapter 219 — Symfony Dependency Injection

Symfony's DependencyInjection component builds and configures an object graph. A service is an object managed by the container; a definition describes how it is created, and an alias maps an abstraction to an implementation.

The container is useful at the composition boundary. Application code should still use constructor injection and explicit types. Autowiring reduces repetitive wiring; it does not decide business ownership, authorization, transaction scope, or secret policy.

## Why this matters

Manual construction works for a small graph but becomes fragile as services gain clients, repositories, clocks, and policies. A container centralizes wiring and can validate the graph during build or cache warmup. The application remains responsible for choosing the right implementation and lifetime.

A service can declare its dependency normally:

```php
<?php

declare(strict_types=1);

final class InvoiceSender
{
    public function __construct(
        private InvoiceRepository $invoices,
        private InvoiceMailer $mailer,
    ) {
    }

    public function send(int $id): void
    {
        $invoice = $this->invoices->find($id);
        if ($invoice === null) {
            throw new DomainException('Invoice unavailable');
        }
        $this->mailer->send($invoice);
    }
}
```

Autowiring can discover the constructor types. If multiple implementations exist, provide an explicit alias or named binding rather than relying on an accidental class.

## Service definitions and aliases

Version-neutral YAML can express the intent:

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        exclude: '../src/{DependencyInjection,Entity,Kernel.php}'

    App\Billing\InvoiceMailer: '@App\Billing\SmtpInvoiceMailer'
```

The exact resource paths and namespace syntax depend on the application layout. Keep the container configuration code-reviewed. A broad resource rule can register classes that should not be services, and an alias can accidentally route production traffic to a fake implementation.

Use explicit arguments for scalar values and security-sensitive configuration. Type a timeout, endpoint, or feature mode at the boundary rather than injecting an unvalidated string everywhere. Symfony's environment processors and secrets facilities should be combined with least-privilege deployment access.

## Public, private, and lazy services

Application services should normally be private and reached through injected dependencies. A public service can be fetched from the container by runtime code, which recreates a service locator. Keep public exposure only for an intentional entry point or integration contract.

Lazy services defer construction until use and can reduce startup work, but they do not remove network latency or failure. A lazy proxy may make a local method call perform I/O for the first time. Document that behavior and set timeouts at the actual client boundary.

Shared services are reused by the container in the normal request lifecycle. Do not put request-specific data, transactions, or user credentials into shared mutable services. Long-running workers need explicit reset behavior for stateful services.

## Factories, decorators, and compiler passes

Use a factory when construction depends on validated runtime or configuration data. Use decoration to add metrics, tracing, authorization, or retry policy while preserving a narrow interface. Ensure decorator order is deliberate; measuring each retry attempt differs from measuring one logical operation.

Compiler passes can modify definitions during container compilation, such as registering tagged handlers. They are powerful and framework-specific. Keep them deterministic, test the compiled graph, and avoid using them to hide business decisions that belong in application code.

A tagged handler map may be appropriate for a command bus, but the handler still validates the command and authorization context. Container discovery does not make arbitrary classes safe to invoke.

## Testing the container

Unit-test services by constructing them directly with fakes or stubs. Add a container compilation test to catch missing aliases, circular dependencies, invalid configuration, and unintended service registration:

```php
final class ContainerSmokeTest extends KernelTestCase
{
    public function testContainerCompiles(): void
    {
        self::bootKernel();

        self::assertNotNull(self::getContainer()->get(InvoiceSender::class));
    }
}
```

The exact test base and test-container behavior vary by Symfony version and application. Do not make every unit test boot the full kernel. Test production and test environment bindings separately when a fake or in-memory adapter is configured.

## Failure and operations

Fail deployment when required bindings, secrets, routes, or configuration are invalid. Cache warmup should run in the build or release stage so a missing class or alias is discovered before traffic arrives. Do not expose the service graph, environment values, or container exception details to clients.

Record the application build, configuration version, and environment identity in safe diagnostics. Monitor dependency failures at adapters rather than treating successful container compilation as proof that the dependency is reachable.

## Exercises

1. Add two `InvoiceMailer` implementations and configure an explicit production alias and test binding.
2. Find a service locator call and replace it with constructor injection and a container entry-point resolution.
3. Write a container compilation test that fails when a required scalar parameter is missing.

## Review questions

- What does Symfony autowiring solve and what does it not decide?
- Why should most application services be private?
- When are factories, decorators, or compiler passes appropriate?
- Which state is unsafe to retain in a shared service?
- Why is container compilation not a provider health check?

## Summary

Use Symfony's container as a composition and validation tool. Prefer constructor injection, explicit aliases and scalar configuration, private services, deliberate lifetimes, and narrow decorators. Test the compiled graph and fail before deployment while keeping business policy and provider health outside container magic.

## References

- [Symfony Service Container](https://symfony.com/doc/current/service_container.html)
- [Symfony Service Container: Autowiring](https://symfony.com/doc/current/service_container/autowiring.html)
- [Symfony Service Container: Service decoration](https://symfony.com/doc/current/service_container/service_decoration.html)
- [Symfony Secrets](https://symfony.com/doc/current/configuration/secrets.html)
