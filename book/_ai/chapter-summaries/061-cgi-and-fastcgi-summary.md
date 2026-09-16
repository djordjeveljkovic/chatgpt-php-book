# AI Summary — Chapter 61 — CGI and FastCGI

- Status: complete
- Volume: Volume 5 — PHP RUNTIME
- Last updated: 2026-09-14

## Written material

Explains CGI and FastCGI as web-server-to-PHP handoff contracts, including process-per-request CGI, persistent FastCGI endpoints, parameter/body/response translation, Unix sockets versus TCP, script mapping, layered timeouts, transport failures, security, performance, and integration testing.

## Concepts already explained

CGI environment/stdin/stdout model, FastCGI record/process model, SAPI boundary, `SCRIPT_FILENAME`, response commitment, endpoint queueing, 502 diagnosis, socket permissions, and server/runtime separation.

## Terminology established

Transport boundary, parameter map, script mapping, FastCGI endpoint, response framing, upstream queue, and boundary attribution.

## Examples used

Request parameter validation, response header/body framing, Unix socket/TCP tradeoffs, mapping failures, body limits, deployment transitions, and layered transport diagnostics.

## Cross-references

Builds on Chapters 4–6 and Chapter 40, leads to FPM in Chapter 62 and request lifecycle in Chapter 63, and references PHP CGI/FPM installation/configuration and SAPI documentation.

## Open threads

FPM pool sizing, worker management, reloads, status, and resource limits are developed in Chapter 62.

## Exact next section

None — chapter complete.

## Technical verification notes

CGI/FastCGI and FPM claims are qualified by SAPI/build/configuration where appropriate and linked to PHP Manual installation, FPM configuration, and SAPI pages.

## Writing notes

Keep protocol, web-server, PHP, and application failures separate when diagnosing the same client-facing gateway error.
