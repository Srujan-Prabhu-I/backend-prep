# Chapter 24: Failure and Resilience

## Where We Are

Failure has appeared in almost every chapter so far, each time as a side effect of something else. Chapter 7 showed that a timeout leaves the outcome of a request unknown (Section 7.15). Chapter 10 set timeouts at every hop and gave the two rules for nesting them (Section 10.10), and introduced health checks, draining and slow start at the load balancer (Section 10.3). Chapter 11 built the capacity model this chapter leans on: Little's Law (Section 11.3), queues and pools (Sections 11.4–11.5), and the shape of overload, from the latency knee to metastable failure (Section 11.8). Chapter 12 showed that availability multiplies along a chain of services (Section 12.11). Chapter 13 made retries safe with idempotency keys (Section 13.5). Chapter 22 failed over a database, and Chapter 23 explained why nodes cannot tell slow from dead.

This chapter is the home of **resilience**: the techniques that keep a system serving when parts of it fail. It starts with the kinds of failure a backend must expect, then builds the toolkit one mechanism at a time: timeouts and deadlines, retries with backoff and jitter, circuit breakers, bulkheads and concurrency limits, rate limiters, load shedding, graceful degradation, health checks and graceful shutdown, and redundancy with its availability arithmetic. It shows how these mechanisms fail together in a **cascading failure**, and how to test them on purpose with failure injection. Every pattern is connected to the Chapter 11 capacity model, because almost every resilience mechanism is a way of protecting a finite number of threads, connections or queue slots.

**After this chapter you can:**
- classify failures (crash, omission, timing, Byzantine), explain partial failure and gray failure, and reason about unknown outcomes;
- set connect, read and total timeouts along a call chain, propagate deadlines, and build a latency budget;
- decide when a retry is safe, and design retries with exponential backoff, jitter and a retry budget;
- compute retry amplification across layers and explain why only one layer should retry;
- explain the circuit breaker state machine, choose its thresholds, and design fallbacks that do not lie;
- isolate dependencies with bulkheads and protect a service with static and adaptive concurrency limits;
- implement fixed-window, sliding-log, sliding-window-counter, token-bucket and leaky-bucket rate limiters, locally and across instances;
- shed load by priority, apply backpressure and admission control, and throttle clients adaptively;
- trace a cascading failure from one slow dependency, and stop and recover from a metastable failure;
- degrade gracefully with feature flags and kill switches;
- implement liveness and readiness checks and a graceful shutdown that loses no requests;
- compute the availability of serial and parallel systems and translate nines into downtime;
- explain chaos engineering and plan a failure-injection experiment;
- diagnose a slow dependency taking a system down, a retry storm after an outage, and mis-nested timeouts.

**Interview questions this chapter prepares you for:**
1. What kinds of failure must a distributed system handle? What is partial failure?
2. How do you set timeouts across a call chain?
3. When is a retry safe?
4. What are exponential backoff and jitter, and why do you need both?
5. Why do retries at several layers amplify load? What is a retry budget?
6. How does a circuit breaker work?
7. What is a fallback, and when is it dangerous?
8. What is a bulkhead? What is an adaptive concurrency limit?
9. How do rate limiters work? Token bucket versus sliding window?
10. How do you rate limit across many instances?
11. What is load shedding, and how does it differ from rate limiting and backpressure?
12. What is a cascading failure, and how do you stop one? What is a metastable failure?
13. What is the difference between liveness and readiness? How does graceful shutdown work?
14. How do you compute the availability of serial and parallel components? What does "four nines" mean?
15. What is chaos engineering?

**Scope.** Here: failure models, timeouts and deadlines, retries, circuit breakers, bulkheads and concurrency limits, rate-limiting algorithms, load shedding and admission control mechanisms, cascading and metastable failures, graceful degradation, health checks and the full graceful-shutdown protocol, redundancy and availability arithmetic, failure-injection testing, resilience diagnosis, and a placement-round section on reliability measures (Section 24.14). Recalled, not re-taught: unknown outcomes (Chapter 7, Section 7.15), timeout nesting rules and deadline propagation (Chapter 10, Section 10.10), load-balancer health checks, draining and slow start (Chapter 10, Section 10.3), signals and container shutdown (Chapter 3, Section 3.6; Chapter 4, Section 4.11), Little's Law, queues and pools (Chapter 11, Sections 11.3–11.5), the overload curve and metastable failure (Chapter 11, Section 11.8), idempotency keys (Chapter 13, Section 13.5), the `429` response and rate-limit headers (Chapter 13, Section 13.13), failover (Chapter 22, Section 22.5), fencing (Chapter 23, Section 23.8). Elsewhere: retries and redelivery in message queues are Chapter 25; SLOs, error budgets, percentiles, autoscaling and incident response are Chapter 26.

---

## 24.1 Failure Models and Partial Failure

### Definitions

A **failure** is a component not doing what its specification promises: a server that stops answering, a disk that returns errors, a call that takes ten seconds instead of fifty milliseconds. A **fault** is the underlying cause (a bug, a power cut, a full disk); a failure is what other components observe. **Resilience** is a system's ability to keep providing acceptable service while some of its components fail, and to return to normal afterwards. **Fault tolerance** is the narrower property of continuing to work correctly despite a specified set of faults. **Reliability** is the broad goal behind all of them: a system that keeps doing what its users need, correctly, over time. In practice it is measured as availability (Section 24.11) and as the share of requests that succeed within their latency target (Chapter 26, Section 26.11). Formally, reliability is the probability of running without failure for a given time (Section 24.14).

Distributed-systems theory sorts failures into a few **failure models**, from mildest to worst:

- A **crash failure** (fail-stop): the component stops and does nothing further. Others may or may not be able to detect that it stopped.
- An **omission failure**: the component fails to send or receive some messages but otherwise works. A dropped packet, a full queue that discards a request, and a lost response are all omissions.
- A **timing failure** (performance failure): the component responds correctly but too late, outside the time its callers expect.
- A **Byzantine failure**: the component behaves arbitrarily, including sending wrong or contradictory answers (Chapter 23, Section 23.13). Backends inside one organisation normally assume this does not happen and defend against corruption with checksums and authentication.

Two practical terms matter more in production:

- **Partial failure** is the normal state of a distributed system: some components work, some do not, and the ones that work cannot be sure which is which. A single program on one machine either runs or crashes; a request that crosses five services can fail in any of them, at any step.
- A **gray failure** is a failure that some observers see and others do not: a node that passes its health checks but fails a fraction of real requests, a disk that is slow but not broken, a network path that drops one packet in a hundred. Gray failures are the hardest kind to detect, because the component's own view of itself is "healthy".

### Why it matters

On one machine, a function call either returns or throws. Across a network, a call can also do something worse: **nothing, for an unknown length of time**. Chapter 7 (Section 7.15) showed what this means for one request: when a response does not arrive, the caller cannot tell whether the request was lost, is still running, or completed with the response lost on the way back. Chapter 23 (Section 23.7) showed the theoretical root: over a channel that can lose messages, no finite exchange gives both sides certainty.

Resilience engineering starts by accepting this. A backend cannot prevent dependencies from failing. It can only decide, in advance, how long to wait, whether to try again, when to stop trying, what to do instead, and how to stop one failing part from consuming the resources that the healthy parts need.

### Mechanism: what a caller can observe

A caller sees only three outcomes of a remote call, and each maps to several possible realities:

```text
 what the caller sees            what may actually have happened
 ─────────────────────────────────────────────────────────────────────────────────────────────
 a success response              the operation happened (the only certain case)
 an error response               the operation did not happen, OR it partly happened
 (4xx, 5xx, an error message)    (a 500 after the database commit but before the response)
 nothing, then a timeout         the request never arrived; it arrived and is still running;
 (or a reset connection)         it completed and the response was lost; the server crashed
                                 before, during or after doing the work
```

Two conclusions follow, and the rest of the chapter depends on both. First, a **timeout is not a failure; it is an unknown**. Treating "timed out" as "did not happen" and acting on that belief (charging again, sending the email again) creates duplicates; the cure is idempotency (Chapter 13, Section 13.5). Second, **slow is worse than dead**. A dead dependency fails fast: connections are refused in a millisecond, and the caller moves on. A slow dependency holds every caller's thread, connection and memory for the full timeout. By Little's Law (Chapter 11, Section 11.3), the number of requests in flight equals the arrival rate times the time each one spends, so a dependency that slows from 50 ms to 5 s multiplies the resources held by a hundred.

### Concrete: why slow is worse than dead

A service receives 200 requests per second, and each request calls a recommendations service. The service has 100 worker threads.

```text
 recommendations healthy:   200 req/s × 0.05 s = 10 threads busy on the call     (fine)
 recommendations DEAD:      connection refused in ~1 ms
                            200 req/s × 0.001 s = 0.2 threads busy               (fine: fast errors)
 recommendations SLOW:      every call waits for the 5 s timeout
                            200 req/s × 5 s = 1,000 threads needed  > 100 available
                            → all 100 threads stuck within 0.5 s; every request queues,
                              including requests that never needed recommendations
```

The dead dependency costs a feature. The slow one takes down the whole service. This single calculation motivates timeouts (Section 24.2), circuit breakers (Section 24.4) and bulkheads (Section 24.5).

### Guarantees and non-guarantees

A failure model is an assumption, not a guarantee: a design that tolerates crashes is correct only as long as components fail by crashing. Real components also fail slowly, partially and inconsistently. No mechanism lets a caller distinguish "slow" from "dead" from "lost" with certainty; it can only choose how long to wait before acting as if one of them were true.

### Failure and diagnosis

- **A gray failure.** One instance of twenty fails 5% of requests, but its health check passes. Evidence: error rate per instance (not per service) shows one outlier; the load balancer still reports it healthy. Fix: outlier detection that ejects instances by their real error rate (passive health checks, Chapter 10, Section 10.3).
- **Duplicates after timeouts.** Customers charged twice after a slow payment provider. Evidence: two provider transactions with the same order, seconds apart, the first logged as a client timeout. Fix: idempotency keys on every retried operation with effects.
- **A slow dependency exhausts a pool.** Evidence: thread dumps (Chapter 11, Section 11.10) with most workers blocked in one client library's socket read; pool saturation metrics at 100%. Fix: Sections 24.2, 24.4 and 24.5.

### Backend connection

Design every remote call for three outcomes, not two: success, failure, and unknown. Assume dependencies will be slow before they are dead, and size the damage with Little's Law: in-flight requests equal rate times latency, so a slowdown is a resource leak with a deadline. Detect failures by what requests experience, not only by what components report about themselves.

---

## 24.2 Timeouts, Deadlines and Latency Budgets

### Definitions

Chapter 10 (Section 10.10) defined a **timeout** (how long one component waits for another), an **idle timeout**, a **deadline** (an absolute time by which the whole request must finish) and a **budget** (the time left before the deadline). It also gave the two nesting rules: request timeouts shrink inward, and idle timeouts grow inward. This section adds the kinds of timeout on a single call, how to choose their values, and how a budget is divided across a request.

On one outbound call there are several distinct timeouts:

- The **connect timeout** bounds establishing the connection: DNS resolution, the TCP handshake and the TLS handshake (Chapters 6 and 8). It should be short, because a healthy server in the same region accepts a connection in a few milliseconds.
- The **read timeout** (socket timeout, response timeout) bounds the wait for data after the request is sent. Some libraries apply it to the **first byte**, others to **any gap** between bytes, so a slow server that trickles one byte every few seconds never trips it.
- The **total timeout** (call timeout, request timeout) bounds the whole call from start to last byte, whatever happens inside it, including retries if the client library performs them.
- The **pool acquisition timeout** bounds the wait for a free connection from the client's pool (Chapter 11, Section 11.5; Chapter 18, Section 18.3).

A **latency budget** is the allocation of a request's total time limit to its steps: so much for authentication, so much for the database, so much for each downstream call.

### Why it exists

A call without a timeout can wait forever, and a thread that waits forever is a thread lost (Section 24.1). A timeout that is too long protects nothing: the resources are held for the whole wait. A timeout that is too short turns ordinary slow responses into errors and retries. And timeouts set independently by different teams on different layers produce the mis-nesting failures of Chapter 10: outer layers giving up while inner layers keep working, so that every slow moment doubles the load.

### Mechanism: choosing the values

A timeout is a prediction: "if no answer has come by now, one is unlikely to come soon enough to be useful". It should be derived from two numbers: the dependency's latency distribution, and what the caller can afford.

```text
 1. MEASURE the dependency's latency under normal load:
      p50 = 40 ms   p99 = 180 ms   p99.9 = 450 ms   (percentiles: Chapter 26)
 2. LOWER BOUND: a little above a high percentile of normal behaviour
      timeout ≥ p99.9 + margin  ≈ 500–600 ms
      (shorter would fail healthy-but-slow requests: at a 200 ms timeout, ~1% of calls fail)
 3. UPPER BOUND: what the caller's own budget allows
      caller's deadline 2 s; this call is one of three sequential steps → at most ~600 ms here
 4. CHOOSE within the bounds; if the lower bound exceeds the upper bound, the design is wrong:
      make the call asynchronous, cache its result, call in parallel, or drop it from this path
 5. CONNECT TIMEOUT separately: same region → 100–500 ms is generous
```

The **read timeout** needs care with streaming or large responses: a gap timeout of 600 ms does not bound a 30 MB download, so add a total timeout as well. And remember that the connect timeout applies **per address**: a client that tries four IP addresses from DNS, each with a 2 s connect timeout, can take 8 s before its "2 s" connect timeout fails.

### Concrete: timeout nesting across the whole path

The full chain for a checkout request, following Chapter 10's rule that request timeouts shrink inward, with each value leaving room for the layer to produce its own response:

```text
 mobile client total timeout ─────────────────────────────────────────────── 10.0 s
  └ CDN / edge origin timeout ─────────────────────────────────────────────── 9.5 s
     └ load balancer response timeout ──────────────────────────────────── 9.0 s
        └ reverse proxy upstream read timeout ──────────────────────────── 8.5 s
           └ application request deadline ─────────────────────────────── 8.0 s
              ├ auth service call (connect 0.2 s, total) ─── 0.5 s
              ├ DB pool acquire ─ 0.3 s
              ├ DB statement_timeout ──── 1.5 s
              ├ payment service call (connect 0.3 s, total) ─────── 4.0 s
              │   └ inside payment service: deadline = what remains of 4.0 s,
              │     provider call total timeout ── 3.0 s
              └ margin to build the response and log ─ 0.5 s
 sequential worst case: 0.5 + 0.3 + 1.5 + 4.0 + 0.5 = 6.8 s < 8.0 s   ✔
```

### Concrete: a mis-nested chain

Each value below looks reasonable on its own. Together they fail:

```text
 client total timeout ────────────────────────── 5 s
  └ application: calls inventory service with
    total timeout 3 s and 2 retries (the HTTP client library retries on timeout)
       attempt 1: 0.0–3.0 s   timeout
       attempt 2: 3.0–6.0 s   ← the client gave up at 5 s; this attempt works for nobody
       attempt 3: 6.0–9.0 s   ← and so does this one
    inventory service: database statement_timeout not set; slow query runs 20 s
 effect of ONE slow query:
    the client sees a timeout at 5 s and retries (another 3 attempts start)
    the inventory database runs 3 copies of the slow query per client attempt
    a 10-second slowdown at the database becomes a 6× load multiplier on it
```

The fault is not any single number. It is that the **total** time an inner layer may spend (3 s × 3 attempts = 9 s) exceeds the outer layer's limit (5 s), and that the innermost layer has no limit at all. The fix: the inventory call's retries must fit inside the remaining budget (a total of at most about 4 s including retries, or no retries at this layer), and the database needs a statement timeout below that. Diagnosis Lab 3 investigates this failure from its symptoms.

### Mechanism: deadline propagation and budget enforcement

Fixed timeouts per call waste budget when earlier steps were slow. A request that already spent 7 s of an 8 s deadline should not grant a downstream call its full 4 s. With **deadline propagation** (Chapter 10, Section 10.10), each service receives the caller's deadline and derives every timeout from what remains:

```pseudo
function handle(request):
    deadline = min(request.header("deadline") or now() + DEFAULT_LIMIT,
                   now() + MAX_LIMIT_FOR_THIS_ENDPOINT)
    ctx = Context(deadline)                       // passed explicitly (Chapter 11, Section 11.6)

    user  = call(auth,    ctx, per_call_max = 0.5 s)
    order = db.query(sql, timeout = remaining(ctx, max = 1.5 s))
    pay   = call(payment, ctx, per_call_max = 4.0 s)
    return response(...)

function call(service, ctx, per_call_max):
    budget = deadline_of(ctx) - now() - SAFETY_MARGIN    // keep time to answer the caller
    if budget <= 0:
        raise DeadlineExceeded                    // do not start work that cannot finish in time
    timeout = min(per_call_max, budget)
    return service.send(request, timeout = timeout,
                        headers = {"deadline": deadline_of(ctx)})   // propagate downstream
```

Three details make it work. Every service checks the remaining budget **before** starting expensive work, and refuses requests whose deadline has already passed (also when taking them off a queue, Chapter 11, Section 11.4). The deadline travels as an absolute time (or a remaining duration converted at each hop, which avoids comparing clocks between machines, Chapter 23, Section 23.6). And when the caller disconnects or the deadline passes, in-progress work is **cancelled**, not finished for nobody.

### Mechanism: latency budgets with fan-out

When a request calls several services **in parallel**, its latency is the **slowest** of them, so the budget applies to the maximum, and the tail of each call matters more than its median (the scatter–gather effect of Chapter 23, Section 23.5). Assume a page calls 5 services in parallel, each with p99 = 200 ms and independent:

```text
 P(all 5 respond within 200 ms) = 0.99^5 ≈ 0.95  → the page's p95 is about the services' p99
 to keep the PAGE's p99 at 200 ms, each service needs p99.8 ≤ 200 ms  (0.998^5 ≈ 0.99)
```

Two tools help. **Optional calls get short timeouts** and the page renders without them (Section 24.9). **Hedged requests** send a second copy of a slow read-only call to another replica after a short delay (for example at the call's p95), and use whichever answer arrives first; they cut tail latency for a few percent extra load, and must only be used for idempotent operations.

### Guarantees and non-guarantees

A timeout guarantees that the caller stops waiting after a bounded time. It does not guarantee that the callee stops working, that the operation failed, or that resources held by the callee are released; cancellation and statement timeouts are needed for that. Deadline propagation guarantees that downstream services know how long they have; it does not guarantee that they honour it.

### Failure and diagnosis

- **Timeouts that never fire.** A read timeout measured between bytes, against a server that trickles data. Evidence: calls lasting far longer than the configured "timeout" in traces. Fix: a total timeout.
- **Connect timeouts much longer than configured.** Multiple DNS addresses tried in sequence. Evidence: connect durations equal to a multiple of the timeout. Fix: shorter per-address timeouts, a total bound.
- **Errors on healthy-but-slow requests.** Timeouts set below the dependency's normal p99.9. Evidence: a steady background rate of timeouts with no incident; the latency histogram shows the timeout cutting through its tail. Fix: derive timeouts from measured percentiles.
- **Work continuing after the caller left.** Evidence: database queries still running for requests that returned `504` long ago; `499`s in the proxy log (Chapter 10, Section 10.8). Fix: statement timeouts, cancellation on disconnect, deadline checks.

### Backend connection

Give every outbound call a connect timeout and a total timeout, derived from the dependency's measured latency and from the caller's budget. Write the timeout chain down per endpoint, check that totals (including retries) shrink inward, propagate deadlines between services, refuse work whose deadline has passed, and cancel work whose caller has gone. When the numbers do not fit, change the design rather than the numbers.

---

## 24.3 Retries: When, How Often, and How Not to Make Things Worse

### Definitions

A **retry** is sending a request again after an attempt failed or timed out. A **retry policy** decides which failures are retried, how many times, and how long to wait between attempts.

- **Exponential backoff** makes each wait longer than the last, typically doubling: 100 ms, 200 ms, 400 ms, up to a **cap**.
- **Jitter** is randomness added to the wait, so that many clients that failed at the same moment do not retry at the same moment.
- A **retry budget** limits retries to a fraction of normal traffic (for example, retries may add at most 10% to the request rate), so that retries cannot multiply load during an outage.
- **Retry amplification** is the multiplication of load when several layers of a call chain each retry.

### Why it exists

Many failures are **transient**: a packet lost, a connection reset by a restarting instance, a brief overload, a leader election in progress. Trying again a moment later succeeds, and the user never notices. But retries are also the most common way a small problem becomes an outage. Each retry is extra load sent to a component that just signalled trouble, and if every client retries immediately, a brief overload becomes permanent (the metastable failure of Chapter 11, Section 11.8). Good retry policy is the difference between hiding a blip and amplifying it.

### Mechanism: when a retry is safe

Two separate questions must both be answered "yes":

1. **Is it safe to repeat the operation?** If the first attempt's outcome is unknown, it may have succeeded. Repeating is safe only for **idempotent** operations: reads, `PUT` and `DELETE` by design (Chapter 7, Section 7.4), and non-idempotent operations made idempotent with an **idempotency key** that the server uses to recognise and deduplicate the repeat (Chapter 13, Section 13.5). A `POST /payments` without a key must never be retried automatically after a timeout.
2. **Is the failure likely to be transient?** Retrying a request that will fail the same way wastes capacity and delays the error.

```text
 outcome                                 retry?   why
 ─────────────────────────────────────────────────────────────────────────────────────────────
 connection refused / connect timeout    yes      the request never reached the server: safe
                                                  even for non-idempotent operations
 connection reset or read timeout        only if  the request may have been processed
                                         idempotent
 408 Request Timeout                     yes*     the server did not finish reading the request
 429 Too Many Requests                   yes, but honour Retry-After; it means "slow down"
                                         later
 502 / 503 / 504                         yes*     usually transient: restart, shedding, slow upstream
                                                  (504: the upstream may still be working on it)
 500 Internal Server Error               rarely*  often a bug that will repeat; retry once at most
 400, 401, 403, 404, 409, 422            no       the request is wrong; it will fail the same way
 * only for idempotent operations or with an idempotency key
```

### Mechanism: exponential backoff with full jitter

Without backoff, a client hammers a struggling server. With backoff but without jitter, clients that failed together retry together, in synchronised waves that hit the server at exactly 100 ms, 300 ms, 700 ms after the failure. **Full jitter** picks each wait uniformly at random between zero and the exponential value, which spreads the retries evenly over time:

```pseudo
function call_with_retries(op, max_attempts = 4, base = 0.1 s, cap = 5 s, ctx):
    for attempt in 0 .. max_attempts - 1:
        result = op(timeout = remaining(ctx))
        if result is success or not retryable(result):
            return result
        if attempt == max_attempts - 1 or not retry_budget.allow():
            return result                                     // give up: return the last error
        ceiling = min(cap, base × 2^attempt)
        wait = random_uniform(0, ceiling)                     // FULL jitter
        if result.retry_after is set:
            wait = max(wait, result.retry_after)              // the server knows better
        if now() + wait >= deadline_of(ctx):
            return result                                     // no time left to try again
        sleep(wait)
```

A schedule for one client, with base 100 ms and cap 5 s:

```text
 attempt   exponential ceiling   full-jitter wait (one random draw)   attempt starts at about
 1         (first try)           -                                     0.00 s
 2         min(5, 0.1 × 2^0) = 0.1 s     0.06 s                       0.06 s + attempt 1's time
 3         min(5, 0.1 × 2^1) = 0.2 s     0.17 s
 4         min(5, 0.1 × 2^2) = 0.4 s     0.11 s
 5         min(5, 0.1 × 2^3) = 0.8 s     0.52 s
 ...       ... 1.6, 3.2, then capped at 5 s
```

And what 1,000 clients that all failed at t = 0 do on their second retry (ceiling 0.2 s):

```text
 no jitter:     all 1,000 retry at exactly t = 0.2 s           → a spike of 1,000 at one instant
 full jitter:   retries spread uniformly over 0 to 0.2 s       → about 50 per 10 ms
```

Variants exist. **Equal jitter** waits half the ceiling plus a random half, guaranteeing some minimum wait. **Decorrelated jitter** bases each wait on the previous one: `wait = min(cap, random(base, previous_wait × 3))`. An analysis by AWS engineers (Marc Brooker's 2015 article on backoff and jitter) found full and decorrelated jitter both far better than no jitter, with full jitter doing the least total work. The precise variant matters much less than having one.

### Concrete: retry amplification across layers

A request passes through three layers, each of which retries a failed call to the layer below. Let each layer make up to 4 attempts (1 try + 3 retries). When the bottom service is failing, every attempt fails, so:

```text
 mobile client      →  API gateway  →  orders service  →  inventory service (failing)
 1 user request        4 attempts       4 × 4 = 16          4 × 4 × 4 = 64 requests

 attempts reaching the failing service = (1 + r)^L  for r retries per layer and L layers
   r = 3, L = 3:  4^3 = 64          r = 2, L = 3:  3^3 = 27  (the figure Chapter 11 quoted,
                                                              with three attempts per layer)
   r = 3, L = 5:  4^5 = 1,024
```

At normal traffic of 1,000 requests per second, the failing inventory service receives 64,000 per second exactly when it is least able to cope, and the healthy layers above it burn their own threads waiting on attempts that cannot succeed. Even a single layer with 3 retries quadruples load during an outage.

The rules that contain it:

1. **Retry at one layer only**, usually the one closest to the user that can still meet the deadline (or the one that holds the idempotency key). Inner layers fail fast and report.
2. **Retry budgets**: allow retries only while they are a small fraction of traffic. One common design is a token bucket of retry permission. Each successful request adds a fraction of a token (say 0.1), each retry spends one, and retries stop when the bucket is empty. During normal operation the occasional transient failure is retried freely; during an outage, when most requests fail, retries are capped at about 10% extra load. gRPC's retry throttling and service meshes such as Envoy implement variants of this.
3. **Do not retry an overloaded service's explicit rejections immediately**: honour `Retry-After` on `429` and `503`.
4. **Stop at the deadline**: a retry that cannot finish before the caller's deadline is pure waste.

### Mechanism: hedging, a retry before failure

A **hedged request** is a retry sent before the first attempt has failed: if no answer has arrived by, say, the call's p95 latency, send a second copy to another replica and take the first response. It attacks tail latency rather than errors. It is safe only for idempotent reads, costs a few percent extra load (only slow requests are hedged), and should be disabled under overload, because then most requests are slow and hedging doubles the load. Chapter 23 (Section 23.5) used it against scatter–gather tails.

> **Framework lens:** Resilience4j's `Retry` module and Spring Retry's `@Retryable` provide configurable attempts, backoff and exception filters; check whether jitter (Resilience4j calls it a randomised wait) is enabled, since it is not always the default. Many HTTP client libraries retry on their own, silently: know whether yours does before adding another layer of retries. Service meshes can retry at the proxy, which is one more layer to count.

### Guarantees and non-guarantees

A retry policy with backoff, jitter and a budget guarantees that transient failures are often hidden, and that retries cannot multiply load beyond the budget. It does not guarantee success, does not make non-idempotent operations safe to repeat, and does not help when the failure is not transient. Jitter guarantees spread-out retries from independent clients, not from a single client that retries too often.

### Failure and diagnosis

- **A retry storm.** After a short outage, request rates to a dependency are many times normal and it cannot recover. Evidence: request rate far above organic traffic, a high ratio of retried requests (log an attempt number on every request), synchronised spikes at regular intervals (no jitter). Diagnosis Lab 2.
- **Duplicate side effects.** Retries of non-idempotent calls after timeouts. Evidence: duplicate rows or charges with close timestamps, client logs showing a timeout and a retry. Fix: idempotency keys, or no automatic retry.
- **Slow failures.** Every request takes seconds to fail during an outage because each of three layers retries three times with backoff. Evidence: error latency much higher than success latency. Fix: one retrying layer, retry budget, deadline checks.

### Backend connection

Retry only idempotent operations (or those protected by idempotency keys), only for transient errors, with exponential backoff, full jitter, a cap, a budget, and a check against the deadline. Retry at exactly one layer of a call chain, and make every other layer fail fast. Put the attempt number in request headers and logs, so that retry traffic is visible as a share of the total.

---

## 24.4 Circuit Breakers and Fallbacks

### Definitions

A **circuit breaker** is a component in the caller that watches the outcomes of calls to one dependency and, when failures exceed a threshold, stops sending calls for a while and fails them immediately instead. Named after the electrical device that cuts a circuit to prevent damage, it has three states:

- **Closed**: calls flow normally; outcomes are recorded.
- **Open**: calls are not sent; they fail immediately (or go to a fallback). The breaker stays open for a **wait duration**.
- **Half-open**: after the wait, a limited number of **probe** calls are allowed through. If they succeed, the breaker closes; if they fail, it opens again.

A **fallback** is what the caller does instead of the real call when it fails or the breaker is open: return cached data, a default value, a reduced response, or a clear error.

### Why it exists

Timeouts bound how long each call waits; they do not stop the caller from making the next call to a dependency that is obviously failing. If a dependency has failed the last 500 calls, the 501st will very likely fail too, after holding a thread for the whole timeout. A breaker converts that slow failure into a fast one: the caller's threads are freed, the user gets an answer (or an error) in microseconds, and the failing dependency stops receiving load, which gives it room to recover. It is the caller's way of applying backpressure to itself.

### Concrete: the state machine

```text
                       failure rate ≥ 50% over the last 100 calls
                       (and at least 20 calls in the window)
          ┌────────────┐ ─────────────────────────────────────────► ┌────────────┐
          │   CLOSED   │                                            │    OPEN    │
          │ calls flow │                                            │ calls fail │
          │ outcomes   │ ◄──────────────┐                           │ FAST (no   │
          │ recorded   │                │                           │ network)   │
          └────────────┘                │                           └─────┬──────┘
                 ▲                      │                                 │ wait 30 s
                 │ 5 of 5 probes        │ any probe fails                 ▼
                 │ succeed              │ (or slow)               ┌──────────────┐
                 │                      └──────────────────────── │  HALF-OPEN   │
                 └─────────────────────────────────────────────── │ allow 5 probe│
                                                                   │ calls; others│
                                                                   │ still fail   │
                                                                   │ fast         │
                                                                   └──────────────┘
```

The same breaker, in pseudocode:

```pseudo
class CircuitBreaker:
    state = CLOSED; window = SlidingWindow(size = 100)       // last 100 outcomes
    opened_at = None; probes_left = 0

    function call(op):
        if state == OPEN:
            if now() - opened_at < WAIT: return fail_fast()  // no network call at all
            state = HALF_OPEN; probes_left = 5; probe_successes = 0
        if state == HALF_OPEN:
            if probes_left == 0: return fail_fast()          // probes already in flight
            probes_left -= 1
        result = op()                                        // with its own timeout
        record(result)
        return result

    function record(result):
        failed = result.is_error or result.latency > SLOW_CALL_THRESHOLD
        if state == HALF_OPEN:
            if failed: trip()
            else:
                probe_successes += 1
                if probe_successes == 5: state = CLOSED; window.clear()
            return
        window.add(failed)
        if window.count() >= 20 and window.failure_rate() >= 0.5: trip()

    function trip(): state = OPEN; opened_at = now()
```

### Mechanism: choosing thresholds

| Setting | What it trades | Guidance |
|---|---|---|
| failure-rate threshold (e.g. 50%) | sensitivity versus false trips | high enough that normal error rates never trip it |
| minimum calls in the window (e.g. 20) | avoid tripping on two failures out of three calls | proportional to traffic: at low traffic, use a time-based window |
| window: count-based (last N calls) or time-based (last N seconds) | count-based reacts at the same number of calls regardless of rate; time-based behaves consistently in time | time-based for low or variable traffic |
| slow-call threshold | treat slow successes as failures | slightly above the dependency's normal p99; slowness is the dangerous failure |
| wait duration in open (e.g. 30 s) | recovery detection speed versus probing a dead dependency | seconds to a minute; some implementations grow it after repeated trips |
| probes in half-open (e.g. 5) | confidence versus load on a recovering dependency | a handful |

What counts as a failure matters as much as the numbers. Timeouts, connection errors and `5xx` count. `4xx` responses usually do **not**: a `404` means the dependency is working correctly, and counting it would let one client's bad requests open the breaker for everyone. Scope matters too: one breaker per dependency (or per dependency endpoint, if endpoints fail independently), per caller instance. A breaker shared across unrelated endpoints opens all of them because one failed.

### Mechanism: fallbacks, and the ones that lie

A fallback decides what the user experiences while the breaker is open. The options, from safest to most dangerous:

| Fallback | Example | Risk |
|---|---|---|
| fail fast with a clear error | `503` with `Retry-After`; "payments are temporarily unavailable" | none hidden; the user knows |
| omit an optional feature | product page without recommendations | low, if the feature truly is optional |
| serve cached or stale data, **labelled** as such | last known order status, "as of 10:02" | stale data presented as stale |
| a static default | default shipping estimate "3–5 days" | low for display, wrong if used for decisions |
| serve stale data **as if current** | a price or stock level from an hour-old cache | **wrong answers that look right** |
| a default that changes a decision | "fraud check unavailable → approve"; "permission check failed → allow"; "stock unknown → in stock" | **silent business errors or security holes** |
| queue the work for later | accept the order, charge later | correct only if the queue is durable and the user is told |

The dangerous fallbacks share one property: they return a plausible value that downstream code or a user will act on as if it were the real answer. A fraud check that "fails open" during an outage approves every fraudulent order in that window; an authorisation check that defaults to "allow" is a vulnerability that an attacker can trigger by overloading the authorisation service. Rules that follow:

- **Security and money fail closed.** If the check that guards access or spending cannot run, the action does not happen.
- **Never present a fallback as the real thing.** Mark fallback responses (a header, a field, a log line, a metric) so that callers, users and dashboards can tell.
- **Fallbacks must be cheap and independent.** A fallback that calls another service, or the same database, can fail at the same moment as the primary path, and a fallback executed only during outages is code that is never tested. Prefer data already in memory.
- **Count fallbacks as errors in monitoring.** A service that "succeeds" with fallbacks during a total dependency outage still has an outage.

### Guarantees and non-guarantees

An open breaker guarantees that the caller does not spend threads, connections or time on a dependency that is failing, and that the dependency receives only probe traffic while open. It does not guarantee that the dependency is actually down (a breaker can trip on a burst of slow calls), does not detect failures faster than its window allows, and does not make fallback data correct. Breakers in many caller instances each decide independently, so a recovering dependency receives probes from all of them.

### Failure and diagnosis

- **A breaker that never opens.** Thresholds too high, slow calls not counted, or only exceptions counted while timeouts are swallowed elsewhere. Evidence: the dependency's latency at the timeout value for minutes while the breaker reports closed; threads exhausted. Fix: count slow calls and timeouts.
- **A breaker that flaps.** Opening and closing every few seconds. Evidence: state-transition events in logs or metrics. Cause: too few minimum calls, a window too short, or probes that succeed against a partially recovered dependency. Fix: larger windows, more probes, a longer wait after repeated trips.
- **A breaker opened by client errors.** Evidence: the breaker trips while the dependency's own dashboards are healthy; the "failures" are `4xx`. Fix: count only failures that indicate the dependency's health.
- **A lying fallback.** Evidence: business anomalies (unexpected approvals, wrong prices) during a dependency incident, while error rates looked normal. Fix: fail closed for decisions; label and count fallbacks.

> **Framework lens:** Resilience4j's `CircuitBreaker` implements exactly this state machine, with count-based or time-based sliding windows, a failure-rate threshold, a slow-call threshold, a minimum number of calls, the wait duration in the open state and the number of calls permitted in half-open; Spring Cloud Circuit Breaker wraps it. Netflix's Hystrix popularised the pattern and is no longer actively developed. Service meshes offer a related mechanism, **outlier detection**, which ejects individual failing instances from a load-balancing pool rather than stopping calls to the whole dependency.

### Backend connection

Put a breaker around every remote dependency whose failure should not consume the caller, with slow calls counted as failures and only server-side errors counted at all. Decide each dependency's fallback deliberately and write it down: fail fast, omit, or serve labelled stale data, and never a fallback that silently changes a decision about money, security or stock. Alert on breaker state and fallback rate, because a breaker that is open is an outage that the user may not see but the business will.

---

## 24.5 Bulkheads and Concurrency Limits

### Definitions

- A **bulkhead** isolates resources so that a failure in one part of a system cannot consume the resources of the others. The name comes from ships, whose hulls are divided into watertight compartments so that one breach floods one compartment, not the whole ship.
- A **concurrency limit** caps the number of requests (or calls) in progress at once. Requests beyond the limit wait in a bounded queue or are rejected immediately.
- A **static limit** is a fixed number chosen in advance. An **adaptive concurrency limit** adjusts itself from observed latency, raising the limit while latency stays low and cutting it when latency rises.

### Why it exists

Section 24.1 showed one slow dependency consuming all 100 worker threads of a service, including the threads needed by requests that never touch that dependency. Timeouts shorten the damage, and breakers stop it once failures are obvious, but in the seconds before a breaker trips, a shared pool is still drained. A bulkhead makes the damage local by construction: the recommendations call can use at most its own 20 permits, so the other 80 threads keep serving checkout. It turns "one slow dependency stops everything" into "one feature degrades".

### Mechanism: what to partition

```text
 WITHOUT BULKHEADS                              WITH BULKHEADS
 ┌───────────────── 100 worker threads ──────┐   ┌──── 100 worker threads ────────────────────┐
 │ checkout │ search │ recommendations (slow)│   │ requests take a worker; each outbound call │
 │ all share one pool; slow calls hold       │   │ also needs a permit from ITS bulkhead:     │
 │ threads → every endpoint starves          │   │   payments   ≤ 30 concurrent calls          │
 └───────────────────────────────────────────┘   │   search     ≤ 25                          │
                                                 │   recommend. ≤ 15  ← full: further calls    │
                                                 │                      fail fast, page renders│
                                                 │                      without them           │
                                                 └────────────────────────────────────────────┘
```

Bulkheads can partition many resources:

| Partition | Isolates | Example |
|---|---|---|
| per-dependency concurrency limits (semaphores, Chapter 5, Section 5.6) | callers from one slow dependency | at most 15 concurrent calls to recommendations |
| per-dependency connection pools | connections | a separate HTTP client pool per downstream host (Chapter 13, Section 13.14) |
| separate thread pools per kind of work | CPU-heavy from I/O-heavy work; background from interactive | report generation in its own executor |
| per-tenant or per-client limits | customers from each other's traffic | a noisy tenant capped at 10% of capacity |
| separate instances or clusters | traffic classes, tenants | an admin and batch API on different instances from the public API |
| **cells** (shards of the whole stack) | failure blast radius | each cell serves a subset of customers with its own copy of every service |

A **semaphore** bulkhead counts permits on the caller's existing thread: cheap, but the caller's thread still blocks for the call's duration. A **thread-pool** bulkhead hands the call to a dedicated pool: the caller can stop waiting at its timeout even if the call thread stays blocked, at the cost of extra threads and a hand-off. With virtual threads or asynchronous I/O (Chapter 5, Section 5.13), a semaphore is usually enough, because waiting threads are cheap. But cheap waiting adds no capacity downstream: the scarce resource is still the dependency's capacity, and that is exactly what the permit count protects.

### Concrete: sizing a bulkhead with Little's Law

The number of permits a dependency needs in normal operation follows from Little's Law (Chapter 11, Section 11.3): concurrency = arrival rate × latency.

```text
 calls to recommendations: 300 per second at peak, latency p99 = 80 ms
 normal concurrency  = 300 × 0.08 = 24 calls in flight at p99 latency
 bulkhead size       = about 1.5–2 × that = 40 permits

 when recommendations slows to 2 s:
 demanded concurrency = 300 × 2 = 600  →  40 permits used, the other 560 calls/s fail fast
 threads held by recommendations: at most 40 (out of, say, 400), whatever the slowdown
```

The bulkhead size is a deliberate statement: "this dependency may never hold more than 40 of our threads". If 40 is not enough at normal latency, it is too small; if 600 is needed during a slowdown, that is what the limit is for.

### Mechanism: concurrency limits on the server side

A bulkhead protects a caller from its dependencies. A **server-side concurrency limit** protects a service from its callers: at most N requests in progress per instance, with excess rejected quickly (`503` with `Retry-After`, or `429` per client). This is the admission control of Chapter 11 (Section 11.8). The right N keeps the service on the flat part of its latency curve, below the knee:

```text
 measured: throughput rises with concurrency up to ~64 in-flight requests, then flattens;
           latency is flat up to ~64, then grows linearly (requests just queue inside)
 limit ≈ 64: beyond it, extra concurrency buys no throughput, only latency
 (Little's Law: at 1,600 req/s and 40 ms, L = 1,600 × 0.04 = 64)
```

### Mechanism: adaptive concurrency limits

A static limit is right only for the conditions it was measured under. When the database slows down, the service's real capacity drops, and a limit of 64 now admits more than it can serve. **Adaptive limits** estimate capacity continuously, borrowing the idea of TCP congestion control (Chapter 6, Section 6.8):

```pseudo
// AIMD (additive increase, multiplicative decrease), evaluated per completed request
limit = 20; min_latency = observed lowest latency (refreshed periodically)

on request completed (latency, dropped):
    if dropped or latency > min_latency × TOLERANCE (e.g. 2×):
        limit = max(MIN_LIMIT, limit × 0.9)        // congestion: back off multiplicatively
    else if in_flight >= limit × 0.9:               // limit actually in use
        limit = min(MAX_LIMIT, limit + 1)          // probe upward slowly
```

A **gradient** variant compares the current latency with the no-load latency and scales the limit by their ratio, so that rising queueing delay shrinks the limit smoothly (Netflix's open-source concurrency-limits library implements this family). The effect is that a service sheds load earlier when it is slower, and admits more when it is fast, without anyone retuning a number. The cost is complexity, and a risk of oscillation if the latency signal is noisy.

### Guarantees and non-guarantees

A bulkhead guarantees that one dependency (or tenant, or class of work) can hold at most its partition of a resource. It does not guarantee that the partition is large enough for normal load, or that the partitioned resources are the scarce ones: isolating threads does nothing if all dependencies share one saturated database. Concurrency limits guarantee bounded in-flight work and therefore bounded latency for accepted requests; they do not choose **which** requests to reject (Section 24.7).

### Failure and diagnosis

- **One dependency starves everything.** Evidence: thread dumps with most threads in one client; latency of unrelated endpoints rising together. Fix: per-dependency bulkheads.
- **Bulkhead rejections in normal traffic.** Evidence: a steady rate of "bulkhead full" errors without any dependency incident. Cause: limit sized below rate × latency. Fix: recompute with Little's Law at peak.
- **Partitions that share a hidden resource.** Separate pools, but every call ends at the same database or the same downstream host. Evidence: all bulkheads fill together. Fix: partition the real bottleneck, or cap the shared one.

> **Framework lens:** Resilience4j offers a `Bulkhead` (semaphore) and a `ThreadPoolBulkhead`; Envoy and other proxies provide per-upstream limits on connections, pending requests and active requests. Servlet containers cap worker threads and accept queues (Chapter 10, Section 10.5), which is a server-side concurrency limit whether or not anyone designed it as one.

### Backend connection

Give every remote dependency its own bounded concurrency (a permit count sized by Little's Law with headroom) and its own connection pool, cap each instance's in-flight requests near the knee of its latency curve, and isolate classes of work and large tenants. Prefer adaptive limits where load and dependency speed vary. The goal is that any single slow component costs a bounded, known share of capacity.

---

## 24.6 Rate Limiting Algorithms

### Definitions

**Rate limiting** caps how many requests a client (identified by user, API key, IP address, tenant or endpoint) may make in a period. Chapter 13 (Section 13.13) placed rate limits in an API and showed the response: `429 Too Many Requests` with `Retry-After`. Chapter 21 (Section 21.8) implemented the simplest one with Redis `INCR`. This section is the home of the algorithms:

- **Fixed window**: count requests per client in fixed intervals (each calendar minute); reject when the count exceeds the limit.
- **Sliding log**: store the timestamp of every request; count those in the last interval.
- **Sliding window counter**: estimate the count over the last interval from the current and previous fixed windows, weighted by overlap.
- **Token bucket**: a bucket holds up to B tokens and gains R tokens per second; each request spends a token; a request with no token is rejected. It allows bursts up to B and a long-run rate R.
- **Leaky bucket**: requests enter a queue that drains at a constant rate R; a full queue rejects. It smooths bursts into an even outflow.

### Why it exists

Rate limits protect capacity and fairness. Without them, one client's bug (a retry loop), one scraper, or one large customer's batch job can consume the capacity everyone else needs, and a credential-stuffing attack can try millions of passwords (Chapter 20, Section 20.2). Rate limits differ from the other mechanisms of this chapter in **whose** behaviour they constrain: a concurrency limit protects the server from total load, while a rate limit holds each client to an agreed share, whatever the server's current state. The two connect through Little's Law (Chapter 11, Section 11.3): a client allowed 50 requests per second against an endpoint that takes 200 ms may hold 50 × 0.2 = 10 requests in flight, so the sum of all clients' limits times the endpoint's latency must fit within the service's concurrency capacity, or the limits promise more than the service can deliver.

### Mechanism: fixed window and its boundary problem

```pseudo
function allow_fixed_window(client, limit = 100, window = 60 s):
    key = client + ":" + floor(now() / window)
    count = counter.increment(key)                  // expires after the window
    return count <= limit
```

Simple and cheap: one counter per client per window. Its flaw is the boundary. A client can send 100 requests at 10:00:59 and 100 more at 10:01:00, which is 200 requests in two seconds while never exceeding "100 per minute" in any calendar minute.

### Mechanism: sliding log

Keep every request's timestamp (for example in a sorted set), drop those older than the window, count the rest. Exact, with no boundary effect, but memory grows with the limit: a limit of 10,000 per hour stores up to 10,000 timestamps per client.

### Mechanism: sliding window counter

The sliding window counter keeps only two counters per client and estimates the sliding count by assuming the previous window's requests were spread evenly:

```pseudo
function allow_sliding_window(client, limit = 100, window = 60 s):
    t = now()
    current_start = floor(t / window) × window
    elapsed_fraction = (t - current_start) / window             // 0.0 … 1.0
    current  = counter.get(client + ":" + current_start)
    previous = counter.get(client + ":" + (current_start - window))
    estimate = previous × (1 - elapsed_fraction) + current
    if estimate >= limit:
        return false
    counter.increment(client + ":" + current_start)            // expire after 2 windows
    return true
```

```text
 limit 100/min; previous minute had 80 requests; we are 15 s into the current minute, which has 30
 estimate = 80 × (1 − 15/60) + 30 = 80 × 0.75 + 30 = 90   → allowed (90 < 100)
 at 10:01:00 after 100 requests at 10:00:59: estimate = 100 × (1 − 0) + 0 = 100 → rejected ✔
```

The estimate is approximate (it assumes even spreading) but close in practice, needs constant memory per client, and removes the boundary burst.

### Mechanism: token bucket

```pseudo
class TokenBucket:
    capacity B = 20                 // maximum burst
    refill_rate R = 5 per second    // long-run average rate
    tokens = B; last = now()

    function allow(cost = 1):
        t = now()                                       // a MONOTONIC clock (Chapter 23, Section 23.6)
        tokens = min(B, tokens + (t - last) × R)        // lazy refill: no timer needed
        last = t
        if tokens >= cost:
            tokens = tokens - cost
            return true
        return false                                    // Retry-After ≈ (cost - tokens) / R
```

```text
 B = 20, R = 5/s; a client idle for a while has 20 tokens
 t = 0.0 s   25 requests arrive at once  → 20 allowed (bucket empty), 5 rejected
 t = 1.0 s   bucket has 5 tokens         → 5 more allowed
 sustained: 5 requests per second, forever; bursts of up to 20 after idle periods
```

The token bucket is the most common general-purpose limiter because its two parameters match how people describe limits: "5 per second on average, bursts of 20". A request's `cost` can vary, so an expensive endpoint can spend 10 tokens.

### Mechanism: leaky bucket

The leaky bucket, used as a **queue**, admits requests into a bounded queue and releases them at a constant rate R; when the queue is full, new requests are rejected. Its output is perfectly smooth, which suits protecting a downstream that cannot absorb bursts (a partner API with a strict per-second limit), at the price of added queueing delay. Used as a **meter** (checking conformance without queueing), it is equivalent to a token bucket, and the **generic cell rate algorithm** (GCRA) implements it with a single stored timestamp per client. Nginx's `limit_req` module is a leaky-bucket limiter, with a `burst` queue and an option to serve queued requests without delay.

### Concrete: the algorithms compared

| Algorithm | State per client | Bursts | Accuracy | Typical use |
|---|---|---|---|---|
| Fixed window | 1 counter | up to 2× limit at window boundaries | approximate at edges | simple quotas, daily limits |
| Sliding log | one timestamp per request | none beyond the limit | exact | low limits where precision matters (login attempts) |
| Sliding window counter | 2 counters | none significant | approximate (assumes even spread) | API limits per minute or hour |
| Token bucket | tokens + timestamp | up to B, by design | exact for its model | API limits with burst allowance; per-client fairness |
| Leaky bucket (queue) | a queue | smoothed out, delayed | exact output rate | protecting a fragile downstream; traffic shaping |

### Mechanism: distributed rate limiting

With 20 instances behind a load balancer, a per-instance limiter of 100 per minute allows up to 2,000 per minute per client, and the effective limit changes with every scale-out. Three designs make the limit global:

1. **A shared counter store**: every instance checks and updates the client's state in one store (usually Redis), atomically. The check-and-update must be **one atomic operation**, or two instances can both read "99" and both allow the 100th request. A Lua script runs atomically on a Redis node (Chapter 21, Section 21.7):

```lua
-- token bucket in Redis: KEYS[1] = bucket key; ARGV = capacity, rate per second, now (ms), cost
local capacity = tonumber(ARGV[1]); local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3]);      local cost = tonumber(ARGV[4])
local state = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(state[1]) or capacity
local ts = tonumber(state[2]) or now
tokens = math.min(capacity, tokens + (now - ts) / 1000 * rate)
local allowed = 0
if tokens >= cost then tokens = tokens - cost; allowed = 1 end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(capacity / rate * 1000) * 2)
return allowed
```

   The timestamp comes from the caller, so instances' clocks must be reasonably synchronised (or the script can read the server's clock with `TIME`). The cost is a network round trip per request, and the store becomes a dependency of every request: decide in advance whether the limiter **fails open** (allow traffic when Redis is unreachable, the usual choice for fairness limits) or **fails closed** (reject, the choice for limits that protect against abuse such as login attempts).

2. **Local limiters with a share of the global limit**: each of N instances enforces limit / N locally, recomputed when N changes. No per-request round trip, but uneven load balancing makes the effective limit approximate.

3. **Local counting with periodic synchronisation**: instances count locally and exchange counts every second or so with a central service, accepting brief overshoot.

Where limits are enforced also matters. Limits at the **edge** (CDN, API gateway, Chapter 10) stop abusive traffic before it costs anything; limits **in the application** can use business identity (tenant, plan) that the edge may not know. Large systems commonly use both: coarse per-IP limits at the edge, per-tenant and per-endpoint limits inside.

### Guarantees and non-guarantees

A correctly implemented rate limiter guarantees that each identified client's accepted requests stay within its configured rate (exactly or approximately, depending on the algorithm). It does not guarantee that the server has capacity for all clients at their limits at once (the sum of limits usually exceeds capacity), does not protect against clients that rotate identities (many IPs, many keys), and does not keep the server safe when total load exceeds capacity: that needs concurrency limits and shedding.

### Failure and diagnosis

- **Boundary bursts.** Evidence: request spikes at the start of each minute from clients at their limit. Cause: fixed windows. Fix: sliding window counter or token bucket.
- **Limits that scale with instances.** Evidence: a client's accepted rate grows after a scale-out. Cause: per-instance limiters. Fix: shared store or limit / N.
- **A limiter outage becomes an API outage.** Evidence: errors from the rate-limit store, followed by all requests failing. Fix: fail open (with an alert) for fairness limits, a local fallback limiter, timeouts on the store call.
- **Legitimate users blocked behind shared IPs.** Evidence: `429`s concentrated on IPs of corporate proxies or carrier NAT (Chapter 6, Section 6.5). Fix: limit by authenticated identity where possible; higher per-IP limits.

### Backend connection

Use a token bucket (or sliding window counter) keyed by the most specific identity you have, enforce it atomically in a shared store when the limit must be global, return `429` with `Retry-After`, and decide per limit whether the limiter fails open or closed. Remember that rate limits allocate capacity between clients; they do not create it, so pair them with server-side concurrency limits.

---

## 24.7 Load Shedding, Backpressure and Admission Control in Practice

### Definitions

Chapter 11 (Section 11.8) defined **backpressure** (a busy component telling its sources to slow down), **admission control** (deciding at the entrance whether to accept work) and **load shedding** (rejecting some work deliberately and cheaply so that the rest completes), and explained why systems need them. This section gives the mechanisms:

- **Priority-based shedding** rejects the least important requests first, according to a priority assigned to each request.
- **Queue disciplines** decide which waiting request is served next and which is dropped: FIFO, **LIFO** (newest first), and **CoDel** (controlled delay, which drops requests that have waited longer than a target).
- **Client-side throttling** has clients reject some of their own requests locally when the server has been rejecting them, so that rejected traffic never crosses the network.

### Why it matters

When offered load exceeds capacity, something will be rejected: either the system chooses what, cheaply and early, or the overload chooses for it, by letting every request time out after consuming resources. Rate limiting (Section 24.6) does not solve this, because each client can be within its limit while the total exceeds capacity, and because capacity itself drops when a dependency slows. Shedding is the mechanism that keeps **goodput** (Chapter 11, Section 11.8) at the server's capacity under any load, instead of collapsing toward zero.

### Mechanism: shed early, shed cheaply

The cost of rejecting a request should be a tiny fraction of the cost of serving it, or shedding becomes its own overload. So shedding happens as early as possible:

```text
 cheapest ─────────────────────────────────────────────────────────────────────────► most expensive
 edge / LB rejects     server rejects at       rejects after reading     rejects after doing
 (no connection to     accept, from a          the request and checking  half the work (worst:
 the app)              connection or           priority and deadline     the work is wasted)
                       concurrency limit
```

A shedding decision looks at two signals: **how overloaded** the instance is (in-flight requests against the concurrency limit, queue wait time, CPU) and **how important** this request is.

```pseudo
// priorities assigned at the edge from the endpoint and caller, carried in a header
CRITICAL   = health checks, logins, checkout, payment callbacks
DEFAULT    = normal user reads and writes
SHEDDABLE  = recommendations, analytics events, prefetch, batch and retry traffic

function admit(request):
    load = in_flight / concurrency_limit                // or queue delay / target
    if load < 0.8:  return ADMIT
    if load < 0.9:  return request.priority >= DEFAULT   ? ADMIT : reject_503()
    if load < 1.0:  return request.priority == CRITICAL  ? ADMIT : reject_503()
    return reject_503()                                  // at the limit: protect what is in flight
```

Requests already past their deadline are always shed, whatever their priority: their client has gone (Chapter 10, Section 10.10).

### Mechanism: queue disciplines under overload

A FIFO queue under sustained overload serves requests in arrival order, so every request waits the full queue length, and once that exceeds clients' timeouts, the server spends all its capacity on requests whose clients have already left. Two disciplines avoid this:

- **CoDel** (controlled delay), originally designed for network routers, watches how long requests have waited. When the minimum waiting time over an interval stays above a target (for example 5 ms in routers; tens of milliseconds for servers), the queue is standing, not absorbing a burst, and requests that have waited too long are dropped.
- **Adaptive LIFO** serves the **newest** request first when the queue is building up. A request that just arrived still has a client waiting for it; one that has waited for seconds probably does not. Under normal load the queue stays FIFO for fairness.

Facebook described running both together in its request queues (Ben Maurer, "Fail at Scale", 2015): during overload, fresh requests are served quickly and stale ones are dropped, keeping goodput high.

```text
 queue under overload, client timeout 1 s, service time 10 ms, 200 requests waiting
 FIFO:          the request at the head has waited ~2 s → its client has gone → wasted work;
                every new request will also wait ~2 s → goodput ≈ 0
 adaptive LIFO: the newest request is served immediately → its client is still there;
                old requests are dropped (or time out) without being served → goodput ≈ capacity
```

### Mechanism: backpressure across a pipeline

Backpressure works only if each stage holds a **bounded** amount of work and propagates fullness upstream (Chapter 11, Section 11.8):

```text
 producer → [bounded queue 1,000] → worker pool → [bounded queue 500] → database writer
              full? producer blocks,                full? workers block,
              slows, or receives 503                which fills queue 1 → producer slows
 one UNBOUNDED queue anywhere absorbs the signal: memory grows until the process dies,
 and latency grows with it, with no error reported until the crash
```

Between services, backpressure is an explicit response: `429` or `503` with `Retry-After`, which only works if clients honour it (Section 24.3). Within message pipelines, the consumer's pace limits the producer through the broker's queue depth (Chapter 25).

### Mechanism: client-side adaptive throttling

When a server is rejecting most requests, each rejection still costs a network round trip and some server work. Google's SRE book describes **client-side throttling**: each client tracks, over a recent window, how many requests it attempted and how many the server accepted, and rejects a fraction of new requests locally, before sending them:

```text
 p(reject locally) = max(0, (requests − K × accepts) / (requests + 1))

 K = 2: while the server accepts more than half of requests, nothing is rejected locally;
 server accepting 10% of 1,000 requests: p = (1,000 − 2 × 100) / 1,001 ≈ 0.8
 → 80% of new requests never leave the client; the server sees a fraction of the load
```

As the server recovers and accepts more, local rejections fall to zero by themselves. It is backpressure that does not depend on the server having capacity left to send rejections.

### Guarantees and non-guarantees

Shedding with priorities guarantees that, under overload, the most important accepted work keeps its latency and the least important is rejected first and cheaply. Adaptive LIFO and CoDel guarantee that capacity is spent on requests whose clients are likely still waiting. None of them guarantees that clients react sensibly to rejections, that priorities are assigned correctly, or that shedding itself is cheap enough if it happens too late in the request.

### Failure and diagnosis

- **Shedding that sheds the wrong things.** Health checks or login requests rejected under load, so instances are removed and users cannot log in. Evidence: `503`s on critical endpoints during overload. Fix: priorities, with health checks and critical paths exempt.
- **Shedding too late to help.** Evidence: CPU at 100% even while most requests are rejected; profiles show rejected requests passing through authentication, parsing and database lookups. Fix: move the decision to the edge or the start of the pipeline.
- **The queue that hides overload.** Latency rises to many seconds while error rates stay at zero. Evidence: queue wait time dominating latency (Chapter 11, Section 11.4). Fix: bounded queues, CoDel-style drops, deadline checks at dequeue.

### Backend connection

Assign a priority to every request at the edge, reject by priority when in-flight work or queue delay crosses thresholds, and do it before expensive work. Bound every queue, drop requests whose deadline has passed, prefer newest-first under overload, and make clients throttle themselves when the server says no. A service that sheds well stays at full goodput under twice its capacity; a service that does not falls to zero.

---

## 24.8 Cascading Failures and Metastable Failures

### Definitions

- A **cascading failure** is a failure that spreads: one component's failure or slowdown increases the load on, or consumes the resources of, other components, which then fail in turn.
- A **metastable failure** (Chapter 11, Section 11.8) is a state in which a system stays failed after its trigger has gone, because the failure itself sustains the load that keeps it down. The term comes from physics, where a metastable state is stable until disturbed and then falls into a different stable state.
- A **retry storm** is a cascade driven by retries (Section 24.3); a **thundering herd** is many clients acting at the same moment (reconnecting, refreshing a cache entry, retrying), introduced with caches in Chapter 21 (Section 21.6).
- A **death spiral** is a cascade within a pool of identical instances: one instance fails, its load moves to the others, the next one fails under the extra load, and so on until none remain.

### Why it matters

Large outages rarely have large causes. Post-incident reviews of major outages describe the same pattern again and again: a small trigger (a slow query, a bad deploy to a fraction of instances, a brief network blip) meets a system whose protections are missing or interact badly, and the failure grows until most of the system is down. Researchers who studied such incidents (for example Bronson and colleagues, "Metastable Failures in Distributed Systems", 2021) found that the sustaining effect, usually retries or a cold cache, mattered more than the trigger. The mechanisms of Sections 24.2–24.7 exist mainly to break these loops.

### Concrete: one slow dependency, without protection

A shop has a front-end API (400 threads per instance, 10 instances), an orders service, and a fraud service that the orders service calls synchronously. Normal: 2,000 requests per second, fraud check 30 ms.

```text
 t = 0      the fraud service's database runs a bad query plan; fraud calls take 8 s
 t = 0–1 s  orders service: 300 checkouts/s × 8 s = 2,400 concurrent calls needed;
            its 200 threads are all blocked on fraud within 0.7 s
 t = 1 s    orders service stops answering anything: order history, order status, checkout
 t = 1–2 s  front-end API: calls to orders now wait for the API's 10 s timeout;
            2,000 req/s, 30% touch orders → 600/s × 10 s = 6,000 threads needed > 4,000;
            the front-end's threads fill up; product pages, search, login all stall
 t = 10 s   clients' 10 s timeouts fire; mobile apps retry 3 times; load triples
 t = 15 s   the load balancer's health checks (which call /health → orders) fail
            → instances removed → remaining instances get more load → death spiral
 t = 20 s   whole site down. Root trigger: one query plan in the fraud database.
 t = 5 min  the fraud query is fixed. The site STAYS DOWN:
            retry traffic at 4× normal, every queue full, caches expired during the outage
            → a metastable failure
```

### Concrete: the same trigger, with protections

```text
 t = 0      fraud calls slow to 8 s
 t = 0–1 s  orders service: fraud call total timeout 800 ms (its p99.9 is 400 ms);
            fraud bulkhead of 40 permits: at most 40 of 200 threads wait on fraud;
            calls beyond 40 fail fast
 t = 2 s    the fraud circuit breaker sees > 50% slow calls in its window → OPEN
            fallback decided in advance: checkouts above 200 € are placed in "pending review"
            (a durable queue), smaller ones proceed with a rule-based local check; the
            response says "order received, confirmation shortly" (never "approved")
 t = 2 s+   order history, order status and the rest of the orders API are unaffected
            (they never shared the fraud bulkhead); the front-end sees normal latency
            health checks are local (Chapter 10, Section 10.3): no instance is removed
            clients see no timeouts, so no retries are triggered
 t = 5 min  the fraud query is fixed; half-open probes succeed; the breaker closes;
            the pending-review queue is processed. Impact: delayed confirmations for
            large orders. No outage.
```

Each protection removed one step of the cascade: timeouts bounded the wait, the bulkhead bounded the threads, the breaker stopped the calls, the fallback kept the business decision honest, local health checks stopped the death spiral, and the absence of client timeouts prevented the retry storm.

### Mechanism: the feedback loops that sustain failures

| Loop | How it sustains the failure | Breaker of the loop |
|---|---|---|
| retries | failures cause retries, which add load, which causes failures | retry budgets, backoff with jitter, one retrying layer (Section 24.3) |
| timeouts plus FIFO queues | requests wait until their clients leave; served work is wasted | deadlines checked at dequeue, LIFO under overload (Section 24.7) |
| cold caches | expired entries during the outage send full traffic to the database (Chapter 21, Section 21.6) | serve stale, warm before traffic, coalescing |
| health-check removal | overloaded instances fail health checks and are removed, overloading the rest | local liveness checks, a minimum healthy count, panic mode (Section 24.10) |
| autoscaling lag and cold starts | new instances start cold and fail under the load they were added for | slow start (Chapter 10, Section 10.3), pre-warming, scaling before saturation |
| reconnect storms | all clients reconnect at once after a restart (Chapter 10, Section 10.9) | jittered reconnect delays |
| garbage collection under memory pressure | full queues fill the heap, GC pauses lengthen, more requests queue | bounded queues, shedding |

### Mechanism: recovering from a metastable failure

A system caught in a sustaining loop does not recover when the trigger is fixed, so recovery means **removing load from outside** and **restoring it gradually**:

```text
 1. CUT LOAD at the edge: shed non-critical traffic, enable kill switches (Section 24.9),
    return 503 with long Retry-After, or block traffic to a fraction of users
 2. STOP THE AMPLIFIERS: disable or reduce retries (configuration or feature flag),
    pause batch jobs and queue consumers that add load
 3. LET THE SYSTEM DRAIN: queues empty, caches refill at low load, pools recover
 4. RESTORE GRADUALLY: admit 10%, then 25%, 50%, 100% of traffic, watching latency and
    error rates at each step (the same idea as slow start)
 5. ONLY THEN investigate the trigger in depth; first, write down the loop that sustained it
```

Restarting everything at once usually makes things worse: every instance starts cold, every cache is empty, and every client reconnects in the same second.

### Guarantees and non-guarantees

The protections of this chapter guarantee that each known feedback loop has a limit: bounded retries, bounded waiting, bounded threads per dependency, bounded queues. They do not guarantee that every loop has been identified, that the limits are set correctly, or that a system recovers by itself: metastable states may require deliberate load reduction even when every component is "working".

### Failure and diagnosis

- **An incident that outlasts its trigger.** Evidence: the root cause was fixed, but traffic at the struggling component is several times organic demand; retry headers or attempt counts show most requests are retries. Diagnosis Lab 2.
- **Unrelated features failing together.** Evidence: latency of endpoints that do not call the slow dependency rises with it; thread dumps show shared pools blocked on one client. Diagnosis Lab 1.
- **Instances removed one after another.** Evidence: the balancer's healthy-target count falling step by step while traffic is constant. Cause: deep health checks or health checks starved by overload. Fix: local checks, minimum healthy percentage.

### Backend connection

For each critical request path, trace what happens when each dependency becomes slow (not dead): which resources it holds, who retries, which health checks depend on it, and which caches will expire. Break every loop you find with a bound. Keep a recovery runbook that cuts load from the outside and restores it in steps, and practise it (Section 24.12) before you need it.

---

## 24.9 Graceful Degradation, Feature Flags and Kill Switches

### Definitions

- **Graceful degradation** is continuing to provide the most important functions, possibly in reduced form, when parts of the system fail or capacity is short, instead of failing completely.
- A **feature flag** (Chapter 12, Section 12.9) is a runtime switch that turns behaviour on or off without a deploy.
- A **kill switch** is a feature flag designed in advance to turn off a non-essential, expensive or risky feature quickly during an incident.
- **Static stability** is the property that a system keeps working in its last known good configuration when a dependency it uses for control (configuration service, service discovery, feature-flag service) is unavailable.

### Why it exists

During an overload or a partial outage, a system rarely needs all its features to deliver its main value. A shop that cannot show recommendations, reviews or personalised banners still sells; a shop whose product page fails because the review service is down does not. Degradation chooses, in advance, what to give up first, so that the choice is not made by whichever dependency happens to fail.

### Mechanism: a degradation ladder

Each service defines levels, from full function to minimal function, and what triggers each:

```text
 level 0  FULL          everything on
 level 1  REDUCED       turn off: personalised recommendations, "people also bought",
                        live stock counts on listing pages (show "in stock" from cache, labelled)
 level 2  ESSENTIAL     turn off: reviews, search suggestions, analytics events, image resizing
                        on the fly (serve originals); product pages served from CDN cache
 level 3  CORE ONLY     browse cached catalogue, checkout, payment, order status; everything
                        else returns a static "temporarily unavailable" panel
 triggers: manual (on-call engineer) or automatic (CPU > 85% for 5 min, dependency breaker open)
```

Turning a level on is a configuration change, not a code change. The page still renders at each level, because the front end treats each optional section as optional: a failed or switched-off section renders as empty or as a placeholder, never as a broken page.

### Mechanism: kill switches that work when you need them

A kill switch is used during incidents, so it must work during incidents:

- **Evaluated locally.** Flag values are cached in each instance and refreshed in the background; if the flag service is down, instances keep the last known values (static stability). A flag lookup that calls a remote service on every request is a new dependency that can take the system down.
- **Safe default.** If no value is known at start-up, the default is the safe one: for a kill switch, usually "feature on" for user features but "off" for risky ones; decide per flag.
- **Tested.** Every kill switch is exercised regularly (in staging and in game days, Section 24.12). An untested switch that throws an exception when flipped turns a degradation into an outage.
- **Fast to propagate.** Seconds, not minutes; and the change is audited (who flipped what, when).

```pseudo
function render_product_page(product_id):
    page = core_section(product_id)                         // must succeed, or the request fails
    if flags.enabled("reviews") :                           // local cache, refreshed every 10 s
        page.reviews = try_with_timeout(reviews.get, 200 ms, fallback = EMPTY_SECTION)
    if flags.enabled("recommendations") and load_level() < 1:
        page.recs = try_with_timeout(recs.get, 150 ms, fallback = EMPTY_SECTION)
    return page
```

### Guarantees and non-guarantees

Graceful degradation guarantees that core functions continue when optional ones fail or are switched off, provided the core path really is independent of them. It does not guarantee that users accept the reduced experience, that the core path has no hidden dependency on an optional service (a shared library that calls it, a shared database), or that flags work during the outage unless they are evaluated locally.

### Failure and diagnosis

- **An optional feature takes down the page.** Evidence: errors on product pages that correlate exactly with a review-service incident. Fix: optional sections behind timeouts, fallbacks and flags.
- **The flag service as a single point of failure.** Evidence: a flag-service outage followed by errors or default behaviour everywhere. Fix: local caching of flag values, safe defaults.
- **A kill switch that fails when used.** Evidence: errors immediately after flipping the switch. Fix: test switches regularly.

### Backend connection

Write down, for each user-facing flow, what is core and what is optional, and make optional parts fail independently, behind timeouts, fallbacks and kill switches evaluated locally. Define degradation levels before an incident, and practise moving between them. During an overload, turning off expensive optional features is often the fastest way to restore core service.

---

## 24.10 Health Checks and Graceful Shutdown

### Definitions

Chapter 10 (Section 10.3) defined **liveness** checks ("should this process be restarted?") and **readiness** checks ("should this instance receive traffic now?"), active and passive health checks, **connection draining** and **slow start**. Chapter 4 (Section 4.11) showed how `SIGTERM` reaches a containerised server and sketched a shutdown sequence. This section assembles the full protocol:

- A **startup check** (startup probe) tells the platform that a slow-starting process has finished initialising, so that liveness checks do not kill it while it is still loading.
- **Graceful shutdown** is stopping an instance without failing any request that was accepted: no new work is routed to it, work in progress finishes, and resources are released, all before the platform kills the process.
- The **propagation delay** is the time between an instance announcing that it is leaving (failing readiness, or being deregistered) and every load balancer, proxy and client actually ceasing to send it new requests.
- **Panic mode** (fail-open health checking) is a load balancer's behaviour of sending traffic to all instances, healthy or not, when too few pass their checks, on the reasoning that a total outage is worse than some errors.

### Why it exists

Instances stop constantly: every deploy replaces all of them, autoscaling removes some every evening, and the platform moves them between machines. If each stop fails the requests in flight, a team that deploys ten times a day produces ten error spikes a day, and engineers learn to fear deploys. Health checks are also a feedback loop: wrongly designed, they remove healthy instances during exactly the overload in which they are most needed (Section 24.8).

### Mechanism: what each check should test

```text
 check       question                                   should test                      on failure
 ──────────────────────────────────────────────────────────────────────────────────────────────────
 startup     "has initialisation finished?"              config loaded, pools created,    keep waiting
                                                         migrations checked, caches warm  (up to a limit)
 liveness    "is this process stuck beyond recovery?"    the process can answer at all:   RESTART the
                                                         event loop not blocked, no       process
                                                         deadlock; nothing external
 readiness   "should I receive traffic now?"             local state: started, not        remove from
                                                         shutting down, not overloaded,   load balancing
                                                         essentials reachable FROM HERE   (do not restart)
```

The central rule is that checks reflect **this instance's** health, not the health of shared dependencies. A readiness check that queries the shared database fails on every instance at once when the database blips, and the load balancer removes all of them: the deep-health-check trap of Chapter 10 (Section 10.3). A liveness check that depends on anything external restarts healthy processes in a loop during a dependency outage, adding cold starts to the incident. Dependency failures are handled by timeouts, breakers and degradation, not by removing instances that cannot fix them.

Load balancers add their own safety: if fewer than a threshold (for example half) of targets pass, many balancers enter **panic mode** and route to all of them. Some cloud load balancers route to every target when all are unhealthy. Know what yours does, because it decides whether a health-check bug becomes a total outage.

### Mechanism: the graceful shutdown protocol

The platform sends `SIGTERM`, waits a **grace period** (for example 30 s), then sends `SIGKILL` (Chapter 4, Section 4.11). Everything must fit inside the grace period:

```text
 t = 0 s    SIGTERM received
            → set shutting_down = true; readiness now returns "not ready"
            → keep ACCEPTING and serving normally (traffic is still arriving)
 t = 0–5 s  PROPAGATION DELAY: load balancers notice the failing readiness check (check
            interval × failure threshold) or process the deregistration; proxies and clients
            with cached endpoint lists update. New requests may still arrive and are served.
 t = 5 s    STOP ACCEPTING: close the listening socket. The kernel refuses new connections
            (Chapter 4: connections already in the accept queue but not yet accepted would be
            reset, which is why the delay above comes first, so the queue is empty by now)
 t = 5 s+   DRAIN in-flight requests:
            → existing keep-alive connections: answer the current request with
              "Connection: close" (HTTP/1.1) or send GOAWAY (HTTP/2, Chapter 7, Section 7.10),
              so clients open new connections to other instances
            → idle keep-alive connections: close them
            → background work: stop taking new queue messages; finish or release
              (negatively acknowledge) the current ones (Chapter 25); stop schedulers
            → wait for in-flight requests, up to a drain timeout (e.g. 15 s)
 t ≤ 20 s   CLEAN UP: flush logs and metrics, close database and HTTP pools, release locks
            and leases, deregister from service discovery if not done
 t ≤ 25 s   EXIT 0
 t = 30 s   (grace period ends: SIGKILL would arrive here; it must never be needed)

 budget: propagation delay + longest request + cleanup < grace period
         5 s + 15 s + 5 s = 25 s < 30 s   ✔
```

The order matters. A server that stops accepting at `t = 0` refuses requests that load balancers are still sending, producing `502`s on every deploy. A server that never sends `Connection: close` or `GOAWAY` leaves clients reusing connections to an instance that is about to disappear, producing resets. Requests that can legitimately run longer than the drain timeout (large uploads, long reports, open WebSockets) need their own design: asynchronous jobs (Chapter 13, Section 13.10; Chapter 25), or client reconnection with resumption (Chapter 10, Section 10.9).

### Mechanism: graceful startup

The reverse also matters. An instance should report ready only after it can serve at full speed: configuration validated (Chapter 12, Section 12.9), connection pools filled to their minimum, critical caches loaded, and, on JIT-compiled runtimes, a short warm-up with synthetic requests (Chapter 2, Section 2.5). The load balancer's slow start (Chapter 10, Section 10.3) then ramps its traffic up over a minute or two. An instance that announces readiness too early fails or slows its first thousand requests, and during a scale-out under load, that can feed the death spiral of Section 24.8.

### Guarantees and non-guarantees

A correct shutdown protocol guarantees that accepted requests complete (within the drain timeout) and that no new requests reach a closing instance after the propagation delay. It does not guarantee completion of work longer than the drain window, does not cover `SIGKILL` (out-of-memory kills, node failures), and depends on the platform's grace period and the load balancer's propagation time, which must be measured. Local health checks guarantee that one dependency's failure does not remove every instance; they do not detect gray failures, which need passive checks on real traffic (Section 24.1).

### Failure and diagnosis

- **Errors on every deploy.** `502`s or connection resets clustered at deploy times. Evidence: error timestamps against instance termination events; access logs showing requests arriving after the instance logged "shutting down", or no shutdown log at all. Diagnosis Lab 4.
- **Instances killed after exactly the grace period** (exit code 137). Evidence: termination time equals the grace period. Cause: `SIGTERM` not reaching the server, or a drain that waits forever. Fix: Chapter 4's signal handling, a bounded drain timeout.
- **Restart loops during a dependency outage.** Evidence: restart counts rising on all instances while a database is down. Cause: liveness checks that test the database. Fix: liveness tests only the process.
- **All instances removed at once.** Evidence: zero healthy targets while each instance's own logs show it was serving. Cause: deep readiness checks. Fix: local readiness; panic-mode thresholds.

### Backend connection

Give every service three cheap checks with clearly separated meanings, keep all of them local, and implement shutdown as a sequence: fail readiness, wait for propagation, stop accepting, close connections politely, drain with a bound, clean up, exit, all inside the grace period. Report ready only when warm. Then test it: deploy under load and count errors; the right number is zero.

---

## 24.11 Redundancy, Failover and Availability Arithmetic

### Definitions

- **Availability** is the fraction of time (or of requests) for which a system provides acceptable service. It is commonly written in **nines**: 99.9% is "three nines", 99.99% "four nines".
- **Redundancy** is having more components than strictly needed, so that the loss of some does not stop the service. **Active–active** redundancy serves traffic on all copies; **active–passive** keeps a standby that takes over on failure (**failover**, Chapter 22, Section 22.5).
- **N+1** (or **N+2**) redundancy provisions one (or two) more components than the load requires, so that one (or two) can fail while the rest carry the full load.
- A **failure domain** is a set of components that can fail together: a machine, a rack, a data centre (an availability **zone**), a region, a software version, a configuration change.

### Why it exists

Every component fails eventually: disks, machines, network links, data centres, and software. Availability cannot be achieved by making one component perfect; it is achieved by combining imperfect components so that the combination fails less often than any of them. The arithmetic of combining them explains both why redundancy works and why long chains of dependencies erode it (Chapter 12, Section 12.11).

### Concrete: nines and downtime

| Availability | Downtime per year | Per 30-day month | Per week |
|---|---|---|---|
| 99% (two nines) | 3.65 days | 7.2 hours | 1.68 hours |
| 99.5% | 1.83 days | 3.6 hours | 50 minutes |
| 99.9% (three nines) | 8.76 hours | 43.2 minutes | 10.1 minutes |
| 99.95% | 4.38 hours | 21.6 minutes | 5.0 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.3 minutes | 1.0 minute |
| 99.999% (five nines) | 5.3 minutes | 26 seconds | 6 seconds |

Each additional nine allows ten times less downtime. Four nines leave about four minutes a month: less than the time it takes a human to be paged, log in and understand an alert. Availability targets above three nines therefore require **automatic** detection and recovery.

### Mechanism: serial and parallel composition

If a request needs **all** of several independent components (a **serial** chain), it succeeds only when all are up:

```text
 A_serial = A1 × A2 × … × An

 load balancer 99.99% → app 99.9% → database 99.95% → payment provider 99.9%
 A = 0.9999 × 0.999 × 0.9995 × 0.999 ≈ 0.9974   → 99.74%: worse than any single component
                                                  (about 23 hours of downtime a year)
```

If a request needs only **one** of several independent redundant components (**parallel**), it fails only when all of them are down:

```text
 A_parallel = 1 − (1 − A1) × (1 − A2) × … × (1 − An)

 two app instances at 99.5% each:     1 − 0.005²  = 0.999975     → 99.9975%
 three app instances at 99.5% each:   1 − 0.005³  ≈ 0.99999988   → about 99.99999%
```

A realistic system combines both:

```text
 load balancer (managed, 99.99%)
   → app tier: 3 instances in parallel, each 99.5%                        ≈ 0.99999988
   → database primary + synchronous standby with automatic failover:
       effective 99.95% (failover takes ~30 s, a few times a year, plus maintenance)
   → payment provider (external, 99.9%, no redundancy)
 A ≈ 0.9999 × 0.99999988 × 0.9995 × 0.999 ≈ 0.9984   → 99.84%, about 14 hours a year

 the weakest SERIAL component dominates: the payment provider alone allows 8.8 hours.
 adding a second payment provider in parallel (99.9% each): 1 − 0.001² = 0.999999
 → A ≈ 0.9994, about 5 hours a year
```

### Mechanism: why real redundancy is weaker than the arithmetic

The parallel formula assumes **independent** failures and **instant**, **perfect** failover. Reality violates all three:

- **Correlated failures.** Replicas in the same zone lose power together; instances running the same version crash on the same input; a bad configuration push reaches every instance in a minute; a certificate expires everywhere at once. Redundancy only multiplies availability across **different failure domains**: spread replicas over zones, roll out changes gradually (Chapter 26), and diversify where it is cheap.
- **Capacity.** Parallel instances help only if the survivors can carry the load. Three instances at 70% utilisation become two at 105% when one fails, and the remaining two collapse (the death spiral of Section 24.8). N+1 means each instance runs at most at (N−1)/N of its safe capacity: with 3 instances and a safe ceiling of 75% utilisation (below the knee of Chapter 11, Section 11.8), each must run at most at 50% in normal operation. Across zones, the rule is the same: to survive losing one of three zones, each zone must be able to absorb half of another's traffic, and ideally without needing to start new capacity during the failure (**static stability**: pre-provisioned capacity does not depend on an autoscaler or control plane that may itself be failing).
- **Failover is a failure mode.** Detection takes time, promotion takes time, clients take time to find the new primary, and failover logic is rarely exercised. A standby that has never taken traffic is a hope, not redundancy.

### Concrete: reliability of a dependency chain with retries

Retries are a form of redundancy in **time**. If a dependency fails each attempt independently with probability p (transient failures only), the probability that all of k attempts fail is p^k:

```text
 p = 1% transient failure per attempt
 1 attempt: 99% success     2 attempts: 99.99%     3 attempts: 99.9999%
 but only for INDEPENDENT, transient failures; during an outage p ≈ 1 and retries add load,
 not availability (Section 24.3)
```

### Guarantees and non-guarantees

Redundancy across independent failure domains guarantees that the loss of one domain leaves the others serving, given enough spare capacity. The arithmetic guarantees nothing about correlated failures, failover duration or capacity, which in practice decide most outages. Availability numbers from providers describe their component, measured their way; your availability is the composition of all components on the request path, including your own code and deployments.

### Failure and diagnosis

- **Redundant instances failing together.** Evidence: all replicas down at the same timestamp; the same exception on all of them, or all in one zone. Cause: correlated failure (bad deploy, shared dependency, zone loss). Fix: zone spreading, staged rollouts, removing shared single points.
- **Overload after losing one instance.** Evidence: utilisation of the survivors jumping above the knee when one instance fails; latency collapse minutes later. Fix: N+1 capacity, measured at peak.
- **Failover that does not work.** Evidence: a standby that fails to promote, or clients that keep connecting to the old primary (Chapter 22, Section 22.5). Fix: regular failover drills.

### Backend connection

Compute the availability of each critical request path from its serial chain, find the weakest component, and decide where redundancy (or degradation that removes the component from the path) pays most. Spread replicas over zones, provision N+1 capacity at peak, prefer designs that keep working without a control plane during a failure, and exercise failover regularly. A nine is earned by automation and testing, not by adding boxes to a diagram.

---

## 24.12 Chaos Engineering and Failure Injection

### Definitions

- **Failure injection** is deliberately causing failures (killing processes, adding latency, dropping packets, failing dependencies) in a controlled way, to verify that a system handles them as designed.
- **Chaos engineering** is the discipline of running such experiments, increasingly in production, to build confidence in a system's behaviour under turbulent conditions. Netflix popularised it with **Chaos Monkey** (2011), a tool that randomly terminates production instances so that every service must tolerate instance loss.
- A **game day** is a planned exercise in which a team injects a failure (or simulates an incident) and practises detection, response and recovery.
- The **blast radius** of an experiment is the share of users or traffic it can affect.

### Why it exists

Resilience mechanisms are code that runs only during failures, which means it is the least tested code in the system. A breaker whose fallback throws an exception, a failover script that references a renamed host, a kill switch that nobody has flipped since it was written, a timeout set in the wrong unit: all of these are found either by injection, at a time of your choosing, or by a real outage, at a time of its choosing.

### Mechanism: running an experiment

```text
 1. STEADY STATE   define normal in terms of user-visible metrics:
                   "checkout success ≥ 99.5%, p99 latency ≤ 800 ms"
 2. HYPOTHESIS     "if the recommendations service adds 3 s of latency, product pages still
                   load within 1 s and checkout is unaffected"
 3. INJECT         add 3 s of latency to recommendations calls from ONE instance of the
                   product service (blast radius: ~5% of product-page traffic)
 4. OBSERVE        compare the experiment group with the control group on the steady-state
                   metrics; watch bulkhead usage, breaker state, fallback rate
 5. ABORT RULES    stop immediately if checkout success < 99% or errors exceed a threshold;
                   the abort mechanism is tested first
 6. LEARN          the hypothesis held, or a weakness was found (the bulkhead was shared
                   with reviews; the breaker never opened because slow calls were not counted)
                   → fix → repeat with a larger blast radius
```

Typical injections, from simple to severe:

| Injection | How (examples) | What it verifies |
|---|---|---|
| kill an instance | terminate a process or container | redundancy, draining, restart, no state in instances |
| add latency to a dependency | service-mesh fault injection; a proxy such as Toxiproxy; `tc qdisc add dev eth0 root netem delay 300ms` on Linux | timeouts, bulkheads, breakers (slowness is the dangerous case) |
| return errors from a dependency | mesh fault injection (abort with 503 for X% of calls) | retries, budgets, fallbacks |
| exhaust a resource | fill a disk, consume memory or CPU, exhaust a connection pool | alerts, limits, shedding |
| lose a zone | drain all traffic from one zone | N+1 capacity, static stability, failover |
| skew a clock | shift one machine's clock | token validation, leases (Chapter 23, Section 23.6) |
| fail the control plane | make the flag service or service discovery unavailable | static stability, local caching |

### Guarantees and non-guarantees

A successful experiment guarantees that the system handled that injected failure, at that scale, on that day. It does not prove the system handles other failures, combinations, or the same failure at larger scale, and production experiments carry real risk to real users, which is why they start small, have tested abort rules, and avoid peak times until confidence is high.

### Failure and diagnosis

- **An experiment that causes an outage.** Evidence: steady-state metrics breach and the abort did not trigger fast enough. Fix: smaller blast radius, automatic abort on metric thresholds, run first in staging.
- **Experiments that never find anything.** Injections too mild, or measured on the wrong metrics (instance health instead of user success). Fix: hypotheses about user-visible behaviour; inject slowness, not only crashes.

### Backend connection

Treat every resilience mechanism in this chapter as untested until a failure has been injected against it. Start with game days in staging (kill an instance, slow a dependency, flip each kill switch), automate the ones that pass, and move carefully to small production experiments with abort rules. Inject latency more often than crashes, because slow dependencies cause the cascades.

---

## 24.13 Diagnosis Toolkit: Seeing Resilience Mechanisms at Work

### Definition

The **resilience diagnosis toolkit** is the set of per-dependency metrics, mechanism states, logs and traces that show how a service is spending its capacity during trouble: what it waits for, what it retries, what it rejects and why.

### Why it matters

During an incident, the questions are always the same: which dependency is slow, who is waiting for it, which protections fired, and is the load organic or self-inflicted? Without per-dependency and per-mechanism signals, every incident starts with guesswork and thread dumps.

### Concrete: the signals

| Question | Signal |
|---|---|
| Which dependency is slow or failing? | per-dependency latency percentiles, error rate and **timeout rate**, from the caller's side |
| What are threads waiting on? | thread dumps grouped by stack (Chapter 11, Section 11.10); in-flight calls per dependency |
| Are protections firing? | circuit-breaker state transitions and open time; bulkhead permits in use and rejections; fallback rate per dependency |
| Is load organic or self-inflicted? | retry ratio (requests with attempt number > 1 divided by all); requests per second compared with the same hour last week |
| Who is being rejected, and why? | rejections by reason (rate limit, concurrency limit, shed by priority, deadline expired) and by client or priority |
| Are queues hiding overload? | queue wait time versus service time per stage (Chapter 11, Section 11.4) |
| Do timeouts nest? | traces with per-span durations: inner spans ending after outer spans did; `499`s in proxy logs |
| Are deploys clean? | errors and resets in the minutes around instance terminations |

Two cheap practices make most of these possible: **propagate and log an attempt number** (and a request ID, Chapter 10, Section 10.11) on every outbound call, and **emit a metric for every decision a resilience mechanism makes** (tripped, rejected, shed, fell back), labelled by dependency and reason.

### Mechanism: symptom to first check

| Symptom | First check | Typical cause |
|---|---|---|
| unrelated endpoints slow together | thread dump; per-dependency in-flight | one slow dependency in a shared pool (no bulkhead) |
| traffic far above normal after an incident | retry ratio | retry storm; no budget or jitter |
| errors at regular intervals | retry timing | synchronised retries without jitter |
| `499`s plus late `200`s for the same request IDs | timeout chain in traces | inner timeouts longer than outer ones |
| healthy dependency, open breaker | what the breaker counts | client errors counted as failures; thresholds too low |
| errors on every deploy | shutdown logs and termination events | no propagation delay; no drain |
| `429`s rising for one tenant only | rate-limit rejections by client | a client's retry loop or batch job |

### Backend connection

Instrument the resilience mechanisms themselves, not only the endpoints: every breaker, bulkhead, limiter and shedder should report its state and its decisions, labelled by dependency and reason, and every request should carry an attempt number. The dashboards built from them answer "what is this service waiting for, and what is it refusing?" in the first minute of an incident.

---

## 24.14 Placement-Round Topics: Reliability Measures and Redundancy Schemes

> **Placement-round focus:** This section is revision material for reliability questions in university courses (software engineering, distributed systems, computer organisation) and placement written tests. It is not part of the main flow. What carries into backend work: availability depends on how often things fail **and** how quickly they recover, which is why fast, automatic recovery (failover, restarts, rollbacks) buys more nines than rarer failures.

### MTTF, MTTR, MTBF and availability

- **MTTF** (mean time to failure): the average time a component runs before it fails.
- **MTTR** (mean time to repair or recover): the average time to restore service after a failure.
- **MTBF** (mean time between failures): the average time from one failure to the next, MTBF = MTTF + MTTR. (Many textbooks use MTBF and MTTF interchangeably when repair time is small.)

```text
 steady-state availability  A = MTTF / (MTTF + MTTR)      (often written MTBF / (MTBF + MTTR))

 a server fails on average every 2,000 hours and takes 4 hours to restore:
 A = 2,000 / (2,000 + 4) ≈ 0.998  → 99.8%
 cut MTTR to 6 minutes (0.1 h) with automatic failover:
 A = 2,000 / 2,000.1 ≈ 0.99995   → 99.995%: the same hardware, 40× less downtime
```

### Failure rate and reliability

The **failure rate** λ is the number of failures per unit time. With a constant failure rate, **reliability** (the probability of running without failure for a time t) is R(t) = e^(−λt), and MTTF = 1/λ. A component with λ = 0.0001 per hour (MTTF 10,000 h) has R(1,000 h) = e^(−0.1) ≈ 0.905.

The constant rate holds only in the middle of a component's life. The **bathtub curve** describes three phases: a high, falling failure rate early (**infant mortality**: manufacturing defects, burn-in failures), a low constant rate during useful life (random failures), and a rising rate at the end (**wear-out**).

### Series, parallel and N-modular redundancy

Reliability combines like availability (Section 24.11): series R = R1 × R2 × …; parallel R = 1 − (1 − R1)(1 − R2)….

**Triple modular redundancy** (TMR) runs three identical modules and a **voter** that outputs the majority answer, so one faulty module (even one producing wrong output, not just stopping) is masked. The system works if at least two of three modules work:

```text
 R_TMR = R³ + 3R²(1 − R) = 3R² − 2R³        (ignoring the voter's own reliability)

 R = 0.9   → R_TMR = 3(0.81) − 2(0.729) = 2.43 − 1.458 = 0.972   (better than one module)
 R = 0.5   → R_TMR = 0.5                                          (no gain)
 R = 0.4   → R_TMR = 0.352                                        (worse than one module)
```

TMR helps only when each module is more likely to work than not. Generalised, **N-modular redundancy** with N = 2f + 1 modules masks f faulty ones by majority vote, the same majority idea as consensus (Chapter 23, Section 23.7). A **k-out-of-n** system works when at least k of n components work; TMR is 2-out-of-3. **N-version programming** runs independently written implementations of the same specification and votes, attempting to mask software bugs that identical copies would share.

### Drill for this section

**C1. A server has MTTF 1,000 hours and MTTR 10 hours. What is its availability, and what are two ways to raise it to 99.9%?**
A strong answer must mention:
- A = 1,000 / 1,010 ≈ 99.0%.
- For 99.9% with the same MTTF, MTTR must fall to about 1 hour (1,000 / 1,001); or MTTF must rise to about 10,000 hours with the same MTTR.
- Redundancy: two such servers in parallel give about 1 − 0.0099² ≈ 99.99% if failures are independent.
*Common weak answer:* improving only MTTF; reducing recovery time is usually cheaper and more effective.

**C2. Compute the reliability of a TMR system whose modules each have reliability 0.95, and explain when TMR is worse than a single module.**
A strong answer must mention:
- R_TMR = 3(0.95)² − 2(0.95)³ = 2.7075 − 1.71475 ≈ 0.993.
- The system works if at least two of three modules work; the voter masks one wrong output.
- When R < 0.5, TMR is less reliable than one module; the voter is itself a single point of failure.
*Common weak answer:* "three copies always triple reliability", ignoring the majority requirement.

---

## Chapter Summary

Distributed systems fail **partially**: some components work while others crash, drop messages, respond too late or, rarely, lie. A remote call has three outcomes, success, failure and **unknown**, and a **slow** dependency is worse than a dead one, because by Little's Law it holds the caller's threads and connections for the whole wait. Resilience is a set of bounds on that damage.

**Timeouts** bound waiting: a short connect timeout, a total timeout derived from the dependency's measured latency and the caller's budget, nested so that totals (including retries) shrink inward, with **deadlines propagated** between services and work cancelled when the caller has gone. **Retries** hide transient failures only when the operation is idempotent (or carries an idempotency key) and the error is transient; they use **exponential backoff with jitter**, a cap, a **retry budget**, and a check against the deadline, and only one layer retries, because (1 + r)^L attempts reach a failing service when L layers each retry r times. **Circuit breakers** stop calling a dependency whose failure rate (counting slow calls) crosses a threshold, fail fast while **open**, and test recovery with **half-open** probes; their **fallbacks** must fail closed for money and security and must never present stale or default data as real.

**Bulkheads** give each dependency, tenant or class of work a bounded share of threads and connections, sized by Little's Law, so that one failure costs a known fraction of capacity; **concurrency limits**, static or **adaptive**, keep each instance below the knee of its latency curve. **Rate limiters** (fixed window, sliding log, sliding window counter, **token bucket**, leaky bucket) hold each client to its share, enforced atomically in a shared store when the limit is global. **Load shedding** by priority, **adaptive LIFO** and **CoDel** queues, backpressure through bounded queues, and **client-side throttling** keep goodput at capacity under overload.

These mechanisms exist to break the feedback loops of **cascading** and **metastable failures**: retries, queues full of abandoned requests, cold caches, health checks that remove overloaded instances, cold starts and reconnect storms. Recovery from a metastable state means cutting load from outside and restoring it gradually. **Graceful degradation** with locally evaluated **kill switches** keeps core functions when optional ones fail. **Health checks** stay local (liveness restarts, readiness routes), and **graceful shutdown** fails readiness, waits out the propagation delay, stops accepting, closes connections politely, drains, cleans up and exits inside the grace period. **Availability** composes: serial chains multiply availabilities, parallel redundancy multiplies unavailabilities, and both are weakened by correlated failures, missing spare capacity and untested failover. **Chaos engineering** tests all of it on purpose, before an outage does.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Timeout | The caller stops waiting after a bound | That the operation failed; that the callee stops working |
| Deadline propagation | Downstream services know the remaining budget | That they honour it |
| Retry with backoff, jitter, budget | Transient failures often hidden; retry load bounded | Safety for non-idempotent operations; success during outages |
| Idempotency key | A repeated request is recognised and not applied twice | Fast or successful completion |
| Circuit breaker | No calls (beyond probes) to a dependency judged failing | That the dependency is really down; correct fallback data |
| Fallback | A response when the dependency fails | That the response is true; safety for decisions |
| Bulkhead | One dependency holds at most its partition | Enough capacity in the partition; isolation of a shared hidden bottleneck |
| Concurrency limit (static or adaptive) | Bounded in-flight work and latency for accepted requests | Which requests are rejected |
| Rate limiter | Each client's accepted rate within its limit | Server capacity for all clients at once; identity that cannot be rotated |
| Priority load shedding | Critical work survives overload first | Correct priorities; cheapness if shedding happens late |
| Adaptive LIFO / CoDel | Capacity spent on requests whose clients still wait | Fairness under sustained overload |
| Kill switch (local evaluation) | Optional features can be turned off during incidents | That the core path is truly independent of them |
| Readiness / liveness checks (local) | Traffic only to started, non-draining instances; stuck processes restarted | Detection of gray failures |
| Graceful shutdown | Accepted requests complete within the drain window | Completion of longer work; survival of SIGKILL |
| Parallel redundancy | Survival of one failure domain's loss, with spare capacity | Independence of failures; instant failover |
| Failure injection | The tested failure was handled, at that scale | Handling of untested failures or combinations |

---

## Interview Drill

**Q1. What kinds of failure must a distributed system handle? What is partial failure? How do reliability, resilience and fault tolerance differ?**
A strong answer must mention:
- Reliability: keeps doing what users need over time (measured as availability and successful requests); resilience: keeps acceptable service while parts fail and recovers; fault tolerance: works correctly despite a specified set of faults.
- Crash, omission, timing (slow) failures; Byzantine named and usually excluded within one organisation.
- Partial failure: some components fail while others work, and callers cannot tell which; gray failures visible to some observers only.
- A remote call has three outcomes: success, failure, unknown; a timeout is an unknown, not a failure.
- Slow is worse than dead: by Little's Law, slowness holds resources (rate × latency).
*Common weak answer:* "Servers either work or crash." Most damaging failures are slow or partial.

**Q2. How do you set timeouts across a call chain?**
A strong answer must mention:
- Connect timeout short; total timeout from the dependency's measured high percentile (above p99.9) and the caller's budget.
- Request timeouts shrink inward, counting retries in each layer's total; idle timeouts grow inward (Chapter 10).
- Deadline propagation: each hop uses min(own limit, remaining budget − margin); refuse expired work; cancel on disconnect.
- Statement timeouts in the database; parallel calls budgeted on the slowest.
*Common weak answer:* "Set every timeout to 30 seconds." Outer layers give up while inner layers keep working, doubling load.

**Q3. When is a retry safe?**
A strong answer must mention:
- When repeating the operation is safe: idempotent methods, or an idempotency key the server deduplicates.
- When the failure is likely transient: connection refused, 503, 429 (after Retry-After), timeouts for idempotent calls; not 4xx client errors.
- Connection refused is safe even for non-idempotent calls, because the request never arrived.
- Within the deadline and the retry budget.
*Common weak answer:* "Retry on any error three times." It duplicates side effects and amplifies outages.

**Q4. What are exponential backoff and jitter, and why do you need both?**
A strong answer must mention:
- Backoff: waits grow exponentially (base × 2^attempt) up to a cap, so failing servers get breathing room.
- Jitter: randomise waits (full jitter: uniform between 0 and the exponential value) so clients that failed together do not retry together.
- Without jitter, synchronised retry waves hit the server at the same instants.
- Honour Retry-After; stop at the deadline.
*Common weak answer:* "Backoff is enough." Backoff alone keeps clients synchronised.

**Q5. Why do retries at several layers amplify load? What is a retry budget?**
A strong answer must mention:
- With L layers each making 1 + r attempts, (1 + r)^L requests reach a failing service: 4^3 = 64 for 3 retries at 3 layers.
- Amplification happens exactly when the service is failing, sustaining the outage.
- Retry at one layer only; inner layers fail fast.
- Retry budget: retries capped at a fraction (for example 10%) of successful traffic, via a token bucket of retry permission.
*Common weak answer:* "Each layer retries, so the request is more reliable." It multiplies load on the failing component.

**Q6. How does a circuit breaker work?**
A strong answer must mention:
- Closed: calls flow, outcomes recorded in a sliding window; trips to open when the failure rate (including slow calls) exceeds a threshold with a minimum call count.
- Open: calls fail fast without network calls, for a wait duration.
- Half-open: a few probe calls; success closes, failure reopens.
- Count only dependency-health failures (timeouts, 5xx, connection errors), not 4xx; one breaker per dependency.
*Common weak answer:* "It retries until the service comes back." A breaker stops calls; it is the opposite of retrying.

**Q7. What is a fallback, and when is it dangerous?**
A strong answer must mention:
- An alternative response when a call fails or the breaker is open: error, omitted feature, labelled stale data, default, deferred work.
- Dangerous when it returns plausible but wrong data acted on as real: stale prices, "fraud check passed", "permission allowed".
- Money and security fail closed; fallbacks labelled and counted as errors; cheap and independent of the failing path.
*Common weak answer:* "Always return a default so the user sees no error." Silent wrong answers are worse than visible errors.

**Q8. What is a bulkhead? What is an adaptive concurrency limit?**
A strong answer must mention:
- Bulkhead: partitioned resources (permits, pools, instances, cells) so one dependency or tenant cannot consume all of them.
- Sizing by Little's Law: normal concurrency = rate × latency, plus headroom; during a slowdown, excess calls fail fast.
- Server-side concurrency limit near the knee of the latency curve.
- Adaptive limits adjust from latency (AIMD or gradient), like TCP congestion control.
*Common weak answer:* "A bulkhead is a separate microservice." It is resource isolation, which can exist inside one process.

**Q9. How do rate limiters work? Token bucket versus sliding window?**
A strong answer must mention:
- Fixed window (boundary bursts up to 2×), sliding log (exact, memory per request), sliding window counter (two counters, weighted estimate), token bucket (rate R, burst B, lazy refill), leaky bucket (smooth output queue).
- Token bucket allows controlled bursts; sliding window counter enforces a rolling count with constant memory.
- Response: 429 with Retry-After.
*Common weak answer:* "Count requests per minute." It misses boundary bursts and burst control.

**Q10. How do you rate limit across many instances?**
A strong answer must mention:
- Per-instance limits multiply with instance count.
- A shared store with an atomic check-and-update (a Redis Lua script implementing the token bucket).
- Alternatives: limit / N locally; local counting with periodic synchronisation; trade-offs in accuracy and latency.
- Fail open or closed when the store is unavailable, decided per limit; edge plus application limits.
*Common weak answer:* "GET the counter, check it, then INCR." Non-atomic check-then-act lets concurrent requests exceed the limit.

**Q11. What is load shedding, and how does it differ from rate limiting and backpressure?**
A strong answer must mention:
- Shedding rejects work cheaply when the server is overloaded, by priority, to keep goodput at capacity.
- Rate limiting holds each client to a share regardless of server state; backpressure signals sources to slow down.
- Shed early (edge, accept), drop expired requests, adaptive LIFO or CoDel queues, client-side throttling.
*Common weak answer:* "They are the same thing." Each answers a different question: whose share, how much total, and who should slow down.

**Q12. What is a cascading failure, and how do you stop one? What is a metastable failure?**
A strong answer must mention:
- A failure that spreads by consuming resources or adding load elsewhere: slow dependency → blocked threads → upstream timeouts → retries → health checks remove instances.
- Protections that break each step: timeouts, bulkheads, breakers, honest fallbacks, local health checks, retry budgets.
- Metastable: a sustaining loop (retries, cold caches, full queues) keeps the system down after the trigger is gone.
- Recovery: cut load from outside, stop amplifiers, drain, restore gradually.
*Common weak answer:* "Fix the root cause and it recovers." Metastable failures persist after the root cause is fixed.

**Q13. What is the difference between liveness and readiness? How does graceful shutdown work?**
A strong answer must mention:
- Liveness: restart a stuck process, tests only the process; readiness: route traffic or not, tests local state; startup checks for slow starts; all local, never shared dependencies.
- Shutdown: on SIGTERM fail readiness, wait for propagation, stop accepting, Connection: close / GOAWAY, drain with a bound, stop consumers, clean up, exit before SIGKILL.
- Budget: propagation delay + longest request + cleanup < grace period.
*Common weak answer:* "Catch SIGTERM and exit." Exiting immediately fails in-flight and still-routed requests.

**Q14. How do you compute the availability of serial and parallel components? What does "four nines" mean?**
A strong answer must mention:
- Serial: product of availabilities (worse than the weakest); parallel: 1 − product of unavailabilities.
- 99.99% ≈ 52.6 minutes a year, 4.3 minutes a month; each nine is 10× less downtime.
- Caveats: correlated failures, failover time, spare capacity (N+1); availability = MTTF / (MTTF + MTTR).
*Common weak answer:* "Two servers double the availability." Parallel redundancy multiplies unavailability, and only if failures are independent and capacity suffices.

**Q15. What is chaos engineering?**
A strong answer must mention:
- Deliberate, controlled failure injection to verify resilience: define steady state, hypothesise, inject, observe, abort rules, learn.
- Small blast radius first; staging before production; latency injection as well as crashes.
- Finds broken fallbacks, untested failover, missing timeouts before real outages do.
*Common weak answer:* "Randomly breaking production." It is a hypothesis-driven experiment with a controlled blast radius and abort conditions.

---

## Scenario Exercises

1. A product page calls six services in parallel. Each has p99 latency 150 ms. The page's p99 is 900 ms, and product managers want 300 ms. Explain the gap, and propose changes to timeouts, fallbacks and calls.
2. A payment API call times out at 10 s. The client library retries it twice. Customers report occasional double charges. Explain exactly how, and design the fix on both the client and the server.
3. After a 2-minute database outage, the orders service stays at 100% error rate for 25 minutes, although the database recovered immediately. Request rates to the orders service are five times normal. Give three hypotheses and the evidence that separates them, then a recovery plan.
4. A team adds a circuit breaker around its inventory service with a fallback of "in stock" so that "customers are never blocked". Discuss the consequences, and propose a better design.
5. Your public API enforces "1,000 requests per hour per API key" with a fixed-window counter in each of 12 instances, each allowing 1,000 / 12 ≈ 83 requests per hour per key. One customer, whose SDK sends everything over a single long-lived HTTP/2 connection through a layer-4 load balancer, is limited after about 85 requests an hour. Another customer sends about 1,900 requests in the two minutes around each hour boundary without being limited. Explain both and redesign.
6. Every deploy of a service produces about 40 `502` errors at the load balancer, although the service handles `SIGTERM` and waits for in-flight requests. Explain what is probably missing and give the corrected sequence with timings.
7. Your system runs in two availability zones, each with enough capacity for 60% of peak traffic. Leadership asks whether it can survive a zone failure. Answer with arithmetic, and propose two alternative designs with their costs.

---

## Diagnosis Lab

**Lab 1: One slow dependency takes the whole system down**
- *Symptom:* At 11:40, every endpoint of the customer API (login, search, profile, orders) starts timing out. CPU on the API instances is 15%. The database and all internal services report healthy.
- *Hypotheses:* (a) a network problem between the load balancer and the API; (b) a slow dependency exhausting the API's workers; (c) a database issue that its own health checks miss.
- *Evidence:* Thread dumps from two API instances show 196 of 200 worker threads blocked in a socket read inside the HTTP client, all calling the third-party address-validation service. Traces show those calls taking 30 s (the client library's default read timeout, never configured), up from 120 ms; only the profile-update endpoint calls that service, but all endpoints share the 200-thread worker pool. The provider's status page reports degraded performance from 11:38. Little's Law: 40 profile updates per second × 30 s = 1,200 threads needed.
- *Root cause:* (b): a slow, optional third-party dependency without a configured timeout, sharing one worker pool with everything else.
- *Fix (now):* Flip the kill switch for address validation (profile updates accept addresses unvalidated and queue them for validation later), then restart nothing: the blocked threads free themselves as calls finish. *Fix (lasting):* explicit connect (300 ms) and total (800 ms) timeouts on the client; a bulkhead of 20 permits for the provider; a circuit breaker counting slow calls; validation moved to an asynchronous job for non-blocking flows.
- *Prevention:* A rule (enforced in code review and a linter) that every outbound client has explicit timeouts; latency injection against each third-party dependency in a game day.

**Lab 2: A retry storm after an outage**
- *Symptom:* The search service was down for 90 seconds during a node failure. It was restarted and is healthy in isolation, but its error rate stays at 70% for 20 minutes; request rate is 4.5× the usual for that hour.
- *Hypotheses:* (a) organic traffic growth coinciding with the incident; (b) retries amplifying load; (c) cold caches making each request more expensive.
- *Evidence:* Last week's traffic at the same hour matches the normal rate, ruling out (a). Request logs carry an attempt header: 78% of requests are retries. Three layers retry: the mobile app (3 retries, fixed 1 s delay, no jitter), the API gateway (2 retries on 503) and the search client in the front-end service (2 retries). Request-rate graphs show sharp spikes every second, aligned with the app's fixed delay. The search cache hit ratio dropped from 90% to 35% after the restart, adding (c) as an amplifier: each request now costs more.
- *Root cause:* (b), amplified by (c): with (1 + 3)(1 + 2)(1 + 2) = 36 possible attempts per user action, the failure generated more load than the restarted service could absorb with a cold cache, a metastable state.
- *Fix (now):* Turn off retries at the gateway and in the front-end via configuration; shed the "search suggestions" endpoint (a sheddable priority); return `503` with `Retry-After: 30` to the app; once the cache warms and error rates fall, restore suggestions.
- *Fix (lasting):* One retrying layer (the app, with full jitter, exponential backoff and honouring `Retry-After`); retry budgets in the gateway as a safety net; request coalescing and warm-up for the search cache (Chapter 21, Section 21.6).
- *Prevention:* Track retry ratio as a key metric with an alert; a game day that restarts search under peak load.

**Lab 3: Mis-nested timeouts**
- *Symptom:* During a period of slow database queries, the reporting API's database CPU jumps from 40% to 100%, although user traffic is flat. Clients see `504` errors; the application logs show many requests completing with `200` after their clients disconnected.
- *Hypotheses:* (a) a new expensive report; (b) duplicate work from retries of requests still running; (c) a database plan regression alone.
- *Evidence:* The proxy logs show `499` (client closed) at exactly 10 s for many requests, followed in the application log by `200` responses for the same request IDs at 25–40 s. The timeout chain, read from configuration: client 10 s; load balancer 60 s; proxy upstream read 60 s; application no request deadline; HTTP client to the reporting database layer 30 s with 2 retries; database `statement_timeout` not set. `pg_stat_activity` shows the same report query running in five copies with identical parameters, started 10 s apart: each client retry (after its 10 s timeout) started a new copy while the old ones continued. A plan regression (c) made one query slow at the start, but the copies made it a database-wide problem.
- *Root cause:* (b), triggered by (c): inner layers allowed far longer than the client's 10 s (90 s with retries at the application, unbounded at the database), so every client retry added a full copy of the work.
- *Fix:* Set the application request deadline to 8 s, propagate it, and derive the database call's timeout from the remaining budget with no retries at that layer; `statement_timeout` of 7 s for the reporting role; cancel queries when the client disconnects; proxy and balancer timeouts at 9 s; long reports moved to an asynchronous job (`202 Accepted` with a status resource, Chapter 13, Section 13.10).
- *Prevention:* A written timeout chain per endpoint checked in review; an alert on `499` rate; traces that flag spans finishing after their parent.

**Lab 4: Errors on every deploy**
- *Symptom:* Each rolling deploy of the cart service produces 30–50 `502` errors and some connection resets, concentrated in the first two seconds after each instance begins terminating.
- *Hypotheses:* (a) the service does not handle `SIGTERM`; (b) the service stops accepting before load balancers stop routing to it; (c) clients reuse keep-alive connections to terminating instances.
- *Evidence:* The service logs "shutdown started" on `SIGTERM` and waits for in-flight requests, so (a) is ruled out. But it closes its listening socket immediately at `SIGTERM`. The load balancer's health check runs every 5 s with a threshold of 2 failures, so it keeps routing new requests to the instance for up to 10 s after readiness fails; those requests are refused (`502`). Resets come from keep-alive connections: the server closes idle connections without signalling, while the balancer has a request in flight on one of them.
- *Root cause:* (b) and (c): no propagation delay, and connections closed without `Connection: close`.
- *Fix:* On `SIGTERM`, fail readiness and keep serving for 12 s (longer than the balancer's detection time), then stop accepting; respond with `Connection: close` during the drain; drain up to 10 s; cleanup; exit by 28 s within a 30 s grace period. Alternatively, have the deploy system deregister the instance from the balancer before sending `SIGTERM`.
- *Prevention:* A deploy-under-load test in the pipeline that fails if any request errors.

---

## Solution Sketches

1. With six parallel calls each at p99 150 ms, the probability that all six are fast is 0.99^6 ≈ 0.94, so about 6% of pages wait for at least one slow call; the page's p99 is set by the services' p99.8 or worse, plus any serial steps. Changes: classify calls into required and optional; give optional calls short timeouts (200 ms) with empty-section fallbacks and render without them; hedge idempotent required calls at their p95; move slow optional content to asynchronous loading after the page renders; cache stable data; and measure each service's p99.9 against the budget.
2. The 10 s timeout fired while the payment was still processing (or after it succeeded with the response lost); each retry created a new charge because the request carried no idempotency key. Fix: the client generates an idempotency key per payment attempt and sends it with every retry; the server stores the key with the result and returns the stored result for repeats (Chapter 13, Section 13.5); the client retries only within its deadline, with backoff and jitter, and treats a timeout as unknown, checking status before declaring failure; the timeout itself should be derived from the provider's latency.
3. Hypotheses: a retry storm (evidence: retry ratio, attempt headers, synchronised spikes); a cold cache making each request more expensive (evidence: hit ratio, per-request database queries); health-check removal or cold restarts shrinking capacity (evidence: healthy-target count, instance restarts). Recovery: shed non-critical endpoints and return `503` with `Retry-After` at the edge; disable retries at inner layers; let caches warm at reduced load; restore traffic in steps; afterwards, one retrying layer with jitter and budgets, cache warm-up and coalescing, local health checks.
4. "In stock" as a fallback accepts orders for items that may be sold out, producing oversells, cancellations and refunds exactly during incidents; the breaker hides the outage from monitoring. Better: for product pages, show "availability unknown" or a labelled cached value; at checkout, the stock decision is made by the database inside the order transaction (Chapter 17, Section 17.13), not by the inventory service's cached view, or orders are accepted as "pending stock confirmation" in a durable queue with a clear customer message; count fallbacks as errors.
5. The first customer's single connection is pinned to one instance, because a layer-4 balancer distributes connections, not requests (Chapter 10, Section 10.3), so only that instance's share of about 83 applies. The second exploits fixed windows: up to 1,000 requests (spread over all instances) in the last minute of one hour and 1,000 more in the first minute of the next, all within each window's limit. Redesign: one global limit per API key in a shared store, checked and updated by an atomic script, using a token bucket (1,000 per hour with a modest burst) or a sliding window counter, so that neither connection placement nor window boundaries matter; return `429` with `Retry-After` and rate-limit headers; keep a local fallback limiter for when the store is unreachable.
6. The service probably closes its listener at `SIGTERM`, while the load balancer keeps routing for its detection interval, and it may close keep-alive connections without warning. Corrected: at t = 0 fail readiness and keep accepting; wait longer than the balancer's detection or deregistration time (for example 10–15 s); then close the listener; send `Connection: close` or `GOAWAY` on remaining connections; drain in-flight requests up to a bound; clean up; exit within the grace period (extended if needed); or deregister before `SIGTERM` via a pre-stop step.
7. No: if one zone fails, the other must serve 100% of peak with capacity for 60%, so it runs at about 167% of its capacity and collapses (and autoscaling during a zone failure may itself be impaired). Alternatives: provision each of the two zones for 100% of peak (cost: about 1.7× today's capacity, idle most of the time); or move to three zones with each sized for 50% of peak (total 150% of peak, surviving any single zone loss with 100% of peak capacity remaining); in either case, add degradation levels that cut load during a zone loss, and test with zone-evacuation game days.

---

**END OF CHAPTER 24**
