# AI Summary — Chapter 3 — PHP as a Language vs PHP as a Runtime

- Status: complete
- Volume: Volume 1 — THE PHP MENTAL MODEL
- Last updated: 2026-09-14

## Written material

Completed the chapter through Summary: distinguishes PHP language semantics from implementation/runtime behavior; explains the source-to-execution path; separates Zend Engine, extensions, SAPI, configuration, process lifetime, environment, and external services; and applies the model to production diagnosis, performance, security, databases, concurrency, and testing.

## Concepts already explained

- Language semantics versus runtime implementation
- Source parsing, compilation, execution, and output
- Zend Engine and OPcache at a conceptual level
- Extensions, APIs, and database drivers
- SAPI, CLI, PHP-FPM, and process lifetime
- `php.ini`, runtime configuration, and environment variables
- External resources and service boundaries
- Layered diagnosis, startup validation, and cross-process invariants

## Terminology established

- Language semantics
- PHP implementation/runtime
- Zend Engine
- extension
- SAPI
- configuration
- environment
- process model
- external service

## Examples used

- Strict scalar function example with `declare(strict_types=1)`
- PDO and extension/driver availability checks
- Runtime diagnostics using `PHP_VERSION`, `PHP_SAPI`, `extension_loaded()`, and `php_ini_loaded_file()`
- Filesystem failure handling
- Required-extension bootstrap validation
- CLI versus FPM deployment comparison

## Cross-references

- [Chapter 1 — What PHP Actually Is](../../volumes/01-the-php-mental-model/001-what-php-actually-is.md)
- [SKELETON.md](../../../SKELETON.md)
- Official PHP Manual sources listed in the chapter

## Open threads

No open writing thread remains for this chapter. Later chapters should deepen the conceptual references to zvals, HashTables, opcodes, OPcache, PHP-FPM, and worker lifecycles without collapsing language semantics into implementation details.

## Exact next section

No next section; the chapter is complete.

## Technical verification notes

Conceptual runtime claims are intentionally qualified. Claims about PHP's general-purpose use, CLI/FPM, extensions, runtime configuration, and FPM behavior were checked against the official PHP Manual; implementation details are deferred to later internals chapters.

## Writing notes

Keep this summary short if the chapter is revised. Do not update global continuation state for this scoped session.
