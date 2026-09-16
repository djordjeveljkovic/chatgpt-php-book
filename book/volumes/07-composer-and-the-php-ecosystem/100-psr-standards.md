---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 100
title: PSR Standards
slug: psr-standards
status: complete
summary: ../../_ai/chapter-summaries/100-psr-standards-summary.md
---

# Chapter 100 — PSR Standards

## Why This Matters

PHP packages are often written and released by teams that do not share an application or framework. A logger, HTTP client, cache, event dispatcher, or clock may be supplied by one package and consumed by another. When both sides agree on a stable boundary, a library can work with more implementations and an application can replace infrastructure with less adapter code.

PHP-FIG publishes recommendations for some of these shared boundaries. The catalog is larger than the few standards most PHP developers encounter every day, and its status labels matter: some documents are accepted, some are still drafts, some have been deprecated or abandoned, and one evolving recommendation is designed to receive revisions. Knowing how to read that catalog helps a team choose an appropriate contract without assuming that every numbered PSR is current, compulsory, or suitable for a particular application.

[Chapter 99](099-php-fig.md) explains PHP-FIG's governance and how its documents are developed. [Chapter 98](098-psr-4.md) explains PSR-4's name-to-path contract in detail. This chapter is a practical tour of the standards by domain; it does not repeat the governance process or PSR-4 mapping rules.

## Mental Model

A PSR is best understood as an agreement at a boundary, not as a feature PHP turns on:

```text
provider implementation  ── implements ──>  shared contract
consumer library          ── depends on ──>  shared contract
application               ── chooses ─────>  provider and policy
```

For example, a library that accepts a PSR logger can send structured records to different logging implementations. The interface does not choose where those records are stored, what retention rules apply, or which data is safe to record. Those decisions remain with the application and implementation.

An accepted standard is a stable published target, but adoption remains voluntary. A draft can change. A deprecated recommendation should not be selected for new work when its maintained successor fits. An evolving recommendation is released in versions and can change over time. Before making a package boundary depend on any of them, check the current PHP-FIG index and the exact document version.

## The Current Catalog

The status list below was checked against the official PHP-FIG [PSR index](https://www.php-fig.org/psr/) and [PER index](https://www.php-fig.org/per/) on 2026-09-15. Statuses can change, so use those indexes as the live source when starting or revising an integration.

### Accepted PSRs by Domain

| Domain | Recommendation | What it standardizes and where it helps |
| --- | --- | --- |
| Basic coding conventions | [PSR-1: Basic Coding Standard](https://www.php-fig.org/psr/psr-1/) — Accepted | A small baseline for shared PHP source, including file and naming conventions. Useful for libraries and shared code; it is not a linter or a complete project style guide. |
| Extended code style | [PSR-12: Extended Coding Style Guide](https://www.php-fig.org/psr/psr-12/) — Accepted | A fuller formatting guide that extends PSR-1 and replaces PSR-2. Teams can use it as a common style target. Formatter configuration and enforcement belong with the tooling in [Chapter 102](102-formatting.md). |
| Autoloading | [PSR-4: Autoloader](https://www.php-fig.org/psr/psr-4/) — Accepted | Defines an interoperable class-name-to-file convention. Composer and other loaders implement it; detailed namespace and path rules are in [Chapter 98](098-psr-4.md). |
| Logging | [PSR-3: Logger Interface](https://www.php-fig.org/psr/psr-3/) — Accepted | Lets a library write messages and structured context through a common logger interface. It does not dictate the backend, persistence, sampling, or application redaction policy. |
| Cache contracts | [PSR-6: Caching Interface](https://www.php-fig.org/psr/psr-6/) and [PSR-16: Simple Cache](https://www.php-fig.org/psr/psr-16/) — Accepted | PSR-6 offers item and pool objects with explicit hit state; PSR-16 offers a smaller key/value API. Both ease cache-provider substitution, but their semantics are not identical. |
| Dependency containers | [PSR-11: Container Interface](https://www.php-fig.org/psr/psr-11/) — Accepted | Standardizes the basic `get()` and `has()` boundary for retrieving entries. It does not define autowiring, service registration, scopes, or an application's dependency-injection design. |
| HTTP messages and links | [PSR-7: HTTP Message Interface](https://www.php-fig.org/psr/psr-7/) and [PSR-13: Link Definition Interfaces](https://www.php-fig.org/psr/psr-13/) — Accepted | PSR-7 describes common request, response, URI, stream, and uploaded-file interfaces. PSR-13 represents hypermedia links separately from the format that serializes them. Neither standard defines an entire web framework or API format. |
| HTTP server pipeline | [PSR-15: HTTP Server Request Handlers](https://www.php-fig.org/psr/psr-15/) and [PSR-17: HTTP Factories](https://www.php-fig.org/psr/psr-17/) — Accepted | PSR-15 provides handler and middleware interfaces; PSR-17 provides factories for PSR-7 message objects. Together they help server-side components compose without depending on one message implementation. |
| Outbound HTTP | [PSR-18: HTTP Client](https://www.php-fig.org/psr/psr-18/) — Accepted | Lets a library send PSR-7 requests through different clients. A well-formed 4xx or 5xx response is a response, not a transport exception under this contract; callers still need application-specific status handling, timeouts, and retry policy. |
| Events | [PSR-14: Event Dispatcher](https://www.php-fig.org/psr/psr-14/) — Accepted | Defines interfaces for dispatching object events and selecting listeners. It is an in-process collaboration contract, not a durable message broker or a promise of asynchronous delivery. |
| Time | [PSR-20: Clock](https://www.php-fig.org/psr/psr-20/) — Accepted | Provides a small clock interface returning `DateTimeImmutable`, which makes current-time-dependent code easier to test with a controlled clock. It is a wall-clock abstraction, not a monotonic elapsed-time timer. |

The breadth of this list is intentional. A project can use one contract without adopting all the others. A library that only needs to log may depend on PSR-3 and remain unaware of the application's HTTP stack, cache, and container.

### Draft, Deprecated, Abandoned, and Evolving Work

The official index currently lists these PSRs as Draft: [PSR-5: PHPDoc Standard](https://github.com/php-fig/fig-standards/blob/master/proposed/phpdoc.md), [PSR-19: PHPDoc tags](https://github.com/php-fig/fig-standards/blob/master/proposed/phpdoc-tags.md), [PSR-21: Internationalization](https://github.com/php-fig/fig-standards/blob/master/proposed/internationalization.md), and [PSR-22: Application Tracing](https://github.com/php-fig/fig-standards/blob/master/proposed/tracing.md). Draft documents can still change. They may be useful to follow or test, but a library should be explicit if its public API relies on one and should not describe a draft as an accepted PSR.

The index marks [PSR-0](https://www.php-fig.org/psr/psr-0/) and [PSR-2](https://www.php-fig.org/psr/psr-2/) Deprecated. PSR-4 is the modern autoloading recommendation, and PSR-12 replaces PSR-2's extended style guidance. These older documents can explain legacy package layouts and existing code, but are poor defaults for new shared code.

The index marks [PSR-8: Huggable Interface](https://github.com/php-fig/fig-standards/tree/master/proposed/psr-8-hug), [PSR-9: Security Advisories](https://github.com/php-fig/fig-standards/blob/master/proposed/security-disclosure-publication.md), and [PSR-10: Security Reporting Process](https://github.com/php-fig/fig-standards/blob/master/proposed/security-reporting-process.md) Abandoned. They are not current accepted security standards. Do not infer that a security process is established just because a numbered proposal once addressed it; choose and document an actively maintained project or ecosystem process for vulnerability intake and disclosure.

PHP-FIG also publishes PHP Evolving Recommendations (PERs). At the checked date, the [Coding Style PER](https://www.php-fig.org/per/coding-style/) is the active evolving style guide, with release 3.1 published. Its text extends and replaces PSR-12 while requiring PSR-1. A PER's versioned releases may evolve; if a team adopts one, it should pin the formatter or ruleset version and decide when to review updates. Teams that need a stable, widely supported code-style target can continue to use accepted PSR-12. The right choice depends on tool support and repository policy, not on assuming that newer text automatically fits every codebase.

Chapter 99 explains the PHP-FIG lifecycle and status vocabulary in more detail. Here, the practical rule is simple: distinguish the document's present status from its number and subject. A proposal from several years ago is not necessarily more stable than a newer recommendation, and accepted does not mean PHP itself enforces compliance.

## How to Choose a Contract

Start with a real package boundary. Identify what the consumer needs from a provider, which behavior must remain consistent, and how often the implementation might change. A standard is useful when it reduces that coordination burden without forcing the application to depend on details it does not need.

Then read the entire specification, not just its interface declarations. Normative prose may define cache misses, HTTP exceptions, event listener behavior, or logging context. Those rules are what let independent implementations interoperate. Also record what the standard leaves open: a PSR-18 client does not set an application's retry budget, and a PSR-3 logger does not decide whether a particular field contains personal data.

Finally check the package and runtime boundary. Confirm the status and version of the standard, the PHP versions supported by its interface package, the implementation package's maintenance state, and any transitive dependencies. Composer declares and locks these packages; [Chapters 94](094-composer-json.md) and [95](095-composer-lock.md) cover that workflow. If a framework already provides a compatible implementation, depend on the interface package your public API actually exposes rather than importing framework-specific classes by accident.

## Example: Stable Logger and Clock Boundaries

An application service can depend on two small contracts without constructing their implementations:

```php
<?php

declare(strict_types=1);

namespace Acme\Billing;

use Psr\Clock\ClockInterface;
use Psr\Log\LoggerInterface;

final class InvoiceAudit
{
    public function __construct(
        private LoggerInterface $logger,
        private ClockInterface $clock,
    ) {
    }

    public function issued(string $invoiceId): void
    {
        $this->logger->info('Invoice issued', [
            'invoice_id' => $invoiceId,
            'occurred_at' => $this->clock->now()->format(DATE_ATOM),
        ]);
    }
}
```

The application composition root supplies the chosen logger and clock. A test can supply a recording logger and a fixed clock, then assert the message, invoice identifier, and timestamp without waiting for real time or writing to a production log backend. The service stays decoupled from the logger vendor and clock implementation.

This contract does not solve every audit requirement. It does not guarantee that the logger persists the entry, that the two fields are sufficient for legal retention, or that the invoice identifier is safe to expose. If audit delivery is mandatory, the application needs an explicit durable write and failure policy. The standard narrows the implementation boundary; it does not replace a reliability or security design.

## Testing and Maintenance

Test behavior at the point where packages meet. For a consumer, use a small fake implementation to verify that it calls the expected methods and handles the standardized results or exceptions. For a provider, use the interface package's conformance tests when available, then add tests for implementation-specific storage, retries, lifecycle, or performance. A matching type declaration alone does not establish behavioral compatibility.

Keep tests focused on the standards that affect the package API. For example:

- A PSR-3 consumer test can inspect the level, message, and context passed to a recording logger.
- A PSR-6 consumer should distinguish a cached `null` from a cache miss through the item's hit state; a PSR-16 consumer must not assume that distinction is available from `get()` alone.
- A PSR-18 client integration should verify that HTTP error status codes remain ordinary responses and that network failures follow the client's documented exception contract.
- A PSR-20-dependent service should use a fixed clock and test boundary timestamps deterministically.
- A PSR-15 middleware test should cover both a generated response and delegation to the next handler.

When a dependency changes, review both the Composer diff and the behavioral contract. An implementation package may claim conformance while changing supported PHP versions, exceptions, or optional capabilities. Run tests on the actual framework and PHP versions the application supports. A static analyzer can catch type incompatibilities, but it cannot prove that a cache is shared between workers or that an HTTP retry is safe; see [Chapter 101](101-static-analysis.md) for the analysis boundary.

## Common Mistakes

- Treating all PSR-numbered documents as accepted, current, or mandatory.
- Depending on Draft prose in a public package while describing it as a stable standard.
- Choosing deprecated PSR-0 or PSR-2 conventions for new code without a compatibility reason.
- Confusing the evolving Coding Style PER with an accepted PSR, or assuming all formatters implement it identically.
- Treating PSR-11 as a complete dependency-injection container specification.
- Assuming PSR-6 and PSR-16 have identical cache-miss and null-value behavior.
- Treating PSR-7 message interfaces as a full HTTP server pipeline or HTTP client.
- Assuming PSR-18 handles HTTP 4xx/5xx responses as exceptions.
- Treating PSR-14 event dispatch as durable, distributed, or necessarily asynchronous.
- Using PSR-20 as a monotonic timer for measuring durations.
- Assuming an interface match guarantees operational reliability, security, or provider behavior.
- Using a static-analysis pass or formatter run as the only evidence of behavioral conformance.

## Exercises

1. Pick two accepted standards from different domains. For each, name a consumer and provider in a real application and state the behavior that must be shared for substitution to work.
2. Write a short decision note for a library deciding between PSR-6 and PSR-16. Include how callers distinguish a missing value from a cached `null`, and whether the library needs pool/item operations.
3. Test a PSR-18 client with a successful response, a 404 response, and a simulated network timeout. Record which cases return a response and which use standardized exceptions.
4. Find a package whose public constructor accepts one of these standard interfaces. Inspect its Composer requirements, PHP version constraints, tests, and changelog. Determine whether the interface dependency is part of its public API.
5. Compare PSR-12 with the current Coding Style PER. Decide which one your team would enforce today, how the tool configuration would be pinned, and how you would review a future style update.
6. Choose one Draft or Abandoned proposal from the official index. Explain why its status changes whether you would use it as a public compatibility contract.
7. Design a fixed-clock test for code that expires a token at a deadline. Explain why substituting a clock is more deterministic than sleeping in a test.

## Review Questions

1. What problem do PHP-FIG standards solve at a provider-consumer boundary?
2. How does an accepted PSR differ from a Draft, a deprecated PSR, and an evolving PER?
3. Which accepted standards cover logging, caches, containers, and time?
4. How do PSR-6 and PSR-16 differ in cache abstraction and hit/miss visibility?
5. Which standards compose an HTTP message, server middleware, factory, and outbound-client stack?
6. What does PSR-18 say about valid 4xx and 5xx responses?
7. What does PSR-11 standardize, and which container features does it leave unspecified?
8. Why should PSR-20 not be used to measure elapsed durations?
9. Why are matching interface signatures insufficient to prove an implementation is safe to substitute?
10. What evidence should accompany adoption of a PSR in a library's public API?

## Summary

PHP-FIG's accepted PSRs cover shared coding conventions, autoloading, logging, caches, containers, HTTP messages and clients, events, hypermedia links, and clocks. The official index also distinguishes Draft, Deprecated, and Abandoned proposals; the Coding Style PER is an evolving alternative to PSR-12. Check current status and document version before depending on a contract.

Use a standard when it removes friction at a real package boundary. Read its behavioral rules, confirm runtime and Composer compatibility, and test both consumer and provider behavior. A PSR can make components easier to replace, but it does not choose application policy or guarantee that an implementation is durable, secure, fast, or operationally correct.

## References

- [PHP-FIG: PSR index and current statuses](https://www.php-fig.org/psr/)
- [PHP-FIG: PER index and releases](https://www.php-fig.org/per/)
- [PHP-FIG: PSR-1 — Basic Coding Standard](https://www.php-fig.org/psr/psr-1/)
- [PHP-FIG: PSR-3 — Logger Interface](https://www.php-fig.org/psr/psr-3/)
- [PHP-FIG: PSR-4 — Autoloader](https://www.php-fig.org/psr/psr-4/)
- [PHP-FIG: PSR-6 — Caching Interface](https://www.php-fig.org/psr/psr-6/)
- [PHP-FIG: PSR-7 — HTTP Message Interface](https://www.php-fig.org/psr/psr-7/)
- [PHP-FIG: PSR-11 — Container Interface](https://www.php-fig.org/psr/psr-11/)
- [PHP-FIG: PSR-12 — Extended Coding Style Guide](https://www.php-fig.org/psr/psr-12/)
- [PHP-FIG: PSR-13 — Link Definition Interfaces](https://www.php-fig.org/psr/psr-13/)
- [PHP-FIG: PSR-14 — Event Dispatcher](https://www.php-fig.org/psr/psr-14/)
- [PHP-FIG: PSR-15 — HTTP Server Request Handlers](https://www.php-fig.org/psr/psr-15/)
- [PHP-FIG: PSR-16 — Simple Cache](https://www.php-fig.org/psr/psr-16/)
- [PHP-FIG: PSR-17 — HTTP Factories](https://www.php-fig.org/psr/psr-17/)
- [PHP-FIG: PSR-18 — HTTP Client](https://www.php-fig.org/psr/psr-18/)
- [PHP-FIG: PSR-20 — Clock](https://www.php-fig.org/psr/psr-20/)
- [PHP-FIG: PER Coding Style](https://www.php-fig.org/per/coding-style/)
- [RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels](https://www.rfc-editor.org/rfc/rfc2119)
- [Chapter 94 — `composer.json`](094-composer-json.md)
- [Chapter 95 — `composer.lock`](095-composer-lock.md)
- [Chapter 98 — PSR-4](098-psr-4.md)
- [Chapter 99 — PHP-FIG](099-php-fig.md)
- [Chapter 101 — Static Analysis](101-static-analysis.md)
- [Chapter 102 — Formatting](102-formatting.md)
