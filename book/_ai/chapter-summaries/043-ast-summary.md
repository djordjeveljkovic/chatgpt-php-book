# AI Summary — Chapter 43 — AST

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 43 explains the AST as the structured intermediate representation between parsing and compilation. It covers node kinds, attributes, line information, parser/compiler/analyzer separation, PHP 7’s AST transition, lossless-source limitations, optional ext/ast distinction, evaluation order, static-analysis limits, performance, security, testing, exercises, and review questions.

## Concepts already explained

Abstract syntax tree, node kind, child, attribute, source line, AST arena/lifetime, parser/compiler separation, structural analysis, semantic/data-flow analysis, lossless source representation.

## Terminology established

Expression tree diagrams; `discountedTotal()` structure; addition versus null-coalescing side-effect example; narrow API-call analysis workflow; ext/ast compatibility exercise.

## Examples used

None.

## Cross-references

Cross-references Chapters 40–42 for the pipeline, lexer, and parser, and Chapter 44 for AST-to-op-array compilation. References include the PHP AST RFC and current php-src `Zend/zend_ast.h`/`zend_compile.c`.

## Open threads

No open chapter-writing thread remains; future internals chapters can build on the distinction between ASTs and runtime values.

## Exact next section

None — chapter complete.

## Technical verification notes

The PHP 7 AST transition is grounded in the official PHP RFC. Current internal node and compiler references use php-src headers/source and are explicitly labeled version-sensitive; ext/ast is identified as separate from the core engine AST.

## Writing notes

Keep this summary short; do not turn conceptual node diagrams into a promise about private node IDs or layouts.
