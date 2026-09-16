# AI Summary — Chapter 48 — Strings Internally

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Chapter 48 explains the conceptual/current PHP-8.4 `zend_string` representation: refcounted metadata, cached hash, explicit byte length, and NUL-terminated buffer. It covers binary safety, embedded NULs, copy-on-write, allocation/concatenation, interning, bytes versus Unicode characters, string offsets, streaming large outputs, security, testing, exercises, and review questions.

## Concepts already explained

`zend_string`, length-counted bytes, C terminator, cached hash, interned string, string copy-on-write, byte length versus character length, encoding boundary, chunked output.

## Terminology established

zval-to-string diagram; byte/encoding distinction; interned-string caveat; allocation/streaming cost model; text versus binary contract.

## Examples used

Embedded-NUL payload, UTF-8 length comparison, string mutation after assignment, large CSV materialization versus `php://output`, and UTF-8 validation.

## Cross-references

Builds on Chapters 46–47, points to Chapters 49–50 for string keys and arrays, and later memory/stream chapters. References include the PHP strings, string functions, mbstring, streams manuals and PHP-8.4 `zend_string.h/.c`.

## Open threads

Later chapters expand HashTables, PHP arrays, memory, copy-on-write, and runtime streams.

## Exact next section

None — chapter complete.

## Technical verification notes

The byte-oriented string model follows the PHP strings manual; structure, hash field, interning, and allocation claims are explicitly tied to PHP-8.4 php-src and labeled implementation-specific. No stable address or exact memory-size claim is made.

## Writing notes

Keep Unicode, normalization, and escaping separate from Zend byte storage. Treat embedded NULs and downstream C/OS boundaries as explicit test cases.
