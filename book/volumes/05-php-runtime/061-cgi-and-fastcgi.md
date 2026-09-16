---
book: The Complete Modern PHP Engineering Book
volume: 5
volume_title: PHP RUNTIME
chapter: 61
title: CGI and FastCGI
slug: cgi-and-fastcgi
status: complete
summary: ../../_ai/chapter-summaries/061-cgi-and-fastcgi-summary.md
---

# Chapter 61 — CGI and FastCGI

## Why This Matters

PHP does not receive an HTTP request directly from a browser. A web server or reverse proxy accepts the connection, applies transport policy, and hands an application request to a PHP SAPI. CGI and FastCGI are two ways to define that handoff. Understanding them explains why a PHP file can run from a terminal but not through a URL, why a bad `SCRIPT_FILENAME` produces confusing errors, and why a FastCGI backend can be overloaded even while the front web server is healthy.

```text
client
  ↓ HTTP/TLS
web server / reverse proxy
  ├─ static response
  └─ CGI environment + body / FastCGI record
          ↓
PHP CGI or FastCGI SAPI
          ↓
PHP entry script → response headers and body
```

CGI is an interface contract. FastCGI keeps a protocol-shaped connection to a persistent application process instead of starting a fresh process for every request. Neither protocol authenticates the caller, validates application input, or guarantees that a request will finish. Those remain responsibilities at later boundaries.

## Mental Model

### CGI: a process per request

In the classic CGI model, the web server starts a CGI executable for a request. Request metadata is placed in environment variables, the request body is available on standard input, and the program writes response headers and body to standard output. The server collects that output and turns it into an HTTP response.

```text
request
  → fork/exec CGI program
  → environment variables + stdin
  → PHP startup and script execution
  → stdout: headers, blank line, body
  → process exits
```

The process boundary provides strong isolation between requests, but process startup, configuration loading, extension initialization, and script compilation can dominate small requests. CGI is simple to reason about and expensive to run at high request rates.

### FastCGI: a persistent protocol and process

FastCGI retains the separation between the web server and application runtime while allowing a PHP process to serve multiple requests. The server connects to a FastCGI endpoint, sends request parameters and body data in protocol records, and reads the application response. A process manager such as PHP-FPM normally owns the endpoint and worker pool; raw `php-cgi` can also be run as a FastCGI server in suitable configurations.

```text
web server
  → connect to Unix socket or TCP endpoint
  → begin request and send parameters/body
  → PHP worker handles one request
  → read status, headers, body, end request
  → worker becomes available again
```

Persistence is about the process, not about granting application state a free lifetime. PHP must initialize and tear down request state for each request; OPcache and loaded code may be shared or retained according to the SAPI/build/configuration, while request variables should not be treated as durable storage.

## Core Concept

### CGI variables are an untrusted translation

The web server maps an HTTP request and its routing decision into values such as `REQUEST_METHOD`, `QUERY_STRING`, `CONTENT_TYPE`, `CONTENT_LENGTH`, `SCRIPT_NAME`, `SCRIPT_FILENAME`, and `PATH_INFO`. PHP exposes many of them through `$_SERVER`. The names and exact values depend on the server and configuration, so application code should not infer trust from a variable merely because it came from a server process.

`SCRIPT_FILENAME` is especially important: the server must map a URL to the intended file. A careless configuration may pass arbitrary path segments to PHP, expose source files, execute files outside the release directory, or route a static file to the interpreter. Use a front controller or an explicit allowlist and test the mapping.

### The response is a stream with a boundary

CGI-style output has response metadata followed by a blank line and body:

```text
Content-Type: text/plain; charset=utf-8
Status: 200 OK

hello
```

In normal PHP code, `header()` and `http_response_code()` express this information. The SAPI and web server translate it to the client-facing protocol. Once headers or body bytes have been committed, later application errors may not be able to change the status. A framework response object delays this boundary and makes it easier to test, but it cannot undo bytes already sent.

### Transport errors are not application errors

Distinguish these observations:

```text
client cannot connect to web server
  → web-server routing/TLS/listener issue
web server cannot connect to FastCGI endpoint
  → socket, process manager, permission, or capacity issue
FastCGI request times out
  → application, dependency, pool, or timeout issue
PHP returns a 4xx/5xx response
  → application or deliberate edge-policy result
```

The same user-visible “502” can result from a dead socket, an invalid response, an upstream timeout, or a process crash. Preserve web-server, FastCGI, and PHP logs with a request identifier so the boundary can be located.

## Practical Example: The Parameter Map

A simplified request adapter makes the translation visible:

```php
<?php

declare(strict_types=1);

$method = $_SERVER['REQUEST_METHOD'] ?? '';
$contentType = $_SERVER['CONTENT_TYPE'] ?? '';
$script = $_SERVER['SCRIPT_FILENAME'] ?? '';

if ($method !== 'POST' || $contentType !== 'application/json') {
    http_response_code(415);
    header('Content-Type: application/json; charset=utf-8');
    echo '{"error":"unsupported request"}';
    exit;
}

if (!str_starts_with($script, '/srv/app/current/public/')) {
    error_log('unexpected script mapping');
    http_response_code(500);
    exit;
}

$body = file_get_contents('php://input');
```

This is only a boundary check, not a complete security policy. A real deployment should make the server configuration enforce the script mapping and should parse and validate the body separately. Never accept a client-provided header as proof that the request originated from a trusted proxy.

## Production Example: Socket versus TCP

FastCGI endpoints are commonly a Unix-domain socket or a TCP address:

```text
Unix socket: /run/php/php-fpm.sock
  + local, permission-controlled, no routable network address
  - path and ownership must match both services

TCP: 127.0.0.1:9000 or an internal service address
  + easy container/service boundary and independent lifecycles
  - firewall, binding, and network access must be controlled
```

The choice is an operational constraint, not a universal performance contest. A local socket that has the wrong group produces connection failures; a TCP listener bound to all interfaces can expose an application protocol to unintended clients. Configure the web server to send only the parameters PHP needs and configure the endpoint so only authorized peers can connect.

## Failure Modes

### Wrong script mapping

Symptoms include “file not found,” a generic gateway error, or execution of a file that should have been static. Check the complete path after symlink/release resolution, document root, front-controller rules, `PATH_INFO`, and whether the web server verifies that a file exists before forwarding it.

### Socket or TCP failures

“Connection refused” suggests no listener or an actively rejected connection; a timeout can indicate a full backlog, a firewall, or a saturated service. Check endpoint existence and ownership, listener state, FPM master/worker health, and web-server upstream timeouts. Do not respond by blindly increasing every timeout: that can turn a small overload into a larger queue.

### Protocol and header mistakes

Malformed FastCGI configuration, invalid response headers, accidental output, or a crashed worker can cause the web server to emit a 502/503. Correlate timestamps and request IDs across logs. The client-facing status is evidence of a boundary failure, not a diagnosis.

### Request body and limits

A proxy, web server, FastCGI configuration, and PHP setting can each impose body, header, or timeout limits. The smallest limit wins, but errors may be reported by different layers. Document the intended budget and test a body just below and just above each boundary. Do not let PHP read an unbounded upload into memory.

## Performance

CGI pays process startup per request. FastCGI amortizes startup and can reuse OPcache, but it introduces a finite pool and queues. The important capacity variables are request arrival rate, service time, number of workers, endpoint backlog, web-server connections, and downstream capacity. Reducing PHP execution time does not help if the database is the bottleneck; increasing workers does not help if it only creates more database contention.

Measure connect time, queue time, PHP execution time, response transfer time, and error rate separately. Warm and cold behavior differ. A local CLI benchmark cannot establish FastCGI latency, because it omits the web-server hop, socket/record handling, pool queue, and production configuration.

## Security

Treat FastCGI as an application control plane, not a public HTTP server. Restrict the endpoint to the intended web server or network. Protect Unix-socket permissions and TCP access controls. Ensure the web server cannot be tricked into setting a sensitive `SCRIPT_FILENAME`, forwarding arbitrary environment values, or executing uploaded files.

Keep PHP source and secrets outside the public document root where possible. Disable verbose error display in production while retaining structured server-side diagnostics. Validate proxy headers against a known proxy boundary; an arbitrary `X-Forwarded-For` value is input, not identity.

## Testing

Test the layers independently and together:

1. Run the entry script through CLI for application behavior, while remembering that CLI is not FastCGI.
2. Test the web server's route-to-script mapping for valid, missing, traversal-like, and static-file paths.
3. Exercise the real Unix socket or TCP endpoint in a container/integration environment.
4. Assert status, headers, body, and logs for malformed requests and upstream failures.
5. Load-test at the configured worker limit and observe queueing, timeouts, and downstream saturation.

Include a deployment test that replaces a release while requests are in flight. Verify that a request sees a complete release and that the old endpoint can drain or fail in the documented way.

## Exercises

1. Draw the byte and process boundaries for one request served by CGI and one served by FastCGI. Mark where environment variables, stdin, stdout, and the database appear.
2. Create a deliberately wrong `SCRIPT_FILENAME` mapping and diagnose it from web-server and PHP logs.
3. Choose Unix socket or TCP for a local container deployment. List the permissions, network, and failure assumptions behind the choice.
4. Send a request with an oversized body and identify which configured layer rejects it.

## Review Questions

1. What problem does FastCGI solve compared with classic CGI, and what new capacity constraint does it introduce?
2. Why is `SCRIPT_FILENAME` a security-sensitive server-to-runtime value?
3. Why can a 502 be caused by the transport boundary rather than by a PHP exception?
4. What is the response boundary, and why can late errors fail to change a response status?
5. Which measurements distinguish endpoint connection failure from a saturated PHP pool?

## Summary

CGI and FastCGI define the handoff between a web server and PHP. CGI starts a process per request; FastCGI keeps a protocol connection to reusable PHP processes. Treat server parameters, script mapping, endpoint permissions, response framing, queueing, and timeout budgets as explicit contracts. Chapter 62 examines the process manager most PHP production deployments use: PHP-FPM.

## References

- [PHP Manual: CGI and command-line usage](https://www.php.net/manual/en/features.commandline.php)
- [PHP Manual: CGI/FastCGI installation](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: PHP-FPM](https://www.php.net/manual/en/install.fpm.php)
- [PHP Manual: PHP-FPM configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: FastCGI Process Manager configuration directives](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: `php_sapi_name`](https://www.php.net/manual/en/function.php-sapi-name.php)
