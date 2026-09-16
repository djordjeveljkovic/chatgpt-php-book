# AI Summary — Chapter 46 — Zend VM

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 46 explains how the Zend VM executes compiled op arrays through execution contexts, current oplines, operands, VM handlers, calls, returns, control-flow jumps, exceptions, cleanup, generators, and fibers. It distinguishes PHP-visible semantics from private executor details and uses a measured performance workflow rather than opcode counting. It includes diagrams, a typed function example, profiling guidance, security, testing, exercises, and review questions.

## Concepts already explained

Zend VM, execution context, `zend_execute_data`, opline, VM handler, dispatch, VM stack concept, userland/internal function boundary, call frame, return slot, exception transfer, suspension, OPcache/JIT caveat.

## Terminology established

Conceptual executor cycle; frame diagram; handler/dispatch distinction; call/return path; VM-versus-native-code distinction; profiler boundary workflow.

## Examples used

`lineTotal()` call/return path, `classify()` branch, report-endpoint bottleneck decomposition, and a CPU-bound positive-sum hypothesis.

## Cross-references

Builds on Chapters 40–45 and points to Chapters 47 and 48 for zvals and strings, Chapter 52 for copy-on-write, Chapter 51 for references, and later chapters for memory, calls, and OPcache. References use PHP-8.4 php-src executor/VM files and PHP manuals for errors, generators, and fibers.

## Open threads

Later chapters expand zvals, strings, HashTables, arrays, references, copy-on-write, function calls, memory management, and OPcache.

## Exact next section

None — chapter complete.

## Technical verification notes

Execution-context and handler claims are tied to PHP-8.4 `Zend/zend_execute.c`, `zend_vm_def.h`, and generated executor source, and are explicitly labeled private/version-sensitive. Dispatch diagrams and opcode shapes are illustrative; PHP semantics are separated from implementation behavior.

## Writing notes

Keep VM details diagnostic and measurement-driven. Do not present `zend_execute_data`, generated executor files, dispatch strategy, or frame offsets as stable application APIs.
