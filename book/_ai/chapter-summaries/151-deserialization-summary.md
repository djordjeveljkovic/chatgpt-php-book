# AI Summary — Chapter 151 — Deserialization

- Status: complete
- Volume: Volume 10 — SECURITY
- Last updated: 2026-09-16

## Written material

Explains native PHP serialization risk, data-only formats, class allow-lists, magic methods and gadget chains, integrity and replay, versioning, testing, exercises, and review questions.

## Concepts already explained

- `unserialize()` on attacker-controlled bytes can instantiate classes and invoke gadget behavior.
- `allowed_classes` is a narrow legacy control, not complete validation; JSON still needs size, depth, shape, authorization, and replay policy.
- Signing authenticates an envelope but does not make an unsafe deserializer safe.

## Terminology established

Deserialization, data-only format, class allow-list, gadget chain, magic method, authenticated envelope, replay, serialized contract.

## Examples used

- Restricted legacy `unserialize()` and bounded JSON command decoding.
- Signed/versioned cookie or queue envelope and migration inventory.

## Cross-references

- [Chapter 133 — Uploads](../../volumes/09-http-and-application-development/133-uploads.md)
- [Chapter 156 — Dependency Security](../../volumes/10-security/156-dependency-security.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 152 — Password Security: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated security proofread. Native serialization claims link to the PHP Manual and OWASP deserialization guidance.
