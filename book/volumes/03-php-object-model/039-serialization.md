---
book: The Complete Modern PHP Engineering Book
volume: 3
volume_title: PHP OBJECT MODEL
chapter: 39
title: Serialization
slug: serialization
status: complete
summary: ../../_ai/chapter-summaries/039-serialization-summary.md
---

# Chapter 39 — Serialization

## Why This Matters

Serialization turns an in-memory value into a representation that can cross a process, request, or storage boundary. That representation becomes a compatibility contract. If it is PHP's native format, it also carries class names and can invoke object hooks during restoration. The convenience is real; so are the versioning and security costs.

Use native PHP serialization for controlled PHP-to-PHP infrastructure where its object graph and type fidelity are valuable. Use JSON, a schema-defined message, or another explicit format for public APIs, untrusted input, and cross-language boundaries.

## Mental Model

```text
object graph ──serialize──▶ bytes/string ──transport/storage──▶ bytes/string
     ▲                                                           │
     └────────────── validate + restore + reinitialize ◀─────────┘
```

Serialization is not encryption, signing, validation, or a database backup. It is an encoding. The receiver must decide whether it trusts the bytes, whether the class definitions are compatible, and whether the restored object is valid for the current application version.

## Minimal Example

```php
<?php

declare(strict_types=1);

final class JobData
{
    public function __construct(
        public readonly string $name,
        public readonly int $attempt,
    ) {}
}

$encoded = serialize(new JobData('send-reminder', 1));
$decoded = unserialize($encoded, [
    'allowed_classes' => [JobData::class],
    'max_depth' => 64,
]);

assert($decoded instanceof JobData);
```

The options restrict classes and nesting depth, but they do not make arbitrary attacker input safe. PHP's manual explicitly warns not to pass untrusted input to `unserialize()`; prefer JSON for that case.

## What Native Serialization Stores

`serialize()` preserves PHP values and object property state, not method implementations. For an object it records the class name and data needed to reconstruct it. The class definition must be loaded, usually through autoloading. If it is missing, PHP can produce an `__PHP_Incomplete_Class`, which is not a usable instance of the original application class.

Resources and some internal objects cannot be meaningfully serialized. Open database connections, sockets, closures, and process handles are not durable object state. Store the configuration or identifier needed to reacquire such a resource, not the resource itself.

## Custom Representation with `__serialize()`

Since PHP 7.4, a class can define `__serialize(): array` and `__unserialize(array $data): void`:

```php
final class ReservationJob
{
    public function __construct(
        private string $reservationId,
        private DateTimeImmutable $runAt,
    ) {}

    public function __serialize(): array
    {
        return [
            'version' => 1,
            'reservation_id' => $this->reservationId,
            'run_at' => $this->runAt->format(DateTimeInterface::ATOM),
        ];
    }

    public function __unserialize(array $data): void
    {
        if (($data['version'] ?? null) !== 1
            || !is_string($data['reservation_id'] ?? null)
            || !is_string($data['run_at'] ?? null)
        ) {
            throw new UnexpectedValueException('Invalid reservation job payload.');
        }

        $this->reservationId = $data['reservation_id'];
        $this->runAt = new DateTimeImmutable($data['run_at']);
    }
}
```

The array is an application-defined representation. It can omit caches, connections, derived fields, and secrets. The version marker allows an explicit migration path, though it does not remove the need to retain readers for old queued or stored payloads.

## Legacy Hooks and Compatibility

`__sleep()` and `__wakeup()` are older hooks. `Serializable` is older still; new code should use `__serialize()` and `__unserialize()`. As of PHP 8.1, a class implementing only `Serializable` produces a deprecation warning. A library supporting PHP before 7.4 may implement both mechanisms during migration, but it must test which format each supported runtime reads.

If both new and old hooks exist, the new hooks take precedence for newly serialized data. Existing legacy payloads may still use the old format. Treat serialized queues and session data as deployable schemas: deploy readers before writers when rolling out a representation change.

## Security: The Critical Boundary

Never call `unserialize()` on attacker-controlled cookies, request fields, uploaded files, or messages without a strong, justified trust model. Object restoration can instantiate classes, invoke `__unserialize()` or `__wakeup()`, autoload classes, and trigger gadget chains in vulnerable dependency graphs. `allowed_classes` is a defense-in-depth control, not permission to accept untrusted serialized data.

For controlled external storage, authenticate the bytes with an HMAC before decoding and use a strict class allow-list. An HMAC proves integrity with a secret; it does not make a malicious party with the secret harmless, and it does not solve schema compatibility. JSON plus validation is usually the safer choice for user-visible data.

## Better Boundary Example

```php
final readonly class QueueEnvelope
{
    /** @param array<string, mixed> $payload */
    public function __construct(
        public string $type,
        public int $version,
        public array $payload,
    ) {}
}

$json = json_encode([
    'type' => 'reservation.created',
    'version' => 1,
    'payload' => ['reservation_id' => 'r-1'],
], JSON_THROW_ON_ERROR);

$data = json_decode($json, true, 64, JSON_THROW_ON_ERROR);
```

JSON does not automatically restore a PHP object, which is a feature at a trust boundary. Validate `type`, `version`, and payload fields before constructing a domain object. A message consumer must also handle duplicate delivery, unknown versions, malformed data, and partial processing.

## Database and Operations

Native serialized strings are opaque to SQL. You cannot efficiently query a serialized reservation by court, date, or status. Store queryable fields in columns and use serialization only for a payload that is read as a whole. Avoid storing native serialized objects as a long-lived public schema unless you own the migration process.

For queued jobs, choose a compatibility policy: old workers may receive new messages during deployment, retries may outlive a release, and a poison payload may repeatedly fail. Include a version, make deserialization failures observable, quarantine irreparable messages, and never retry a deterministic schema error forever.

## Performance

Serialization walks the reachable value graph and allocates a representation; deserialization allocates it again and may invoke user code. Time and space are proportional to the encoded graph in the normal case, with additional costs for repeated copies and class hooks. JSON may be more interoperable but can lose PHP-specific types and has its own encoding/decoding cost. Benchmark representative payload sizes and measure queue latency, memory, and failure rates.

## Testing

Test round trips and hostile or old inputs:

```php
$job = new ReservationJob('r-1', new DateTimeImmutable('2026-09-14T12:00:00+00:00'));
$restored = unserialize(serialize($job), [
    'allowed_classes' => [ReservationJob::class],
]);

assert($restored instanceof ReservationJob);

$blocked = unserialize(serialize($job), ['allowed_classes' => false]);
assert($blocked instanceof __PHP_Incomplete_Class);
```

Also test missing fields, wrong types, unsupported versions, trailing data, malformed bytes, unavailable classes, and exceptions from hooks. Use integration tests for queue compatibility across the previous and current application versions. Never put real credentials in fixtures merely to test serialization.

## Common Mistakes

- Treating serialized PHP as encrypted or signed.
- Passing request data directly to `unserialize()`.
- Assuming `allowed_classes` makes untrusted input safe.
- Serializing database connections, caches, or resources.
- Changing private property names or class namespaces without a migration plan.
- Using opaque serialized blobs for fields the database needs to filter or index.

## Senior Engineer Thinking

Choose a representation by boundary: native serialization for controlled, homogeneous PHP internals; JSON or a schema format for APIs and heterogeneous systems. Define ownership, versioning, trust, observability, and rollback before shipping a payload. The hard problem is not producing bytes; it is keeping old bytes safe and meaningful after code changes.

## Exercises

1. Add a versioned `__serialize()` representation to a job and write a reader for version 1 and version 2.
2. Replace native serialization in a hypothetical HTTP cookie with signed JSON. List the protections each layer provides.
3. Design a queue poison-message policy for malformed serialized data, including metrics and operator recovery.

## Review Questions

1. What does native object serialization preserve and what does it not preserve?
2. Why are `__serialize()` and `__unserialize()` preferable to legacy hooks for new code?
3. Why is `allowed_classes` not a complete security solution?
4. When is JSON a better boundary format?
5. What operational problems can old queue payloads create during deployment?

## Summary

Serialization is a representation and compatibility boundary. Native PHP serialization preserves PHP-specific structure but carries class and hook behavior, so it belongs only in controlled trust domains. Use explicit custom representations, version them, validate them, authenticate stored bytes when required, and prefer schema-defined formats such as JSON for external or untrusted data.

## Official References

- [PHP Manual: Object Serialization](https://www.php.net/manual/en/language.oop5.serialization.php)
- [PHP Manual: `serialize()`](https://www.php.net/manual/en/function.serialize.php)
- [PHP Manual: `unserialize()` and security warning](https://www.php.net/manual/en/function.unserialize.php)
- [PHP Manual: Magic Methods](https://www.php.net/manual/en/language.oop5.magic.php)
- [PHP Manual: Serializable interface](https://www.php.net/manual/en/class.serializable.php)
