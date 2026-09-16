---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 138
title: WebSocket Concepts
slug: websocket-concepts
status: complete
summary: ../../_ai/chapter-summaries/138-websocket-concepts-summary.md
---

# Chapter 138 — WebSocket Concepts

## Why This Matters

WebSocket provides a long-lived, full-duplex connection: the client and server can send messages independently after an HTTP-based opening handshake. That is useful for collaborative editing, presence, multiplayer state, and interactive dashboards where polling or one-way streaming is not enough.

WebSocket is a transport. It does not define authentication, message schemas, authorization, persistence, delivery guarantees, or reconnect behavior. A production design must add those contracts and must assume that connections disappear at any time.

## Opening Handshake and Frames

A client begins with an HTTP request containing an Upgrade request and a random Sec-WebSocket-Key. The server accepts with a calculated Sec-WebSocket-Accept value, then both sides exchange framed messages on the upgraded connection. Text messages are UTF-8; binary messages have application-defined encoding. Control frames support close, ping, and pong and have strict size constraints.

The handshake is not a substitute for authentication. Authenticate the connection using the normal origin and credential policy, then authorize each channel or message. Reject an unexpected Origin for browser clients when cross-site connections would be unsafe. Never implement the wire protocol by concatenating strings; use a tested WebSocket server or library that handles masking, fragmentation, limits, and close codes.

## A Message Contract

Design messages as versioned envelopes rather than exposing internal objects.

```json
{
  "type": "cursor.moved",
  "version": 1,
  "request_id": "req_abc",
  "room_id": "room_7",
  "payload": {"user_id": "u_2", "x": 14, "y": 8}
}
```

Bound the frame and decoded payload sizes before parsing or allocating large structures. Validate the type, version, room, and payload shape. A request ID lets the client correlate an error or acknowledgement; it is not automatically an idempotency key. If a message changes durable state, give it an explicit command name and apply the same authorization and concurrency rules as an HTTP command.

## PHP Process Model

PHP-FPM is optimized for request/response work. A connection that stays open occupies a worker, so thousands of sockets can exhaust a pool even when CPU use is low. A dedicated long-running PHP process using an event-loop library can manage connections, or a gateway can own WebSocket connections and forward authenticated messages to ordinary application workers. The choice depends on connection count, latency, deployment model, and operational skills.

A simplified application boundary might look like this:

```php
<?php

declare(strict_types=1);

final class RoomMessageHandler
{
    public function __construct(
        private readonly RoomAuthorizer $authorizer,
        private readonly RoomPublisher $publisher,
    ) {}

    public function handle(Connection $connection, string $text): void
    {
        $message = json_decode($text, true, 32, JSON_THROW_ON_ERROR);
        if (!is_array($message) || !isset($message['type'], $message['room_id'])) {
            $connection->close(1007, 'Invalid message');
            return;
        }

        $roomId = (string) $message['room_id'];
        if (!$this->authorizer->canPublish($connection->principal(), $roomId)) {
            $connection->send(json_encode([
                'type' => 'error',
                'code' => 'forbidden',
            ], JSON_THROW_ON_ERROR));
            return;
        }

        $this->publisher->publish($roomId, $message);
    }
}
```

Connection and library types vary; the important boundary is that transport parsing calls a domain-aware handler. Do not let a socket callback directly update arbitrary database rows. Use an application service that validates state transitions and applies transactions.

## Reconnect, Ordering, and Delivery

Clients should reconnect with bounded exponential backoff and jitter. On reconnect, authenticate again and resubscribe; do not assume a connection's authorization remains valid. If messages can be missed, include a stream sequence or durable cursor and offer a catch-up operation. If messages are ephemeral presence updates, dropping stale data may be preferable to replaying it.

A WebSocket send often means the server queued bytes in user-space; it does not mean the peer has processed the message. Define acknowledgements for commands that matter. Persist durable commands before acknowledging them, and make retries idempotent with a command ID and unique constraint. Use per-room or per-aggregate sequence numbers when consumers need ordering. Global ordering across all rooms increases coordination cost and usually provides little value.

## Backpressure and Resource Limits

A slow client can cause its outgoing queue to grow. Set limits on queued bytes, messages per connection, rooms per connection, and connection lifetime. Drop coalescible state updates, disconnect a client that cannot keep up, or move large transfers to an HTTP download. Never allow one connection to consume unbounded PHP memory.

Apply input rate limits and idle timeouts. A ping/pong heartbeat detects dead network paths, but a pong does not prove that the client is healthy at the application level. Close connections with an appropriate protocol close code and record the reason without echoing attacker-controlled text into logs.

## Scaling and Shared State

A connection is owned by one process, while a user's data may be produced by another process. Use a broker or shared pub/sub layer to route room events to all connection owners. Treat the broker as a delivery mechanism, not necessarily a durable log; choose persistence when clients need replay. Presence state needs expiry because a process can crash before sending a disconnect event.

Load balancing can use a connection affinity policy, but affinity does not remove the need for shared distribution or reconnect handling. During deployment, drain listeners, stop accepting new connections, send a close/reconnect signal, and let clients reconnect to healthy instances. Track the version of each connection so mixed deployments can be diagnosed.

## Security

Use TLS in production and authenticate during the handshake or an explicitly documented first message. Prefer credentials that are not exposed in URLs. Validate Origin for browser contexts, enforce tenant and room membership on subscriptions and commands, and avoid returning one room's errors or metadata to another tenant.

Apply per-connection and per-principal quotas. Masking is a protocol requirement for browser-to-server frames, not an authorization measure. Sanitize or encode message content when rendering it in a browser; WebSocket data does not bypass XSS concerns.

## Testing and Operations

Test handshake rejection, origin policy, invalid UTF-8, fragmented messages, oversized frames, malformed envelopes, reconnect, duplicate commands, slow consumers, idle timeouts, broker loss, and graceful draining. Use a fake clock for backoff and presence expiry. Load test connection count and message fan-out separately; a system can handle idle sockets but fail when one message is broadcast to every subscriber.

Monitor active connections by version and tenant, handshake failures, authorization failures, incoming/outgoing rates, queue depth, disconnect codes, reconnect latency, broker lag, and memory per connection. Sample message payloads only under an explicit privacy policy.

## Common Mistakes

- Assuming a WebSocket connection is a durable queue.
- Authenticating once and never checking authorization for subscriptions.
- Sending unbounded queues to slow clients.
- Running thousands of long-lived sockets in an ordinary PHP-FPM pool without capacity analysis.
- Treating a successful socket write as durable business completion.
- Broadcasting from process-local state while deploying multiple instances.

## Senior Engineer Thinking

Choose WebSocket when independent client-to-server and server-to-client messages justify a persistent channel. Put domain guarantees in commands, transactions, acknowledgements, and durable cursors. Treat the connection as disposable and make reconnect, backpressure, fan-out, and deployment behavior part of the design.

## Exercises

1. Define a message envelope and acknowledgement flow for a collaborative document editor.
2. Calculate a memory budget for 10,000 connections with a 64 KiB outgoing queue cap.
3. Design room fan-out across three connection processes when the broker is temporarily unavailable.

## Review Questions

1. What does the opening handshake establish, and what does it leave to the application?
2. Why is a socket write not the same as a durable command acknowledgement?
3. How can a service recover messages after a connection drops?
4. What are two independent forms of backpressure a WebSocket service must control?

## Summary

WebSocket is a full-duplex framed transport, not a complete messaging system. Use a tested protocol implementation, validate versioned envelopes, authenticate and authorize connections and messages, bound queues and payloads, add acknowledgements or cursors for durable work, and design reconnection and multi-instance fan-out explicitly.

## References

- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
- [WHATWG WebSocket API](https://websockets.spec.whatwg.org/)
- [MDN: The WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [PHP-FPM documentation](https://www.php.net/manual/en/install.fpm.php)

