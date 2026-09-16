# AI Summary — Chapter 52 — Copy-on-Write

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Explains copy-on-write as the implementation of PHP value semantics for arrays and strings; sharing and first-write separation; parameter behavior; zval/refcount approximations; nested arrays versus nested objects; allocating transformations; in-place versus value-oriented APIs; performance and peak memory; long-running workers; security, testing, exercises, and review questions.

## Concepts already explained

Payload sharing, delayed separation, refcounted storage, first-write cost, COW across function calls, non-deep-cloning behavior, object identity, temporary allocation peaks, streaming alternatives, worker retention, and measurement boundaries.

## Terminology established

Copy-on-write (COW), payload, separation, refcounted storage, zval, first write, shallow sharing, deep cloning, peak memory.

## Examples used

String and array separation, read-only and mutating function parameters, nested arrays with a shared nested object, array/string transformations, an in-place uppercase function, a COW-preserving value function, and a COW behavior test.

## Cross-references

Links to Chapter 51 for references and Chapter 53 for object identity. Supports later memory-manager and worker chapters without changing them.

## Open threads

Future chapters can quantify allocator behavior, garbage collection, OPcache/JIT effects, and SAPI-specific resident memory.

## Exact next section

Chapter complete; no next section in this chapter.

## Technical verification notes

COW claims were checked against the [PHP implicit move optimization RFC](https://wiki.php.net/rfc/implicit_move_optimisation), current zval copy helpers in [`zend_execute.h`](https://github.com/php/php-src/blob/master/Zend/zend_execute.h), destruction/copy constructors in [`zend_variables.h`](https://github.com/php/php-src/blob/master/Zend/zend_variables.h), and core definitions in [`zend_types.h`](https://github.com/php/php-src/blob/master/Zend/zend_types.h). The text distinguishes conceptual behavior from build/version-specific fast paths.

## Writing notes

Status is complete. The chapter avoids promising exact O(1)/O(n) behavior for every operation and emphasizes measuring first-write and peak-memory costs.
