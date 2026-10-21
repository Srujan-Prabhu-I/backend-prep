# Chapter 7: HTTP: The Language of Client–Server Communication

## Where We Are

Chapter 6 opened the network between two machines: how bytes are wrapped in TCP segments, IP packets and Ethernet frames, how DNS turns a name into an address, and how TCP delivers a reliable, ordered byte stream (or QUIC delivers several independent streams over UDP). Section 6.9 ended on a boundary: **TCP carries bytes, not messages.** When a server's `read` returns 312 bytes, TCP cannot say where one request ends and the next begins, what the client is asking for, what format the bytes are in, whether a cache in the middle may keep the answer, or whether the client may safely send the request again if the connection breaks.

**HTTP** is the application-layer protocol that answers those questions for the web and for most service-to-service APIs. It defines how a client asks a server to act on a resource (a **request**), how the server reports the outcome (a **response**), how intermediaries such as proxies, gateways and caches can route and cache traffic without understanding your business logic, and how all of this is framed on the wire in three versions: HTTP/1.1 (text over TCP), HTTP/2 (binary frames over TCP) and HTTP/3 (binary frames over QUIC).

This chapter is the protocol itself. Every later chapter that touches the web relies on it: browsers (Chapter 9), servers and proxies (Chapter 10), API design (Chapter 13), sessions and cookies (Chapter 19), caching layers (Chapter 21), and retries (Chapter 24).

**After this chapter you can:**
- explain what "HTTP is stateless" means and does not mean, and separate HTTP's semantics from its three wire formats;
- take a URL apart, percent-encode correctly, and say which part of a URL goes where (and which part never leaves the client);
- read and write an HTTP/1.1 message byte by byte, explain how a receiver finds the end of a message (`Content-Length` versus chunked), and trace a request-smuggling attack;
- classify every method by safety, idempotency and cacheability, and state the difference between `PUT`, `PATCH` and `POST`;
- choose the right status code and state what each one does **not** tell the client;
- explain the main headers, media types, character encoding, content negotiation, compression and range requests;
- distinguish a connection, a request, a stream and a thread, and explain keep-alive, pipelining and head-of-line blocking;
- explain how HTTP/2 frames, streams, HPACK and flow control work, and what HTTP/3 changes;
- state how each redirect code treats the method and body;
- design HTTP caching precisely: freshness, validation, `no-cache` versus `no-store`, `private` versus `public`, `s-maxage`, `Vary`;
- use `ETag` and `If-Match` to prevent lost updates;
- explain the wire syntax of cookies, the limits servers enforce, and why a request that timed out has an unknown outcome;
- read an annotated `curl -v` trace and a hex dump of a request.

**Interview questions this chapter prepares you for:**
1. What is the structure of an HTTP request and response on the wire? How does the receiver find the end of the headers and the end of the body?
2. What is the difference between `Content-Length` and chunked encoding? What is request smuggling?
3. What is the difference between a safe and an idempotent method? Why is `DELETE` idempotent even if the second call returns `404`?
4. What is the difference between `PUT`, `PATCH` and `POST`?
5. What is the difference between `401` and `403`? When would you return `404` instead of `403`?
6. What is the difference between `400` and `422`, and among `500`, `502`, `503` and `504`?
7. What does a `200 OK`, or a timeout, tell you about whether the server's transaction committed?
8. How do `301`, `302`, `303`, `307` and `308` differ for a `POST`?
9. What does keep-alive do? Why does an idle keep-alive connection not need a thread on a modern server?
10. HTTP/1.1 versus HTTP/2 versus HTTP/3: what changed and why? Why does HTTP/2 still suffer from head-of-line blocking?
11. How does HTTP caching work (`Cache-Control`, `ETag`, `304`)? What is the difference between `no-cache` and `no-store`?
12. How do you stop two clients from overwriting each other's changes over stateless HTTP?
13. A `POST /payments` times out. Why is retrying it dangerous, and how do you make it safe?

**Scope.** Here: HTTP's message model, URIs, HTTP/1.1 framing, methods, status codes, headers, representations, negotiation, connections, HTTP/2 and HTTP/3 framing, redirects, caching, conditional requests, the wire syntax of cookies, limits and timeouts, and wire-level tracing. TCP, UDP, QUIC as a transport and DNS are Chapter 6. TLS and certificates (the "S" in HTTPS) are Chapter 8. What browsers do with HTTP (the same-origin policy, CORS, cookie security in the browser) is Chapter 9; sessions and CSRF are Chapter 19. Reverse proxies, CDNs and gateways are Chapter 10. How servers execute many requests concurrently is Chapter 11. REST API design, idempotency keys, pagination and error formats are Chapter 13. Application caches such as Redis are Chapter 21.

---

## 7.1 Why HTTP Exists

### Definition

**HTTP** (Hypertext Transfer Protocol) is a **stateless, application-level request–response protocol**. A **client** (in HTTP terms, a **user agent**) sends a request naming a method and a target **resource**; an **origin server** (the server that owns the resource) returns a response carrying a status code, header fields and an optional body; zero or more **intermediaries** may stand between them. A **resource** is anything that can be named by a URI: a document, an order, a collection of orders, a search. HTTP's meaning is specified once, in **RFC 9110** (HTTP Semantics), and its caching in **RFC 9111**; each wire format has its own specification: **RFC 9112** (HTTP/1.1), **RFC 9113** (HTTP/2), **RFC 9114** (HTTP/3).

### Why it exists

Build a distributed system directly on TCP sockets, and every pair of services must invent answers to the same questions:

1. **Framing.** Where does one message end and the next begin? (Chapter 6, Section 6.9.)
2. **Addressing.** How does one address and port serve thousands of different endpoints (`/users/42`, `/orders`, `/health`), and several domain names?
3. **Intent.** Is the caller only reading (so the request may be retried, prefetched or cached), setting state in a repeatable way, or causing a one-off side effect?
4. **Outcome.** Did it succeed? Did the caller send something invalid, or lack permission? Is the server broken, or a server behind it?
5. **Format.** Is the body JSON in UTF-8, a PNG image, compressed HTML?
6. **Intermediaries.** How can a load balancer, cache or gateway route, compress, rate-limit or cache traffic without knowing your data model?
7. **Evolution.** How can the wire format improve (HTTP/1.1 → HTTP/2 → HTTP/3) without changing any application code?

HTTP answers all seven with one shared contract. Because browsers, mobile SDKs, `curl`, proxies, CDNs and server frameworks all speak it, any of them can read the start of a message (`GET /catalog/items/99`, `Cache-Control: public, max-age=60`, `503`) and act correctly without parsing the body.

### Mechanism 1: what "stateless" means

A common interview answer is that "HTTP is stateless, so the server stores nothing" or "so every request opens a new connection". Both are wrong.

- **What it means.** Each request is **self-contained** at the protocol level: a server or intermediary can interpret request *N* from its own method, target, headers and body, without remembering request *N − 1*. There is no protocol-level "current directory" or "logged-in user". Contrast a stateful protocol such as FTP, where a `RETR file.txt` command means something only after earlier `USER`, `PASS` and `CWD` commands on the same connection.
- **Why it matters.** Because each request carries its own context (for example a session cookie or an `Authorization` token), an L7 load balancer may send request 1 to server A and request 2, even on the same client connection, to server B (Chapter 6, Section 6.13). Statelessness is what lets HTTP services scale horizontally.
- **What it does not mean.** The transport underneath has state (a TCP connection; HTTP/2 even keeps per-connection header-compression state, Section 7.10). And applications have plenty of state: orders in a database, sessions in a store. They build conversations *on top of* HTTP by sending an identifier (a cookie, a token, a resource URL) with every request.

### Mechanism 2: one set of semantics, three wire formats

```text
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ application: handlers, domain objects, JSON or Protobuf encoding          │
 ├──────────────────────────────────────────────────────────────────────────┤
 │ HTTP semantics and caching (RFC 9110, 9111)                               │
 │   methods, target URI, status codes, header fields, content              │
 ├────────────────────────┬────────────────────────┬────────────────────────┤
 │ HTTP/1.1 (RFC 9112)    │ HTTP/2 (RFC 9113)      │ HTTP/3 (RFC 9114)      │
 │ text lines, CRLF,      │ binary frames,         │ binary frames on       │
 │ Content-Length or      │ multiplexed streams,   │ QUIC streams,          │
 │ chunked; one exchange  │ HPACK header           │ QPACK header           │
 │ at a time per conn.    │ compression            │ compression            │
 ├────────────────────────┴────────────────────────┼────────────────────────┤
 │ TLS (optional for HTTP/1.1; Chapter 8)           │ QUIC, with TLS 1.3     │
 ├──────────────────────────────────────────────────┤ built in (Chapter 6)   │
 │ TCP (Chapter 6)                                  ├────────────────────────┤
 │                                                  │ UDP                    │
 └──────────────────────────────────────────────────┴────────────────────────┘
```

Moving from HTTP/1.1 to HTTP/2 or HTTP/3 changes nothing in the top two boxes. `POST /api/orders` with a JSON body that returns `201 Created` and `Location: /api/orders/901` means exactly the same thing whether its bytes travel as text lines, as HTTP/2 frames or as HTTP/3 frames. This is why a proxy can accept HTTP/2 from browsers and speak HTTP/1.1 to your application.

### Concrete: the roles along the path

```text
 user agent (browser, mobile app, curl, another service)
      │
      ▼  optional: chosen by the client or its network
 forward proxy
      │
 ═════╪═════ internet ══════════════════════════════════
      ▼  run by the server side, near users
 CDN / shared cache
      │
      ▼  run by the server side: TLS, routing, rate limits
 reverse proxy / API gateway / L7 load balancer
      │
      ▼  where the resource lives and the business logic runs
 origin server (your application process)
```

- The **user agent** starts requests. The **origin server** is the one that owns the resource; even when a CDN answers from its cache, your application is still "the origin".
- **Intermediaries** are HTTP-aware hops in between. A **forward proxy** acts for the client (a corporate egress proxy). A **reverse proxy** or **gateway** acts for the server: clients connect to it as if it were the origin, and it forwards to the real servers (Chapter 10). A **tunnel** is a blind relay, usually opened with the `CONNECT` method, that copies bytes (often an encrypted TLS session) without reading the HTTP inside.

### Guarantees and non-guarantees

HTTP guarantees a uniform way to name resources, state intent, describe content and report outcomes, over any reliable transport.

It does **not** provide confidentiality or authentication by itself: over plain `http://`, headers, cookies, tokens and bodies cross the network in clear text (HTTPS, Chapter 8). It does not force servers to respect method semantics: a server can delete data in a `GET` handler, and then a crawler or a browser prefetch will delete it too. And it is client-initiated: a server cannot send an unrequested message except by keeping a response open (**server-sent events**) or by switching the connection to another protocol (**WebSocket**, via `101 Switching Protocols`; both are covered in Chapter 10).

### Failure and diagnosis

**State hidden in a "stateless" tier.** A developer keeps each user's shopping cart in a map in the application process's memory. With one instance in testing, everything works. In production, behind a load balancer with four instances, `POST /cart/items` lands on instance 1 and the next `GET /cart` lands on instance 2, which returns an empty cart. Evidence: the load balancer's access log shows the failing requests going to a different backend from the requests that wrote. Fix: keep shared state in a shared store (a database or a session store, Chapters 14 and 19) or in the request itself (a signed token, Chapter 8), and keep instances stateless. "Sticky sessions" at the load balancer hide the bug until an instance restarts.

### Backend connection

Your handler is almost never talking directly to the client's code. Between them sit a client library, perhaps a forward proxy, a CDN, a load balancer or gateway, and your server's own HTTP parser. Each makes decisions (cache or not, retry or not, buffer the body or not, reject an oversized header or not) purely from the method, status code and headers you send. Getting those right is how you control what the whole path does.

---

## 7.2 URIs, URLs, and the Request Target

### Definitions

A **URI** (Uniform Resource Identifier, RFC 3986) is a string that identifies a resource, such as `https://api.example.com/v1/users/42` or `urn:isbn:9780131103627`. A **URL** (Uniform Resource Locator) is a URI that also says how to reach the resource over a network: it has a **scheme** (`http`, `https`) and an **authority** (host and optional port).

**Percent-encoding** writes a byte as `%` followed by two hexadecimal digits (a space, byte `0x20`, becomes `%20`). It lets characters that have a structural meaning in a URI, and non-ASCII characters (encoded first as their UTF-8 bytes, Section 7.7), appear as plain data.

The **request target** is the part of the URL that the client sends in the request so that the server knows which resource is meant: in HTTP/1.1, the second word of the request line; in HTTP/2 and HTTP/3, the `:path` and `:authority` fields (Section 7.10).

### Why they exist

A URI combines *how to reach a server* and *which resource on it* in one string that can be linked, bookmarked, logged and put in headers. Because URI parsers split on delimiter characters (`:`, `/`, `?`, `#`, `&`, `=`), data that contains those characters must be encoded, or it will be misread as structure. And a client does not send the URL as a whole: each part goes to a different place.

### Mechanism 1: the parts of a URL and where each goes

```text
 https://alice:secret@api.example.com:8443/v1/orders/42?status=OPEN&limit=10#items
 └─┬─┘   └────┬─────┘ └──────┬──────┘ └┬─┘└─────┬─────┘ └────────┬────────┘ └─┬─┘
 scheme    userinfo        host      port     path              query      fragment
           └───────────────┬──────────────┘
                       authority
```

| Part | Example | Sent to the server? | Used for |
|---|---|---|---|
| scheme | `https` | implied by the connection (explicit in HTTP/2 and HTTP/3 as `:scheme`) | whether to use TLS (`https`, default port 443) or not (`http`, port 80) |
| userinfo | `alice:secret@` | should not be used | deprecated for HTTP: credentials in URLs leak into logs, history and the `Referer` header |
| host | `api.example.com` | **yes**, in the `Host` header (or `:authority`) | DNS lookup (Chapter 6); TLS server name (Chapter 8); **virtual hosting**: one address serving many domains, told apart by `Host` |
| port | `8443` | as the TCP or UDP destination port; in `Host` if not the default | Chapter 6 |
| path | `/v1/orders/42` | **yes**, on the request line | the hierarchical name of the resource; **case-sensitive** (unless the server normalises it); an empty path is sent as `/` |
| query | `status=OPEN&limit=10` | **yes**, on the request line after `?` | parameters: filters, sorting, pagination |
| fragment | `items` | **never** | used only by the client (a browser scrolls to the element with that id; a single-page app may route on it) |

The fragment is a classic trap. A search for `c#` written into a URL as `/search?q=c#` without encoding sends `GET /search?q=c HTTP/1.1`: the browser treated `#` as the start of a fragment and dropped everything after it. The server only ever sees `q=c`.

### Mechanism 2: percent-encoding

RFC 3986 divides ASCII characters into:

- **unreserved**: `A–Z a–z 0–9 - . _ ~`, which never need encoding;
- **reserved**: the delimiters `: / ? # [ ] @` and `! $ & ' ( ) * + , ; =`, which must be encoded when they are *data* rather than structure.

Any other character, and any reserved character used as data, is written as its UTF-8 bytes, each as `%HH`:

| Character | Bytes | Encoded |
|---|---|---|
| space | `20` | `%20` (or `+` in form encoding, below) |
| `#` | `23` | `%23`, so `c#` becomes `c%23` |
| `&` | `26` | `%26`, so `Ben & Jerry` in a parameter does not split it in two |
| `/` | `2F` | `%2F` |
| `é` (U+00E9) | `C3 A9` | `%C3%A9` |

**The `+` trap.** HTML form encoding (`application/x-www-form-urlencoded`, Section 7.7), which most servers also apply when decoding query strings, writes a space as `+`. So a literal plus sign in a parameter must be sent as `%2B`. Otherwise `?email=alice+test@example.com` is decoded as `alice test@example.com`.

### Mechanism 3: the four forms of the request target

| Form | Request line | When |
|---|---|---|
| origin form | `GET /v1/orders/42?status=OPEN HTTP/1.1` plus `Host: api.example.com` | almost every request to an origin server or reverse proxy: just the path and query |
| absolute form | `GET http://api.example.com/v1/orders/42 HTTP/1.1` | requests sent through a forward proxy, which needs the whole URL |
| authority form | `CONNECT api.example.com:443 HTTP/1.1` | only with `CONNECT`, asking a proxy to open a tunnel to `host:port` |
| asterisk form | `OPTIONS * HTTP/1.1` | only with `OPTIONS`, asking about the server as a whole |

### Concrete: path or query?

```text
 /v1/users/42/orders?status=OPEN&sort=-created_at&limit=20
 └────────┬────────┘ └──────────────────┬────────────────┘
   which resource                  how to filter, sort, page or shape it
```

A value belongs in the **path** if removing it changes *which* resource is addressed (user 42's orders, not all orders). It belongs in the **query** if it refines a request for the same resource: filters, sort order, pagination, fields to include. Chapter 13 develops this into API design.

### Guarantees and non-guarantees

The standards guarantee that a compliant parser splits a URI into the same parts every time, and that the origin form plus the `Host` header reconstruct the full target.

They do not guarantee unlimited length: the RFCs set no maximum, but browsers, proxies and servers enforce their own limits (commonly a few kilobytes up to a few tens of kilobytes), and a server rejects a longer target with **`414 URI Too Long`**. They do not keep URLs private: over HTTPS the path and query are encrypted on the wire (Chapter 8), but full URLs are routinely written to access logs, proxy logs, browser history and analytics. Never put passwords, tokens or personal data in a query string.

### Failure and diagnosis

- **Double encoding.** A service builds a URL by concatenating an already-encoded value and passes it to a client library that encodes it again. The space became `%20` the first time; the `%` becomes `%25` the second time, so the wire shows `%2520` and the server decodes it to the literal text `%20`. Evidence: `%25` sequences in the access log. Fix: build URLs with a URI builder that encodes each component exactly once, never by string concatenation.
- **Normalisation mismatch between proxy and application.** A proxy allows `/public/*` and blocks `/admin/*`. An attacker requests `/public/..%2Fadmin/users`. The proxy checks the raw string, sees a `/public/` prefix and forwards it; the application decodes `%2F` to `/`, resolves `..`, and serves `/admin/users`. Evidence: requests with encoded slashes or dot segments in the logs. Fix: normalise (or reject) encoded slashes and dot segments at the edge, and enforce authorisation in the application, not only by proxy path rules.

> **Framework lens:** Server frameworks (Spring MVC's `@PathVariable` and `@RequestParam`, Express's `req.params` and `req.query`, Go's `net/http`) decode path variables and query parameters once before your handler runs. For outbound calls, Spring's `RestClient` or `UriComponentsBuilder` and Java's `URI` constructors encode template variables correctly; concatenating strings bypasses that and lets reserved characters in data inject parameters or cut the URL short at a `#`.

### Backend connection

Every URL your service builds is input to someone else's parser, and every URL your service receives is someone else's encoding. Build them with structured builders, log them knowing they may hold sensitive data, and never let a security decision depend on a string comparison made before the application's own decoding.

---

## 7.3 The HTTP/1.1 Wire Format and Framing

### Definitions

An **HTTP/1.1 message** (RFC 9112) is a **start line** (a *request line* for requests, a *status line* for responses), zero or more **header fields** (each a `name: value` line), an **empty line**, and an optional **body**. Every line ends with **CRLF**: the two bytes `0x0D 0x0A`, written `\r\n`. The header section therefore ends with the four bytes `\r\n\r\n`.

**Framing** is the set of rules a receiver uses to find where a message's body ends, and so where the next message on the same connection begins. In HTTP/1.1 it is decided by `Content-Length`, by `Transfer-Encoding: chunked`, or, for some responses, by the connection closing.

**Request smuggling** is an attack that exploits two HTTP processors in a row (for example a front-end proxy and a back-end server sharing a persistent connection) that apply *different* framing rules to the same bytes. They disagree about where a request ends, and the attacker's leftover bytes are taken as the start of the next request, which may be another user's.

### Why it exists

TCP provides no boundaries (Chapter 6, Section 6.9). The earliest HTTP solved framing by closing the connection after each response: the end of the stream marked the end of the body. That makes every request pay a new TCP handshake (and TLS handshake, and slow start, Chapter 6, Section 6.12). Once HTTP/1.1 made **persistent connections** the default, with several requests and responses following each other on one connection, both sides needed an exact, byte-level way to find the end of each message.

### Mechanism 1: the message layout

```text
 ┌──────────────────────────────────────────────────────────────────┐
 │ start line                                                       │
 │   request:  METHOD SP request-target SP HTTP/1.1 CRLF            │
 │   response: HTTP/1.1 SP status-code SP reason-phrase CRLF        │
 ├──────────────────────────────────────────────────────────────────┤
 │ header fields, zero or more                                      │
 │   field-name ":" optional-space field-value optional-space CRLF  │
 ├──────────────────────────────────────────────────────────────────┤
 │ empty line: CRLF        (with the previous CRLF: "\r\n\r\n")     │
 ├──────────────────────────────────────────────────────────────────┤
 │ body (optional): bytes, length decided by the framing rules      │
 └──────────────────────────────────────────────────────────────────┘
```

How a parser reads a request from the socket's receive buffer:

1. **Lines end in CRLF.** RFC 9112 allows a lenient parser to accept a bare LF, but hardened servers and proxies reject bare CR or LF in the header section with `400 Bad Request`, because tolerating them differently in two processors is one way to cause smuggling.
2. **The header section ends at the first `\r\n\r\n`.** The parser scans for those four bytes. Everything before them is text metadata; the next byte is the first byte of the body, which may be any bytes at all, including `\r\n\r\n`.
3. **Field names are case-insensitive.** `Content-Type`, `content-type` and `CONTENT-TYPE` are the same field. (HTTP/2 and HTTP/3 require names in lower case on the wire.)
4. **The reason phrase is for humans.** In `HTTP/1.1 200 OK`, a client acts only on `200`. HTTP/2 and HTTP/3 do not send a reason phrase at all.

### Concrete 1: a request and response, line by line

```text
 ── client → server ─────────────────────────────────────────────────────
 POST /v1/orders HTTP/1.1\r\n            request line: method, target, version
 Host: api.example.com\r\n               required in HTTP/1.1 (virtual hosting)
 User-Agent: order-cli/2.1\r\n           which client software
 Content-Type: application/json\r\n      how to interpret the body
 Accept: application/json\r\n            what the client wants back
 Content-Length: 23\r\n                  FRAMING: the body is exactly 23 bytes
 \r\n                                    end of the header section
 {"sku":"KB-42","qty":1}                 23 bytes; no CRLF after the body

 ── server → client ─────────────────────────────────────────────────────
 HTTP/1.1 201 Created\r\n                status line
 Date: Sat, 17 Oct 2026 14:10:00 GMT\r\n
 Content-Type: application/json\r\n
 Location: /v1/orders/9001\r\n           where the new resource lives
 Content-Length: 35\r\n                  FRAMING: 35 bytes
 \r\n
 {"orderId":9001,"status":"CREATED"}     35 bytes
```

After the `\r\n\r\n`, the server's parser reads exactly 23 bytes and hands them to the JSON decoder. If a 24th byte is already in the receive buffer, it is the first byte of the **next** request on this connection, and the parser leaves it there.

**Which rule frames a given message.** The receiver decides in this order (RFC 9112, Section 6.3):

1. Responses to `HEAD`, and responses with status `1xx`, `204` or `304`, never have a body, whatever their headers say.
2. A successful response to `CONNECT` turns the connection into a tunnel.
3. If `Transfer-Encoding` is present, it wins; for a request, its last coding must be `chunked`, or the request is rejected.
4. Otherwise a valid `Content-Length` gives the body length.
5. Otherwise a request has no body, and a response's body runs until the server closes the connection. Such a "close-delimited" response cannot be distinguished from one that was cut off by a failure, and the connection cannot be reused.

### Mechanism 2: `Content-Length` versus chunked

A server that must stream a 500 MB CSV export from a database cursor cannot know its length in advance. To send `Content-Length` first, it would have to build the whole export in memory: 100 concurrent exports would need 50 GB, and the first byte would leave only when the last row was ready.

**Chunked transfer coding** (`Transfer-Encoding: chunked`) solves this. The sender omits `Content-Length` and sends the body as a series of **chunks**. Each chunk is its size **in hexadecimal**, CRLF, that many bytes, CRLF. A chunk of size 0 marks the end, optionally followed by trailer fields, and then a final CRLF. A plain chunked body therefore ends with the five bytes `0\r\n\r\n`.

| | `Content-Length: N` | `Transfer-Encoding: chunked` |
|---|---|---|
| Use when | the size is known before the headers are sent | the size is not known: streamed exports, on-the-fly compression, long-running streams |
| Sender memory | must know or hold the whole body first | constant: send each buffer as it fills |
| Time to first byte | after the whole body is ready | as soon as the first chunk is |
| End of body found by | counting N bytes after the header section | reading chunks until the size-0 chunk |
| In HTTP/2 and HTTP/3 | optional, and checked against the actual data | **forbidden**: both protocols frame data themselves (Section 7.10) |

### Concrete 2: a chunked response on the wire

The 35-byte JSON body from above, sent as a 16-byte chunk (hex `10`) and a 19-byte chunk (hex `13`):

```text
 HTTP/1.1 200 OK\r\n
 Content-Type: application/json\r\n
 Transfer-Encoding: chunked\r\n         no Content-Length
 \r\n
 10\r\n                                 chunk size 0x10 = 16
 {"orderId":9001,\r\n                   16 bytes of data, then CRLF
 13\r\n                                 chunk size 0x13 = 19
 "status":"CREATED"}\r\n                19 bytes of data, then CRLF
 0\r\n                                  last chunk
 \r\n                                   end of message (no trailers)
```

The receiver strips the chunk framing and passes 35 contiguous bytes to the application. Chunking is a property of this hop's transfer, not of the content: a proxy may receive a chunked response and forward it with a `Content-Length`, or the reverse.

### Mechanism 3: request smuggling

A reverse proxy or CDN receives requests from many users and forwards them to the origin over a small pool of **shared persistent connections**:

```text
 attacker ──┐                                   ┌── one persistent connection,
            ├──► front end (proxy / CDN) ═══════╡   used for many users' requests
 victim   ──┘                                   └──► back end (origin server)
```

Suppose the front end frames requests by `Content-Length` and ignores `Transfer-Encoding` (for example because the attacker wrote it in a slightly malformed way that only the back end accepts), while the back end honours `Transfer-Encoding: chunked`. This is a **CL.TE** disagreement. The attacker sends:

```text
 POST /search HTTP/1.1\r\n
 Host: shop.example.com\r\n
 Content-Length: 37\r\n
 Transfer-Encoding: chunked\r\n
 \r\n
 0\r\n                                   ┐
 \r\n                                    │ 37 bytes of body, by Content-Length:
 GET /admin HTTP/1.1\r\n                 │ 3 + 2 + 21 + 11
 X-Ignore: X                             ┘ (no CRLF at the end)
```

1. **The front end** reads `Content-Length: 37`, takes the 37 bytes after the header section as the body of one harmless `POST /search`, and forwards all of it on the shared connection.
2. **The back end** sees `Transfer-Encoding: chunked`, reads the chunk `0\r\n\r\n` (the end of the body after 5 bytes), and finishes `POST /search` there. The remaining 32 bytes, `GET /admin HTTP/1.1\r\nX-Ignore: X`, stay in its buffer as the **start of the next request**.
3. **The victim's** request arrives at the front end and is forwarded on the same connection, right after those bytes. The back end now reads:

```text
 GET /admin HTTP/1.1\r\n
 X-Ignore: XGET /profile HTTP/1.1\r\n      the victim's request line became a header value
 Host: shop.example.com\r\n
 Cookie: session=VICTIM\r\n                the victim's credentials
 \r\n
```

The back end executes `GET /admin` with the victim's cookie, and the response goes to the victim. Variants reverse the roles (**TE.CL**) or use two conflicting `Content-Length` fields. The common cause is always two parsers disagreeing about framing.

The defences follow from the mechanism:

- **What the RFC requires.** When both fields are present, `Transfer-Encoding` overrides `Content-Length`; such a message "ought to be handled as an error", a server that processes it must close the connection after responding, and an intermediary that forwards it must remove `Content-Length` first. Several different `Content-Length` values are an unrecoverable error.
- **What to deploy.** Front ends that reject any ambiguous request (both fields, malformed `Transfer-Encoding`, duplicate lengths, bare CR or LF) with `400` and close the connection; front ends and back ends from versions that implement these rules strictly; and HTTP/2 between the proxy and the origin where possible, because HTTP/2 frames carry explicit binary lengths (a proxy that converts HTTP/2 back into HTTP/1.1 must still validate lengths).

### Guarantees and non-guarantees

HTTP/1.1 framing guarantees that, when both ends follow RFC 9112 strictly, many requests and responses of any length can share one connection without corrupting each other.

It does not guarantee that a declared length matches what was sent. A sender that writes fewer bytes than its `Content-Length` leaves the receiver waiting for bytes that never come, until a timeout. One that writes more leaves extra bytes that the receiver parses as the next message, usually producing `400 Bad Request` on an innocent request.

### Failure and diagnosis

**`Content-Length` counted in characters instead of bytes.** A webhook client sets `Content-Length` to the length of the JSON *string*, then sends a body containing `"name": "René"` or `"price": "₹500"`. In UTF-8, `é` is 2 bytes and `₹` is 3 bytes (Section 7.7), so a 100-character string may be 104 bytes. The server reads 100 bytes, fails to parse truncated JSON, and treats the 4 leftover bytes as the start of the next request. Evidence: failures only for non-ASCII content; `400` responses on the *next* request on that connection. Fix: encode to bytes first and use the byte array's length, or let the HTTP library set the header.

### Backend connection

A `400 Bad Request` in the proxy's access log that never appears in the application's logs, or a client that receives 99% of a response and then waits until a timeout, both point to framing: a bare-LF line ending, a length counted in characters, or a proxy that changed the transfer coding. Look at the raw bytes (Section 7.16) before reading application code.

---

## 7.4 Methods: Safety, Idempotency, Cacheability

### Definitions

An **HTTP method** is the first word of the request line (`GET`, `POST`, ...). It states the **purpose** of the request to the origin server and to every intermediary.

A method is **safe** if its defined meaning is read-only: the client neither requests nor expects any change of state on the server. (A server may still log the request or update a hit counter; what matters is that the client did not ask for a change and cannot be held responsible for one.)

A method is **idempotent** if sending the same request several times has the same **intended effect on the server's state** as sending it once. The *response* may differ: the first `DELETE` may return `204` and the second `404`; the state (the resource is gone) is the same.

A response is **cacheable** if a cache may store it and reuse it for later requests, subject to the caching rules of Section 7.12.

### Why it exists

Intermediaries and clients make automatic decisions on the method alone. A browser prefetches links with `GET`; a crawler follows every `GET` link it finds; a client library retries a `GET` or `PUT` after a broken connection but should not retry a `POST`; a cache stores `GET` responses and never stores `DELETE` responses. These are only correct if servers implement each method with the meaning the protocol declares. That is why **`GET` must never change state**: if `GET /orders/42/cancel` cancels an order, a link prefetcher or crawler cancels orders.

Idempotency matters most because of the unknown outcome of Chapter 6 (Section 6.9): when a response is lost, an idempotent request can be repeated without changing the final state.

### Mechanism: the method table

| Method | Meaning | Safe | Idempotent | Response cacheable | Request body |
|---|---|---|---|---|---|
| `GET` | send me a representation of the resource | yes | yes | yes | no defined meaning; do not use |
| `HEAD` | like `GET`, but headers only | yes | yes | yes | no defined meaning |
| `OPTIONS` | which communication options apply to this resource (also used by browsers for CORS checks, Chapter 9) | yes | yes | no | allowed, rarely used |
| `TRACE` | echo the request back (usually disabled) | yes | yes | no | no |
| `PUT` | **replace** the resource's state with the representation enclosed, creating it if absent | no | **yes** | no | yes |
| `DELETE` | remove the resource | no | **yes** | no | no defined meaning |
| `POST` | process the enclosed data according to the resource's own semantics (typically: create a subordinate resource, start an action) | no | **no** | only with explicit freshness information; rarely implemented | yes |
| `PATCH` (RFC 5789) | apply a set of **changes** described in the body | no | **not in general** | no | yes |
| `CONNECT` | open a tunnel to the named authority | no | no | no | no |

### `PUT`, `PATCH` and `POST`

- **`PUT /v1/users/42`** with a full representation means "make user 42 look exactly like this". Sending it twice leaves the same state, so it is idempotent, and a client that knows the resource's URL can use it to create *or* replace. Fields omitted from the body are, by the meaning of `PUT`, removed or reset; using `PUT` for partial updates is a common source of accidental data loss.
- **`PATCH /v1/users/42`** sends instructions. Whether a patch is idempotent depends on the instructions. A JSON Merge Patch (`application/merge-patch+json`, RFC 7396) such as `{"email": "new@example.com"}` sets a field and is idempotent in practice. A JSON Patch (`application/json-patch+json`, RFC 6902) operation such as "append this item to the list" or a custom `{"op": "increment", "path": "/balance", "value": 10}` is not: applying it twice adds twice.
- **`POST /v1/orders`** asks the collection to process the body, typically creating a new order with a new identifier chosen by the server. Two identical `POST`s create two orders. This is why `POST` is neither safe nor idempotent, and why a timed-out `POST` cannot simply be retried (Section 7.15; idempotency keys, Chapter 13).

### Guarantees and non-guarantees

Method semantics are a **contract that the server must implement**; HTTP itself enforces nothing. Idempotency is about server *state*, not responses, and not side effects such as log lines or metrics. A `PUT` is idempotent only if the server's handler really replaces state; a handler that appends to an audit table on every `PUT` is still idempotent for the resource, but a handler that increments a version counter visible to clients changes what a later `GET` returns. Safe methods can still be expensive: an unbounded `GET /reports?range=all` can take down a database.

### Failure and diagnosis

- **State-changing `GET`.** Symptom: orders cancelled or emails sent with no user action; requests from crawlers, link-preview bots or browser prefetchers in the access log just before each event. Fix: make the action a `POST` (or `DELETE`), and protect it as a state-changing request (Chapter 19).
- **Retrying non-idempotent requests.** Symptom: duplicate orders or charges, created milliseconds or seconds apart, after a network blip or a deployment. Evidence: client logs show a retry after a timeout or reset; the server logs two `POST`s with identical bodies. Fix: retry only idempotent methods automatically, and make `POST` safe to retry with an idempotency key (Chapter 13).
- **`PUT` used as a partial update.** Symptom: fields mysteriously reset to null after an edit from an older client that does not know about a new field. Fix: use `PATCH` for partial updates, or make clients send the full representation they read.

### Backend connection

Choose methods by their contract, because the whole path relies on it: `GET` for reads (and make them cacheable where possible), `PUT` and `DELETE` when the client names the resource and the operation is a replacement or removal, `POST` for creation and actions, and `PATCH` for partial changes. Then make sure each handler actually has the property its method promises.

---

## 7.5 Status Codes

### Definition

A **status code** is the three-digit number in a response's status line that classifies the outcome of the request. The first digit gives the **class**:

| Class | Meaning |
|---|---|
| `1xx` informational | an interim response; the final response follows |
| `2xx` successful | the request was received, understood and accepted |
| `3xx` redirection | the client must take further action, usually another request |
| `4xx` client error | the request is at fault (malformed, unauthorised, conflicting...) |
| `5xx` server error | the server, or one behind it, failed to fulfil a valid request |

A client that does not recognise a code treats it as the `x00` code of its class.

### Why it exists

Clients and intermediaries need a machine-readable outcome to decide what to do next: render the result, follow a redirect, ask for credentials, back off and retry, fail over to another server, alert someone. The body explains details to humans and to application code; the status code drives generic behaviour.

### Mechanism: the codes to know, and what each does not tell you

**Success**

| Code | Meaning | What it does **not** tell you |
|---|---|---|
| `200 OK` | succeeded; the body is the result | that anything was stored durably, unless the API promises so; that the body is the shape you expected |
| `201 Created` | a new resource was created; `Location` gives its URL | that the resource is visible to every reader yet (replicas may lag, Chapter 22) |
| `202 Accepted` | accepted for processing, not yet done | that processing will succeed, or when it will finish (Chapter 13 designs status polling) |
| `204 No Content` | succeeded; no body | anything beyond success |
| `206 Partial Content` | here is the requested range (Section 7.8) | that the rest of the resource has not changed meanwhile, unless you used `If-Range` |

**Redirection** (details in Section 7.11)

| Code | Meaning | What it does not tell you |
|---|---|---|
| `301`, `308` | moved permanently | `301` does not promise the method is kept for `POST`; `308` does |
| `302`, `307` | temporarily elsewhere | `302` does not promise the method is kept; `307` does |
| `303 See Other` | look at this other URL with `GET` (typically after a `POST`) | anything about the original request beyond its success |
| `304 Not Modified` | your cached copy is still valid (Section 7.12) | the content; there is no body |

**Client errors**

| Code | Meaning | What it does not tell you |
|---|---|---|
| `400 Bad Request` | the request is malformed (bad syntax, bad framing, unparseable body) | which part, unless the body says (use RFC 9457 problem details, below) |
| `401 Unauthorized` | the request lacks valid **authentication**; the response must include a `WWW-Authenticate` header naming how to authenticate | whether the resource exists, or whether valid credentials would be enough (despite its name, it is about *authentication*) |
| `403 Forbidden` | the server understood who you are (or does not need to) and **refuses**: you are not allowed | whether the resource exists; whether other credentials would work |
| `404 Not Found` | no current representation at this URL, **or the server will not admit one exists** | that it never existed or never will |
| `405 Method Not Allowed` | the resource exists but does not support this method; `Allow` lists those it does | |
| `408 Request Timeout` | the server gave up waiting for the client to finish sending | anything about processing: nothing was processed |
| `409 Conflict` | the request conflicts with the resource's current state (a duplicate unique key, an invalid state transition) | how to resolve the conflict, unless the body says |
| `412 Precondition Failed` | a condition the client set (such as `If-Match`, Section 7.13) is false | what the current state is; fetch it again |
| `413 Content Too Large` | the body exceeds the server's limit | the limit, unless stated |
| `415 Unsupported Media Type` | the server does not accept this `Content-Type` (or `Content-Encoding`) | |
| `422 Unprocessable Content` | the syntax is fine but the content is semantically invalid (a negative quantity, an end date before the start date) | |
| `428 Precondition Required` | the server requires a conditional request here (Section 7.13) | |
| `429 Too Many Requests` | rate limited; `Retry-After` may say when to try again | that the request was not processed in part; whose quota was exceeded |
| `431 Request Header Fields Too Large` | headers too big (often oversized cookies or tokens) | |

**Server errors**

| Code | Meaning | What it does not tell you |
|---|---|---|
| `500 Internal Server Error` | the server hit an unexpected condition (an unhandled exception) | **whether the operation's effects happened**: the failure may have come after a commit |
| `502 Bad Gateway` | a gateway or proxy got an invalid response from the server behind it, or could not get one at all (refused, reset) | whether the upstream processed the request |
| `503 Service Unavailable` | temporarily unable to handle the request (overloaded, in maintenance, no healthy backends); `Retry-After` may say when to retry | how long it will last, unless stated |
| `504 Gateway Timeout` | a gateway or proxy did not get a response from upstream in time | **whether the upstream is still working on it, or already finished**: the outcome is unknown |

### The distinctions interviews ask about

- **`401` versus `403`.** `401` means "I do not know who you are (or your credentials are invalid or expired): authenticate". `403` means "I know who you are, and you may not do this". A client should prompt for login or refresh a token on `401`, and should not on `403`.
- **`403` versus `404`.** Returning `403` for another tenant's resource confirms that it exists, which leaks information (an attacker can enumerate order numbers). Many APIs deliberately return `404` for resources the caller may not see, so that "forbidden" and "absent" look the same (Chapter 20).
- **`400` versus `422`.** `400` for requests the server cannot parse or frame; `422` for requests it can parse whose content breaks the rules. Some APIs use `400` for both; be consistent and explain the error in the body.
- **`500`, `502`, `503`, `504`.** `500` is *this* server failing. `502` and `504` are reported by a *gateway* about the server behind it: bad or missing response, or no response in time. `503` is a deliberate "not now", often with `Retry-After`. Chapter 10 traces where in a proxy each one is generated.

**Status codes and the outcome.** A `5xx`, a `504` especially, does not prove the operation failed. A gateway that times out after 30 seconds returns `504` while the application may still be running the request, and may commit it a second later. A `200` from a gateway proves only that the gateway received a `200`. Treat any failure that is not a clear `4xx` rejection as an unknown outcome for non-idempotent operations (Section 7.15).

### Concrete: a structured error body

Status codes are coarse. **RFC 9457** (Problem Details for HTTP APIs) defines a standard JSON body, with media type `application/problem+json`, that adds details a program can use:

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/insufficient-stock",
  "title": "Insufficient stock",
  "status": 422,
  "detail": "Only 2 units of KB-42 are available; 5 were requested.",
  "instance": "/v1/orders/requests/7f3a",
  "available": 2
}
```

`type` is a URI identifying the kind of problem (it may point to documentation), `title` a short summary, `status` a copy of the status code, `detail` an explanation of this occurrence, `instance` a reference to it; extra members such as `available` are allowed. Chapter 13 makes this an API design standard.

> **Framework lens:** Spring Framework 6 provides a `ProblemDetail` type and can render exceptions as `application/problem+json`. Frameworks map an unhandled exception to `500` by default, which is why a handler that commits and then throws while building its response produces a `500` for an operation that succeeded.

### Guarantees and non-guarantees

The status code guarantees a classification that every client and intermediary understands. It does not describe the cause, it does not prove that effects did or did not happen (for `5xx` and gateway codes especially), and servers do not always choose correctly.

### Failure and diagnosis

- **`200` with an error inside.** An API returns `200 OK` with `{"error": "payment declined"}`. Caches may store it, monitoring counts it as success, and client libraries do not raise errors. Evidence: error rates that look perfect while users complain. Fix: use the right status code and put the details in the body.
- **`502` and `504` floods during deployments.** The load balancer forwards to an instance that is shutting down (connections refused or reset, giving `502`) or still starting (timeouts, giving `504`). Evidence: errors line up with deploy times and with specific backends in the load balancer's log. Fix: readiness checks and graceful shutdown with a drain period (Chapter 4, Section 4.11; Chapter 24).
- **`401` loops.** A client refreshes its token on every `403` as if it were a `401`, or the server returns `401` without `WWW-Authenticate`, and the client cannot tell what to do. Fix: use the codes as defined.

### Backend connection

Pick status codes for the *generic* behaviour they trigger (retry or not, cache or not, prompt for login or not) and put everything specific in a problem-details body. Treat `5xx` and timeouts from a dependency as "outcome unknown" whenever the request was not idempotent.

---

## 7.6 Header Fields

### Definition

A **header field** is a `name: value` line in the header section of a message that carries metadata about the message, the sender, the requested resource or the enclosed representation. Names are case-insensitive. A field that appears several times can usually be combined into one, with the values separated by commas (`Accept: a` plus `Accept: b` equals `Accept: a, b`); `Set-Cookie` is the exception and must stay as separate lines (Section 7.14).

### Why it exists

Headers keep the protocol extensible. New behaviour (a cache directive, a tracing identifier, a security policy) is added as a new field, and implementations that do not understand a field ignore it. They also keep metadata separate from content, so intermediaries can act on a message without reading its body.

### Mechanism: the families of headers

| Family | Purpose | Examples |
|---|---|---|
| request context | who is asking, for what, under which conditions | `Host`, `User-Agent`, `Referer`, `Authorization`, `Cookie`, `If-None-Match`, `If-Match` |
| content negotiation | what the client can accept | `Accept`, `Accept-Encoding`, `Accept-Language` |
| response control | what the client should do next | `Location`, `Retry-After`, `WWW-Authenticate`, `Allow`, `Set-Cookie` |
| representation metadata | how to interpret the body | `Content-Type`, `Content-Encoding`, `Content-Language`, `Content-Length`, `Content-Range`, `ETag`, `Last-Modified` |
| caching | who may store the response, for how long | `Cache-Control`, `Expires`, `Age`, `Vary` (Section 7.12) |
| connection (hop-by-hop) | about *this* connection only; a proxy must not forward them | `Connection`, `Keep-Alive`, `Transfer-Encoding`, `TE`, `Upgrade` |

The distinction between **end-to-end** and **hop-by-hop** fields matters for proxies. End-to-end fields describe the message and travel all the way. Hop-by-hop fields describe one connection: a proxy consumes them and generates its own on the next hop. The `Connection` header lists any further fields that are hop-by-hop for this message.

The headers you meet in every service:

- **`Host`** names the target host and port. It is **required** in HTTP/1.1 (a request without it gets `400`), and is what lets one IP address serve many sites (virtual hosting). Reverse proxies route on it.
- **`User-Agent`** identifies the client software. Useful in logs; never a security control, since clients can send anything.
- **`Accept-*`** state the client's preferences (Section 7.8).
- **`Content-Type`** states the media type of the body (Section 7.7). A server that receives a body should check it and answer `415` if it is not supported.
- **`Authorization`** carries credentials. `Basic` sends `base64(user:password)`, which is an encoding, not encryption (Chapter 8), so it is only acceptable over HTTPS. `Bearer` sends a token (Chapter 20).
- **Tracing and forwarding headers.** `traceparent` (W3C Trace Context) carries a trace identifier from service to service so that one request can be followed through many (Chapter 26). `X-Forwarded-For`, `X-Forwarded-Proto` and the standard `Forwarded` field carry the original client's address and scheme past proxies (Chapter 10).

**Custom headers.** Applications may define their own fields. The old convention of an `X-` prefix is deprecated (RFC 6648) because successful experimental fields then needed renaming; choose a clear name instead (`Idempotency-Key`, Chapter 13).

### Concrete: a request's headers, read as decisions

```http
GET /v1/products/42 HTTP/1.1
Host: api.example.com                       which virtual host (and backend pool)
User-Agent: shop-ios/5.3 (iPhone; iOS 18)   logs and analytics only
Accept: application/json                    the representation wanted
Accept-Encoding: br, gzip                   compressions the client can decode
Accept-Language: hi-IN, en;q=0.8            language preferences, with weights
Authorization: Bearer eyJhbGciOi...         who is calling (Chapter 20)
If-None-Match: "v14"                        "only send the body if it changed" (Section 7.12)
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

### Guarantees and non-guarantees

Headers guarantee an extensible channel for metadata that every hop can read. They are not authenticated: any client can send any header, including `X-Forwarded-For: 127.0.0.1` or a forged `X-User-Id`. Only headers set by infrastructure you control, after it has removed any client-supplied copies, can be trusted. Headers are also not unlimited: servers cap the size of the header section (commonly in the range of 8 KB to tens of kilobytes; check your server and proxy), and exceed it to get `431` or `400`.

### Failure and diagnosis

- **Header bloat.** Requests start failing with `431 Request Header Fields Too Large`, or `400` from a proxy, for some users only. The cause is usually cookies that grew (many cookies set over time, or large ones) or a large token in `Authorization`. Evidence: the failing requests' header size in a capture, compared with the server's and proxy's limits. Fix: keep cookies and tokens small; store bulky data on the server and send a reference.
- **Trusting a client-supplied header.** An application reads the caller's address from `X-Forwarded-For` to decide access, and an attacker sends the header themselves. Fix: make the edge proxy overwrite or strip such headers, and trust only the values it sets (Chapter 10).

### Backend connection

Treat request headers as untrusted input, like the body. Put routing and caching decisions in standard headers so that the whole path can act on them, propagate tracing headers on every outbound call, and keep the header section small.

---

## 7.7 Bodies and Representations

### Definitions

A **representation** is the data that stands for a resource's state at a point in time, in a particular format, together with metadata that describes it. HTTP transfers representations, not objects: a body is a sequence of bytes, and the representation headers say how to interpret them.

A **media type** (MIME type) names the format: a `type/subtype` pair with optional parameters, such as `application/json`, `text/html; charset=utf-8`, `image/png`, `multipart/form-data; boundary=x7f`.

A **character encoding** (charset) is the rule that maps text characters to bytes. **UTF-8** encodes every Unicode character in 1 to 4 bytes, and every ASCII character as the same single byte as in ASCII.

### Why it exists

"HTTP does not send objects." An order in your program is a structure in memory; to cross the network it must be **serialised** into bytes (Chapter 1, Section 1.4), and the receiver must know which format and which character encoding were used to turn the bytes back into data. `Content-Type` carries that knowledge. Without it, the receiver guesses, and guesses go wrong.

### Mechanism 1: the common body formats

**JSON** (`application/json`) is the default for APIs. RFC 8259 requires JSON exchanged between systems to be UTF-8, so a `charset` parameter adds nothing.

```http
Content-Type: application/json

{"sku":"KB-42","qty":1,"note":"gift wrap!"}
```

**Form encoding** (`application/x-www-form-urlencoded`) is what an HTML form sends by default: `name=value` pairs joined by `&`, percent-encoded, with spaces as `+` (Section 7.2):

```http
Content-Type: application/x-www-form-urlencoded

sku=KB-42&qty=1&note=gift+wrap%21
```

**Multipart** (`multipart/form-data`) sends several parts, such as form fields plus files, in one body. Each part has its own small header section; a **boundary** string, chosen so that it does not occur in the data and declared in `Content-Type`, separates the parts:

```http
Content-Type: multipart/form-data; boundary=x7f3a

--x7f3a
Content-Disposition: form-data; name="orderId"

9001
--x7f3a
Content-Disposition: form-data; name="receipt"; filename="receipt.pdf"
Content-Type: application/pdf

%PDF-1.7 ...binary bytes of the file...
--x7f3a--
```

The final boundary has two trailing hyphens. Binary data travels as raw bytes; there is no need to Base64-encode files inside multipart.

Other common types: `application/octet-stream` for arbitrary binary data, `text/plain`, `text/html`, `application/xml`, and `application/x-protobuf` or `application/grpc` for Protocol Buffers.

### Mechanism 2: characters, bytes and mojibake

```text
 character   code point   UTF-8 bytes     bytes misread as ISO-8859-1 (Latin-1)
 A           U+0041       41              A
 é           U+00E9       C3 A9           Ã©
 ₹           U+20B9       E2 82 B9        â‚¹ (Windows-1252) or unprintable (Latin-1)
 😀          U+1F600      F0 9F 98 80     ðŸ˜€ (Windows-1252)
```

**Mojibake** is garbled text produced by decoding bytes with the wrong character encoding. A server that writes `René` as UTF-8 (`52 65 6E C3 A9`) to a client that decodes Latin-1 displays `RenÃ©`. The bytes were transferred perfectly; the interpretation was wrong. Two lessons:

- Declare the charset for text types (`text/html; charset=utf-8`), and use UTF-8 everywhere: in HTTP bodies, in databases, in files.
- Lengths in HTTP are in **bytes**. `"René"` is 4 characters and 5 bytes (Section 7.3's `Content-Length` failure).

### Guarantees and non-guarantees

`Content-Type` guarantees a declared interpretation. It does not guarantee that the body actually is what it claims: a file uploaded as `image/png` may be a script. Browsers may also "sniff" content and second-guess the declared type unless told not to (Chapter 9). Parsing untrusted bodies is a security boundary: validate size, type and structure before use.

### Failure and diagnosis

- **Mojibake.** Accented names turn into `Ã©` sequences, in the UI or, worse, in the database. Evidence: inspect the raw bytes (`xxd`, Section 7.16) to see whether they are UTF-8 (`C3 A9`) or already double-encoded (`C3 83 C2 A9`). Fix: UTF-8 at every hop, and explicit charsets where the media type needs them.
- **Large numbers in JSON.** A 64-bit order ID such as `9007199254740993` arrives in a JavaScript client as `9007199254740992`, because JavaScript numbers are 64-bit floating point and represent integers exactly only up to 2^53. Fix: send large identifiers as strings.
- **Wrong or missing `Content-Type`.** A client posts JSON with `Content-Type: text/plain`, and the framework refuses to bind it (`415`) or passes an empty object. Evidence: the request's headers. Fix: send the correct type; make servers reject what they cannot parse rather than guess.

### Backend connection

Be strict about what you accept and explicit about what you send: check `Content-Type` and size limits before parsing, send a correct `Content-Type` with a charset when it applies, and remember that every body is bytes that someone else will interpret.

---

## 7.8 Content Negotiation, Compression, and Range Requests

### Definitions

**Content negotiation** is the selection of one representation of a resource, among several available, using the client's stated preferences in `Accept` (media type), `Accept-Language` and `Accept-Encoding`. Preferences may carry weights, **q-values** from 0 to 1 (`en;q=0.8`), where higher is preferred.

A **content coding** is a transformation applied to a representation, usually compression: `gzip`, `br` (Brotli), `zstd` (Zstandard). The server states the one it applied in `Content-Encoding`. It is part of the representation and is end to end, unlike `Transfer-Encoding`, which applies to one hop.

A **range request** asks for only part of a representation, with `Range: bytes=start-end`; the server answers `206 Partial Content` with that part.

### Why they exist

One resource can be served in several forms (JSON or XML, English or Hindi, compressed or not), and the client knows which it can use. Compression exists because text formats compress well: JSON and HTML often shrink to a fraction of their size, which matters on slow mobile links and on metered bandwidth. Range requests exist so that large transfers can resume after a failure, video players can seek, and downloads can be split across connections.

### Mechanism 1: negotiation and compression

```text
 request                                   response
 Accept: application/json                  Content-Type: application/json
 Accept-Language: hi-IN, en;q=0.8          Content-Language: hi-IN
 Accept-Encoding: br, gzip                 Content-Encoding: br
                                           Vary: Accept-Language, Accept-Encoding
```

The server picks the best match it can produce. It should list in **`Vary`** every request header that influenced its choice, so that caches do not serve a Hindi, Brotli-compressed response to a client that asked for English or cannot decode Brotli (Section 7.12). If nothing acceptable exists, the server may answer `406 Not Acceptable`, but most APIs simply return their default.

Compression is a trade of CPU for bytes. Compressing a 200 KB JSON response to perhaps 30 KB saves time on a slow link and costs a little CPU on both ends. It is wasted on data that is already compressed (JPEG, PNG, MP4, ZIP), and for very small responses the overhead can outweigh the saving. Static assets are commonly compressed once at build time; dynamic responses are compressed on the fly, often by the reverse proxy. (Compressing responses that contain both secrets and attacker-influenced text can leak the secrets through response sizes, even over TLS; Chapter 8 explains such side channels.)

### Mechanism 2: `Expect: 100-continue`

Before sending a large body, a client can send only the headers with **`Expect: 100-continue`** and wait. If the server would accept the request, it answers with an interim `100 Continue` and the client sends the body. If not, it answers at once with a final status, such as `401`, `413` or `417 Expectation Failed`, and the client never uploads the body it would have rejected. Clients such as `curl` do this automatically for large uploads; a server or proxy that ignores the header makes them pause for a short timeout before sending the body anyway, which shows up as an unexplained delay on uploads.

### Mechanism 3: range requests

```text
 client: GET /videos/intro.mp4 HTTP/1.1
         Range: bytes=0-1048575                   first 1 MiB

 server: HTTP/1.1 206 Partial Content
         Accept-Ranges: bytes
         Content-Range: bytes 0-1048575/52428800   this part / total size
         Content-Length: 1048576
         ETag: "a1b2c3"

 resume later, only if the file has not changed:
 client: GET /videos/intro.mp4 HTTP/1.1
         Range: bytes=1048576-
         If-Range: "a1b2c3"
 server: 206 with the rest if the ETag still matches; otherwise 200 with the whole file
```

`Accept-Ranges: bytes` advertises support. A range outside the representation gets **`416 Range Not Satisfiable`** with `Content-Range: bytes */52428800` stating the real size. `If-Range` prevents a resumed download from stitching together parts of two different versions of the file. A request for several ranges gets a `multipart/byteranges` response.

### Guarantees and non-guarantees

Negotiation guarantees that a server can serve the best available representation for each client. It does not guarantee that the server honours the preferences (it may ignore them), or that caches keep variants apart unless `Vary` says so. Range requests guarantee byte-exact parts of one version of a representation only when combined with a validator (`If-Range`).

### Failure and diagnosis

- **Compressed responses served to clients that cannot decode them**, or the wrong language served from a CDN. Cause: a missing `Vary` header. Evidence: the cached response's headers; `curl -H 'Accept-Encoding: identity' -v` receiving a compressed body. Fix: send `Vary` for every negotiated header.
- **Double compression or a wrong `Content-Length`.** An application compresses its response and a proxy compresses it again, or a proxy decompresses but keeps the old `Content-Length`. Clients see garbage or truncated bodies. Fix: compress in one place, and let the component that compresses set the framing.
- **Upload delays of about a second.** Each large upload stalls before the body is sent. Cause: `Expect: 100-continue` sent to a server or proxy that does not answer it. Fix: support it, or turn it off in the client.

### Backend connection

Let the reverse proxy or CDN do compression for dynamic responses unless you have a reason not to, mark every negotiated response with `Vary`, and support range requests with strong validators on any endpoint that serves large files.

---

## 7.9 Connections: Persistence, Pipelining, Head-of-Line Blocking

### Definitions

A **persistent connection** (keep-alive) is a transport connection that carries many request–response exchanges over its lifetime instead of one. In HTTP/1.1 connections are persistent by default; either side sends `Connection: close` to end the connection after the current exchange.

**Pipelining** is sending several HTTP/1.1 requests on one connection without waiting for each response. Responses must still come back in request order.

**HTTP-level head-of-line blocking** is the delay of a response that is ready, because a slower response ahead of it on the same connection must be sent first.

### Why it exists

HTTP/1.0 opened a new TCP connection for every request and closed it after the response. A page with 50 images paid 50 handshakes, 50 slow starts (Chapter 6, Section 6.12) and, for HTTPS, 50 TLS handshakes, and left 50 `TIME_WAIT` entries (Chapter 6, Section 6.8). Persistent connections pay the setup once and keep a warmed-up congestion window.

### Mechanism 1: connection reuse and idle timeouts

```text
 TCP (+ TLS) handshake             paid once
 GET /a ─────────►  ◄──── 200
 GET /b ─────────►  ◄──── 200      no new handshake, larger cwnd
 ... idle ...                      both sides keep the connection open
 idle timeout: one side closes     FIN (Chapter 6, Section 6.8)
```

Both sides keep idle connections only for a limited time, the **idle timeout** (configurable on every server, proxy and client pool, with defaults that vary by product). Servers close idle connections to free file descriptors and memory; clients' pools close them to avoid using connections the server has already given up on.

### Mechanism 2: pipelining, and why it failed

HTTP/1.1 has no way to tell responses apart except their order. If a client pipelines `GET /report` (10 seconds) and `GET /logo.png` (5 ms), the logo's response must wait 10 seconds behind the report: head-of-line blocking at the HTTP layer. Many proxies also handled pipelined requests incorrectly. Browsers disabled pipelining, and it is effectively unused. Instead, browsers open **several connections per origin** (commonly up to six) to get parallelism, which multiplies handshakes and server connections. HTTP/2 was designed to remove that workaround (Section 7.10).

### Mechanism 3: connection, request, stream and thread are different things

Four lifetimes are easy to confuse, and confusing them leads to wrong capacity reasoning:

```text
 HTTP/1.1: one TCP connection, requests one after another
   TCP connection (4-tuple; lives for minutes)
     ├── request 1 → response 1     handled by worker thread T7
     ├── request 2 → response 2     handled by T3 (any free worker)
     └── idle                       no thread at all; only a socket waiting in epoll

 HTTP/2: one TCP connection, many concurrent streams
   TCP connection
     ├── stream 1 → request A       worker T1
     ├── stream 3 → request B       worker T4
     ├── stream 5 → request C       waiting for a worker
     └── stream 7 → request D       worker T9
```

- A **connection** is a transport-level channel (Chapter 6): it has a handshake, buffers and an idle timeout.
- A **request** is one semantic exchange; many share one connection.
- A **stream** (HTTP/2 and HTTP/3) is the slot in which one exchange travels within a multiplexed connection. It is not a thread.
- A **thread** (or virtual thread, or event-loop callback, Chapter 5, Section 5.13) is what executes your handler. It is usually assigned when a complete request is ready, and released when the response is written.

On a modern server, an idle keep-alive connection holds **no worker thread**: its socket sits in the server's `epoll` set (Chapter 4, Section 4.5) until bytes arrive. It still holds a file descriptor, buffer memory and a slot in the load balancer's connection table, which is why idle timeouts exist. (Old servers with blocking I/O did tie one thread to each connection, idle or not; that model is where the myth comes from.) Chapter 11 develops how servers map these lifetimes onto threads and queues.

> **Java lens:** Tomcat's NIO connector (the default in Spring Boot) parks idle keep-alive sockets in a poller thread and hands sockets with readable data to a worker pool. It therefore has separate limits for **connections** (`maxConnections`) and **worker threads** (`maxThreads`), plus an accept backlog (`acceptCount`, the `listen` backlog of Chapter 4, Section 4.6). Reaching the connection limit and reaching the thread limit produce different symptoms.

### Guarantees and non-guarantees

Persistent connections guarantee that later requests on a connection skip connection setup. They do not guarantee that a connection is still usable when you pick it from a pool (the server may have closed it), that requests on one HTTP/1.1 connection run in parallel (they do not), or that keeping more connections open is free.

### Failure and diagnosis

**The keep-alive race.** A server closes a connection after 60 seconds of idleness. The client's pool believes it is good for 90 seconds. At second 60.001, the client sends a request on it just as the server's FIN is in flight. The request either reaches a socket that is already closing (the server's kernel answers with a reset) or the client reads end-of-file immediately; either way, no response. Evidence: `Connection reset` or "connection closed before response" errors on the first request after a pause, on connections that were idle for about the server's timeout; a capture shows the server's FIN crossing the client's request. Fix: make the client's idle timeout shorter than the server's (and the server's shorter than any load balancer's in between). Many clients retry this specific failure automatically, but only for idempotent requests, because they cannot prove the server did not read the request.

### Backend connection

Configure idle timeouts as a chain: client pool < load balancer < server, each shorter than the next hop's. Size connection limits and thread limits separately. And reuse connections: one shared client per downstream service, with a pool sized for its concurrency.

---

## 7.10 HTTP/2 and HTTP/3

### Definitions

**HTTP/2** (RFC 9113) carries HTTP semantics over one TCP connection as **binary frames**. Each request–response exchange travels in its own **stream**, identified by a number, and frames of many streams are interleaved on the connection (**multiplexing**). Header fields are compressed with **HPACK** (RFC 7541).

**HTTP/3** (RFC 9114) carries the same semantics over **QUIC** (Chapter 6, Section 6.10): each request uses its own QUIC stream, and header fields are compressed with **QPACK** (RFC 9204).

### Why they exist

HTTP/1.1 has three costs that grow with modern applications: one exchange at a time per connection (so clients open several connections and still suffer HTTP-level head-of-line blocking), verbose headers repeated on every request (cookies, user agent and tokens can be hundreds of bytes each time), and text parsing whose ambiguities cause bugs such as smuggling. HTTP/2 fixes all three on TCP. TCP's own head-of-line blocking (Chapter 6, Section 6.10) remains, and HTTP/3 removes that by moving to QUIC.

### Mechanism 1: HTTP/2 frames and streams

Over TLS, the client and server agree on HTTP/2 during the TLS handshake using ALPN (application-layer protocol negotiation, Chapter 8), with the identifier `h2`. The client then sends a fixed **connection preface** (`PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`) followed by a `SETTINGS` frame, and the server sends its own `SETTINGS`.

Every frame starts with a 9-byte header:

```text
 +-----------------------------------------------+
 |                 Length (24 bits)              |   payload size
 +---------------+---------------+---------------+
 |   Type (8)    |   Flags (8)   |
 +-+-------------+---------------+-------------------------------+
 |R|                 Stream Identifier (31 bits)                 |   0 = the connection itself
 +-+-------------------------------------------------------------+
 |                   Frame Payload ...                           |
 +---------------------------------------------------------------+
```

| Frame type | Purpose |
|---|---|
| `HEADERS` | starts a stream; carries the compressed header block of a request or response |
| `DATA` | carries body bytes |
| `SETTINGS` | connection parameters: maximum concurrent streams, initial window size, header table size |
| `WINDOW_UPDATE` | grants flow-control credit (Mechanism 3) |
| `RST_STREAM` | cancels one stream without touching the others |
| `GOAWAY` | "I am closing this connection; I processed streams up to number N" |
| `PING` | measures RTT and checks liveness |
| `PUSH_PROMISE` | server push (see below) |
| `CONTINUATION` | continues a header block too large for one frame |

The start line becomes **pseudo-header fields**, whose names begin with a colon: `:method`, `:scheme`, `:authority` (in place of `Host`) and `:path` for requests, `:status` for responses. Streams opened by the client have odd numbers (1, 3, 5, ...), stream numbers are never reused on a connection, and a frame with the **END_STREAM** flag closes that side of the stream: the equivalent of "end of body".

**One connection, many requests.** The timeline below compares fetching three resources over one HTTP/1.1 connection and over one HTTP/2 connection, where `/a` is slow:

```text
 HTTP/1.1, one connection (no pipelining)
 time ──────────────────────────────────────────────────────────────►
 req a ─►[ server works on a ......... ]◄─ resp a
                                          req b ─►[b]◄─ resp b
                                                         req c ─►[c]◄─ resp c
 b and c wait behind a: head-of-line blocking at the HTTP layer

 HTTP/2, one connection
 time ──────────────────────────────────────────────────────────────►
 HEADERS(1: a) HEADERS(3: b) HEADERS(5: c) ─►   all sent at once
                 ◄─ HEADERS+DATA(3: b)          b finished first
                 ◄─ HEADERS+DATA(5: c)
 ◄────────────────────────── HEADERS+DATA(1: a) a finishes last; nobody waited for it
```

**Stopping cleanly.** `RST_STREAM` lets a client cancel one request (a user navigated away) without losing the connection. `GOAWAY` lets a server shut down gracefully: it names the highest stream it processed, so the client knows that any higher-numbered request was **not processed** and can be retried safely on a new connection, even if it was a `POST`. This is a stronger guarantee than HTTP/1.1 can give when a connection closes.

**Server push** (`PUSH_PROMISE`) let a server send responses the client had not requested yet, such as a stylesheet with a page. It was hard to use well (it often pushed resources the browser already had cached), and major browsers have removed support. The `103 Early Hints` interim response, which tells the browser what to fetch early, is the usual replacement.

### Mechanism 2: HPACK

Headers repeat heavily: every request to the same API carries the same `:authority`, `user-agent`, `accept` and often the same long `authorization` or `cookie`. HPACK compresses them with three tools:

1. A **static table** of 61 common fields and values (`:method: GET` is entry 2, `:path: /` is entry 4); a field in it is sent as a single index, about one byte.
2. A **dynamic table**, a per-connection list of recently sent fields that both ends maintain identically; sending a field once adds it, and later requests refer to it by index. Its size is bounded by a setting (4,096 bytes by default).
3. **Huffman coding** of literal strings.

```text
 request 1: :method GET (static index) · :path /v1/orders/42 (literal, Huffman)
            authorization: Bearer eyJ... (literal, ~900 bytes; added to the dynamic table)
 request 2: :method GET · :path /v1/orders/43 (literal)
            authorization: (dynamic table index; ~1 byte instead of ~900)
```

The dynamic table is **connection state** that both ends must keep in sync, which is one reason HTTP/2 needs frames delivered in order and is a reason HPACK is a security-sensitive parser. HPACK was designed so that an attacker cannot learn secret header values by observing compressed sizes, a weakness of earlier, general-purpose header compression.

### Mechanism 3: HTTP/2 flow control and concurrency

TCP's window (Chapter 6, Section 6.8) controls the connection as a whole. If one stream's consumer stops reading, TCP cannot slow just that stream; its data would fill the shared buffers and stall every other stream. HTTP/2 therefore adds its own flow control, for `DATA` frames only, at two levels: a **window per stream** and a **window for the connection**, each 65,535 bytes initially (changeable by `SETTINGS`). A sender may send `DATA` only within both windows; the receiver grants more with `WINDOW_UPDATE` as its application consumes data.

A peer also limits how many streams may be open at once (`SETTINGS_MAX_CONCURRENT_STREAMS`). Requests beyond that limit wait on the client, or the client opens another connection.

### Mechanism 4: what HTTP/2 does not fix, and HTTP/3

All of HTTP/2's streams share one TCP byte stream. When a TCP segment is lost, the kernel holds back every later byte until it is retransmitted (Chapter 6, Section 6.10), so frames of every stream stall, even though they belong to independent requests. On a lossy mobile link, HTTP/2's single connection can do worse than HTTP/1.1's six.

**HTTP/3** maps each request onto its own QUIC stream. A lost packet delays only the streams whose data it carried. The rest of HTTP/3 mirrors HTTP/2: `HEADERS` and `DATA` frames (carried on the request's QUIC stream), a control stream for `SETTINGS` and `GOAWAY`, and **QPACK**, a version of HPACK redesigned so that header decoding does not force one stream to wait for another's packets. QUIC also merges the transport and TLS handshakes into one round trip (Chapter 6, Section 6.12).

**How a client discovers HTTP/3.** A client usually first connects over TCP (HTTP/1.1 or HTTP/2). The server announces HTTP/3 with a header such as `Alt-Svc: h3=":443"; ma=86400` ("the same service is available over HTTP/3 on UDP port 443, for the next 86,400 seconds"), or through a DNS record of type `HTTPS`. Later connections try QUIC and fall back to TCP if UDP is blocked.

### Comparison

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP (+ TLS) | TCP + TLS (in practice) | QUIC (TLS 1.3 built in) over UDP |
| Framing | text lines; `Content-Length` or chunked | binary frames with lengths | binary frames on QUIC streams |
| Concurrency on one connection | one exchange at a time | many streams, interleaved | many streams, independent |
| Header compression | none | HPACK | QPACK |
| HTTP-level head-of-line blocking | yes | no | no |
| Transport-level head-of-line blocking | yes (per connection) | yes: one loss stalls all streams | no: only the affected stream |
| New connection cost (HTTPS) | TCP + TLS round trips | TCP + TLS round trips | one round trip (0 on resumption, with replay limits) |

### Guarantees and non-guarantees

HTTP/2 and HTTP/3 guarantee the same semantics as HTTP/1.1 with less overhead per request and with many concurrent exchanges per connection. They do not make a slow server faster, they do not remove loss (HTTP/2 still suffers from TCP head-of-line blocking), and they concentrate traffic: one connection now carries everything a client sends, which changes load balancing (Chapter 6, Section 6.13) and makes per-connection limits more important.

### Failure and diagnosis

- **Load pinned to one backend.** gRPC or HTTP/2 clients behind an L4 load balancer keep sending everything to the backends they first connected to (Chapter 6, Section 6.13). Fix: balance per request at L7, or on the client.
- **Stalls under loss.** An HTTP/2 API used from mobile networks has much worse tail latency than its HTTP/1.1 version. Evidence: retransmissions in captures; all streams on a connection stalling together. Fix: HTTP/3 at the edge.
- **Stream-reset floods.** An attacker opens streams and immediately cancels them with `RST_STREAM`, over and over, making the server start and abandon work far faster than any concurrency limit would allow (the "rapid reset" attack disclosed in 2023). Evidence: very high rates of stream resets per connection in server metrics. Fix: server and proxy versions that limit resets per connection, and connection-level rate limits.
- **Requests queued on the client.** Latency rises although the server is idle, because the client has hit the server's maximum concurrent streams on its single connection. Evidence: client-side queueing time; the server's `SETTINGS`. Fix: allow more streams, or let the client open more connections.

### Backend connection

Most services today receive HTTP/2 or HTTP/3 at the edge and speak HTTP/1.1 or HTTP/2 to the application behind the proxy (Chapter 10). Application code sees the same requests either way. What changes is the operational picture: fewer, busier connections; load balancing that must understand streams; and new limits (concurrent streams, flow-control windows) that can cause queueing.

---

## 7.11 Redirects

### Definition

A **redirect** is a `3xx` response that tells the client to make a new request to another URI, given in the **`Location`** header. Redirects differ in two ways: whether they are **permanent** (the client and caches may remember them and go straight to the new URI next time) and whether the client must **repeat the same method and body** or may switch to `GET`.

### Why it exists

Resources move: a site moves from `http` to `https`, an API from `/v1` to `/v2`, a domain is renamed. Redirects let old links keep working. They also support a key web pattern: after a form `POST`, redirect the browser to a page it can safely reload, so that pressing Reload does not submit the form again (**Post/Redirect/Get**).

### Mechanism: the five codes

| Code | Permanent? | Method on the new request | Typical use |
|---|---|---|---|
| `301 Moved Permanently` | yes | should stay the same, but clients may (and browsers do) change `POST` to `GET`, for historical reasons | moving pages permanently; `GET` only |
| `302 Found` | no | same caveat as 301: `POST` is commonly changed to `GET` | temporary moves; legacy |
| `303 See Other` | no | always `GET` | Post/Redirect/Get: "the result of your `POST` is at this URL" |
| `307 Temporary Redirect` | no | **must** be the same, with the same body | temporarily moving an API endpoint that receives `POST`s |
| `308 Permanent Redirect` | yes | **must** be the same, with the same body | permanently moving an API endpoint |

```text
 Post/Redirect/Get
 browser                                         server
   │ POST /orders  (form data) ──────────────────►│ creates order 9001
   │◄──────────────── 303 See Other               │
   │                  Location: /orders/9001      │
   │ GET /orders/9001 ───────────────────────────►│
   │◄──────────────── 200 OK (order page)         │
 pressing Reload now repeats the GET, not the POST
```

Redirect responses also interact with caching: `301` and `308` are cacheable by default, and browsers keep them, sometimes for a long time.

### Guarantees and non-guarantees

A `307` or `308` guarantees that a compliant client repeats the same method and body. A `301` or `302` guarantees nothing about the method for `POST`. No redirect guarantees that the client follows it: browsers do, but many server-side HTTP clients do not by default (`curl` follows only with `-L`; Java's `java.net.http.HttpClient` follows none unless configured), and clients typically drop the `Authorization` header when a redirect goes to another host, so that credentials do not leak.

### Failure and diagnosis

- **A lost request body.** An API endpoint moved and the server answers `POST`s with `301`; clients re-send as `GET` with no body and get `405` or create nothing. Fix: use `308` (or `307`) for anything that receives non-`GET` methods.
- **A permanent redirect that cannot be undone.** A misconfigured `301` from `/` to a maintenance page is cached by browsers; after the fix, users who saw it keep being sent there. Evidence: the problem persists only in browsers that visited during the window. Fix: use `302` or `307` for anything that might change, and be deliberate with `301` and `308`.
- **Redirect loops.** A proxy terminates HTTPS and forwards plain HTTP to the application, which "redirects to HTTPS"; the browser follows, and the cycle repeats until it gives up ("too many redirects"). Fix: the application must learn the original scheme from the proxy (`X-Forwarded-Proto`, Chapter 10).
- **Open redirects.** An endpoint such as `/login?next=https://evil.example` redirects wherever its parameter says, which phishing pages abuse to borrow a trusted domain. Fix: allow only relative paths or a list of known destinations (Chapter 9).

### Backend connection

Use `308` and `307` for APIs, `303` after form submissions, and permanent codes only when the move really is permanent. A redirect from `http` to `https` still sends the first request in clear text; Chapter 8's HSTS closes that gap.

---

## 7.12 HTTP Caching in Depth: Freshness, Validation (`ETag`), `Cache-Control`, and `Vary`

### Definitions

An **HTTP cache** (standardized in **RFC 9111: HTTP Caching**) is a local store of response messages, located either inside a single user's client (**private cache**, such as a browser cache) or inside an intermediary serving many users (**shared cache**, such as a CDN edge node or reverse proxy). It satisfies future equivalent requests either **without contacting the origin server at all (freshness)** or by **asking the origin whether the stored copy is still valid (validation)**.

**Freshness lifetime** is the time window (`max-age=N` or `s-maxage=N` seconds) during which a cached response may be served directly to a caller with **zero origin network traffic (`0 RTT` to origin)**.

**Cache validation (revalidation)** is the process by which a cache holding a **stale** response sends a **conditional request** (`If-None-Match` with an **`ETag`** or `If-Modified-Since` with a timestamp) to the origin server; if the resource has not changed, the origin responds with **`304 Not Modified` (headers only, no body)** and the cache refreshes its entry's timer and serves the stored body.

### Why it exists

The fastest network request is the one you never make; the second fastest is the one that transfers zero body bytes.

Consider a product-catalog page or a static JavaScript bundle (`app.8f91a.js`, 400 KB) requested 50,000 times per second globally:
1. If a browser's **private cache** knows `app.8f91a.js` is **fresh** (`Cache-Control: public, max-age=31536000, immutable`), the browser reads the 400 KB file from local RAM/SSD in **0.1 milliseconds** with zero packets sent on the network.
2. If a user in Hyderabad requests a product page for the first time, a **shared CDN cache** in Hyderabad (5 ms RTT away) can serve a cached `GET /v1/products/42` (`Cache-Control: public, s-maxage=60`) in **5 ms** instead of crossing submarine cables to the origin server in Virginia (220 ms RTT), shielding your origin database from most of its read load.
3. Even after those 60 seconds expire (**stale**), if Product 42's price has not changed, the CDN sends a tiny `GET /v1/products/42` with `If-None-Match: "v14"` to the origin, receives a 150-byte **`304 Not Modified`** header block, and resets the 60-second timer without re-transferring the product payload.

*(Scope note: This section covers **protocol-level HTTP caching** defined by RFC 9111 across browsers, reverse proxies, and CDNs. In-application caching inside your backend using Redis or Caffeine is covered in Chapter 21.)*

### Mechanism 1: The Complete HTTP Caching Decision Flowchart

Every standards-compliant HTTP cache (browser or CDN) executes this exact state machine on a `GET` or `HEAD` request:

```
                       Incoming GET /v1/products/42
                                    │
                                    ▼
                 ┌─────────────────────────────────────┐
                 │ 1. Is a matching response in cache? │
                 │    (Matches URI + Vary headers)     │
                 └──────┬───────────────────────┬──────┘
                     NO │                       │ YES
                        ▼                       ▼
          Forward GET to Origin      ┌──────────────────────────────────────┐
                        │            │ 2. Does it have "no-cache" OR is     │
                        │            │    Age >= Freshness Lifetime (Stale)?│
                        │            └──────┬────────────────────────┬──────┘
                        │                NO │ (It is FRESH.)         │ YES (Needs Validation.)
                        │                   ▼                        ▼
                        │        ┌─────────────────────┐  ┌────────────────────────────────────┐
                        │        │ CACHE HIT (Fresh).  │  │ 3. Send Conditional GET to Origin: │
                        │        │ Serve stored copy   │  │    If-None-Match: "etag-v14"       │
                        │        │ immediately.        │  │    (or If-Modified-Since: <date>)  │
                        │        │ (0 packets to origin│  └──────────┬─────────────────┬───────┘
                        │        └─────────────────────┘             │                 │
                        │                                 Origin says│      Origin says│
                        │                                 304 Not    │      200 OK     │
                        ▼                                 Modified   ▼      (New body) ▼
          ┌───────────────────────────┐                ┌───────────────────┐  ┌───────────────────┐
          │ Origin returns 200 OK     │                │ REVALIDATED HIT.  │  │ CACHE REFRESH.    │
          │ (Store in cache if allowed│                │ Update headers &  │  │ Replace cached    │
          │  by Cache-Control; return │                │ reset max-age;    │  │ entry with new    │
          │  200 OK + body to client) │                │ serve stored body!│  │ 200 OK + new body!│
          └───────────────────────────┘                └───────────────────┘  └───────────────────┘
```

Look at Step 2 (`Age >= Freshness Lifetime`): how does a cache know whether a stored response is **fresh** or **stale**?
* **Freshness Lifetime:** Computed in strict priority order:
  1. In a **shared cache (CDN)** only: if `Cache-Control` contains **`s-maxage=N`**, the freshness lifetime is N seconds.
  2. Otherwise, if `Cache-Control` contains **`max-age=N`**, the freshness lifetime is N seconds.
  3. Otherwise, if the legacy **`Expires`** header is present (`Expires: Sat, 17 Oct 2026 15:00:00 GMT`), freshness lifetime is Expires − Date.
  4. Otherwise, a cache **may assign a heuristic lifetime** to a response that does not forbid caching. When a `Last-Modified` date is present, a common choice is a fraction of the time since that date (RFC 9111 mentions 10%). A response that says nothing about caching can therefore still be cached for hours, which is why every route needs explicit directives (Diagnosis Lab 4).
* **Current `Age`:** The total seconds elapsed since the response was generated (or last validated) by the origin server. When a CDN serves a cached response that it fetched from the origin 18 seconds ago, the CDN adds the header **`Age: 18`**. If `max-age=60` and `Age: 18`, a downstream browser cache knows that only 60 − 18 = **42 seconds** of freshness remain.

### Mechanism 2: Cache-Control Directives (`no-cache` vs. `no-store`, `private` vs. `public`)

The most common HTTP caching interview question is: ***"What is the difference between `Cache-Control: no-cache` and `Cache-Control: no-store`?"***
Many people guess that `no-cache` means "do not cache this response." **That is wrong.**

* **`Cache-Control: no-cache` permits storage, but requires successful validation before reuse.** It does not require the cache to store the response; an implementation may choose not to cache it.
* **`Cache-Control: no-store` prohibits an HTTP cache from storing this exchange.** It is not a secure-erasure guarantee for previously stored copies, application logs, screenshots, or non-cache history mechanisms.

The response directives:

| `Cache-Control` Directive | Who It Applies To | Exact Protocol Meaning | Typical Backend Use Case |
|---|---|---|---|
| **`no-store`** | Private & Shared Caches | **Do not store this exchange in an HTTP cache.** This does not erase earlier copies or govern all application storage. | Highly sensitive data (bank account numbers, medical records, one-time recovery codes). |
| **`no-cache`** | Private & Shared Caches | **May store the response in cache**, but **MUST revalidate** with the origin (`If-None-Match` / `ETag`) before serving it on every single request. | Dynamic HTML pages (`index.html`), mutable API resources where you want instant updates (`200`) when data changes, but cheap header-only `304` responses when unchanged. |
| **`private`** | **Browser (Private) Cache ONLY** | Only the single end-user's private cache (browser) may store this response; **shared caches (CDNs/proxies) MUST NOT store it.** | Personalized responses (`GET /v1/me/profile`, `GET /v1/dashboard`) that belong to one logged-in user. |
| **`public`** | Private & Shared Caches | Explicitly permits shared caches (CDNs) to store the response (even if the request carried an `Authorization` header, which otherwise blocks shared caching by default.). | Public catalog pages, public API docs, static assets (`logo.png`, `app.abc12.js`). |
| **`max-age=N`** | Private & Shared Caches | The response is **fresh for N seconds** from when it was generated at the origin. | `max-age=60` on a public API response; `max-age=31536000` (1 year) on content-hashed static files. |
| **`s-maxage=N`** | **Shared Caches (CDNs) ONLY** | Overrides `max-age` **strictly for shared caches (CDNs)**. Ignored by browser private caches. | `Cache-Control: public, max-age=0, s-maxage=300`: tells the **CDN** to cache for 5 minutes (absorbing traffic spikes, while allowing your backend to purge the CDN instantly on update.), while telling **browsers** (`max-age=0`) to always check with the CDN so users never get stuck with stale data in their local browser. |
| **`must-revalidate`** | Private & Shared Caches | Once the response becomes stale (`Age >= max-age`), the cache **MUST NOT** serve the stale copy if the origin is unreachable; it must revalidate or return `504`. | Financial quotes or inventory counts where serving stale data after expiry is worse than an error. |
| **`immutable`** *(RFC 8246)* | Private Caches (Browsers) | Tells the browser: *"The bytes at this exact URL will **never** change during its `max-age` lifetime; do not even send a conditional `If-None-Match` revalidation if the user clicks Reload."* | Content-hashed bundles (`main.9f8a12.js`, `styles.44c01.css`) where any code change produces a brand-new filename URL. |
| **`stale-while-revalidate=M`** *(RFC 5861)* | Private & Shared Caches | If a request arrives within M seconds after `max-age` expired, serve the slightly stale cached copy **immediately (`0 ms` wait)** and revalidate with the origin **asynchronously in the background**. | High-traffic news feeds, product listings, and API gateways where hiding origin tail latency is critical. |
| **`stale-if-error=K`** *(RFC 5861)* | Private & Shared Caches | If the response is stale and the origin returns `500`/`502`/`503`/`504` (or times out), keep serving the stale cached copy for up to K seconds instead of showing users an error page. | Graceful degradation on catalog/content APIs during an origin database outage. |

### Mechanism 3: Validation Validators: `ETag` (`If-None-Match`) vs. `Last-Modified` (`If-Modified-Since`)

When a cached response needs validation (either because `max-age` expired or because `Cache-Control: no-cache` was used), how does the cache ask the origin *"Has this resource changed since the copy I have?"*

HTTP provides two validator pairs:

1. **Timestamp Validator (`Last-Modified` + `If-Modified-Since`):**
   * Origin sends `Last-Modified: Sat, 17 Oct 2026 10:00:00 GMT` on the initial `200 OK`.
   * On revalidation, client sends `If-Modified-Since: Sat, 17 Oct 2026 10:00:00 GMT`.
   * **Why `Last-Modified` is flawed (three classic limitations):**
     1. **1-second resolution limit:** HTTP date formats have a resolution of **1 second**. If a resource is updated twice within the same second, `Last-Modified` cannot detect the second update.
     2. **Clock skew / no-op rebuilds:** If a deployment script touches or rebuilds a file without changing its contents, the timestamp changes and triggers unnecessary full downloads.
     3. **Hard to compute on dynamic composite JSON:** If an API endpoint joins 4 database tables, what is its single `Last-Modified` date?

2. **Opaque Entity Tag Validator (`ETag` + `If-None-Match`, preferred):**
   * An **`ETag`** is an opaque ASCII string quoted in double quotes (such as `ETag: "33a64df5"` or `ETag: "v14"`) generated by the server, typically a fast hash (CRC32c, xxHash, MD5/SHA-1 prefix) of the response bytes, or the database row's `version` column (`"user-42-v7"`).
   * **Strong `ETag` (`"abc"`) vs. Weak `ETag` (`W/"abc"`):**
     * A **strong `ETag` (`"abc"`)** guarantees the representation is **byte-for-byte identical** (including compression headers), and is required if you want to use the `ETag` for byte-range requests (`If-Range`).
     * A **weak `ETag` (`W/"abc"`)** guarantees the two representations are **semantically equivalent** even if their exact bytes differ slightly (for example, before vs. after on-the-fly gzip compression, which is why Nginx automatically converts strong `ETag`s into weak `W/"..."` `ETag`s when compressing responses on the fly.).
   * On revalidation, the client sends **`If-None-Match: "v14"`** ("Send me the body *only if none of your current ETags match `"v14"`*"). If the server's current `ETag` is still `"v14"`, the server skips serializing/sending the body and returns **`304 Not Modified`**.

### Mechanism 4: The `Vary` Header and How Shared Caches Key Responses

By default, a shared CDN cache uses the **request URI** (`https://api.example.com/v1/products/42`) as the primary cache key.
What happens if your endpoint returns different bytes for the same URI depending on a request header, for example:
* `Accept-Encoding: br` vs. `Accept-Encoding: gzip`
* `Accept-Language: hi` vs. `Accept-Language: en`

If the CDN keyed *only* by URI, User 1 (`Accept-Language: hi`) would populate the cache with Hindi JSON, and User 2 (`Accept-Language: en`) would be served Hindi JSON from the CDN.

Adding **`Vary: Accept-Encoding, Accept-Language`** to your origin response tells the CDN: ***"Include the values of `Accept-Encoding` and `Accept-Language` in the cache key alongside the URI."***

**The `Vary` Cache-Explosion Trap:** What if a developer carelessly writes **`Vary: User-Agent`** or **`Vary: Cookie`** on a public CDN response?
Because there are **tens of thousands of distinct `User-Agent` strings** (every browser build, OS patch, and phone model differs.) and every user has a unique `Cookie`, adding `Vary: User-Agent` or `Vary: Cookie` partitions the CDN cache into millions of tiny buckets, collapsing your CDN cache hit ratio toward **0%**. (If a response depends on `Cookie` or `Authorization`, mark it `Cache-Control: private` so it is cached only in the user's own browser, or normalize the request into a low-cardinality header like `X-Device-Class: mobile|desktop` at the CDN edge.)

### Guarantees and non-guarantees

RFC 9111 guarantees that compliant shared caches will **never** store a response containing `Cache-Control: private` or `Cache-Control: no-store`, and will **never** serve a cached response to a request carrying an `Authorization` header unless the response explicitly opts into shared caching via `public`, `s-maxage`, or `must-revalidate` (RFC 9111 §3.5).

HTTP caching does **not** guarantee that a buggy application framework won't accidentally overwrite your headers or omit `Cache-Control: private` on a cookie-authenticated endpoint.

### Failure and diagnosis

* **The Cross-User Data Leak via CDN (Caching a `Set-Cookie` / Personalized Response):**
  * *Symptom:* User A logs in, visits `/account/summary`, and sees **User B's name, email, and order history**.
  * *Mechanism:* The `/account/summary` endpoint authenticated users via a session `Cookie`, and a well-meaning developer configured the CDN to "cache all `GET 200 OK` responses for 60 seconds" (or put `Cache-Control: public, max-age=60` on a global controller filter.). Unlike `Authorization` (which RFC 9111 treats as private by default), **RFC 9111 does NOT forbid a shared cache from caching a response to a request that contained a `Cookie` header if `max-age` / `public` is present.** The CDN cached User B's `/account/summary` HTML (and sometimes even User B's `Set-Cookie` header.) and served it to User A.
  * *Fix:* Every authenticated or personalized endpoint MUST return **`Cache-Control: private, no-cache`** (or **`Cache-Control: no-store`**), and CDNs must be configured to bypass cache when session cookies are present or strip `Set-Cookie` before caching public pages.

### Backend connection

Whenever you build a `GET` endpoint, make an explicit choice among three standard `Cache-Control` profiles:
1. **Sensitive / user-specific API response:** `Cache-Control: private, no-cache` (or `no-store`).
2. **Public dynamic API / catalog response:** `Cache-Control: public, max-age=0, s-maxage=60, stale-while-revalidate=30` + `ETag` + `Vary: Accept-Encoding`.
3. **Versioned / content-hashed static asset (`/static/app.9f8a12.js`):** `Cache-Control: public, max-age=31536000, immutable` + `Vary: Accept-Encoding`.

---

## 7.13 Conditional Requests and Optimistic Concurrency Control (`If-Match` → `412`)

### Definitions

A **conditional request** (RFC 9110 §13) is an HTTP request that includes one or more **precondition header fields** (`If-None-Match`, `If-Modified-Since`, `If-Match`, `If-Unmodified-Since`) that test the current state of the target resource on the origin server **before** the server executes the method.

**Optimistic concurrency control over HTTP** is the pattern in which a client reads a resource's current **`ETag`** via `GET` and includes **`If-Match: "<etag>"`** on its subsequent `PUT`, `PATCH`, or `DELETE` request; if another writer modified the resource in the meantime (changing its `ETag`), the server rejects the write with **`412 Precondition Failed`**, preventing the **lost-update anomaly**.

### Why it exists

In Section 7.12, we used **`If-None-Match`** on **`GET`** requests for **cache validation** (*"Give me the body only if the `ETag` does NOT match what I already have"* → `304 Not Modified`).

Now consider the opposite problem on **writes (`PUT`, `PATCH`, `DELETE`)**: because HTTP is **stateless**, a server does not hold a database row lock across the 30 seconds while a user (or an automated workflow) is editing a document or order in their browser.

Without a precondition check, watch what happens when **Alice** and **Bob** both load `/v1/products/42` (`price: 100, stock: 50`) at the same time:
1. Alice reads the product JSON (`price: 100, stock: 50`), changes `price` to `120`, and sends `PUT /v1/products/42` with `{"price": 120, "stock": 50}`.
2. Two seconds later, Bob, who is still looking at the old JSON (`price: 100, stock: 50`), decrements `stock` to `49` and sends `PUT /v1/products/42` with `{"price": 100, "stock": 49}`.
3. Bob's `PUT` **silently overwrites Alice's price change (`120` → `100`)**. Alice's update vanished into thin air without either Alice, Bob, or the server noticing (**the Lost Update problem**, which we will also study at the SQL transaction level in Chapter 17).

### Mechanism & Concrete: Preventing Lost Updates with `ETag` + `If-Match` → `412 Precondition Failed`

HTTP solves the Lost Update problem across stateless network calls using **HTTP-level Compare-And-Swap (`ETag` + `If-Match`)**:

```
  Alice (Client A)                         Origin Server (Product 42)                  Bob (Client B)
       │                                     [State: v1, price=100, stock=50]                │
       │ 1. GET /v1/products/42                         │                                    │
       ├───────────────────────────────────────────────►│                                    │
       │ 2. 200 OK, ETag: "v1"                          │                                    │
       │    {"price":100, "stock":50}                   │    3. GET /v1/products/42          │
       │◄───────────────────────────────────────────────┤◄───────────────────────────────────┤
       │                                                │    4. 200 OK, ETag: "v1"           │
       │ 5. PUT /v1/products/42                         │       {"price":100, "stock":50}    │
       │    If-Match: "v1"                              ├───────────────────────────────────►│
       │    {"price":120, "stock":50}                   │                                    │
       ├───────────────────────────────────────────────►│                                    │
       │                                    (Checks If-Match: "v1" == current ETag "v1".     │
       │                                     Executes SQL: UPDATE products SET price=120,    │
       │                                     version=2 WHERE id=42 AND version=1;            │
       │                                     State is now ETag "v2".)                        │
       │ 6. 200 OK, ETag: "v2"                          │                                    │
       │◄───────────────────────────────────────────────┤                                    │
       │                                                │    7. PUT /v1/products/42          │
       │                                                │       If-Match: "v1"               │
       │                                                │       {"price":100, "stock":49}    │
       │                                                │◄───────────────────────────────────┤
       │                                    (Checks If-Match: "v1" != current ETag "v2".     │
       │                                     Refuses write BEFORE touching database.)        │
       │                                                │                                    │
       │                                                │    8. 412 Precondition Failed      │
       │                                                ├───────────────────────────────────►│
       │                                                │    (Bob knows a concurrent edit    │
       │                                                │     happened; re-GETs "v2", merges │
       │                                                │     price=120 & stock=49, and      │
       │                                                │     retries with If-Match: "v2".)  │
```

Compare the four conditional headers side by side so you never mix them up in an interview:

| Request Header | Validator Used | Typical Method | When Condition Succeeds vs. When It Fails |
|---|---|---|---|
| **`If-None-Match: "v1"`** | **`ETag`** | `GET`, `HEAD` *(Cache validation)* | If current ETag ≠ `"v1"` → **`200 OK`** (send new body). If current ETag = `"v1"` → **`304 Not Modified`** (no body.). |
| **`If-None-Match: *`** | **`ETag`** | `PUT` *(Create-only)* | *"Execute this `PUT` ONLY if **no representation currently exists** at this URI."* If the resource already exists → **`412 Precondition Failed`** (prevents overwriting an existing file/object.). |
| **`If-Match: "v1"`** | **`ETag`** | `PUT`, `PATCH`, `DELETE` *(Optimistic locking)* | If current ETag = `"v1"` → **Execute write (`200`/`204`)**. If current ETag ≠ `"v1"` → **`412 Precondition Failed`**. |
| **`If-Modified-Since` / `If-Unmodified-Since`** | `Last-Modified` date | `GET` / `PUT` | Date-based equivalents of `If-None-Match` (`304`) and `If-Match` (`412`). Inferior to `ETag` because of 1-second timestamp granularity. |

*(A related status code, **`428 Precondition Required`**, RFC 6585: If your API **requires** clients to send `If-Match` on every `PUT /v1/products/{id}` and a lazy client sends a `PUT` without any `If-Match` header at all, your server can reject the unguarded write with **`428 Precondition Required`**.)*

### Guarantees and non-guarantees

`ETag` + `If-Match` guarantees protection against lost updates across stateless HTTP hops **if and only if** the server evaluates the `If-Match` version check **atomically** with the database write.

`If-Match` also uses **strong comparison**: a weak ETag (`W/"..."`, Section 7.12) never matches it. A resource that supports conditional updates must therefore issue strong ETags. It does not merge concurrent changes (the client must re-read and retry), and it protects nothing for clients that omit the header, unless the server demands it with `428`.

### Failure and diagnosis

* **The Check-Then-Act Race Inside the Server (`If-Match` checked in Java, overwritten in SQL):**
  * *Buggy implementation:* A developer writes a controller that runs `SELECT version FROM products WHERE id = 42` to check `If-Match: "v1"`, sees `version == 1`, and then 5 milliseconds later runs `UPDATE products SET price = 120, version = 2 WHERE id = 42` **without `AND version = 1` in the `WHERE` clause**.
  * *Mechanism:* Just like the check-then-act thread race in Chapter 5 (Section 5.2), if Alice's request and Bob's request both run the `SELECT` at the same millisecond, both see `version == 1`, both pass the Java `if` check, and both execute the unguarded `UPDATE`.
  * *Fix:* Enforce the version check atomically in the database statement itself (`UPDATE products SET ..., version = version + 1 WHERE id = 42 AND version = 1`) and return `412 Precondition Failed` if `rowsUpdated == 0` (Chapter 17).

### Backend connection

Cloud object stores (AWS S3, Google Cloud Storage, Azure Blob Storage) and modern REST APIs (GitHub, Stripe, Kubernetes API Server via `resourceVersion`) rely heavily on `If-Match` and `If-None-Match: *` to build distributed locks, leader election, and conflict-free updates directly over HTTP without stateful connections.

---

## 7.14 Cookies as a Protocol Mechanism (`Set-Cookie` and `Cookie` Wire Syntax)

### Definitions

An **HTTP cookie** (standardized in **RFC 6265**) is a small piece of state (`name=value`) issued by a server in a **`Set-Cookie` response header field** that a compliant user agent stores locally and automatically attaches back in a **`Cookie` request header field** on subsequent requests that match the cookie's **scope** (`Domain` and `Path`) and security rules.

### Why it exists

Because HTTP is stateless (Section 7.1), a web server that sends an HTML page to a browser has no built-in protocol way to know whether the next HTTP request arriving 2 seconds later comes from the same logged-in user.

While programmatic API clients (mobile apps, backend services, and single-page applications, or SPAs: web front ends whose JavaScript calls APIs directly) often attach an explicit `Authorization: Bearer <token>` header in application code on every request, traditional browser navigation (clicking an `<a href="/settings">` link, loading `<img src="/avatar">`, or submitting an HTML `<form>`) does **not** let JavaScript attach custom `Authorization` headers to the browser's native page-load request. Cookies solve this at the HTTP protocol level: the server hands the browser a `Set-Cookie` header once (e.g., at login), and the browser can attach `Cookie: ...` to eligible requests, subject to the cookie's own attributes (below) and the browser's privacy settings (Chapter 9).

*(Scope boundary: This section covers only the **wire syntax and protocol transport** of `Set-Cookie` and `Cookie`. How browsers isolate cookies under the Same-Origin Policy, why automatic cookie attachment creates **CSRF (Cross-Site Request Forgery)** vulnerabilities, and how server-side sessions vs. JWTs are architected live in Chapters 9 and 19.)*

### Mechanism & Concrete: Exact Wire Syntax of `Set-Cookie` vs. `Cookie`

Look closely at the asymmetry between the **response header (`Set-Cookie`)** and the **request header (`Cookie`)**:

```
  1. SERVER TO CLIENT (Response Header: Sets TWO cookies with attributes)
  ────────────────────────────────────────────────────────────────────────
  HTTP/1.1 200 OK\r\n
  Content-Type: application/json\r\n
  Set-Cookie: sid=98a1b4f2; Path=/; Domain=example.com; Max-Age=3600; Secure; HttpOnly; SameSite=Lax\r\n
  Set-Cookie: theme=dark; Path=/; Max-Age=31536000; Secure; SameSite=Lax\r\n
  \r\n
  {"loggedIn":true}


  2. CLIENT TO SERVER (Subsequent Request Header: Sends ONLY name=value pairs.)
  ──────────────────────────────────────────────────────────────────────────────
  GET /v1/orders HTTP/1.1\r\n
  Host: api.example.com\r\n
  Cookie: sid=98a1b4f2; theme=dark\r\n
  \r\n
```

Three wire-level protocol rules you must know:

1. **Multiple `Set-Cookie` headers MUST NOT be folded into one comma-separated line:** Unlike fields defined with comma-separated list syntax (RFC 9110 §5.3), `Set-Cookie` cannot be combined with commas because the `Expires=Wed, 21 Oct 2026...` attribute contains a literal comma inside the date. Every cookie being set requires its own separate `Set-Cookie:` header line.
2. **The client's `Cookie:` request header sends ONLY `name1=val1; name2=val2` (separated by `; `):** The browser **never** echoes the attributes (`Path`, `Domain`, `Max-Age`, `Secure`, `HttpOnly`, `SameSite`) back to the server in the `Cookie` request header. Those attributes were instructions from the server to the browser's cookie jar telling the browser *when* and *where* to store and send the cookie.
3. **Summary of `Set-Cookie` attributes on the wire (RFC 6265):**

| Attribute | Wire Example | What It Instructs the User Agent (Browser) to Do |
|---|---|---|
| **`Max-Age=N`** / **`Expires=<date>`** | `Max-Age=3600` | Persist the cookie for N seconds (`Max-Age` takes precedence over `Expires`). If **both are omitted**, it is a **session cookie**, whose lifetime follows the browser’s session policy; session restoration can retain it across restarts. To **delete** a cookie immediately, send `Max-Age=0`. |
| **`Domain=<domain>`** | `Domain=example.com` | If **omitted** (*host-only cookie*, the safer choice), the cookie is sent **only** to the exact host that set it (`app.example.com`). If set to `Domain=example.com`, the cookie is also sent to **all subdomains** (`api.example.com`, `blog.example.com`). |
| **`Path=<path>`** | `Path=/` | Use cookie path matching: exact match, or a prefix ending at a slash boundary. `/app` matches `/app/x`, not `/apple`. Path is not a security isolation boundary. |
| **`Secure`** | `Secure` | Transmit this cookie **only over encrypted (`https://`) connections**, never over plaintext `http://`. |
| **`HttpOnly`** | `HttpOnly` | Hide this cookie from client-side JavaScript (`document.cookie` cannot read it, mitigating token theft via XSS, Chapter 9). |
| **`SameSite`** | `SameSite=Lax` (`Strict` / `Lax` / `None`) | Controls whether the browser attaches the cookie on cross-site requests initiated from third-party websites (mitigating CSRF, Chapters 9 and 19). |

### Guarantees and non-guarantees

Cookies guarantee that a compliant browser returns matching cookies on matching requests, as their attributes allow. They do not guarantee that every client accepts, keeps or sends them (many non-browser clients ignore cookies entirely).

Cookies do **not** guarantee:
* **Integrity or confidentiality of the cookie value against the client itself:** Even with `HttpOnly` and `Secure`, a user sitting at their own laptop can open Chrome DevTools (`Application → Cookies`) or run `curl -H "Cookie: role=admin"` and edit any cookie value they want. If your server stores `Set-Cookie: userId=42` without a cryptographic **HMAC signature** (a keyed tag that proves the value was not changed, Chapter 8, Section 8.5) or an **opaque server-side session lookup** (Chapter 19), any user can change `userId=42` to `userId=1` and hijack another account.

### Failure and diagnosis

* **Deleting a cookie fails (`Set-Cookie: sid=; Max-Age=0` doesn't clear the old cookie):**
  * *Mechanism:* A browser identifies a cookie in its jar by the **3-tuple `{Name, Domain, Path}`**. If the login endpoint set `Set-Cookie: sid=abc; Domain=example.com; Path=/app`, and the logout endpoint sends `Set-Cookie: sid=; Max-Age=0; Path=/` (with a different `Path` or omitting `Domain=example.com`), the browser treats that as a *different* cookie and leaves the original `sid=abc` cookie alive.
  * *Fix:* When expiring a cookie (`Max-Age=0`), always pass the **exact same `Domain` and `Path` attributes** used when creating it.

### Backend connection

Keep cookies tiny (just a single opaque session ID like `sid=98a1...`, `< 100 bytes`), because **every byte in the `Cookie` header is re-uploaded by the user's browser on every single request to that domain**.

---

## 7.15 Limits, Timeouts, Malformed Messages, and Why a Retry After a Timeout Is Ambiguous

### Definitions

**Parser resource limits** are the hard byte-size and time ceilings (`maxHeaderSize`, `maxBodySize`, `headerReadTimeout`, `keepAliveTimeout`) enforced by reverse proxies and web containers while reading an incoming HTTP message from a socket, before invoking application handlers.

**Outcome ambiguity on timeout** is the fundamental distributed-systems rule (Chapter 1, Section 1.2; Chapter 6, Section 6.9) that when a client experiences a read timeout or connection drop after sending an HTTP request, **the client cannot distinguish whether the request failed before reaching the server, failed during processing, or succeeded completely and lost only the response packet**.

### Why they exist

As we noted in Section 7.3, **the HTTP parser is the first line of defense for your server's memory and sockets.**
Suppose your server had no header timeout and no body size limit:
1. **Slowloris, a denial-of-service (DoS) attack** (an attack that makes a service unavailable by exhausting its resources rather than by breaking in): An attacker opens 8,000 TCP connections to your server and sends 1 byte of an incomplete header (`X-a: b`) every 10 seconds, never sending the terminating `\r\n\r\n`. Without a strict **header read timeout** (e.g., `5–10 seconds`), those 8,000 trickle connections hold all 8,000 connection slots hostage with almost zero attacker bandwidth.
2. **Heap exhaustion DoS:** An attacker sends `Content-Length: 2000000000` (2 GB) on 10 connections; without a body limit (`413 Content Too Large`), a naive parser buffers the stream until the process runs out of memory and crashes (in Java, with `OutOfMemoryError`).

Malformed messages are the third case. Section 7.3 showed that framing is a security boundary: a server that cannot parse a message cannot know where the next one on the connection begins, so it answers `400 Bad Request` and closes the connection. Messages that parse correctly but exceed a limit get the specific codes listed in Section 7.5 (`414` for the target, `431` for headers, `413` for the body, `408` for a client that sends too slowly).

### Mechanism 1: The Four HTTP Timeouts and the Keep-Alive Race Condition

Every production HTTP hop (Client, Load Balancer, Reverse Proxy, Origin Server) has **four distinct timeouts**, and mixing them up causes outages:

```
  Client / Proxy                                                         Origin Server
        │                                                                      │
        │── 1. TCP SYN ───────────────────────────────────────────────────────►│
        │◄─ 2. TCP SYN-ACK   [Bounded by: CONNECT TIMEOUT, e.g., 500ms–2s] ────┤
        │                                                                      │
        │── 3. Request Headers & Body ────────────────────────────────────────►│
        │                    [Bounded on Server by: HEADER / BODY READ TIMEOUT]│
        │                                                                      ├──┐ Server executes
        │                    [Bounded on Client by: READ / RESPONSE TIMEOUT]   │  │ handler / SQL
        │◄─ 4. Response Headers & Body ────────────────────────────────────────◄──┘
        │                                                                      │
        │   ... Connection now sits IDLE in Keep-Alive Pool ...                │
        │                    [Bounded on both sides by: KEEP-ALIVE IDLE TIMEOUT]
```

The keep-alive idle race between a pooled client and a server that closes idle connections is in Section 7.9. The rule from there applies to every hop: retire idle connections on the client side before the peer, or any device in between, gives up on them.

### Mechanism 2: Why a Timeout Leaves the Outcome **UNKNOWN** (And How Idempotency Keys Solve It)

Suppose your mobile app or checkout microservice sends `POST /v1/payments` (`{"amount": 100, "currency": "USD"}`) with a 5-second read timeout. At `t = 5.000s`, no response has arrived, and your HTTP client throws `SocketTimeoutException: Read timed out` (or receives `504 Gateway Timeout` from a proxy).

**Did the customer's card get charged $100?**
Look at the **three physically indistinguishable timelines** on the wire:

```
  TIMELINE A (Failed BEFORE reaching server: Card NOT charged)
  Client ── POST /v1/payments ──✕ (Dropped in network / proxy before reaching server)

  TIMELINE B (Failed DURING processing: Card NOT charged, DB rolled back)
  Client ── POST /v1/payments ──► Server starts DB tx ──✕ (Server crashes before commit)

  TIMELINE C (SUCCEEDED on server, failed AFTER commit: Card WAS CHARGED $100.)
  Client ── POST /v1/payments ──► Server charges $100 & commits DB tx.
         ◄───────✕─────────────── Server sends "HTTP/1.1 201 Created" (Takes 5.2s, or packet dropped.)
  (Client timed out at 5.0s.)
```

From the client's perspective, **Timelines A, B, and C produce the exact same exception (`Read timed out` / `504`)**.
* If the client assumes the request failed and retries `POST /v1/payments` without an idempotency key, then in Timeline C the customer is **charged $200 (double charge)**.
* If the client never retries, then in Timelines A and B the customer's order fails.

**The solution (which we will implement end to end in Chapters 13 and 24):**
1. For **safe/idempotent methods** (`GET`, `HEAD`, `PUT`, `DELETE`, `OPTIONS`), retrying on a timeout or `502`/`503`/`504` (with **exponential backoff**, doubling the wait after each failed attempt, and **jitter**, a random spread added to each wait so that many clients do not retry in lockstep; both in Chapter 24) is safe by protocol design because executing the operation a second time produces the same server state.
2. For **non-idempotent methods** (`POST`, `PATCH`), when the API supports idempotency keys, the client should generate a unique **`Idempotency-Key` value** (a UUID is one option) *before* the first attempt (`Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000`) and send that **exact same key** on every retry of that logical operation. The server stores the key atomically with the payment transaction in its database: if Timeline C happened, the retry sees that `550e8400-...` already committed and simply replays the stored `201 Created` response without charging the card a second time.

### Guarantees and non-guarantees

Timeouts guarantee that a calling thread or connection slot will not stay blocked forever when a peer or network path hangs.

Timeouts **never** guarantee that the remote server aborted the work you asked it to do. Unless you propagate a deadline (and the server actively cancels its database query when the client disconnects), a server thread will happily keep running a 45-second query long after the client timed out at 5 seconds.

### Failure and diagnosis

* **`413 Content Too Large` with zero entries in your application logs:**
  * *Diagnosis:* Check the `Server` header on the `413` response (`Server: nginx` or `awselb/2.0`). If Nginx rejected a 5 MB upload before forwarding it (Nginx's `client_max_body_size` defaults to 1 MB at the time of writing), your Spring/Node/Go process never saw a single byte of the request. Both the reverse proxy's `client_max_body_size` **and** the application container's multipart limit (`spring.servlet.multipart.max-file-size`) must be aligned.

### Backend connection

Every outbound HTTP client in your codebase must configure **three explicit numbers**:
1. **Connect timeout** (e.g., `250–1,000 ms` inside a data centre; `2–5 s` external).
2. **Read / response timeout** (sized to the upstream's p99.9 latency, e.g., `2–10 s`).
3. **Pool idle timeout** (strictly shorter than the upstream server's `keepAliveTimeout` and any NAT/LB idle timeout.).

---

---

## 7.16 End-to-End Trace: Raw Bytes and `curl -v`

### Definition

A **wire trace** is a record of the exact bytes or events of an exchange, as opposed to what application code believes it sent. Two views are used daily: a **hex dump** of the bytes (with `xxd` or a packet capture), and the **verbose output** of a client such as `curl -v`, which prints the connection steps, the request headers it sent (lines starting `>`) and the response headers it received (lines starting `<`).

### Why it matters

Most HTTP bugs are disagreements about bytes: a missing CRLF, a length counted in characters, a header a proxy added or removed, a body compressed twice. Logs written by application code show the application's intent; a trace shows what actually crossed the wire. Reading one fluently is the fastest way to settle "who is wrong".

### Concrete 1: the bytes of a request

The request `GET /v1/orders/42 HTTP/1.1`, with one header, is 53 bytes:

```text
$ printf 'GET /v1/orders/42 HTTP/1.1\r\nHost: api.example.com\r\n\r\n' | xxd
00000000: 4745 5420 2f76 312f 6f72 6465 7273 2f34  GET /v1/orders/4
00000010: 3220 4854 5450 2f31 2e31 0d0a 486f 7374  2 HTTP/1.1..Host
00000020: 3a20 6170 692e 6578 616d 706c 652e 636f  : api.example.co
00000030: 6d0d 0a0d 0a                             m....
```

Each line shows the offset in hexadecimal, 16 bytes in hex, and the same bytes as text (non-printable bytes as dots). Things to see:

- `20` is a space: the request line is three words separated by single spaces.
- `0d0a` (offset `0x1A`) is the CRLF that ends the request line; `0d0a 0d0a` at the end is the CRLF ending the `Host` line followed by the empty line that ends the header section.
- The total length is `0x35` = 53 bytes, and there is no body: a `GET` without `Content-Length` or `Transfer-Encoding` has a zero-length body (Section 7.3).

A dump like this is how you spot a bare `0a` without `0d` (a client that sends LF line endings), a UTF-8 byte sequence inside a header, or a body length that does not match `Content-Length`.

### Concrete 2: annotated `curl -v`

The exact wording of `curl`'s verbose output varies between versions; the structure below is representative.

```text
$ curl -v http://api.example.com/v1/orders/42 -H 'Accept: application/json'
*   Trying 93.184.216.34:80...                                  ① DNS done; TCP connect starts
* Connected to api.example.com (93.184.216.34) port 80          ② handshake complete (one RTT)
> GET /v1/orders/42 HTTP/1.1                                    ③ request line
> Host: api.example.com                                         ④ added by curl from the URL
> User-Agent: curl/8.5.0
> Accept: application/json                                      ⑤ our -H header replaced curl's default
>                                                               ⑥ empty line: end of headers, no body
< HTTP/1.1 200 OK                                               ⑦ status line
< Date: Sat, 17 Oct 2026 14:10:00 GMT
< Content-Type: application/json
< Content-Length: 59                                            ⑧ framing: read exactly 59 bytes
< Cache-Control: private, no-cache                              ⑨ browser may store, must revalidate
< ETag: "v7"                                                    ⑩ validator for If-None-Match / If-Match
<
{"orderId":42,"status":"PAID","total":120,"currency":"INR"}    ⑪ the 59-byte body
* Connection #0 to host api.example.com left intact             ⑫ kept alive: it could be reused
```

Lines starting with `*` are curl's own notes, `>` lines are what it sent, `<` lines are what it received. Line ⑫ says the connection was persistent and is still open (Section 7.9); a server that sent `Connection: close` would make curl report the connection closed instead.

For an `https://` URL, curl prints extra `*` lines between ② and ③ for the TLS handshake: the protocol version, the cipher, the certificate it checked, and the result of ALPN, the negotiation that picks HTTP/2 or HTTP/1.1 (Chapter 8 annotates these). If HTTP/2 is chosen, the request line appears as `> GET /v1/orders/42 HTTP/2`, the status line as `< HTTP/2 200` with no reason phrase, and response header names appear in lower case (Section 7.10).

### Concrete 3: where the time goes

`curl -w` prints timing variables, each measured from the start of the command (so they accumulate):

```text
$ curl -s -o /dev/null -w 'dns     %{time_namelookup}\nconnect %{time_connect}\ntls     %{time_appconnect}\nttfb    %{time_starttransfer}\ntotal   %{time_total}\n' https://api.example.com/v1/orders/42
dns     0.051
connect 0.102
tls     0.155
ttfb    0.221
total   0.222
```

Subtracting neighbours gives each phase: DNS 51 ms, TCP handshake 51 ms, TLS 53 ms, then 66 ms from the end of the handshake to the first response byte (one round trip plus about 15 ms of server time), and 1 ms to receive the body. This is the cold-request budget of Chapter 6 (Section 6.12) measured for real. A slow `dns` points at resolution; a slow `connect` at the network or a full accept queue; a slow `tls` at the handshake or certificate checks; a large gap between `tls` and `ttfb` at the server itself.

### Guarantees and non-guarantees

A client-side trace shows exactly what this client sent and received. It does not show what happened beyond the next hop: a proxy may have changed headers, retried, or answered from cache on the way. Compare traces on both sides of a proxy (or use a header the origin adds, such as a request ID) before concluding which component misbehaved.

### Failure and diagnosis

Habits that settle most HTTP disputes:

- Reproduce the call with `curl -v`, using the same method, headers and body as the failing client, and compare the `>` lines with what the client claims to send.
- When the body or framing is suspect, capture the bytes (`tcpdump -A` or `-X` for plain HTTP; Chapter 6, Section 6.14) or dump them with `xxd`, and check line endings and lengths.
- When latency is suspect, use `curl -w` timings to find the phase before looking at application code.

### Backend connection

Make your own services easy to trace: return a request ID header on every response and log it, log the status code and the bytes sent, and keep a `curl` command for each important endpoint in your runbook so that anyone can reproduce a call in seconds.

---

## Chapter Summary

HTTP gives TCP's byte stream structure and meaning. It is **stateless**: each request carries everything needed to interpret it, which is what lets any server or intermediary handle any request, while applications build sessions on top with cookies and tokens. Its **semantics** (methods, URIs, status codes, header fields, content; RFC 9110 and 9111) are separate from its **wire formats** (HTTP/1.1, HTTP/2, HTTP/3), so the same request means the same thing over any of them. A **URL** names a scheme, an authority (sent in `Host` and used for DNS and TLS), a path and query (sent on the request line), and a fragment that never leaves the client; data inside it must be **percent-encoded** exactly once.

An HTTP/1.1 message is a start line, header lines and an empty line, all ending in **CRLF**, followed by a body whose end is **framed** by `Content-Length` (a byte count) or by **chunked** encoding (hex-sized chunks ending in a zero chunk). Two processors that frame the same bytes differently enable **request smuggling**, so ambiguous messages must be rejected.

**Methods** state intent. **Safe** methods (`GET`, `HEAD`, `OPTIONS`) request no change; **idempotent** ones (the safe ones plus `PUT` and `DELETE`) have the same effect on state however often they are repeated; `POST` and, in general, `PATCH` are neither. Caches, crawlers, prefetchers and retry logic all rely on this, so `GET` must never change state. **Status codes** classify outcomes: `401` means "authenticate", `403` "not allowed", `404` may hide existence, `409` and `412` report conflicts, `422` invalid content, `429` rate limiting, and `500`, `502`, `503`, `504` this server, a bad upstream response, deliberate unavailability and an upstream timeout. No `5xx` or timeout proves that an operation did not happen. **Headers** carry metadata, are untrusted input, and are limited in size; **representations** are bytes plus a media type and charset (use UTF-8, count lengths in bytes), chosen by **content negotiation**, often **compressed**, and fetchable in **ranges**.

**Persistent connections** amortise handshakes; HTTP/1.1 still serves one exchange at a time per connection, so a slow response blocks later ones. A **connection**, a **request**, a **stream** and a **thread** are different things with different lifetimes; an idle connection needs no thread on a modern server, but it holds a socket until an **idle timeout**, and mismatched idle timeouts cause the keep-alive race. **HTTP/2** multiplexes many **streams** as binary **frames** over one TCP connection, compresses headers with **HPACK**, adds per-stream flow control, and can say exactly which requests were not processed (`GOAWAY`); TCP's own head-of-line blocking remains, which **HTTP/3** removes by running over **QUIC**. **Redirects** differ in permanence and in whether the method is kept (`307` and `308` keep it, `303` switches to `GET`, `301` and `302` may).

**HTTP caching** serves a stored response while it is **fresh** (`s-maxage`, `max-age`, `Expires`, or a heuristic), and **validates** it with `ETag`/`If-None-Match` or `Last-Modified`/`If-Modified-Since` when stale, getting a bodiless `304` if nothing changed. `no-cache` means "store but always validate", `no-store` means "do not store", `private` keeps responses out of shared caches, and `Vary` adds request headers to the cache key; personalised responses must be marked `private`, because cookies alone do not stop a shared cache. **Conditional writes** with `If-Match` turn an ETag into an optimistic lock: a stale write gets `412`, provided the server checks and writes atomically. **Cookies** are set with `Set-Cookie` (one per header, with attributes) and returned as plain `name=value` pairs in `Cookie`. Servers enforce **limits and timeouts** and reject malformed messages; clients need explicit connect, read and total timeouts. And when a request times out, its outcome is **unknown**: only idempotent requests, or requests protected by an idempotency key, can be retried safely.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Statelessness | Each request is interpretable on its own; any instance can serve it | That the application has no state; that the transport has no state |
| URI parsing | The same components every time; origin form plus `Host` rebuild the target | Unlimited length; privacy of URLs in logs |
| HTTP/1.1 framing | Many messages share one connection without corruption when both ends parse strictly | That the declared length matches the bytes sent; agreement between two lenient parsers |
| Safe methods | The client requested no state change | That the server's handler honours it; that the request is cheap |
| Idempotent methods | Repeating has the same effect on state as sending once | Identical responses; absence of side effects such as logs |
| Status codes | A classification every client and intermediary understands | The cause; whether effects happened (for `5xx` and timeouts) |
| Headers | An extensible metadata channel | Authenticity of any client-supplied header; unlimited size |
| `Content-Type` | A declared interpretation of the body | That the body actually matches it |
| Content negotiation | The server can pick the best representation it has | That it honours preferences; that caches keep variants apart without `Vary` |
| Persistent connections | No new handshake for later requests | That a pooled connection is still open; parallelism on one HTTP/1.1 connection |
| HTTP/2 | Many concurrent streams per connection; compressed headers; `GOAWAY` names unprocessed streams | Freedom from TCP head-of-line blocking; balanced load behind L4 balancers |
| HTTP/3 | Streams independent of each other's losses; a merged transport and TLS handshake | Freedom from loss; passage through networks that block UDP |
| `307` / `308` | The method and body are repeated | That the client follows redirects at all |
| `Cache-Control: no-store` | Compliant caches do not store the response | Removal of earlier copies; absence from logs |
| `Cache-Control: no-cache` | Validation before every reuse | That the response is not stored |
| `private` | Shared caches do not store the response | Protection if a CDN is configured to override headers |
| Freshness lifetime | Reuse without contacting the origin until it expires | That the data is current |
| ETag + `If-Match` | A write applies only to the version the client read, if checked atomically | Merging of concurrent changes; protection of clients that omit the header |
| Cookies | Matching cookies are returned on matching requests | Integrity or secrecy of the value against the client; delivery by non-browser clients |
| Timeouts | The caller stops waiting | That the server stopped working, or that the operation failed |

---

## Interview Drill

**Q1. Describe an HTTP/1.1 request and response on the wire. How does the receiver find the end of the headers and of the body?**
A strong answer must mention:
- Start line (method, target, version; or version, status, reason), header lines, an empty line, optional body; every line ends in CRLF.
- The header section ends at the first `\r\n\r\n`; the body may contain any bytes.
- The body's end: `Content-Length` bytes, or chunked encoding until the zero-size chunk; some responses never have a body (`HEAD`, `1xx`, `204`, `304`); otherwise a response may end with the connection.
- Header names are case-insensitive; `Host` is required in HTTP/1.1.
*Common weak answer:* "a request has a URL, headers and a JSON body", with no framing. Without framing there is no way to share a connection.

**Q2. What is the difference between `Content-Length` and chunked encoding, and how does request smuggling work?**
A strong answer must mention:
- `Content-Length` declares the byte count in advance; chunked sends hex-sized chunks and ends with a zero chunk, so the sender need not know the size.
- If both are present, `Transfer-Encoding` takes precedence, and the message should be treated as suspicious.
- Smuggling: a front end and a back end on a shared connection frame the same request differently (CL.TE, TE.CL, duplicate lengths); leftover bytes become the start of the next user's request.
- Defences: reject ambiguous requests and close the connection, strict parsers at both ends, HTTP/2 to the origin.
*Common weak answer:* "chunked is for big files", with no mention of framing disagreement as a security problem. Chunking exists for bodies of unknown length, and its clash with `Content-Length` is exactly what attackers exploit.

**Q3. What is the difference between safe and idempotent methods? Why is `DELETE` idempotent if the second call returns `404`?**
A strong answer must mention:
- Safe: the client requests no state change (`GET`, `HEAD`, `OPTIONS`, `TRACE`).
- Idempotent: N identical requests leave the same server state as one (the safe methods plus `PUT` and `DELETE`).
- Idempotency is about state, not the response: after one or two `DELETE`s the resource is gone either way.
- Why it matters: automatic retries after lost responses, prefetching, crawling and caching rely on these properties; `GET` must not change state.
*Common weak answer:* "idempotent means it returns the same response every time." Idempotency is about server state; a repeated `DELETE` legitimately returns a different status.

**Q4. What are the differences between `PUT`, `PATCH` and `POST`?**
A strong answer must mention:
- `PUT`: replace the resource at a client-known URI with the full representation; idempotent; omitted fields are reset.
- `PATCH`: apply a description of changes; idempotent only if the changes are (setting a field yes, incrementing or appending no).
- `POST`: let the resource process the data, typically creating a subordinate resource with a server-chosen ID; not idempotent.
- Consequence: a timed-out `POST` cannot simply be retried; use idempotency keys (Chapter 13).
*Common weak answer:* "`PUT` updates and `POST` creates", which misses replacement semantics and idempotency.

**Q5. What is the difference between `401` and `403`? When would you return `404` instead of `403`?**
A strong answer must mention:
- `401`: missing, invalid or expired authentication; the response must carry `WWW-Authenticate`; the client should authenticate or refresh credentials.
- `403`: the identity is known (or irrelevant) and the action is refused; re-authenticating will not help.
- Returning `404` for resources the caller may not see avoids confirming that they exist (no enumeration).
- Be consistent so clients can act on the code.
*Common weak answer:* treating them as interchangeable "access denied" codes. Clients must act differently on them: re-authenticate on `401`, stop on `403`.

**Q6. Distinguish `400` and `422`, and `500`, `502`, `503`, `504`.**
A strong answer must mention:
- `400`: the request cannot be parsed or is malformed; `422`: parseable but semantically invalid.
- `500`: this server failed unexpectedly.
- `502`: a gateway got an invalid response or no usable response (refused, reset) from upstream; `504`: a gateway got no response in time.
- `503`: deliberately unavailable now (overload, maintenance, no healthy backends), possibly with `Retry-After`.
- None of the `5xx` codes proves the operation did not happen; `504` in particular leaves the outcome unknown.
*Common weak answer:* "5xx means the request failed and is safe to retry." A `500` or `504` can follow a successful commit, so a blind retry can perform the operation twice.

**Q7. What does a `200 OK`, or a timeout, tell you about whether the server's transaction committed?**
A strong answer must mention:
- A `200` from the application means the handler reported success; whether that implies a durable commit depends on the API's contract.
- A `200` from a gateway proves only what the gateway received.
- A timeout or a lost connection means the outcome is unknown: the request may have failed before arriving, during processing, or after committing.
- Safe recovery: retry only idempotent requests; protect others with idempotency keys stored atomically with the change; or query the outcome.
*Common weak answer:* "a timeout means it failed, so retry." The request may have committed before its response was lost.

**Q8. How do `301`, `302`, `303`, `307` and `308` differ for a `POST`?**
A strong answer must mention:
- `301` and `308` are permanent; `302`, `303` and `307` temporary.
- `301` and `302` allow (and browsers perform) a switch from `POST` to `GET`; `303` always switches to `GET`; `307` and `308` must repeat the method and body.
- Post/Redirect/Get uses `303`; moved APIs should use `308` or `307`.
- Permanent redirects are cached by browsers, so a mistaken `301` is hard to undo.
*Common weak answer:* "`301` is permanent and `302` is temporary", nothing about the method. For a `POST`, whether the method and body are kept is the difference that breaks clients.

**Q9. What does keep-alive do? Why does an idle keep-alive connection not need a thread on a modern server?**
A strong answer must mention:
- HTTP/1.1 connections are persistent by default; reuse skips TCP and TLS handshakes and keeps a grown congestion window.
- One exchange at a time per HTTP/1.1 connection; pipelining failed because of in-order responses (HTTP-level head-of-line blocking).
- Connection, request, stream and thread are different lifetimes; non-blocking servers park idle sockets in `epoll` and assign a worker only when a request arrives.
- Idle connections still cost file descriptors and memory, hence idle timeouts; client idle timeouts must be shorter than the server's to avoid the keep-alive race.
*Common weak answer:* confusing HTTP keep-alive with TCP keepalive, or assuming one thread per connection. The first is connection reuse, the second a liveness probe; and non-blocking servers hold idle sockets without any thread.

**Q10. Compare HTTP/1.1, HTTP/2 and HTTP/3. Why does HTTP/2 still suffer from head-of-line blocking?**
A strong answer must mention:
- HTTP/1.1: text framing, one exchange at a time per connection, several connections per origin.
- HTTP/2: binary frames with stream IDs multiplexed over one TCP connection, HPACK header compression, per-stream and per-connection flow control, `RST_STREAM`, `GOAWAY`.
- HTTP/2 removes HTTP-level head-of-line blocking but not TCP's: one lost segment stalls every stream on the connection.
- HTTP/3 maps requests onto QUIC streams (independent loss recovery), uses QPACK, and combines transport and TLS setup; it is discovered via `Alt-Svc` or DNS and falls back to TCP.
*Common weak answer:* "HTTP/2 is faster because it is binary." The gains come from multiplexing and header compression, and under packet loss HTTP/2 can be slower than HTTP/1.1.

**Q11. How does HTTP caching work, and what is the difference between `no-cache` and `no-store`?**
A strong answer must mention:
- Freshness: `s-maxage` (shared caches), `max-age`, `Expires`, or a heuristic; `Age` reports time already spent in caches.
- Validation of stale responses with `If-None-Match` (ETag) or `If-Modified-Since`, returning a bodiless `304` if unchanged.
- `no-cache`: may store, must validate before every reuse; `no-store`: must not store.
- `private` versus `public`; `Vary` adds request headers to the cache key; personalised responses need `private` because cookies do not prevent shared caching.
*Common weak answer:* "`no-cache` disables caching." It allows storage and forces validation before reuse; `no-store` is the directive that forbids storing.

**Q12. How do you stop two clients from overwriting each other's changes over stateless HTTP?**
A strong answer must mention:
- Return a strong `ETag` (a version) with every read.
- Clients send `If-Match: "<etag>"` on `PUT`, `PATCH` or `DELETE`; a mismatch gets `412 Precondition Failed`, and the client re-reads and retries.
- The server must check and write atomically (`UPDATE ... WHERE id = ? AND version = ?`, then check the row count); a separate read-then-write reintroduces the race.
- `428 Precondition Required` can force clients to send the condition; `If-None-Match: *` gives create-only semantics.
*Common weak answer:* "use a database lock", which cannot be held across a user's think time over stateless HTTP.

**Q13. A `POST /payments` times out. Why is retrying dangerous, and how do you make it safe?**
A strong answer must mention:
- The timeout does not say whether the payment happened (failed before, during, or after commit).
- Blind retry risks a double charge; no retry risks an unrecorded charge or a lost order.
- Make it safe with an idempotency key generated before the first attempt and reused on every retry, stored by the server atomically with the payment, so a retry returns the original result.
- Retry with backoff and a bound (Chapter 24); reconcile when the outcome stays unknown.
*Common weak answer:* "retry three times", ignoring duplicates. Every retry of a payment that already succeeded is another charge.

**Q14. Why must `GET` never change state? Give a concrete failure.**
A strong answer must mention:
- Clients and intermediaries treat `GET` as safe: browsers prefetch it, crawlers follow it, caches replay it, client libraries retry it.
- A state-changing `GET` (for example a cancel link) is triggered by any of these without user intent.
- It also bypasses protections designed for state-changing requests (Chapter 19).
- Fix: use `POST` or `DELETE` for actions.
*Common weak answer:* "it is a REST convention", with no mechanism. The danger is concrete: automated agents send `GET`s with no user intent at all.

**Q15. What is the difference between `Content-Encoding` and `Transfer-Encoding`, and why does `Vary` matter for compression?**
A strong answer must mention:
- `Content-Encoding` (gzip, br, zstd) is part of the representation and end to end; `Transfer-Encoding` (chunked) applies to one hop and is forbidden in HTTP/2 and HTTP/3.
- The server chooses a coding from `Accept-Encoding` and should send `Vary: Accept-Encoding`.
- Without `Vary`, a cache can serve a compressed body to a client that cannot decode it.
- Compression trades CPU for bytes and is pointless for already-compressed formats.
*Common weak answer:* treating the two headers as synonyms. One describes the representation end to end, the other one hop's framing; a proxy may change the second but not the first.

**Q16. A client's `Content-Length` is set from a string's length, and some requests hang or corrupt the next request. Why?**
A strong answer must mention:
- `Content-Length` counts bytes; string length counts characters; UTF-8 characters outside ASCII take 2 to 4 bytes.
- A too-small length makes the server stop early and parse the leftover bytes as the next request; a too-large one makes it wait for bytes that never come.
- Evidence: failures only with non-ASCII content; `400` on the following request.
- Fix: encode to bytes first, or let the library set the length.
*Common weak answer:* blaming the server's JSON parser. The parser is handed the wrong number of bytes; the length header is the bug.

**Q17. A team keeps a shopping cart in each instance's memory behind a load balancer. What goes wrong, and what does it have to do with HTTP being stateless?**
A strong answer must mention:
- Stateless HTTP lets the balancer send consecutive requests from one user to different instances.
- In-memory state on one instance is invisible to the others, so carts disappear or flip between versions.
- Sticky sessions hide the problem until an instance restarts or is removed.
- Fix: shared state in a store, or state carried in the request (a signed token).
*Common weak answer:* "enable sticky sessions." That hides the bug until an instance restarts or is removed, and it unbalances load.

---

## Scenario Exercises

1. Choose a status code, and explain what the client should do next, for each: (a) a valid JSON order whose quantity is negative; (b) an expired access token; (c) a valid token for a user who may not see another tenant's invoice; (d) an `If-Match` that no longer matches; (e) a gateway whose upstream did not answer within 30 seconds; (f) a client exceeding its rate limit; (g) an upload of 2 GB to an endpoint that accepts 100 MB.
2. Design `Cache-Control` (and any `ETag` and `Vary`) for: a user's account page; a public product listing that changes a few times an hour; `app.4d9e21.js`; the `index.html` that loads it; and a public API response whose content depends on `Accept-Language`. For each, say which caches may store it and for how long.
3. A mobile app's API moves from `/v1/orders` to `/v2/orders`. The team configures a `301` from the old path to the new one. Older app versions start failing to create orders, while viewing orders still works. Explain why, and the correct redirect.
4. An HTTP/2 API serves mobile users. Its p99 latency is much worse than the HTTP/1.1 version it replaced, although server-side timings are unchanged. Explain the mechanism and two ways to improve it.
5. A checkout service calls `POST /payments` on a provider through a gateway. It receives `504 Gateway Timeout` after 30 seconds. What can it conclude about the payment? What should it do next, and what should have been in place before the request was sent?
6. Two engineers disagree about where a header is being lost: the client team says they send `Idempotency-Key`; the service team says it never arrives. There is a reverse proxy in between. Describe how you would settle it with traces, and what you would look for.
7. Users report that the "account summary" page sometimes shows another person's name. The page has been fine for a year; last week the team added a CDN in front of the site. List what you would check, in order, and the most likely root cause.

---

## Diagnosis Lab

**Lab 1: Another user's account page**
- *Symptom:* A few days after a CDN was introduced, users occasionally see someone else's name and recent orders on `/account/summary`. The problem appears in bursts and cannot be reproduced on demand.
- *Hypotheses:* (a) a session-handling bug in the application that mixes up sessions; (b) the CDN is caching the personalised page and serving it to other users; (c) a browser extension.
- *Evidence:* `curl -v https://shop.example.com/account/summary` with a valid session cookie shows `Cache-Control: public, max-age=60`, added by a global response filter introduced for the product pages, and the CDN's response headers show a cache hit with a non-zero `Age`. The CDN's logs show the URL served from cache to many different client addresses. Application logs show no cross-session reads, which argues against (a).
- *Root cause:* The page was marked publicly cacheable, and the CDN stored the first logged-in user's version and served it to everyone for 60 seconds. A request carrying a `Cookie` header does not stop a shared cache from storing the response (Section 7.12).
- *Fix:* `Cache-Control: private, no-cache` on all personalised responses (and `no-store` where data is sensitive), a CDN rule that bypasses the cache for requests carrying the session cookie and never caches responses with `Set-Cookie`, then a purge of the affected URLs.
- *Prevention:* Apply caching headers per route, not globally; add an automated test that every authenticated route returns `private` or `no-store`; alert on cache hits for authenticated paths.

**Lab 2: Duplicate orders after a configuration change**
- *Symptom:* After a routine change to the reverse proxy's upstream settings, a small number of customers receive two identical orders, created about 30 seconds apart. Most duplicates coincide with slow database moments.
- *Hypotheses:* (a) users double-clicking; (b) the mobile app retrying after its own timeout; (c) the proxy retrying `POST`s on another upstream after a timeout.
- *Evidence:* The duplicate pairs share the same client connection and the same request ID in the proxy log, which rules out (a) and (b): the client sent one request. The proxy log shows the first upstream attempt timing out at 30 seconds and a second attempt to a different instance, which succeeded. The configuration diff shows that the change enabled retrying requests on the next upstream after timeouts *including non-idempotent methods*. The first instance's logs show that the original `POST` did commit, at second 31.
- *Root cause:* The proxy treated a timeout as a failure and retried a non-idempotent request (Section 7.4). The first attempt had not failed; it was slow, and both attempts committed.
- *Fix:* Revert the retry setting so that the proxy retries only idempotent requests (or only failures where the upstream provably did not receive the request, such as a refused connection). Add idempotency keys to order creation so that any retry, from any layer, is harmless (Chapter 13).
- *Prevention:* Review retry settings in every layer (client library, proxy, service mesh) together; count retries per layer in metrics.

**Lab 3: Intermittent `502` errors between the load balancer and the application**
- *Symptom:* A Node.js service behind an L7 load balancer returns `502 Bad Gateway` for about 0.1% of requests, spread evenly through the day, more often at quiet times than at peaks. The application's logs contain no record of the failed requests.
- *Hypotheses:* (a) the application crashes occasionally; (b) a keep-alive race between the balancer's pooled connections and the application's idle timeout; (c) network loss.
- *Evidence:* No restarts or crashes in the process metrics, against (a). The balancer's error log gives the reason "upstream closed connection" or a reset for each `502`. The balancer keeps idle backend connections for 60 seconds. The Node.js server's `keepAliveTimeout` is 5 seconds (its default at the time of writing). A capture on an application host shows the application sending FIN on a connection idle for 5 seconds while, at almost the same instant, the balancer sends a new request on it, which the application's kernel answers with a reset.
- *Root cause:* The backend gave up on idle connections much sooner than the balancer, so the balancer sometimes reused a connection the backend was closing (Section 7.9). Quiet periods produce more idle connections, hence more errors.
- *Fix:* Make the application's idle timeout longer than the balancer's (for example 65 seconds against 60), so that the balancer always closes first.
- *Prevention:* Document idle timeouts for every hop and keep them ordered (each client side shorter than its server side); alert on the balancer's `502` reason codes, not only the count.

**Lab 4: Users stuck on an old version after a release**
- *Symptom:* After a front-end release, a portion of users keep loading the old JavaScript for hours, and some hit errors because the old code calls an API field that was renamed.
- *Hypotheses:* (a) the CDN did not receive the new files; (b) browsers are reusing a cached `index.html` that references the old bundle; (c) a service worker in the old version is serving cached files.
- *Evidence:* The CDN serves the new `index.html` when requested directly with `curl`, against (a). `curl -sI` on `index.html` shows no `Cache-Control` header at all, but a `Last-Modified` date from two weeks before the release. The bundle names contain no content hash (`app.js`), so the new release reused the old file name. No service worker is registered on the site.
- *Root cause:* With no explicit caching directives, browsers applied heuristic freshness based on `Last-Modified` (Section 7.12) and kept the old `index.html` for hours; because the bundle name had not changed, even users who fetched the new HTML could get the old `app.js` from cache.
- *Fix:* Serve `index.html` with `Cache-Control: no-cache` (always revalidate, cheap with an `ETag`); name bundles by content hash and serve them with `public, max-age=31536000, immutable`; keep the old API field working until old clients have aged out.
- *Prevention:* An explicit caching policy for every route type (Section 7.12, Backend connection), checked in the deployment pipeline.

---

## Solution Sketches

1. (a) `422` (or `400` if the API uses it for validation), with a problem-details body naming the field; fix the input, do not retry. (b) `401` with `WWW-Authenticate`; refresh the token or log in again, then retry. (c) `404` (to avoid revealing that the invoice exists) or `403`; do not retry. (d) `412`; re-read the resource, reapply the change to the new version, retry with the new ETag. (e) `504`; the outcome is unknown, so retry only if the operation is idempotent or carries an idempotency key, otherwise check its status. (f) `429` with `Retry-After`; wait at least that long, with backoff. (g) `413`; do not retry the same body; use a smaller upload or a resumable upload flow.
2. Account page: `private, no-cache` plus an `ETag` (browser only, always revalidated), or `no-store` for sensitive details. Product listing: `public, max-age=0, s-maxage=60, stale-while-revalidate=30`, an `ETag`, `Vary: Accept-Encoding`; the CDN keeps it up to a minute (purged on change), browsers revalidate. `app.4d9e21.js`: `public, max-age=31536000, immutable`; any cache, for a year, because a new version gets a new name. `index.html`: `no-cache` with an `ETag`; stored but revalidated on every load, so new releases appear at once. Language-dependent API response: as for the listing, plus `Vary: Accept-Language, Accept-Encoding`, so each language is cached separately.
3. Old clients send `POST /v1/orders`; on `301` many HTTP clients (and all browsers) repeat the request as `GET /v2/orders` without the body, which lists orders instead of creating one (or fails). `GET` requests are unaffected, which is why viewing works. Use `308 Permanent Redirect` (or `307` while testing), which requires the method and body to be repeated; better still, keep `/v1/orders` working for old clients.
4. All of a client's requests share one TCP connection; on lossy mobile links, every lost segment stalls every stream until it is retransmitted (TCP head-of-line blocking), and loss also shrinks the single connection's congestion window. HTTP/1.1 spread requests over several connections, so one loss stalled fewer of them. Improvements: serve HTTP/3 over QUIC at the edge, so a loss stalls only its own stream; move the edge closer to users to shorten RTTs and recovery time; check that congestion control and initial windows on the edge are appropriate.
5. Nothing definite: the gateway gave up waiting, but the provider may have charged the card before, at or after that moment, and may still be processing. Do not blindly retry. If the request carried an idempotency key, retry with the same key (with backoff) and the provider will return the original result or perform it once. Otherwise query the provider for a payment with this order's reference, and reconcile. Beforehand: an idempotency key on every payment request, a client timeout chosen with the provider's latency in mind, and a reconciliation process for outcomes that stay unknown.
6. Capture what leaves the client (`curl -v` with the same headers, or a capture on the client host) and what arrives at the service (a capture on the service host, or temporary logging of raw request headers), for the same request ID. If the header is present before the proxy and absent after, inspect the proxy's configuration: header allow-lists, removal of unknown or "unsafe" headers, normalisation of names with underscores (some proxies drop such names by default), or a size limit. If it is absent on the client side, the client code is wrong.
7. Check: whether the page's responses carry `public` or a long `max-age` (`curl -v` with and without a session cookie); whether the CDN reports cache hits for `/account/summary` and with what `Age`; the CDN's caching rules (does it cache everything with a 200 status? does it ignore `Cookie`?); whether any response with `Set-Cookie` was cached; and whether the timing matches the CDN rollout. The most likely root cause is that the CDN is storing the personalised page (through a public `Cache-Control` or a cache-everything rule) and serving it to other users, as in Diagnosis Lab 1.

---

**END OF CHAPTER 7**
