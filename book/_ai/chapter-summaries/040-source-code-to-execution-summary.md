# AI Summary — Chapter 40 — Source Code to Execution

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 40 explains the conceptual pipeline from PHP source through lexical scanning, parsing, AST construction, compilation to Zend op arrays, and Zend VM execution. It distinguishes file compilation from execution, normal compilation from OPcache reuse, and language behavior from implementation details. It includes syntax-versus-runtime failure examples, include/eval boundaries, deployment/cache considerations, performance, security, testing guidance, exercises, and review questions.

## Concepts already explained

Source-to-execution pipeline, source/tokens/AST/opcodes, op array, Zend VM, compile-time versus runtime behavior, include/eval compilation, OPcache, syntax checking, SAPI/deployment boundaries.

## Terminology established

Source-to-execution diagram; typed `subtotal()` example; syntax-error versus runtime-exception fixtures; CLI lint/configuration commands; deployment/cache pipeline.

## Examples used

None.

## Cross-references

Cross-references Chapters 41–45 for front-end and opcode details, Chapter 59 for OPcache, and earlier Volume I discussions of execution context and lifecycle. References include the PHP Manual, AST RFC, and php-src compiler header.

## Open threads

No open chapter-writing thread remains; later chapters expand the pipeline’s individual phases.

## Exact next section

None — chapter complete.

## Technical verification notes

General behavior is grounded in the PHP command-line options and OPcache manuals, the PHP 7 AST RFC, and current php-src `Zend/zend_compile.h`. Internal layouts and cache behavior are explicitly labeled version/build-sensitive.

## Writing notes

Keep this summary short; do not infer stable opcode or AST APIs from the chapter’s conceptual diagrams.
