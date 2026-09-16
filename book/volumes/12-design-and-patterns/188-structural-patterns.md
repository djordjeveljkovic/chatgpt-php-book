---
book: The Complete Modern PHP Engineering Book
volume: 12
volume_title: DESIGN AND PATTERNS
chapter: 188
title: Structural Patterns
slug: structural-patterns
status: complete
summary: ../../_ai/chapter-summaries/188-structural-patterns-summary.md
---

# Chapter 188 — Structural Patterns

Structural patterns organize objects so existing components can work together. They address incompatible interfaces, cross-cutting behavior, simplified entry points, and variable object composition.

The useful question is not “which pattern name applies?” It is “which boundary or composition problem is making change expensive?” An adapter can isolate a vendor API, a decorator can add policy without changing a core implementation, and a facade can give a caller one safe operation. Each also adds a layer that needs ownership and tests.

## Why this matters

A third-party mail client may expose `deliver(array $message): void`, while the application needs `sendInvoice(Invoice $invoice): DeliveryResult`. Calling the vendor everywhere spreads its types and failure behavior. An adapter keeps that translation in one place.

```php
<?php

declare(strict_types=1);

interface InvoiceMailer
{
    public function send(Invoice $invoice): DeliveryResult;
}

final class VendorMailerAdapter implements InvoiceMailer
{
    public function __construct(private VendorMailer $vendor)
    {
    }

    public function send(Invoice $invoice): DeliveryResult
    {
        $this->vendor->deliver([
            'to' => $invoice->recipient,
            'subject' => 'Invoice ' . $invoice->id,
            'body' => $invoice->renderedBody,
        ]);

        return DeliveryResult::accepted();
    }
}
```

The adapter maps domain data to vendor data and maps vendor failures to application results or exceptions. It should not silently discard a provider rejection.

## Decorators

A decorator wraps an object that implements the same interface and adds behavior such as metrics, authorization, caching, retries, or tracing:

```php
final class MeteredInvoiceMailer implements InvoiceMailer
{
    public function __construct(
        private InvoiceMailer $inner,
        private Metrics $metrics,
    ) {
    }

    public function send(Invoice $invoice): DeliveryResult
    {
        $start = hrtime(true);

        try {
            $result = $this->inner->send($invoice);
            $this->metrics->increment('invoice_mail.success');

            return $result;
        } catch (Throwable $exception) {
            $this->metrics->increment('invoice_mail.failure');
            throw $exception;
        } finally {
            $this->metrics->observe(
                'invoice_mail.duration_ms',
                (hrtime(true) - $start) / 1_000_000,
            );
        }
    }
}
```

Decorators should preserve the wrapped contract. A retry decorator must know whether sending is idempotent, which failures are retryable, and how to bound attempts and delay. A cache decorator must define freshness and invalidation. Adding a wrapper does not remove those design obligations.

Compose decorators at the composition root:

```php
$mailer = new VendorMailerAdapter($vendorMailer);
$mailer = new RetryingInvoiceMailer($mailer, $clock, $retryPolicy);
$mailer = new MeteredInvoiceMailer($mailer, $metrics);
$service = new InvoiceService($mailer);
```

The order matters. Metrics around retries may count attempts or logical operations depending on where the wrapper is placed. Name and test the chosen policy.

## Facades

A facade offers a small entry point to a subsystem. It is useful when a caller should not coordinate several components or know their order:

```php
final class CheckoutFacade
{
    public function __construct(
        private CartRepository $carts,
        private PaymentAuthorizer $payments,
        private OrderRepository $orders,
    ) {
    }

    public function checkout(int $cartId, int $customerId, string $requestKey): Order
    {
        $cart = $this->carts->ownedBy($cartId, $customerId);
        if ($cart === null) {
            throw new DomainException('Cart unavailable');
        }

        $authorization = $this->payments->authorize($cart->total(), $requestKey);
        return $this->orders->createFromAuthorizedCart($cart, $authorization);
    }
}
```

A facade should coordinate; it should not become a new god class. When the workflow gains independent policies, extract domain services or application commands with focused contracts. Keep authorization, idempotency, and transaction boundaries explicit rather than hiding them in a convenience method.

## Composite and proxy

A composite presents a group of objects through the same interface as one object. A notification service can send to several channels, but it must define whether one failure stops all delivery, whether order matters, and how results are aggregated:

```php
final class CompositeNotifier implements Notifier
{
    /** @param list<Notifier> $notifiers */
    public function __construct(private array $notifiers)
    {
    }

    public function notify(Notification $notification): void
    {
        foreach ($this->notifiers as $notifier) {
            $notifier->notify($notification);
        }
    }
}
```

A proxy controls access to another object, often for lazy loading, authorization, caching, or a remote boundary. A proxy can make a local call perform I/O, so document latency and failure. Do not let lazy loading create hidden N+1 queries in a loop.

## Structural patterns and type safety

Keep wrappers on narrow interfaces. If a decorator must inspect every implementation-specific method, the interface is too broad or the decorator belongs elsewhere. Use final classes when extension would break invariants. PHP's type declarations and readonly value objects can make structure explicit without introducing a pattern class for every field.

Prefer composition over inheritance when the behavior can be assembled. Inheritance creates a stronger coupling to protected state and lifecycle; a decorator or adapter can often change independently. See [Chapter 183 — KISS](./183-kiss.md) and [Chapter 184 — YAGNI](./184-yagni.md) when deciding whether a layer earns its cost.

## Testing and operations

Test adapters against the vendor contract, decorators against pass-through and added policy, facades against workflow outcomes, and composites against failure aggregation. Use a spy or fake to inspect calls without coupling every test to the wrapper chain. Add integration tests for serialization, network behavior, transactions, and retries.

Instrument at stable boundaries. A decorator that adds tracing should preserve correlation context and avoid recording tokens or personal data. A retry decorator should expose attempt count and final outcome. A proxy should expose cache hit/miss or remote latency so its hidden cost is observable.

## Exercises

1. Wrap a payment adapter with metrics and bounded retry. State which failures are retryable and how idempotency is preserved.
2. Build an adapter around a legacy client and write a contract test for the application interface.
3. Review a facade with eight dependencies. Split one independent policy into a focused collaborator.
4. Define failure aggregation for a composite notification sender and test partial delivery.

## Review questions

- When does an adapter protect the application from a vendor API?
- Why must decorator ordering be explicit?
- How can a facade become a god class?
- Which semantics must a composite define when one child fails?
- Why should proxies document hidden I/O and latency?

## Summary

Use adapters to translate boundaries, decorators to compose policy, facades to coordinate workflows, and composites or proxies when their semantics are explicit. Keep interfaces narrow, preserve contracts, make retries and failures visible, and observe the operational cost of every layer.

## References

- [PHP manual: Interfaces](https://www.php.net/manual/en/language.oop5.interfaces.php)
- [PHP manual: Object composition](https://www.php.net/manual/en/language.oop5.basic.php)
- [Refactoring.Guru: Structural Design Patterns](https://refactoring.guru/design-patterns/structural-patterns)
- [Martin Fowler: Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)
