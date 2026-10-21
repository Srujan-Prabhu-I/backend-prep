# Chapter 10: Inside the Server: From Network Bytes to Application Code

## Where We Are

Chapter 1 drew the path of a request as a few boxes: a client, a load balancer, a reverse proxy, application servers, a database. Since then, each arrow between those boxes has been opened up: the socket calls and kernel queues on each machine (Chapter 4, Section 4.6), the TCP connection and its failure modes (Chapter 6), the HTTP messages and their framing, caching and status codes (Chapter 7), the TLS that protects them (Chapter 8), and the browser that sends most of them (Chapter 9). Chapter 6 (Section 6.13) also introduced the difference between balancing per connection (L4) and per request (L7).

This chapter opens the **boxes**. It follows one request from the edge of your network to the line of code that handles it: through CDNs, firewalls and DDoS protection at the edge; through reverse proxies and load balancers that terminate TLS, route, buffer, rewrite headers and choose a backend; into the server process, where bytes from an accepted socket are parsed, routed and passed through a chain of middleware to a handler; and back out as a response. Along the way it compares the ways servers are built (a thread per connection, worker processes, an event loop, async runtimes), explains which component produces which `5xx` error, covers connections that stay open for hours (WebSockets, server-sent events), and sets out how timeouts at every hop must fit inside each other.

How a server sizes its pools and queues and behaves under overload is Chapter 11. This chapter is the structure; Chapter 11 is the capacity.

**After this chapter you can:**
- explain what a CDN, a web application firewall and DDoS protection do at the edge, and what each cannot do;
- distinguish forward and reverse proxies, list what a reverse proxy does, and handle `X-Forwarded-For` and `Forwarded` without trusting forged values;
- compare load-balancing algorithms and predict their behaviour on uneven workloads, configure active and passive health checks, and explain sticky sessions and connection draining;
- distinguish an API gateway, a load balancer and a service mesh;
- trace a request inside a server: listening socket, accept queue, acceptor, worker or event loop, parser, router, middleware chain, handler, response writer;
- write a middleware chain and explain the onion model, flushing and streaming;
- compare thread-per-connection, worker-process, event-loop and async server architectures, with Tomcat, Spring's `DispatcherServlet`, Gunicorn and Node as examples;
- say which component generates a `500`, `502`, `503` or `504`, and why;
- explain WebSockets, server-sent events and long polling, and their effect on proxies and capacity;
- nest timeouts correctly across client, load balancer, proxy, application and database;
- diagnose a `502`, a `504`, a rejected request body and a missing client IP.

**Interview questions this chapter prepares you for:**
1. What is a CDN, and what does it do for a backend?
2. What is the difference between a forward proxy and a reverse proxy? What does a reverse proxy do?
3. What is the difference between L4 and L7 load balancing?
4. Which load-balancing algorithms do you know, and when does each fail?
5. What are health checks? What is the difference between active and passive checks, and between liveness and readiness?
6. What are sticky sessions, and what do they cost?
7. What is the difference between an API gateway, a load balancer and a service mesh?
8. What happens between `accept()` and your handler code?
9. What is middleware? Explain the filter chain.
10. Compare thread-per-request, worker processes and event-loop servers.
11. Who produces a `502`, a `503` and a `504`?
12. How do WebSockets work? How do they differ from server-sent events and long polling?
13. How should timeouts be set across the hops of a request?
14. How does a server behind a proxy learn the client's real IP address, and how can that go wrong?

**Scope.** Here: the edge, proxies, load balancers, gateways and meshes, the inside of a server process, server architectures, the origin of `5xx` errors, long-lived connections and timeout nesting. Recalled, not re-taught: sockets and kernel queues (Chapter 4, Section 4.6), I/O models and `epoll` (Chapter 4, Section 4.5), event loops and thread pools (Chapter 5, Sections 5.11–5.13), L4 versus L7 (Chapter 6, Section 6.13), HTTP connections, status codes and caching (Chapter 7), TLS termination topologies and mTLS (Chapter 8, Section 8.10), and CORS and security headers (Chapter 9). Sizing pools and queues, Little's Law and overload are Chapter 11. Retries, circuit breakers and rate-limiting algorithms are Chapter 24. Deployments and orchestration are Chapter 26.

---

## 10.1 The Edge: CDNs, WAFs and DDoS Protection

### Definitions

The **edge** is the part of a system that faces the internet: the first components a user's request reaches, usually run by a specialised provider and located close to users.

A **CDN** (content delivery network) is a geographically distributed set of caching reverse proxies (**edge servers**, grouped in **points of presence**, PoPs). Users are routed to a nearby PoP, which serves cached responses directly and forwards cache misses to your servers, called the **origin**.

A **WAF** (web application firewall) is a filter that inspects HTTP requests (L7, unlike the packet filters of Chapter 6, Section 6.5) and blocks those matching attack patterns or policy rules.

**DDoS protection** is the capacity and filtering that keep a service available during a **distributed denial-of-service** attack, in which many machines send traffic at once to exhaust its network, connection or processing capacity.

### Why they exist

Distance costs round trips (Chapter 6, Section 6.12): a user 200 ms from your data centre pays that on every uncached request, several times on a cold connection. Serving from a PoP a few milliseconds away removes most of that, and every cache hit is a request your origin never sees. Attacks scale the same way: a botnet can send more traffic than any single data centre's links can absorb, so the absorbing has to happen at a network that is bigger than the attack and spread across the world.

### Mechanism 1: how a CDN serves a request

```text
 user in Hyderabad ──► DNS (or anycast, Chapter 6, Section 6.15) picks the nearest PoP
                        │
                        ▼
                 ┌──────────────────┐   cache key: method + host + path + (query, Vary headers)
                 │ edge PoP (Hyd.)  │── HIT (fresh) ─► respond in a few ms; origin sees nothing
                 └──────┬───────────┘
                        │ MISS or stale
                        ▼
                 ┌──────────────────┐   ORIGIN SHIELD: one designated PoP in front of the origin;
                 │ shield PoP (Mum.)│   other PoPs fetch through it, so a miss in 50 PoPs becomes
                 └──────┬───────────┘   one request to the origin, not 50
                        │ MISS
                        ▼
                 your origin (load balancer → app), over a warm, pooled connection
```

- **Caching follows HTTP.** The CDN is a shared cache in the sense of Chapter 7 (Section 7.12): `s-maxage`, `public`, `private`, `Vary` and validators decide what it stores and for how long. Most CDNs also let you override that with their own rules, which is powerful and dangerous (Chapter 7's Diagnosis Lab 1).
- **Purging.** When content changes before its freshness ends, the origin asks the CDN to **purge** (invalidate) URLs or tags. Purges take time to reach every PoP; design for a short period where both versions are served.
- **Not only cache hits help.** Even uncacheable requests benefit: the PoP terminates TLS close to the user (saving round trips on the cold handshake) and forwards over long-lived, warm connections to the origin.

### Mechanism 2: WAF and DDoS protection

**A WAF** evaluates each request against rules: signatures of known attacks (patterns of SQL injection or XSS payloads, Chapters 18 and 9), protocol violations, known-bad sources, and custom rules ("block requests to `/admin` from outside our office range"). Matches are blocked or logged. Rules are generic and therefore produce **false positives** (blocking a legitimate request whose JSON happens to look like an attack) and **false negatives** (missing a payload written slightly differently). A WAF reduces exposure; it does not replace fixing vulnerabilities.

**DDoS protection** works at two levels:

| Attack layer | Example | Defence |
|---|---|---|
| Network and transport (L3/L4): volume | huge floods of UDP or SYN packets, often amplified (Chapter 6, Section 6.10) | absorb with a network larger than the attack (often anycast, so the flood is split across many PoPs); filter at the edge; SYN cookies (Chapter 6, Section 6.8) |
| Application (L7): cost | many valid-looking HTTP requests to expensive endpoints (search, login) | rate limits per client or key (Chapter 24), challenges to tell humans from bots, caching, and making expensive endpoints cheaper |

### Guarantees and non-guarantees

A CDN guarantees faster delivery of what it may cache and less load on the origin; it does not make uncacheable responses fast, does not stop you from caching what you should not, and depends on correct headers. A WAF guarantees that requests matching its rules are blocked; it does not guarantee that attacks are caught or that legitimate requests are allowed. DDoS protection keeps the service reachable during attacks within its capacity; it does not make an inefficient endpoint cheap.

### Failure and diagnosis

- **Origin overload after a purge.** A global purge empties every PoP at once; all of them miss together and the origin receives the full traffic for a while (a **thundering herd** at the cache, Chapter 21). Evidence: origin request rate jumps right after the purge. Fixes: an origin shield, purging by URL or tag rather than everything, serving stale content while revalidating.
- **WAF false positive.** A legitimate form submission is blocked with a `403` from the edge, not from your application. Evidence: the response comes from the edge (its `Server` header, its error page); your application logs nothing; the WAF log names the rule. Fix: tune or exclude that rule for that route, with care.
- **Real client IPs lost.** The origin sees only CDN addresses. Section 10.2 covers how to recover them safely.

### Backend connection

Design responses for the edge: correct caching headers on everything, URLs that make cache keys predictable, a purge strategy, and endpoints whose cost does not let a few clients exhaust you. Treat the CDN's cache-override rules as code that needs review, because a wrong rule can serve private data to strangers.

---

## 10.2 Forward and Reverse Proxies

### Definitions

A **proxy** is an intermediary that receives requests and sends them on. A **forward proxy** acts **for clients**: clients are configured to use it to reach any server (a corporate egress proxy, a content filter). A **reverse proxy** acts **for servers**: clients connect to it as if it were the server, and it forwards requests to one or more backend servers that clients never see directly (Chapter 7, Section 7.1 introduced both roles).

### Why reverse proxies exist

Application servers are good at running application code and poor at the edge jobs around it: terminating TLS efficiently for thousands of connections, protecting slow worker pools from slow clients, compressing, enforcing request limits, routing to the right service, and presenting one address for many backends. Putting those jobs in one well-tested component (Nginx, HAProxy, Envoy, a cloud load balancer) keeps them consistent and out of every application.

### Mechanism: what a reverse proxy does

```text
 client ══TLS══► [ REVERSE PROXY ] ──(plain or re-encrypted HTTP)──► app instances
                  1. terminate TLS (certificates live here; Chapter 8, Section 8.10)
                  2. read and check the request: size limits, timeouts, malformed framing → 400/413/431
                  3. route by host and path: api.example.com/orders → orders pool
                  4. buffer: read the whole request body before forwarding; read the whole
                     response quickly from the app and feed it to the slow client itself
                  5. compress responses (gzip/br) if the app did not
                  6. rate-limit and reject abusive clients (algorithms: Chapter 24)
                  7. rewrite headers: add X-Forwarded-For/-Proto/-Host or Forwarded, add a request
                     ID, remove hop-by-hop and internal headers
                  8. choose a backend (load balancing, Section 10.3) over pooled keep-alive connections
```

**Buffering deserves attention.** A client on a slow mobile link may take 10 seconds to upload a 2 MB body. If the proxy forwarded bytes as they arrived, an application worker would be tied up for those 10 seconds doing nothing. With **request buffering**, the proxy collects the whole body first and hands it to the application in milliseconds. **Response buffering** works the other way: the application writes its response at full speed to the proxy and is freed, and the proxy trickles it to the slow client. This also blunts slow-client attacks (Chapter 7, Section 7.15). The cost: buffering adds latency for large bodies, uses proxy memory or disk, and breaks streaming responses (Section 10.9), so it is turned off for routes that stream.

### Concrete: the full request path, with TLS and headers

```text
 browser 203.0.113.7
    │  TLS session 1 (certificate for shop.example.com, ALPN h2)
    ▼
 CDN edge PoP  198.51.100.20     ── TLS 1 terminates here; cache lookup (§10.1)
    │  adds: X-Forwarded-For: 203.0.113.7   X-Forwarded-Proto: https   (+ its own request ID)
    │  TLS session 2 (CDN → origin, re-encrypted, certificate for origin.example.com)
    ▼
 load balancer 10.0.0.5           ── TLS 2 terminates here; chooses a proxy/app pool (§10.3)
    │  appends: X-Forwarded-For: 203.0.113.7, 198.51.100.20
    │  plain HTTP or TLS session 3 inside the private network (Chapter 8, Section 8.10)
    ▼
 reverse proxy / gateway          ── routing, limits, buffering, rate limits
    │  adds: X-Request-Id: 7f3a... (if absent)   strips: client-supplied internal headers
    ▼
 app instance 10.0.3.7:8080       ── TCP peer = proxy; client identity only from trusted headers
```

Each arrow is a separate TCP (and possibly TLS) connection, usually pooled and kept alive between the layers, with its own timeouts (Section 10.10).

### Mechanism: forwarding headers and the client's address

After the proxy, the application's TCP peer is the proxy (Chapter 6, Section 6.13), so the client's address, the original scheme (`https`) and the original `Host` are lost unless the proxy passes them on in headers:

```http
X-Forwarded-For: 203.0.113.7                 client address (each proxy appends the address it received from)
X-Forwarded-Proto: https                     scheme the client used
X-Forwarded-Host: shop.example.com           host the client asked for
Forwarded: for=203.0.113.7;proto=https;host=shop.example.com    the standard form (RFC 7239)
```

With several proxies in a row, each **appends** the address of the peer it received the connection from:

```text
 client 203.0.113.7 ──► CDN edge 198.51.100.20 ──► load balancer 10.0.0.5 ──► app
 header as it reaches the app:
   X-Forwarded-For: 203.0.113.7, 198.51.100.20
   (and the app's own TCP peer is 10.0.0.5)
```

**The trust problem.** The header is just text in the request, and the **first** value came from the client. A client can send its own:

```text
 attacker 192.0.2.66 sends:   X-Forwarded-For: 127.0.0.1
 after the CDN and LB append: X-Forwarded-For: 127.0.0.1, 192.0.2.66, 198.51.100.20
```

An application that takes the **leftmost** value believes the request came from `127.0.0.1`: it may skip rate limits, allow access to "internal-only" endpoints, or write a forged address into audit logs. The correct procedure:

1. Configure the list of **trusted proxies** (your load balancer's and CDN's address ranges).
2. Start from the TCP peer address. If it is not a trusted proxy, it is the client; ignore the header entirely.
3. Otherwise, read `X-Forwarded-For` **from right to left**, skipping addresses that belong to trusted proxies. The **first untrusted address** is the client.

In the example: peer `10.0.0.5` is trusted; from the right, `198.51.100.20` is the trusted CDN; the next, `192.0.2.66`, is not trusted, so that is the client. The forged `127.0.0.1` is never reached. The simplest robust alternative is to have the **outermost** trusted proxy **overwrite** the header with the address it actually saw, and make every inner component trust only that one header from only that proxy. The same applies to `X-Forwarded-Proto` and `X-Forwarded-Host`: accept them only from trusted proxies.

For L4 load balancers, which do not parse HTTP, the **PROXY protocol** prepends a small header with the client's address to the TCP stream (Chapter 6, Section 6.13); the receiving server must be configured to expect it, and only from the balancer.

> **Framework lens:** Most frameworks have a setting for this: Spring Boot's `server.forward-headers-strategy` (with Tomcat's `RemoteIpValve` and its internal-proxies setting), Express's `trust proxy`, Nginx's `real_ip` module with `set_real_ip_from`. Each needs the list of trusted proxies; leaving it unset either ignores the headers (every user appears to be the load balancer) or trusts them from anyone.

### Guarantees and non-guarantees

A reverse proxy guarantees one controlled entry point, where limits, TLS and routing are applied consistently. It does not guarantee anything about headers that arrive from outside: forwarding headers are trustworthy only when you know which component wrote them. It also adds a hop, and a component whose failure affects everything behind it, so it must itself be redundant.

### Failure and diagnosis

- **Every user has the same IP.** Rate limits trip for everyone at once, or logs show only the load balancer's address. The application is reading the TCP peer address and ignoring forwarding headers. Diagnosis Lab 4 covers it.
- **Spoofed client addresses.** Audit logs show requests "from" `127.0.0.1` or internal ranges that arrived from the internet. The application trusts the leftmost `X-Forwarded-For` value. Fix: the right-to-left procedure, with a trusted-proxy list.
- **Streaming responses arrive all at once.** Server-sent events or a progress stream reach the browser only at the end. Response buffering at the proxy. Fix: disable buffering for that route (Section 10.9).

### Backend connection

Every server you write runs behind at least one reverse proxy. Decide which proxy owns TLS, limits, compression and forwarding headers; configure the application to trust forwarding headers only from that proxy; and remember that the proxy's limits and timeouts (Section 10.10) are part of your API's behaviour.

---

## 10.3 Load Balancers: Algorithms, Health Checks, Stickiness, Draining

### Definitions

A **load balancer** distributes incoming connections or requests across a **pool** of backend instances (also called **targets** or **upstreams**). Chapter 6 (Section 6.13) defined **L4** balancers, which choose per connection from addresses and ports, and **L7** balancers, which terminate HTTP and choose per request.

A **load-balancing algorithm** is the rule for choosing a backend. A **health check** is how the balancer decides which backends may receive traffic. A **sticky session** (session affinity) is a rule that sends a given client's requests to the same backend. **Connection draining** (deregistration delay) is letting a backend that is being removed finish its in-flight work before it is cut off.

### Why they exist

One instance has finite capacity, and instances fail. A balancer turns many instances into one service: it spreads load so that no instance is overwhelmed while others idle, routes around failures, and lets instances be added and removed (for scaling and deployments) without clients noticing.

### Mechanism 1: the algorithms

| Algorithm | Rule | Good at | Fails when |
|---|---|---|---|
| **Round robin** | backends in turn: 1, 2, 3, 1, 2, 3 | identical backends, uniform requests | request costs vary, or backends differ in power |
| **Weighted round robin** | in turn, in proportion to weights (4:1 for a bigger machine) | backends of different sizes | costs vary; weights go stale |
| **Least connections** (least outstanding requests) | the backend with the fewest in-flight requests | uneven request costs; slow backends naturally get less | the count is not a good proxy for load (long idle WebSockets count as connections); several balancers acting on stale counts herd to the same backend |
| **Least response time** (often an exponentially weighted average of latency, combined with in-flight count) | the backend that has recently been fastest | backends with different speeds | it reacts to noise; a backend that fails fast looks "fastest" |
| **Random** | a random backend | simple, no shared state across balancer nodes | small pools see uneven load by chance |
| **Power of two choices** | pick **two** backends at random, send to the less loaded | near least-connections quality with little coordination; avoids herding | rarely; it is a strong default |
| **Hashing** (of client IP, a header, a cookie, the URL) | the same key always maps to the same backend | affinity: caches local to a backend, sticky clients | uneven keys (one huge customer); adding or removing a backend remaps keys |
| **Consistent hashing** | keys and backends placed on a ring; a key goes to the next backend clockwise, with many virtual points per backend | affinity that survives pool changes: removing one backend remaps only its share of keys | hot keys still overload one backend (Chapter 23) |

**Why "power of two choices" works.** Least connections needs an accurate, global view of load; many balancer nodes each acting on slightly stale counts all pick the same "least loaded" backend at the same moment and overload it. Pure random needs no view but lets queues grow unevenly. Sampling just two and taking the better one keeps nearly all the benefit of least connections (the longest queue grows far more slowly than with random choice) while two balancer nodes rarely sample the same pair.

### Concrete: round robin versus least connections with uneven costs

Two identical backends, A and B, each with **4 workers**. Requests arrive every **100 ms**, alternating **heavy** (900 ms of work) and **light** (10 ms).

**Round robin** sends request 1 to A, 2 to B, 3 to A, and so on. Every heavy request lands on A, every light one on B:

```text
 A receives a heavy request every 200 ms, each lasting 900 ms
   → concurrent heavy requests on A ≈ 900 / 200 = 4.5   > 4 workers
   → A's queue grows without bound; its latency climbs every second
 B receives a light request every 200 ms, each lasting 10 ms
   → B is busy 10 / 200 = 5% of the time, nearly idle
```

**Least connections** sends each request to whichever backend has fewer requests in flight. A light request finishes almost at once, so the counts are driven by the heavy requests, and new heavy requests go to whichever backend has fewer of them:

```text
 heavy requests now split roughly evenly: each backend gets one every ~400 ms
   → concurrent heavy requests per backend ≈ 900 / 400 = 2.25   < 4 workers
   → both backends stable; no queue builds
```

Same hardware, same traffic: one algorithm produces an overloaded backend next to an idle one, the other a healthy system. The lesson is that **round robin balances request counts, not load**. Real traffic rarely alternates so neatly, but mixtures of cheap and expensive endpoints are normal.

### Mechanism 2: health checks

- **Active checks.** The balancer requests a health endpoint (`GET /healthz`) on each backend every few seconds. After a configured number of consecutive failures (say 3) it marks the backend unhealthy and stops sending traffic; after a number of successes it restores it. Thresholds trade speed of detection against flapping.
- **Passive checks** (outlier detection). The balancer watches real traffic: a backend that returns `5xx` errors or times out on several requests in a row is ejected for a while. This catches failures that the health endpoint does not reveal.

What should the health endpoint check?

- A **liveness** check answers "is this process able to run at all?" and should be cheap and local. If it fails, the process is restarted (Chapter 26).
- A **readiness** check answers "should this instance receive traffic now?": it fails while starting up, while warming caches, while draining for shutdown, or when the instance cannot reach something it absolutely needs.

The classic mistake is a **deep** health check that calls the database and every dependency. When the shared database has a brief problem, every instance fails its check at once, the balancer removes all of them, and a partial degradation becomes a total outage. Keep health checks about the instance itself; handle dependency failures in the request path (Chapter 24).

### Mechanism 3: sticky sessions and their costs

Stickiness pins a client to one backend, using a cookie the balancer sets (`AWSALB=...`, or one of its own), a hash of the client IP, or an application cookie. It exists mainly to support applications that keep per-user state in memory (Chapter 7's Section 7.1 failure). Its costs:

- **Uneven load**: some backends accumulate heavy users; new backends get no existing sessions.
- **Fragility**: when a backend dies or is replaced, its users lose their in-memory state anyway.
- **Slower scaling and deployments**: draining takes as long as the stickiness lasts.

Prefer stateless instances with shared session storage (Chapter 19); use stickiness only for performance (a warm local cache), never for correctness.

### Mechanism 4: connection draining and slow start

When a backend is removed (scale-in, deployment), the balancer stops sending it **new** requests but lets **in-flight** requests finish, for up to a **draining timeout** (often tens of seconds). The backend, told to shut down (Chapter 4, Section 4.11), should stop accepting, finish what it has, close idle keep-alive connections, and exit. Without draining, every deployment cuts off requests mid-flight.

The reverse is **slow start**: a newly added backend is given a small share of traffic at first, rising over a minute or two, so that a cold instance (empty caches, an unwarmed JIT, Chapter 2, Section 2.5) is not hit with full load immediately.

### Guarantees and non-guarantees

A balancer guarantees that traffic is spread by its algorithm among the backends it considers healthy. It does not guarantee even **load** (only even counts, for round robin), does not detect every failure (a backend can pass its health check and still fail requests), does not preserve in-flight work through abrupt removals without draining, and is itself a component that must be redundant.

### Failure and diagnosis

- **One hot instance.** CPU at 100% on one backend, others idle. Causes: round robin on uneven requests, a hot key under hashing, a long-lived HTTP/2 or WebSocket connection pinned to one backend (Chapter 6, Section 6.13), or stickiness. Evidence: per-backend request rate, latency and in-flight counts on the balancer.
- **Total outage from a shared dependency blip.** Deep health checks removed every backend. Evidence: the balancer reports zero healthy targets exactly while the dependency hiccuped, and returns `503`s itself (Section 10.8). Fix: local readiness checks.
- **Errors during every deployment.** Draining missing or shorter than the longest request, or instances that exit on `SIGTERM` without finishing in-flight requests. Evidence: `502`s and resets clustered at deploy times.

### Backend connection

Choose least-outstanding-requests or power-of-two-choices for HTTP services with mixed endpoints; use hashing only when you need affinity, and consistent hashing when the pool changes often. Make readiness reflect only what the instance itself can do, support graceful shutdown that matches the balancer's draining time, and keep instances stateless so that stickiness is never required.

---

## 10.4 API Gateways, Load Balancers and Service Meshes

### Definitions

An **API gateway** is a reverse proxy specialised for APIs: a single entry point for external clients that, besides routing, handles cross-cutting **API** concerns: authentication of API keys or tokens, per-client rate limits and quotas, request and response transformation, versioned routing, and sometimes aggregation of several backend calls into one response.

A **service mesh** is an infrastructure layer for **service-to-service** traffic inside a system: a proxy (a **sidecar**) runs next to every service instance and handles its network traffic, configured centrally by a **control plane**. The mesh provides mTLS between services (Chapter 8, Section 8.10), per-request load balancing, retries and timeouts, and uniform telemetry, without changes to application code.

### Why each exists

All three sit on the request path and overlap, so the question is what each is *for*. A load balancer exists to spread traffic and survive instance failure. A gateway exists because public APIs need the same policies (who is calling, how much they may call, which version they get) on every endpoint, enforced in one place. A mesh exists because, with dozens of services calling each other, every team would otherwise reimplement mTLS, retries, timeouts and metrics in every client library and language.

### Mechanism: where each sits

```text
                         NORTH–SOUTH (clients ↔ system)               EAST–WEST (service ↔ service)
 internet clients ──► [ CDN ] ──► [ LOAD BALANCER ] ──► [ API GATEWAY ] ──► orders ──sidecar══mTLS══sidecar──► payments
                                   spread load,          authN of keys,            │                            │
                                   health checks,        quotas, routing           └─sidecar══mTLS══sidecar──► inventory
                                   TLS termination       by API and version
                                                                                    mesh: identity, retries,
                                                                                    timeouts, per-request LB,
                                                                                    telemetry for every call
```

| | Load balancer | API gateway | Service mesh |
|---|---|---|---|
| Main job | distribute traffic, survive failures | apply API policy at the entry point | secure and manage service-to-service calls |
| Traffic | north–south (and internal pools) | north–south | east–west |
| Typical features | L4/L7 balancing, health checks, TLS | API keys and tokens, rate limits and quotas, transformations, routing by path and version | mTLS identity, retries, timeouts, circuit breaking, per-request balancing, metrics and traces |
| Where it runs | a dedicated tier | a dedicated tier | a proxy beside every instance, plus a control plane |
| Cost | low | one more hop and a policy store | a proxy per instance (memory, CPU, latency per hop), operational complexity |

In practice the lines blur: many L7 load balancers do some gateway work, and gateways are often built from the same proxy software as meshes.

### Guarantees and non-guarantees

A gateway guarantees that its policies apply to every request that passes through it; it does not protect services that can be reached without passing through it, so internal services must not be exposed around it. A mesh guarantees consistent mTLS and policies for traffic through its proxies; it does not remove the need for the application to authorise requests (Chapter 20), and retries added in a mesh on top of retries in the application multiply load during failures (Chapter 24).

### Failure and diagnosis

- **Gateway as a single point of failure or bottleneck.** All API traffic queues behind an under-provisioned gateway, or one bad configuration push breaks every API. Fix: redundancy, gradual configuration rollout, and capacity planning like any tier.
- **Retry storms through layers.** The client retries 3 times, the gateway 3 times, the mesh 3 times: one failing request becomes up to 27 calls to an already struggling service. Evidence: request rates to the failing service multiply during the incident. Fix: retry at one layer, with budgets (Chapter 24).

### Backend connection

For a small system, one L7 load balancer or reverse proxy is enough. Add a gateway when you expose APIs to third parties and need keys, quotas and versioning in one place; add a mesh when the number of services makes per-service mTLS, timeouts and telemetry unmanageable. Each one is another hop with its own timeouts and error codes (Sections 10.8 and 10.10).

---

## 10.5 The Server Process: From Listening Socket to Parsed Request

### Definition

An **HTTP server** (or application server) is a process that listens on a port, accepts TCP connections, reads and parses HTTP requests from them, dispatches each request to application code, and writes the responses back (Chapter 1, Section 1.5 gave the working definition). Its **front end** (sockets, parsing, connection management) is usually provided by a library or container; the **application** is your handlers.

### Why the internals matter

Most production performance problems in a server live in the gaps between these stages: connections waiting in a kernel queue that no metric shows, requests waiting for a free worker, parsing limits that reject requests before your code sees them, idle keep-alive connections consuming slots. Knowing the stages tells you where to look.

### Mechanism: the path of a request inside the process

```text
 kernel                                   server process
 ┌───────────────────────────┐
 │ listening socket :8080    │
 │  SYN queue → ACCEPT QUEUE │ (handshakes completed by the kernel: Chapter 4, Section 4.6)
 └────────────┬──────────────┘
              │ accept()
              ▼
 ┌─────────────────────────────┐    a dedicated ACCEPTOR thread (or the event loop itself)
 │ ACCEPTOR                    │    takes connections off the accept queue and registers
 └────────────┬────────────────┘    each new socket with the I/O mechanism
              ▼
 ┌─────────────────────────────┐    idle connections wait here costing no thread: the socket
 │ POLLER / EVENT LOOP (epoll) │    sits in an epoll set until bytes arrive
 └────────────┬────────────────┘    (Chapter 4, Section 4.5)
              │ "socket readable"
              ▼
 ┌─────────────────────────────┐    reads bytes into a buffer; finds the request line and
 │ PARSER (HTTP/1.1 or /2)     │    headers (Chapter 7, Section 7.3); enforces limits: header
 └────────────┬────────────────┘    size, URI length, body size, read timeouts → 400/413/414/431/408
              ▼
 ┌─────────────────────────────┐    a WORKER (a pool thread, a virtual thread, or a callback
 │ WORKER                      │    on the event loop) takes the request object
 └────────────┬────────────────┘
              ▼
   ROUTER → MIDDLEWARE CHAIN → HANDLER → RESPONSE WRITER        (Section 10.6)
              │
              ▼
   response bytes → socket send buffer → client; connection returns to the poller (keep-alive)
```

Stage by stage:

1. **Listening and the accept queue.** The kernel completes handshakes and queues established connections even when the application is busy (Chapter 4, Section 4.6). The queue's length (the `listen` backlog, capped by `somaxconn` on Linux) is the first buffer in the system, and the only one invisible to application metrics.
2. **Accepting.** One or a few acceptor threads loop on `accept()`. Several processes can share the load with `SO_REUSEPORT`, each with its own queue (Chapter 6, Section 6.11).
3. **Waiting for data.** A new connection, and every kept-alive connection between requests, waits in the poller. Only when bytes arrive does it need any processing.
4. **Parsing and limits.** The parser accumulates bytes until it has a complete header section (and a body, per the framing rules), rejecting anything malformed or over the limits. A request that is rejected here never reaches your code or your application logs; only the server's access log, if any, records it.
5. **Handing to a worker.** In a thread-pool server, the parsed request is queued for a worker thread; if all workers are busy, it waits in that queue (the second buffer). In an event-loop server, the handler runs on the loop thread itself.
6. **Keep-alive.** After the response, the connection returns to the poller for the next request, until the client or the server's idle timeout closes it (Chapter 7, Section 7.9). HTTP/2 connections carry many requests at once; the server tracks streams per connection and limits how many can be open.

### Concrete: the queues a request can wait in

```text
 client ─► [LB queue] ─► kernel accept queue ─► server's "connection limit" ─► worker queue ─► worker
             (Ch 6)         (backlog)              (max connections)            (pool queue)
 a request that takes 3 s end to end may spend 2.9 s waiting in these queues and 0.1 s in your code
```

Each queue has a limit and a behaviour when full: the accept queue drops handshake completions (clients time out, Chapter 4, Section 4.6); a connection limit stops accepting (connections wait in the kernel); a full worker queue rejects requests (often with `503`) or blocks. Chapter 11 sizes them.

### Guarantees and non-guarantees

The server guarantees that requests passing its parser are well-formed and within limits, and that each reaches exactly one handler invocation. It does not guarantee that a client whose connection is accepted will be served promptly (queues), that a request rejected by the parser is logged by your application, or that a client is still waiting when its request finally reaches a worker.

### Failure and diagnosis

- **High latency with idle CPU and fast handlers.** Requests wait in a queue before the handler: workers all blocked on a slow dependency, or the accept queue full. Evidence: handler timing (in traces) is small while total latency (at the load balancer) is large; `ss -ltn` shows the accept queue at its limit (Chapter 4, Section 4.6); thread dumps show all workers waiting on the same call (Chapter 5, Section 5.14).
- **Requests rejected before the application sees them.** Clients get `431` or `413`, and the application logs nothing. Evidence: the proxy's or server's access log has the status; the application log does not. Fix: raise the specific limit if the requests are legitimate, or fix the client (oversized cookies, uploads that should go elsewhere).

### Backend connection

Instrument the stages, not just the handler: queue time before a worker picks the request up, handler time, and total time at the load balancer. The difference between them is where requests wait, and waiting is the most common cause of latency that "isn't in our code".

---

## 10.6 Dispatch: Routing, Middleware, Handlers, Response Writing

### Definitions

**Routing** maps a request (method and path, sometimes host and headers) to a **handler**, extracting path parameters on the way (`/orders/{id}`). **Middleware** (also called **filters** or **interceptors**) is code that runs around every handler, or a group of them, for cross-cutting concerns: logging, authentication, compression, error mapping, timing. The **response writer** sends the status line, headers and body back to the client, and can **flush** (push buffered bytes to the socket now) or **stream** (send the body in pieces as it is produced).

### Why the pattern exists

Every request needs the same things done around the business logic: a request ID assigned, the caller authenticated, the start time recorded, exceptions turned into proper error responses, the response compressed and logged. Writing these into every handler duplicates them and gets them inconsistent. A chain of middleware applies them uniformly, in a defined order, and lets each concern be written once.

### Mechanism 1: routing

```text
 route table                                   request                       result
 GET    /orders            → listOrders        GET /orders/42               → getOrder(id = "42")
 POST   /orders            → createOrder       DELETE /orders/42            → 405 Method Not Allowed
 GET    /orders/{id}       → getOrder                                           (path matches, method does
 GET    /health            → health                                             not; Allow: GET)
                                               GET /orderz                  → 404 Not Found
```

Routers typically match the most specific path first, return `404` when no path matches and `405` (with an `Allow` header) when the path matches but the method does not (Chapter 7, Section 7.5).

### Mechanism 2: the middleware chain (the onion model)

Each middleware receives the request and a function, `next`, that runs the rest of the chain. Code before `next` runs on the way **in**, code after it on the way **out**, so the chain wraps the handler like the layers of an onion:

```pseudo
function requestId(req, next):
    req.id = req.header("X-Request-Id") or newRandomId()
    res = next(req)                           // everything inside runs here
    res.header("X-Request-Id", req.id)
    return res

function timing(req, next):
    start = now()
    res = next(req)
    log.info(req.method, req.path, res.status, now() - start, req.id)
    return res

function errors(req, next):
    try:
        return next(req)
    catch ValidationError as e:
        return problem(422, e)                // RFC 9457 body (Chapter 7, Section 7.5)
    catch any as e:
        log.error(e, req.id)
        return problem(500, "internal error") // never leak stack traces to clients

function authenticate(req, next):
    user = verify(req.header("Authorization"))   // Chapter 20
    if user is null: return response(401)        // short-circuit: the handler never runs
    req.user = user
    return next(req)

chain = [requestId, timing, errors, authenticate]      // order matters
handle(req) = requestId(req, r1 => timing(r1, r2 => errors(r2, r3 => authenticate(r3, router))))
```

```text
 request ──► requestId ──► timing ──► errors ──► authenticate ──► router ──► handler
 response ◄─ requestId ◄── timing ◄── errors ◄── authenticate ◄───────────── handler
```

Order is a design decision. Here, `timing` sits outside `errors`, so it logs the final status even for failed requests; `authenticate` sits inside `errors`, so a crash in authentication still produces a clean `500`; `requestId` is outermost, so every log line and every response, including errors, carries the ID. Middleware can **short-circuit** (authentication returning `401` without calling `next`).

### Mechanism 3: invoking the handler and writing the response

1. **Binding.** The framework converts the request into the handler's inputs: path and query parameters (decoded, Chapter 7, Section 7.2), headers, and the body **deserialised** from JSON or another format according to `Content-Type` (Chapter 7, Section 7.7). Malformed input becomes `400` or `415`; invalid content becomes `422`.
2. **Running the handler**, which calls services, databases and other APIs.
3. **Serialising** the result and setting status and headers.
4. **Writing.** The response writer buffers output. When the buffer fills, or when the handler **flushes**, the status line and headers are written to the socket: the response is **committed**. From that moment the status cannot change. An exception thrown after a partial body has been sent cannot become a `500`; the server can only abort the connection (with HTTP/1.1, the client sees a truncated body or a reset; with HTTP/2, a `RST_STREAM`).
5. **Streaming.** For large or open-ended responses (exports, server-sent events, Section 10.9), the handler writes and flushes in pieces; HTTP/1.1 carries them as chunks, HTTP/2 as `DATA` frames (Chapter 7, Sections 7.3 and 7.10). The socket's send buffer and TCP flow control (Chapter 6, Section 6.8) provide **backpressure**: when a slow client stops reading, writes block (or, in non-blocking servers, the framework stops pulling data), so a well-written streaming handler does not buffer the whole result in memory.

### Guarantees and non-guarantees

The chain guarantees that every request passes the same middleware in the same order, and that short-circuiting middleware stops the handler from running. It does not guarantee that middleware sees every failure (a failure before routing, in the parser, never enters it), that a response can be changed after it is committed, or that a streaming response reaches a client that disconnected.

### Failure and diagnosis

- **Missing logs or request IDs on errors.** The logging or ID middleware sits inside the error handler, or a crash happens in a filter that runs before it. Evidence: `500`s in the proxy's log with no matching application log line. Fix: put request-ID and logging middleware outermost.
- **"The status is 200 but the body is cut off."** The handler started streaming, then failed. Evidence: truncated bodies or `RST_STREAM` / connection resets in the client's log; an exception in the server log after the response started. Fix: validate and fetch what can fail before writing the first byte; for streams, define an in-band error message.
- **Memory spikes on large exports.** The handler builds the whole result in memory before writing. Fix: stream from the database cursor to the response, flushing in chunks.

### Backend connection

Put cross-cutting concerns in middleware, ordered deliberately: request ID and logging outermost, then error mapping, then authentication, then the rest. Validate early, commit late, and stream large results with backpressure instead of buffering them.

---

## 10.7 Server Architectures

### Definition

A **server architecture** (execution model) is how a server maps connections and requests onto operating-system processes and threads (Chapter 3), and therefore how it waits for I/O. Chapter 5 (Section 5.13) compared the execution models in general; this section shows how real servers use them.

### Why there are several

A request handler spends most of its time waiting: for the client's bytes, for the database, for another service. Each architecture answers the same question differently: **who pays for the waiting?** A thread, a process, or nothing (an event loop). The answer decides memory per connection, how many connections a server can hold, what happens when one handler blocks or crashes, and how much of the machine's CPU is used.

### Mechanism: the four families

**1. Thread per connection (or per request), blocking I/O.**

```text
 acceptor ─► connection 1 ─► thread T1: read request, run handler (blocks on DB), write, wait for next
             connection 2 ─► thread T2: ...
             (one thread is tied up for the whole life of a connection, idle or not)
```

The classic Java servlet container model, and the simplest to program: handler code is ordinary sequential blocking code. Each thread costs a stack (typically hundreds of KB to 1 MB) and scheduling work (Chapter 3, Section 3.11), so a server holds thousands of connections at most. Most modern servers keep the simple handler model but free threads during **idle** time: an NIO poller holds idle keep-alive connections, and a pool thread is assigned only when a request arrives (**thread per request**). A handler that blocks on the database still holds its thread for that time.

**2. Worker processes (pre-fork).**

```text
 master process ─ fork ─► worker 1 (handles one request at a time, or a few threads)
                ─ fork ─► worker 2
                ─ fork ─► worker N        workers share the listening socket (or each has one
                                          with SO_REUSEPORT); the master restarts dead workers
```

Used by Gunicorn for Python, PHP-FPM, and Apache's prefork model. Each worker is an isolated process (Chapter 3, Section 3.9), so a crash or memory leak affects one worker, and in Python it sidesteps the GIL for CPU parallelism (Chapter 5, Section 5.13). The cost is memory (each process has its own heap) and few concurrent requests per worker: a synchronous worker handling one request at a time is fully occupied while that request waits on the database.

**3. Event loop.**

```text
 one thread per core, each running a loop over epoll:
   ready: conn 17 readable → parse a bit → handler runs until it awaits I/O → register callback
   ready: DB result for conn 4 → resume its handler → write response
   ready: conn 230 readable → ...
 tens of thousands of connections per thread; no thread waits; every handler must never block
```

Node.js runs JavaScript on one event-loop thread per process; Nginx runs one event loop per worker process (typically one per core). Memory per connection is small (a socket and some buffers), so idle and long-lived connections are cheap. The rule that makes it work is absolute: **nothing may block the loop**. One handler doing a CPU-heavy computation or a blocking call stalls every connection on that thread (Chapter 5, Section 5.13).

**4. Async runtimes and lightweight threads.**

- **Futures and async/await** over an event loop (Netty in Java, Python's asyncio servers such as Uvicorn, Rust's Tokio, .NET): event-loop efficiency with code that reads more sequentially.
- **Lightweight threads** scheduled by the runtime (Go's goroutines, Java's virtual threads; Chapter 5, Section 5.13): write ordinary blocking code; when a goroutine or virtual thread blocks on I/O, the runtime parks it and runs another on the same OS thread. Go's `net/http` starts a goroutine per connection.

### Concrete: comparison

| | Thread per connection / request | Worker processes | Event loop | Lightweight threads |
|---|---|---|---|---|
| Who pays for waiting | an OS thread | a whole process | nobody (a registered callback) | a cheap runtime thread |
| Idle connections | costly (per-connection) or cheap (with an NIO poller) | costly | cheap | cheap |
| Programming model | sequential, blocking | sequential, blocking | non-blocking, callbacks or async/await | sequential, blocking |
| What breaks it | too many threads; all threads blocked on one slow dependency | too few workers for the wait time; memory | **any blocking or CPU-heavy work on the loop** | pinning, blocking calls the runtime cannot intercept, unbounded concurrency |
| Failure isolation | a thread's crash can take the process | strong: per process | weak: one loop serves many connections | as for threads |
| Typical examples | classic servlet containers; Tomcat NIO (thread per request) | Gunicorn (sync workers), PHP-FPM | Node.js, Nginx | Go `net/http`, Java virtual threads |

Lightweight threads and event loops make **waiting** cheap. They do not add CPU, database connections or downstream capacity (the accuracy point of Chapter 5, Section 5.13): a server that can hold 100,000 concurrent requests still has a pool of perhaps 20 database connections behind it. Chapter 11 is about that mismatch.

> **Java lens: Tomcat and the Servlet API.** A Tomcat **connector** (the NIO connector by default in Spring Boot) has an **acceptor** thread that takes connections from the kernel, a **poller** that watches all connections with Java NIO selectors (`epoll` on Linux), and an **executor**, the worker thread pool. Its main limits map onto Section 10.5's queues: `acceptCount` is the `listen` backlog; `maxConnections` caps connections held (idle ones included); `maxThreads` caps workers. When a connection becomes readable, the poller hands it to a worker, which parses the request into an `HttpServletRequest`, runs the configured **filters** (`javax`/`jakarta.servlet.Filter`, each calling `chain.doFilter(req, res)`: the onion of Section 10.6), and invokes a **servlet**'s `service` method; the response goes out through `HttpServletResponse`, and `isCommitted()` reports whether headers have been sent. With virtual threads enabled, each request runs on a virtual thread instead of a pool thread.
>
> **Framework lens: Spring's `DispatcherServlet`.** Spring MVC registers a single servlet, the `DispatcherServlet`, behind the filter chain. For each request it asks its `HandlerMapping`s which controller method matches (the routing table built from `@GetMapping` and friends), runs `HandlerInterceptor`s (`preHandle`, `postHandle`, `afterCompletion`: another onion, inside the servlet), uses a `HandlerAdapter` to call the method with arguments bound from the request (path variables, `@RequestBody` deserialised by an `HttpMessageConverter`), converts the return value back through a message converter, and routes exceptions to `@ExceptionHandler` / `@ControllerAdvice` methods. The magic of an annotated controller is this dispatch table, built at startup.
>
> **Other runtimes.** Gunicorn runs a master process and N worker processes; its documentation suggests about (2 × cores) + 1 workers as a starting point for synchronous workers, and offers threaded and async worker types (or Uvicorn workers for asyncio applications) when requests spend most of their time waiting. Node's `http` module runs handlers on the event loop; to use several cores, run several Node processes (the `cluster` module or a process manager) behind a load balancer, and keep CPU-heavy work off the loop (worker threads).

### Guarantees and non-guarantees

Each architecture guarantees its own trade-off: thread and process models give simple blocking code, event loops give cheap connections, lightweight threads give both for intercepted I/O. None of them guarantees throughput beyond the slowest downstream resource, and none protects you from the specific way it fails: thread exhaustion, worker starvation, a blocked loop, or unbounded concurrency hammering a database.

### Failure and diagnosis

- **Thread-pool server: everything stalls when one dependency is slow.** All `maxThreads` workers are waiting on the same slow call; new requests queue. Evidence: a thread dump with hundreds of threads in the same stack frame (a socket read in the HTTP client of the slow dependency). Fix: timeouts on the dependency (Section 10.10), bulkheads (Chapter 24).
- **Event-loop server: all requests slow at once, CPU at 100% of one core.** A handler is doing CPU-heavy work or a synchronous call. Evidence: event-loop lag metrics (Chapter 4, Section 4.5) and a CPU profile pointing at one function. Fix: move it to a worker thread or another service.
- **Worker-process server: timeouts under modest load.** Too few synchronous workers for the time requests spend waiting; Gunicorn kills workers that exceed its timeout. Evidence: worker timeout messages in the logs and every worker busy. Fix: more workers, or threaded or async workers for I/O-bound applications.

### Backend connection

Know which model your server uses and its failure signature, because the same symptom ("slow under load") has different causes and fixes in each. Whatever the model, the real limits are downstream (database connections, other services' capacity) and in the queues of Section 10.5, which Chapter 11 sizes.

---

## 10.8 Error Handling Across Layers: Who Produces Which 5xx

### Definition

A `5xx` status says that something on the server side failed (Chapter 7, Section 7.5). In a layered system, the response the client sees may have been **generated** by any component on the path: the CDN, the load balancer, the reverse proxy, the gateway, or the application. Knowing which one, and why, is the first step of every incident.

### Why it matters

A `502` from the load balancer and a `500` from the application need opposite investigations: one is about connections between components, the other about code. Engineers who treat all `5xx` errors as "the app is broken" spend the first half of an incident in the wrong logs.

### Mechanism: the decision table

| Status | Usually generated by | What happened | Evidence | First check |
|---|---|---|---|---|
| **500** Internal Server Error | the **application** | an unhandled exception, or an error mapped to 500 by the error middleware | a stack trace in the application log with the request ID | the code path in the trace |
| **502** Bad Gateway | a **proxy or load balancer** | it could not get a valid response from the upstream: connection **refused** (nothing listening, wrong port or bind address), connection **reset** or **closed early** (app crashed or was killed mid-request; the keep-alive race of Chapter 7, Section 7.9), or an **invalid** response (garbage, protocol mismatch such as TLS sent to a plain-HTTP port) | the proxy's error log names the reason (for example Nginx's `connect() failed (111: Connection refused) while connecting to upstream` or `upstream prematurely closed connection`); usually no application log line at all | is the upstream up, on the right address and port; did instances restart or get killed (OOM, Chapter 3, Section 3.17) |
| **503** Service Unavailable | the **load balancer** (no healthy targets), a **proxy** (upstreams all marked down, or rate limiting configured to answer 503), or the **application** on purpose (load shedding, maintenance mode, an open circuit breaker; Chapter 24) | no capacity to serve right now, by state or by decision | the balancer's target-health view; `Retry-After` headers; application metrics for shed requests | target health and why targets are failing their checks |
| **504** Gateway Timeout | a **proxy or load balancer** | the upstream accepted the request but did not respond within the proxy's timeout | the proxy log's upstream time equals its timeout (for example `upstream timed out (110: Connection timed out) while reading response header`); the application may log the same request **completing later with 200** | the slow dependency or query; the timeout chain (Section 10.10) |

Three refinements:

- **A 504 does not mean the work failed.** The proxy stopped waiting; the application may finish and commit a few seconds later (Chapter 7, Section 7.15). For non-idempotent requests, the outcome is unknown.
- **Some proxies use non-standard codes in their logs** for the client side. Nginx, for example, logs `499` when the **client** closed the connection before a response was ready: a sign that clients time out before the proxy does.
- **The application can produce 502 and 504 too**, when it is itself acting as a gateway to another service and maps that service's failure onto these codes. Your request ID and error-body format tell you whether the response is yours.

### Concrete: telling the layers apart from one response

```text
 HTTP/2 504
 server: awselb/2.0                          ← generated by the load balancer, not your app
 content-type: text/html                     ← an HTML error page, while your API returns JSON
 (no x-request-id header)                    ← your middleware never ran on this response
```

The `Server` header, the format of the body, and the presence of your own headers identify the component that wrote the response. Then read that component's log for the same moment.

### Guarantees and non-guarantees

Status codes guarantee only a class of failure. A `502` does not say which of refused, reset or invalid occurred; a `504` does not say whether the work completed; a `503` does not say whether the cause is health, capacity or policy. The components' own logs do.

### Failure and diagnosis

- **`502` spikes during deploys.** Instances terminate with requests in flight, or the balancer still routes to instances that have stopped listening. Fix: draining (Section 10.3) plus graceful shutdown (Chapter 4, Section 4.11).
- **Steady low rate of `502`s.** The keep-alive race between a balancer and an application whose idle timeout is shorter (Chapter 7's Diagnosis Lab 3).
- **`504`s with healthy-looking application metrics.** The slow part is outside the handler's timing (queueing, Section 10.5) or in a dependency. Compare the proxy's upstream time with the handler's own time.

### Backend connection

Make every layer identifiable: your responses carry a request ID and a consistent error format (RFC 9457, Chapter 7, Section 7.5), your proxies log upstream status and time per request, and dashboards split `5xx` by the component that generated them.

---

## 10.9 Long-Lived Connections: WebSockets, SSE, Long Polling

### Definitions

- **Long polling**: the client sends an ordinary request, and the server holds it open until there is news (or a timeout), answers, and the client immediately sends the next request.
- **Server-sent events** (SSE): one HTTP response that never ends; the server writes events into it as they happen, in a simple text format (`Content-Type: text/event-stream`). One direction only, server to client.
- **WebSocket** (RFC 6455): a protocol that starts as an HTTP request and **upgrades** the connection into a full-duplex channel of **frames** that either side can send at any time.

### Why they exist

HTTP is client-initiated (Chapter 7, Section 7.1): a server cannot send an unrequested message. Chat, live dashboards, notifications, collaborative editing and multiplayer games need the server to push data as soon as it exists. Polling every few seconds wastes requests when nothing changes and adds delay when something does. These three techniques keep a channel open instead.

### Mechanism 1: the WebSocket upgrade and frames

```http
GET /chat HTTP/1.1
Host: chat.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==           a random 16-byte value, Base64-encoded
Sec-WebSocket-Version: 13
Origin: https://app.example.com                         the server must check it (see below)

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=     Base64(SHA-1(key + a fixed GUID from the RFC))
```

The `Sec-WebSocket-Accept` value proves that the server understood the WebSocket handshake (it is not a security mechanism; it stops a non-WebSocket server or cache from accidentally completing it). After the blank line that ends the `101` response, the TCP connection no longer carries HTTP: it carries frames.

```text
 frame header
 ┌─┬───┬────────┬─┬─────────────┬────────────────────────┬───────────────┬──────────────┐
 │F│RSV│ opcode │M│ payload len │ extended length         │ masking key   │ payload      │
 │I│   │ 4 bits │A│ 7 bits      │ 16 or 64 bits if len is │ 4 bytes, only │ (XOR-masked  │
 │N│   │        │S│             │ 126 or 127              │ if MASK = 1   │ if masked)   │
 │ │   │        │K│             │                         │               │              │
 └─┴───┴────────┴─┴─────────────┴────────────────────────┴───────────────┴──────────────┘
 opcodes: 0x1 text, 0x2 binary, 0x0 continuation (FIN = 0 means more fragments follow),
          0x8 close (with a status code: 1000 normal, 1001 going away ...), 0x9 ping, 0xA pong
 frames from client to server MUST be masked; frames from server to client are not
```

Messages can be split into several frames; **ping** and **pong** frames check that the other side is still there and keep idle connections alive through middleboxes. **Close** frames end the connection cleanly, with a status code (`1001` "going away" is what a server sends when it shuts down).

Because a WebSocket is opened by a browser **with cookies**, and the same-origin policy does not restrict WebSocket connections, the server must **check the `Origin` header** during the handshake against an allowlist. Otherwise any website can open a WebSocket to your server as the logged-in user (cross-site WebSocket hijacking; the CSRF family of Chapter 19).

### Mechanism 2: SSE

```http
GET /orders/9001/events HTTP/2
Accept: text/event-stream
Last-Event-ID: 41                       sent by the browser on reconnect: "resume after event 41"

HTTP/2 200
Content-Type: text/event-stream
Cache-Control: no-store

id: 42
event: status
data: {"orderId":9001,"status":"PAID"}

id: 43
event: status
data: {"orderId":9001,"status":"SHIPPED"}

: keep-alive comment line, sent every 20 s so proxies see traffic

```

Each event is a few `field: value` lines ended by a blank line. The browser's `EventSource` API parses them, reconnects automatically if the connection drops (after the `retry:` interval, if the server sent one), and sends `Last-Event-ID` so the server can replay what was missed. SSE is ordinary HTTP, so it passes through proxies and works with HTTP/2 (many streams on one connection, avoiding HTTP/1.1's six-connections-per-host limit, Chapter 9, Section 9.1). It is server-to-client only; the client sends data with normal requests.

| | Long polling | SSE | WebSocket |
|---|---|---|---|
| Direction | server to client (one message per request) | server to client | both ways |
| Protocol | plain HTTP | plain HTTP, streaming response | HTTP upgrade, then its own frames |
| Reconnect and resume | client re-requests | built in (`Last-Event-ID`) | application must build it |
| Through proxies and CDNs | easy | easy if buffering is off | needs proxies that support the upgrade |
| Typical use | fallback, rare updates | notifications, live status, progress, streaming text | chat, collaborative editing, games, two-way streams |

### Mechanism 3: what long-lived connections do to proxies and capacity

- **Connection count becomes the limit, not request rate.** 100,000 connected users mean 100,000 open connections, each a file descriptor (Chapter 4, Section 4.9) and some kilobytes of buffers at **every** layer: load balancer, proxy, application. A thread-per-connection server cannot hold that many; event loops and lightweight threads can (Section 10.7).
- **Idle timeouts kill quiet connections.** Load balancers, proxies and NAT gateways close connections with no traffic after their idle timeout (Chapter 6, Section 6.5). A WebSocket with no messages for 90 seconds through a 60-second balancer is cut. Fix: ping/pong or SSE comment lines more often than the shortest idle timeout on the path.
- **Proxies must be configured for it.** WebSocket upgrades must be passed through (some proxies need explicit settings for the `Upgrade` and `Connection` headers); SSE needs response **buffering off** (Section 10.2), or events arrive in bursts; read timeouts must be longer than the gap between messages.
- **Load balancing happens once per connection.** A WebSocket stays on the backend chosen at the upgrade for hours. After scaling out, new instances get only new connections, so old instances stay overloaded until users reconnect: least-connections helps choose well at connect time, but cannot move existing connections.
- **Deploys become reconnect storms.** Restarting an instance drops every connection on it; if all clients reconnect at the same instant, the remaining instances get a burst of handshakes (TLS included). Fix: drain gradually (send close frames over a few minutes), and have clients reconnect with **randomised backoff** (Chapter 24).
- **Fan-out across instances.** A user connected to instance A must receive an event produced on instance B. Instances therefore share a publish/subscribe channel (a message broker or Redis pub/sub, Chapters 21 and 25) and each pushes to its own connections.

### Guarantees and non-guarantees

A WebSocket gives an ordered, reliable stream of messages over its TCP connection while it lasts; SSE adds automatic reconnection and resumption by event ID. Neither guarantees delivery across a dropped connection (messages sent while it was down are lost unless the application replays them), exactly-once processing (Chapter 6, Section 6.9), or that the client is still present (only heartbeats tell).

### Failure and diagnosis

- **Connections drop every 60 seconds exactly.** An idle timeout somewhere on the path; heartbeats too infrequent or absent. Evidence: close codes and the regularity of the interval; the timeout values of each hop.
- **SSE events arrive all at once.** Response buffering at a proxy. Evidence: the events reach the browser together when the buffer fills or the response ends. Fix: turn buffering off for the route.
- **Memory grows with connected users and never falls.** Connections whose clients vanished are never closed. Fix: heartbeats with a deadline, and closing connections that miss them.

### Backend connection

Choose the simplest tool: SSE for server-to-client updates, WebSockets only when the client must also stream to the server. Plan capacity in connections, set heartbeats below every idle timeout, disable buffering where you stream, check `Origin` on upgrades, and design reconnection with backoff and resumption from the start.

---

## 10.10 Timeouts at Each Hop, and How They Must Nest

### Definitions

A **timeout** bounds how long one component waits for another. An **idle timeout** bounds how long a connection may stay open with no traffic. A **deadline** is an absolute time by which the whole request must be finished; each hop's remaining **budget** is the deadline minus now.

### Why nesting matters

Every hop on the path has its own timeouts, set by different teams and products. If they are not chosen together, two things go wrong. When an inner timeout is **longer** than an outer one, the outer component gives up while the inner one keeps working: the client sees a timeout and probably retries, while the server is still doing the original work, so a slow moment doubles the load. When idle timeouts are in the wrong order, connections are reused just as the other side closes them (the keep-alive race, Chapter 7, Section 7.9).

### Mechanism: two rules

**Rule 1: request timeouts shrink as you go inward.** Each component must give up **before** the component that is waiting for it, leaving itself time to clean up and respond.

```text
 client ────────────────────────────────────────────────────── 10 s total
   load balancer response timeout ─────────────────────────── 9 s
     proxy upstream read timeout ────────────────────────── 8 s
       application request deadline ─────────────────────── 7 s
         call to payment service (connect 300 ms) ────── 3 s
         DB pool acquire ── 0.5 s
         DB statement timeout ───── 2 s
```

With these values a slow database query is cut at 2 s, the application returns a clear error (or a degraded answer) within its 7 s, and every outer layer receives a real response instead of timing out itself.

**The wrong way round.** The proxy waits 60 s, the application has no deadline, the database has no statement timeout, and the client gives up after 10 s:

```text
 t = 10 s   client times out, shows an error, and the user (or the client library) retries
 t = 10 s   proxy, application and database are all still working on the first request
 t = 20 s   the retry is also running; the database now executes two copies of the slow query
 t = 60 s   the proxy finally returns 504 to a client that left 50 seconds ago
```

**Rule 2: idle timeouts grow as you go inward.** For each persistent connection, the **client** side must close idle connections **before** the server side does, so a client never sends a request on a connection the server is closing:

```text
 browser/CDN → load balancer:  client's idle < LB's client-side idle timeout (e.g. 60 s)
 load balancer → application:  LB's backend idle (60 s) < application keep-alive timeout (65 s)
 application → database pool:  pool's idle timeout < database's and any NAT's idle timeout (Chapter 6, Section 6.5)
```

### Concrete: deadline propagation

Fixed per-hop timeouts waste budget: if the client has already waited 6 s, an application that grants a downstream call its full 3 s may overrun the client's 10 s. **Deadline propagation** passes the remaining budget along: the caller sends its deadline (gRPC does this with a `grpc-timeout` header; HTTP services often use a custom header), and each service sets its own timeouts to `min(own limit, remaining budget − margin)`. When the client disconnects, a well-built server **cancels** the work in progress (stops the query, abandons downstream calls), instead of finishing it for nobody (Chapter 5, Section 5.12 covered cancellation).

### Guarantees and non-guarantees

Correctly nested timeouts guarantee that each layer gets a real answer (a response or a clear error) before it gives up itself, and that abandoned work is bounded. They do not guarantee that the operation failed when a timeout fires (Chapter 7, Section 7.15), and they do not make retries safe: retries need idempotency (Chapter 13) and budgets (Chapter 24).

### Failure and diagnosis

- **Many `499`s or client timeouts, followed later by `200`s in the application log** for the same request IDs. Clients give up before the server does. Fix: shrink the inner timeouts below the client's.
- **Database load doubles during slowdowns.** Retries of requests whose original work is still running. Evidence: duplicate queries with identical parameters seconds apart. Fix: statement timeouts below the application deadline, cancellation on client disconnect, and retry budgets.
- **A dependency with no timeout.** One slow call blocks a worker forever (Section 10.7). Evidence: thread dumps with threads stuck in a socket read. Fix: every outbound call gets connect and read timeouts.

### Backend connection

Write the timeout chain down for each request path, from the client to the database, and check that request timeouts decrease and idle timeouts increase inward. Give every outbound call explicit connect and read timeouts, set statement timeouts in the database, propagate deadlines between services, and cancel work when the caller has gone.

---

## 10.11 Diagnosis Toolkit: Reading the Proxy's Evidence

### What to look at

The reverse proxy and load balancer see every request, including those your application never receives, so their logs are the first evidence for any `5xx`, `4xx` without application logs, or latency problem. Make sure the access log records, per request:

```text
 time  client_ip  method path  status  request_time  upstream_addr  upstream_status  upstream_response_time  request_id
 14:02:07 203.0.113.7 POST /v1/orders 504  8.001  10.0.3.7:8080  504  8.000  7f3a...
          ↑ total time at the proxy                ↑ which instance   ↑ what the upstream returned (or "-")
                                                                           ↑ time waiting for the upstream
```

Nginx, for example, provides these as `$request_time`, `$upstream_addr`, `$upstream_status` and `$upstream_response_time`. Reading them:

- `status` = 502 and `upstream_status` empty or 502, with tiny upstream time: the connection to the upstream failed; check the error log for refused or reset.
- `status` = 504 and `upstream_response_time` equal to the timeout: the upstream was too slow.
- `request_time` much larger than `upstream_response_time`: time was spent with the client (a slow upload or download), not in the application.
- the same `request_id` in the application log: the request reached the application, and its log tells the rest.

The proxy's **error log** explains failures in words (connection refused, upstream timed out, client body too large, upstream closed the connection early). The application's own logs and traces (Chapter 26) take over once the request ID shows the request arrived.

### Concrete: the proxy's error log, decoded

The access log says **what** happened; the error log says **why**. These are Nginx's messages; other proxies log equivalents in their own words:

| Error-log message (abridged) | Client saw | What it means |
|---|---|---|
| `connect() failed (111: Connection refused) while connecting to upstream` | 502 | nothing listening at the upstream address and port (wrong port, loopback bind, process down: Diagnosis Lab 1) |
| `no live upstreams while connecting to upstream` | 502 | every upstream in the pool was marked failed; the proxy had nowhere to send the request |
| `upstream prematurely closed connection while reading response header from upstream` | 502 | the upstream closed the connection before answering: a crash or kill mid-request, or the keep-alive race (Chapter 7, Section 7.9) |
| `recv() failed (104: Connection reset by peer) while reading response header from upstream` | 502 | the upstream reset the connection (Chapter 6, Section 6.8) |
| `upstream timed out (110: Connection timed out) while reading response header from upstream` | 504 | the upstream accepted the request but did not answer within the proxy's read timeout (Diagnosis Lab 2) |
| `client intended to send too large body: N bytes` | 413 | the request body exceeded the proxy's limit (Diagnosis Lab 3) |

The numbers in parentheses are the operating system's error codes (`ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`), the same events Chapter 6 (Section 6.14) mapped to the wire, now seen from the proxy's side.

### Concrete: following one request through every layer

A user reports that one checkout took more than a second. Its request ID, returned in a response header (Section 10.6), is `7f3a`. Each layer's log for that ID gives a total time, and each total should contain the next layer's:

```text
 CDN log           7f3a  status 200  cache MISS  total 1,240 ms   origin time 1,190 ms
 load balancer     7f3a  status 200  target 10.0.3.7  request→response 1,182 ms
 reverse proxy     7f3a  status 200  request_time 1.178  upstream_response_time 1.175
 application       7f3a  POST /v1/checkout 200  handler 1,168 ms
                         ├─ DB pool wait       4 ms
                         ├─ SELECT … FOR UPDATE   1,102 ms   ← the time is here
                         └─ other work         62 ms
 database          slow-query log: SELECT ... FOR UPDATE, 1,101 ms, 1,089 ms of it waiting for a lock
```

The procedure: start at the outermost layer and move inward while each inner layer's time still accounts for the outer one's. Here every layer passes the time inward, until the application, where one statement explains it, and the database, which shows the statement mostly **waited for a row lock** held by another transaction (Chapter 17). When an outer total is much larger than the next inner one, the time was spent in that layer itself, its queues, or the network between them (for example a CDN total far above its origin time: the user's own link or a slow upload).

To identify which component **generated** an error, without logs, reproduce the request with `curl -v` and read the response: the `Server` header, the body's format, and whether your own headers (request ID, `Content-Type: application/problem+json`) are present (Section 10.8).

### Backend connection

Agree on the request ID across layers, log upstream status and timing at the proxy, and keep the error log. Most of the diagnosis labs below are solved in a few minutes with exactly these fields.

---

## Chapter Summary

A request from the internet first meets the **edge**: a CDN that serves cached responses from a nearby point of presence, shields the origin and terminates TLS close to the user; a WAF that blocks requests matching attack patterns; and DDoS protection that absorbs floods with capacity larger than the attack. It then passes **reverse proxies** and **load balancers**, which terminate TLS, enforce limits, route by host and path, buffer slow clients, compress, rate-limit, and add forwarding headers. Those headers are only trustworthy from proxies you know: the client is the first untrusted address reading `X-Forwarded-For` from the right. A balancer picks a backend by an **algorithm**: round robin balances counts, not load; least connections and power of two choices handle uneven costs; consistent hashing gives affinity that survives pool changes. It also uses active and passive **health checks** (local readiness, never deep dependency checks), avoids **sticky sessions** where it can, and **drains** instances before removing them. An **API gateway** adds API policy at the entry; a **service mesh** handles service-to-service mTLS, retries and telemetry through sidecars.

Inside the server, the kernel completes handshakes into the **accept queue**; an acceptor takes connections; a poller holds idle ones for free; a parser enforces limits before any code runs; a worker, a virtual thread or an event-loop callback runs the request through the **router**, an onion of **middleware** (request ID and logging outermost, error mapping, authentication) and the **handler**; and the response writer commits headers, after which the status cannot change, flushes and streams with backpressure. Servers differ in who pays for waiting: **threads** (simple, limited), **worker processes** (isolated, memory-hungry), **event loops** (cheap connections, nothing may block) and **lightweight threads** (both, for intercepted I/O). None adds downstream capacity.

**500** comes from the application; **502** from a proxy that could not get a valid response (refused, reset, invalid); **503** from a balancer with no healthy targets or a deliberate refusal; **504** from a proxy that stopped waiting, while the work may still complete. **WebSockets** upgrade a request with `101` into a framed, full-duplex channel; **SSE** streams events in one long response with built-in resumption; both turn capacity planning into connection counting and need heartbeats below idle timeouts, unbuffered proxies, `Origin` checks and reconnection with backoff. Finally, **request timeouts shrink inward and idle timeouts grow inward**, deadlines should propagate, and abandoned work should be cancelled.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| CDN | Fast delivery and origin offload for cacheable responses | Speed for uncacheable responses; safety of misconfigured caching |
| WAF | Requests matching its rules are blocked | That attacks are caught, or that legitimate requests are not |
| DDoS protection | Availability under attacks within its capacity | Cheap expensive endpoints; protection from application bugs |
| Reverse proxy | One controlled entry point for TLS, limits, routing | Trustworthy forwarding headers from outside; its own availability without redundancy |
| `X-Forwarded-For` | A chain of addresses appended by proxies | Truth of any value not written by a proxy you trust |
| Round robin | Equal request counts per backend | Equal load when request costs vary |
| Least connections / power of two choices | Load tracking in-flight work | Good choices when the count is not a load measure (idle long-lived connections) |
| Consistent hashing | Affinity that mostly survives pool changes | Even load with hot keys |
| Health checks | Backends failing checks get no traffic | That passing backends serve requests correctly |
| Connection draining | In-flight requests finish before removal (up to the timeout) | Completion of work longer than the draining time |
| Middleware chain | Every request passes the same ordered concerns | Coverage of failures before routing; changes after commit |
| Event loop / lightweight threads | Cheap waiting and idle connections | More CPU, connections or downstream capacity |
| 502 / 504 | The proxy could not get, or stopped waiting for, a response | Whether the upstream processed the request |
| WebSocket | Ordered, reliable messages while the connection lasts | Delivery across reconnects; client presence without heartbeats |
| SSE | Automatic reconnection and resumption by event ID | Delivery if the server does not replay missed events |
| Nested timeouts | A real answer at each layer before it gives up | That a timed-out operation failed; safe retries |

---

## Interview Drill

**Q1. What is a CDN, and what does it do for a backend?**
A strong answer must mention:
- Edge servers in points of presence close to users; requests routed to the nearest one.
- Cache hits served at the edge by HTTP caching rules (`s-maxage`, `Vary`); misses go to the origin, often through an origin shield.
- TLS terminated near the user and warm connections to the origin help even uncacheable requests.
- Purging, and the risk of caching private responses through wrong headers or override rules.
*Common weak answer:* "A CDN stores static files." It omits the cache rules, the origin shield, the TLS benefit and the danger of caching personal data.

**Q2. Forward versus reverse proxy? What does a reverse proxy do?**
A strong answer must mention:
- Forward proxy acts for clients reaching any server; reverse proxy acts for servers, which clients never see.
- TLS termination, limits and malformed-request rejection, routing by host and path.
- Request and response buffering to protect workers from slow clients (and its cost for streaming).
- Compression, rate limiting, header rewriting, forwarding headers, load balancing over pooled connections.
*Common weak answer:* "A reverse proxy hides the server." Hiding is a side effect; the answer is the list of jobs it takes from applications.

**Q3. What is the difference between L4 and L7 load balancing?**
A strong answer must mention:
- L4 decides once per connection from addresses and ports, without parsing the payload; L7 terminates HTTP and decides per request.
- L7 can route on path, host and headers, retry, and rewrite; L4 supports any protocol at lower cost.
- Long-lived HTTP/2, gRPC and WebSocket connections pin load behind L4 (and per connection even at L7 for WebSockets).
- Client IP visibility: preserved or passed by the PROXY protocol at L4; via forwarding headers at L7.
*Common weak answer:* "L4 is TCP and L7 is HTTP." True but without the per-connection consequence it explains nothing.

**Q4. Which load-balancing algorithms do you know, and when does each fail?**
A strong answer must mention:
- Round robin and weighted round robin: fail with uneven request costs.
- Least connections: good for uneven costs; herds with stale counts across many balancers; misleads with idle long-lived connections.
- Power of two choices: near least-connections quality without coordination.
- Hashing and consistent hashing: affinity; consistent hashing limits remapping when the pool changes; hot keys remain a problem.
- A worked example: round robin putting all heavy requests on one backend.
*Common weak answer:* naming algorithms without saying what each one balances and when it breaks. Interviewers ask precisely because round robin balances counts, not load, and that difference causes real outages.

**Q5. What are health checks? Active versus passive, liveness versus readiness?**
A strong answer must mention:
- Active: the balancer probes an endpoint on a schedule with failure and success thresholds; passive: it ejects backends whose real requests fail.
- Liveness: should this process be restarted; readiness: should it receive traffic now (startup, draining, missing essentials).
- Deep checks of shared dependencies can remove every instance at once and turn a blip into an outage.
- Draining and slow start around removals and additions.
*Common weak answer:* "The health check calls the database to make sure everything works." That is exactly the check that causes total outages.

**Q6. What are sticky sessions, and what do they cost?**
A strong answer must mention:
- Affinity by balancer cookie, IP hash or application cookie, so a client keeps hitting one backend.
- Costs: uneven load, new instances getting no traffic, lost state when an instance dies, slow draining.
- Usually a workaround for state in instance memory; the fix is stateless instances with shared session storage.
- Acceptable for performance (warm caches), not for correctness.
*Common weak answer:* "Sticky sessions keep users logged in." Then any instance failure logs them out; shared sessions are the real fix.

**Q7. API gateway versus load balancer versus service mesh?**
A strong answer must mention:
- Load balancer: spread traffic, health checks, TLS.
- Gateway: north–south API policy (keys, tokens, quotas, rate limits, transformation, versioned routing).
- Mesh: east–west sidecars giving mTLS identity, retries, timeouts, per-request balancing and telemetry.
- Each is another hop with its own failure modes; retries in several layers multiply.
*Common weak answer:* treating them as three names for a proxy, without the traffic direction and the job of each. Without that, you cannot say which one a system needs, or what each adds in hops and failure modes.

**Q8. What happens between `accept()` and your handler?**
A strong answer must mention:
- The kernel's accept queue; the acceptor; registration with the poller; idle connections waiting without a thread.
- Parsing with limits (header size, body size, timeouts) that can reject requests before any application code runs.
- Hand-off to a worker (pool thread, virtual thread or loop callback), possibly through a queue.
- Routing, the middleware chain, binding and deserialisation, then the handler; keep-alive returns the connection to the poller.
*Common weak answer:* "The framework calls my controller." It hides the queues where most latency and many failures occur.

**Q9. What is middleware? Explain the filter chain.**
A strong answer must mention:
- Functions wrapped around the handler, each calling `next`: code before runs on the way in, after on the way out (the onion).
- Cross-cutting concerns: request IDs, logging, timing, error mapping, authentication, compression.
- Order matters (request ID and logging outermost; error mapping outside authentication), and middleware can short-circuit.
- After the response is committed, middleware cannot change the status.
*Common weak answer:* "Middleware runs before the controller." It also runs after, and its order changes behaviour.

**Q10. Compare thread-per-request, worker-process and event-loop servers.**
A strong answer must mention:
- Who pays for waiting: a thread, a process, or nothing.
- Thread-per-request: simple blocking code; exhausted when all threads wait on one slow dependency.
- Worker processes: isolation and (in Python) CPU parallelism; memory cost and few concurrent requests per worker.
- Event loop: many cheap connections; any blocking or CPU-heavy handler stalls everything; lightweight threads combine the models; none adds downstream capacity.
*Common weak answer:* "Event loops are faster." They make waiting cheaper; they are slower for CPU-bound handlers.

**Q11. Who produces a 502, a 503 and a 504?**
A strong answer must mention:
- 502: a proxy or balancer that got no valid response (refused, reset, closed early, garbage).
- 503: a balancer with no healthy targets, a proxy's rate limit, or the application deliberately shedding load or in maintenance.
- 504: a proxy or balancer whose timeout expired while waiting; the upstream may still finish.
- 500: the application's unhandled error; identify the component by `Server` header, body format and request ID.
*Common weak answer:* "They all mean the backend is down." They point to different components and different investigations.

**Q12. How do WebSockets work? How do they compare with SSE and long polling?**
A strong answer must mention:
- An HTTP `GET` with `Upgrade: websocket` and `Sec-WebSocket-Key`; `101 Switching Protocols` with `Sec-WebSocket-Accept`; then frames (opcodes, masking from client, ping/pong, close).
- Full-duplex over one TCP connection; the server must check `Origin`.
- SSE: one streaming HTTP response, server to client, with automatic reconnection and `Last-Event-ID`; long polling: held requests.
- Effects: connection counts, idle timeouts and heartbeats, proxy configuration, per-connection balancing, reconnect storms, pub/sub fan-out.
*Common weak answer:* "WebSockets are faster HTTP." They are a different protocol after the upgrade, with very different operational costs.

**Q13. How should timeouts be set across the hops of a request?**
A strong answer must mention:
- Request timeouts shrink inward: client > load balancer > proxy > application deadline > dependency calls and DB statement timeouts.
- Otherwise outer layers give up while inner ones keep working, retries pile onto unfinished work.
- Idle timeouts grow inward: each connection's client side closes idle connections before its server side.
- Deadline propagation and cancellation on client disconnect; every outbound call needs connect and read timeouts.
*Common weak answer:* "Set long timeouts so requests don't fail." Long inner timeouts turn slow moments into overload.

**Q14. How does a server behind proxies learn the client's real IP, and how can that go wrong?**
A strong answer must mention:
- Proxies append the address they saw to `X-Forwarded-For` (or use `Forwarded`); L4 balancers can use the PROXY protocol.
- The leftmost value is written by the client and can be forged.
- Configure trusted proxies, start from the TCP peer, read right to left, and take the first untrusted address; or have the outermost proxy overwrite the header.
- Mistakes: everyone appears as the balancer (headers ignored), or spoofed addresses bypass rate limits and access rules (headers over-trusted).
*Common weak answer:* "Read the first IP in `X-Forwarded-For`." That is the attacker-controlled one.

**Q15. Why should response buffering be turned off for some routes?**
A strong answer must mention:
- Buffering lets the proxy absorb slow clients and frees application workers quickly.
- It holds streamed output (SSE, progress streams, large exports) until the buffer fills or the response ends.
- So streaming routes need buffering off, and their timeouts set for long-lived responses.
- Request buffering similarly delays very large uploads but protects workers.
*Common weak answer:* "Buffering makes responses faster." It protects workers; for streams it breaks the user experience.

**Q16. What does a 504 tell you about whether the request succeeded?**
A strong answer must mention:
- Only that the proxy stopped waiting.
- The application may still complete and commit; its log may show a later 200 for the same request ID.
- For non-idempotent operations the outcome is unknown; retries need idempotency keys (Chapter 13).
- Fix the timeout chain so the application answers before the proxy gives up.
*Common weak answer:* "The request failed, so it is safe to retry." It may have succeeded.

---

## Scenario Exercises

1. Choose a load-balancing algorithm, with a reason, for: (a) a JSON API whose endpoints range from 5 ms to 2 s; (b) a service where each instance keeps a large in-memory cache of per-tenant data; (c) a fleet of WebSocket chat servers that scales up every evening.
2. A service's health endpoint checks the database, Redis and two downstream APIs. One of the downstream APIs has a 30-second outage, and during it the load balancer returns `503` for every request to your service. Explain the chain of events and redesign the health checks.
3. Your middleware chain is `[authenticate, rateLimit, requestId, logging, errors]`. List what goes wrong with this order, and propose a better one.
4. A Node.js API's p99 latency jumps from 40 ms to 3 s whenever someone downloads the monthly CSV export, and only on the instance serving the export. Explain the mechanism and two fixes.
5. Each deployment of your WebSocket notification service causes a spike of connection errors and CPU on the instances that stay up. Explain why and design a better rollout.
6. Your request path has these timeouts: client 30 s, load balancer 60 s, reverse proxy 120 s, application none, database statement none. Users report occasional double orders after slow periods. Explain the connection and fix the configuration.

---

## Diagnosis Lab

**Lab 1: Every request returns 502 after moving to containers**
- *Symptom:* After the orders service moved from a VM to a container, the reverse proxy returns `502 Bad Gateway` for every request. The container is running and its logs show "Server started on port 8080".
- *Hypotheses:* (a) the application crashes on each request; (b) the proxy cannot connect to the application; (c) the proxy points at the wrong address or port.
- *Evidence:* The proxy's error log: `connect() failed (111: Connection refused) while connecting to upstream, upstream: "http://10.0.3.7:8080/v1/orders"`. The access log shows `upstream_status` 502 with an upstream time of 0.001 s: the connection failed at once, so (a) is unlikely. Inside the container, `ss -ltnp` shows the application listening on `127.0.0.1:8080`, not `0.0.0.0:8080`. The application has no log line for any of the failed requests.
- *Root cause:* The application binds to loopback, which inside a container is reachable only from the container itself (Chapter 4, Section 4.6). The proxy's connection attempts reach the container's network namespace and are refused.
- *Fix:* Bind to `0.0.0.0` (or the container's interface).
- *Prevention:* A readiness check that the balancer or orchestrator performs **over the network**, as clients will, so an instance that cannot be reached never receives traffic.

**Lab 2: 504 on the monthly report, while the application "succeeds"**
- *Symptom:* Since the data grew, `GET /v1/reports/monthly` returns `504 Gateway Timeout` after exactly 60 seconds. The application log shows the same requests completing with `200` after about 75 seconds. Users retry, and the database's CPU climbs each time.
- *Hypotheses:* (a) the proxy's timeout is too short; (b) the report query has become slow; (c) retries are adding load.
- *Evidence:* The proxy access log: `status 504, upstream_response_time 60.000`. The application traces show 74 s spent in one SQL query. The database shows several identical copies of that query running at once, started a minute apart, matching user retries. The proxy's upstream read timeout is 60 s; the application has no request deadline; the database has no statement timeout.
- *Root cause:* A slow query (b) exceeding the proxy timeout, with a timeout chain in the wrong order (Section 10.10), so the abandoned work kept running while retries started more of it (c).
- *Fix:* Make the report asynchronous: `POST` creates a report job and returns `202 Accepted` with a status URL (Chapter 13), and a worker computes it (Chapter 25). Speed up the query (an index or a precomputed summary, Chapter 16). Set a database statement timeout and an application deadline below the proxy's. Raising the proxy timeout alone would only hide the problem.
- *Prevention:* Every synchronous endpoint has a latency budget below the proxy's timeout, and long work goes through jobs.

**Lab 3: Photo uploads fail with 413, and the application never sees them**
- *Symptom:* The mobile app's profile-photo upload fails for most users with `413`. The application's logs show no failed requests at all.
- *Hypotheses:* (a) the application rejects large files; (b) the proxy rejects them before forwarding; (c) the CDN or gateway has its own limit.
- *Evidence:* The error response is an HTML page with the proxy's `Server` header and no request ID, while the application's errors are JSON with a request ID. The proxy's error log: `client intended to send too large body: 3481203 bytes`. The proxy's body-size limit is at its default of 1 MB. The application's own multipart limit is also small (Spring Boot's default maximum file size is 1 MB at the time of writing), so raising only the proxy's limit would move the error into the application.
- *Root cause:* The proxy's request-body limit (b), below the size of modern phone photos, with a matching low limit in the application behind it.
- *Fix:* Decide the allowed size (say 10 MB), set it consistently at the CDN, gateway, proxy and application, and return a clear error above it. Better for large media: have the client upload directly to object storage with a short-lived signed URL, so the bytes never pass through the application servers.
- *Prevention:* Document size limits per route at every layer; test uploads at the limit and just above it in the pipeline.

**Lab 4: Everyone is rate-limited after adding a CDN**
- *Symptom:* After a CDN was placed in front of the load balancer, users are randomly rejected with `429 Too Many Requests`, and the access logs show only a handful of client IP addresses, all belonging to the CDN.
- *Hypotheses:* (a) an attack; (b) the rate limiter keys on an address that is now the CDN's; (c) a misconfigured CDN rule.
- *Evidence:* Request volume is normal. The application takes the client address from `X-Forwarded-For` by the right-to-left rule, trusting only the load balancer's range. A request's header reads `X-Forwarded-For: 203.0.113.7, 198.51.100.20`: the client, then the CDN edge. Since the CDN's range is not in the trusted list, the first untrusted address from the right is the CDN edge `198.51.100.20`, and thousands of users share each edge address, so they share a rate-limit bucket.
- *Root cause:* The trusted-proxy configuration was not updated for the new hop (b).
- *Fix:* Add the CDN's published address ranges to the trusted proxies, so that the next address to the left, the real client, is used. (Many CDNs also send their own client-IP header; trust it only on connections that come from the CDN's ranges.) Then verify that a request sent directly to the load balancer with a forged `X-Forwarded-For` still cannot choose its own address.
- *Prevention:* Treat every new proxy layer as a change to the client-IP logic, and test both the real-IP and the spoofing case.

---

## Solution Sketches

1. (a) Least outstanding requests, or power of two choices: costs vary 400-fold, so round robin would stack slow requests on some instances. (b) Consistent hashing on the tenant ID, so each tenant's data stays warm on one instance and adding or removing an instance remaps only a fraction of tenants; watch for one huge tenant and give it its own capacity. (c) Least connections at connect time, so new connections go to new instances; because existing connections stay where they are, add gradual rebalancing (ask a share of clients on busy instances to reconnect, with jitter).
2. The health check fails whenever any dependency fails, so during the downstream API's outage every instance failed its check, the balancer considered no target healthy, and it returned `503` for all requests, including those that never needed that API. Redesign: liveness checks only that the process is alive; readiness checks only the instance's own state (started, not draining, maybe its database pool initialised), not shared dependencies; handle dependency failures per request, with timeouts, fallbacks and circuit breakers (Chapter 24).
3. With `authenticate` first, unauthenticated floods reach authentication code before rate limiting; failed authentications get no request ID and are not logged by `logging`; errors thrown in `authenticate` and `rateLimit` are outside `errors`, so they surface as raw failures. Better: `[requestId, logging (timing), errors, rateLimit, authenticate, ...]`, so every response, including rejections, has an ID, is logged with its final status, and maps errors cleanly. (Some systems put a coarse IP-based rate limit before authentication and a per-user limit after it.)
4. The export runs on Node's single event-loop thread: building or serialising the CSV is CPU work that blocks the loop, so every other request on that process waits until it finishes. Fixes: stream the export (read rows with a cursor and write chunks, so the loop is released between chunks); move the CPU-heavy part to a worker thread or a separate service; or make the export an asynchronous job whose result is downloaded later.
5. Restarting an instance closes all its connections at once; all those clients reconnect immediately to the remaining instances, which perform thousands of TLS handshakes and WebSocket upgrades together (CPU spike), and some are refused (errors). Better: drain gradually, sending close frames with code `1001` to a fraction of connections at a time over several minutes; clients reconnect with randomised exponential backoff; keep spare capacity during rollouts; make clients resume from the last event they received.
6. The client gives up at 30 s, but the inner layers keep working for up to 60 s, 120 s or indefinitely; users (or client libraries) retry `POST /orders`, and both the original and the retry eventually commit, creating two orders. Fix: request timeouts that shrink inward (for example client 30 s, load balancer 28 s, proxy 27 s, application deadline 25 s, statement timeout below that), cancellation of work when the client disconnects, and idempotency keys on order creation so that a retry can never create a second order (Chapter 13).

---

**END OF CHAPTER 10**
