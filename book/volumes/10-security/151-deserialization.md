---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 151
title: Deserialization
slug: deserialization
status: complete
summary: ../../_ai/chapter-summaries/151-deserialization-summary.md
---

# Chapter 151 — Deserialization

## Why This Matters

Serialization turns program state into data; deserialization turns data back into values or objects. If an attacker controls serialized input, the process may instantiate unexpected classes, invoke magic methods, consume excessive resources, or trigger a gadget chain in dependent code. PHP's native `unserialize()` is therefore a dangerous boundary for untrusted bytes.

## Mental Model

```text
untrusted bytes → parse as inert data → validate shape → construct domain object
```

Prefer JSON or another data-only format for external messages. JSON does not make every payload safe—deep nesting, huge numbers, and malicious strings still require limits—but it does not invoke PHP object construction in the same way as native PHP serialization.

## Native PHP Serialization

Do not call `unserialize()` on request data, cookies, queue messages, cache values, or uploaded files unless the complete producer and class allow-list are trusted and controlled. The `allowed_classes` option limits class instantiation but does not turn malformed input into a validated domain command:

```php
$value = unserialize($payload, ['allowed_classes' => false]);
if (!is_array($value)) {
    throw new InvalidArgumentException('Invalid payload');
}
```

This can be appropriate for a tightly controlled legacy migration, with size limits and an explicit shape validator. Never use an object from the result before checking its type and fields. A class allow-list must be narrow and versioned; accepting a broad namespace is not a policy.

## Data-Only Formats and Validation

Decode JSON with exceptions and validate its shape at the boundary:

```php
<?php

declare(strict_types=1);

function decodeCommand(string $json): array
{
    if (strlen($json) > 100_000) {
        throw new InvalidArgumentException('Payload too large');
    }

    $value = json_decode($json, true, 32, JSON_THROW_ON_ERROR);
    if (!is_array($value) || !is_string($value['action'] ?? null)) {
        throw new InvalidArgumentException('Invalid command');
    }

    return $value;
}
```

The resulting array remains untrusted until authorization and domain validation. Convert it into a typed command object only after required fields, ranges, and actor permissions are established.

## Gadget Chains and Magic Methods

PHP objects can define `__wakeup()`, `__unserialize()`, `__destruct()`, and other magic methods. A vulnerable chain may be reachable without an obvious dangerous call in the entry-point file because Composer dependencies supply classes and methods. Patching dependencies and removing native deserialization of untrusted data are stronger controls than trying to enumerate every possible gadget.

Do not assume that a payload is safe because it contains no class name you recognize. Autoloading, version changes, and framework behavior can change the available gadget surface. Treat serialized blobs in caches and queues as sensitive data with an explicit trust boundary.

## Integrity, Confidentiality, and Replay

Signing or encrypting a payload can authenticate its producer and protect confidentiality, but it does not make an unsafe deserializer safe. Verify a MAC before parsing when the protocol supports it, include a version and audience, and define expiry and replay behavior. Key rotation and invalidation need an operational plan.

Never use `serialize()` output as a portable public API or a long-lived database contract without a migration policy. Class renames, private property encoding, PHP versions, and dependency changes can make it impossible or unsafe to restore.

## Testing and Common Mistakes

Test malformed, truncated, oversized, deeply nested, wrong-version, replayed, and unknown-field inputs. Scan dependencies and exercise the actual queue/cache boundary. Do not put secrets into serialized values that may appear in logs or dumps.

Common mistakes include unserializing cookies, accepting serialized job bodies from an untrusted broker, relying on `allowed_classes` as full validation, and deserializing before checking a signature or size limit.

## Senior Engineer Thinking

Make external data inert first. Use data-only formats, strict size/depth limits, schema validation, authenticated envelopes, versioning, and explicit construction. If legacy PHP serialization is unavoidable, isolate it, allow-list classes narrowly, patch dependencies, and plan its removal.

## Exercises

1. Replace a serialized cookie with a signed JSON envelope and define expiry.
2. Add shape validation and maximum depth to a JSON command.
3. Inventory cache, queue, and database values that use native PHP serialization and classify their trust boundaries.

## Review Questions

1. Why is `unserialize()` dangerous on attacker-controlled data?
2. What does `allowed_classes => false` protect and what remains to validate?
3. Why does signing not make an unsafe deserializer safe?
4. Which compatibility problems make native serialized data a poor public contract?

## Summary

Treat native PHP deserialization as a privileged operation. Prefer bounded, validated data-only formats; authenticate and version envelopes; keep untrusted values inert until validation; and isolate legacy serialized caches or messages with narrow class policies and a removal plan.

## References

- [PHP Manual: `unserialize`](https://www.php.net/manual/en/function.unserialize.php)
- [PHP Manual: Serialization](https://www.php.net/manual/en/language.oop5.serialization.php)
- [OWASP: Deserialization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html)
