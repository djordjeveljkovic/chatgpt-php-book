# AI Summary — Chapter 186 — Abstraction

- Status: complete
- Volume: Volume 12 — DESIGN AND PATTERNS
- Last updated: 2026-09-16

## Written material

Explains useful versus leaky abstractions, stable concepts, interfaces, abstract classes, value objects, ports and adapters, abstraction cost, testing, exercises, and review questions.

## Concepts already explained

An abstraction reduces caller decisions while preserving important failure and cost semantics. Keep contracts narrow and typed, translate vendor details at adapters, and use value objects for invariants rather than adding needless interfaces.

## Terminology established

Abstraction, leaky abstraction, port, adapter, value object, contract, interface segregation.

## Examples used

Fraud and payment ports, a validated `Money` value object, and a provider adapter translating external responses.

## Cross-references

- [Chapter 185 — Dependency Injection](../../volumes/12-design-and-patterns/185-dependency-injection.md)
- [Chapter 187 — Creational Patterns](../../volumes/12-design-and-patterns/187-creational-patterns.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 187 — Creational Patterns: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XII proofread. Abstraction guidance links to PHP documentation and Fowler.
