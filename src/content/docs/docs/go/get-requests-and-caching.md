---
title: GET, QUERY, and Caching
---

Connect supports performing idempotent, side-effect free requests using an HTTP
GET-based protocol. This makes it easier to cache certain kinds of requests in
the browser, on your CDN, or in proxies and other middleboxes. Connect also
supports the HTTP QUERY method for the same procedures, which sends the message
in the request body instead of the URL. See [HTTP QUERY](#http-query) below.

:::note
This functionality is **only** supported when using the Connect
protocol&mdash;using a Connect client with a Connect service. When using gRPC
clients with Connect servers, or Connect clients with gRPC servers, all
requests will use HTTP POST.

If you need HTTP GET support when talking to a vanilla gRPC server, you could
use a proxy. Envoy supports translating between Connect clients and gRPC servers
using the [Connect-gRPC Bridge][connect-grpc-bridge-docs].
:::

If you are using clients to make query-style requests, you may want the ability
to use Connect HTTP GET request support. To opt-in for a given procedure, you
must mark it as being side-effect free using the
[`MethodOptions.IdempotencyLevel`][idempotency-level] option:

```protobuf
service GreetService {
  rpc Greet(GreetRequest) returns (GreetResponse) {
    option idempotency_level = NO_SIDE_EFFECTS;
  }
}
```

Handlers will automatically support GET requests using this option.

It is still necessary to opt-in to HTTP GET on your client, as well. If you are
using a Go client, you would specify the `connecthttp.WithHTTPGet` option when
creating the transport.

```go
client := greetv1connect.NewGreetServiceClient(
	connect.NewClient(
		connecthttp.NewTransport(
			http.DefaultClient,
			"http://localhost:8080",
			connecthttp.WithHTTPGet(),
		),
	),
)
```

Methods annotated as side-effect free will use GET requests. All other requests
will continue to use POST.

For other clients, see their respective documentation pages:

* [Connect Node](/docs/node/get-requests-and-caching/)
* [Connect Web](/docs/web/get-requests-and-caching/)
* [Connect Kotlin](/docs/kotlin/get-requests-and-caching/)
* [Connect Python](/docs/python/get-requests-and-caching/)

## HTTP QUERY

GET puts the whole message in the URL, so large messages can run into URL length
limits, and binary messages must be base64-encoded. The HTTP QUERY method
([RFC 10008][rfc-10008]) is safe and idempotent like GET, but carries the
message in the request body like POST. Connect's [unary QUERY
request](/docs/protocol/#unary-query-request) is a unary POST request with the
method changed to QUERY and nothing else.

QUERY is available for the same procedures as GET: unary procedures marked
`NO_SIDE_EFFECTS`. It is only used with the Connect protocol. gRPC and gRPC-Web
always use POST.

Handlers accept QUERY requests by default, so you don't need to change
anything on your server. To turn QUERY off, pass `connecthttp.WithoutHTTPQuery`
when mounting the server. Procedures then respond to QUERY with
`405 Method Not Allowed`, and their `Allow` header omits QUERY:

```go
connecthttp.Mount(mux, server, connecthttp.WithoutHTTPQuery())
```

Clients must opt in with `connecthttp.WithHTTPQuery`:

```go
client := greetv1connect.NewGreetServiceClient(
	connect.NewClient(
		connecthttp.NewTransport(
			http.DefaultClient,
			"http://localhost:8080",
			connecthttp.WithHTTPQuery(),
		),
	),
)
```

Methods annotated as side-effect free will use QUERY requests. All other
requests will continue to use POST.

QUERY replaces POST, not GET. If you enable both `WithHTTPGet` and
`WithHTTPQuery`, requests that can be sent as GET still are. A request that
would otherwise be sent as POST is sent as QUERY instead. In particular, when
you set a size limit with `WithHTTPGetMaxURLSize(bytes, true)`, requests whose
URL would exceed the limit fall back to QUERY rather than POST. With `false`, a
request over the limit still fails with `CodeResourceExhausted`:

```go
connecthttp.NewTransport(
	http.DefaultClient,
	"http://localhost:8080",
	connecthttp.WithHTTPGet(),
	connecthttp.WithHTTPGetMaxURLSize(4096, true),
	connecthttp.WithHTTPQuery(),
)
```

If the server or something between you and the server rejects a QUERY request,
for example with a 405 or 501, the call fails with an error. Connect never
retries a rejected QUERY request as POST, so only enable `WithHTTPQuery` for
servers that support it.

A few practical points before you enable QUERY:

* Browsers always send a CORS preflight before a cross-origin QUERY request,
  because QUERY isn't one of the methods allowed without one. See the
  [CORS documentation](/docs/cors/).
* Some proxies, CDNs, and firewalls reject methods they don't recognize, and
  caches generally don't store QUERY responses yet, so don't expect cache hits from QUERY today. Check that your infrastructure
  forwards QUERY before relying on it.
* Clients encode QUERY messages deterministically when the codec supports it,
  because caches key on the request body.

## Caching

Using GET or QUERY requests will not necessarily automatically make browsers or
proxies cache your RPCs. To ensure that requests are allowed to be cached, a handler
should also set the appropriate headers.

For example, you may wish to set the `Cache-Control` header with a `max-age`
directive:

```go
callInfo, ok := connect.CallInfoForServerContext(ctx)
if ok {
	callInfo.ResponseHeader().Set("Cache-Control", "max-age=604800")
}
```

This would instruct agents and proxies that the request may be cached for up to
7 days, after which it must be re-requested. There are other
[`Cache-Control` Response Directives][cache-control-response-directives] that
may be useful for your application as well; for example, the `private` directive
would specify that the request should only be cached in private caches, such as
the user agent itself, and *not* CDNs or reverse proxies&mdash;this would be
appropriate, for example, for authenticated requests.

QUERY responses are cacheable in principle, but caches generally don't store
them yet, so don't expect cache hits from QUERY today. Handlers respond to QUERY requests with
`Vary: Accept-Encoding`, as they do for GET.

Handlers can also support HTTP [conditional
requests][conditional-requests], for both GET and QUERY. Set an `Etag` header identifying the current
version of the response. When a client repeats the request, its cache sends
that value back in the `If-None-Match` header. If the ETag still matches, the
handler can skip its work and return `connecthttp.NewNotModifiedError()` to
tell the client its cached copy is still fresh:

```go
callInfo, ok := connect.CallInfoForServerContext(ctx)
if !ok {
	return nil, connect.NewError(connect.CodeInternal, "no call info in context")
}
callInfo.ResponseHeader().Set("Etag", etag)
serverInfo, ok := connecthttp.ServerInfoForContext(ctx)
cacheable := ok &&
	(serverInfo.HTTPMethod() == http.MethodGet || serverInfo.HTTPMethod() == "QUERY")
if cacheable && callInfo.RequestHeader().Get("If-None-Match") == etag {
	return nil, connecthttp.NewNotModifiedError()
}
// ...build the response as usual...
```

"Not modified" is not a failure. Like `io.EOF`, it's a sentinel error used as
a signal, because a handler has no other way to return without a response
message. On the wire it becomes HTTP's 304 Not Modified status, which caches
treat as a successful revalidation. Clients detect the signal with
`connecthttp.IsNotModifiedError` and reuse the cached response.

## Distinguishing GET and QUERY Requests

In some cases, you might want to introduce behavior that only occurs when
handling HTTP GET requests. This can be accomplished using the `HTTPMethod`
method on the `connecthttp.ServerInfo` type in context, which returns `"POST"`,
`"GET"`, or `"QUERY"`:

```go
callInfo, ok := connect.CallInfoForServerContext(ctx)
info, httpOK := connecthttp.ServerInfoForContext(ctx)
if ok && httpOK && info.HTTPMethod() == http.MethodGet {
	callInfo.ResponseHeader().Set("Cache-Control", "max-age=604800")
}
```

[connect-grpc-bridge-docs]: https://www.envoyproxy.io/docs/envoy/v1.26.0/configuration/http/http_filters/connect_grpc_bridge_filter#config-http-filters-connect-grpc-bridge
[cache-control-response-directives]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#response_directives
[conditional-requests]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Conditional_requests
[rfc-10008]: https://www.rfc-editor.org/rfc/rfc10008.html
[idempotency-level]: https://github.com/protocolbuffers/protobuf/blob/e5679c01e8f47e8a5e7172444676bda1c2ada875/src/google/protobuf/descriptor.proto#L795
