# AI Summary — Chapter 58 — Extensions

- Status: complete
- Volume: Volume 4 — PHP UNDER THE HOOD
- Last updated: 2026-09-14

## Written material

Explains internal versus userland extensions, loaded versus built-in modules, CLI/FPM differences, module/request lifecycle hooks, process versus request state, Zend extension API and ABI boundaries, function registration and arginfo, native failures and ownership, deployment requirements, performance, security, testing, exercises, and review questions.

## Concepts already explained

Extension boundary, internal function, arginfo, function table, MINIT/RINIT/RSHUTDOWN/MSHUTDOWN, module globals, ABI compatibility, native memory, linked-library dependencies, SAPI-specific configuration.

## Terminology established

Extension, SAPI, shared module, internal function, arginfo, module lifecycle, request lifecycle, ABI, ZTS, native boundary, trusted computing base.

## Examples used

CLI and extension inspection; a simplified native `example_add()` function; startup requirements; an extension-backed image adapter; a bad SAPI assumption; a runtime requirement checker.

## Cross-references

Connects extension memory and cleanup to Chapter 56, lifecycle/collection to Chapter 57, and OPcache as an extension to Chapter 59. Points to Volume V for SAPI and worker behavior.

## Open threads

Later production chapters should apply extension compatibility matrices, native-library security review, and SAPI-specific smoke tests.

## Exact next section

Chapter 59 — OPcache: the Why This Matters section.

## Technical verification notes

Uses PHP Internals Book, PHP manual extension-loading/build references, Composer platform-dependency documentation, and php-src `ext`/`Zend` trees. Native macro signatures and ABI details are explicitly version-sensitive; the C snippet is labeled illustrative rather than a complete extension.
