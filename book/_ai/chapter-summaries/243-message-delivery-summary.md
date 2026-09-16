# AI Summary — Chapter 243 — Message Delivery

- Status: complete
- Volume: Volume 16 — DISTRIBUTED SYSTEMS
- Last updated: 2026-09-16

## Written material

Explains at-most-once, at-least-once, and effectively-once business effects; publish and consume crash windows; outbox and inbox patterns; message envelopes; acknowledgment and leases; ordering and partitioning; schema evolution; PHP worker lifecycle; testing; and security. Includes a typed message envelope.

## Concepts already explained

Delivery contract, at-most-once, at-least-once, effectively-once effect, publish gap, consume gap, outbox, inbox, lease, partition, and schema evolution.

## Terminology established

Message identity, operation identity, event identity, schema version, acknowledgment, visibility lease, hot partition, and delayed event.

## Examples used

Delivery crash-window diagram, message-envelope fields, typed MessageEnvelope and handler, outbox/inbox flow, lease behavior, and partitioning policies.

## Cross-references

Chapters 234, 242, and the next chapter on queues.

## Open threads

Continue with queue topology, consumer capacity, and backlog in Chapter 244.

## Exact next section

Chapter 244 — Queues: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP example. Local Markdown links resolved and git diff --check passed. No live broker integration was run.

## Writing notes

Distinguishes broker delivery guarantees from effectively-once business outcomes.
