---
title: "008: HTTP QUERY"
---

This RFC proposes adding the HTTP QUERY method ([RFC 10008][rfc10008]) to the
Connect protocol as a third unary request form, alongside POST and GET. A QUERY
request is a unary POST request with the method changed to QUERY and nothing
else. It is allowed for exactly the RPCs that allow GET today: unary RPCs
declared with `idempotency_level = NO_SIDE_EFFECTS`. Servers accept it by
default and can turn it off. Clients send it only when the user opts in, and
never fall back to POST on their own.

The proposed change to the protocol specification accompanies this RFC on the
same branch of this repository. This document focuses on the rationale and the
anticipated impact on users and implementations.

## Why

Connect's GET support exists so that side-effect-free RPCs can be cached and
retried by browsers and intermediaries. It works, but it puts the whole request
message in the URL, which causes three problems:

* **URL length limits.** Browsers, proxies, CDNs, and servers all cap URL
  length, often at a few kilobytes. Messages that exceed the cap can't use GET.
  Today clients either fail or fall back to POST, which gives up the benefits
  of GET.
* **Base64 for binary payloads.** Protobuf messages are binary, so they have to
  be base64-encoded and percent-encoded to fit in a URL. This inflates the
  payload and makes requests harder to read and debug.
* **Request data in URLs.** URLs are far more likely than request content to be
  logged, recorded in browser history, or sent in `Referer` headers. Some
  request messages shouldn't end up there.

POST avoids all of these, but it carries no semantics: HTTP treats POST as
neither safe nor idempotent, so intermediaries and HTTP clients may not
automatically retry it, and caches don't store its responses by default.

QUERY, standardized in RFC 10008, is the method that fills this gap. It carries
its request in the content, like POST, but is registered as safe and
idempotent, so it can be retried automatically after a connection failure.
Its responses are cacheable, with a cache key that incorporates the request
content and its media type and encoding. Redirects (301, 302, 307, and 308)
preserve the method, and conditional requests work as they do for GET. For
Connect, this means large and binary messages can use a method that signals
"no side effects" without any URL encoding.

We don't want to oversell caching. QUERY is new, and cache support for it is
young: many caches will not store QUERY responses at first, and caches that do
must read the whole request content to compute a key, which is more work than
keying on a URL. For now, the dependable benefits are the semantics (retry
safety, intent visible to infrastructure) and the removal of GET's URL
constraints. Cache hits are a bonus that will grow as support does.

## The design

The detailed rules are in the accompanying specification change. In summary:

### Wire format

A **Unary-Query-Request** is a **Unary-Request** with a different method. In
the notation of the protocol specification:

* **Unary-Query-Request** &rarr; Unary-Query-Request-Headers Bare-Message
* **Unary-Query-Request-Headers** &rarr; Unary-Query-Call-Specification \*Leading-Metadata
* **Unary-Query-Call-Specification** &rarr; Method-Query Path Unary-Content-Type \[Connect-Protocol-Version\] \[Timeout\] \[Content-Encoding\] \[Accept-Encoding\]
* **Method-Query** &rarr; ":method QUERY"

Every rule that applies to a unary POST request applies unchanged: the same
path, `Content-Type: application/<codec>`, request content, `Content-Encoding`,
`Accept-Encoding`, `Connect-Protocol-Version`, `Connect-Timeout-Ms`, and
metadata headers. The response is an ordinary **Unary-Response**, including the
error format and HTTP status mapping. For example:

```http
> QUERY /connectrpc.greet.v1.GreetService/Greet HTTP/1.1
> Host: demo.connectrpc.com
> Content-Type: application/json
>
> {"name": "Buf"}

< HTTP/1.1 200 OK
< Content-Type: application/json
<
< {"greeting": "Hello, Buf!"}
```

None of the GET query parameters are used, and any URI query is ignored as it is
for POST. The method token is the exact uppercase string `QUERY`. QUERY applies
to the Connect protocol only; gRPC and gRPC-Web remain POST-only.

### Eligibility

A procedure accepts QUERY if and only if it is unary and declared
`idempotency_level = NO_SIDE_EFFECTS`. This is the same test that gates GET, and
implementations reuse it rather than adding a second one. Streaming procedures
and procedures with any other idempotency level never accept QUERY.

### Server behavior

Servers accept QUERY for every eligible procedure by default and provide an
option to disable it, shaped to fit each implementation's existing server
options. A disabled server behaves as if QUERY were an unknown method.

A QUERY request to an ineligible procedure, or to any procedure when QUERY is
disabled, receives `405 Method Not Allowed` with an `Allow` header. `Allow`
lists QUERY exactly when the procedure would accept it. Otherwise, the request
is handled exactly like a unary POST: content type and codec negotiation,
compression, timeouts, size limits, and interceptors. Handlers and interceptors
that can observe the HTTP method see `QUERY`.

A missing or unsupported `Content-Type` receives `415 Unsupported Media Type`,
as for POST. Servers that already advertise supported types with `Accept-Post`
on 415 responses should send `Accept-Query` on 415 responses to QUERY requests.
RFC 10008 defines `Accept-Query` as a Structured Field List ([RFC 9651][rfc9651])
of media types, so the value is a list of tokens such as
`application/proto, application/json`. It lists only types usable with QUERY:
unary Connect content types, and no gRPC, gRPC-Web, or streaming types.

Wherever an implementation treats GET specially on the response side for the
benefit of caches, such as adding `Vary` or answering with `304`, it does the
same for QUERY. It doesn't add `Vary: Content-Type`, because RFC 10008 already
puts the request content type in the cache key.

### Client behavior

QUERY is off by default. Each client gets an option that mirrors its existing
"use HTTP GET" option in naming, placement, and documentation. The option only
affects eligible procedures, using the same idempotency information that gates
GET. Everything else keeps using POST. gRPC and gRPC-Web transports ignore the
option, as they ignore the GET option.

Requests are built exactly as for POST with only the method changed, and no
size limit applies. If an implementation uses deterministic serialization for
GET requests, it uses it for QUERY requests too, because cache keys include the
request content.

### Precedence between GET, QUERY, and POST

QUERY replaces POST. It does not replace GET. When a client has GET enabled and
a request would be sent with GET today (for example, because it fits any
configured size limit), it is still sent with GET. Otherwise, if QUERY is
enabled, the request is sent with QUERY. Otherwise it is sent with POST. In
particular, where the GET option has a mode that falls back to POST when the URL
is too long, that fallback becomes QUERY once QUERY is enabled.

### No automatic downgrade

If a server or an intermediary rejects a QUERY request (with 405, 501, or
anything else), the client surfaces the error like any other unary HTTP error
and doesn't retry with POST. Clients ignore `Accept-Query` on responses.

### CORS

QUERY isn't a CORS-safelisted method, so browsers always send a preflight
request before a cross-origin QUERY request. Servers and CORS middleware that
support cross-origin use must list QUERY in `Access-Control-Allow-Methods`.
Same-origin requests need no preflight. Documentation and CORS helpers
for each implementation need to cover this.

### Out of scope

* Using `Location`, `Content-Location`, or `303` to hand out GET-able result
  URIs.
* Sending `Accept-Query` on successful responses, or acting on it in clients.
* Automatic method negotiation or discovery.
* QUERY for streaming RPCs.

## Anticipated complications

* **Intermediaries that don't know QUERY.** Some firewalls, API gateways, CDNs,
  and reverse proxies reject methods they don't recognize, answer them with 405
  or 501, or strip them. Because QUERY is opt-in for clients, this only affects
  users who turn it on, and the failure is a clear HTTP error. They need to
  check their path end to end. We deliberately don't mitigate this with an
  automatic downgrade (see below).
* **Caches that don't key on request content.** A cache that treats QUERY like
  GET and keys on the URL alone could serve one query's response for another.
  RFC 10008 requires the cache key to incorporate the request content, and
  conforming caches do. Users who put a cache in front of a Connect server
  should confirm theirs does, or that it doesn't cache QUERY at all. Several
  caches don't cache QUERY yet, in which case users get correct behavior but
  none of the caching benefit.
* **Mandatory preflight in browsers.** Browsers send a CORS preflight before
  every cross-origin QUERY request, even when GET would have been a simple
  request. Cross-origin browser clients pay an extra round trip unless the
  preflight response is cached with `Access-Control-Max-Age`. Servers must allow
  QUERY in their CORS configuration or the browser blocks the call.
* **HTTP stacks that reject the method token.** Some HTTP clients, servers,
  and proxies validate methods against a fixed list and refuse unknown ones, and
  some don't permit a body on a method they don't recognize. Implementations
  need to check their stacks and document any that can't send or receive QUERY.
  The method must be sent as the exact string `QUERY`, because `fetch` does not
  normalize the case of this method.
* **Middleware and access rules keyed on POST.** Authentication, authorization,
  rate-limiting, logging, and metrics code that matches on the method POST won't
  match QUERY requests. Users who enable QUERY need to review this
  configuration. Handler-side code that reads the method will see `QUERY`.
* **Servers accept a new method by default.** After upgrading, a server begins
  accepting QUERY for its eligible procedures without any configuration. We
  think this is acceptable because those procedures have already been declared
  to have no side effects, are already reachable with GET, and are already
  reachable with POST, so QUERY exposes no new capability. Handler logic,
  authentication, and size limits are unchanged. The exception is the
  middleware case above: access rules that match on method, and any
  infrastructure that blocks unknown methods and expects the server to do the
  same, may now see traffic they didn't before. Users who need the old
  behavior can disable QUERY with the server option.
* **No automatic downgrade.** A client that enables QUERY and meets an
  intermediary that rejects it gets an error and has to fix the path or turn the
  option off. This is less forgiving than silently falling back, but it keeps
  behavior predictable, avoids a doubled request and a hidden latency cost, and
  doesn't silently drop whatever semantics the user turned QUERY on for.

## Alternatives

* **Do nothing**: GET and POST continue to work. Users with large or sensitive
  messages stay on POST and lose the safe and idempotent semantics, or accept
  GET's URL constraints. As QUERY support spreads through HTTP infrastructure,
  Connect would be unable to benefit from it.
* **Automatic downgrade to POST**: Clients could try QUERY and fall back to POST
  when it's rejected. This would hide failures behind extra round trips, require
  clients to remember per-server state, and make it hard to tell which method a
  call used. It could also turn a rejected call into a duplicated one if the
  first request partly succeeded. Explicit opt-in and explicit errors are easier
  to reason about.
* **QUERY replacing GET outright**: It would be simpler to say that QUERY is the
  new way to send side-effect-free calls. But GET is understood by every cache,
  browser, and proxy, and can be typed into a URL bar. For requests that fit in
  a URL, GET remains the most widely compatible and most cacheable option.
  QUERY replaces POST as the fallback instead.
* **Extending eligibility to `IDEMPOTENT` procedures**: RFC 10008 defines QUERY
  as safe. An `IDEMPOTENT` procedure may change state, and intermediaries and
  caches may treat QUERY as free of side effects. Only `NO_SIDE_EFFECTS`
  procedures are eligible, which is also the existing test for GET.
* **GET with a body**: RFC 9110 gives content in a GET request no defined
  semantics, and many servers, proxies, and caches drop or reject it. That
  wouldn't be interoperable, and it would mean that GET requests no longer carry
  all their information in the URL.
* **Server opt-in instead of on by default**: Servers could accept QUERY only
  when configured to. This avoids the new-method concern above, but it means
  that clients get a 405 from every server that hasn't been configured, which
  would make the client option unreliable and require changes in every server
  deployment. Because eligible procedures are already declared side-effect free
  and already served with GET, we think on by default, with an off switch, is
  the better trade-off.

## References

* [RFC 10008: The HTTP QUERY Method][rfc10008]
* [RFC 9110: HTTP Semantics][rfc9110]
* [RFC 9111: HTTP Caching][rfc9111]
* [RFC 9651: Structured Field Values for HTTP][rfc9651]
* [Connect protocol specification](/docs/protocol/), in particular the
  [Unary-Get-Request](/docs/protocol/#unary-get-request) and
  [Unary-Query-Request](/docs/protocol/#unary-query-request) sections
* [Get requests and caching](/docs/go/get-requests-and-caching/), the existing
  documentation for GET support

[rfc10008]: https://www.rfc-editor.org/rfc/rfc10008.html
[rfc9110]: https://www.rfc-editor.org/rfc/rfc9110.html
[rfc9111]: https://www.rfc-editor.org/rfc/rfc9111.html
[rfc9651]: https://www.rfc-editor.org/rfc/rfc9651.html
