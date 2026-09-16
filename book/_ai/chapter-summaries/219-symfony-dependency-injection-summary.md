# AI Summary — Chapter 219 — Symfony Dependency Injection

- Status: complete
- Volume: Volume 14 — LARAVEL AND SYMFONY
- Last updated: 2026-09-16

## Written material

Explains Symfony service definitions, autowiring, aliases, scalar configuration, private and lazy services, lifetimes, factories, decorators, compiler passes, container tests, and deployment failures.

## Concepts already explained

The container is a composition and validation tool. Constructor injection remains the application contract; autowiring does not decide business policy or provider health. Shared mutable request state is unsafe, especially in long-running workers.

## Terminology established

Service container, autowiring, autoconfiguration, alias, private service, lazy service, decorator, compiler pass, composition boundary.

## Examples used

An invoice sender, YAML service configuration and alias, a container smoke test, and service-lifetime guidance.

## Cross-references

- [Chapter 185 — Dependency Injection](../../volumes/12-design-and-patterns/185-dependency-injection.md)
- [Chapter 220 — Symfony HttpKernel](../../volumes/14-laravel-and-symfony/220-symfony-httpkernel.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 220 — Symfony HttpKernel: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated Volume XIV proofread. Symfony container guidance links to official documentation.
