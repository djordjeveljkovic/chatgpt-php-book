# AI Summary — Chapter 44 — Compilation

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 44 explains compilation as AST traversal into executable Zend structures, especially function/file op arrays containing opcodes and metadata. It covers compiler context, calls, branches and jumps, declarations, compile-time versus runtime knowledge, includes/eval, closures/generators/fibers, OPcache separation, performance, security, testing, exercises, and review questions.

## Concepts already explained

Compilation, `zend_op_array`, opcode buffer, literals, compiled variables, jump targets, exception regions, runtime cache, compiler context, constant folding, cold versus cached compilation.

## Terminology established

Typed `add()` example; conceptual call and branch instruction sequences; compile-time/runtime knowledge example; PHP compiler versus database optimizer diagram; cache/performance experiments.

## Examples used

None.

## Cross-references

Cross-references Chapters 40–43 for the pipeline, parser, and AST, Chapter 45 for opcode details, and Chapter 59 for OPcache. References include current php-src compiler source/header, PHP CLI options, AST RFC, and OPcache manual.

## Open threads

No open chapter-writing thread remains; later chapters cover VM execution, zvals, calls, memory, and OPcache internals.

## Exact next section

None — chapter complete.

## Technical verification notes

Compiler entry points and `zend_op_array` fields are based on current php-src `Zend/zend_compile.h` and `zend_compile.c`; instruction sequences are labeled illustrative. OPcache and lint claims reference the PHP Manual. Implementation details are version/build-sensitive.

## Writing notes

Keep this summary short and distinguish core compilation from optimizer passes, cached code, and runtime execution.
