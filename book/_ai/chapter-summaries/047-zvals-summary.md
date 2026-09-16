# AI Summary — Chapter 47 — zvals

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 47 explains zvals as tagged Zend runtime value containers. It distinguishes inline scalars from pointers to managed strings, arrays, objects, resources, and references; covers refcounted payloads, copy-on-write, object identity, explicit references, temporary/undefined states, memory measurement, performance, security, testing, exercises, and review questions.

## Concepts already explained

zval, `zend_value`, type tag, refcounted payload, copy-on-write preview, object identity, `zend_reference`, explicit aliasing, `IS_UNDEF`, temporary values, payload lifetime.

## Terminology established

zval/payload graph; scalar-inline versus payload-pointer model; shared-then-separate assignment sequence; object identity diagram; reference alias diagram.

## Examples used

Array assignment and mutation, object assignment, explicit reference aliasing, a 100,000-row memory experiment, and a behavior fixture for independent array copies.

## Cross-references

Builds on Chapters 45–46; points to Chapter 48 for strings, Chapters 49–50 for HashTables/arrays, Chapter 51 for references, Chapter 52 for copy-on-write, and Chapter 38 for cloning. References use PHP-8.4 `zend_types.h` and PHP manuals.

## Open threads

Detailed references and copy-on-write behavior remain in Chapters 51–52; array/string/object payload layouts continue in adjacent chapters.

## Exact next section

None — chapter complete.

## Technical verification notes

The zval/value model is grounded in PHP-8.4 `Zend/zend_types.h` and is explicitly conceptual/version-sensitive. Memory figures are intentionally not asserted; measurements must record build, architecture, allocator, input shape, and process conditions.

## Writing notes

Preserve the distinction between PHP variables, zval values, and refcounted payloads. Do not treat private tags, field offsets, or `var_dump()` output as a stable ABI or memory-layout inspection tool.
