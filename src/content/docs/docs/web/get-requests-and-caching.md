---
title: GET, QUERY, and Caching
---

Connect supports performing idempotent, side-effect free requests using an HTTP
GET-based protocol. This makes it easier to cache certain kinds of requests in
the browser, on your CDN, or in proxies and other middleboxes. Connect also
supports the HTTP QUERY method for the same procedures, which sends the message
in the request body instead of the URL. See [HTTP QUERY](#http-query) below.

First, configure your server to handle HTTP GET requests using Connect. Refer
to the documentation that corresponds to your server:

* [Connect Go](/docs/go/get-requests-and-caching/)
* [Connect Node](/docs/node/get-requests-and-caching/)
* [Connect Python](/docs/python/get-requests-and-caching/)

Afterwards, ensure that a new enough version of `@connectrpc/connect-web` is set
up; HTTP GET support is available in Connect Web v0.9.0 or newer. Then, you can
specify the `useHttpGet` option when creating the Connect transport:

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

## HTTP QUERY

GET puts the whole message in the URL, so large messages can run into URL length
limits, and binary messages must be base64-encoded. The HTTP QUERY method
([RFC 10008](https://www.rfc-editor.org/rfc/rfc10008.html)) is safe and
idempotent like GET, but carries the message in the request body like POST.
Connect's [unary QUERY request](/docs/protocol/#unary-query-request) is a unary
POST request with the method changed to QUERY and nothing else.

QUERY is available for the same procedures as GET: unary procedures marked
`NO_SIDE_EFFECTS`. Servers accept it by default, and can turn it off. See the
server documentation linked above for details.

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

If the server or something between the browser and the server rejects a QUERY
request, for example with a 405 or 501, the call fails with an error. Connect
never retries a rejected QUERY request as POST, so only enable `useHttpQuery`
for servers that support it.

A few practical points before you enable QUERY:

* Browsers always send a CORS preflight before a cross-origin QUERY request,
  because QUERY isn't one of the methods allowed without one. Unlike GET, there
  is no way to avoid the preflight. Your server must allow QUERY in
  `Access-Control-Allow-Methods`. See the [CORS documentation](/docs/cors/).
* Some proxies, CDNs, and firewalls reject methods they don't recognize, and
  many caches don't store QUERY responses yet. Check that your infrastructure
  forwards QUERY before relying on it.
