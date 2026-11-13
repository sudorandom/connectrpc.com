---
title: GET, QUERY, and Caching
---

Connect supports HTTP GET for idempotent, side-effect-free RPCs. This makes
the requests cacheable by browsers, CDNs, and intermediate proxies. Connect also
supports the HTTP QUERY method for the same RPCs, which sends the message in the
request body instead of the URL. See [HTTP QUERY](#http-query) below.

First, configure your server to handle HTTP GET. See the server-side guide
for your backend:

* [Connect Go](/docs/go/get-requests-and-caching/)
* [Connect Node](/docs/node/get-requests-and-caching/)
* [Connect Python](/docs/python/get-requests-and-caching/)

To opt a procedure into GET, mark it as side-effect-free with
[`MethodOptions.IdempotencyLevel`][idempotency-level]:

```protobuf
service ElizaService {
  rpc Say(SayRequest) returns (SayResponse) {
    option idempotency_level = NO_SIDE_EFFECTS;
  }
}
```

GET requests are off by default. Opt in by setting `getConfiguration` on
`ProtocolClientConfig`:

```kotlin
val config = ProtocolClientConfig(
    getConfiguration = GETConfiguration.Enabled,
    ...
)
```

`GETConfiguration.EnabledWithFallback(maxMessageBytes)` is also available; it
sends a GET only when the encoded message fits within `maxMessageBytes` and
falls back to POST otherwise.

## HTTP QUERY

GET puts the whole message in the URL, so large messages can run into URL length
limits, and binary messages must be base64-encoded. The HTTP QUERY method
([RFC 10008][rfc-10008]) is safe and idempotent like GET, but carries the
message in the request body like POST. Connect's [unary QUERY
request](/docs/protocol/#unary-query-request) is a unary POST request with the
method changed to QUERY and nothing else.

QUERY is available for the same RPCs as GET: unary RPCs marked
`NO_SIDE_EFFECTS`. Servers accept it by default, and can turn it off. See the
server documentation linked above for details. The gRPC and gRPC-Web protocols
ignore the option and always use POST.

QUERY requests are off by default. Opt in by setting `queryConfiguration` on
`ProtocolClientConfig`:

```kotlin
val config = ProtocolClientConfig(
    queryConfiguration = QUERYConfiguration.Enabled,
    ...
)
```

QUERY replaces POST, not GET. If you enable both `getConfiguration` and
`queryConfiguration`, requests that qualify for GET are still sent with GET.
With `GETConfiguration.EnabledWithFallback(maxMessageBytes)`, a message that
doesn't fit within `maxMessageBytes` is sent with QUERY instead of POST.

If the server or something between you and the server rejects a QUERY request,
for example with a 405 or 501, the call fails with an error. The client never
retries a rejected QUERY request as POST, so only enable it for servers that
support it.

Some proxies, CDNs, and firewalls reject methods they don't recognize, and caches generally don't store QUERY responses yet, so don't expect cache hits from QUERY today. Check that your infrastructure forwards
QUERY before relying on it.

[rfc-10008]: https://www.rfc-editor.org/rfc/rfc10008.html
[idempotency-level]: https://github.com/protocolbuffers/protobuf/blob/e5679c01e8f47e8a5e7172444676bda1c2ada875/src/google/protobuf/descriptor.proto#L795
