# AI Summary — Chapter 31 — Abstract Classes

- Status: complete
- Volume: Volume 3 — PHP OBJECT MODEL
- Last updated: 2026-09-14

## Written material

Complete chapter covering partially implemented parents, abstract methods, template workflows, final invariants, constructor behavior, protected compatibility, retry caveats, performance, security, testing, mistakes, exercises, and review questions.

## Concepts already explained

Abstract class, abstract method, concrete descendant, template method, extension point, shared invariant, protected compatibility surface, retry budget.

## Terminology established

Partially implemented parent, family of algorithms, dependency landfill, final workflow, non-idempotent operation.

## Examples used

`MessageHandler`; `Formatter`/`CsvFormatter`; `RetryingClient`; reservation policy template; PHPUnit formatter test.

## Cross-references

Chapter 28 inheritance; Chapter 29 composition; Chapter 30 interfaces; later `final` chapter. The outline and teaching rules are in [SKELETON.md](../../../SKELETON.md).

## Open threads

No open chapter-writing threads. Later work can connect abstract workflows to `final`, traits, and explicit policy objects.

## Exact next section

Chapter complete; next chapter is Chapter 32 — Traits.

## Technical verification notes

The chapter distinguishes PHP’s instantiation/signature rules from behavioral correctness and warns that generic retry loops do not establish idempotency.

## Writing notes

Summary synchronized with the complete chapter on 2026-09-14.
