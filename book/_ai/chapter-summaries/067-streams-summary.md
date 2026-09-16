# AI Summary — Chapter 67 — Streams

- Status: complete
- Volume: Volume 5 — PHP RUNTIME
- Last updated: 2026-09-14

## Written material

Explains PHP stream wrappers and resources, ownership, modes, partial reads/writes, EOF, bounded buffering, timeouts, contexts, multiplexing, filters, process pipes, and cleanup.

## Concepts already explained

Stream lifecycle, seekability, `fopen()`, `fread()`, `fwrite()`, `fgets()`, `feof()`, `stream_get_meta_data()`, `stream_set_timeout()`, `stream_select()`, and `stream_copy_to_stream()`.

## Terminology established

Stream resource, wrapper, bounded transfer, short read, backpressure, deadline, timeout, ownership.

## Examples used

Bounded `php://temp` read; explicit copy loop; streaming importer; remote stream context; SSRF and unbounded-read failure.

## Cross-references

Chapter 66 for cancellation; Chapter 68 for filesystem streams; Chapter 69 for process pipes.

## Open threads

Use stream lifecycle rules for filesystem and process boundaries; connect timeouts to idempotent retries.

## Exact next section

Chapter 68 — Filesystem: the Why This Matters section.

## Technical verification notes

References use official PHP Streams documentation. Exact blocking, seekability, and timeout behavior must be tested for the target wrapper and platform.
