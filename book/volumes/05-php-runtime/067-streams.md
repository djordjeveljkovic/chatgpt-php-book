---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 67
title: Streams
slug: streams
status: complete
summary: ../../_ai/chapter-summaries/067-streams-summary.md
---

# Chapter 67 — Streams

## Why This Matters

Files, HTTP bodies, sockets, temporary storage, standard input, and child-process pipes look different at the operating-system level, but PHP gives many of them a common interface: streams. That uniformity is powerful and dangerous when code treats a stream as an always-readable string.

A stream may be partial, blocking, seekable, timed out, buffered, filtered, or already at end-of-file. Production code must define ownership, read boundaries, timeout behavior, and cleanup. A stream is a resource with a lifecycle, not merely an argument to `fgets()`.

## Mental Model

```text
producer ──bytes──▶ PHP stream resource ──read/write──▶ consumer
                         │
              wrapper + mode + position + metadata
                         │
                 optional filters/context
```

```text
open → validate mode → configure timeout/context
     → transfer bounded chunks → check progress/EOF
     → flush if writing → close exactly once
```

The abstraction hides some differences but not all. A regular file is commonly seekable; a network socket may not be. A pipe can block because its peer is not reading. `stream_get_contents()` is appropriate only when the size is bounded by a known budget.

## Core Concept

PHP stream wrappers include `file://`, `php://memory`, `php://temp`, `php://stdin`, and `php://stdout`. The default wrapper is often `file://`, but wrapper names are input and should not be accepted from untrusted data without an allowlist.

`fopen()` creates a resource; `fread()` may return fewer bytes; `fwrite()` may write fewer bytes; `feof()` reports an end-of-file condition rather than guaranteeing the next read is empty; `stream_get_meta_data()` exposes wrapper, mode, seekability, and timeout state; `stream_set_timeout()` affects supported blocking streams; `stream_copy_to_stream()` transfers without materializing the whole body; `fclose()` releases the resource.

## Minimal Example: Bounded Reading

```php
<?php

declare(strict_types=1);

function readAtMost($stream, int $maximumBytes): string
{
    if (! is_resource($stream)) {
        throw new InvalidArgumentException('Expected an open stream.');
    }

    $contents = stream_get_contents($stream, $maximumBytes + 1);
    if ($contents === false) {
        throw new RuntimeException('Stream read failed.');
    }
    if (strlen($contents) > $maximumBytes) {
        throw new LengthException('Input exceeds the limit.');
    }

    return $contents;
}

$stream = fopen('php://temp', 'w+b');
if ($stream === false) {
    throw new RuntimeException('Could not open temporary stream.');
}

try {
    fwrite($stream, "hello\n");
    rewind($stream);
    echo readAtMost($stream, 1024);
} finally {
    fclose($stream);
}
```

The extra byte distinguishes “within the limit” from “at least one byte over.”

## How It Works

Reading is not guaranteed to fill a buffer. A robust copy loop handles short reads and writes:

```php
<?php

declare(strict_types=1);

function copyStream($source, $destination): int
{
    $total = 0;
    while (! feof($source)) {
        $chunk = fread($source, 8192);
        if ($chunk === false) {
            throw new RuntimeException('Read failed.');
        }
        if ($chunk === '') {
            if (feof($source)) { break; }
            continue;
        }

        $offset = 0;
        while ($offset < strlen($chunk)) {
            $written = fwrite($destination, substr($chunk, $offset));
            if ($written === false || $written === 0) {
                throw new RuntimeException('Write failed or made no progress.');
            }
            $offset += $written;
        }
        $total += strlen($chunk);
    }
    return $total;
}
```

For ordinary files, `stream_copy_to_stream()` may be clearer. The explicit loop is useful when adding limits, progress, cancellation, or validation.

## Practical Example: Streaming an Import

```php
<?php

declare(strict_types=1);

function importLines(string $path, callable $consume): int
{
    $handle = fopen($path, 'rb');
    if ($handle === false) {
        throw new RuntimeException('Unable to open import.');
    }

    $count = 0;
    try {
        while (($line = fgets($handle)) !== false) {
            $line = rtrim($line, "\r\n");
            if ($line === '') { continue; }
            $consume($line);
            $count++;
        }
        if (! feof($handle)) {
            throw new RuntimeException('Import ended with an I/O error.');
        }
    } finally {
        fclose($handle);
    }
    return $count;
}
```

Space is approximately O(line length), not O(file size), assuming the consumer does not accumulate every row. Batch writes, transaction boundaries, malformed-row policy, and restart checkpoints remain part of the importer design.

## Blocking, Timeouts, and Multiplexing

For a socket or pipe, a read can wait indefinitely unless the stream or surrounding loop has a timeout. `stream_set_timeout()` reports timeout state through metadata for supported blocking streams; it does not create a complete asynchronous networking library. `stream_select()` can wait for readiness with a bounded timeout, but has platform and stream-type limitations.

```text
wait until ready or deadline
       ↓
read/write a bounded amount
       ↓
check progress, EOF, timeout, cancellation
       ↓
retry only under an explicit policy
```

A timeout is a budget, not proof that a peer did nothing. It may still process a request after the client gives up, so external side effects need idempotency keys.

## Bad Example and Better Example

```php
$body = file_get_contents($_GET['url']);
```

This combines an untrusted wrapper/URL, unbounded memory, unspecified timeout behavior, and unclear errors. It can become SSRF or a worker memory incident.

Use an approved source, a context, a size limit, and cleanup:

```php
$context = stream_context_create([
    'http' => ['timeout' => 3.0, 'follow_location' => 0],
]);
$stream = fopen($approvedUrl, 'rb', false, $context);
if ($stream === false) {
    throw new RuntimeException('Remote resource unavailable.');
}
try {
    $body = readAtMost($stream, 2_000_000);
} finally {
    fclose($stream);
}
```

URL parsing, egress policy, private-network blocking, DNS-rebinding defenses, and content validation are still required; a stream context is not a complete SSRF policy.

## Performance

Chunking bounds peak PHP memory but does not make transfer free. Cost includes bytes, system calls, copies, filters, compression, and downstream backpressure. Choose chunk size by measurement. `php://temp` provides seekable scratch storage that can spill to a temporary file after its threshold; use it without assuming payloads are tiny.

## Security

Treat paths, wrappers, filenames, and destinations as untrusted. Allowlist protocols, enforce body and decompression limits, avoid dynamic includes, prevent SSRF, and do not copy secrets into debug buffers. Validate content independently of filenames and client headers.

## Testing

Test empty streams, short reads, write failures, truncation, exact limits, over-limit input, and timeout state. Use `php://temp` for unit tests. Integration-test real sockets and process pipes for partial transfer and close behavior. Assert `finally` closes resources when parsing or persistence throws.

## Common Mistakes

- Assuming one read or write transfers the requested size.
- Using `feof()` before attempting a read.
- Loading unbounded input into memory.
- Treating a timeout as proof an external side effect did not happen.
- Forgetting cleanup on exceptional paths.
- Assuming every stream is seekable or supports identical metadata.

## Senior Engineer Thinking

For every stream, record its owner, maximum data, blocking deadline, and partial-transfer outcome. This applies equally to uploads, logs, exports, sockets, and process pipes.

## Exercises

1. Implement bounded copy with progress and a deadline.
2. Build a line importer that rejects oversized lines without loading the file.
3. Test a slowly producing process pipe and prove the reader cannot deadlock.
4. Design an allowlist for remote image fetching, including DNS, size, timeout, and content rules.

## Review Questions

1. Why can a successful read return fewer bytes than requested?
2. What does `feof()` tell you?
3. Why does a timeout not prove a remote operation was not performed?
4. When is `stream_copy_to_stream()` preferable to a string?
5. Which stream properties must be tested rather than assumed?

## Summary

PHP streams provide a common resource model for files, sockets, standard I/O, temporary storage, and pipes. Production code bounds memory and time, handles partial progress and EOF, secures wrappers, and closes resources on every path. The next chapter applies these rules to filesystem paths, permissions, atomic replacement, and locking.

## References

- [PHP Manual: Streams](https://www.php.net/manual/en/book.stream.php)
- [PHP Manual: Stream Functions](https://www.php.net/manual/en/ref.stream.php)
- [PHP Manual: `fopen()`](https://www.php.net/manual/en/function.fopen.php)
- [PHP Manual: `stream_set_timeout()`](https://www.php.net/manual/en/function.stream-set-timeout.php)
- [PHP Manual: `stream_select()`](https://www.php.net/manual/en/function.stream-select.php)
- [PHP Manual: `stream_copy_to_stream()`](https://www.php.net/manual/en/function.stream-copy-to-stream.php)
