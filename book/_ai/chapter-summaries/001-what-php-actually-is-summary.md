# AI Summary — Chapter 1 — What PHP Actually Is

- Status: complete
- Volume: Volume 1 — THE PHP MENTAL MODEL
- Last updated: 2026-09-14

## Written material

Written the full chapter: why the definition of PHP matters; a layered mental model; PHP as language, runtime, and application environment; a minimal CLI/output example; the role of Zend; the source-to-execution pipeline; parsing versus execution; output and side-effect boundaries; process lifecycles; a reservation example showing computation versus adapters and concurrency boundaries; testing guidance; common mistakes; senior-engineering questions; exercises; and review questions.

## Concepts already explained

- Language, runtime, application environment
- Server-side execution
- Process model
- PHP extensions
- Zend Engine
- Entry point and output consumer
- Source-to-execution pipeline: source, parsing, compilation, Zend VM execution, and result boundary
- Parse-time versus execution-time failure
- Short-lived request lifecycle versus long-running process lifecycle
- Boundary separation between computation, adapters, and external side effects
- Concurrency limits of a read-then-write check in application code

## Terminology established

- PHP language
- PHP runtime
- application environment
- execution context
- process model
- Zend Engine
- extension

## Examples used

- A minimal strict-types PHP program assigning a string and writing it to output.
- A conceptual browser/web-server/PHP-FPM/application/dependency path.
- A small typed `subtotal()` function used to distinguish compilation from execution.
- A throwing script used to distinguish a valid program that fails during execution from malformed source.
- A lifecycle diagram comparing CLI scripts, web requests, and long-running jobs.
- A reservation `TimeRange` example, first as a pure availability check and then behind a storage interface.
- A PHPUnit example for adjacent and contained time ranges.

## Cross-references

- [PHP Manual: Introduction](https://www.php.net/intro)
- [PHP Manual: Basic syntax](https://www.php.net/manual/en/language.basic-syntax.php)
- [PHP Manual: Language Reference](https://www.php.net/manual/en/langref.php)

## Open threads

- Carry the language/runtime/environment distinction into Chapter 2's historical explanation.
- Revisit process lifetime and entry points in Chapters 4–6 without duplicating this chapter.

## Exact next section

Chapter 2 — Why PHP Became What It Is: the Why This Matters section.

## Technical verification notes

Claims about PHP's general-purpose/server-side role, PHP tags, and the language/runtime distinction were checked against the official PHP Manual. Zend Engine internals have been described only at a conceptual level so far.

## Writing notes

Keep this summary short and update it after every writing session. Chapter 1 is complete; preserve its vocabulary when reviewing later chapters.
