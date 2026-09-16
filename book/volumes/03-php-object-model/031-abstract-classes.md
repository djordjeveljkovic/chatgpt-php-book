---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 31
title: Abstract Classes
slug: abstract-classes
status: complete
summary: ../../_ai/chapter-summaries/031-abstract-classes-summary.md
---

# Chapter 31 — Abstract Classes

## Why This Matters

An abstract class is a partially implemented parent. It can hold state and concrete behavior while requiring children to implement selected operations. That makes it useful for a family of algorithms with a stable workflow and controlled extension points.

It is also a strong coupling mechanism. Every child inherits the parent’s state, constructor assumptions, visibility choices, and future changes. Use an abstract class when shared implementation and a shared invariant are central to the model; use an interface when you need a capability that unrelated classes can provide.

## Mental Model

An abstract class is a class-shaped template that cannot be instantiated directly. It can have abstract methods, concrete methods, properties, constants, and a constructor. A concrete descendant must implement all inherited abstract methods before it can be instantiated.

The key design question is where variation belongs. Put invariant workflow in concrete methods, expose a small protected extension point, and keep the parent’s state valid. Do not make every method abstract just to call the result an abstraction.

## Core Concept

```php
<?php

declare(strict_types=1);

abstract class MessageHandler
{
    final public function handle(string $payload): void
    {
        $message = $this->decode($payload);
        $this->validate($message);
        $this->process($message);
    }

    abstract protected function process(array $message): void;

    protected function decode(string $payload): array
    {
        $message = json_decode($payload, true, flags: JSON_THROW_ON_ERROR);

        if (!is_array($message)) {
            throw new UnexpectedValueException('Message must be an object');
        }

        return $message;
    }

    protected function validate(array $message): void
    {
        if (!isset($message['type'])) {
            throw new InvalidArgumentException('Message type is required');
        }
    }
}
```

The parent owns the order of operations. The child supplies only domain-specific processing. Making `handle` final prevents a descendant from bypassing validation accidentally.

## How It Works

PHP records the abstract methods and rejects instantiation of a class that still has an abstract contract. Calls to concrete methods use normal method dispatch. The abstract class does not execute as a separate runtime object; its inherited concrete methods become part of the child’s behavior.

This is a useful conceptual model for Zend behavior. Exact method-table representation is an implementation detail. What matters to application design is that the child is coupled to the parent’s method signatures and protected surface at declaration time.

## What PHP Does

An abstract method has a declaration but no body:

```php
abstract class Formatter
{
    abstract public function format(array $data): string;
}

final class CsvFormatter extends Formatter
{
    public function format(array $data): string
    {
        return implode(',', array_map(
            static fn (mixed $value): string => (string) $value,
            $data,
        ));
    }
}
```

`new Formatter()` is invalid, while `new CsvFormatter()` is valid. The child’s method must be signature-compatible with the abstract declaration. A descendant can itself remain abstract when it implements only part of the contract.

## Practical Example

Shared retry policy can be appropriate when every implementation truly follows the same policy:

```php
abstract class RetryingClient
{
    public function __construct(private int $attempts = 3)
    {
        if ($attempts < 1) {
            throw new InvalidArgumentException('At least one attempt is required');
        }
    }

    final public function send(Request $request): Response
    {
        $last = null;

        for ($attempt = 1; $attempt <= $this->attempts; $attempt++) {
            try {
                return $this->sendOnce($request);
            } catch (TransientTransportFailure $exception) {
                $last = $exception;
            }
        }

        if ($last === null) {
            throw new LogicException('Retry loop ended without a result or failure');
        }

        throw $last;
    }

    abstract protected function sendOnce(Request $request): Response;
}
```

The parent can centralize the attempt count, but production code still needs backoff, timeouts, a retry budget, and idempotency. A base class should not imply that every operation is safe to repeat merely because it can be retried mechanically.

## Production Example

A framework adapter may use an abstract base for a stable lifecycle—authenticate, validate input, execute, map output—while concrete adapters provide provider-specific calls. Keep the base class close to that bounded family. If an adapter later needs streaming, two-phase operations, or different authentication, do not keep adding flags to preserve the hierarchy; split the contract or compose a policy object.

## Bad Example

```php
abstract class BaseService
{
    protected PDO $pdo;
    protected Logger $logger;
    protected Cache $cache;
    protected Mailer $mailer;
    // Every child inherits every infrastructure concern.
}
```

This is a dependency landfill. The base class exposes mutable protected state and makes unrelated subclasses depend on one another’s assumptions. It is difficult to test and difficult to remove.

## Better Example

Keep a base class narrow, or use composition:

```php
abstract class ReservationPolicy
{
    final public function check(ReservationDraft $draft): void
    {
        $this->checkCommonRules($draft);
        $this->checkSpecificRules($draft);
    }

    private function checkCommonRules(ReservationDraft $draft): void
    {
        if ($draft->range()->isEmpty()) {
            throw new DomainException('A reservation needs a positive duration');
        }
    }

    abstract protected function checkSpecificRules(ReservationDraft $draft): void;
}
```

The common rule is private because descendants should not replace it. The protected method is a deliberately small extension point. If policies need unrelated dependencies or different workflows, an interface plus composed validators is a better fit.

## Edge Cases

- An abstract class can implement an interface without implementing every method; it passes the remaining obligations to concrete descendants.
- An abstract class cannot be `final`, because `final` forbids extension.
- A child constructor does not automatically run the parent constructor when the child declares its own constructor.
- Protected properties and methods are available to descendants, so changing them can be a breaking change for subclasses even if no public signature changes.
- A base class should not call overridable methods from its constructor unless the lifecycle is carefully designed; the child may not be initialized yet.
- An abstract class provides one inheritance slot, so choosing it prevents the class from extending another parent.

## Performance

The cost of an abstract class is usually design complexity, not dispatch speed. Shared concrete code can reduce duplication, while an over-general base class can cause extra conditionals, allocations, or calls. Measure real workflows; do not select a hierarchy based on imagined micro-optimizations.

## Security

Protected extension points are attack surfaces when subclasses come from plugins or untrusted integrations. Enforce authentication and authorization in non-overridable boundary code where possible. Do not rely on every subclass remembering to call a protected security method; make the invariant part of a final workflow or an external policy that the caller cannot bypass.

## Testing

Test the template once for common invariants and test each child’s variation:

```php
final class CsvFormatterTest extends TestCase
{
    public function testConcreteFormatterImplementsTheAbstractOperation(): void
    {
        $formatter = new CsvFormatter();

        self::assertSame('Ada,42', $formatter->format(['Ada', 42]));
    }
}
```

For a handler base class, test that malformed messages never reach `process`, that validation happens before side effects, and that a child cannot bypass the workflow. Integration tests should verify actual provider behavior rather than assuming all subclasses have identical failure semantics.

## Common Mistakes

- Using an abstract class solely to share a static helper.
- Exposing broad protected mutable state.
- Calling overridable methods from a parent constructor.
- Adding flags until one base class supports unrelated algorithms.
- Assuming a shared retry loop is safe for every operation.
- Forgetting that protected members are part of the subclass compatibility surface.

## Senior Engineer Thinking

An abstract class is a policy decision about shared implementation and shared evolution. Write down the invariant the parent owns, the exact variation children provide, and the methods that must never be bypassed. If the only commonality is a name or a few independent utilities, use an interface or a function. If the workflow is stable and the extension point is narrow, an abstract class can make the correct sequence hard to violate.

## Exercises

1. Build an abstract `Importer` with a final `run()` workflow and one concrete CSV importer. Identify which steps are invariant.
2. Find a base class with a protected property and redesign it so the property cannot be mutated by children.
3. Add a retry policy to a subclass. Decide which failures are retryable and how idempotency is guaranteed.
4. Convert an abstract “service base” into a composition graph with explicit dependencies.

## Review Questions

1. What can an abstract class provide that an interface cannot?
2. Why should a shared workflow often be final?
3. Why are protected members a long-term compatibility cost?
4. When is an interface a better choice than an abstract class?
5. What makes a retrying abstract client unsafe for a non-idempotent operation?

## Summary

Abstract classes combine shared state and implementation with required extension points. They work best for a small family with a stable invariant workflow. Keep the parent narrow, protect initialization and security rules, and choose interfaces or composition when implementations do not share the same lifecycle.
