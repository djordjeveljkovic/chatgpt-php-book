# AI Summary — Chapter 45 — Opcodes

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 45 explains opcodes as version-specific Zend VM instructions in op arrays. It covers opcode operands and types, constants/compiled variables/temporaries, calls, jumps, exception metadata, illustrative instruction outlines, OPcache opcode dumps, third-party disassembler caveats, performance measurement, security, testing, exercises, and review questions.

## Concepts already explained

Opcode, `zend_op`, operand/result, `zend_op_array`, `IS_CONST`, `IS_CV`, `IS_TMP_VAR`, `IS_VAR`, VM handler, dispatch, optimizer output, opcode dump.

## Terminology established

Assignment/addition outline; function-call/echo outline; OPcache pre/post-optimization diagnostic command; profiler hypothesis workflow; opcode snapshot testing guidance.

## Examples used

None.

## Cross-references

Cross-references Chapters 40 and 44 for the pipeline/compiler, and future chapters on the Zend VM, zvals, references, calls, and OPcache. References include current php-src opcode/compiler/VM files, contributor guide, and OPcache manual.

## Open threads

No open chapter-writing thread remains; Chapter 46 can explain how the VM dispatches these instructions.

## Exact next section

None — chapter complete.

## Technical verification notes

Opcode names, IDs, operand modes, and `zend_op` fields are grounded in current php-src `Zend/zend_vm_opcodes.h` and `Zend/zend_compile.h` and explicitly treated as build/version-specific. OPcache dump flags are taken from the PHP Manual. Opcode diagrams are illustrative, not claimed dumps.

## Writing notes

Keep this summary short and never present private opcode sequences as application-level guarantees.
