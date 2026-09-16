# AI Summary — Chapter 63 — Request Lifecycle

- Status: complete
- Volume: Volume 5 — PHP RUNTIME
- Last updated: 2026-09-14

## Written material

Explains request startup, SAPI input, bootstrap, compile/load, application execution, response emission, shutdown, teardown, worker reuse, request/process/shared/deployment scopes, side-effect ambiguity, `register_shutdown_function()`, `fastcgi_finish_request()`, failure modes, security, performance, and lifecycle testing.

## Concepts already explained

Request scope, worker scope, shared-service scope, response commitment, client disconnect, durable side effect, best-effort shutdown, post-response worker occupancy, timeout/fatal boundaries, and request-state leakage.

## Terminology established

Nested request lifecycle, response boundary, durable success point, request context, teardown, and ambiguous outcome.

## Examples used

Front-controller phases, explicit `RequestContext`, shutdown timing, commit-then-disconnect scenario, post-response FPM work, timeout/fatal cases, and lifecycle test matrix.

## Cross-references

Connects Chapters 40–59, transport/FPM Chapters 61–62, and later worker/long-running chapters 64–65. References PHP shutdown, FastCGI finish, error handling, output control, and FPM configuration manuals.

## Open threads

Worker capacity and concurrent request failure modes follow in Chapter 64.

## Exact next section

None — chapter complete.

## Technical verification notes

`register_shutdown_function()` and `fastcgi_finish_request()` behavior is linked to the PHP Manual. Cleanup and exact teardown ordering are presented as best-effort/version- and SAPI-sensitive rather than universal guarantees.

## Writing notes

Keep response delivery, durable side effects, and process cleanup as separate claims in incident analysis.
