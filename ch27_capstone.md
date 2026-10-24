# Chapter 27: Capstone: End-to-End Request Lifecycle and Systematic Diagnosis

## Where We Are

Twenty-six chapters have built a backend from the bottom up. Part I explained the machine and the operating system: how a program runs (Chapter 2), processes, threads, scheduling and memory (Chapter 3), files, I/O and isolation (Chapter 4), and concurrency (Chapter 5). Part II covered networks and web protocols: networking (Chapter 6), HTTP (Chapter 7), cryptography and TLS (Chapter 8), and the browser (Chapter 9). Part III followed a request into the server and the application (Chapters 10–13). Part IV stored its data (Chapters 14–18). Part V gave it state and identity (Chapters 19–20). Part VI made it fast and distributed (Chapters 21–23). Part VII kept it running (Chapters 24–26).

This chapter teaches nothing new. Its job is **synthesis**: putting the pieces back together so that you can explain a whole system, trace any request through it, and diagnose failures that cross several layers at once. Every claim in it points back to the chapter that taught it. It answers the most famous backend interview question twice (in five minutes and in thirty), traces a write request through every layer, states a reusable diagnosis method, applies it to six case studies, walks through two timed system-design exercises, maps how every chapter connects, and ends with a 100-question readiness checklist and a plan for what to learn next.

**After this chapter you can:**
- answer "what happens when you type a URL and press Enter?" at two depths, naming the mechanism and the chapter behind every step;
- trace a write request (placing an order) from the browser through every layer to the database, the outbox and the asynchronous consumers, and say what each step guarantees and how it fails;
- apply one diagnosis method (symptom, scope, hypothesis, evidence, isolate the layer, fix, verify, prevent) to failures you have never seen;
- reason through six cross-layer incidents: tail latency, oversell, a login that fails only in production, partial failure in an asynchronous payment flow, a cascading outage, and stale reads after failover;
- run a timed system-design interview for a URL shortener and an order-and-payment system using the book's building blocks;
- see how all 27 chapters depend on each other, and assess your own readiness question by question.

**Interview questions this chapter prepares you for:**
1. What happens when you type a URL into the browser and press Enter?
2. Walk through what happens when a user places an order, from the click to the confirmation email.
3. Where can a request fail on its way through a system, and what does each failure look like to the caller?
4. How do you approach debugging a production problem you have never seen before?
5. A service's p99 latency doubled. How do you tell GC pauses, N+1 queries and connection-pool exhaustion apart?
6. How do you prevent two customers from buying the last item, and where exactly is the check made?
7. Login works on your laptop but fails in production. What are the likely causes, and how do you find the real one?
8. A payment succeeded but the order shows as failed. How can that happen, and how do you design against it?
9. How can one slow dependency take down a whole system, and how do you prevent it?
10. Users see old data after a database failover. Why, and what do you do about it?
11. Design a URL shortener.
12. Design an order and payment system.
13. How do the layers of a backend depend on each other?
14. Which guarantees does each layer provide, and which must the application provide for itself?

**Scope.** Synthesis only: the two-depth URL answer, the full write-request lifecycle, the diagnosis method, six case studies, two system-design walkthroughs, the concept map, the readiness checklist and next steps. No concept is introduced here; every mechanism is recalled from its home chapter, cited as "Chapter N, Section N.M" or "§N.M".

---

## 27.1 What Happens When You Type a URL and Press Enter

### Definitions

The **request lifecycle** is the complete sequence of steps from a user's action to the result appearing on their screen, across every component involved: browser, operating systems, network, edge, servers, application code, data stores, and back. **Synthesis**, in this chapter, means explaining that sequence as one connected story, in which each step's guarantees and failure modes come from a mechanism taught earlier.

### Why it matters

This question appears in backend, full-stack and systems interviews because it has no bottom. A weak answer lists five nouns ("DNS, TCP, HTTP, server, render"). A strong answer chooses a depth that fits the time available, names the mechanism at each step, and can go deeper anywhere the interviewer asks. Chapter 9 (Section 9.2) answered it from the browser's point of view; this section answers it across the whole book, at two depths.

### Concrete: the 5-minute interview answer

The user types `https://shop.example/products/17` and presses Enter.

1. **The browser decides what the input is.** It parses the URL (scheme, host, path, Chapter 7, Section 7.2), checks whether it must use HTTPS for this host (HSTS, §8.11), and checks its own HTTP cache for a fresh copy (§7.12). Assume a miss.
2. **DNS turns the name into an address.** The browser asks its cache, then the operating system's resolver, which asks a recursive resolver; on a miss, the resolver walks from the root to the `.example` servers to the domain's authoritative server, and caches the answer for its TTL (§6.6). The answer is usually a CDN edge address near the user.
3. **The browser opens a connection.** A TCP three-way handshake (§6.8), or QUIC over UDP for HTTP/3 (§6.10). Then a TLS 1.3 handshake: the browser sends the host name (SNI) and supported protocols (ALPN), the server proves its identity with a certificate chain the browser validates, and both derive session keys by ephemeral Diffie–Hellman, in one round trip (§8.7–8.9).
4. **The browser sends an HTTP request.** `GET /products/17` with headers, cookies for this site (§9.5) and accepted encodings, usually over HTTP/2 (§7.3, §7.10).
5. **The edge answers or forwards.** The CDN returns a cached copy if it has one; otherwise it forwards to the origin's load balancer, which picks a healthy instance; a reverse proxy in front of the application adds forwarding headers and a request ID (§10.1–10.3).
6. **The server runs the application.** The server accepts the connection, parses the request, and hands it to a thread or event loop (§10.5, §11.2). Middleware authenticates the session and checks limits; the router calls the handler; the handler reads the product from a cache or the database, through a connection pool, using an index (§10.6, §21.3, §18.3, §16.2).
7. **The response travels back.** The server serialises the response, compresses it, and sets caching headers (§12.7, §7.8, §7.12); the CDN may store it on the way back.
8. **The browser renders the page.** It parses HTML into the DOM and CSS into the CSSOM, runs scripts, computes layout and paints; each image, stylesheet and script is another request, mostly from cache or the CDN over the same connection (§9.3).

Five minutes is enough to say each step in two or three sentences and to offer: "I can go deeper into any of these."

### Concrete: the 30-minute deep version

The deep version follows the same eight steps, opening each one up to its mechanisms and its failure modes. Each line names what a strong candidate can explain on request.

**Step 1: the browser and the URL (Chapters 7, 8, 9)**
- URL parsing and normalisation; the browser's multi-process architecture: a network process does the fetching, a sandboxed renderer process per site does the rendering (§9.1).
- HSTS: if the host is on the preload list or was seen with `Strict-Transport-Security`, the browser never sends a plaintext request, which closes the SSL-stripping window of a first `http://` request (§8.11).
- The HTTP cache: a fresh response (within `max-age`) is used with no request at all; a stale one may be revalidated with `If-None-Match` and an `ETag`, returning `304 Not Modified` (§7.12–7.13). Many browsers also partition their caches by top-level site (§9.1).

**Step 2: DNS (Chapter 6)**
- The resolution chain: browser cache, OS stub resolver (`/etc/hosts`, `nsswitch`), recursive resolver, root, TLD and authoritative servers; iterative versus recursive queries; A and AAAA records; CNAMEs that point to a CDN; TTLs and negative caching (§6.6).
- Why "DNS propagation" is a misnomer: nothing propagates; caches expire.
- Failure modes and evidence: `NXDOMAIN` versus `SERVFAIL` versus a timeout, all visible with `dig` (§6.14); a low TTL makes failover faster and raises resolver load.

**Step 3: the connection (Chapters 4, 6, 8)**
- Underneath: the browser's process asks the kernel for a socket, the kernel picks an ephemeral source port, and the 4-tuple identifies the connection (§6.7, §4.6).
- The TCP handshake with sequence numbers, the server's listen backlog and accept queue (§6.8, §4.6), and what congestion control means for the first round trips of a new connection: slow start limits how much data the server can send before acknowledgements arrive, so many small round trips add latency (§6.8, §6.12).
- Packets cross the local network (ARP for the gateway's MAC address, §6.2), NAT at the home router (§6.5), and many IP routers that forward by longest-prefix match (§6.3).
- TLS 1.3: ClientHello with key shares, SNI and ALPN; ServerHello; the server's certificate chain and `CertificateVerify` signature; `Finished`; one round trip, or zero with resumption and its replay risk (§8.9). The browser validates the chain to a trusted root, the host name against the certificate, and the validity period (§8.8). What HTTPS still exposes: IP addresses and, without ECH, the host name (§8.12).
- With HTTP/3, QUIC combines the transport and TLS handshakes and avoids TCP's head-of-line blocking (§6.10, §7.10).

**Step 4: the HTTP request (Chapters 7, 9, 19)**
- The message format: method, target, headers, body, and how the receiver finds the end of each (§7.3); in HTTP/2, binary frames on multiplexed streams with compressed headers (§7.10).
- Which cookies are sent: domain and path match, `Secure`, `SameSite`, and expiry rules (§9.5), each a security control (§19.2).
- Connection reuse: later requests to the same origin use the same connection (keep-alive, §7.9; HTTP/2 multiplexing, §7.10).

**Step 5: the edge (Chapters 10, 21, 24)**
- The CDN point of presence: TLS terminated at the edge, cache key built from the URL and `Vary`, hit or miss, origin shielding; the WAF inspects requests; DDoS protection absorbs floods (§10.1, §21.2).
- The load balancer: L4 versus L7, the algorithm (least outstanding requests, power of two choices), health checks that only route to ready instances (§10.3, §6.13).
- The reverse proxy: connection pooling to the backends, buffering, `X-Forwarded-For` and `X-Forwarded-Proto`, request IDs, and the timeout chain that must shrink inward (§10.2, §10.10).
- Failure modes and who produces them: `502` (backend refused or reset), `503` (no healthy target, or shedding), `504` (backend too slow), each traceable in the proxy's logs (§10.8, §10.11).

**Step 6: the server and the application (Chapters 3, 4, 5, 10, 11, 12, 18, 16, 15, 21)**
- The server process: an acceptor thread or event loop takes connections from the kernel's accept queue (§10.5), `epoll` reports which sockets are readable (§4.5), the parser reads the request with size and time limits (§10.5, §7.15).
- The execution model: a thread from a pool, a virtual thread, or an event-loop callback (§11.2, §5.13); the request's context (user, trace ID, deadline) travels with it (§11.6).
- Middleware, in onion order: logging and tracing, rate limiting, session lookup or token validation, authorisation, then routing to the handler (§10.6, §19.3, §20.5, §20.9).
- The application: binding and validation, the service layer, the repository (§12.2, §12.6). For a product page, a cache-aside read: Redis first, the database on a miss (§21.3).
- The database call: a connection borrowed from the pool (§18.3), a prepared statement with bound parameters (§18.6), the planner choosing an index scan (§16.9), B+tree descent through pages that are, with luck, in the buffer pool (§16.2, §15.3); otherwise the storage device reads them (§2.4). PostgreSQL serves the connection with a dedicated backend process, MySQL with a thread (§15.1).
- Underneath it all: system calls, context switches and the scheduler (§3.2, §3.11, §3.10), virtual memory and the page cache (§3.13, §4.3), and the runtime's garbage collector (§2.8).

**Step 7: the response (Chapters 7, 12, 21)**
- Serialisation to JSON or HTML (§12.7), compression negotiated by `Accept-Encoding` (§7.8), `Cache-Control` and `ETag` so that the browser and the CDN can reuse it (§7.12), and security headers such as CSP and `X-Content-Type-Options` (§9.8–9.9).
- On the way out: the server writes to the socket's send buffer; the kernel and TCP deliver it, retransmitting lost segments (§6.8); the proxy and the CDN pass it on and may cache it.

**Step 8: rendering (Chapter 9)**
- The renderer parses HTML incrementally, discovers subresources, blocks on synchronous scripts and render-blocking CSS, builds the DOM and CSSOM, runs layout and paint, and composites layers on the GPU (§9.3).
- Scripts then call APIs with `fetch`; cross-origin calls go through CORS with possible preflights (§9.4, §9.7).
- Whether the HTML arrived complete (server-side rendering) or as a shell filled by JavaScript (client-side rendering) changes what the backend serves and caches (§9.10).

**Throughout: what an interviewer may probe**
- Latency budget: one round trip to a nearby CDN takes a few milliseconds, a cross-continent round trip about 100–150 ms; DNS, TCP and TLS each add round trips on a cold connection, which is why connection reuse, HTTP/2 and edge termination matter (§6.12, §2.2's latency numbers).
- Where it can fail and how each failure looks (Section 27.2's step table and Section 27.3).

### Guarantees and non-guarantees

Each step's guarantee is local: DNS returns an answer (possibly stale), TCP delivers bytes in order (not messages, and not "processed", §6.9), TLS protects bytes between its two endpoints (§8.12), HTTP defines the meaning of the exchange (§7.1), and the application decides correctness. No step guarantees the next one: a page can arrive over a perfectly secure connection and still be wrong, stale, or someone else's data from a misconfigured cache (§19.2).

### Failure and diagnosis

The same journey is the map for diagnosis: a failure belongs to one step, and each step has its tool. Name resolution: `dig` (§6.14). Connection: `ss`, `tcpdump`, connect versus read timeouts (§6.14). TLS: `openssl s_client` (§8.13). HTTP: `curl -v` (§7.16) and DevTools (§9.11). Edge and proxy: their logs and the request ID (§10.11). Application: traces, metrics, logs, thread dumps (§26.6–26.8, §11.10). Database: `EXPLAIN ANALYZE`, lock and activity views (§16.12, §17.14, §18.17). Section 27.3 turns this into a method.

### Backend connection

Being able to tell this story at any depth is the clearest evidence that the book's model has become yours. Practise both versions aloud: the five-minute one until it is smooth, and the deep one by letting someone interrupt you at random steps and asking for the mechanism and the failure mode there.

---

## 27.2 The Full Lifecycle of a Write Request: Placing an Order

### Definitions

A **write request** changes the system of record (§1.6): here, `POST /api/orders` creates an order, reserves stock and starts payment. Unlike the read of Section 27.1, it must be **correct under concurrency, retries and partial failure**: it must not oversell, must not charge twice, and must not lose the events that drive everything after it.

### Why it matters

Chapter 1 (Section 1.10) traced "click Buy" at the level of components and promised the details. This section delivers them, one layer at a time, with the chapter that explains each step. Most of the book's hardest material (idempotency, transactions, isolation, outbox, sagas, resilience) exists because of this one request.

### Concrete: the sequence diagram

```text
 BROWSER          EDGE / LB / PROXY       ORDERS SERVICE (app)          REDIS      POSTGRESQL (primary   BROKER      CONSUMERS
 (SPA, §9.4)      (§10.1–10.3)            (§10.5–10.6, §11–12)          (§21)      + sync standby, §22)  (§25.4)     (§25)
    │ click "Place order"
    │ fetch POST /api/orders  (reuses TLS connection, HTTP/2)
    │   Cookie: __Host-sid=…   X-CSRF-Token: k29…   Idempotency-Key: 7f3c-…
    │──────────────────────────►│ WAF checks; POST not cached; LB picks a ready instance
    │                           │ proxy: X-Forwarded-For, request ID, timeout 9 s, traceparent
    │                           │──────────────────────────►│ accept, parse (limits), worker / virtual thread
    │                           │                           │ middleware: trace span, rate limit ───► INCR rl:…  (§24.6)
    │                           │                           │ session lookup ────────────────────────► GET sess:… (§19.3)
    │                           │                           │ CSRF / Origin check (§19.6); authorise user 81 (§20.9)
    │                           │                           │ validate body → DTO → domain command (§12.3, §12.6)
    │                           │                           │ idempotency: INSERT key (7f3c-…, IN_PROGRESS) ─────────────────►│ (§13.5)
    │                           │                           │ prices from cache (display only) ─────► GET product:17 (§21.3)
    │                           │                           │ borrow DB connection (pool, 0.3 s timeout, §18.3)
    │                           │                           │ BEGIN ─────────────────────────────────────────────────────────►│
    │                           │                           │ UPDATE stock SET qty = qty − 1 WHERE sku = 17 AND qty ≥ 1 ──────►│ 1 row (§17.11)
    │                           │                           │ INSERT orders (PENDING), order_lines ──────────────────────────►│ constraints (§14.3)
    │                           │                           │ INSERT outbox ("OrderPlaced", id = message ID) ─────────────────►│ (§25.8)
    │                           │                           │ UPDATE idempotency key → result (201, order 9001) ──────────────►│
    │                           │                           │ COMMIT ────────────────────────────────────────────────────────►│ WAL fsync (§15.4)
    │                           │                           │                                           │ standby flushed (§22.3)
    │                           │                           │◄───────────────────────────────────────── │ commit acknowledged
    │                           │                           │ return connection to pool; serialise 201 + Location; log; metrics
    │                           │◄──────────────────────────│ 201 Created  {order 9001, status PENDING}
    │◄──────────────────────────│ (proxy logs status, duration, upstream time, request ID)
    │ show "Order received; payment processing"
    │                                                       OUTBOX RELAY (separate process)
    │                                                       reads unpublished rows ───────────────────────►│ publish key=9001
    │                                                       marks published                                 │──► acks=all ──►│
    │                                                                                                       │  orchestrator: saga step 1
    │                                                                                                       │  → payments (timeout, retry, breaker, §24)
    │                                                                                                       │  → inventory confirm, capture (pivot, §25.9)
    │                                                                                                       │  email consumer: inbox dedupe (§25.6)
    │                                                                                                       │  search index via CDC (§22.10)
    │                                                                                                       │  cache invalidation (§21.5)
    │ later: page polls /orders/9001 or receives a push; shows CONFIRMED
```

### Concrete: the step table

| # | Step | What happens | What it guarantees | How it fails, and the evidence | Chapter |
|---|---|---|---|---|---|
| 1 | Click to `fetch` | the page's script builds a JSON body and a cross-origin-safe request on the same site | nothing yet; the browser attaches cookies by its rules | cookie missing (cross-site, `SameSite`), CORS preflight refused: DevTools | §9.4, §9.5, §9.7 |
| 2 | Connection reuse | the existing TLS connection carries the request over HTTP/2 | confidentiality and integrity in transit | a reset connection; retries are safe only with the idempotency key | §7.10, §8.9 |
| 3 | Edge, WAF, load balancer | `POST` bypasses the CDN cache; the balancer chooses a ready instance | routing to instances that pass readiness | `502`/`503`/`504` generated here; proxy logs | §10.1, §10.3, §10.8 |
| 4 | Reverse proxy | forwarding headers, request ID, trace context, upstream timeout | a real answer before the client gives up, if timeouts nest | `499` then late `200`: inner timeouts too long | §10.2, §10.10, §26.8 |
| 5 | Accept and parse | the server reads the request under size and time limits | malformed or oversized requests rejected early | slow clients holding workers; `413`, `431` | §10.5, §7.15 |
| 6 | Execution model | a worker thread, virtual thread or event-loop task runs the request | bounded concurrency if limits are set | all workers blocked on a slow call: thread dumps | §11.2, §11.10, §24.5 |
| 7 | Rate limit | a token bucket per user in Redis, checked atomically | each user within their share | Redis down: fail open or closed, by design | §24.6, §21.8 |
| 8 | Session and CSRF | session found by hashed ID in Redis; CSRF token or `Origin` verified | the request comes from user 81's own page | random logouts (session store eviction), `403` CSRF mismatch | §19.3, §19.6, §19.8 |
| 9 | Authorisation | the user may create orders; later reads scoped by owner | object-level access control | IDOR if a later endpoint skips the scoping | §20.9 |
| 10 | Validation and mapping | body bound to a DTO, validated, mapped to a domain command | only well-formed, permitted fields reach the domain | mass assignment if entities are bound directly | §12.3, §12.6 |
| 11 | Idempotency key | the key is claimed in the database, or the stored result returned | a retried request does not create a second order | two concurrent retries: the unique key makes one wait | §13.5 |
| 12 | Cached prices | prices read from Redis for display; the order recomputes from the database | fast reads; decisions never made from cached data | stale price shown: the order still uses the database value | §21.3, §21.5 |
| 13 | Borrow a connection | the pool hands out a connection, or waits up to its acquisition timeout | bounded database concurrency | pool exhaustion: pending borrowers, acquisition timeouts | §18.3, §18.4 |
| 14 | Transaction: stock | conditional `UPDATE … WHERE qty ≥ 1`, checked by affected rows | no oversell, even at Read Committed | zero rows updated → "out of stock" answer | §17.11, §17.13 |
| 15 | Transaction: order rows | inserts with primary, foreign and unique keys | integrity enforced by the database | constraint violation → rollback; `409` or `422` | §14.2, §14.3 |
| 16 | Transaction: outbox | the event row committed with the order | the event exists if and only if the order does | no outbox row = a dual write somewhere (Diagnosis Lab 4 of Ch 25) | §25.8 |
| 17 | Commit | WAL flushed locally and on the synchronous standby | durability that survives the loss of the primary | commit latency = fsync plus standby round trip | §15.4, §22.3 |
| 18 | Release and respond | connection returned; `201 Created` with `Location`; logs, metrics, span | the client learns the outcome | a lost response leaves the outcome unknown to the client, so it retries with the same key | §13.4, §7.15, §26.6 |
| 19 | Outbox relay | unpublished rows published to the broker with `acks=all`, then marked | at-least-once publication, in per-order key order | relay down: outbox age grows; alert | §25.8, §25.4 |
| 20 | Saga | the orchestrator authorises payment, confirms stock, captures (the pivot), confirms | eventual consistent outcome or compensation | stuck sagas, step timeouts, compensation failures | §25.9, §25.12 |
| 21 | Payment provider call | timeout, retry with backoff and an idempotency key, behind a breaker and bulkhead | no double charge; bounded damage from a slow provider | a slow provider holding threads, without these protections | §24.2–24.5 |
| 22 | Provider webhook | the provider's signed callback is verified with HMAC and processed idempotently | authentic, deduplicated confirmation | forged or replayed webhook without verification | §8.5, §13.10 |
| 23 | Consumers | email (inbox dedupe), search index (CDC), cache invalidation, analytics | each effect once, eventually | lag, duplicates, poison messages: lag graphs, DLQs | §25.6, §25.7, §25.11 |
| 24 | Client sees the result | polling the status resource, or a push over SSE/WebSocket | read-your-writes for the user's own order if read from the primary | "order not found" from a lagging replica | §13.10, §10.9, §22.4 |

### Mechanism: what the design of each step protects against

Reading the table by column gives the book's main lessons in one place:

- **Concurrency** is handled where the data is: a conditional update and constraints in the database (steps 14–15), not locks in application code, which do not span instances (§11.7).
- **Retries** are safe because every step with an effect is idempotent: the API key (step 11), the provider's key (step 21), the consumers' inboxes (step 23), and webhook deduplication (step 22).
- **Partial failure** is handled by moving everything that can fail independently behind the outbox (step 16) and the saga (step 20), so that the user's request depends only on one database transaction.
- **Slowness** is bounded by nested timeouts (step 4), pool acquisition timeouts (step 13), bulkheads and breakers (step 21).
- **Truth** lives in one place: the database's committed state. Caches (step 12), search indexes and read models (step 23) are derived and may lag.

### Guarantees and non-guarantees

This design guarantees that an accepted order is recorded exactly once, never oversells, produces its event, and ends confirmed or cleanly compensated. It does not guarantee instant confirmation, that the user never sees "processing", or that derived views (search, cached lists, replicas) are current at every moment.

### Failure and diagnosis

Every row's "how it fails" column is a diagnosis entry point. When an order goes wrong, find its request ID or order ID, follow the trace across services (§26.8), check the database rows (order, outbox, idempotency key), the broker (published? consumed? in a DLQ?) and the saga state, in that order. Each of those artefacts exists precisely so that this question has an answer.

### Backend connection

When you design any write path, draw this table for it: each step, what it guarantees, how it fails, and how you would see the failure. If a row has no guarantee, no failure handling, or no evidence, that row is where the next incident will come from.

---

## 27.3 A Reusable Diagnosis Method

### Definitions

**Systematic diagnosis** is finding the cause of a failure by a repeatable sequence of steps, in which every conclusion rests on evidence that could have proved it wrong. The method used throughout this book's Diagnosis Labs has eight steps:

1. **Symptom**: what is observed, stated precisely (what, how much, since when, measured where).
2. **Scope**: who and what is affected, and who is not (which endpoints, users, regions, instances, versions).
3. **Hypotheses**: several candidate causes, each tied to a mechanism.
4. **Evidence**: for each hypothesis, an observation that would confirm or refute it, gathered with the right tool.
5. **Isolate the layer**: narrow the failure to one layer and one component.
6. **Fix**: mitigate first if users are hurting, then correct the cause.
7. **Verify**: show that the symptom is gone because of the fix, not by coincidence.
8. **Prevent**: change the system (tests, alerts, limits, designs) so that this class of failure cannot recur silently.

### Why it exists

Production failures rarely announce their layer. A timeout can be DNS, a full pool, a lock, a slow disk, a garbage-collection pause or a dependency three services away. Without a method, engineers chase the first plausible idea, change several things at once, and declare victory when the graph improves, often for an unrelated reason. The method replaces intuition with a short chain of falsifiable steps, and the book's chapters supply the mechanisms that generate the hypotheses and the tools that produce the evidence.

### Mechanism: the eight steps, with the questions each asks

```text
 SYMPTOM     "p99 of POST /orders rose from 300 ms to 2.4 s at 14:05, measured at the load balancer"
             not "the site is slow"; use SLIs and percentiles (§26.9, §26.11)
 SCOPE       which routes / regions / instances / versions / customers? what changed at 14:05?
             (deploy and config annotations, traffic, dependencies: §26.4–26.5, §26.13)
             what is NOT affected? (often the most informative fact)
 HYPOTHESES  at least three, from different layers; each a mechanism, not a guess:
             "pool exhaustion: hold time rose, so pending borrowers queue" (§18.3, §11.3)
 EVIDENCE    for each: the observation that separates it from the others, and the tool
             "if pool exhaustion: pending > 0, acquisition timeouts, threads parked in borrow"
 ISOLATE     follow the evidence to one layer: RED per service and dependency, then USE for the
             resource where time originates (§26.10); traces show the span (§26.8)
 FIX         mitigate first (roll back, kill switch, shed, fail over: §26.13, §24.9), preserving
             evidence; then the real correction
 VERIFY      the symptom metric returns to baseline AFTER the fix and the mechanism's evidence
             disappears (pending borrowers back to zero), ideally with the cause reproduced in a test
 PREVENT     a test, an alert on the leading indicator, a limit, a design change; a postmortem
             action with an owner (§26.13)
```

### Concrete: where the evidence comes from, layer by layer

| Layer | Typical symptoms | First evidence and tools | Chapter |
|---|---|---|---|
| Browser | CORS errors, missing cookies, slow rendering | DevTools network panel, cookie blocking reasons, console | §9.11, §19.8 |
| DNS | "unknown host" errors, `NXDOMAIN`, slow first requests | `dig`, resolver timing | §6.6, §6.14 |
| TCP and network | connect timeouts, resets, refused connections | `ss`, `tcpdump`, retransmission counters, `traceroute` | §6.14 |
| TLS | handshake failures, certificate errors | `openssl s_client`, error codes | §8.13 |
| HTTP | wrong status codes, framing problems | `curl -v`, raw bytes | §7.16 |
| Edge and proxy | `502`, `503`, `504`, `499` | proxy access and error logs with request IDs | §10.8, §10.11 |
| Server and concurrency | latency with idle CPU, stuck workers | thread dumps, pool metrics, Little's Law | §11.10, §11.3 |
| Operating system | OOM kills, throttling, file-descriptor exhaustion, I/O waits | `dmesg`, `cpu.stat`, `lsof`, `iostat`, `strace` | §3.19, §4.12 |
| Runtime | GC pauses, heap growth | GC logs, heap profiles | §2.8, §26.12 |
| Application | errors, slow code paths | logs by trace ID, CPU flame graphs | §26.6, §26.12 |
| Database queries | slow queries, wrong plans | `EXPLAIN ANALYZE`, `pg_stat_statements` | §16.12, §18.17 |
| Database concurrency | lock waits, deadlocks, serialisation failures | `pg_locks`, `pg_blocking_pids`, deadlock logs | §17.14 |
| Storage | commit latency, checkpoints, disk full | wait events, checkpoint logs, slot retention | §15.10 |
| Cache | low hit ratio, stale data, hot keys | `INFO stats`, `SLOWLOG`, `--hotkeys` | §21.10 |
| Replication | stale reads, lag, failover issues | `pg_stat_replication`, replica status | §22.11 |
| Sharded and coordinated systems | one hot shard, fan-out latency, lock conflicts | per-shard metrics, fan-out traces, clock offsets | §23.12 |
| Resilience mechanisms | breaker trips, retries, shedding | breaker state, retry ratio, rejection reasons | §24.13 |
| Messaging | lag, duplicates, stuck messages | consumer lag, DLQs, outbox age | §25.13 |
| Whole system | SLO burn, user-visible errors | RED/USE dashboards, traces, alerts | §26.10, §26.11 |

### Guarantees and non-guarantees

The method guarantees that each conclusion rests on evidence that could have refuted it, and that the fix is verified against the mechanism, not just the graph. It does not guarantee a fast answer when the system emits too little evidence (which is an observability problem, §26.6), and it does not replace the knowledge needed to form good hypotheses: that is what the other 26 chapters are for.

### Failure and diagnosis

The method itself has failure modes. **Anchoring**: the first hypothesis is the only one investigated. **Changing several things at once**, so that nobody knows which change worked. **Confusing correlation with cause**: the graph improved after a restart that merely cleared the symptom (a memory leak returns in six hours). **Fixing the symptom only**: raising a pool size that was saturated because queries became slow. Each is avoided by the steps above: several hypotheses, one change at a time, and verification against the mechanism.

### Backend connection

Write down symptom, scope and hypotheses at the start of every investigation, even a small one; it takes two minutes and prevents hours of wandering. Build systems that make step 4 cheap: request IDs, traces, per-dependency metrics, saturation gauges, and deploy annotations are evidence collected in advance.

---

## 27.4 Case Studies Across the Layers

Each case follows the method of Section 27.3 and requires at least three chapters to solve.

### Case 1: A p99 latency spike: GC pauses, N+1 queries or pool exhaustion?

- *Symptom:* The p99 of `GET /customers/{id}/orders` rose from 250 ms to 1.9 s over two days; the p50 rose only from 40 to 55 ms (§26.9). Error rate is flat.
- *Scope:* One endpoint, all instances, all regions. The other endpoints of the same service are unaffected. A release went out two days ago; traffic is up 10% (seasonal).
- *Hypotheses (each a mechanism, each with a distinguishing signature):*

| Hypothesis | Mechanism | Signature that separates it |
|---|---|---|
| (a) GC pauses | stop-the-world collections freeze all threads (§2.8) | **all** endpoints of the instance slow at the same moments; GC log pauses line up with slow requests; heap occupancy high after collections |
| (b) N+1 queries | the ORM lazily loads each order's lines, one query per order (§18.10–18.11) | slowness grows with the number of orders per customer; traces show a "comb" of many small identical queries; `pg_stat_statements` shows a query whose calls per request are far above 1 |
| (c) pool exhaustion | requests hold connections longer, so borrowers queue (§18.3, §11.3) | pool pending count > 0 and acquisition time rising; thread dumps show threads parked in the pool's borrow; slowness affects every endpoint that uses the database |

- *Evidence:* GC logs show pauses of 15–30 ms, unchanged and spread evenly, and other endpoints are unaffected, which rules out (a). Pool metrics show occasional pending borrowers, but only during the slow requests themselves, and acquisition time is a small share of request time: a consequence, not the cause, so (c) is secondary. Tail-sampled traces of slow requests (§26.8) show 1 query for orders followed by up to 300 queries for order lines, 3–5 ms each. Slow requests belong to customers with long histories; the release changed the DTO mapping to include line items, touching a lazy collection inside a loop.
- *Root cause:* (b): an N+1 introduced by a mapping change; its cost scales with history length, so it shows in the tail, not the median. Each slow request also holds its connection for the full 1–2 s, which created the secondary pool pressure.
- *Fix:* Load lines with a join fetch or a single batched query (`WHERE order_id = ANY($1)`), page the order list (keyset pagination, §13.6), and return a summary DTO without lines in the list view.
- *Verification:* Queries per request fall to 2; p99 returns to 240 ms; pool pending returns to zero.
- *Prevention:* A query-count assertion in integration tests for list endpoints (§18.11); a metric of queries per request; tail-based trace sampling kept on.
- *Chapters combined:* 2 (GC), 11 (Little's Law and pools), 13 (pagination), 18 (ORM, N+1, pools), 26 (percentiles, traces).

### Case 2: Inventory oversell under concurrency

- *Symptom:* During a flash sale of 500 units, 537 orders were confirmed. Warehouse staff report negative stock.
- *Scope:* Only the flash-sale product; only during the first two minutes; orders from all instances.
- *Hypotheses:* (a) a read-modify-write race in the application (§17.3's lost update); (b) the stock check made from cached data (§21.5); (c) duplicate orders from client retries (§13.5); (d) an in-process lock assumed to protect stock across instances (§11.7).
- *Evidence:* Order IDs and idempotency keys are all distinct, and no customer has two orders with one key, ruling out (c). The code reads `stock` with `SELECT`, checks `stock >= qty` in the application, then writes `UPDATE … SET stock = $new`, inside a transaction at Read Committed; an in-process lock (a mutex, §5.5) wraps it, which serialises requests only within one of 12 instances, so (a) and (d) are the same bug. Logs show the availability badge reading stock from Redis, but the order path reads the database, so (b) explains a misleading display, not the oversell. Reproducing with 50 concurrent requests in a test environment confirms lost updates: two transactions read 3, both write 2.
- *Root cause:* (a)+(d): a lost update between instances. Read Committed does not prevent it in PostgreSQL or InnoDB, and the application lock does not span instances (§17.3–17.4, §11.7).
- *Fix:* Make the decision atomic in the database: `UPDATE stock SET qty = qty − $n WHERE sku = $sku AND qty >= $n` and check the affected-row count (§17.11), or `SELECT … FOR UPDATE`, or a `CHECK (qty >= 0)` constraint as a backstop (§17.13). Remove the application lock. Keep Redis for display only, labelled as approximate.
- *Verification:* The concurrent test now confirms exactly 500 orders and rejects the rest with a clear "sold out".
- *Prevention:* A concurrency test in the pipeline for every invariant-protecting write; the database constraint as a guard; a review rule that decisions are never made from cached data (§21.5).
- *Chapters combined:* 11 (coordination across instances), 13 (idempotency), 17 (anomalies, isolation, atomic operations), 21 (cache consistency).

### Case 3: Login works locally, fails in production

- *Symptom:* After deploying a new single-page front end, users can submit the login form and see "welcome", but the next request shows them logged out. Locally, everything works.
- *Scope:* Every browser in production; staging (on a different domain setup) works; local development (front end and API both on `localhost`) works.
- *Hypotheses:* (a) the session cookie is rejected or not sent (attributes, cross-site rules, §9.5, §19.2); (b) CORS with credentials misconfigured (§9.7); (c) the application believes requests are plain HTTP behind the TLS-terminating proxy, so it sets cookies or redirects wrongly (§10.2, §8.10); (d) sessions not shared between instances (§19.4).
- *Evidence:* In DevTools (§9.11, §19.8), the login response carries `Set-Cookie: SESSION=…; Secure; HttpOnly; SameSite=Lax`, and the cookie is stored. The front end is served from `app.shop.example` and calls `https://api.shop-platform.net` (a different registrable domain), so the next `fetch` is a cross-site request: the browser does not send a `SameSite=Lax` cookie on it. Locally both were on `localhost`, the same site. Separately, the server log shows `scheme=http` for every request, and the framework's redirect after login builds `http://` URLs. The application trusts no forwarding headers, so it does not know the proxy terminated TLS. Sessions are stored in Redis, ruling out (d).
- *Root cause:* (a), with (c) as a second, independent defect: a cross-site API domain made the session cookie unsendable under `SameSite=Lax`, and the missing trust of `X-Forwarded-Proto` produced HTTP redirects (which HSTS then upgraded, hiding the defect in most browsers). Making the cookie `SameSite=None` would only trade this for third-party-cookie blocking in many browsers (§19.2) and open CSRF (§19.6).
- *Fix:* Serve the API on the same site (`api.shop.example`) or route API calls through the front end's origin (`/api/…` via the reverse proxy); keep `SameSite=Lax`, `Secure`, `HttpOnly`, a `__Host-` prefix; configure CORS for the exact origin with credentials only if a cross-origin call remains (§9.7); configure the application to trust forwarding headers from the proxy only (§10.2).
- *Verification:* Login followed by API calls succeeds in each major browser with default privacy settings; logs show `scheme=https`.
- *Prevention:* An end-to-end test that runs against the production domain layout; a staging environment whose domains mirror production's site boundaries.
- *Chapters combined:* 8 (TLS termination), 9 (site versus origin, CORS, DevTools), 10 (proxies and forwarding headers), 19 (cookies, sessions, CSRF).

### Case 4: Partial failure in an asynchronous payment workflow

- *Symptom:* Some customers are charged, but their orders show "payment failed" and are cancelled. Support refunds them manually.
- *Scope:* About 0.3% of orders, concentrated in periods when the payment provider is slow; only card payments.
- *Hypotheses:* (a) a timeout treated as a failure while the charge actually succeeded (§7.15, §24.1); (b) a retry creating a second charge on a cancelled order (§24.3); (c) the provider's success webhook lost or rejected (§13.10); (d) a saga compensation that cannot reverse a capture (§25.9).
- *Evidence:* The provider's dashboard shows one successful charge per affected order, so (b) is ruled out. The payments service log shows `authorize` calls timing out at 2 s, then the saga marking the step failed and compensating (cancelling the order) immediately. The provider's logs show those authorisations completing at 2.3–3.5 s. The success webhooks arrived seconds later and were rejected with `409 Conflict` because the order was already cancelled. So (c) contributed, though the webhook was not lost but refused. The authorisation was never voided, and was later captured by a nightly batch that captured all open authorisations.
- *Root cause:* (a): an unknown outcome treated as a failure. The saga compensated without first establishing what happened, and the webhook handler did not handle "success for a cancelled order" as a case requiring compensation (§25.12's late-reply handling).
- *Fix:* On timeout, mark the step `UNKNOWN`, query the provider's status by the idempotency key, and decide only then (§24.1, §25.9's failure matrix); handle late success webhooks for cancelled orders by voiding or refunding automatically; make the nightly capture job capture only authorisations whose orders are confirmed; derive the provider timeout from its measured p99.9 (§24.2).
- *Verification:* A failure-injection test that delays provider responses beyond the timeout (§24.12) ends every order consistently: confirmed and charged, or cancelled and not charged.
- *Prevention:* The saga's failure matrix reviewed for every step's "unknown" outcome; a reconciliation job comparing provider charges with order states daily, alerting on mismatches.
- *Chapters combined:* 7 (unknown outcomes), 13 (webhooks, idempotency keys), 24 (timeouts, failure injection), 25 (sagas, late replies, reconciliation).

### Case 5: A cascading outage from one slow dependency

- *Symptom:* At 09:12, every endpoint of the public API returns errors or times out. The incident lasts 40 minutes, although the trigger was fixed at 09:20.
- *Scope:* All endpoints and regions. The database is healthy; CPU on API instances is low.
- *Hypotheses:* (a) a network or load-balancer problem (§10.3); (b) a shared dependency slowing and consuming all worker threads (§24.1, §11.3); (c) a database problem hidden behind pool metrics (§18.3).
- *Evidence:* Thread dumps (§11.10) show nearly all workers blocked in the HTTP client calling the token-introspection endpoint of the identity provider, which every request uses to validate opaque access tokens (§20.5). Its latency rose from 20 ms to 6 s at 09:12 after a bad deploy of the identity provider. The API's introspection client had a 10 s timeout, no bulkhead, no breaker, and no cache of introspection results. Little's Law: 3,000 requests/s × 6 s = 18,000 threads needed, against 4,000 available. The load balancer's health check called `/health`, which validated a test token: instances failed health checks and were removed, and the remainder overloaded faster (§24.8's death spiral). Mobile clients retried three times with no jitter (§24.3). After the identity fix at 09:20, retry traffic at 3× normal and cold caches kept the system down until load was cut at the edge at 09:45: a metastable failure (§11.8).
- *Root cause:* (b): an unprotected synchronous dependency on the hot path of every request, amplified by deep health checks and synchronised retries.
- *Fix (now):* Shed load at the edge, disable client retries via configuration, restore traffic gradually (§24.8). *Fix (lasting):* cache introspection results for a short TTL bounded by token lifetime, or move to self-contained short-lived JWTs validated locally with a cached key set (§20.5, §20.6); introspection timeout of 300 ms with a bulkhead and a breaker (§24.2–24.5); local health checks (§24.10); retries with jitter and a budget (§24.3); a fail-closed policy for authentication when introspection is unavailable, with cached results honoured until expiry.
- *Verification:* A game day injecting 5 s of latency into introspection (§24.12): the API serves cached validations and fails new tokens fast; no instance is removed.
- *Prevention:* A dependency review listing every synchronous call on the hot path with its timeout, bulkhead, breaker and fallback; burn-rate alerts (§26.11).
- *Chapters combined:* 10 (health checks), 11 (Little's Law, metastability), 20 (token validation), 24 (timeouts, bulkheads, breakers, retries, cascades).

### Case 6: Stale reads after a database failover

- *Symptom:* After an automatic failover at 03:10, customers report that recent profile changes "reverted"; a few orders placed just before 03:10 have disappeared; and for 20 minutes afterwards some users see their own new orders missing from their order list.
- *Scope:* Profile changes and orders in the 10 seconds before the failover; the order-list symptom for users served by read replicas.
- *Hypotheses:* (a) asynchronous replication lost acknowledged commits at failover (§22.3, §22.5); (b) read replicas lagging behind the new primary after the topology change (§22.4, §18.16); (c) caches holding values the database no longer has (§21.5).
- *Evidence:* The old primary's WAL, recovered after the incident, contains commits after the last position the promoted standby had received, which confirms (a): replication to that standby was asynchronous. Some profile values shown to users match those lost commits: they were cached in Redis by read paths before the failover, and expired at different times over the next hour, so the "reverts" were caches expiring onto the older database state, confirming (c). The order-list symptom matches replica lag: after failover, the replicas re-synchronised from the new primary and lagged by up to 90 s, and the application routed list reads to replicas without read-your-writes protection, confirming (b).
- *Root cause:* All three, compounding: an asynchronous failover's loss window (a), caches serving values from the lost history (c), and replicas lagging after the topology change with no read-your-writes routing (b).
- *Fix:* Recover the lost transactions from the old primary's WAL and reconcile (§22.5); flush caches for keys written in the loss window (or all, with a warm-up plan, §21.6); route a user's reads to the primary for a short time after their writes, or wait for the replica to reach the write's log position (§22.4); make at least one standby synchronous (or quorum) in another zone for the order database (§22.3).
- *Verification:* A failover drill under write load: no acknowledged order is lost; users see their writes immediately after.
- *Prevention:* Documented loss windows per database; cache invalidation tied to failover events; replica lag alerts; regular failover drills (§24.11).
- *Chapters combined:* 18 (read/write splitting), 21 (cache consistency), 22 (synchronous replication, failover, lag anomalies), 24 (redundancy and drills).

---

## 27.5 Two Timed System-Design Walkthroughs

### Definitions

A **system-design interview** asks a candidate to design a system from vague requirements in 35–45 minutes, explaining trade-offs aloud. It tests whether the building blocks of this book can be combined under time pressure. A workable structure, used in both walkthroughs below:

```text
 1. requirements      (5 min)   functional, non-functional, explicit out-of-scope
 2. estimates         (5 min)   traffic, storage, read/write ratio; numbers that decide the design
 3. API               (5 min)   endpoints, methods, status codes, idempotency (Chapter 13)
 4. data model        (5 min)   entities, keys, access patterns, the store (Chapters 14, 23)
 5. high-level design (10 min)  components and request paths
 6. deep dives        (10 min)  the two or three hardest problems
 7. failures and ops  (5 min)   what breaks, how it is detected, how it degrades (Chapters 24, 26)
```

### Why it matters

These interviews are the bridge from this book to industry roles. They reward exactly what the book teaches: knowing what each component guarantees, choosing by access patterns and failure behaviour rather than fashion, and doing the arithmetic.

### Concrete: walkthrough 1, a URL shortener

**1. Requirements (0:00–0:05).** Create a short link for a long URL (optionally a custom alias and an expiry); redirect short links to the long URL; count clicks per link. Non-functional: redirects are the hot path and must be fast (p99 under about 50 ms, served close to users) and highly available; link creation can be slower. Out of scope: user accounts beyond an API key, detailed analytics dashboards.

**2. Estimates (0:05–0:10).**

```text
 new links: 100 million per month ≈ 100e6 / 2.6e6 s ≈ 40 per second (peak ~4× → 150/s)
 redirects: read:write 100:1 → ~4,000/s average, ~15,000/s peak
 storage: 5 years × 1.2 billion links/year = 6 billion links × ~500 bytes ≈ 3 TB
 key space: base62 with 7 characters = 62^7 ≈ 3.5 × 10^12 codes: ample for 6 × 10^9
```

**3. API (0:10–0:15).**

```http
POST /v1/links            {"url": "...", "alias": "spring-sale", "expires_at": "..."}
                          Idempotency-Key: ...           → 201 Created, Location: /v1/links/aZ3k9Qp
GET  /{code}              → 302 Found (or 307), Location: https://long.example/...
GET  /v1/links/{code}     → 200 {url, created_at, clicks}
```

Choosing `302`/`307` over `301` (§7.11): browsers and caches store a `301` permanently, so later edits, expiry and click counting stop working for those users. Creation uses an idempotency key (§13.5) so a retried request does not create two codes.

**4. Data model (0:15–0:20).** One access pattern dominates: look up a long URL by its code. `links(code PRIMARY KEY, long_url, owner_key, created_at, expires_at)`. At 3 TB and 15,000 reads per second (most absorbed by caches, below), a relational database with read replicas works; a key–value store partitioned by code also fits the single access pattern (§23.10–23.11). The decision rests on operations and integrity needs, not slogans. Custom aliases need a uniqueness guarantee: the primary-key constraint (§14.3).

**Code generation**, the main design choice:

| Option | How | Trade-off |
|---|---|---|
| random 7-character code | CSPRNG (§8.2), insert, retry on primary-key conflict | unpredictable; collisions rare (6 × 10^9 of 3.5 × 10^12 used: about 0.2% chance per insert late in life) and handled by the constraint |
| counter + base62 | a database sequence, encoded | no collisions; codes are sequential, so links are enumerable (a privacy problem, like the IDOR of §20.9) |
| counter blocks | each instance reserves a block of 1,000 numbers, then applies a reversible scrambling permutation | no per-insert coordination, not enumerable; more moving parts |

Random codes with a uniqueness constraint are the simplest correct choice.

**5. High-level design (0:20–0:30).**

```text
 CREATE: client → API (rate limited per key, §24.6) → validate URL (scheme allowlist; do not fetch it,
         or treat any fetching for previews as SSRF-sensitive, §13.14) → INSERT → 201
 REDIRECT: browser → CDN edge (short-TTL cache of redirects, §10.1, §21.2)
                    → miss: LB → redirect service → Redis (cache-aside, §21.3) → miss: DB replica
                    → 302 Location
                    → click event (code, timestamp, coarse geo) to a log/broker, asynchronously (§25.3)
 ANALYTICS: consumers aggregate clicks per code per hour into a counts table; the link page reads it
```

**6. Deep dives (0:30–0:40).**
- *Hot links.* Popularity is Zipf-like (§21.1): a small cache serves most redirects. A viral link is a hot key (§21.6): the CDN and an in-process L1 cache absorb it (§21.9). TTLs are short (minutes) so expiry and deletion take effect, with explicit purge on delete.
- *Click counting at 15,000/s.* Never `UPDATE clicks = clicks + 1` per redirect on one row (a hot row, lock contention, §17.5). Publish click events and aggregate in batches, or use sharded counters (§23.4's salting); counts become eventually consistent, which is acceptable.
- *Expiry.* Check `expires_at` on the read path (cheap); delete expired rows in background batches (§18.14).
- *Abuse.* Phishing and malware links: scan URLs at creation and on reports; rate limits per key and per IP (§24.6); a kill switch to disable a code instantly at the edge (§24.9).

**7. Failures and operations (0:40–0:45).** Redirects are read-only and cacheable, so they can survive a database outage for cached codes; creation fails closed. Multi-zone deployment with N+1 capacity (§24.11). SLIs: redirect success and latency at the edge; creation success (§26.11). Alerts on cache hit ratio, because the database is sized for it (§21.1).

### Concrete: walkthrough 2, an order and payment system

**1. Requirements.** Customers place orders for items in stock and pay by card; they see order status; warehouses receive confirmed orders; customers get emails. Non-functional: never oversell, never double-charge, no lost orders; checkout p99 under 1 s; peak 10× average during sales. Out of scope: catalogue management, search, shipping logistics.

**2. Estimates.**

```text
 1 million orders/day ≈ 11.6 orders/s average; peak 10× ≈ 120 orders/s
 each order: ~3 lines; order + lines + outbox ≈ 2 KB → 2 GB/day, ~730 GB/year: one primary
 with replicas and partitioned tables (§16.14) suffices; sharding not needed yet (§23.1)
 reads (status pages, history) ~20× writes → replicas and caches for reads
```

**3. API.**

```http
POST /v1/orders           Idempotency-Key: ...   {"lines": [...], "payment_method_id": "..."}
                          → 202 Accepted {order_id, status: "PENDING", status_url}
GET  /v1/orders/{id}      → 200 {status: PENDING | CONFIRMED | CANCELLED, ...}   (owner-scoped, §20.9)
POST /v1/payment-webhooks → 200 (HMAC-verified, deduplicated by event ID, §8.5, §13.10)
```

`202` with a status resource (§13.10) because payment completes asynchronously; the idempotency key makes retries safe (§13.5).

**4. Data model.** `orders`, `order_lines`, `stock (sku, qty, CHECK qty >= 0)`, `payments (order_id, provider_ref, state, idempotency_key UNIQUE)`, `outbox`, `idempotency_keys`, `saga_state` (§14.2–14.3, §25.8–25.9). The stock decision is a conditional update in the order transaction (§17.11). Money is stored as integer minor units, never floats (§14.12).

**5. High-level design.**

```text
 client → edge/LB → orders service: one transaction (stock UPDATE, order rows, outbox, idempotency)
        ← 202
 outbox relay → broker (keyed by order ID, acks=all) → saga orchestrator in the orders service:
   authorise payment (payments service → provider, with idempotency key, timeout, breaker)
   → confirm stock reservation → capture (pivot) → confirm order → events → email, warehouse
 provider webhooks → payments service (verify HMAC, dedupe, update payment state, outbox event)
 status reads → primary for the order's owner right after writes (read-your-writes), replicas otherwise
```

This is the workflow of Chapter 25 (Section 25.12) and the lifecycle of Section 27.2.

**6. Deep dives.**
- *No oversell:* conditional update and a `CHECK` constraint (§17.13); release stock in compensation; flash sales may queue orders per SKU to reduce contention on one hot row (§17.5).
- *No double charge:* idempotency keys at the API and at the provider; unknown outcomes resolved by status query, never by assuming failure (Case 4).
- *No lost events:* the transactional outbox (§25.8); consumers idempotent with inbox tables (§25.6); DLQs with alerts (§25.7).
- *The saga's failure matrix:* compensations for each step before the pivot, retries after it, step timeouts shorter than authorisation validity (§25.9).

**7. Failures and operations.** Provider slow or down: breaker opens, orders remain `PENDING` with a clear message, sagas retry within their timeouts (§24.4). Broker down: orders still accepted (the outbox holds events). Primary failover: a synchronous standby in another zone so no acknowledged order is lost (§22.3). SLOs on checkout success and latency, and on "orders confirmed within 2 minutes" (§26.11). A daily reconciliation between provider charges and order states (Case 4).

### Guarantees and non-guarantees

A structured walkthrough guarantees that the requirements, numbers, interfaces, data, design, risks and operations are all addressed in the time available. It does not guarantee a single right answer: interviewers look for justified trade-offs, and the same requirements admit several good designs.

### Failure and diagnosis

Common failures in these interviews: drawing boxes before stating requirements and numbers; naming products instead of mechanisms ("use Kafka" without saying what ordering or delivery guarantee is needed); ignoring failure modes; and over-engineering (sharding a dataset that fits on one machine, §23.1). Each is fixed by the seven-step structure.

### Backend connection

Practise these two designs aloud against a timer, then repeat the structure for others (a notification service, a rate limiter, a chat system, a news feed, a file-upload service), each time naming the mechanism and chapter behind every choice.

---

## 27.6 The Concept Map: How Every Chapter Connects

### Definitions

A **concept map** shows the dependencies between ideas: which concepts are built from which, and which later topics rely on them. For this book, it shows why the chapters come in their order and where to go back when a later topic feels shaky.

### Why it matters

Backend knowledge is a stack: transactions make sense only after storage, storage only after files and memory, idempotency only after unknown outcomes, unknown outcomes only after TCP. Gaps at the bottom show up as confusion at the top. A map tells you which earlier chapter to revisit when a later one does not click.

### Concrete: the map

```text
                               ┌──────────────────────────────────────────────┐
                               │ 1  The web application: the whole picture   │
                               └───────────────────┬──────────────────────────┘
          PART I: THE MACHINE                       │
   2 How a program runs ──► 3 Processes, threads, ──► 4 Files, I/O, IPC, ──► 5 Concurrency
     (CPU, memory, GC)        scheduling, memory      isolation (cgroups)     (races, locks, pools,
          │                         │                      │                    event loops)
          │                         │                      │                         │
          PART II: NETWORKS         ▼                      ▼                         │
   6 Networking (IP, DNS, TCP) ──► 7 HTTP ──► 8 Crypto & TLS ──► 9 Browser & its security
          │                         │             │                  │               │
          PART III: SERVER          ▼             ▼                  ▼               ▼
   10 Inside the server (edge, LB, proxy, accept → dispatch, timeouts) ──► 11 Concurrency in a
          │                                                                  server (Little's Law,
          ▼                                                                  queues, overload)
   12 Application architecture ──► 13 Designing web APIs (REST, idempotency, evolution, SSRF)
          │                                │
          PART IV: DATA                    ▼
   14 Relational modelling & SQL ──► 15 Storage engines & durability ──► 16 Indexing & query
          │                                  │                             execution
          ▼                                  ▼                                 │
   17 Transactions, isolation, concurrency control ◄──────────────────────────┘
          │
          ▼
   18 Application–database integration (pools, injection, ORM, N+1, migrations, replicas)
          │
          PART V: STATE & IDENTITY
   19 Cookies, sessions, CSRF ──► 20 Authentication, authorisation, identity, security synthesis
          │                                │
          PART VI: PERFORMANCE & DISTRIBUTED DATA
   21 Caching ──► 22 Replication & consistency ──► 23 Partitioning, coordination, data stores
          │                │                              │
          PART VII: RELIABILITY & OPERATIONS
   24 Failure & resilience ──► 25 Async work & messaging ──► 26 Running in production
          └──────────────────────────────┬───────────────────────────────┘
                                         ▼
                   27 Capstone: lifecycle, diagnosis, design, readiness
```

### Concrete: the key cross-links

The arrows above show the main line. The links below are the ones that cross Parts, and they are where most interview follow-up questions live:

| Idea | First taught | Also used in |
|---|---|---|
| Memory hierarchy and latency numbers | 2.2 | caching (21.1), buffer pool (15.3), capacity estimates (27.5) |
| Garbage collection and tail latency | 2.8 | lock leases paused by GC (23.8), percentiles (26.9), Case 1 |
| Processes vs threads; signals | 3.3, 3.7, 3.6 | PostgreSQL vs MySQL process model (15.1), graceful shutdown (24.10) |
| Page cache and fsync | 4.3, 4.4 | WAL and durability (15.4), commit latency (22.3) |
| epoll and I/O models | 4.5 | event-loop servers (10.7, 11.2), Redis's single thread (21.7) |
| cgroups and containers | 4.10 | requests, limits and throttling (26.3) |
| Races, CAS, locks | 5.2, 5.4, 5.5 | database locks and MVCC (17.5–17.6), Redis atomic commands (21.7), distributed locks (23.8) |
| Thread pools and futures | 5.11, 5.12 | server execution models (11.2), bulkheads (24.5) |
| TCP's non-guarantees | 6.9 | unknown outcomes (7.15), idempotency (13.5), delivery semantics (25.5) |
| DNS and TTLs | 6.6 | CDN routing (10.1), failover propagation (22.5) |
| HTTP caching | 7.12 | CDNs (10.1), cache layers (21.2) |
| Conditional requests | 7.13 | API concurrency control (13.7), optimistic locking (17.9) |
| Cookies (wire) | 7.14 | browser rules (9.5), sessions and CSRF (19.2–19.6) |
| HMAC and signatures | 8.5, 8.6 | webhooks (13.10), signed cookies (19.4), JWTs (20.5) |
| Same-origin policy and CORS | 9.6, 9.7 | CSRF (19.6), token storage (20.6) |
| Timeouts nesting | 10.10 | latency budgets and deadlines (24.2) |
| Little's Law | 11.3 | pool sizing (18.4), bulkheads (24.5), capacity planning (26.2), consumer sizing (25.11) |
| Overload and metastability | 11.8 | cascading failures (24.8), Case 5 |
| Idempotency keys | 13.5 | outbox and consumers (25.6), sagas (25.9), Case 4 |
| Schema evolution | 13.11 | event schemas (25.10), deploy compatibility (26.4) |
| Constraints | 14.3 | concurrency guards (17.11), idempotency (25.6) |
| WAL | 15.4 | replication (22.2), CDC (22.10), outbox via CDC (25.8) |
| Isolation anomalies | 17.3 | lost updates in ORMs (18.13), cache races (21.5), Case 2 |
| Replica lag | 18.16 | lag anomalies (22.4), Case 6 |
| Sessions | 19.3 | Redis as a session store (21.8), stateless instances (26.2) |
| Consistency models and CAP | 22.8, 22.9 | consensus (23.7), choosing data stores (23.11) |
| Consensus and fencing | 23.7, 23.8 | failover (22.5), leader election for jobs (25.2) |
| Retries and breakers | 24.3, 24.4 | message retries and DLQs (25.7), Case 5 |
| Outbox and sagas | 25.8, 25.9 | the order lifecycle (27.2), design walkthrough 2 (27.5) |
| Observability | 26.6–26.10 | the diagnosis method (27.3) |

### Guarantees and non-guarantees

The map guarantees a route back to the foundation of any topic. It does not capture every link (the book has hundreds of cross-references), and it does not replace rereading: when a later chapter feels unclear, the map shows **where** to look, and the earlier section still has to be studied.

### Failure and diagnosis

The typical learning failure is studying the top of the stack without the bottom: memorising "use an outbox" without understanding why the dual write fails (TCP's unknown outcome, §6.9; the database's commit point, §15.4). The symptom is an answer that collapses at the first "why?". The cure is to follow the map down until the "why" is answered by a mechanism.

### Backend connection

Use the map in two directions. When a production problem appears at the top (a duplicate email), walk down to the mechanisms that explain it. When you learn a new technology, place it on the map: which mechanisms it implements, which guarantees it gives, and which of the book's failure modes it is subject to.

---

## 27.7 Readiness Self-Assessment: 100 Questions

### Definitions

The **readiness checklist** is a set of 100 questions drawn from the book's interview drills, grouped by Part and mapped to the chapter that teaches each answer. A question counts as **ready** when you can answer it aloud, without notes, covering the points listed under that question in its chapter's Interview Drill, and handle one follow-up "why?".

### Why it exists

Reading builds recognition; interviews require recall and explanation. The checklist turns the book into a practice schedule, and its chapter mapping turns every unready answer into a specific section to revisit.

### Mechanism: how to use it

```text
 1. answer each question aloud (or in writing) in 1–3 minutes, without notes
 2. compare with the chapter's Interview Drill: did you name the mechanism, the trade-off,
    and at least one failure mode?
 3. mark it: ready / shaky / not ready
 4. for shaky and not-ready answers, reread the chapter's section, then re-test after two days
    and again after a week (spaced repetition)
 5. aim for 90+ ready before interviews; re-run the whole list every few weeks
```

### Concrete: the checklist

**Part I: The Machine and the Operating System (22)**

- [ ] 1. What is the client–server model, and why don't we run everything on the client? *(Ch 1)*
- [ ] 2. What is a trust boundary, and why isn't client-side validation sufficient? *(Ch 1)*
- [ ] 3. What is the backend, and how does it differ from the frontend? *(Ch 1)*
- [ ] 4. What is the memory hierarchy, and why do latency numbers matter for design? *(Ch 2)*
- [ ] 5. HDD versus SSD: how does each work, and how does that affect random versus sequential I/O? *(Ch 2)*
- [ ] 6. How does a tracing garbage collector work, and how can GC affect a web service's latency? *(Ch 2)*
- [ ] 7. Why does a process use more memory than its heap, and why does that matter in containers? *(Ch 2)*
- [ ] 8. What is the difference between a process and a thread? *(Ch 3)*
- [ ] 9. What is a system call, and why is it more expensive than a function call? *(Ch 3)*
- [ ] 10. What happens during a context switch, and why is it expensive? *(Ch 3)*
- [ ] 11. How does virtual memory work? What are page tables, page faults and the TLB? *(Ch 3)*
- [ ] 12. What is the difference between SIGTERM and SIGKILL? How does a server shut down gracefully? *(Ch 3)*
- [ ] 13. What is a file descriptor, and what causes "Too many open files"? *(Ch 4)*
- [ ] 14. What does it mean when `write()` returns? Why does `fsync` matter? *(Ch 4)*
- [ ] 15. Blocking, non-blocking and asynchronous I/O: what is the difference, and how does `epoll` work? *(Ch 4)*
- [ ] 16. Walk through the system calls a TCP server makes. What is the listen backlog? *(Ch 4)*
- [ ] 17. What is the difference between a virtual machine and a container? What are namespaces and cgroups? *(Ch 4)*
- [ ] 18. What is a race condition, and how do you prevent one? *(Ch 5)*
- [ ] 19. What is a memory model? What is a visibility problem? *(Ch 5)*
- [ ] 20. What are the four conditions for deadlock? How do you prevent, avoid and detect deadlock? *(Ch 5)*
- [ ] 21. How do you size a thread pool? *(Ch 5)*
- [ ] 22. Compare OS threads, event loops, coroutines, virtual threads and goroutines. *(Ch 5)*

**Part II: Networks and Web Protocols (16)**

- [ ] 23. What happens at each layer when an application sends a message? How do the TCP/IP and OSI models relate? *(Ch 6)*
- [ ] 24. Walk through a DNS lookup of `api.example.com`. What is the difference between a recursive resolver and an authoritative server? *(Ch 6)*
- [ ] 25. Explain the three-way handshake and the close. What are `TIME_WAIT` and `CLOSE_WAIT`? *(Ch 6)*
- [ ] 26. What does TCP not guarantee? Why can't it give message boundaries or exactly-once payments? *(Ch 6)*
- [ ] 27. Distinguish "connection refused", a connect timeout, a read timeout and a reset by what happened on the wire. *(Ch 6)*
- [ ] 28. Describe an HTTP/1.1 request and response on the wire. How does the receiver find the end of the body? *(Ch 7)*
- [ ] 29. What is the difference between safe and idempotent methods? *(Ch 7)*
- [ ] 30. Distinguish `400`, `401`, `403`, `404`, `422`, `500`, `502`, `503` and `504`. *(Ch 7)*
- [ ] 31. How does HTTP caching work, and what is the difference between `no-cache` and `no-store`? *(Ch 7)*
- [ ] 32. What is the difference between encoding, hashing and encryption? Is Base64 encryption? *(Ch 8)*
- [ ] 33. Symmetric versus asymmetric cryptography: why doesn't TLS encrypt everything with RSA? *(Ch 8)*
- [ ] 34. Walk through the TLS 1.3 handshake. How does a client validate a certificate? *(Ch 8)*
- [ ] 35. What does HTTPS not protect against? *(Ch 8)*
- [ ] 36. What is the same-origin policy, and how does CORS work? *(Ch 9)*
- [ ] 37. What is XSS, and how do you prevent stored, reflected and DOM-based XSS? *(Ch 9)*
- [ ] 38. Where should a web application store an access token? *(Ch 9)*

**Part III: The Server and the Application (16)**

- [ ] 39. Forward versus reverse proxy? L4 versus L7 load balancing? *(Ch 10)*
- [ ] 40. What happens between `accept()` and your handler? What is middleware? *(Ch 10)*
- [ ] 41. Who produces a `502`, a `503` and a `504`? *(Ch 10)*
- [ ] 42. How should timeouts be set across the hops of a request? *(Ch 10)*
- [ ] 43. What is Little's Law, and how do you use it to size a pool? *(Ch 11)*
- [ ] 44. A downstream slows from 50 ms to 2 s. What happens? *(Ch 11)*
- [ ] 45. What happens to latency as load approaches capacity? What is backpressure? *(Ch 11)*
- [ ] 46. Why does a lock in your code not stop two instances processing the same order? *(Ch 11)*
- [ ] 47. What is dependency injection, and why use it? *(Ch 12)*
- [ ] 48. Walk through what happens to a JSON request body, from bytes to a database row and back. *(Ch 12)*
- [ ] 49. Monolith versus microservices: what are the trade-offs? What are a modular monolith and a distributed monolith? *(Ch 12)*
- [ ] 50. What makes an API RESTful? *(Ch 13)*
- [ ] 51. How do idempotency keys work, including two retries at once? *(Ch 13)*
- [ ] 52. How do you design pagination? Why is offset slow? *(Ch 13)*
- [ ] 53. How do you version an API, and how do you deprecate a version? *(Ch 13)*
- [ ] 54. REST versus gRPC versus GraphQL: when would you choose each? *(Ch 13)*

**Part IV: Data Persistence (20)**

- [ ] 55. What is normalisation? Explain 1NF, 2NF, 3NF and BCNF with an example. *(Ch 14)*
- [ ] 56. What types of joins are there? Left join versus anti-join? *(Ch 14)*
- [ ] 57. What are window functions? `ROW_NUMBER` versus `RANK` versus `DENSE_RANK`? *(Ch 14)*
- [ ] 58. How does `NULL` behave in comparisons, aggregates and `NOT IN`? *(Ch 14)*
- [ ] 59. What is WAL, and why is it faster than writing pages at commit? *(Ch 15)*
- [ ] 60. Walk through crash recovery. *(Ch 15)*
- [ ] 61. B-tree versus LSM-tree storage: how do they differ, and when would you choose each? *(Ch 15)*
- [ ] 62. Logical versus physical backups; how does point-in-time recovery work; what are RPO and RTO? *(Ch 15)*
- [ ] 63. How does a B+tree index work? Why not a binary tree or a hash table? *(Ch 16)*
- [ ] 64. How should you order the columns of a composite index? What is a covering index? *(Ch 16)*
- [ ] 65. How does the planner choose a plan, and why did a query suddenly become slow without a code change? *(Ch 16)*
- [ ] 66. How do you read `EXPLAIN ANALYZE`? *(Ch 16)*
- [ ] 67. Explain ACID precisely. What implements each property? *(Ch 17)*
- [ ] 68. What are the isolation levels, and what do PostgreSQL and MySQL actually do at each? *(Ch 17)*
- [ ] 69. How does MVCC work? How do PostgreSQL and InnoDB differ? *(Ch 17)*
- [ ] 70. How do you prevent two customers from buying the last item? *(Ch 17)*
- [ ] 71. How does a connection pool work, and how do you size it? *(Ch 18)*
- [ ] 72. What is SQL injection, and why do prepared statements prevent it? *(Ch 18)*
- [ ] 73. What is the N+1 problem? How do you detect and fix it? *(Ch 18)*
- [ ] 74. Where should transaction boundaries go, and why should you never call a remote service inside a transaction? *(Ch 18)*

**Part V: State, Identity and Security (8)**

- [ ] 75. Why is HTTP stateless, and how do sessions add state on top of it? *(Ch 19)*
- [ ] 76. What does each cookie attribute protect against? *(Ch 19)*
- [ ] 77. What is CSRF, and how do you prevent it? When is `SameSite` enough? *(Ch 19)*
- [ ] 78. How should passwords be stored? Why is SHA-256 wrong? *(Ch 20)*
- [ ] 79. What is a JWT? How is it validated, and what are its pitfalls? *(Ch 20)*
- [ ] 80. Sessions versus JWTs: what are the trade-offs? *(Ch 20)*
- [ ] 81. Explain the OAuth 2.0 authorization code flow with PKCE. How does OpenID Connect differ from OAuth? *(Ch 20)*
- [ ] 82. What is IDOR (BOLA), and how do you prevent it? *(Ch 20)*

**Part VI: Performance and Distributed Data (10)**

- [ ] 83. Cache-aside versus read-through versus write-through versus write-behind? *(Ch 21)*
- [ ] 84. How do you keep a cache consistent with the database? What race makes it stale? *(Ch 21)*
- [ ] 85. What is a cache stampede, and how do you prevent it? *(Ch 21)*
- [ ] 86. Synchronous versus asynchronous replication: what are the trade-offs? *(Ch 22)*
- [ ] 87. What is replication lag, and which anomalies does it cause? *(Ch 22)*
- [ ] 88. What is linearizability? How does it differ from eventual consistency? *(Ch 22)*
- [ ] 89. Explain the CAP theorem properly. What is PACELC? *(Ch 22)*
- [ ] 90. Range versus hash partitioning? What is consistent hashing? *(Ch 23)*
- [ ] 91. How do you build a correct distributed lock? What is a fencing token? *(Ch 23)*
- [ ] 92. How does two-phase commit work, and why does it block? *(Ch 23)*

**Part VII: Reliability and Operations (8)**

- [ ] 93. When is a retry safe? What are exponential backoff and jitter? *(Ch 24)*
- [ ] 94. How does a circuit breaker work? *(Ch 24)*
- [ ] 95. What is a cascading failure, and how do you stop one? *(Ch 24)*
- [ ] 96. What do at-most-once, at-least-once and exactly-once delivery mean? *(Ch 25)*
- [ ] 97. What is the dual-write problem? What is the transactional outbox pattern? *(Ch 25)*
- [ ] 98. What is a saga? Choreography versus orchestration? *(Ch 25)*
- [ ] 99. Why use p99 and not averages? *(Ch 26)*
- [ ] 100. What are SLIs, SLOs, SLAs and error budgets? *(Ch 26)*

The full bank of drill questions, including the placement-round sections, is in each chapter's Interview Drill; this checklist is the core.

### Guarantees and non-guarantees

Answering all 100 well guarantees command of the book's core. It does not guarantee interview success, which also depends on data structures and algorithms, framework experience, communication and practice under time pressure (Section 27.8).

### Failure and diagnosis

The checklist fails when used passively: reading the questions and feeling familiar with them is not readiness. The test is speaking the answer, including the mechanism and one failure mode, to someone (or a recording) who can ask "why?".

### Backend connection

Use the checklist as a recurring exam, not a one-time read. The questions you answer worst point to the chapters to reread; the ones you answer best are your strengths to steer interviews toward.

---

## 27.8 Where to Go Next

### Definitions

This book gives the **foundations and internals**: what happens and why. The next stage is **fluency**: building, operating and discussing real systems quickly, with real tools. It has five parts: implementation practice, projects, data structures and algorithms, framework depth, and system-design practice.

### Why it matters

Understanding a mechanism and implementing it under deadline are different skills. Interviews and jobs test both: whether you know why an outbox is needed, and whether you can build one in a framework, with tests, in an afternoon.

### Mechanism: a path from here

**1. Implementation practice: build the mechanisms yourself, small.** Each exercise exercises one or more chapters:

| Exercise | Chapters |
|---|---|
| a minimal HTTP/1.1 server on raw sockets: parsing, keep-alive, timeouts, a thread pool, then an event loop | 4, 5, 7, 10, 11 |
| an LRU cache with TTLs, then a thread-safe version, then a small Redis-protocol server | 5, 21 |
| a token-bucket rate limiter, local and in Redis with a Lua script | 21, 24 |
| a job queue on PostgreSQL with `SKIP LOCKED`, retries with backoff, and a dead state | 17, 25 |
| a transactional outbox with a polling relay and an idempotent consumer | 17, 25 |
| JWT validation from scratch (then never use your own in production) | 8, 20 |
| a key–value store with a write-ahead log and crash recovery | 4, 15 |

**2. One end-to-end project, built properly.** Build the order-and-payment system of Section 27.5: an API with idempotency keys, PostgreSQL with migrations and constraints, Redis sessions and caching, an outbox and a broker, a saga with a fake payment provider that times out and fails on purpose, structured logs, metrics and traces, a container image, a deployment with readiness probes and graceful shutdown, load tests that find its knee, and SLOs with alerts. Then break it deliberately (Section 24.12) and diagnose it with Section 27.3's method. One project done this way teaches more than ten tutorials.

**3. Data structures and algorithms.** Interviews for backend roles still include coding rounds. Keep a daily practice going alongside this material: arrays and hashing, two pointers and sliding windows, stacks and queues, trees and graphs (BFS, DFS, topological sort), heaps, binary search, dynamic programming, and the classic designs that bridge to this book (LRU cache, rate limiter, consistent hashing, a bounded blocking queue).

**4. Framework depth.** With the mechanisms understood, a framework becomes a set of answers to known questions: how it wires components (Chapter 12, Sections 12.4–12.5), how it draws transaction boundaries and where they fail (Chapter 18, Section 18.12), how its ORM tracks and loads data (Chapter 18, Sections 18.8–18.11), how it implements sessions, CSRF protection and OAuth 2.0 (Chapters 19–20), how it consumes messages (Chapter 25), how it exposes health, metrics and traces (Chapter 26), and how it shuts down gracefully (Chapter 4, Section 4.11; Chapter 24, Section 24.10). Study a framework by asking each of those questions and finding where its documentation answers them. On the front end, single-page applications meet the backend at the points Chapter 9 explained: fetch, CORS, cookies, token storage and rendering strategies.

> **Framework lens:** For Java and Spring Boot, the questions map to Spring's dependency injection and its proxies, `@Transactional` and its self-invocation and propagation pitfalls, Spring Data JPA and Hibernate's persistence context and fetch strategies, Spring Security's session management, CSRF and OAuth 2.0 client and resource-server support, Spring for Apache Kafka, Spring Boot Actuator with Micrometer for metrics and tracing, and `server.shutdown=graceful`. The Java and Framework lens boxes throughout the book are the reading list. For React front ends, the same Chapter 9 boundaries apply whatever the framework.

**5. System-design practice.** Use the seven-step structure of Section 27.5 on a list of classic problems: URL shortener, rate limiter, notification service, chat system, news feed, file storage and upload, search autocomplete, web crawler, ticket booking (an oversell problem, Case 2), payment system, metrics and logging pipeline, distributed job scheduler. For each, name the mechanism and chapter behind every choice, do the arithmetic, and describe the failure modes.

**Topics beyond this book, worth learning next:**
- **Stream processing**: computing continuously over event streams (windows, event time versus processing time, watermarks, stateful operators), built on the logs of Chapter 25.
- **Hands-on orchestration and cloud platforms**: the concrete tools behind Chapter 26's concepts (a Kubernetes cluster, one cloud provider's networking, identity and managed databases).
- **Deeper distributed systems**: formal consistency models, Paxos variants, Byzantine fault tolerance (Chapter 23's placement section was only the start), and reading classic papers (Dynamo, Spanner, Raft, the Kafka paper).
- **Performance engineering**: benchmarking methodology, CPU microarchitecture effects, and kernel-level tracing with eBPF (Chapter 26, Section 26.12).
- **Security in depth**: penetration testing basics, secure code review, and threat modelling practice (Chapter 20, Section 20.12).

### Guarantees and non-guarantees

The path above guarantees steady progress from understanding to fluency if followed with real practice. It does not guarantee speed: fluency comes from repetition, and from building and breaking real systems more than from reading about them.

### Failure and diagnosis

The common failure after a book like this is collecting more theory instead of building. The symptom is fluent explanations and slow, uncertain implementations. The cure is the project of step 2, with deliberate failure injection and diagnosis.

### Backend connection

Return to this book as a reference while you build. When a framework does something surprising, find the mechanism underneath it here; when production breaks, start at Section 27.3.

---

## Chapter Summary

A backend is a stack of mechanisms, and every user action crosses all of them. A **URL typed into a browser** becomes a DNS lookup through caches and authoritative servers, a TCP (or QUIC) connection, a TLS handshake that authenticates the server and derives keys, an HTTP request carrying cookies by the browser's rules, an edge that caches or forwards, a load balancer that routes to ready instances, a proxy that sets the timeout chain, a server that accepts and dispatches to a thread or event loop, middleware that authenticates and limits, application code that validates and reads through caches and connection pools into indexed, buffered storage, and a response that is serialised, compressed, cached and rendered. The answer can be told in five minutes or thirty, and each step's guarantee is local: no layer guarantees the next.

A **write request** adds correctness under concurrency, retries and partial failure: idempotency keys at the API, an atomic conditional update and constraints in one database transaction, an outbox row committed with the business change, a commit made durable by the WAL and a synchronous standby, and everything else (payment, emails, indexes, caches) driven asynchronously by a saga and idempotent consumers, with nested timeouts, bulkheads and breakers bounding the damage of slow dependencies. Truth lives in the database's committed state; everything else is derived and may lag.

**Diagnosis** follows one method: state the symptom precisely, scope it (and what is not affected), form several mechanism-based hypotheses, gather evidence that separates them with each layer's tool, isolate the layer, mitigate then fix, verify against the mechanism, and prevent recurrence. The six case studies show why real failures need several chapters at once: tail latency from an N+1 that also stressed a pool; an oversell that isolation levels and in-process locks did not stop; a login broken by site boundaries and proxy headers; a payment flow that treated an unknown outcome as failure; a cascade from one unprotected dependency; and stale reads after a failover that lost commits, left caches wrong and lagged replicas. **System-design interviews** apply the same building blocks under time pressure, from requirements and arithmetic to failure modes and operations. The **concept map** shows how each chapter rests on earlier ones, the **100-question checklist** measures readiness, and the path forward is building, breaking and diagnosing real systems.

---

## Guarantees vs. Non-Guarantees

| Layer or mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| DNS (§6.6) | A name resolves to an address, cached for its TTL | Freshness during changes; that the address is reachable |
| TCP (§6.8–6.9) | A reliable, ordered byte stream between two sockets | Message boundaries; that the application processed the data; exactly-once effects |
| TLS (§8.9, §8.12) | Confidentiality, integrity and server authentication between its endpoints | Safety at the endpoints; hiding IPs and (without ECH) host names |
| HTTP semantics (§7.4, §7.15) | Defined meanings for methods and status codes | Knowing the outcome after a timeout |
| Browser policies (§9.5–9.7) | Same-origin isolation, cookie rules, CORS for reads | Protection against CSRF without defences; protection against XSS in your origin |
| Edge and load balancer (§10.1, §10.3) | Routing to ready instances; caching by rules | Correct caching of personalised responses unless marked |
| Server concurrency limits (§11.8, §24.5) | Bounded in-flight work for accepted requests | Which requests are rejected without priorities |
| Database transaction (§17.2) | Atomic, durable changes with the isolation level's guarantees | Prevention of anomalies the level allows (lost updates at Read Committed with read-modify-write) |
| Constraints and conditional updates (§14.3, §17.11) | Invariants enforced by the database under any concurrency | Business rules nobody encoded |
| Replication (§22.3) | Copies on other machines; no acknowledged loss with synchronous standbys | Current reads on async replicas; protection from replicated mistakes |
| Cache (§21.1, §21.5) | Faster reads on hits | Freshness; atomicity with the database |
| Outbox + broker (§25.8, §25.4) | Every committed change produces its event, at least once, ordered per key | Exactly-once delivery; immediate publication |
| Idempotent consumers and keys (§13.5, §25.6) | Repeats do not repeat effects | Effects outside the deduplicated store unless keys are passed on |
| Saga (§25.9) | An eventual consistent outcome or compensation | Isolation; instant completion |
| Resilience mechanisms (§24) | Bounded damage from slow or failing dependencies | That fallbacks are correct; protection against untested failures |
| Observability (§26.6–26.11) | Evidence to answer questions about the running system | Answers to questions the telemetry never recorded |
| Diagnosis method (§27.3) | Conclusions backed by refutable evidence | Speed without observability; hypotheses without knowledge |

---

## Interview Drill

**Q1. What happens when you type a URL into the browser and press Enter?**
A strong answer must mention:
- URL parsing, HSTS, the browser cache; DNS resolution through caches to authoritative servers.
- TCP (or QUIC) and TLS 1.3 with SNI, ALPN and certificate validation; the HTTP request with cookies.
- CDN, load balancer, reverse proxy; server accept, parse, thread or event loop, middleware, handler, cache, pooled database query via an index and the buffer pool.
- Response serialisation, compression, caching headers; browser rendering (DOM, CSSOM, layout, paint) and subresources.
- The ability to go deeper into any step, with its failure modes.
*Common weak answer:* "DNS, then HTTP, then the server returns HTML." It names layers without mechanisms or failures.

**Q2. Walk through what happens when a user places an order, from the click to the confirmation email.**
A strong answer must mention:
- Request with session cookie, CSRF protection and an idempotency key; edge, LB, proxy timeouts.
- Middleware (rate limit, session, authorisation), validation, idempotency claim.
- One transaction: conditional stock update, order rows, outbox row; WAL and synchronous standby commit.
- `201`/`202`; outbox relay to the broker; saga for payment with timeouts, retries and idempotency; idempotent email consumer.
*Common weak answer:* "The server saves the order and sends an email." It skips concurrency, retries and partial failure, which are the whole difficulty.

**Q3. Where can a request fail on its way through a system, and what does each failure look like to the caller?**
A strong answer must mention:
- DNS (`NXDOMAIN`, timeouts), TCP (refused versus connect timeout versus reset), TLS (certificate errors).
- Edge and proxy (`502`, `503`, `504`, `499`), server (worker exhaustion, `413`/`431`), application (`4xx`/`5xx`).
- Database (pool timeouts, lock waits, deadlocks, serialisation failures), downstream dependencies (timeouts, breakers open).
- A timeout anywhere means an unknown outcome.
*Common weak answer:* "It returns a 500." Most failures produce other signals, and some produce none.

**Q4. How do you approach debugging a production problem you have never seen before?**
A strong answer must mention:
- Symptom stated precisely; scope including what is not affected; what changed.
- Several mechanism-based hypotheses with evidence that separates them.
- Isolate the layer with RED, USE, traces; confirm with profiles, dumps, logs.
- Mitigate first; fix; verify against the mechanism; prevent with tests, alerts or design.
*Common weak answer:* "Look at the logs and restart the service." Restarting destroys evidence and rarely fixes the cause.

**Q5. A service's p99 latency doubled. How do you tell GC pauses, N+1 queries and connection-pool exhaustion apart?**
A strong answer must mention:
- GC: all endpoints of an instance slow together; GC logs align with slow requests.
- N+1: slowness scales with data size; traces show many small identical queries; queries per request high.
- Pool exhaustion: pending borrowers and acquisition time rise; threads parked in borrow; all database-using endpoints affected.
- One can cause another (N+1 holds connections longer); verify the fix against each mechanism.
*Common weak answer:* "Increase the pool size." That treats a symptom and can overload the database.

**Q6. How do you prevent two customers from buying the last item, and where exactly is the check made?**
A strong answer must mention:
- In the database, atomically: conditional update with row-count check, `FOR UPDATE`, or a constraint.
- Read Committed with read-modify-write in the application allows lost updates; application locks do not span instances.
- Never decide from cached stock.
- Concurrency tests prove it.
*Common weak answer:* "Put a lock around the purchase method." An in-process lock protects one process, not twelve instances.

**Q7. Login works on your laptop but fails in production. What are the likely causes?**
A strong answer must mention:
- Cookies not sent: cross-site API domains and `SameSite`, `Secure` over HTTP, `Domain`/`Path` scope, third-party cookie blocking.
- CORS with credentials misconfigured.
- Proxy headers not trusted: the app thinks it is on HTTP; wrong redirects and cookie flags.
- Sessions not shared between instances; DevTools and server logs as evidence.
*Common weak answer:* "Set SameSite=None." It trades one failure for third-party blocking and CSRF exposure.

**Q8. A payment succeeded but the order shows as failed. How can that happen, and how do you design against it?**
A strong answer must mention:
- A timeout treated as failure: the outcome was unknown, and the charge completed later.
- Compensation without checking status; late success webhooks rejected.
- Design: unknown state, status query by idempotency key, late-reply compensation, reconciliation jobs.
- Timeouts derived from measured latency; failure-injection tests.
*Common weak answer:* "Increase the timeout." Any timeout can expire before an operation completes.

**Q9. How can one slow dependency take down a whole system, and how do you prevent it?**
A strong answer must mention:
- Little's Law: slowness multiplies in-flight requests until threads and pools are exhausted, affecting unrelated endpoints.
- Amplifiers: deep health checks removing instances, synchronised retries, cold caches: a metastable failure.
- Prevention: timeouts from measured latency, bulkheads, breakers, honest fallbacks, local health checks, retry budgets with jitter.
- Recovery: cut load from outside and restore gradually.
*Common weak answer:* "Add more servers." More callers make the slow dependency slower.

**Q10. Users see old data after a database failover. Why, and what do you do about it?**
A strong answer must mention:
- Asynchronous replication lost acknowledged commits (the loss window).
- Caches holding values from the lost history; replicas lagging after the topology change.
- Recovery from the old primary's WAL; cache invalidation; read-your-writes routing.
- Synchronous or quorum standby for critical data; failover drills.
*Common weak answer:* "Replication lag; it will catch up." Lost commits never catch up.

**Q11. Design a URL shortener.**
A strong answer must mention:
- Requirements and arithmetic (writes ~40/s, reads ~4,000/s, ~3 TB over five years, 62^7 codes).
- API with `302`/`307` redirects and idempotent creation; random codes with a uniqueness constraint (or scrambled counters).
- Read path through CDN, cache-aside and replicas; Zipf popularity; hot keys.
- Asynchronous click counting; abuse controls; failure behaviour.
*Common weak answer:* "Hash the URL with MD5 and take 7 characters." It ignores collisions, custom aliases and why the design follows the read path.

**Q12. Design an order and payment system.**
A strong answer must mention:
- `202` with a status resource and idempotency keys; owner-scoped reads.
- One transaction with conditional stock update and outbox; integer money.
- Saga with failure matrix, pivot, idempotent provider calls, webhook verification.
- Synchronous standby, SLOs, reconciliation, breaker on the provider.
*Common weak answer:* "Use a distributed transaction across services." Two-phase commit blocks and most participants cannot take part.

**Q13. How do the layers of a backend depend on each other?**
A strong answer must mention:
- Machine and OS (memory, processes, I/O) under networking and protocols, under servers and applications, under data, state, distribution and operations.
- Concrete links: TCP's non-guarantees → idempotency → outbox and consumers; page cache and fsync → WAL → replication and CDC; Little's Law → pools, bulkheads, capacity planning.
- Using the map to trace a problem down to its mechanism.
*Common weak answer:* listing technologies without any dependency between them.

**Q14. Which guarantees does each layer provide, and which must the application provide for itself?**
A strong answer must mention:
- TCP: ordered bytes, not processing; TLS: transit protection, not endpoint safety; HTTP: semantics, not outcomes after timeouts.
- Database: transactions and constraints, within the isolation level's limits; replicas and caches may be stale.
- Brokers: at-least-once, not exactly-once effects.
- The application provides idempotency, invariants via the database, reconciliation, and correct handling of unknown outcomes.
*Common weak answer:* "The infrastructure handles reliability." Every layer leaves gaps the application must close.

---

## Scenario Exercises

1. Users in one country report that the site never loads, while everyone else is fine. The status page is green. Give three hypotheses across different layers and the evidence that separates them.
2. After moving the API behind a new edge provider, a few `POST /payments` requests per day are executed twice, although clients swear they sent them once. Explain the possible mechanisms and the fix.
3. Every morning, checkout p99 rises steadily from 08:00 and recovers only after a nightly maintenance job at 02:00. No deploys happen in between. Propose a diagnosis plan and your leading hypotheses.
4. After a monolith was split into services, checkout fails for about a minute whenever the recommendations service deploys, although checkout does not need recommendations. Explain how this can happen and how to fix it.
5. A monthly billing job, moved from a server's cron to the orchestrator's scheduled jobs, ran twice this month and once not at all the previous month. Explain both events and redesign the job.

---

## Diagnosis Lab

**Lab 1: Intermittent 502s at the load balancer**
- *Symptom:* About 0.05% of requests to the API return `502` from the load balancer, spread evenly through the day, more during quiet periods than busy ones. Application logs show no errors for those requests.
- *Hypotheses:* (a) instances crashing or restarting (§24.10); (b) the keep-alive race between the balancer and the application, where idle timeouts are ordered the wrong way (§7.9, §10.10); (c) a network problem resetting connections (§6.14).
- *Evidence:* No restarts or deploys correlate with the errors, ruling out (a). The balancer's logs show "upstream connection reset" on requests sent over **reused** connections that had been idle for about 60 seconds. The application server's keep-alive idle timeout is 60 s; the balancer's backend idle timeout is 350 s. So the application closes idle connections just as the balancer reuses them: a request sent at the same moment as the server's FIN receives a reset. A packet capture (§6.14) on one instance shows the FIN from the server and the request from the balancer crossing within milliseconds. The pattern is worse in quiet periods because more connections sit idle long enough to hit the timeout.
- *Root cause:* (b): idle timeouts ordered against Chapter 10's rule, which requires that the client side (the balancer) close idle connections before the server side does (§10.10).
- *Fix:* Set the application's keep-alive timeout above the balancer's backend idle timeout (or lower the balancer's below the application's), for example balancer 60 s, application 65 s.
- *Prevention:* The timeout chain documented per hop, idle timeouts growing inward, and a check in configuration review.

**Lab 2: The database disk that filled itself**
- *Symptom:* At 04:30 the primary database stops accepting writes: "No space left on device". The data size has grown only slightly in months.
- *Hypotheses:* (a) table growth or bloat (§17.6); (b) WAL retained for a consumer that is not reading (§15.10, §22.2); (c) logs or temporary files filling the disk (§4.12).
- *Evidence:* Table and index sizes are stable, ruling out (a); temporary and log directories are small, ruling out (c). The WAL directory holds 480 GB. `pg_replication_slots` shows a logical slot named for the CDC connector that feeds the search index (§22.10), `active = false`, with retained WAL growing since the connector's last deploy nine days ago, when its configuration was broken. Search results had been going stale for nine days, but nobody alerted on connector lag or outbox age (§25.13).
- *Root cause:* (b): an inactive replication slot retaining WAL indefinitely, with no alert on slot retention.
- *Fix (now):* Free space carefully: if the connector cannot be restored quickly, drop the slot (accepting that the search index must be rebuilt from a snapshot), then restore writes; never delete WAL files by hand. *Fix (lasting):* alerts on retained WAL per slot and on connector lag; a cap on slot retention where the database supports it, accepting that a too-slow consumer must re-snapshot; a runbook for rebuilding derived stores.
- *Prevention:* Every derived store (search, cache, analytics) monitored like a replica, by its position and its lag (§22.11, §26.11).

---

## Solution Sketches

1. (a) DNS: that country's resolvers return a wrong or stale address (an expired record cached, or a geo-DNS mapping to a failed point of presence); evidence: `dig` from that country against public and local resolvers, compared with elsewhere (§6.6, §6.14). (b) Edge: the CDN point of presence serving that region is failing (TLS errors, a bad certificate deployment there, or overload); evidence: requests from that region via a VPN or probes, the CDN's per-region error metrics, `openssl s_client` to the regional edge (§10.1, §8.13). (c) Network: a routing problem or national filtering between that country and the edge; evidence: `traceroute` from the region, TCP connect failures rather than HTTP errors (§6.3). The status page is green because it measures from elsewhere: synthetic checks should run from several regions (§26.11).
2. Possible mechanisms: the edge or load balancer retries a request to another origin instance when the first connection resets or times out after the request was sent, which is safe only for idempotent requests (§24.3, §7.15); or clients' HTTP libraries retry silently on connection errors. Evidence: two arrivals at different instances with the same client connection details within seconds, proxy logs showing an upstream retry. Fix: idempotency keys required on `POST /payments` and enforced by the server (§13.5); configure the edge to never retry non-idempotent methods; pass the key to the payment provider.
3. Hypotheses: a long-running transaction or query started by a process during the day (or left open after the nightly job) holds back VACUUM, so table and index bloat grow through the morning and queries slow until the 02:00 maintenance job (which ends it or vacuums) (§17.12, §17.6); or a cache that is warmed at 02:00 and degrades through the day (§21.6); or a replica serving reads accumulating lag under daytime load (§22.4). Plan: correlate p99 with dead-tuple counts and the oldest transaction age in `pg_stat_activity` (§17.14), cache hit ratio, and replica lag; check `EXPLAIN ANALYZE` of the checkout queries at 08:00 versus 14:00 (§16.12).
4. Checkout probably calls recommendations synchronously somewhere (a shared library, a page fragment, or a "post-checkout suggestions" call) without a short timeout and fallback, or through a shared thread pool; during the recommendations deploy, its instances drain or start cold, calls slow down or fail, and checkout's threads block (§24.1, §24.5). Or recommendations' deploy lacks graceful shutdown and slow start, producing resets and slow cold instances (§24.10, §10.3). Fix: make the call asynchronous or optional with a short timeout, bulkhead and breaker, and an empty fallback (§24.9); fix recommendations' shutdown and readiness; trace a checkout during a recommendations deploy to confirm.
5. Twice: the orchestrator started the job again after a node failure or a timeout while the first run was still going, or the job's "already ran" check was missing (§26.3, §25.2). Not at all: the scheduler was unavailable at the scheduled time (or the job's start deadline passed) and missed runs were not caught up. Redesign: make the job idempotent per billing period, with a period claim in the database (`INSERT INTO billing_runs (period) … ON CONFLICT DO NOTHING`, §11.7), per-customer idempotency keys for charges (§13.5), a policy for missed runs (run late, catch up), a concurrency policy forbidding overlapping runs, and an alert when a period has no completed run by a deadline (§26.11).

---

**END OF CHAPTER 27**
