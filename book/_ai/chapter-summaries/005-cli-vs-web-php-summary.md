# AI Summary — Chapter 5 — CLI vs Web PHP

- Status: complete
- Volume: Volume 1 — THE PHP MENTAL MODEL
- Last updated: 2026-09-14

## Written material

Completed the chapter through Summary: why execution context matters; a SAPI and adapter mental model; CLI execution; web requests through PHP-FPM; application workers; input/output boundaries; configuration; process and request lifetime; failure and recovery; security; performance and capacity; testing; bad and better designs; common mistakes; senior-engineer guidance; exercises; and review questions.

## Concepts already explained

- CLI, web/FPM, and application-worker execution contracts
- SAPI as the boundary around the PHP runtime
- CLI arguments and standard streams
- HTTP request/response input and output
- FPM pools, child processes, and `pm.max_children`
- Long-running worker loops, acknowledgment, retries, and duplicate delivery
- Configuration differences between runtimes and permitted `ini_set()` scope
- Process lifetime versus request/message lifetime
- Explicit adapters around reusable application logic
- Context-specific failure, security, capacity, and testing implications

## Terminology established

- SAPI
- CLI
- FPM child process
- application worker
- adapter
- process lifetime
- request lifetime
- message acknowledgment
- exit status
- standard input/output/error

## Examples used

- A typed CLI greeting command with arguments, standard error, and exit codes
- A minimal HTTP greeting endpoint with status and content type
- Conceptual worker receive/handle/acknowledge flow
- Diagnostic output for SAPI, PHP version, and loaded `php.ini`
- Shared typed input and service examples
- A bad environment-coupled function and an adapter-oriented design

## Cross-references

- [SKELETON.md](../../../SKELETON.md)
- [AI_AUTHORING_GUIDE.md](../../../AI_AUTHORING_GUIDE.md)
- [PHP Manual: CLI usage](https://www.php.net/manual/en/features.commandline.usage.php)
- [PHP Manual: FPM](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: Configuration modes](https://www.php.net/manual/en/configuration.changes.modes.php)

## Open threads

- Chapter 6 can develop the HTTP request/response lifecycle in more detail.
- Later runtime chapters can deepen SAPI, FPM, request cleanup, and worker-process internals.
- Later operations and distributed-systems chapters can expand queue delivery, retries, idempotency, and capacity planning.

## Exact next section

Chapter 6 — The Request/Response Model.

## Technical verification notes

Official PHP Manual pages were checked for CLI input modes and streams, `PHP_SAPI`, FPM process-pool behavior and `pm.max_children`, FPM configuration security warnings, and configuration-setting scope. The chapter labels the worker lifecycle and adapter design as application-level concepts rather than PHP language guarantees.

## Writing notes

The chapter is complete. Preserve the distinction between language, SAPI, process model, and application boundary when extending adjacent chapters.
