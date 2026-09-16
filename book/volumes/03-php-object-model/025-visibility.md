---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 25
title: Visibility
slug: visibility
status: complete
summary: ../../_ai/chapter-summaries/025-visibility-summary.md
---

# Chapter 25 — Visibility

## Why This Matters

Visibility is how a class answers “who is allowed to depend on this?” A public member is an API promise. A protected member is an extension seam for a class hierarchy. A private member is an implementation detail of the class that declares it. Choosing `public` because it is convenient transfers the burden of every invariant to every caller.

Visibility does not provide security against a compromised process, nor does it replace authorization. It provides compile/runtime-enforced design boundaries inside a PHP program. Those boundaries make refactoring safer because callers can depend on fewer details.

## Mental Model

PHP has three ordinary visibility levels:

```php
final class Account
{
    public function deposit(int $cents): void {}

    protected function audit(string $event): void {}

    private int $balance = 0;
}
```

- `public`: accessible from any scope with the object or class.
- `protected`: accessible inside the declaring class and subclasses.
- `private`: accessible only inside the class that declares the member.

The same keywords apply to methods and properties. Since PHP 7.1 they also apply to class constants. Explicit visibility is clearer than relying on the historical default of public.

## Core Concept

Expose intent, hide representation:

```php
final class Reservation
{
    private string $status = 'pending';

    public function confirm(): void
    {
        if ($this->status !== 'pending') {
            throw new DomainException('Reservation is not pending.');
        }

        $this->status = 'confirmed';
    }

    public function status(): string
    {
        return $this->status;
    }
}
```

The caller can request confirmation and observe status, but cannot assign `confirmed` or an invented status directly. This is encapsulation: the object owns the representation and the rules for changing it.

Encapsulation is not the same as hiding every fact. A useful public query is part of the contract. A private property with no meaningful way to observe required behavior can be just as awkward as a public property.

## How It Works

PHP checks the calling scope when code accesses a member. An attempt to access a private or protected member from an unauthorized scope raises an error. A child class has access to inherited protected members, but a private member remains private to the class that declared it:

```php
class Base
{
    private string $secret = 'base';

    protected function baseRule(): string
    {
        return $this->secret;
    }
}

final class Child extends Base
{
    public function reveal(): string
    {
        // return $this->secret; // Error: not visible here.
        return $this->baseRule();
    }
}
```

This distinction matters when inheritance is involved. A child can have a property with the same short name as a parent’s private property; that is not access to the parent’s storage. Prefer composition when a hierarchy would require fragile knowledge of protected internals. Inheritance receives fuller treatment in Chapter 28.

## What PHP Does

Visibility is part of a member’s class metadata. It applies to the access operation, not to the string name alone. Reflection can inspect or manipulate members when explicitly used, but reflection is an advanced tool and should not be used to bypass domain contracts in ordinary application code.

Class constants can be public, protected, or private:

```php
final class ReservationStatus
{
    public const PENDING = 'pending';
    private const ALL = [self::PENDING, 'confirmed', 'cancelled'];

    public static function isValid(string $status): bool
    {
        return in_array($status, self::ALL, true);
    }
}
```

A public constant is a stable name for consumers; a private constant supports implementation. Do not expose a constant merely because a test needs it.

## Minimal Example

An account controls balance changes:

```php
final class Wallet
{
    private int $cents = 0;

    public function add(int $cents): void
    {
        if ($cents < 0) {
            throw new InvalidArgumentException('Amount cannot be negative.');
        }

        $this->cents += $cents;
    }

    public function cents(): int
    {
        return $this->cents;
    }
}
```

A caller can use a valid operation without knowing whether the wallet stores integer cents, a `Money` object, or a ledger reference. That representation can change without changing the public method contract.

## Practical Example

Protected members can be an intentional extension seam, but they couple subclasses to internals:

```php
abstract class Notification
{
    final public function deliver(): void
    {
        $payload = $this->buildPayload();
        $this->send($payload);
    }

    abstract protected function buildPayload(): array;

    protected function send(array $payload): void
    {
        // Transport boundary.
    }
}
```

Here `deliver()` is public and fixed, while subclasses provide a protected payload implementation. This may be reasonable in a closed hierarchy. In an application with many independent integrations, an interface plus composition often gives a cleaner boundary because implementations do not need access to base-class state.

## Production Example: PHP 8.4 Property Visibility

Modern PHP can express different read and write visibility for typed properties:

```php
final class ImportReport
{
    public private(set) int $processed = 0;

    public function recordOne(): void
    {
        $this->processed++;
    }
}

$report = new ImportReport();
$report->recordOne();
assert($report->processed === 1);
// $report->processed = 100; // Error: private(set) is not writable here.
```

Asymmetric visibility was added in PHP 8.4. `public private(set)` means callers can read the property but only the declaring class can write it. The setter visibility must be equal to or more restrictive than the getter visibility, and the property must be typed. This is a useful middle ground for simple state that should be observable directly but controlled on writes.

Use it when direct property reads are genuinely clearer than a query method. It does not validate arbitrary writes inside the class, make a nested mutable value immutable, or authorize the caller. Property hooks, also available in PHP 8.4, can add behavior at the property boundary; keep expensive I/O out of such access syntax.

## Bad Example

```php
final class User
{
    public string $role;
    public bool $isAdmin;
}

$user->role = 'admin';
$user->isAdmin = true;
```

The representation has contradictory states: `role` and `isAdmin` can disagree. Every caller can create privilege-looking data without authorization or audit.

## Better Example

Keep authorization decisions in a controlled API:

```php
final class User
{
    public function __construct(private string $role = 'member')
    {
        if (!in_array($role, ['member', 'admin'], true)) {
            throw new InvalidArgumentException('Unknown role.');
        }
    }

    public function isAdmin(): bool
    {
        return $this->role === 'admin';
    }

    public function promoteBy(Actor $actor): void
    {
        if (!$actor->isAllowedToPromoteUsers()) {
            throw new DomainException('Actor is not allowed to promote users.');
        }

        $this->role = 'admin';
    }
}
```

Visibility protects the `role` property, while the method adds a domain check. The application must still persist the result transactionally and ensure the actor’s permissions are current.

## Edge Cases

- A method declared without a visibility modifier is public for compatibility; explicit modifiers are preferred.
- Private methods are not overridden in the same way as protected/public methods. A child method with the same name is a separate member from the parent’s private method.
- Widening visibility can be part of a compatible inheritance design, but narrowing an inherited public/protected contract is generally not allowed.
- `protected` is not “private plus subclasses forever.” Any subclass can depend on it, making future changes harder.
- `private(set)` is a PHP 8.4 feature and should not be used in a codebase whose supported minimum PHP version is older.
- Visibility is not an authorization mechanism. Authorization belongs to application/domain policy and must be checked for each operation.

## Performance

Visibility checks are not a reason to make everything public. The runtime cost is normally negligible next to I/O and allocation, while the maintenance benefit of a narrow API is substantial. Avoid reflection-based member access in hot paths unless measured and justified.

## Security

Private properties do not encrypt secrets and protected methods do not restrict an attacker who controls the process. Do not log objects indiscriminately. Keep authorization methods explicit and pass actor/resource context rather than relying on a caller reaching a private field.

## Testing

Test public behavior and failure at the boundary:

```php
function test_wallet_cannot_be_decreased_through_public_api(): void
{
    $wallet = new Wallet();
    $wallet->add(500);

    try {
        $wallet->add(-1);
        throw new RuntimeException('Expected InvalidArgumentException.');
    } catch (InvalidArgumentException) {
        assert($wallet->cents() === 500);
    }
}

function test_property_can_be_read_but_not_written_externally(): void
{
    $report = new ImportReport();
    $report->recordOne();

    assert($report->processed === 1);
    // In a PHP 8.4+ test, assert that external assignment throws an Error.
}
```

A test that reaches into private state with reflection is testing the representation. Prefer tests that would remain valid if the class changes from a scalar to a value object or ledger.

## Common Mistakes

- Making all properties public “for simplicity.”
- Using protected state as a convenient communication channel between a base class and many subclasses.
- Calling private members a security boundary.
- Exposing both a mutable property and a method that is supposed to control it.
- Using asymmetric visibility without checking the project’s PHP minimum version.
- Testing private implementation details instead of public behavior.

## Senior Engineer Thinking

For each public member, ask whether you are willing to support it for every caller and future version. For each protected member, count how many subclasses depend on it; that is your actual extension API. For each private member, identify the public operation that preserves its invariant.

Visibility is a change-management tool. A narrow public surface reduces the number of contracts you must preserve, while a deliberately designed extension surface makes future substitution possible without exposing all internals.

## Exercises

1. Refactor the public `User` fields into a controlled role API. Add authorization and persistence failure cases to the design.
2. Build a closed `Notification` hierarchy using protected extension points, then rewrite it using composition. Compare coupling and test setup.
3. On PHP 8.4+, convert a read-only counter property to `public private(set)` and test external read/write behavior.
4. Inventory the public methods of a class in your codebase. Mark each as observation, state transition, or external side effect, and identify ambiguous names.

## Review Questions

1. What promise does a public member make?
2. How do protected and private differ in an inheritance hierarchy?
3. Why is protected state expensive as an extension mechanism?
4. What problem does `public private(set)` solve, and what does it not solve?
5. Why is visibility not authorization or encryption?

## Summary

Visibility defines who may depend on a class member. Public members are APIs, protected members are inheritance seams, and private members are implementation details of their declaring class. Hide representation behind intention-revealing operations, use protected deliberately, and consider PHP 8.4 asymmetric visibility for typed state that is readable but class-controlled on writes. Constructors are the next boundary: they determine which object states may exist at all.
