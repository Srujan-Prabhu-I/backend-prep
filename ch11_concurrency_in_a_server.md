# Chapter 11: Concurrency in a Server: Execution Models, Capacity, and Overload

## Where We Are

Chapter 5 built the tools of concurrency: races and critical sections, the memory model, atomics and locks, semaphores and condition variables, thread pools and their sizing formula, futures, and the execution models (thread per task, event loop, virtual threads and goroutines, actors). Chapter 10 then opened the server process: the accept queue, the acceptor and poller, the parser, the worker hand-off, the middleware chain, and the four server architectures.

This chapter puts the two together and adds numbers. A server is a concurrent program whose tasks are requests. Hundreds of them overlap; they share some state and must not share other state; they wait in queues and compete for pools; and when more arrive than the server can finish, latency does not rise gently, it collapses. The questions here are the ones that decide whether a service survives production: how many requests can it handle at once, how big should each pool be, why does a slow dependency take down everything, why do more threads or more instances sometimes make it worse, how do in-process locks fail when there are ten instances, and what should a server do when it is overloaded.

The central tool is **Little's Law**, a one-line relationship between throughput, latency and concurrency that turns most capacity questions into arithmetic.

**After this chapter you can:**
- separate shared state from request-scoped state in a server, and find the shared mutable state that needs protection;
- compare thread pools, event loops, virtual threads and worker processes on one workload, with numbers, including what each does when a call blocks;
- apply Little's Law to size concurrency and pools, and predict what a slow downstream does to a server;
- list every queue a request passes through, and explain why unbounded queues hide overload;
- explain why resource pools are separate bottlenecks, and why more threads or instances can increase database contention;
- write thread-safe application services, and keep request context correct across thread pools and async code;
- explain why in-process locks fail with several replicas, and what to use instead;
- describe saturation, the latency knee, collapse, retry storms, backpressure, bounded queues, admission control and load shedding;
- distinguish an API's style (blocking, non-blocking, reactive) from the execution model underneath, and decide when reactive programming is worth it;
- read a thread dump to diagnose pool exhaustion, a stalled downstream, lock contention and CPU saturation.

**Interview questions this chapter prepares you for:**
1. How many requests can a server handle concurrently?
2. What is Little's Law? How do you use it to size a thread pool or a connection pool?
3. A downstream service slows from 50 ms to 2 s. What happens to your service, and why?
4. Why can adding threads make a service slower?
5. Thread-per-request versus event loop versus virtual threads: what does each do with a blocking call?
6. Why is shared mutable state in a singleton service a bug? How do you fix it?
7. How do you pass request context (a user, a trace ID) through thread pools and async code?
8. You have ten instances of a service. Why does a lock in your code not prevent two of them from processing the same order?
9. What happens to latency as load approaches capacity?
10. What is backpressure? What are bounded queues, admission control and load shedding?
11. Is reactive programming faster? When is it worth it?
12. How do you tell CPU saturation from lock contention, and both from a slow downstream?

**Scope.** Here: concurrency at the level of requests, execution models compared on one workload, capacity arithmetic, queues and pools, thread safety in services and request context, coordination across instances, overload behaviour, and API styles. Recalled, not re-taught: the primitives and execution models of Chapter 5 (Sections 5.2–5.14) and the server internals and architectures of Chapter 10 (Sections 10.5–10.7). Database locks and transactions are Chapter 17; connection pools inside drivers and ORMs are Chapter 18; distributed locks and leader election are Chapter 23; the detailed mechanisms of retries, circuit breakers, rate limiters and load shedding are Chapter 24; capacity planning across a fleet and autoscaling are Chapter 26.

---

## 11.1 Concurrency at the Request Level

### Definitions

**Request-level concurrency** is the overlap in time of many requests being processed by one server: at any instant, some are being parsed, some are waiting for the database, some are writing responses.

**Request-scoped state** is data that belongs to one request and is visible only to the code handling it: the request object, the authenticated user, the order being built, the database transaction, the trace ID.

**Shared state** is data visible to every request handled by the process: configuration, singleton services, caches, connection pools, counters, rate limiters. It is **shared mutable state** if any request can change it (Chapter 5, Section 5.2).

### Why it matters

Concurrency bugs in servers almost always come from confusing the two kinds of state: request data stored somewhere shared (a field of a singleton, a static variable, a thread-local that is not cleared), or shared data changed without protection (a plain map used as a cache). The first leaks one user's data into another's request; the second corrupts shared structures under load. Neither shows up in a test that sends one request at a time.

### Mechanism: where each kind of state lives

```text
 PROCESS (one per instance)
 ┌───────────────────────────────────────────────────────────────────────────────┐
 │ SHARED, read-only after start: configuration, routing table, compiled templates │  safe
 │ SHARED, mutable: caches, counters, rate limiters, connection pools, metrics      │  needs a strategy
 │                                                                                │  (Chapter 5, §5.10)
 │   request A (thread T3)          request B (thread T9)        request C (loop cb) │
 │   ┌───────────────────────┐      ┌───────────────────────┐    ┌───────────────┐  │
 │   │ req, user = alice     │      │ req, user = bob       │    │ req, user = … │  │  request-scoped:
 │   │ order draft, tx, trace│      │ cart, tx, trace       │    │               │  │  locals, parameters,
 │   └───────────────────────┘      └───────────────────────┘    └───────────────┘  │  request objects
 └───────────────────────────────────────────────────────────────────────────────┘
```

The safe default follows Chapter 5's thread-safety strategies (Section 5.10): **request-scoped data stays in local variables, parameters and request objects** (confinement); **shared data is immutable** after startup where possible; and the remaining shared mutable state is delegated to thread-safe components (a concurrent map, an atomic counter, a pool) or protected by a lock. Section 11.6 shows the classic violation.

### Concrete: classifying the state of a typical service

| State | Shared across requests? | Mutable? | What keeps it safe |
|---|---|---|---|
| configuration loaded at startup | yes | no (after startup) | immutability, published safely before the first request (Chapter 5, Section 5.3) |
| singleton services (`OrderService`, `PricingService`) | yes | **should not be** | stateless design: fields hold only collaborators (Section 11.6) |
| caches (products, exchange rates) | yes | yes | a thread-safe, **bounded** cache with atomic load-or-compute |
| counters and metrics | yes | yes | atomic or striped counters (Chapter 5, Sections 5.4 and 5.9) |
| connection pools, HTTP clients | yes | yes, internally | designed to be shared: create one, use it from all requests |
| in-memory rate-limiter state | yes | yes | atomics or a lock; and correct only per instance (Section 11.7) |
| the request object, headers, parsed body | no | yes | confinement to the request's call chain |
| authenticated user, tenant, locale | no | no | confinement; propagation across threads (Section 11.6) |
| the database transaction | no | yes | bound to one request and one call chain (Chapter 18) |
| request ID, trace context, log context | no | no | per-request context, cleared after the request (Section 11.6) |
| reusable per-thread scratch buffers | per thread | yes | a thread-local that never escapes the thread |

A correct handler keeps request data local and uses shared state only through thread-safe components:

```pseudo
class ProductHandler:
    productCache = BoundedConcurrentCache(maxEntries = 10000, ttl = 60s)   // shared, mutable, thread-safe
    pricing                                                              // a stateless collaborator

    function get(req):
        user    = req.user                                  // request-scoped: a local
        id      = req.pathParam("id")                       // request-scoped: a local
        product = productCache.getOrLoad(id, () -> db.loadProduct(id))   // atomic load-or-compute
        price   = pricing.priceFor(product, user)           // per-user: computed, never cached by product
        return json(product, price)
```

The subtle trap is the cache **key**. A shared cache is safe only if its key includes everything its value depends on. Caching `pricing.priceFor(product, user)` under the product ID alone would serve one customer's negotiated price to every other customer: the same mistake as a CDN caching a personalised page without the right `Vary` (Chapter 7, Section 7.12), inside your own process.

### Guarantees and non-guarantees

A server framework guarantees that each request gets its own request object and that handlers run concurrently. It does not guarantee that anything you share is safe: singleton services, static fields and caches are shared across all concurrent requests by default, and the framework knows nothing about what your code stores in them.

### Failure and diagnosis

The symptoms are distinctive: data from one user appearing in another user's response, totals that are wrong only under load, intermittent exceptions from collections modified during iteration. Evidence: the bug disappears when the server handles one request at a time (a single worker thread, a load test with concurrency 1) and reappears under parallel load (Chapter 5, Section 5.14 described stress testing).

### Backend connection

For every field and static variable in a service, ask: "is this request data?" (move it to a local or parameter) or "is this shared?" (then what makes it safe?). That one question prevents most request-level concurrency bugs.

---

## 11.2 Execution Models Compared on One Workload

### Definition

An **execution model** is how a server maps concurrent requests onto threads and processes (Chapter 5, Section 5.13; Chapter 10, Section 10.7 described the server forms). This section compares them on the same workload, with numbers.

### Why compare on one workload

Statements such as "event loops scale better" or "virtual threads make blocking free" are true only for particular workloads. Fixing one workload and running the arithmetic shows exactly what each model costs, where each one breaks, and what none of them changes.

### Concrete: the workload

```text
 offered load:         1,000 requests per second
 per request:          2 ms of CPU (parsing, business logic, serialisation)
                       98 ms waiting (two database queries and one HTTP call to another service)
                       → 100 ms total latency
 machine:              4 cores
 downstream:           a database behind a pool of 20 connections; the HTTP service is shared
```

Two quantities follow, whatever the model (Section 11.3 explains why):

```text
 CPU needed:           1,000 req/s × 2 ms = 2 core-seconds per second = 2 of 4 cores (50%)
 requests in flight:   1,000 req/s × 0.1 s = 100 at any moment, on average
```

So the server must hold **about 100 requests at once**, each spending 98% of its time waiting.

### Mechanism: what each model does with this workload

**1. Thread pool with platform threads** (thread per request, Tomcat-style, Chapter 10, Section 10.7). Each in-flight request occupies one OS thread for its whole 100 ms, including the 98 ms of waiting: the thread **blocks** in the database driver or HTTP client, parked by the kernel. So the pool needs at least **100 threads**; a pool of 200 has headroom. Each platform thread reserves a stack (commonly up to 1 MB of address space, with much less actually used) and costs a context switch whenever it wakes (Chapter 3, Section 3.11). At 100–200 threads this is fine. At 2,000 it becomes heavy.

**2. Event loop** (Node.js style). One thread runs all handlers; while a request waits, the loop runs others, so waiting costs nothing but a callback and some memory. The CPU is the limit: one loop uses at most one core, and each request needs 2 ms of it, so one loop handles at most **500 requests per second**. This workload needs at least **2 processes** (with headroom, 4, one per core) behind a balancer. The model's weakness is a **blocking call**: if one handler does 50 ms of synchronous work (a CPU-heavy computation, a synchronous file read, a blocking library), the loop serves nobody else for those 50 ms, and that process's capacity falls toward 1 / 0.052 s ≈ 19 requests per second.

**3. Virtual threads or goroutines.** Code is written like the thread-pool version, with blocking calls, but when a virtual thread blocks on I/O, the runtime **unmounts** it from its carrier OS thread and runs another (Chapter 5, Section 5.13). 100, or 10,000, waiting virtual threads cost only their small, heap-allocated stacks. CPU parallelism comes from a few carrier threads, about one per core. Blocking is cheap **only** for operations the runtime can intercept; a virtual thread that blocks while **pinned** (in some runtimes, inside certain synchronized regions or native calls) holds its carrier like a platform thread.

**4. Worker processes, synchronous** (Gunicorn sync workers, Chapter 10, Section 10.7). Each process handles one request at a time and blocks while it waits. To hold 100 requests at once, it needs **100 processes**. If each process uses 100 MB of memory, that is **10 GB** for a workload that uses only 2 cores. Synchronous workers are the wrong model for wait-heavy work; threaded or async workers fix it.

### Concrete: the comparison

| | Platform thread pool | Event loop | Virtual threads / goroutines | Sync worker processes |
|---|---|---|---|---|
| What holds a waiting request | a blocked OS thread | a registered callback / suspended async function | a parked virtual thread (heap object) | a blocked whole process |
| Minimum "slots" for 100 in flight | 100 threads | 1 loop holds them all (CPU needs 2+ processes) | 100 virtual threads (cheap) | 100 processes |
| Memory for the slots | ~100 stacks: modest | tiny | tiny | ~10 GB at 100 MB each |
| A **blocking** call inside a handler | blocks that thread only | **stalls every request on that loop** | parks the virtual thread (unless pinned) | blocks that process only |
| A **CPU-heavy** handler (50 ms) | uses a core; others continue on other threads | **stalls the loop** | uses a carrier; others continue on other carriers | uses a core; others continue |
| Cores used | all | one per process | all | all |
| What it does not change | the database still has 20 connections | the same | the same | the same |

The last row is the one that matters most. Every model can **hold** 100 waiting requests; the 98 ms of waiting is spent in the **database and the other service**, and none of the models makes those faster or bigger (Chapter 5, Section 5.13's accuracy point). With a pool of 20 database connections and, say, 15 ms of each request spent holding one, the pool can serve at most 20 / 0.015 s ≈ **1,333 requests per second**, whatever the execution model: enough for today's 1,000, and the ceiling for tomorrow's growth no matter how cheaply the server holds waiting requests. Section 11.3 explains the arithmetic and Section 11.5 the consequences.

### Guarantees and non-guarantees

Each model guarantees its own cost of waiting: a thread, a callback, a virtual thread, a process. None guarantees throughput beyond the slowest shared resource, and each has its own failure: thread exhaustion, a blocked loop, pinning, memory.

### Failure and diagnosis

- **Platform threads exhausted.** All threads blocked in the same downstream call; new requests queue (Section 11.4). Evidence: a thread dump with every worker in the same frame (Section 11.10).
- **Event loop blocked.** All requests on one process slow together; that process uses 100% of one core. Evidence: event-loop lag metrics, a CPU profile (Chapter 4, Section 4.5).
- **Virtual-thread pinning.** Latency rises as if threads were scarce although few carriers are busy with CPU. Evidence: runtime diagnostics that report pinned threads (the JDK can report them when configured to).

### Backend connection

Pick the model for the work: virtual threads or goroutines (or async runtimes) for wait-heavy services with many concurrent requests, a platform thread pool when concurrency is modest, an event loop when the whole stack is non-blocking, and multiple processes where isolation or a GIL demands it. Then size the **downstream** pools, because that is where the limit really is.

---

## 11.3 Little's Law and Capacity Arithmetic

### Definition

**Little's Law** states that in any stable system, the average number of items in the system equals the average arrival rate multiplied by the average time each item spends in the system:

```text
 L = λ × W
 L  average number of requests in the system (concurrency, "in flight")
 λ  average arrival rate = throughput, in a stable system (requests per second)
 W  average time a request spends in the system (latency, in seconds)
```

It holds for a whole server, a single pool, a queue, or a database, provided you measure L, λ and W for the same boundary and the system is stable (on average, as many requests leave as arrive). It needs no assumption about the distribution of arrivals or service times.

### Why it exists

It is the bridge between the three numbers everyone measures. Given any two, it gives the third: how many threads or connections a target load needs, what throughput a pool can support, and what happens to concurrency when latency changes. It also exposes the most dangerous dynamic in servers: **when latency rises, the concurrency needed to keep the same throughput rises with it**, and pools are finite.

### Mechanism: three worked examples

**Example 1: how much concurrency does a target need?** A service must handle **2,000 requests per second** at **150 ms** average latency.

```text
 L = 2,000 /s × 0.150 s = 300 requests in flight
```

A platform-thread server needs at least 300 threads (more for headroom and variance); an event-loop or virtual-thread server must be able to hold 300 in-flight requests, and the downstream pools must support whatever share of those 300 hold their resources at once.

**Example 2: a downstream slows from 50 ms to 2 s.** A service handles **1,000 requests per second**; each request makes one call to a recommendation service that normally takes **50 ms**, plus **10 ms** of its own work. It has **200 worker threads** (a common default for Tomcat at the time of writing).

```text
 normal:   W = 0.060 s     L = 1,000 × 0.060 = 60 threads busy        (of 200: fine)

 the recommendation service slows to 2 s:
           W = 2.010 s     L needed = 1,000 × 2.010 = 2,010 threads   (only 200 exist)

 maximum throughput with 200 threads: λ = L / W = 200 / 2.010 ≈ 100 requests/s
 offered: 1,000 requests/s → 900 requests/s more arrive than can be served
```

Within a fraction of a second all 200 threads are blocked waiting for the slow service; new requests queue (Section 11.4), queue time grows by about 0.9 seconds every second, clients time out, and **every** endpoint of the service fails, including those that never call the recommendation service, because they wait for the same threads. One slow dependency has become a total outage. The fixes follow directly from the arithmetic: a **timeout** on the call (cap W; Chapter 10, Section 10.10), a **bulkhead** that limits how many threads may wait on that dependency (Chapter 24), and a **fallback** (serve the page without recommendations).

**Example 3: what can a connection pool support?** A database pool has **10 connections**. A request holds a connection for the duration of its three queries, **15 ms** in total.

```text
 maximum throughput through the pool = L / W = 10 / 0.015 s ≈ 667 requests/s
```

Now a developer wraps the request in a transaction that **also includes a 200 ms HTTP call** to a payment provider, holding the connection the whole time:

```text
 W per connection = 0.215 s  →  maximum throughput = 10 / 0.215 ≈ 46 requests/s
```

The same pool now supports less than a fifteenth of the load. The lesson: what matters for a pool is **how long each user holds it**, and holding a connection (or a lock) across slow, unrelated work divides capacity (Chapter 18 covers transaction boundaries).

### Mechanism: pool sizing

Chapter 5 (Section 5.11) gave the classic sizing formula for a pool of threads doing mixed work:

```text
 threads ≈ N_cores × U_target × (1 + W / C)
 W/C = ratio of waiting time to computing time per task
```

For the workload of Section 11.2 (4 cores, 80% target utilisation, 98 ms waiting / 2 ms computing):

```text
 threads ≈ 4 × 0.8 × (1 + 98 / 2) = 4 × 0.8 × 50 = 160
```

That agrees with Little's Law (100 needed at the average, plus headroom). Two cautions:

- **CPU-bound pools** (W/C near 0) should be close to the core count. More threads only add context switching.
- **The formula sizes the thread pool, not the downstream.** If those 160 threads each hold a database connection for most of their waiting time, the database pool must support it, and the database must cope with that many concurrent queries (Section 11.5). Sizing usually works **backwards** from the scarcest resource: how many concurrent queries does the database handle well, how long is each connection held, therefore what throughput can each instance and the whole fleet support.

### Guarantees and non-guarantees

Little's Law guarantees the relationship between averages in a stable system. It does not describe a system that is not stable (when arrivals exceed capacity, L grows without bound and the averages are meaningless), it says nothing about the tail (p99 latency needs more than averages; Chapter 26), and it requires the three numbers to be measured over the same boundary and period.

### Failure and diagnosis

- **Using the wrong boundary.** Measuring W at the handler but L for the whole server (including queue time) gives inconsistent numbers. Always pick one boundary: "the database pool", "the server from accept to last byte".
- **Sizing for the average only.** Traffic and latency vary; a pool sized for exactly the average is saturated half the time. Provision for peaks and keep utilisation of each pool well below 100% (Section 11.8 shows why).

### Backend connection

Keep a one-page capacity model for each service: target throughput and latency, the concurrency Little's Law implies at each pool (threads, database connections, HTTP client connections), and the throughput each pool caps. When a dependency's latency changes, rerun the numbers before production does it for you.

---

## 11.4 Queues Everywhere

### Definition

A **queue** is a place where a request waits for a resource that is busy. Between a client and the database, a request may wait in half a dozen queues, each with its own capacity (**bounded** or **unbounded**) and its own behaviour when full.

### Why it matters

Queues are where latency hides. A request whose handler takes 20 ms can take 3 s end to end because it waited 2.98 s in queues no dashboard shows. And an **unbounded** queue converts overload into ever-growing latency instead of errors, so the system looks healthy (no errors, normal CPU) right up to the moment it fails completely.

### Mechanism: the queues on one request's path

```text
 client
   │
   ▼
 [1] load balancer: connection and request queues          (Chapter 10, Section 10.3)
   │
   ▼
 [2] kernel accept queue (listen backlog)                    invisible to the application
   │                                                         (Chapter 4, Section 4.6)
   ▼
 [3] server connection limit: connections beyond it wait in the kernel
   │
   ▼
 [4] executor / worker queue: parsed requests waiting for a free worker thread
   │
   ▼
 handler running ──► [5] DB connection-pool wait queue: threads waiting for a free connection
   │                         │
   │                         ▼
   │                  [6] inside the database: lock waits, its own CPU run queue,
   │                      I/O queues (Chapters 15–17)
   │
   └──────────────► [7] HTTP client pool wait queue: waiting for a connection to another service
                            │
                            ▼
                     that service's queues [1]–[7] all over again
```

Each request's latency is the sum of its waits plus its actual work. Under light load the waits are near zero; near saturation they dominate (Section 11.8).

### Mechanism: why unbounded queues hide overload

Suppose a server can complete **500 requests per second** and **600 per second** arrive. The extra 100 per second must go somewhere:

```text
 bounded queue (say 200)                      unbounded queue
 t = 2 s   queue full                         t = 2 s    queue 200, wait 0.4 s
 after:    100 req/s rejected at once         t = 60 s   queue 6,000, wait 12 s
           (fast 503s; 500 req/s served       t = 300 s  queue 30,000, wait 60 s
            within normal latency)                       memory grows; every request
                                                         now exceeds the client's timeout:
                                                         the server is busy 100% of the
                                                         time answering clients that left
```

With the unbounded queue, the server keeps working at full capacity, yet its **goodput** (useful completed work) falls toward zero, because every response arrives after the client has given up. There is no error rate to alert on until something runs out of memory. The bounded queue sheds the excess immediately and keeps serving 500 requests per second **within their deadlines**. Bounded queues turn overload into a visible, fast error that clients and balancers can act on (retry elsewhere, back off).

**Order matters too.** Under overload, a first-in-first-out queue serves the **oldest** requests first: precisely the ones most likely to have timed out already. Some systems switch to last-in-first-out, or drop requests whose queue time already exceeds their deadline, when queues build up; Chapter 24 covers such policies.

### Concrete: where one request's 900 ms went

A service at about 85% utilisation of its worker pool reports a p50 of 900 ms for an endpoint whose handler "only takes 90 ms". Timestamps at each stage show where the time went:

```text
 stage                                   time      kind
 load balancer queue                       2 ms    wait
 kernel accept queue                       0 ms    wait
 executor queue (waiting for a worker)   380 ms    wait   ← 85% busy pool: queues build (§11.8)
 handler CPU                               4 ms    work
 DB connection-pool wait                 310 ms    wait   ← 10 connections, held too long elsewhere
 DB query (25 ms of it a row-lock wait)   40 ms    work + wait
 HTTP client pool wait (pricing service) 120 ms    wait   ← per-host pool of 20 is full
 pricing service call                     44 ms    work (theirs)
 ───────────────────────────────────────────────────────
 total                                   900 ms    of which ~90 ms is anyone's actual work
```

Each queue obeys Little's Law on its own: at 400 requests per second, an average executor wait of 0.38 s means about 400 × 0.38 ≈ 152 requests sitting in the executor queue at any moment, a number a queue-length metric confirms. The fixes differ per line: the executor wait falls if handler time falls or capacity rises; the pool wait falls when connections are held for less time (Section 11.5); the client-pool wait points at the pricing service's latency or the pool's size. Without per-stage timing, all of this is reported as "the handler is slow".

**Queue order under overload.** Clients time out after 1 s, and under overload the executor queue holds 2 s of work. With first-in-first-out order, every request a worker picks up has already waited about 2 s: its client has left, so the work is wasted and goodput is zero even though the server is busy. If workers **drop requests whose deadline has passed** when they take them from the queue (and some systems switch to serving the newest requests first when the queue is long), the server spends its capacity on requests that can still be delivered, and goodput recovers to roughly its capacity. Chapter 24 covers these policies.

### Concrete: measuring queue time

Instrument at least three timestamps per request: when it arrived at the server (or at the load balancer), when a worker started on it, and when the handler finished. Their differences give **queue time** and **service time** separately. A rise in queue time with flat service time is the signature of saturation; a rise in service time is the signature of a slow dependency or more work per request.

### Guarantees and non-guarantees

A bounded queue guarantees a maximum wait and bounded memory; it does not guarantee that rejected requests are retried sensibly (Section 11.8). An unbounded queue guarantees that nothing is rejected, which is exactly its danger: it guarantees nothing about latency or memory.

### Failure and diagnosis

- **Latency climbs for minutes, then the process dies of out-of-memory.** An unbounded executor queue (a common default in thread-pool libraries) absorbing overload. Evidence: queue length metrics rising linearly, heap usage rising with it, normal error rate until the crash.
- **Clients time out, but server logs show requests "completing successfully" much later.** Requests waited in a queue past their client's timeout. Evidence: server-side queue time larger than the client timeout; access logs with `499`-style client-closed codes (Chapter 10, Section 10.8).

### Backend connection

Every queue in your service should be **bounded**, **measured** (length and wait time) and have a deliberate **rejection behaviour**. The limits should be set so that a request that waits in all of them still finishes within its deadline.

---

## 11.5 Resource Pools as Separate Bottlenecks

### Definition

A **resource pool** holds a fixed number of expensive, reusable resources (database connections, HTTP connections to one host, permits to call a dependency) and hands them out to callers, who wait when none is free (Chapter 5, Sections 5.6 and 5.11 built pools from semaphores and queues). Each pool is a **separate capacity limit** with its own wait queue, independent of the thread pool.

### Why it matters

A server's thread pool is rarely its real limit. Requests need several pools, and the smallest effective one decides throughput. Worse, the obvious reaction to slowness ("add threads", "add instances") often increases pressure on the shared resource behind the pool and makes everything slower.

### Concrete: the pool-starvation timeline

A service has **200 worker threads** and a database pool of **10 connections** (typical defaults for Tomcat and HikariCP at the time of writing). Each request holds a connection for **20 ms**. Normally fine: the pool supports 10 / 0.02 = 500 requests per second. Then one endpoint starts running a query that takes **2 s**.

```text
 t = 0.0 s   slow requests begin arriving at 10/s among 400/s of normal traffic
 t = 0.0–1.0 each slow request holds a connection for 2 s; after 1 s, all 10 connections
             are held by slow queries (10/s × 2 s = 20 would be needed; only 10 exist)
 t = 1.0 s   normal requests (20 ms of DB work each) now wait for a connection behind
             slow ones: pool wait time jumps from ~0 to hundreds of milliseconds
 t = 1.5 s   200 worker threads: ~190 of them are parked in "get connection from pool",
             waiting; the rest are running slow queries
 t = 2.0 s   new requests wait in the executor queue (Section 11.4): no free threads
 t = 30 s    the pool's acquisition timeout (30 s by default in HikariCP) expires for
             the oldest waiters: "Connection is not available, request timed out"
             errors begin; clients timed out long ago
 result:     every endpoint fails, although only one is slow and the database CPU is low
```

The fix is not more threads (they would only join the wait) and usually not a bigger pool (Section below): it is a **timeout** on the slow query (a statement timeout) and on pool acquisition (short, a few hundred milliseconds, so requests fail fast), a **separate pool or bulkhead** for the slow endpoint so it cannot take all the connections (Chapter 24), and fixing the query (Chapter 16).

### Mechanism: why more threads, connections or instances can make it worse

The database is a server too, with finite cores, disks and locks. Increasing concurrency **at** it does not increase its capacity:

```text
 database throughput vs concurrent queries (shape)

 throughput
   ▲                ●●●●●
   │            ●●●       ●●●
   │         ●●              ●●●
   │       ●                    ●●●     beyond the peak: more concurrency, LESS throughput
   │     ●                          ●●●
   │   ●
   └──────────────────────────────────────►  concurrent queries
        ~ a small multiple of the database's cores
```

Beyond a point (often a small multiple of the database server's cores, depending on how much of each query waits on I/O), extra concurrent queries add **contention**: more context switches (in PostgreSQL, each connection is a separate OS process; Chapter 3, Section 3.9), more lock waits on hot rows and internal structures (Chapter 17), more pressure on the buffer cache (Chapter 15). Throughput flattens and then falls, and every query gets slower. A commonly cited starting point for a PostgreSQL pool is about **twice the database's core count plus the number of independent disks**, then adjust by measurement; HikariCP's documentation discusses the same reasoning.

**Instances multiply pools.** 20 instances × a pool of 20 = **400 connections**. PostgreSQL's default `max_connections` is 100 at the time of writing; with 400 configured, new instances fail to connect at startup or under load, and if the limit is raised, the database drowns in concurrent queries. Scaling out the application tier **without** sizing the total connection count can take down the database. Connection poolers in front of the database (such as PgBouncer) and careful per-instance pool sizes address it (Chapter 18).

**HTTP client pools behave the same way.** A client pool that allows, say, 20 connections per destination host caps calls to that service at 20 in flight; a slow dependency fills it and requests queue in the client's own wait queue (queue [7] in Section 11.4). Too large a pool simply moves the overload onto the dependency.

### Guarantees and non-guarantees

A pool guarantees that no more than N callers use the resource at once, protecting it. It does not guarantee fairness, does not guarantee short waits (unless acquisition has a timeout), and cannot add capacity to the resource behind it.

### Failure and diagnosis

- **Pool exhaustion.** Evidence: pool metrics (active = maximum, pending > 0, rising acquisition time); thread dumps with many threads parked in the pool's acquire method (Section 11.10); error messages from the pool about acquisition timeouts. Then find who holds the connections: long queries (the database's own view of active queries), or connections held across slow non-database work (traces showing spans inside a transaction).
- **Connection limit at the database.** Evidence: `too many connections` (or similar) errors from the database during scale-out or deploys, when old and new instances overlap.

### Backend connection

Size pools **backwards from the shared resource**: how much concurrency it handles well, divided among instances. Give every pool a short acquisition timeout and metrics; keep connections out of slow, non-database work; and treat "add more threads" as a hypothesis to test with Little's Law, not a fix.

---

## 11.6 Thread Safety in Application Code, and Request Context

### Definitions

A **singleton service** is an object of which the application creates one instance, shared by every request (the default lifetime of components in most dependency-injection containers, Chapter 12). **Static or global state** is shared by the whole process.

**Request context** is the set of values that describe the current request and must follow it through all the code that handles it: the authenticated user, the trace and request IDs, the locale, the database transaction. A **thread-local** variable gives each thread its own copy of a value (Chapter 3, Section 3.7), and is the traditional way frameworks keep request context available without passing it as a parameter.

### Why it exists

Frameworks create services once and share them, because creating a service per request would be wasteful. That makes every field of a service shared state. And frameworks keep request context in thread-locals because, in thread-per-request servers, "the current thread" means "the current request". Both assumptions break in predictable ways: fields used for request data, and thread-locals that do not follow work onto other threads.

### Concrete: the shared-state bug in a singleton service

```pseudo
class InvoiceService:                     // one instance, shared by all requests
    currentCustomer = null                // BUG: request data in a field

    function generate(customerId):
        currentCustomer = customers.load(customerId)
        lines = buildLines()              // reads currentCustomer
        return render(lines)              // reads currentCustomer

    function buildLines():
        return orders.forCustomer(currentCustomer.id)
```

```text
 thread T1 (request for Alice)              thread T2 (request for Bob)          field currentCustomer
 currentCustomer = Alice                                                         Alice
                                            currentCustomer = Bob                Bob
 buildLines(): orders for currentCustomer → Bob's orders                         Bob
 render(): Alice's invoice with Bob's orders                                     Bob
```

Alice receives an invoice containing Bob's orders: a data leak, not just a wrong total. It never happens in a single-user test, and under load it happens rarely and unpredictably, the race condition of Chapter 5 (Section 5.2) in its most common server form.

**The fix: keep request data in locals and parameters.**

```pseudo
class InvoiceService:                     // stateless: no mutable fields
    function generate(customerId):
        customer = customers.load(customerId)      // local to this call
        lines = buildLines(customer)
        return render(customer, lines)

    function buildLines(customer):
        return orders.forCustomer(customer.id)
```

Services should be **stateless**: their fields hold only collaborators (other services, configuration) set once at startup. Mutable shared state that is genuinely needed (a cache, a counter, a rate limiter) belongs in a thread-safe structure: a concurrent map with atomic compound operations (`computeIfAbsent`, Chapter 5, Section 5.10), an atomic counter, or a dedicated component with its own locking. A plain map used as a cache, a non-thread-safe formatter or parser kept in a field, and lazily initialised fields without safe publication (Chapter 5, Section 5.3) are the usual offenders.

### Mechanism: request context, thread-locals and their two failure modes

```text
 thread-per-request: the request's context lives in thread-locals of its worker thread

 worker T7 ── request A: set user=alice, traceId=a1 ──► handler ──► clear ──► request B: set user=bob ...
```

**Failure 1: leaking between requests.** Pool threads are reused. If a thread-local is set at the start of a request and **not cleared** at the end (an exception path skips the cleanup), the next request on that thread starts with the previous request's user or trace ID. Always clear in a `finally` block, or in middleware that wraps the whole request (Chapter 10, Section 10.6).

**Failure 2: lost across thread boundaries.** As soon as work moves to another thread, the thread-locals stay behind:

```pseudo
function placeOrder(req):                 // runs on worker T7; user = alice in a thread-local
    future = executor.submit(() ->
        audit.record("order placed", currentUser()))   // runs on pool thread P2: currentUser() is null
                                                        // (or, worse, a stale value left on P2)
```

The same happens with callbacks and continuations in futures (Chapter 5, Section 5.12), event-loop code, and reactive pipelines, where one request's code may run on many threads. The fixes:

- **Pass context explicitly** as a parameter or a context object. Clearest and safest.
- **Capture and restore**: wrap tasks so that the submitting thread's context is captured when the task is created and installed on the executing thread for the task's duration, then cleared. Context-propagation libraries (OpenTelemetry's, for tracing) and framework hooks do exactly this.
- **Use the runtime's scoped context mechanism** where it exists: Go passes a `context.Context` value explicitly through calls; recent JDKs provide scoped values, which bind a value for the duration of a call tree and are inherited by structured child tasks.

> **Framework lens:** Spring keeps several kinds of context in thread-locals: the security context (`SecurityContextHolder`, thread-local by default), the transaction (`@Transactional` binds the transaction to the current thread), the request attributes behind **request-scoped** beans (a request-scoped bean injected into a singleton is a proxy that looks up the current request's instance on each call), and the logging MDC. Consequences: a method annotated `@Async` runs on another thread **without** the caller's security context, MDC or transaction; it is not part of the caller's transaction at all. Spring offers `TaskDecorator` (to copy context onto task threads) and `DelegatingSecurityContextExecutor` for the security context; the transaction must be started in the `@Async` method itself if it needs one.

### Guarantees and non-guarantees

A stateless service guarantees that concurrent requests cannot interfere through it. Thread-locals guarantee per-thread isolation, which equals per-request isolation **only** while one request runs entirely on one thread and the values are cleared afterwards. Nothing propagates context across threads automatically unless a mechanism is configured to.

### Failure and diagnosis

- **Users seeing other users' data.** Search services for mutable fields and static variables touched by request handling; reproduce with a concurrent load test that checks every response belongs to its caller.
- **Missing user or trace ID in logs of async work.** Context not propagated to the executor. Evidence: log lines from pool threads with empty MDC fields; traces that break into separate pieces at async boundaries.
- **Stale context.** A request is processed with the previous request's tenant or user. Evidence: correlation of the wrong context with a particular pool thread name. Fix: clear in `finally`, set context in outermost middleware.

### Backend connection

Write services without mutable fields; put shared mutable state only in purpose-built thread-safe components; set request context in outermost middleware and clear it in `finally`; and every time work crosses a thread or an async boundary, decide how its context travels.

---

## 11.7 Coordination Across Instances

### Definition

**Cross-instance coordination** is making several replicas of a service (several processes, usually on several machines, Chapter 1, Section 1.8) behave correctly when they act on the same data or the same task. In-process tools (locks, atomics, synchronized blocks, in-memory maps) coordinate **threads within one process** only.

### Why it matters

Every production service runs several instances behind a load balancer. Code that is correct with one instance (a lock around "check stock, then decrement", a scheduled job, an in-memory cache, a per-process rate limiter) becomes wrong with two, and nothing in the code shows it.

### Mechanism: why in-process locks fail

```text
 instance A (its own lock L)                  instance B (its own, separate lock L)
 lock(L)                                      lock(L)                  ← both succeed: different locks
 stock = db.read(item 7) → 1                  stock = db.read(item 7) → 1
 db.write(item 7, stock - 1) → 0              db.write(item 7, stock - 1) → 0
 confirm order                                confirm order            ← the last unit sold twice
 unlock(L)                                    unlock(L)
```

This is Chapter 5's check-then-act race (Section 5.2), now between processes. Each instance's lock protects only that instance's threads.

### Mechanism: what to use instead

Coordination must happen in a component that **all instances share**, usually the database:

1. **An atomic conditional update** (the most common fix): `UPDATE items SET stock = stock - 1 WHERE id = 7 AND stock > 0`, then check that one row was updated. The database makes the check and the act one step (Chapter 5, Section 5.2's rule, enforced by the system of record).
2. **Constraints**: a unique index on `(order_id)` in a payments table makes a second insert of the same payment fail, whichever instance tries it (Chapter 14).
3. **Optimistic concurrency** with a version column (Chapter 7, Section 7.13): write only if the version is unchanged.
4. **Row locks** taken in a transaction (`SELECT ... FOR UPDATE`), held until commit: pessimistic concurrency control in the database (Chapter 17).
5. **A distributed lock** (a lease in a coordination service or a database row), for work that is not a single database update; it needs **fencing tokens** to stay safe when a lock holder pauses and its lease expires (Chapter 23).
6. **Single-writer routing**: send every operation for one key (one account, one order) to the same partition of a message queue, consumed by one worker at a time, so that operations on a key are naturally serialised (Chapter 25).

### Concrete: two instances, one last unit, in SQL

The atomic conditional update from fix 1, executed by two instances at the same moment for the last unit of item 7, in PostgreSQL or MySQL/InnoDB at their default isolation levels:

```sql
-- instance A                                         -- instance B
BEGIN;                                                BEGIN;
UPDATE items SET stock = stock - 1
 WHERE id = 7 AND stock > 0;
-- UPDATE 1   (A now holds the row lock on item 7)
                                                      UPDATE items SET stock = stock - 1
                                                       WHERE id = 7 AND stock > 0;
                                                      -- blocks: waits for A's row lock
COMMIT;                                               -- unblocks; re-evaluates the condition
                                                      -- against the committed row: stock = 0
                                                      -- UPDATE 0
                                                      COMMIT;
-- application: 1 row → confirm the order             -- application: 0 rows → "out of stock"
```

Both instances ran the same code; the database serialised the two updates on the row and evaluated the condition atomically with each change. The application's only job is to **check the affected-row count** and act on it. (Why the second statement re-checks the condition after waiting, and how isolation levels change such behaviour, is Chapter 17.)

The same idea, a single atomic statement whose outcome tells each instance whether it won, also makes a scheduled job run once per period across all instances:

```sql
-- every instance tries this when the monthly job fires; only the first insert succeeds
INSERT INTO job_runs (job, period, started_by)
VALUES ('monthly-statements', '2026-10', 'instance-3')
ON CONFLICT (job, period) DO NOTHING;        -- PostgreSQL syntax; returns 1 row for the winner, 0 for others
```

The primary key on `(job, period)` is the lock. Only the instance that inserted the row runs the job. A crash mid-run needs a status column and a recovery rule (a lease, Chapter 23).

### Other single-instance assumptions that break

- **Scheduled jobs run on every instance.** A nightly "send invoices" job scheduled in code runs once per instance: ten instances send ten copies. Fix: a lock or lease so only one runs (in Java, libraries such as ShedLock do this with a database row), or a separate scheduler that triggers the job once.
- **In-memory caches diverge.** Instance A updated its cached price; instance B still has the old one. Fix: short lifetimes, invalidation messages, or a shared cache (Chapter 21).
- **Per-instance rate limits.** "100 requests per minute per user" enforced in memory becomes 100 per minute **per instance**, up to 1,000 with ten instances. Fix: a shared counter (Chapter 21) or limits enforced at the gateway (Chapter 10, Section 10.4).
- **In-memory sessions**: Chapter 7, Section 7.1's failure, and Chapter 19's subject.

### Guarantees and non-guarantees

In-process synchronisation guarantees mutual exclusion among threads of one process only. Database atomic statements and constraints guarantee correctness across instances for the data they cover. Distributed locks guarantee mutual exclusion only with leases and fencing, and only for clients that use them (Chapter 23).

### Failure and diagnosis

Overselling, duplicate emails or invoices, duplicate payments, inconsistent prices between requests: all appear only with more than one instance and only under concurrent load. Evidence: duplicates whose log lines come from **different instance identifiers** at almost the same time. That one field in the logs distinguishes cross-instance races from in-process ones.

### Backend connection

Assume from day one that every service runs as several instances. Put every invariant that must hold across requests ("stock never negative", "one payment per order", "one invoice per month") into the shared system of record as an atomic statement, a constraint or a lock, and keep in-process locks for in-process data only.

---

## 11.8 Overload: Saturation, Collapse, Backpressure, Shedding

### Definitions

- **Utilisation** (ρ) of a resource is the fraction of time it is busy: arrival rate × service time per request, divided by the number of servers (threads, cores, connections) behind it.
- **Saturation** is the state in which a resource is fully used and work waits for it.
- **Overload** is offered load above capacity: more work arrives per second than can be completed.
- **Goodput** is useful work completed: responses delivered within their deadline. Under overload, throughput (work done) and goodput (work that mattered) diverge.
- **Backpressure** is a signal from a busy component to whoever feeds it to slow down, propagated upstream until it reaches the source of the load.
- **Admission control** decides at the entrance whether to accept a request at all.
- **Load shedding** rejects some work deliberately and cheaply so that the rest completes.

### Why it matters

Servers are not linear. Below saturation, latency barely changes with load; near it, latency rises steeply; past it, without protection, it collapses: goodput falls toward zero while every resource is busy. Clients then retry, adding more load. Understanding the shape of this curve is what lets you set limits, alerts and capacity before an incident sets them for you.

### Mechanism 1: why latency explodes near capacity

Requests do not arrive evenly; they come in random bursts. Even when the average arrival rate is below capacity, bursts create queues, and the busier the resource, the longer each burst takes to clear. For the simplest queueing model (one server, random arrivals and service times, the "M/M/1" queue), the average time in the system is:

```text
 W = S / (1 − ρ)        S = service time when idle, ρ = utilisation

 ρ = 0.50  →  W = 2 × S
 ρ = 0.70  →  W = 3.3 × S
 ρ = 0.80  →  W = 5 × S
 ρ = 0.90  →  W = 10 × S
 ρ = 0.95  →  W = 20 × S
 ρ = 0.99  →  W = 100 × S
```

Real systems with many servers and less variable work behave better than this single-server model, but the shape is universal: a **knee** around 70–85% utilisation, after which small increases in load cause large increases in latency. That is why capacity is planned to keep critical resources well below 100% at peak.

```text
 latency
   ▲
   │                                             ╱ collapse: queues grow without bound,
   │                                            ╱  requests time out, retries add load;
   │                                          ╱    goodput → 0
   │                                        ╱
   │                                     ╱   ← the knee (~70–85% utilisation)
   │                              ___──
   │________________________─────
   └────────────────────────────────────────────────────►  offered load
   0%                                     100% capacity
```

### Mechanism 2: how overload feeds itself

```text
 1. load crosses capacity (a traffic spike, or a dependency slows and capacity drops: §11.3)
 2. queues grow; latency rises past clients' timeouts
 3. clients time out and RETRY: offered load rises further (each layer that retries multiplies
    it: 3 retries at each of 3 layers can turn one request into 27, Chapter 10, Section 10.4)
 4. the server spends its capacity on requests whose clients have already left
 5. goodput collapses; the incident outlasts its trigger
```

The last line has a name: a **metastable failure**. The original trigger (a brief spike, a slow minute at the database) is gone, but retries and full queues keep the load above capacity, so the system stays down until load is cut from outside (shedding, disabling retries, restarting with traffic held back). Systems avoid it by making overload **visible and cheap**: bounded queues, fast rejections, and clients that back off instead of hammering.

### Mechanism 3: the defences (concepts; mechanisms in Chapter 24)

- **Bounded queues** everywhere (Section 11.4): excess work is refused at once instead of waiting until it is useless.
- **Backpressure**: when a component is busy, its callers must feel it. Within a process, a bounded queue blocks or rejects producers (Chapter 5, Section 5.7's producer–consumer). Between machines, TCP flow control slows senders whose receivers stop reading (Chapter 6, Section 6.8). In HTTP, a `429` or `503` with `Retry-After` tells clients to slow down (Chapter 7, Section 7.5). In reactive streams, consumers explicitly request how many items they can take (Section 11.9). Backpressure only works if every layer honours it; one unbounded buffer anywhere in the chain absorbs the signal.
- **Admission control**: cap the number of requests in progress per instance (a concurrency limit), and reject beyond it. **Adaptive** limits adjust the cap from observed latency, much as TCP congestion control adjusts its window (Chapter 6, Section 6.8): raise it slowly while latency is good, cut it when latency rises.
- **Load shedding**: when overloaded, reject the least important work first and cheaply, before doing any expensive processing: health checks and paying users' checkouts survive, analytics and recommendation calls are dropped.
- **Deadlines and cancellation**: drop requests whose deadline has already passed (Chapter 10, Section 10.10) rather than doing work nobody will receive.
- **Client discipline**: timeouts, retries with exponential backoff and jitter, retry budgets, and circuit breakers that stop calling a failing dependency (Chapter 24).

### Guarantees and non-guarantees

Bounded queues and admission control guarantee that a server's latency for **accepted** work stays bounded under overload, and that the excess is visible as rejections. They do not guarantee that clients react sensibly to rejections, that the right requests are the ones rejected (without priorities), or that the system recovers by itself after a metastable failure.

### Failure and diagnosis

- **The cliff.** Latency flat at 50 ms up to 800 requests per second, 3 s at 900. Evidence: a load test plotted as latency against load shows the knee; utilisation of some resource (threads, a pool, database CPU) approaches 100% at the same point. That resource is the bottleneck.
- **The incident that does not end.** The trigger is fixed, but error rates stay high. Evidence: request rates far above normal traffic (retries), queues full, goodput near zero. Recovery: cut load (shed at the edge, pause retrying clients, scale out with traffic held back), then restore gradually.

### Backend connection

Load-test to find each service's knee, keep peak utilisation well below it, alert on queue time and pool saturation rather than only on errors, make overload fail fast and visibly, and make sure every client of your service (including your own other services) backs off.

---

## 11.9 Blocking, Non-Blocking and Reactive APIs

### Definitions

An API's **style** is how a caller receives a result:

- **Blocking**: the call returns the result (or throws) when it is ready; the calling thread waits.
- **Non-blocking / asynchronous**: the call returns immediately with a **future** or takes a **callback**, and the result arrives later (Chapter 5, Section 5.12).
- **Reactive streams**: the call returns a **publisher** of zero or more items over time; a **subscriber** receives them and controls the flow by **requesting** how many it can handle (the reactive-streams model, with its `request(n)` demand signal).

The **execution model** (Section 11.2) is which threads actually run the work and wait.

### Why the distinction matters

The two are often confused: "reactive is faster", "async means non-blocking", "virtual threads make reactive obsolete". The style decides how code is written and composed; the model decides what waiting costs. Each style can sit on several models, with very different results.

### Mechanism: style versus model

| API style | On platform threads | On an event loop | On virtual threads / goroutines |
|---|---|---|---|
| **Blocking** | classic thread-per-request; waiting costs a thread | **wrong**: blocks the loop for everyone | cheap waiting with ordinary code |
| **Futures / callbacks** | possible; waiting costs nothing if the library is truly non-blocking, but continuations run on some pool | natural fit (Node, Netty) | possible; little benefit over blocking code |
| **Reactive streams** | possible but wasteful | natural fit (Spring WebFlux on Netty, Project Reactor) | possible; its benefit is backpressure and composition, not cheaper waiting |

Two consequences:

- **One blocking call in a non-blocking stack ruins it.** A reactive service on an event loop that calls a blocking database driver (classic JDBC is blocking) stalls the loop thread for the whole query; all other requests on that thread wait. Such stacks need non-blocking drivers throughout (R2DBC is a reactive alternative to JDBC), or must move blocking calls to a separate thread pool explicitly. Tools exist that detect blocking calls on loop threads during testing (BlockHound, for Reactor).
- **Async style on top of blocking libraries is not non-blocking.** Wrapping a blocking call in a future that runs on a thread pool frees the caller's thread but still occupies a pool thread for the whole wait.

### Concrete: one operation in three styles

The operation: build a dashboard by fetching a user's profile and their recent orders **concurrently** from two services, combining the results, with an overall deadline of 800 ms.

**Blocking code on virtual threads** (structured concurrency):

```pseudo
function dashboard(userId):                                // runs on a virtual thread
    with scope = TaskScope(deadline = now() + 800ms):
        profile = scope.fork(() -> profileClient.get(userId))      // blocking call, own virtual thread
        orders  = scope.fork(() -> orderClient.recent(userId))     // blocking call, own virtual thread
        scope.join()                                       // waits for both; at the deadline, or if
                                                           // either fails, cancels the other
    return Dashboard(profile.result(), orders.result())
```

**Futures** (non-blocking clients, Chapter 5, Section 5.12):

```pseudo
function dashboard(userId): Future<Dashboard>
    profileF = profileClient.getAsync(userId)              // returns at once with a future
    ordersF  = orderClient.recentAsync(userId)
    return allOf(profileF, ordersF)
           .orTimeout(800ms)
           .thenApply(_ -> Dashboard(profileF.join(), ordersF.join()))
```

**Reactive streams** (publishers that do nothing until subscribed):

```pseudo
function dashboard(userId): Mono<Dashboard>
    return zip(profileClient.get(userId),                  // Mono<Profile>
               orderClient.recent(userId))                 // Mono<List<Order>>
           .map((profile, orders) -> Dashboard(profile, orders))
           .timeout(800ms)
```

All three run the two calls concurrently and give up at 800 ms. The first reads like sequential code, and its stack traces show your own call chain. The second and third need every client to be non-blocking, and their error and context handling run through the library. For this request–response case, none is faster than the others; they differ in readability, debuggability and which runtime they need.

Where reactive streams do something the others cannot express as directly is **streaming with demand**:

```pseudo
// stream 2 million rows to a slow client without holding them in memory
function exportOrders(month): Flux<Line>
    return orderRepository.streamByMonth(month)          // a publisher reading rows from a cursor
           .map(row -> csvLine(row))                      // transform each row as it flows
           // the HTTP layer subscribes and requests more lines only as the client's
           // socket accepts them: a slow reader slows the database cursor, end to end
```

The demand signal travels from the client's TCP window (Chapter 6, Section 6.8), through the response writer, back to the cursor. Building the same with blocking code is possible (a loop that writes and flushes in chunks, Chapter 10, Section 10.6), but it is the reactive model's natural shape.

### When reactive is worth it

**Is reactive faster?** Not per request: the work is the same, and the extra machinery adds some overhead. What reactive gives is:

1. **Backpressure end to end**: streaming large or unbounded data (a result set of millions of rows, a feed of events to many clients) where a slow consumer must slow the producer instead of filling memory.
2. **High concurrency on few threads**, which mattered greatly when the only alternative was thousands of platform threads. Lightweight threads (Section 11.2) now give most of that benefit with plain blocking code.
3. **Declarative composition** of many concurrent operations (call five services, combine, time out, fall back) in one expression.

And what it costs: code that is harder to read and debug (stack traces show the reactive library's internals, not your call chain), context that must be propagated explicitly (Section 11.6), a requirement that **every** library in the request path be non-blocking, and a learning curve for the whole team.

A reasonable rule: use reactive streams where you genuinely need streaming with backpressure, or where the whole stack is already non-blocking; for ordinary request–response services that spend their time waiting on databases and APIs, blocking code on virtual threads (or a well-sized thread pool) is simpler and performs comparably.

> **Framework lens:** Spring offers both stacks: Spring MVC (the Servlet API, blocking, thread per request; Chapter 10, Section 10.7) and Spring WebFlux (reactive, on Netty's event loop by default), with Project Reactor's `Mono` (zero or one item) and `Flux` (zero to many) as the publisher types. Calling a blocking API inside a WebFlux handler without moving it to a separate scheduler blocks a Netty event-loop thread. With virtual threads enabled, Spring MVC's blocking model handles high concurrency without the reactive programming model.

### Guarantees and non-guarantees

Reactive streams guarantee that a subscriber never receives more items than it requested. Non-blocking APIs guarantee that the calling thread is not held while waiting. Neither guarantees that the underlying libraries are non-blocking, that the system is faster, or that downstream capacity is larger.

### Failure and diagnosis

- **A reactive service slower than the blocking one it replaced.** Evidence: event-loop threads blocked in a JDBC or file call (thread dump or a blocking-call detector), few loop threads and long queues. Fix: non-blocking drivers, or isolate blocking calls on a bounded pool.
- **Lost context in reactive pipelines.** Logs without trace IDs, security context missing in later operators. Fix: the library's context mechanism (Reactor's context) and context-propagation integration.

### Backend connection

Choose the API style for how the code must compose and stream, and the execution model for how cheaply it must wait. Do not adopt reactive programming for speed; adopt it for backpressure and streaming, or not at all.

---

## 11.10 Diagnosis: Reading Thread Dumps and Telling Bottlenecks Apart

### Definition

A **thread dump** is a snapshot of every thread in a process at one instant: its name, its state (running, blocked on a lock, waiting, parked) and its stack of method calls. A **bottleneck signature** is the combination of a thread dump, CPU use and pool metrics that identifies which resource is limiting a server.

### Why

When a server is slow or stuck, the most direct evidence is **what every thread is doing right now**. Chapter 5 (Section 5.14) explained how to take a thread dump and what the thread states mean. This section applies it to the server failures of this chapter.

### Concrete: an annotated thread dump

The format below is a JVM's (as printed by `jstack` or `jcmd <pid> Thread.print`); other runtimes produce similar listings (Go's goroutine dump on `SIGQUIT`, Python's `faulthandler`). Line numbers and addresses are shortened.

```text
"http-nio-8080-exec-17" #63 daemon prio=5 ... waiting on condition
   java.lang.Thread.State: TIMED_WAITING (parking)                     ← parked with a timeout
        at jdk.internal.misc.Unsafe.park(Native Method)
        - parking to wait for <0x...a1b2> (a java.util.concurrent.SynchronousQueue$TransferStack)
        at java.util.concurrent.SynchronousQueue.poll(...)
        at com.zaxxer.hikari.util.ConcurrentBag.borrow(...)            ← waiting to borrow
        at com.zaxxer.hikari.pool.HikariPool.getConnection(...)           a DB connection
        at ...OrderRepository.findOpen(...)                            ← from this code path
        at ...OrderController.list(...)

"http-nio-8080-exec-42" #88 daemon prio=5 ... runnable
   java.lang.Thread.State: RUNNABLE                                    ← "runnable", yet waiting:
        at sun.nio.ch.SocketDispatcher.read0(Native Method)               a thread blocked in a
        at sun.nio.ch.NioSocketImpl.read(...)                             native socket read shows
        at ...HttpClient ... readResponse(...)                            as RUNNABLE in the JVM
        at ...PaymentClient.charge(...)                                ← waiting on the payment API
        at ...CheckoutService.placeOrder(...)                          ← inside @Transactional
        at ...                                                            (holding a DB connection)

"http-nio-8080-exec-5" #51 daemon prio=5 ... waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)                 ← waiting for a lock
        at ...TokenProvider.getToken(TokenProvider.java)
        - waiting to lock <0x...c3d4> (a ...TokenProvider)              ← this monitor...
        at ...PartnerClient.call(...)

"http-nio-8080-exec-9" #55 daemon prio=5 ... runnable
   java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.SocketDispatcher.read0(Native Method)
        at ...AuthClient.refreshToken(...)                              ← ...is held by this thread,
        at ...TokenProvider.getToken(TokenProvider.java)                   which is doing network
        - locked <0x...c3d4> (a ...TokenProvider)                          I/O while holding it
```

How to read a dump of hundreds of threads:

1. **Group threads by their top frames.** Fifty threads with the same stack mean one thing is happening fifty times.
2. **For `BLOCKED` threads**, find the monitor address in "waiting to lock" and search for the one thread that shows "locked" with that address: that thread, and what it is doing while holding the lock, is the cause.
3. **For `WAITING` or `TIMED_WAITING` threads parked inside a pool or queue** (`getConnection`, `borrow`, `take`), the pool is exhausted; find the threads that **hold** its resources and what they are waiting for.
4. **For `RUNNABLE` threads in socket reads**, they are waiting on the network: see which client and which dependency.
5. **Take two or three dumps a few seconds apart.** Threads stuck in the same place across dumps are stuck; threads that move are just busy.

### Concrete: telling the bottlenecks apart

| Bottleneck | CPU | Thread dump | Pools and queues | Downstream |
|---|---|---|---|---|
| **CPU saturation** | near 100% of all cores | most threads `RUNNABLE` in **your** compute frames (serialisation, regexes, loops) | worker queue growing | normal |
| **Lock contention** | low to moderate | many threads `BLOCKED` on the **same** monitor (or parked on the same lock), one holder | worker queue growing | normal |
| **Pool exhaustion** | low | many threads parked in the pool's acquire method; holders doing something slow | pool at maximum with waiters; acquisition time high | maybe normal (holders waste connections) or slow |
| **Slow downstream** | low | many threads `RUNNABLE` in **socket reads** to the same dependency | HTTP client pool full; worker pool full | that dependency's latency high |
| **Event-loop block** | 100% of **one** core per process | the loop thread in a long compute frame or a blocking call | event-loop lag high | normal |

A CPU profile (sampling where CPU time goes, Chapter 26) confirms CPU saturation; a lock-contention profile (Chapter 5, Section 5.14) confirms contention; pool metrics and traces confirm exhaustion and slow downstreams.

### Backend connection

Make thread dumps (or their equivalent) easy to take in production, name thread pools meaningfully so that dumps are readable, export pool and queue metrics, and learn the five patterns in the table: most "the server is stuck" incidents are one of them.

---

## Chapter Summary

A server overlaps many requests. Each request's data must stay in **request scope** (locals, parameters, request objects), shared state must be read-only or deliberately thread-safe, and singleton services must have no mutable fields: the classic server concurrency bug is request data stored in a field of a shared service, which leaks one user's data into another's response. Request context in **thread-locals** must be cleared after each request and propagated explicitly whenever work moves to another thread or into async code.

On one workload (1,000 requests per second, 2 ms of CPU and 98 ms of waiting each), every **execution model** can hold the 100 concurrent requests that implies: a platform thread pool with about 100–200 threads, an event loop with two or more processes (but stalled by any blocking or CPU-heavy handler), virtual threads or goroutines cheaply (unless pinned), synchronous worker processes only at a prohibitive memory cost. None of them adds capacity to the database or the services behind it. **Little's Law**, L = λ × W, turns capacity questions into arithmetic: the concurrency needed for a target, the throughput a pool supports, and what happens when a downstream slows from 50 ms to 2 s: the concurrency needed rises fortyfold, the 200 threads that were plenty cap throughput at about 100 requests per second, and every endpoint fails.

Requests wait in a chain of **queues** (load balancer, accept queue, connection limit, executor, connection pool, database, client pools of other services); unbounded queues turn overload into unbounded latency with no errors until memory runs out, while bounded queues make it fast and visible. **Pools** are separate bottlenecks: their capacity is size divided by holding time, so holding a connection across slow work divides capacity, and adding threads, connections or instances beyond what the shared database handles well increases contention and lowers throughput. Across **instances**, in-process locks protect nothing; invariants belong in the database (atomic conditional updates, constraints, versions, row locks), in distributed locks with fencing, or in single-writer routing. Near capacity, latency follows a **knee** and then **collapses**; retries feed overload into **metastable failures**; and the defences are bounded queues, **backpressure**, admission control, **load shedding**, deadlines and disciplined clients. Finally, an API's style (blocking, futures, reactive) is independent of the execution model underneath; reactive streams earn their complexity for end-to-end backpressure and streaming, not for speed. Thread dumps, pool metrics and profiles distinguish CPU saturation, lock contention, pool exhaustion, slow downstreams and blocked event loops.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Request-scoped state (locals, parameters) | No interference between concurrent requests | Safety of anything the code stores in shared places |
| Stateless singleton services | Concurrent requests cannot interfere through the service | Thread safety of the components it calls |
| Thread-locals | Per-thread isolation | Per-request isolation if not cleared; propagation to other threads |
| Platform thread pool | Bounded threads; simple blocking code | Enough threads when downstream latency rises; downstream capacity |
| Event loop | Cheap waiting for many requests | Progress if a handler blocks or computes for long; more than one core per process |
| Virtual threads / goroutines | Cheap waiting for intercepted blocking I/O | Cheap waiting when pinned; more CPU, connections or downstream capacity |
| Little's Law | L = λ × W for averages in a stable system | Anything about tails; validity when the system is unstable |
| Bounded queue | Maximum wait and memory; visible rejection | Sensible client reactions to rejection |
| Unbounded queue | Nothing is rejected | Any bound on latency or memory |
| Resource pool | At most N concurrent users of the resource | Short waits (without acquisition timeouts); more capacity in the resource |
| In-process lock | Mutual exclusion among threads of one process | Anything across instances |
| Database atomic update / constraint | Correctness across all instances for that data | Coordination of work outside the database |
| Backpressure | Producers slow to consumers' pace when every layer honours it | Effect if any layer has an unbounded buffer |
| Admission control / load shedding | Bounded latency for accepted work under overload | That the right work is shed without priorities |
| Reactive streams | Subscribers never get more than they requested | Non-blocking libraries; higher speed |

---

## Interview Drill

**Q1. How many requests can a server handle concurrently?**
A strong answer must mention:
- It depends on the execution model's cost of a waiting request (thread, process, callback, virtual thread) and on memory.
- More importantly, on the downstream pools and resources: concurrency beyond what the database and dependencies support only queues.
- Little's Law gives the concurrency needed: target throughput × latency.
- And on queue limits: beyond capacity, extra concurrency is waiting, not work.
*Common weak answer:* "As many as the thread pool size." Waiting requests can exceed that on other models, and the real limit is usually downstream.

**Q2. What is Little's Law, and how do you use it to size a pool?**
A strong answer must mention:
- L = λ × W: average in-flight = arrival rate × time in system, for a stable system and a consistent boundary.
- Concurrency needed: 2,000 req/s × 0.15 s = 300.
- Pool capacity: size / holding time (10 connections / 15 ms ≈ 667 req/s; 10 / 215 ms ≈ 46 req/s when held across an HTTP call).
- Combine with the sizing formula N_cores × U × (1 + W/C) for thread pools, and size backwards from the scarcest shared resource.
*Common weak answer:* stating the formula without applying it to a pool or a latency change. The value of the law is the arithmetic: without it you cannot predict that a slower dependency multiplies the concurrency you need.

**Q3. A downstream slows from 50 ms to 2 s. What happens?**
A strong answer must mention:
- Latency per request rises ~35×, so the concurrency needed for the same throughput rises with it (60 → 2,010 threads at 1,000 req/s).
- With 200 threads, throughput caps at ~100 req/s; the excess queues; all threads block on the slow call.
- Every endpoint fails, including those not using the dependency, because they share the threads.
- Fixes: timeouts, a bulkhead limiting concurrent calls to that dependency, fallbacks, and circuit breakers (Chapter 24).
*Common weak answer:* "Only the endpoint that calls it gets slow." Shared threads make it a total outage.

**Q4. Why can adding threads make a service slower?**
A strong answer must mention:
- Extra threads just wait in the same downstream pool or lock; they add context switching and memory.
- More concurrent queries at the database increase contention (locks, process switching in PostgreSQL, cache pressure): its throughput peaks and then falls.
- Instances multiply pools (20 × 20 = 400 connections) and can exceed the database's connection limit.
- The cure is to find the bottleneck resource and protect it, not to add concurrency.
*Common weak answer:* "More threads means more parallelism." Only if the bottleneck is not shared.

**Q5. Thread-per-request versus event loop versus virtual threads: what does each do with a blocking call?**
A strong answer must mention:
- Thread-per-request: the thread blocks; others continue; waiting costs a thread.
- Event loop: a blocking call stalls every request on that loop; requires non-blocking libraries throughout.
- Virtual threads: the runtime parks the virtual thread and reuses the carrier, unless pinned.
- A worked workload: 100 in flight needs ~100–200 threads, 2+ event-loop processes for CPU, cheap virtual threads; none adds downstream capacity.
*Common weak answer:* "Event loops are always better." They fail badly on blocking or CPU-heavy handlers.

**Q6. Why is shared mutable state in a singleton service a bug, and how do you fix it?**
A strong answer must mention:
- One instance serves all concurrent requests; a field holding request data is overwritten by other requests mid-flight.
- The concrete failure: one user's invoice containing another user's orders.
- Fix: keep request data in locals and parameters; services hold only collaborators and immutable configuration.
- Genuine shared mutable state goes in thread-safe structures (concurrent maps with atomic operations, atomics, dedicated components).
*Common weak answer:* "Make the method synchronized." That serialises the service and hides the design error.

**Q7. How do you pass request context through thread pools and async code?**
A strong answer must mention:
- Thread-locals hold context per thread; they must be cleared after each request (pool threads are reused).
- They do not follow work to other threads: executors, futures' continuations, event loops, reactive pipelines.
- Options: explicit parameters, capture-and-restore wrappers (task decorators, context-propagation libraries), scoped mechanisms (`context.Context` in Go, scoped values in recent JDKs).
- In Spring, `@Async` loses the security context, MDC and transaction unless configured.
*Common weak answer:* "Use a thread-local." That is the source of both the leak and the loss.

**Q8. Why does a lock in your code not stop two instances processing the same order?**
A strong answer must mention:
- Each process has its own lock; in-process locks coordinate threads of one process only.
- The check-then-act race reappears between instances.
- Fixes: atomic conditional updates and constraints in the database, versions, row locks (Chapter 17), distributed locks with fencing (Chapter 23), single-writer routing (Chapter 25).
- Related traps: scheduled jobs on every instance, per-instance caches and rate limits.
*Common weak answer:* "Use a static lock." Static is still per process.

**Q9. What happens to latency as load approaches capacity?**
A strong answer must mention:
- Bursts form queues even below capacity; waiting grows as 1/(1 − ρ) in the simple model (5× at 80%, 10× at 90%).
- A knee around 70–85% utilisation, then a steep rise.
- Beyond capacity, unbounded queues grow without limit; goodput collapses as responses arrive after clients left.
- Hence capacity headroom and alerts on queue time and saturation, not just errors.
*Common weak answer:* "Latency rises linearly with load." It stays flat, then rises steeply, then collapses.

**Q10. What is backpressure? What are bounded queues, admission control and load shedding?**
A strong answer must mention:
- Backpressure: a busy component signals its producers to slow down, layer by layer (bounded blocking queues, TCP flow control, 429/503 with `Retry-After`, reactive `request(n)`).
- Bounded queues make overload visible and fast instead of unbounded latency.
- Admission control caps in-flight work per instance (fixed or adaptive limits).
- Load shedding rejects the least important work first, cheaply, before expensive processing; one unbounded buffer anywhere defeats backpressure.
*Common weak answer:* "Backpressure means rejecting requests." Rejection is one form; the point is propagating the slowdown to the source.

**Q11. What is a metastable failure, and how do retry storms cause one?**
A strong answer must mention:
- A state where the system stays overloaded after the trigger is gone.
- Timeouts cause retries; retries multiply load (across layers); full queues keep latency above timeouts.
- Recovery needs load cut from outside; prevention needs backoff with jitter, retry budgets, bounded queues and shedding.
- Retrying at one layer only.
*Common weak answer:* "Retries make the system more reliable." Unbounded retries are a common cause of outages.

**Q12. Is reactive programming faster? When is it worth it?**
A strong answer must mention:
- API style (blocking, futures, reactive) is independent of the execution model (threads, loops, virtual threads).
- Not faster per request; its benefits are end-to-end backpressure, streaming, high concurrency on few threads, and composition.
- Costs: every library must be non-blocking (one blocking call stalls the loop), harder debugging, explicit context propagation.
- With virtual threads, blocking code serves most request–response workloads as well.
*Common weak answer:* "Reactive is faster because it is non-blocking." Waiting is cheaper, not the work, and only if nothing blocks.

**Q13. How do you tell CPU saturation from lock contention, and both from a slow downstream?**
A strong answer must mention:
- CPU saturation: CPU near 100%, threads `RUNNABLE` in your own compute code; confirm with a CPU profile.
- Lock contention: CPU low, many threads `BLOCKED` on one monitor whose holder is doing something slow.
- Slow downstream: CPU low, threads `RUNNABLE` in socket reads to one dependency, its latency high.
- Pool exhaustion: threads parked in the pool's acquire, pool at maximum with waiters; take several dumps and compare.
*Common weak answer:* "Look at CPU: if it is high, add instances." Low CPU with a stuck server is the more common case and needs the thread dump.

**Q14. Why do unbounded queues hide overload?**
A strong answer must mention:
- They accept everything, so overload shows up as growing wait time, not errors.
- Requests wait past their clients' timeouts; the server works at full capacity on responses nobody receives.
- Memory grows until the process fails, with no error rate to alert on before.
- Bounded queues reject the excess quickly and keep accepted latency within deadlines.
*Common weak answer:* "Unbounded queues prevent dropped requests, so they are safer." They convert drops into timeouts and crashes.

**Q15. Why is holding a database connection across an HTTP call dangerous?**
A strong answer must mention:
- Pool capacity = size / holding time; holding a connection for 200 ms of unrelated waiting divides capacity many times.
- A slow HTTP dependency then exhausts the database pool and stalls every endpoint that needs the database.
- The database also sees idle-in-transaction sessions holding locks.
- Fix: do external calls outside the transaction (and design for their failure, Chapters 13 and 25).
*Common weak answer:* "It's fine as long as the pool is large enough." No pool is large enough when the external call slows.

**Q16. How should pools be sized across many instances?**
A strong answer must mention:
- Start from the shared resource: how many concurrent queries the database handles well (often a small multiple of its cores).
- Divide among instances: per-instance pool = total budget / number of instances (plus headroom for deploy overlap).
- Check the database's connection limit; use a pooler when instances are many.
- Re-check whenever the number of instances changes (autoscaling multiplies connections).
*Common weak answer:* "Set every pool to 50 to be safe." 50 × 20 instances = 1,000 connections at the database.

---

## Scenario Exercises

1. A service handles 800 requests per second at 120 ms average latency on 8 instances. Product wants 3,000 requests per second, and the latency will rise to 180 ms because of a new downstream call. How many requests will be in flight across the fleet, and per instance? What else must you check before saying "add instances"?
2. A Java service has 200 Tomcat threads and a 10-connection database pool. A new feature calls an external tax API inside the same `@Transactional` method that saves the order. Explain what will happen when the tax API's latency rises to 3 s, with numbers, and how to restructure the code.
3. An engineer proposes raising the database pool from 10 to 100 on each of 30 instances to fix "connection not available" errors. Explain what will happen at the database and what to do instead.
4. A singleton `PricingService` keeps `lastQuotedPrices` in a plain hash map that request handlers read and write. Under load, some requests throw exceptions and some customers see prices meant for others. Explain both failures and fix the design.
5. A nightly job scheduled with an in-code cron annotation emails monthly statements. After moving from 1 to 6 instances, customers receive six statements. Give two fixes and their trade-offs.
6. During a brief database slowdown, your API's error rate rises to 40%, and it stays there for 45 minutes after the database recovers. Explain the mechanism and how you would recover and prevent a repeat.

---

## Diagnosis Lab

**Lab 1: "Connection is not available" under modest load**
- *Symptom:* A checkout service starts failing with `HikariPool-1 - Connection is not available, request timed out after 30000ms.` at only 60 requests per second, far below its tested capacity. Database CPU is 15%.
- *Hypotheses:* (a) the pool is too small for the load; (b) slow queries hold connections; (c) connections are held during non-database work.
- *Evidence:* A thread dump shows ~190 request threads parked in `HikariPool.getConnection`, and 10 threads `RUNNABLE` in a socket read inside `PaymentClient.charge`, called from a `@Transactional` method. The database's activity view (`pg_stat_activity` in PostgreSQL) shows 10 sessions in state `idle in transaction`, each idle for about 2 s. The payment provider's latency rose from 150 ms to 2 s an hour ago. At 2 s held per checkout, Little's Law gives 10 / 2 s = 5 checkouts per second as the pool's maximum, which rules out (a) and (b).
- *Root cause:* (c). The checkout transaction holds its connection while waiting for the payment API, so the pool's capacity is set by the external API's latency.
- *Fix:* Restructure: save the order in a short transaction, call the payment API with no connection held, then record the result in a second short transaction, with an idempotency key and a recovery path for unknown outcomes (Chapters 13 and 25). Shorten the pool acquisition timeout so failures are fast. Alert on long `idle in transaction` sessions.
- *Prevention:* Code review rule: no network calls inside a transaction; trace spans that show non-database work inside a transaction.

**Lab 2: One slow dependency stops the whole service**
- *Symptom:* The product-page service stops answering every endpoint, including `/health` and `/api/cart`, which do not use recommendations. The recommendations service is having an incident.
- *Hypotheses:* (a) a crash; (b) thread exhaustion behind the slow dependency; (c) a database problem.
- *Evidence:* The process is alive with low CPU. A thread dump shows all 200 worker threads `RUNNABLE` in a socket read inside `RecommendationClient.fetch`. The HTTP client for recommendations has **no read timeout** configured. The health endpoint's requests sit in the executor queue, so the load balancer marks instances unhealthy one after another. Database metrics are normal.
- *Root cause:* (b): the slow dependency held every shared worker thread; with no timeout, they would wait indefinitely; everything else queued behind them.
- *Fix (now):* Disable the recommendations call by feature flag, or restart instances with it disabled. *Fix (lasting):* connect and read timeouts on every client (sized from the latency budget, Chapter 10, Section 10.10), a bulkhead that caps concurrent calls to recommendations (say 20 threads), a fallback (the page without recommendations), and a circuit breaker (Chapter 24). Serve health checks from a path that does not share the worker pool, or keep them trivially cheap.
- *Prevention:* Chaos or load tests that slow each dependency in turn and verify the rest of the service keeps working.

**Lab 3: Two slow services in one week, two different causes**
- *Symptom A:* The search service's p99 rose from 80 ms to 4 s after a release. CPU is at 100% on every core.
- *Evidence A:* Thread dumps show most worker threads `RUNNABLE` in a regular-expression engine, called from a new input-validation step. A CPU profile attributes 85% of CPU time to that regex. Some requests contain long query strings on which the regex backtracks exponentially.
- *Root cause A:* CPU saturation from a pathological regex. Fix: rewrite the expression without nested repetition (or use an engine with linear-time guarantees), cap input length before validation, and add the long inputs to tests.
- *Symptom B:* The partner-integration service's p99 rose from 100 ms to 3 s every hour, for about a minute. CPU is at 25%.
- *Evidence B:* A thread dump taken during the slow minute shows 150 threads `BLOCKED` waiting to lock the same `TokenProvider` monitor. One thread holds it and is in a socket read inside `AuthClient.refreshToken`: the partner's access token expires hourly, and `getToken()` is `synchronized` and refreshes the token over the network while holding the lock. The auth server takes about 3 s to respond.
- *Root cause B:* Lock contention: every request needs the lock, and the lock is held across slow network I/O once an hour.
- *Fix B:* Refresh the token **before** it expires, in the background; let readers use the current token without locking (an atomic reference, Chapter 5, Section 5.10); never hold a lock during I/O; if several threads find the token missing, let one refresh while the others wait on that single refresh (a "single-flight").
- *Lesson:* Same symptom (p99 up), opposite evidence: CPU high with threads computing, versus CPU low with threads blocked on one lock. The thread dump and a profile distinguish them in minutes.

---

## Solution Sketches

1. Fleet-wide in flight: L = 3,000 × 0.18 = **540** (today 800 × 0.12 = 96). On 8 instances that is about 68 per instance (from 12), more with fewer instances. Check: the thread pool or execution model per instance (68 plus headroom); every downstream pool: database connections (how long each request holds one × 3,000 req/s = concurrent connections needed, versus the database's limit and the per-instance pools), the new downstream's capacity at 3,000 req/s and its own latency under that load; CPU per request × 3,000; and the latency knee from a load test. "Add instances" only helps if the shared resources can take the multiplied load.
2. With the tax call inside the transaction, each order holds a connection for ~3 s: the pool supports 10 / 3 ≈ 3.3 orders per second. Above that, threads wait for connections; at 200 threads all blocked, every endpoint that needs the database stalls, and acquisition timeouts begin after 30 s. Restructure: compute the tax before the transaction (or reserve the order, call the tax API with no connection held, then finalise in a short transaction), with a timeout on the tax call, a fallback or a retry path, and idempotency on the final write.
3. 30 × 100 = 3,000 connections. The database refuses connections beyond its configured limit (often 100–few hundred), so many instances fail; if the limit were raised, thousands of concurrent queries would overwhelm the database's CPU and locks and lower throughput for everyone. Instead: find why connections are held long (slow queries, connections held across other work), fix that, size the total connection budget from what the database handles well, divide it among instances, add a pooler if instances are many, and use short acquisition timeouts.
4. A plain hash map is not thread-safe: concurrent writes can corrupt its internal structure, causing exceptions (or worse, in some implementations, infinite loops), and the "last quoted prices" field mixes request data across users, so one customer's quote can be read by another's request. Fix: make the service stateless (quotes are computed or loaded per request and returned, not stored in a field); if a cache of prices per product is needed, use a concurrent map with atomic operations (or a cache library), keyed by product, never by "last request".
5. (a) A lock or lease per job run, in the database (one row per job and period, acquired with an atomic update or a library such as ShedLock): simple, but every instance still schedules and contends, and a crashed holder needs a lease expiry. (b) Move scheduling out of the application to a single scheduler (a job service or the orchestrator's scheduled jobs, Chapter 26) that triggers the work once, ideally by putting a message on a queue processed by one consumer: cleaner, more infrastructure. In both cases make the work itself idempotent (one statement per customer and month, enforced by a unique constraint), so a double run cannot send twice.
6. A metastable failure: the slowdown pushed latency past clients' timeouts; clients and upstream services retried, possibly at several layers; queues filled; the service spent its capacity on requests whose callers had already left, so goodput stayed low and retries kept load above capacity after the database recovered. Recover by cutting load: shed at the edge, temporarily disable retries or reduce client concurrency, then restore gradually. Prevent with bounded queues, deadlines (drop expired requests), retries with exponential backoff, jitter and budgets at one layer only, circuit breakers, and admission control (Chapter 24).

---

**END OF CHAPTER 11**
