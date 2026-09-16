# AI Summary — Chapter 4 — How PHP Applications Execute

- Status: complete
- Volume: Volume 1 — THE PHP MENTAL MODEL
- Last updated: 2026-09-14

## Written material

Completed the chapter: why execution must be reconstructed as a sequence; the source-to-execution pipeline; bootstrap and input translation; output and side-effect boundaries; HTTP, CLI, queue, scheduled, and test entry points; short-lived versus long-lived process lifecycles; production failure analysis; a transport-independent service example; a boundary-leaking import example and improvement; performance, security, testing, mistakes, exercises, review questions, and summary.

## Concepts already explained

- Entry point and execution context
- Load, parse/prepare, execute, and expose effects
- Bootstrap, input translation, output adaptation, and cleanup
- Process boundary versus unit-of-work/request boundary
- Short-lived invocation versus long-lived worker
- External side effects, partial success, duplicate execution, and idempotency concerns
- Startup cost, application work, boundary cost, and OPcache at a conceptual level

## Terminology established

- Entry point
- Process boundary
- Unit of work
- Bootstrap
- Adapter
- Short-lived invocation
- Long-lived worker
- Request-local, process-local, and durable state

## Examples used

- A source-to-effect execution pipeline.
- HTTP, CLI, queue, scheduled-task, and test entry-point diagrams.
- A transport-independent `Greeting` service with a CLI adapter.
- A boundary-leaking `ImportUsers` class and a result-returning improvement.
- A production order request lifecycle and failure analysis.

## Cross-references

- [SKELETON.md](../../../SKELETON.md) for the authoritative outline and chapter format.
- Chapter 5 for a concrete CLI-versus-web comparison.
- Volume IV for parser, compiler, opcode, and other runtime internals.
- [PHP Manual: Command line usage](https://www.php.net/manual/en/features.commandline.usage.php)
- [PHP Manual: FastCGI Process Manager (FPM)](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: OPcache](https://www.php.net/manual/en/book.opcache.php)

## Open threads

No required Chapter 4 material remains for this writing pass. Detailed parser/compiler/opcode, PHP-FPM, and request-lifecycle internals are intentionally deferred to Volume IV.

## Exact next section

Chapter complete; no next section.

## Technical verification notes

Conceptual claims and the CLI, FPM, and OPcache descriptions were checked against the linked official PHP Manual pages. The chapter avoids implementation-level parser, opcode, and process-manager internals reserved for Volume IV.

## Writing notes

Keep this summary short and update it after every writing session.
