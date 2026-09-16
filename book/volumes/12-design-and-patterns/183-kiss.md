---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 183
title: KISS
slug: kiss
status: complete
summary: ../../_ai/chapter-summaries/183-kiss-summary.md
---

# Chapter 183 — KISS

## Why This Matters

KISS means “Keep It Simple, Stupid,” usually expressed more usefully as “keep the design as simple as the requirements allow.” Simplicity reduces the number of states a system can enter, shortens the path from a failure to its cause, and lowers the cost of onboarding and change.

Simple does not mean short, naïve, or under-engineered. A transaction, authorization check, retry limit, or validation rule may add necessary complexity. KISS asks whether each part of the design earns its complexity and whether a reader can understand its behavior under failure.

## Define the Necessary Behavior

Write the current contract before choosing an abstraction. A function that maps a known status to a label can be a direct match; a plugin registry, reflection scan, and container configuration add moving parts without solving a requirement.

~~~php
<?php

declare(strict_types=1);

function displayStatus(string $status): string
{
    return match ($status) {
        'pending' => 'Awaiting confirmation',
        'confirmed' => 'Confirmed',
        'cancelled' => 'Cancelled',
        default => 'Unknown',
    };
}
~~~

If status labels become tenant-configurable or are supplied by a translation service, the requirements change and an abstraction may be justified. Keep the initial design explicit until the variation is real.

## Complexity Is a Budget

Complexity appears in code paths, dependencies, configuration, deployment steps, data states, and operational procedures. A five-line helper can have more complexity than a larger function if it hides reflection, global state, or a network call. Evaluate:

- how many states and branches exist;
- how many collaborators and configuration values are required;
- what can fail and how it is recovered;
- the runtime cost in CPU, memory, queries, and connections;
- how an operator diagnoses and rolls back the behavior.

A design with one extra class can be simpler if it isolates a volatile provider. A design with ten interfaces can be harder to operate if every deployment must configure them differently. Simplicity is judged at the system boundary, not by a source-file line count.

## Prefer Explicit Data Flow

Explicit parameters and return types make dependencies visible. Avoid hidden reads from globals, environment variables, static registries, and ambient request state inside domain code. Dependency injection can simplify reasoning even when it introduces a constructor parameter.

~~~php
<?php

declare(strict_types=1);

final readonly class TaxCalculator
{
    public function __construct(private int $rateBasisPoints)
    {
        if ($rateBasisPoints < 0 || $rateBasisPoints > 10_000) {
            throw new InvalidArgumentException('Invalid tax rate');
        }
    }

    public function taxFor(int $amountMinor): int
    {
        if ($amountMinor < 0) {
            throw new InvalidArgumentException('Invalid amount');
        }

        return intdiv($amountMinor * $this->rateBasisPoints + 5_000, 10_000);
    }
}
~~~

The rounding rule is visible and testable. The service does not silently read a deployment variable or global locale. If tax rules become jurisdiction-specific, add an explicit policy boundary rather than hiding a lookup in this class.

## Keep Failure Paths Understandable

A simple success path with an unclear failure path is not simple. Define timeouts, error types, retries, and partial effects at the boundary. A direct HTTP call with no timeout is shorter but can occupy workers indefinitely. A small client adapter with a timeout and a typed failure may be simpler to operate.

Avoid catch-all recovery that turns every exception into a default result. The caller should know whether to retry, ask for input, report a conflict, or alert an operator. Use structured errors and correlation IDs where they help diagnose a distributed path.

## Choose Data Structures and Algorithms Deliberately

KISS does not mean ignoring performance. A linear scan of a short list may be clearer and faster to maintain than a general index; the same scan can be an outage when the list grows to millions of records. State the expected size, query cost, and growth assumption.

Use a PHP array when its map or list behavior is clear. Introduce a specialized structure when complexity, memory, or invariants justify it. Keep database filtering and ordering in the database when that reduces transferred data and makes the query plan appropriate. Simple code that performs an unbounded query is not a simple system.

## Abstraction and Configuration

A framework, container, or pattern can simplify a repeated operational concern when the team understands its behavior. It can also move complexity into convention, generated configuration, or lifecycle hooks. Evaluate the whole path:

1. Can a new engineer locate the behavior?
2. Can a failure be reproduced locally?
3. Is the lifecycle and ownership explicit?
4. Can the component be replaced without changing domain rules?
5. Does the abstraction remove more complexity than it introduces?

Centralize configuration that must agree, but avoid a single giant configuration object passed everywhere. Narrow configuration objects make invalid combinations harder to construct.

## Testing Simplicity

Tests should exercise the contract at the smallest useful boundary. A simple pure calculation deserves a unit test; a transaction requires an integration test; a browser redirect requires an end-to-end test. Forcing every test through the browser is operationally complex and slow.

Use table-driven tests for a finite set of rules and property-based tests for broad input domains. A clear test name and failure message are part of the design. Delete helper layers that obscure the scenario or duplicate the production implementation.

## Common Mistakes

- Equating fewer lines with a simpler system.
- Hiding dependencies in globals or service locators.
- Omitting timeouts and recovery to keep an example short.
- Choosing a complex abstraction for a hypothetical variation.
- Ignoring data growth and query cost.
- Converting every failure into a default value.
- Testing a simple rule through an unnecessarily large stack.

## Senior Engineer Thinking

Keep the behavior, dependencies, data flow, and failure policy visible. Spend complexity where requirements demand it, such as authorization, transactions, concurrency, and recovery. Remove complexity that only supports speculation or hides the path an operator and maintainer must understand.

## Exercises

1. Take a configurable status system and identify which requirements justify a strategy or translation boundary.
2. Refactor a function that reads global configuration into explicit typed input.
3. Compare a PHP loop, a collection pipeline, and a database query for a growing dataset.
4. Document the timeout, retry, and failure contract for one downstream call.

## Review Questions

1. Why is simplicity more than source-file length?
2. Which complexity is necessary for a reliable system?
3. How do explicit parameters improve design?
4. Why can a default error result be more complex operationally?
5. When should an abstraction be introduced?

## Summary

KISS keeps behavior, data flow, failure policy, and operational cost understandable. Use direct code for stable requirements, add abstractions for demonstrated variation or volatile boundaries, and retain necessary complexity for security, transactions, concurrency, performance, and recovery. Judge simplicity at the system boundary and test each concern at the smallest useful layer.

## References

- [The Manifesto for Agile Software Development](https://agilemanifesto.org/)
- [PHP type declarations](https://www.php.net/manual/en/language.types.declarations.php)
- [PHP match expression](https://www.php.net/manual/en/control-structures.match.php)
- [Martin Fowler: Is Design Dead?](https://martinfowler.com/articles/designDead.html)

