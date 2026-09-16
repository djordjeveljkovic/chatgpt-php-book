---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 147
title: SSRF
slug: ssrf
status: complete
summary: ../../_ai/chapter-summaries/147-ssrf-summary.md
---

# Chapter 147 — SSRF

## Why This Matters

Server-side request forgery (SSRF) occurs when an attacker controls a URL or destination that the server fetches. The server may reach private services, cloud metadata, loopback administration endpoints, or internal networks unavailable from the public internet. DNS rebinding and redirects can defeat a single string check.

## Mental Model

```text
untrusted URL → parse and policy-check → resolve/connect safely
             → restrict redirects, protocols, ports, and response size
```

The strongest design is an allow-list of named remote resources rather than arbitrary URLs. If arbitrary destinations are a product requirement, use a dedicated egress proxy with network policy and monitoring.

## Validation Is Not One Regex

Parse the URL, allow only `https` (or an explicitly required scheme), restrict ports and hosts, resolve DNS, and reject private, loopback, link-local, multicast, and reserved address ranges for the deployment network. Re-check the address at connection time when the client supports it. Disable or strictly validate redirects; the final destination needs the same policy.

Do not trust the `Host` header or a DNS result captured earlier in the request. IPv4, IPv6, integer, hexadecimal, and encoded forms can represent the same address. Use a mature HTTP client and network egress controls rather than maintaining a partial parser.

## Resource Limits and Isolation

Set connection, TLS, total, redirect, and read timeouts; cap response bytes and decompression; restrict methods and headers; and avoid forwarding application credentials. Run fetch workers with a network identity that cannot reach administrative planes. Log destination policy decisions, response class, duration, and blocked attempts without secrets.

## PHP Boundary

```php
<?php

declare(strict_types=1);

function allowedEndpoint(string $name): string
{
    $endpoints = [
        'weather' => 'https://weather.example.test/current',
        'billing' => 'https://billing.example.test/status',
    ];

    return $endpoints[$name] ?? throw new InvalidArgumentException('Unknown endpoint');
}
```

Named endpoints remove URL choice from the client. A real implementation still needs TLS verification, timeouts, response-size limits, and authentication appropriate to the upstream.

## Testing and Common Mistakes

Test private IPv4/IPv6 ranges, redirects to private addresses, DNS changes, non-HTTP schemes, unusual ports, oversized responses, slow connections, TLS failures, and credential forwarding. Common mistakes include blocking only `127.0.0.1`, validating before a redirect, allowing `file://` or `gopher://`, and assuming a public hostname cannot resolve privately.

## Senior Engineer Thinking

SSRF is an egress authorization problem. Restrict destinations by name and network policy, make every request bounded, isolate the fetcher, and treat DNS and redirects as part of the security boundary. Application validation and infrastructure egress controls should reinforce each other.

## Exercises

1. Define an allow-list for three upstream operations and their permitted methods and ports.
2. Design a redirect policy that revalidates every target.
3. Add response-size and timeout limits to an HTTP client and test slow/oversized responses.

## Review Questions

1. Why is checking a URL's text insufficient?
2. Which address ranges should an arbitrary fetcher normally reject?
3. Why must redirects and DNS resolution be included in policy?
4. What infrastructure control limits SSRF impact if application validation fails?

## Summary

Prevent SSRF with named destination allow-lists or a controlled egress proxy, strict scheme/host/address/port policy, redirect and DNS revalidation, bounded HTTP clients, least-privilege network identity, and monitoring. A URL regex is not a network security boundary.

## References

- [OWASP: Server-Side Request Forgery Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP: SSRF](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [PHP Manual: cURL](https://www.php.net/manual/en/book.curl.php)
