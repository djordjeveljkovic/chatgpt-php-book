# AI Summary — Chapter 185 — Dependency Injection

- Status: complete
- Volume: Volume 12 — DESIGN AND PATTERNS
- Last updated: 2026-09-16

## Written material

Explains explicit dependencies, constructor injection, composition roots, containers versus service locators, lifetimes, mutable state, testing, exercises, and review questions.

## Concepts already explained

Dependency injection makes collaborators and object lifetimes explicit. Compose graphs at application boundaries, keep containers out of business logic, use interfaces at meaningful boundaries, and combine substitutable unit tests with real adapter checks.

## Terminology established

Dependency injection, hidden dependency, composition root, service locator, object lifetime, null object.

## Examples used

An invoice service with repository and mailer ports, composition-root wiring, a null mailer, and in-memory test collaborators.

## Cross-references

- [Chapter 184 — YAGNI](../../volumes/12-design-and-patterns/184-yagni.md)
- [Chapter 186 — Abstraction](../../volumes/12-design-and-patterns/186-abstraction.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 186 — Abstraction: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XII proofread. Dependency-injection guidance links to PHP-FIG and Fowler.
