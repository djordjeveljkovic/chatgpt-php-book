# AI Summary — Chapter 54 — Function Calls

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Explains the conceptual call sequence; left-to-right eager argument evaluation; named/default/variadic/reference binding; arrays, objects, and references at parameter boundaries; userland versus internal functions; opcodes; `zend_execute_data` frames and VM stack; return cleanup; recursion/re-entrancy; callback and dynamic-call costs; production visibility, security, testing, exercises, and review questions.

## Concepts already explained

Argument evaluation, callable resolution, parameter binding, COW at call boundaries, object sharing, reference mutation, `execute_data`, VM stack frames, op_arrays, internal function entry, return zvals, cleanup, recursion, re-entrancy, and profiling call overhead.

## Terminology established

Call frame, `execute_data`, `zend_function`, op_array, `INIT_FCALL`, `SEND_*`, `DO_FCALL`, internal function, dynamic call, re-entrancy, return zval.

## Examples used

Evaluation-order logging, named arguments, mixed parameter value kinds, a conceptual VM stack diagram, recursive depth, hidden side-effecting arguments, explicit service-boundary preparation, callback versus direct-loop reasoning, and call-boundary tests.

## Cross-references

Links to Chapters 40–46 for source/AST/opcode/VM context, Chapters 51–53 for value boundaries, and the official function-argument manual.

## Open threads

Future chapters can connect call frames to generators, Fibers, OPcache/JIT, signals, and long-running worker stack retention.

## Exact next section

Chapter complete; no next section in this chapter.

## Technical verification notes

Argument semantics checked against the [PHP function arguments manual](https://www.php.net/manual/en/functions.arguments.php). VM claims were checked against current call-frame helpers in [`zend_execute.h`](https://github.com/php/php-src/blob/master/Zend/zend_execute.h), the C call API in [`zend_API.h`](https://github.com/php/php-src/blob/master/Zend/zend_API.h), and call opcode definitions in [`zend_vm_opcodes.h`](https://github.com/php/php-src/blob/master/Zend/zend_vm_opcodes.h). The text avoids claiming that every build executes the same unoptimized opcode path.

## Writing notes

Status is complete. The chapter treats calls as semantic and operational boundaries without recommending indiscriminate inlining or abstraction removal.
