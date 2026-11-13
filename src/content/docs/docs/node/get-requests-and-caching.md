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
service ElizaService {
  rpc Say(stream SayRequest) returns (SayResponse) {
    option idempotency_level = NO_SIDE_EFFECTS;
  }
}
```

Handlers will automatically support GET requests using this option, but ensure
you have a recent enough version of Connect Node; v0.9.0 or newer is required.

It is still necessary to opt-in to HTTP GET on your client, as well. If you are
using a Node client, you need to specify the `useHttpGet` option when creating
the Connect transport:

```js
const transport = createConnectTransport({
  baseUrl: "https://demo.connectrpc.com",
  useHttpGet: true,
});
const client = createClient(ElizaService, transport);
const response = await client.say(request);
console.log(response);
```

Methods annotated as side-effect free will use GET requests. All other requests
will continue to use POST.

For other clients, see their respective documentation pages:

* [Connect Go](/docs/go/get-requests-and-caching/)
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
`NO_SIDE_EFFECTS`. It is only used with the Connect protocol. gRPC and gRPC-web
always use POST.

Handlers accept QUERY requests by default, so you don't need to change anything
on your server. QUERY requires a recent Node.js release. See the
[Connect-ES documentation](https://github.com/connectrpc/connect-es) for the
supported versions. [Fastify](https://fastify.dev/) routes QUERY requests since
v5.11.0, and earlier versions respond with a 404. To turn QUERY off, set
`allowHttpQuery` to `false` in the router options of your adapter or plugin.
Procedures then respond to QUERY with `405 Method Not Allowed`:

```ts
await server.register(fastifyConnectPlugin, {
  routes,
  allowHttpQuery: false,
});
```

Clients must opt in with the `useHttpQuery` option when creating the Connect
transport:

```js
const transport = createConnectTransport({
  baseUrl: "https://demo.connectrpc.com",
  useHttpQuery: true,
});
const client = createClient(ElizaService, transport);
const response = await client.say(request);
console.log(response);
```

Methods annotated as side-effect free will use QUERY requests. All other
requests will continue to use POST.

QUERY replaces POST, not GET. If you enable both `useHttpGet` and
`useHttpQuery`, side-effect free requests are still sent with GET.

If the server or something between you and the server rejects a QUERY request,
for example with a 405 or 501, the call fails with an error. Connect never
retries a rejected QUERY request as POST, so only enable `useHttpQuery` for
servers that support it.

A few practical points before you enable QUERY:

* Browsers always send a CORS preflight before a cross-origin QUERY request,
  because QUERY isn't one of the methods allowed without one. The `cors` object
  from `@connectrpc/connect` includes QUERY in `cors.allowedMethods`. See the
  [CORS documentation](/docs/cors/).
* Some proxies, CDNs, and firewalls reject methods they don't recognize, and
  caches generally don't store QUERY responses yet, so don't expect cache hits from QUERY today. Check that your infrastructure
  forwards QUERY before relying on it.

## Caching

Using GET or QUERY requests will not necessarily automatically make browsers or
proxies cache your RPCs. To ensure that requests are allowed to be cached, a handler
should also set the appropriate headers.

For example, you may wish to set the `Cache-Control` header with a `max-age`
directive:

```js
say(request, context) {
  // ...
  context.responseHeader.set("Cache-Control", "max-age=604800");
}
```

This would instruct agents and proxies that the request may be cached for up to
7 days, after which it must be re-requested. There are other
[`Cache-Control` Response Directives][cache-control-response-directives] that
may be useful for your application as well; for example, the `private` directive
would specify that the request should only be cached in private caches, such as
the user agent itself, and *not* CDNs or reverse proxies&mdash;this would be
appropriate, for example, for authenticated requests.

## Distinguishing GET and QUERY Requests

In some cases, you might want to introduce behavior that only occurs when
handling HTTP GET requests. This can be accomplished using
`context.requestMethod`, which is `"POST"`, `"GET"`, or `"QUERY"`:

```ts
if (context.requestMethod == "GET") {
  context.responseHeader.set("Cache-Control", "max-age=604800");
}
```

[connect-grpc-bridge-docs]: https://www.envoyproxy.io/docs/envoy/v1.26.0/configuration/http/http_filters/connect_grpc_bridge_filter#config-http-filters-connect-grpc-bridge
[rfc-10008]: https://www.rfc-editor.org/rfc/rfc10008.html
[idempotency-level]: https://github.com/protocolbuffers/protobuf/blob/e5679c01e8f47e8a5e7172444676bda1c2ada875/src/google/protobuf/descriptor.proto#L795
[cache-control-response-directives]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#response_directives
