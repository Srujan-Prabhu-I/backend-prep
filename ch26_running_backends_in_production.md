# Chapter 26: Running Backends in Production: Deployment, Scaling, and Observability

## Where We Are

The previous chapters built a backend from the machine up: processes, containers and cgroups (Chapters 3–4), networks and protocols (Chapters 6–9), servers and their concurrency (Chapters 10–11), application structure and APIs (Chapters 12–13), data (Chapters 14–23), and the resilience patterns that keep it serving when parts fail (Chapter 24) and the asynchronous machinery behind it (Chapter 25). Along the way, many chapters pointed here for the same few things: how a service is deployed and scaled, how an orchestrator uses the cgroup files of Chapter 4, why averages lie about latency (Chapter 2, Section 2.8), and the traces, metrics and profiles that so many diagnosis sections assumed.

This chapter is the home of **running backends in production**. It follows code from a merged change to a running, scaled service: build artifacts and images, scaling and capacity planning, the concepts behind container orchestration, and deployment strategies that let you release ten times a day and roll back in seconds. Then it turns to **observability**, the ability to answer questions about a running system from the evidence it emits: logs, metrics and traces, percentiles and tail latency, the golden signals with the RED and USE methods, service-level objectives and error budgets, profiling in production, and incident response. It ends by walking from an alert to a root cause with all three signals.

**After this chapter you can:**
- trace a change from source to running service: artifact, image, configuration, environment, and the checks of a delivery pipeline;
- choose between vertical and horizontal scaling, explain why statelessness enables scaling, configure autoscaling with an understanding of its lag, and plan capacity with Little's Law;
- explain an orchestrator's desired state, reconciliation, scheduling, service discovery and probes, and map resource requests and limits to Chapter 4's cgroups;
- compare rolling, blue–green and canary deployments with their rollback paths, and deploy schema changes without downtime;
- deliver configuration and secrets to production safely, and roll configuration out like code;
- write structured logs with correlation IDs and without personal data, choose metric types and avoid cardinality explosions, and read a distributed trace;
- explain why averages hide tail latency, compute percentiles from histograms, and quantify fan-out amplification;
- apply the golden signals, RED and USE methods;
- define SLIs, SLOs and SLAs, compute and spend an error budget, and alert on symptoms with burn rates;
- read a CPU flame graph and use heap profiles and thread dumps in production;
- run an incident: triage, mitigate before diagnosing, communicate, and write a blameless postmortem.

**Interview questions this chapter prepares you for:**
1. What happens between merging a change and it running in production?
2. Horizontal versus vertical scaling? Why must services be stateless to scale?
3. How does autoscaling work, and why does it lag?
4. What does a container orchestrator do? What are desired state, scheduling and service discovery?
5. What is the difference between liveness and readiness probes, and what goes wrong with a bad liveness probe?
6. What are resource requests and limits, and how do they relate to cgroups and CPU throttling?
7. Rolling versus blue–green versus canary deployments?
8. How do you deploy a database schema change without downtime?
9. Logs versus metrics versus traces? How does distributed tracing work?
10. What is metric cardinality, and why can a label take down your monitoring?
11. Why use p99 and not averages? How are percentiles computed from histograms?
12. What are the golden signals, and the RED and USE methods?
13. What are SLIs, SLOs, SLAs and error budgets? How should you alert?
14. How do you read a CPU flame graph?
15. How do you debug a production latency spike, and how should an incident be run?

**Scope.** Here: build and delivery, scaling and capacity planning, orchestration concepts, deployment strategies, configuration and secrets in production, logs, metrics, traces, percentiles, the golden signals and RED/USE, SLOs and alerting, production profiling, incident response, diagnosis from alert to root cause, and a placement-round section on intrusion detection and malware (Section 26.14). Recalled, not re-taught: cgroups, namespaces and container images (Chapter 4, Section 4.10), running a server in a container (Chapter 4, Section 4.11), signals (Chapter 3, Section 3.6), thread dumps and profiling tools (Chapter 5, Section 5.14; Chapter 11, Section 11.10), Little's Law (Chapter 11, Section 11.3), health checks and draining (Chapter 10, Section 10.3), request IDs through proxies (Chapter 10, Section 10.11), configuration and feature flags (Chapter 12, Section 12.9), zero-downtime migrations (Chapter 18, Section 18.15), secrets management (Chapter 20, Section 20.11), scaling definitions (Chapter 23, Section 23.1), resilience mechanisms, graceful shutdown and availability arithmetic (Chapter 24). No product tutorials: products appear only as examples and in lens boxes.

---

## 26.1 From Code to Running Service

### Definitions

- A **build artifact** is the output of building source code: a compiled binary, a JAR file, a bundle of interpreted code with its dependencies. A **container image** (Chapter 4, Section 4.10) packages the artifact with its runtime and operating-system libraries as layered, read-only filesystems.
- An **environment** is one place the software runs, with its own configuration and data: development, staging, production.
- A **release** is an artifact plus the configuration for one environment. A **deployment** is the act of making a release run.
- **Continuous integration** (CI) builds and tests every change automatically as it is merged. **Continuous delivery** (CD) keeps every passing change deployable, and **continuous deployment** deploys it automatically. A **delivery pipeline** is the sequence of automated steps from commit to production.
- The **software supply chain** is everything that goes into an artifact: source, dependencies, build tools, base images. A **software bill of materials** (SBOM) lists an artifact's components and versions.
- **Operations** is the work of deploying, running, observing and repairing software in production. **DevOps** is the practice of having the same teams build and operate their software, with automation (pipelines, infrastructure declared as code, monitoring) replacing hand-offs between separate development and operations groups. **Site reliability engineering** (SRE), a discipline that originated at Google, applies software engineering to operations: reliability targets expressed as SLOs and error budgets (Section 26.11), automation instead of manual work, and blameless incident review (Section 26.13).

### Why it exists

A backend is not "the code". It is a specific build of the code, on a specific runtime, with specific dependencies, configuration and secrets, running on specific machines. Most production failures that are not bugs come from differences between these: a library version that differs between laptop and server, a configuration value that differs between staging and production, an image rebuilt with a different base. A disciplined path from commit to production removes those differences, makes every deploy repeatable and every rollback possible.

### Mechanism: the delivery pipeline

```text
 commit ─► CI: build ─► unit and integration tests (Chapter 12, Section 12.12) ─► static checks
            │         (lint, security analysis, dependency vulnerability scan, secret scan)
            ▼
        ARTIFACT + IMAGE, built ONCE, tagged with the commit (app:3f9a1c2), never rebuilt per env
            │  image scan for known-vulnerable packages; SBOM generated; image SIGNED
            ▼
        registry (immutable tags)
            │
            ├─► staging: SAME image + staging config ─► automated end-to-end and contract tests
            │
            └─► production: SAME image + production config ─► progressive rollout (Section 26.4)
                 with automatic rollback on SLO regression (Section 26.11)
```

The central rule is **build once, deploy many** (Chapter 12, Section 12.9): the image tested in staging is byte-for-byte the image that runs in production; only configuration differs. Rebuilding per environment means production runs something nobody tested.

### Mechanism: the supply chain

Chapter 20's map of the OWASP Top 10 (Section 20.12) sent two risks here: **vulnerable and outdated components** and **software and data integrity failures**. Each pipeline step addresses part of them:

| Risk | Pipeline control |
|---|---|
| a dependency with a known vulnerability | dependency scanning against vulnerability databases on every build, and continuously for deployed artifacts (new vulnerabilities are found in old versions); automated update proposals; a patching policy with deadlines by severity |
| a base image full of unneeded, vulnerable packages | minimal base images (Chapter 4, Section 4.10); image scanning; regular rebuilds on patched bases |
| a tampered artifact or image | signing images in the pipeline and verifying signatures before deployment; recording **provenance** (which source and pipeline produced the artifact) |
| a malicious or typo-squatted package | pinned versions with lock files and checksums; a vetted internal mirror; review of new dependencies |
| secrets in source or images | secret scanning in CI; secrets delivered at run time, never built in (Chapter 20, Section 20.11) |
| "what is running, exactly?" | an SBOM per image, so that a newly disclosed vulnerability can be matched to affected services in minutes |

### Concrete: what one running instance consists of

```text
 instance orders-7c9f (production, zone b)
   image        registry/orders@sha256:9d1e…   (tag 3f9a1c2; signed; SBOM attached)
     ├ base layer: minimal OS libraries        (scanned 2026-10-23: 0 critical)
     ├ runtime layer: language runtime 21.0.4
     └ app layer: orders service build 3f9a1c2
   configuration  from the environment's config store, version 412 (Section 26.5)
   secrets        fetched at start-up from the secret manager with workload identity
   resources      CPU request 1, limit 2; memory limit 2 GiB → cgroup cpu.weight, cpu.max,
                  memory.max (Section 26.3)
   identity       service account "orders" (Chapter 20, Section 20.10)
```

Every line is something that can differ between two instances that "run the same service", and every line should be recorded, so that during an incident "what changed?" has an answer.

### Guarantees and non-guarantees

An immutable, signed image built once guarantees that every environment runs the same bits and that what runs is what the pipeline produced. It does not guarantee that configuration, data or traffic are the same across environments, that the dependencies are free of undiscovered vulnerabilities, or that tests covered the behaviour that fails.

### Failure and diagnosis

- **"It worked in staging."** Evidence: the production image digest differs from staging's (rebuilt), or the configuration diff between environments shows the cause. Fix: build once; diff configuration as part of the release.
- **A vulnerable library found in production months later.** Evidence: an advisory; the SBOMs show which images contain the version. Fix: rebuild with the patched version, roll out, and add continuous scanning of deployed images.
- **An unknown image running.** Evidence: the running digest has no pipeline record or signature. Fix: admission checks that refuse unsigned images; investigate as a security incident.

### Backend connection

Make the path from commit to production automated, repeatable and recorded: one immutable image per commit, scanned and signed, promoted unchanged through environments with only configuration varying. Record what runs where (digest, configuration version, deploy time), because "what changed in the last hour?" is the first question of almost every incident.

---

## 26.2 Scaling and Capacity Planning

### Definitions

Chapter 23 (Section 23.1) defined **scalability**, **vertical scaling** (a bigger machine) and **horizontal scaling** (more machines). This section applies them to the application tier:

- A **stateless service** keeps no client-specific or request-spanning state in an instance's memory or local disk between requests; all such state lives in shared stores (databases, caches, session stores). Any instance can serve any request (Chapter 11, Section 11.6; Chapter 19, Section 19.4).
- **Autoscaling** adjusts the number of instances (horizontal) or their size (vertical) automatically from a measured signal.
- **Capacity planning** is estimating the resources needed to serve expected load at the required latency, with headroom for failures and growth.

### Why it exists

Load changes: by hour, by weekday, by season, by marketing campaign. Provisioning for the peak all the time wastes money; provisioning for the average fails at the peak. Horizontal scaling of stateless instances is the standard answer for the application tier, because it has no ceiling, survives instance loss, and can follow load automatically. The data tier scales by the different and harder means of Chapters 22–23.

### Mechanism: why statelessness enables horizontal scaling

```text
 STATEFUL instance                              STATELESS instances
 session, cart, uploaded file on local disk     state in Redis / DB / object storage
 → requests must return to the same instance     → any request to any instance
 → adding an instance helps only NEW users       → adding an instance helps immediately
 → removing an instance loses its users' state   → removing one loses nothing (after draining)
 → sticky routing, uneven load (Chapter 10)      → least-load routing, even spread
```

A service is ready to scale horizontally when an instance can be killed at any moment without user-visible loss, beyond requests in flight (which graceful shutdown handles, Chapter 24, Section 24.10). Typical leftovers that break this: in-memory sessions, local file uploads, in-process caches treated as authoritative, scheduled jobs that run on every instance (Chapter 25, Section 25.2), and per-instance rate limits (Chapter 24, Section 24.6).

### Mechanism: autoscaling signals and lag

An autoscaler compares a signal with a target and adjusts the instance count, typically as:

```text
 desired instances = ceil( current instances × current signal / target signal )
 e.g. 10 instances at 85% average CPU, target 60%  →  ceil(10 × 85 / 60) = 15
```

| Signal | Good for | Weakness |
|---|---|---|
| CPU utilisation | CPU-bound services | misleading for I/O-bound services (threads wait, CPU stays low while latency explodes) |
| requests per second per instance | uniform request costs | ignores cost differences between endpoints |
| in-flight requests (concurrency) per instance | most services; tracks Little's Law directly | needs the right target from load tests |
| queue depth or consumer lag (Chapter 25, Section 25.11) | workers and consumers | parallelism capped by partition count |
| latency | the user-visible symptom | lags behind the cause; scaling does not fix a slow dependency |
| schedule | predictable daily or event peaks | wrong when reality differs |

Autoscaling is not instant. The delay from a load increase to new capacity serving traffic is the sum of several parts:

```text
 metric collection and averaging window      ~15–60 s   (signals are smoothed to avoid flapping)
 autoscaler evaluation interval              ~15–30 s
 provisioning: new machine (if needed)       ~1–3 min   (much less if spare nodes exist)
 image pull and process start                ~10–60 s
 warm-up: pools, caches, JIT (Chapter 2)     ~10–120 s  (readiness reported only after it)
 load-balancer slow start (Chapter 10)       ~30–120 s
 TOTAL                                       often 2–6 minutes
```

So autoscaling handles gradual changes (a daily curve) well and sudden spikes badly. Spikes are absorbed by **headroom** (run below the knee, Chapter 11, Section 11.8), by **scheduled scaling** before known events, and by the shedding and limits of Chapter 24 for the minutes before new capacity arrives. Scale-in should be slower than scale-out (a **cooldown**), so that a dip does not remove capacity just before the next rise.

### Concrete: capacity planning with Little's Law

```text
 target peak: 6,000 requests/s; measured mean latency at healthy load: 80 ms
 in-flight requests at peak (Little's Law, Chapter 11, Section 11.3) = 6,000 × 0.08 = 480
 per-instance safe concurrency (from a load test, at the knee of the latency curve): 40
 → at the knee: 480 / 40 = 12 instances
 headroom for latency: run at ~70% of the knee → 12 / 0.7 ≈ 17
 N+1 across 3 zones (Chapter 24, Section 24.11): survive losing a zone → each zone must hold
   17 / 2 = 8.5 → 9 per zone → 27 instances at peak
 downstream check: 27 instances × 10 DB connections each = 270 > the database's 200 limit
   → a pooler or a smaller per-instance pool (Chapter 18, Section 18.5) before scaling to 27
```

The last line is the step most often missed: scaling the application tier multiplies everything each instance holds downstream (database connections, connections to other services, cache connections, rate-limit share). The scarcest downstream resource sets the real limit (Chapter 11, Section 11.5).

Running in several **regions** (for latency, or to survive the loss of a region) multiplies the same arithmetic: each region needs capacity for its own traffic plus whatever it absorbs when another region fails, and the data tier must be designed for it (replication across regions, Chapter 22, with its latency and consistency trade-offs).

### Mechanism: load testing, the evidence behind the numbers

Every number above ("safe concurrency 40 per instance", "the knee") comes from a **load test**: generating artificial traffic against a service to measure how it behaves. Four kinds answer different questions:

| Test | Traffic shape | Question it answers |
|---|---|---|
| **load test** | the expected peak, held steady | does the service meet its latency target at expected peak? |
| **stress test** | increasing until it breaks | where is the knee, which resource saturates first, and does it fail gracefully (shedding) or collapse? (Chapter 11, Section 11.8) |
| **soak test** (endurance) | normal load for hours or days | do memory, connections, file descriptors or disk grow slowly (leaks)? |
| **spike test** | a sudden jump, for example 5× in seconds | do headroom, admission control and autoscaling (with its lag) absorb a surge? |

Three rules make the results trustworthy. Use production-like data and traffic mix, because caches, indexes and skew (Chapter 21, Section 21.1) change everything. Generate requests at a fixed rate (an **open model**) rather than waiting for each response before sending the next (a **closed model**), or stalls hide themselves (coordinated omission, Section 26.9). And watch the downstream resources (database, pools, dependencies), not only the service under test, because they are usually what breaks first.

### Guarantees and non-guarantees

Stateless instances guarantee that capacity can be added and removed without losing state. Autoscaling guarantees that capacity eventually follows its signal. Neither guarantees capacity during the minutes of scaling lag, that the signal reflects the real bottleneck, or that downstream dependencies can absorb the extra instances.

### Failure and diagnosis

- **Scaling does not help.** More instances, same latency. Evidence: the bottleneck is downstream (database CPU, a partner API, connection limits), visible in per-dependency latency and in USE metrics for the shared resource (Section 26.10). Fix: scale or protect the bottleneck; adding callers makes it worse.
- **Flapping.** Instance count oscillating every few minutes. Evidence: autoscaler events; the signal crossing the target repeatedly. Fix: longer windows, a cooldown, a dead band around the target.
- **Spike outages despite autoscaling.** Evidence: latency collapses within the first minute of a spike; new instances arrive after four minutes. Fix: headroom, scheduled scaling, admission control.
- **A new instance makes things worse.** Evidence: errors from fresh instances in their first minute. Cause: cold starts reported ready too early. Fix: warm-up before readiness, slow start.

### Backend connection

Make every service stateless enough to be killed at any time, choose an autoscaling signal that tracks the real bottleneck (concurrency or queue lag rather than CPU for I/O-bound services), keep headroom for the minutes autoscaling needs, and plan capacity with Little's Law down to the scarcest downstream resource. Scale the application tier only as far as its dependencies can follow.

---

## 26.3 Orchestration Concepts

### Definitions

- A **container orchestrator** is a system that runs containers across a cluster of machines (**nodes**) and keeps them running according to a declared specification. Kubernetes is the most widely used; the concepts here apply to others.
- **Desired state** is the declaration of what should run: "6 instances of image orders@sha256:9d1e…, each with 1 CPU and 2 GiB, reachable as the service `orders`". **Actual state** is what is running now.
- A **reconciliation loop** (controller) repeatedly compares desired and actual state and acts to close the difference: starts missing instances, replaces failed ones, removes extra ones.
- **Scheduling** is choosing a node for each new instance, based on its resource requests and constraints.
- A **pod** (in Kubernetes) is the smallest deployable unit: one or more containers always scheduled together on one node, sharing a network namespace (Chapter 4, Section 4.10). A **sidecar** is a helper container in the same pod (a proxy, a log shipper).
- **Service discovery** is how a client finds the current network addresses of the instances of a service, which change whenever instances are replaced.
- **Resource requests** are the resources an instance is guaranteed and is scheduled by; **limits** are the maximum it may use.

### Why it exists

Instances fail, nodes fail, deploys replace everything, and autoscaling changes counts every hour. Doing this by hand (or with scripts that run commands in order) breaks as soon as something fails halfway. An orchestrator turns operations into declarations: you state the result, and controllers keep pushing reality toward it, forever, including after failures nobody anticipated.

### Mechanism: desired state and reconciliation

```text
 desired: orders × 6                              controller loop (every few seconds, and on events)
 actual:  orders × 6 running                      observe → compare → act
   node 3 dies at 10:02 → actual: 4 running        4 < 6 → create 2 instances
                                                   scheduler: pick nodes with free requested CPU and
                                                   memory, honouring zone spreading
   10:03: 6 running again                          nothing to do
 deploy at 11:00: desired image changes            old ≠ new → replace instances gradually,
                                                   per the rollout strategy (Section 26.4)
```

The same pattern runs everywhere in an orchestrator: one controller keeps instance counts, another keeps service endpoints up to date, another restarts failed containers, another scales on metrics. None of them executes a plan once; each converges continuously.

The same declarative idea applies below the orchestrator. **Infrastructure as code** (IaC) describes networks, load balancers, databases, DNS records and permissions in versioned files; a tool compares the files with what exists, shows the planned changes, and applies them. Infrastructure changes then get what code changes get: review, history, repeatability across environments, and rollback, instead of hand-made changes in a console that nobody can reproduce (Terraform and OpenTofu are common examples). The incident risk is the same as for configuration (Section 26.5): one reviewed-but-wrong file can change every environment at once, so infrastructure changes are planned, reviewed and applied in stages too.

### Mechanism: requests, limits and cgroups

Chapter 4 (Section 4.10) and Chapter 3 (Section 3.12) explained the kernel side. The orchestrator writes the cgroup files:

| Setting | Scheduler uses it to | Becomes (cgroup v2) | Effect at run time |
|---|---|---|---|
| CPU request | place the instance on a node with that much unrequested CPU | `cpu.weight` | the instance's share of CPU when the node is contended |
| CPU limit | nothing | `cpu.max` (quota per period) | **throttling**: the instance is paused for the rest of each period once it has used its quota, even if the node is idle |
| memory request | place the instance | used for eviction decisions under node memory pressure | instances using more than requested are evicted first |
| memory limit | nothing | `memory.max` | exceeding it → OOM kill, exit code 137 (Chapter 4, Section 4.10) |

CPU limits cause one of the most confusing production symptoms: **high tail latency at low average CPU**. A service with a limit of 1 CPU (100 ms of CPU per 100 ms period) that receives a burst using 4 threads for 30 ms each consumes its whole quota in 25 ms of wall time, then is throttled for the remaining 75 ms of the period: requests in that burst wait 75 ms doing nothing, while the CPU graph, averaged over a minute, shows 30%. The evidence is the throttling counters (`nr_throttled`, `throttled_usec` in `cpu.stat`, Chapter 4). Common practice is to set memory limits always (to contain leaks) and to be careful with CPU limits for latency-sensitive services, using requests to secure a fair share instead. Runtimes must also be told about their limits: thread pools and heaps sized from the host's CPU count and memory, rather than the cgroup's, overload the instance (Chapter 4, Section 4.10).

### Mechanism: probes and their failure modes

Chapter 10 (Section 10.3) and Chapter 24 (Section 24.10) defined the checks. The orchestrator acts on them:

| Probe | Orchestrator's action on failure | Should check | Misconfiguration and its symptom |
|---|---|---|---|
| startup | waits; kills after the startup deadline | initialisation finished | too short a deadline → slow-starting instances killed before they ever start |
| liveness | **restarts** the container | the process can respond at all | depends on the database → all instances restart when the database blips; too short a timeout → restarts under load |
| readiness | removes the instance from service endpoints (no traffic), does not restart | ready for traffic now: warmed, not draining, not overloaded | depends on shared dependencies → all instances removed at once; never fails → traffic to instances still starting |

### Concrete: a restart loop from a misconfigured liveness probe

```text
 liveness probe: GET /health every 10 s, timeout 1 s, 3 failures → restart
 /health runs "SELECT 1" on the database and checks two downstream services

 14:00:00  the database's latency rises to 1.5 s under a heavy batch job
 14:00:30  /health takes > 1 s on every instance → 3 consecutive failures → ALL instances restarted
 14:00:40  instances start: JIT cold, pools empty, caches empty (Chapter 2, Section 2.5);
           start-up itself queries the database → slower still
 14:01:10  liveness fails again during warm-up (database still slow) → restarted again
 ...       restart counts climbing; the service is down although no instance was ever broken;
           each restart adds connection storms to the struggling database (a cascade, Chapter 24)
 fix: liveness checks only the process (an endpoint that returns immediately unless the event
      loop or worker pool is stuck); dependencies handled by timeouts, breakers and degradation;
      a startup probe with a generous deadline protects warm-up
```

### Mechanism: service discovery

Instances get new addresses every time they are replaced, so clients cannot be configured with addresses. Two designs:

```text
 SERVER-SIDE DISCOVERY                         CLIENT-SIDE DISCOVERY
 client → stable name / virtual IP             client asks a registry for the current endpoint
          (DNS name "orders", Chapter 6) →      list, caches it, and balances across it itself
          a proxy or kernel rules forward      (or a sidecar proxy does it for the client:
          to a current, ready instance          the service-mesh model, Chapter 10, Section 10.4)
 simple for clients; one more hop or           no extra hop; per-request balancing; but every
 per-connection balancing only                 client must refresh its list and handle stale entries
```

Either way, the endpoint list contains only **ready** instances, which is how readiness probes and graceful shutdown keep traffic away from starting and draining instances. Stale caches of the list are why the propagation delay of Chapter 24 (Section 24.10) exists: a client holding yesterday's list sends requests to instances that are gone.

Orchestrators also run **scheduled jobs** (Chapter 11, Section 11.7 pointed here): a cron-like resource that starts a container on a schedule, once per period across the cluster, which removes the "every instance runs the cron job" problem (Chapter 25, Section 25.2), though jobs must still be idempotent, because a job can be started twice around node failures.

> **Framework lens:** In Kubernetes terms: a Deployment declares desired replicas and the rollout strategy, a ReplicaSet keeps the count, the scheduler places Pods, a Service gives a stable name and virtual IP over ready Pods (EndpointSlices hold the current list), probes are declared per container, the HorizontalPodAutoscaler implements the formula of Section 26.2, and a CronJob runs scheduled jobs. These names are product vocabulary; the concepts above are what matter.

### Guarantees and non-guarantees

An orchestrator guarantees continuous convergence toward the declared state: failed instances replaced, counts maintained, unready instances out of endpoint lists. It does not guarantee that the declared state is correct (a bad image is rolled out as faithfully as a good one), that probes are meaningful, that limits fit the workload, or that there is enough cluster capacity to schedule what is requested.

### Failure and diagnosis

- **Restart loops.** Evidence: restart counts rising; the "last termination reason" showing liveness failures or OOM kills (exit 137). Fix: local liveness; memory limits that fit the heap plus non-heap memory.
- **Latency spikes with low CPU.** Evidence: `cpu.stat` throttling counters rising with p99 latency. Fix: raise or remove CPU limits for latency-sensitive services; size runtime thread pools to the quota.
- **Instances that cannot be scheduled.** Evidence: instances pending with "insufficient CPU" or "insufficient memory" events. Fix: right-size requests (they reserve capacity even when unused), add nodes, or reduce replicas.
- **Requests to instances that no longer exist.** Evidence: connection errors right after rollouts. Cause: stale discovery caches, no propagation delay. Fix: Chapter 24's shutdown protocol; clients that refresh endpoints.

### Backend connection

Declare what you want and let controllers converge, but remember that the orchestrator only acts on what you tell it: probes that are local and meaningful, requests that reflect real usage, memory limits that include non-heap memory, CPU limits chosen knowing that they throttle, and runtimes that read their cgroup limits. When something misbehaves in an orchestrated system, read the orchestrator's events and the container's termination reason before reading the application's logs.

---

## 26.4 Deployment Strategies

### Definitions

- A **rolling deployment** replaces instances of the old version with the new one in small batches until all run the new version.
- A **blue–green deployment** runs two complete environments: **blue** (current) and **green** (new). Traffic switches from blue to green all at once; blue stays ready for an instant switch back.
- A **canary deployment** sends a small share of traffic to the new version first, compares its behaviour with the old version, and increases the share step by step only while the comparison stays healthy.
- A **rollback** returns to the previous version. **Roll-forward** fixes a problem by deploying a new version instead.
- **Decoupling deploy from release**: deploying code with new behaviour switched off behind a feature flag (Chapter 12, Section 12.9), and releasing it later, separately, by flipping the flag.

### Why it exists

Most production incidents are caused by changes: code, configuration, infrastructure. A deployment strategy decides how much of the system a bad change can affect before it is noticed, how quickly it is noticed, and how quickly it can be undone. "Deploy everything at once and hope" maximises all three costs.

### Concrete: the three strategies, with rollback paths

```text
 ROLLING (batch of 2 of 6, max 1 extra instance)
 t0  v1 v1 v1 v1 v1 v1
 t1  v2 v2 v1 v1 v1 v1        each new instance must pass readiness before the next batch
 t2  v2 v2 v2 v2 v1 v1        old instances drained gracefully (Chapter 24, Section 24.10)
 t3  v2 v2 v2 v2 v2 v2
 rollback: roll v1 back out the same way (minutes); during the rollout, BOTH versions serve
 traffic, so v1 and v2 must be compatible with each other and with the database
 cost: no extra capacity beyond a small surge

 BLUE–GREEN
 router ──► BLUE (v1) × 6          GREEN (v2) × 6 started, tested with synthetic traffic
 switch:  router ──► GREEN (v2)    all traffic at once; BLUE kept running, idle
 rollback: switch the router back to BLUE (seconds), as long as BLUE is kept and the database
 is still compatible with v1
 cost: double capacity during the switch; a single big moment of risk (all users at once)

 CANARY
 step 1:  v2 gets  1% of traffic   compare v2 with v1: error rate, p99 latency, business metrics
 step 2:  v2 gets 10%              (automated canary analysis: same time window, same traffic mix)
 step 3:  v2 gets 50%
 step 4:  v2 gets 100%
 rollback: set v2's share to 0% at ANY step (seconds); only that step's share of users was affected
 cost: routing that can split traffic; good metrics per version; a slower rollout
```

| | Rolling | Blue–green | Canary |
|---|---|---|---|
| Blast radius of a bad version | grows batch by batch (up to 100% if unnoticed) | 100% at the switch | 1%, then 10%, … |
| Rollback speed | minutes (another rollout) | seconds (switch back) | seconds (weight to 0) |
| Extra capacity | small surge | 2× during the switch | small |
| Mixed versions serving at once | yes, during the rollout | briefly, at the switch | yes, by design |
| Detects problems that need real traffic | late, if at all | at 100% | early, at small share |

Canaries are the strongest protection against bad releases because they combine a small blast radius with real traffic, but they depend on observability that can compare versions (metrics labelled by version, Section 26.7) and on traffic large enough for 1% to be statistically meaningful. Many teams combine strategies: a canary for the first steps, then a rolling completion. **Configuration changes** deserve the same treatment (Chapter 12, Section 12.9): a bad configuration value pushed to every instance at once is as dangerous as bad code, so stage configuration rollouts by instance, zone or region too.

### Mechanism: feature flags in deployment

Feature flags let a deploy ship code that is inactive, and a release activate it gradually:

```text
 deploy v2 (flag "new-checkout" = off for everyone)        → no behaviour change; safe rollback
 release: flag on for employees → 1% of users → 10% → 100%  → each step reversible in seconds
 cleanup: remove the flag and the old code path             → flags left behind become debt
```

This separates the risk of new **code** (caught by the deploy strategy) from the risk of new **behaviour** (caught by the gradual release), and lets a feature be switched off without a deploy (a kill switch, Chapter 24, Section 24.9).

### Mechanism: backward-compatible deploys with database migrations

Every strategy above runs old and new code at the same time against **one** database: during a rolling deploy for minutes, during a canary for hours, and after a rollback the old code runs against whatever schema the new code left. Chapter 18 (Section 18.15) gave the method, **expand and contract**; the deployment rule that goes with it is that **every schema state must work with both the current and the previous version of the code**:

```text
 renaming column customers.phone → phone_number, without downtime
 deploy A (expand):   migration adds phone_number (nullable); code v2 WRITES both columns,
                      READS phone (old)                        → v1 and v2 both work
 backfill:            copy phone → phone_number in batches (Chapter 18, Section 18.14)
 deploy B:            code v3 READS phone_number, still WRITES both  → v2 and v3 both work
 deploy C (contract): code v4 stops writing phone                    → v3 and v4 both work
 later migration:     drop phone, once no running or rollback version uses it
 never: a single deploy that renames the column and changes the code together;
        during the rollout, old instances fail on the missing column, and rollback is impossible
```

The same rule covers API and message formats (Chapter 13, Sections 13.9 and 13.11; Chapter 25, Section 25.10): during a rollout, the new version receives requests and messages from old clients and old producers, and its own messages are read by old consumers.

### Guarantees and non-guarantees

A canary with automated analysis guarantees that a regression visible in the compared metrics affects only the canary's share of traffic before rollback. Blue–green guarantees a fast, complete switch back while blue is kept. Rolling deployments guarantee gradual replacement with limited extra capacity. None of them guarantees detection of problems the metrics do not measure, problems that appear only at full load or after hours (memory leaks, slow data corruption), or rollback after an incompatible schema change.

### Failure and diagnosis

- **A bad release reaches everyone.** Evidence: error rate rising in step with the rollout percentage. Fix: canary with automatic rollback on SLO regression; smaller first steps.
- **Errors only during deploys.** Evidence: errors in the minutes of each rollout, from mixed versions or from shutdown. Causes: incompatible API, message or schema changes between versions; missing graceful shutdown. Fix: expand and contract; Chapter 24's shutdown protocol.
- **Rollback fails.** Evidence: the previous version errors on start or on queries after rollback. Cause: a migration the old code cannot handle. Fix: every migration compatible with the previous version; contract steps only after the expanded version has been stable.
- **A canary that hides problems.** Evidence: the canary passed, the full rollout failed. Cause: too little traffic in the canary, a canary served by different infrastructure, or metrics that do not cover the failing behaviour. Fix: longer canary steps, version-labelled metrics for key business flows.

### Backend connection

Deploy small changes often, through a canary (or at least a rolling deploy with automatic halting) that compares the new version with the old on error rate, latency and a key business metric, and rolls back automatically. Keep every pair of adjacent versions compatible with each other and with the database, separate deploys from releases with flags, and treat configuration changes as deployments.

---

## 26.5 Configuration and Secret Delivery in Production

### Definitions

Chapter 12 (Section 12.9) defined **configuration** (everything that varies between deployments), the twelve-factor rule of keeping it out of the artifact, typed configuration validated at start-up, and the ways secrets are delivered. Chapter 20 (Section 20.11) covered secret managers and the lifecycle of a secret. This section adds what production adds:

- **Static configuration** is read at start-up; changing it means restarting or replacing instances. **Dynamic configuration** is read at run time from a configuration service and can change without a restart.
- A **configuration version** is an immutable, numbered snapshot of an environment's configuration, so that every instance can report which one it runs and any change can be rolled back.

### Why it exists

Configuration changes cause outages as often as code changes: a timeout reduced from 2,000 to 200 by a typo in the unit, a feature flag enabled for 100% instead of 1%, a connection-pool size multiplied across 200 instances. Unlike code, configuration often bypasses review, tests and staged rollout. Production configuration needs the same discipline as code because it **is** code, written in a different file.

### Mechanism: configuration as a release

```text
 config repository (reviewed like code) ─► validation in CI (schema, types, ranges:
                                            "timeout_ms must be 50–30,000")
       ─► version 413 published to the config store
       ─► staged rollout: canary instances → one zone → all zones   (Section 26.4)
       ─► each instance reports "config_version=413" in a metric and in its logs
       ─► rollback = publish version 412 again
 secrets: referenced by name in configuration ("db_password: secret://orders/db"), resolved at
          run time from the secret manager with the instance's workload identity
          (Chapter 20, Sections 20.10–20.11); never stored in the config repository
```

Dynamic configuration is powerful (kill switches and flags work through it, Chapter 24, Section 24.9) and dangerous for the same reason: one change reaches every instance within seconds. Instances should cache the last good configuration and keep running on it if the configuration service is unavailable (static stability), and should reject a new configuration that fails validation rather than apply it partially.

### Guarantees and non-guarantees

Versioned, validated, staged configuration guarantees that every change is reviewed, traceable, gradual and reversible. It does not guarantee that a valid value is a good one (a valid timeout can still be too short), or that dynamic changes are safe at full speed.

### Failure and diagnosis

- **An outage right after a configuration change.** Evidence: the incident's start aligns with a configuration version change in the instances' reported versions. Fix: roll back the version; then add range validation and staging for that key.
- **Instances running different configurations.** Evidence: the `config_version` metric shows two values for longer than a rollout. Cause: failed reloads, instances that cannot reach the store. Fix: alerting on version skew.
- **A secret rotated, instances failing.** Evidence: authentication errors after rotation (Chapter 20, Section 20.11). Fix: the accept-both overlap phase; instances that re-read secrets.

### Backend connection

Keep configuration in a reviewed repository, validate types and ranges in the pipeline, publish immutable versions, roll them out in stages, and make every instance report the version it runs. Reference secrets by name and resolve them at run time. Then "what changed?" covers configuration as precisely as it covers code.

---

## 26.6 Observability and Logs

### Definitions

- **Observability** is the ability to understand a system's internal state from the evidence it emits, well enough to answer new questions about its behaviour without deploying new code. **Monitoring** is watching known indicators and alerting when they cross thresholds; it answers questions asked in advance. Observability is what you need for the questions nobody asked in advance.
- **Telemetry** is the evidence: the three main **signals** are **logs** (timestamped records of discrete events), **metrics** (numeric measurements aggregated over time) and **traces** (the path and timing of individual requests through a system). Profiles (Section 26.12) are often called the fourth.
- A **structured log** is a log record written as fields (keys and values, usually JSON) rather than as free text.
- A **log level** (severity) classifies records: `DEBUG` (detail for developers), `INFO` (normal significant events), `WARN` (something unexpected that was handled), `ERROR` (an operation failed), and in some systems `TRACE` and `FATAL`.
- A **correlation ID** (request ID) is an identifier attached to everything done on behalf of one request, across services; a **trace ID** is the same idea in tracing (Section 26.8), and often the same value.

### Why it exists

A production system cannot be stepped through in a debugger. When a customer reports that their payment failed at 14:02, the only way to know what happened is evidence recorded at the time. Each signal answers a different kind of question: metrics answer "how much, how often, how fast, in aggregate?" cheaply; traces answer "where did this request spend its time?"; logs answer "what exactly happened, with which values?". Observability practice is choosing what evidence to record so that the questions of a future incident have answers.

### Mechanism: the three signals compared

| | Logs | Metrics | Traces |
|---|---|---|---|
| Unit | one event | a number per time interval, per label set | one request's tree of operations (spans) |
| Answers | what happened, with details | how much, how often, how fast, trends | where the time went, across services |
| Cost driver | volume (events × size) | number of distinct time series (cardinality, Section 26.7) | requests traced × spans (hence sampling) |
| Retention | days to weeks (longer for audit logs) | months to years (aggregated) | days |
| Best for | debugging a specific case; audit | alerting, dashboards, capacity | latency diagnosis, dependency mapping |
| Weak at | aggregate questions at scale | individual cases | aggregate rates (sampled) |

The signals work as a chain during diagnosis: a **metric** alerts ("p99 latency high"), **traces** for slow requests show which service and which call are slow, and **logs** for those traces (found by trace ID) show the error or the values involved (Section 26.13 and Diagnosis Lab 1).

### Concrete: a structured log line

```json
{
  "ts": "2026-10-24T14:02:11.482Z",
  "level": "ERROR",
  "service": "payments",
  "version": "3f9a1c2",
  "instance": "payments-6d8b-x2k9p",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "request_id": "req-7f3a91",
  "event": "provider_call_failed",
  "msg": "card authorisation failed",
  "order_id": "9001",
  "provider": "acme-pay",
  "attempt": 2,
  "duration_ms": 3004,
  "error.type": "timeout",
  "user_id_hash": "u:5f2c…"
}
```

Each field earns its place. `ts` in UTC with milliseconds lets events from different machines be merged (with the clock caveats of Chapter 23, Section 23.6). `service`, `version` and `instance` show whether the problem is one instance or one release. `trace_id` and `span_id` link the line to its trace; `request_id` to proxy logs (Chapter 10, Section 10.11). `event` is a stable machine-readable name that can be counted, while `msg` is for humans. Business identifiers (`order_id`) let support find the case. The user appears only as a hash.

### Mechanism: log hygiene

- **Levels mean something.** `ERROR` means an operation failed and someone may need to act; a handled, expected condition (a user's invalid input, a cache miss) is `INFO` or `DEBUG`. When everything is an error, nothing is.
- **Log once per failure, at the boundary that handles it** (Chapter 12, Section 12.8): not once in every layer it passes through.
- **Personal data and secrets do not enter logs.** Never log passwords, tokens, session IDs, API keys, card numbers or full request bodies (Chapter 20, Sections 20.2 and 20.11). Log identifiers and hashes instead of emails and names. Enforce it with **redaction at the logging layer**: the logging library or the log pipeline masks fields with known sensitive names (`password`, `authorization`, `token`) and patterns (card numbers, key prefixes) before records leave the process, so that one careless log statement does not become a breach. Logs are copied to many systems and kept for weeks; treat them as data that will leak.
- **Volume is a cost and a risk.** A hot path that logs at `INFO` for every request can produce terabytes a day and slow the request itself. Sample high-volume success logs; keep all errors; use metrics for counts.
- **Logs go to standard output** in containers (Chapter 4, Section 4.11) and are shipped by an agent to a central store, where they are indexed by fields and searchable across instances.

### Mechanism: audit logs and security monitoring

Chapter 20 (Section 20.12) defined the audit trail and what it must record. In production, audit events are a separate stream from debugging logs: written as structured events, shipped to storage that the producing services cannot modify or delete (append-only, separate access control), retained for the period that regulation or investigation needs, and watched. **Security monitoring** turns them into alerts: bursts of failed logins (credential stuffing, Chapter 20, Section 20.2), refresh-token reuse detections (Section 20.6), bursts of authorisation denials across tenants (IDOR probing, Section 20.9), secret reads by unusual identities, and administrative actions outside working patterns. Organisations often send these streams to a **SIEM** (security information and event management system) that correlates events across systems (Section 26.14).

### Guarantees and non-guarantees

Structured, correlated logs guarantee that a specific event can be found and connected to its request and trace, if it was logged and retained. They do not guarantee that the event you need was logged, that logs arrived (pipelines drop records under overload), that timestamps from different machines order events correctly, or that logs are free of personal data unless redaction is enforced.

### Failure and diagnosis

- **The log you need does not exist.** Evidence: an incident where the failing step logged nothing, or logged only "error". Fix: log failures with context at the handling boundary; review logging in postmortems.
- **Secrets or personal data in logs.** Evidence: a scan of the log store finds tokens or emails. Fix: redaction at the logging layer, rotate exposed secrets (Chapter 20, Section 20.11), purge affected logs as policy requires.
- **Logging slows the service.** Evidence: profiles showing time in log formatting or synchronous writes; latency tied to log volume. Fix: asynchronous appenders with bounded buffers, sampling, fewer logs on hot paths.
- **Logs that cannot be joined.** Evidence: no trace or request ID in application logs. Fix: propagate the context and add it to every record automatically (in a logging context bound to the request, Chapter 11, Section 11.6).

### Backend connection

Emit structured logs with a stable event name, service, version, instance, trace ID and request ID on every record, at levels that mean something, with personal data and secrets masked by the logging layer itself. Ship audit events to a separate, protected stream and alert on the security patterns they reveal. Use logs for details and specific cases; use metrics for counting.

---

## 26.7 Metrics

### Definitions

- A **metric** is a numeric measurement identified by a name and a set of **labels** (dimensions): `http_requests_total{service="orders", route="/orders/{id}", method="GET", status="200"}`.
- A **time series** is the sequence of values of one metric with one exact combination of label values over time.
- **Cardinality** is the number of distinct time series a metric produces: the product of the number of distinct values of each label.
- The common **metric types**:
  - a **counter** only increases (and resets to zero on restart): requests served, errors, bytes sent; one reads its **rate** of increase;
  - a **gauge** goes up and down: queue depth, memory in use, in-flight requests, temperature;
  - a **histogram** counts observations into **buckets** by value (requests that took ≤ 10 ms, ≤ 25 ms, ≤ 50 ms, …), plus their sum and count, from which distributions and percentiles are estimated (Section 26.9);
  - a **summary** computes quantiles inside the process over a recent window.

### Why it exists

Metrics are the cheapest way to watch everything all the time. A counter incremented a million times a second costs a few nanoseconds per increment and produces one number per scrape interval, regardless of traffic. That makes metrics the basis of dashboards, alerts, autoscaling and capacity planning, where logs and traces would be too expensive or too slow to aggregate.

### Concrete: metric types with examples

| Type | Example | How it is read | Pitfall |
|---|---|---|---|
| counter | `http_requests_total{route, status}` | rate per second over a window: requests/s, errors/s, error ratio | reading the raw value (it only grows); resets on restart (rate functions handle them) |
| gauge | `db_pool_in_use{pool}`, `queue_depth{queue}`, `jvm_heap_used_bytes` | current value, min, max over a window | a gauge sampled every 15 s misses spikes between samples |
| histogram | `http_request_duration_seconds_bucket{route, le="0.05"}` | percentiles across instances and time; share of requests under a threshold | bucket boundaries chosen badly hide the region you care about |
| summary | `rpc_latency{quantile="0.99"}` computed in the process | per-instance quantiles | quantiles cannot be averaged or combined across instances |

The histogram-versus-summary distinction matters in practice. A histogram's buckets can be **summed across instances** and then turned into a percentile for the whole service. Summaries' quantiles cannot: the average of 20 instances' p99s is not the service's p99 (Section 26.9). Histograms are the usual choice for latency.

### Mechanism: how metrics are collected

```text
 in the process: instrumentation updates counters, gauges and histogram buckets in memory
                 (atomic increments, Chapter 5, Section 5.4; no I/O per request)
 collection:     PULL: a metrics server scrapes each instance's /metrics endpoint every 15–60 s
                 PUSH: the process sends aggregates to a collector periodically
 storage:        a time-series database stores (series, timestamp, value) compactly
 queries:        rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
                 → the 5xx error ratio over the last 5 minutes, per series or summed
```

### Mechanism: the high-cardinality pitfall

Every distinct label combination is a separate series that must be stored, indexed and queried. Cardinality multiplies:

```text
 http_request_duration_seconds_bucket{service, instance, route, method, status, le}
   services 30 × instances 10 × routes 40 × methods 3 × statuses 8 × buckets 12
   = 30 × 10 × 40 × 3 × 8 × 12 ≈ 3.5 million series          (already large)

 someone adds label user_id (500,000 active users):
   3.5 million × 500,000 = 1.7 × 10^12 series                  → the metrics system falls over
 also dangerous: raw URL paths ("/orders/9001" instead of the route template "/orders/{id}"),
 request IDs, email addresses, full error messages, timestamps as label values
```

Symptoms are memory exhaustion and slow queries in the metrics system, sometimes failing alerting for **every** service at once. The rule: **labels have bounded, small value sets** (route templates, status classes, regions, versions). Per-user, per-order or per-request detail belongs in logs and traces, which are built for high-cardinality data.

### Mechanism: what to measure

A useful service exposes, at minimum: the RED metrics for every endpoint and every dependency it calls (rate, errors, duration histograms; Section 26.10); USE metrics for its resources (pool utilisation and waiters, queue depths, thread usage, GC time, CPU throttling); the state of its resilience mechanisms (breaker state, shed and rate-limited requests, retries; Chapter 24, Section 24.13); and a few **business metrics** (orders placed per minute, payments succeeded), which often reveal failures that technical metrics miss. Label request metrics with `version`, so that a canary can be compared with the stable version (Section 26.4).

### Guarantees and non-guarantees

Metrics guarantee cheap, continuous, aggregated measurements at the resolution of their scrape interval and buckets. They do not show individual requests, do not capture what happens between samples (gauges), cannot be combined if computed as per-instance quantiles, and break the monitoring system itself if their cardinality is unbounded.

### Failure and diagnosis

- **A cardinality explosion.** Evidence: the metrics system's memory and series count jumping after a deploy; one metric dominating the series count. Fix: remove the unbounded label; enforce label allowlists and series limits per service.
- **Averaged quantiles.** Evidence: a dashboard's "p99" lower than any individual instance's real tail; computed as `avg(p99)`. Fix: histograms aggregated before computing percentiles.
- **Invisible spikes.** Evidence: users report brief slowdowns that no graph shows. Cause: gauges sampled every minute, averages over long windows. Fix: histograms (which count every request), shorter windows for alerts, max-over-time for gauges.

### Backend connection

Instrument every endpoint and dependency with request counters and latency histograms, every pool and queue with gauges, every resilience mechanism with its decisions, and a few business flows end to end. Keep label values bounded and low in number, label by version for deployment comparisons, and push high-cardinality detail into logs and traces.

---

## 26.8 Distributed Tracing

### Definitions

- A **trace** records the path of one request through a distributed system as a tree of **spans**.
- A **span** is one timed operation: a request handled by a service, a database query, an outbound call, a message published or consumed. It has a name, a start time, a duration, a **span ID**, its **parent span ID**, the **trace ID** shared by all spans of the trace, attributes (key–value details), and a status.
- **Context propagation** carries the trace ID and the current span ID from one service to the next with each call, so that the receiving service's spans join the same trace. The W3C **Trace Context** standard defines the `traceparent` header for HTTP (Chapter 7, Section 7.6); message headers carry it through brokers (Chapter 25).
- **Sampling** decides which requests are traced. **Head-based sampling** decides at the start of a request (for example, trace 1%); **tail-based sampling** collects spans for all requests briefly and decides at the end, keeping the slow and the failed ones.
- **OpenTelemetry** is a vendor-neutral open standard and set of libraries for producing logs, metrics and traces: an API that code calls, SDKs that implement it, a wire protocol (OTLP) and a **collector** that receives, processes and forwards telemetry to whatever back-end stores it.

### Why it exists

In a single process, a profiler or a stack trace shows where time goes. Across ten services, no single process sees the whole request: each service's metrics say "I was fast" or "I was slow", but not why the user waited 2.3 seconds. A trace stitches the request back together, showing which calls were sequential, which parallel, which slow, which retried, and which failed.

### Mechanism: context propagation

```text
 client → gateway: no traceparent → gateway starts a trace
          traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
                       ver  trace ID (16 bytes, hex)        parent span ID     flags (01 = sampled)
 gateway → orders:  same trace ID, parent span ID = the gateway's outbound call span
 orders → payments: same trace ID, parent = orders' outbound call span
 orders → broker:   traceparent in the message headers → the consumer's span joins the trace
 each service sends its finished spans (asynchronously, batched) to a collector; the back-end
 assembles the tree by trace ID and parent span IDs
```

Propagation must be unbroken: one service or one asynchronous hop (a thread pool, a queue) that drops the context splits the trace in two (Chapter 11, Section 11.6 on carrying request context across threads).

### Concrete: a trace waterfall across four services

```text
 trace 4bf92f35…  POST /checkout   total 2,310 ms   status ERROR→recovered (fallback)
 0 ms        500        1000       1500       2000       2310
 |----------|----------|----------|----------|----------|
 gateway  POST /checkout ███████████████████████████████████████████████████  2,310 ms
   auth   verify token   ██                                                       38 ms
   orders POST /orders     ████████████████████████████████████████████████    2,240 ms
     db   SELECT cart       █                                                     12 ms
     inventory POST /reserve  ████                                               180 ms
       db   UPDATE stock         ██                                               95 ms
     payments POST /authorize      ████████████████████████████████████  1,640 ms
       http acme-pay attempt 1      ███████████████████   timeout             1,000 ms ✕
       http acme-pay attempt 2                           ██████████  OK          580 ms
     db   INSERT order + outbox                                            ██     41 ms
     (fallback: recommendations skipped, breaker open)                              0 ms
```

Reading it: the request took 2.3 seconds. Authentication, the cart and inventory are fast. Payments took 1.64 seconds because the provider's first attempt timed out at 1 second and the retry succeeded. The fix to investigate is the provider's latency and the timeout and retry policy (Chapter 24, Sections 24.2–24.3), not the orders service, whose own metrics show only "slow". The staircase pattern of sequential spans (Chapter 12, Section 12.11) and the comb of many small database spans (the N+1 problem, Chapter 18, Section 18.11) are the other shapes worth recognising.

### Mechanism: sampling

Tracing every request of a busy system is expensive (each request produces tens of spans). Head-based sampling at 1% is cheap but keeps a random 1%, so the one slow request in ten thousand is probably not in it. Tail-based sampling keeps what matters (all errors, all requests slower than a threshold, plus a small random share) at the cost of buffering spans in collectors until each trace completes. A common combination: head sampling for baseline traffic, plus forced sampling for errors and for requests carrying a debug flag.

### Guarantees and non-guarantees

A propagated, sampled trace guarantees an accurate timeline of that request's spans in every instrumented component. It does not cover uninstrumented components (they appear as gaps), does not include unsampled requests, and does not order spans across machines more precisely than their clocks allow (child spans can appear to start before their parents by a few milliseconds of clock skew).

### Failure and diagnosis

- **Broken traces.** Evidence: traces that end at one service, and orphan traces starting mid-way. Cause: a hop that drops `traceparent` (a proxy, a thread pool, a message consumer). Fix: propagate context in every client, executor and message handler.
- **The slow request is never traced.** Evidence: sampling at a low head rate; slow traces missing. Fix: tail-based sampling for slow and failed requests.
- **Tracing overhead.** Evidence: CPU and network used by span export. Fix: batching, sampling, fewer spans on very hot inner loops.

### Backend connection

Propagate trace context through every HTTP call, message and thread hand-off, instrument incoming requests, outbound calls, database queries and message handling as spans, put the trace ID in every log line, and sample so that errors and slow requests are always kept. Learn to read waterfalls: where the long bar is, whether calls are sequential or parallel, and whether retries hide inside a span.

---

## 26.9 Percentiles and Tail Latency

### Definitions

- A **percentile** pN of a set of measurements is the value below which N% of them fall: the **p50** (median) is the middle value, the **p99** the value that 99% of requests beat. Chapter 2 (Section 2.8) introduced the p99; this section is its home.
- **Tail latency** is the latency of the slowest requests: the p99, p99.9 and maximum.
- A **latency distribution** is the full shape of how many requests took how long, usually drawn as a histogram.

### Why it exists

Latency is not a single number. A service's requests take different times: most are quick, some wait for a lock, a cache miss, a garbage-collection pause, a retry. The average blends them into a value that describes almost no actual request. Users experience individual requests, and heavy users experience many of them, so the tail is what decides whether a system feels fast.

### Concrete: an average that lies

One minute of a service's requests, 10,000 in total:

```text
 latency bucket       requests     share
 0–20 ms                 7,800     78%
 20–50 ms                1,600     16%
 50–200 ms                 450      4.5%
 200 ms–2 s                120      1.2%
 2–5 s (GC pauses, a slow lock, retries)  30   0.3%

 average ≈ 45 ms     p50 ≈ 12 ms     p90 ≈ 35 ms     p99 ≈ 480 ms     p99.9 ≈ 2.6 s
 (computed from the individual measurements, not from the bucket boundaries)
```

The average (about 45 ms) is almost four times the median and less than a tenth of the p99. It describes no group of requests: most users see about 12 ms, one request in a hundred takes about half a second, and one in a thousand takes several seconds. A dashboard showing "average latency 45 ms" would hide both the excellent typical case and the bad tail. Averages also move for the wrong reasons: a few extreme outliers can shift them, while a doubling of p99 can leave them almost unchanged.

The tail matters more than its percentage suggests. A user's page view often makes many requests, and a user's session makes many page views: a user who makes 100 requests has a 1 − 0.99^100 ≈ 63% chance of meeting at least one request slower than the p99. The p99 is not "1% of users"; it is something most active users see.

### Mechanism: percentiles from histograms

Percentiles cannot be computed from averages, and they cannot be averaged. A service of 20 instances whose p99s are 100, 110, …, 290 ms does not have a p99 equal to the mean of those numbers: the real service-wide p99 depends on how many requests each instance served and how their distributions overlap. The correct method is to **sum the histograms** (bucket counts) of all instances and compute the percentile from the merged buckets:

```text
 merged histogram for one minute (cumulative counts, "le" = less than or equal)
   le 0.025 s: 8,900     le 0.05 s: 9,400     le 0.1 s: 9,700     le 0.25 s: 9,880
   le 0.5 s:   9,930     le 1 s:    9,968     le 2.5 s: 9,990     le +Inf:  10,000
 p99 = the value at rank 0.99 × 10,000 = 9,900
       rank 9,900 falls in the bucket (0.25 s, 0.5 s], which holds ranks 9,880–9,930
       linear interpolation: 0.25 + (9,900 − 9,880) / (9,930 − 9,880) × (0.5 − 0.25) = 0.35 s
```

The result is an estimate whose precision depends on bucket boundaries: it can only say "somewhere in the bucket, probably about here". Choose buckets dense around your latency targets (for an SLO of 300 ms, buckets at 200, 250, 300, 350 ms rather than 100 ms and 1 s). Some systems use exponentially spaced or adaptive buckets that keep a bounded relative error across the range.

### Mechanism: coordinated omission, briefly

Load generators and latency recorders often measure the wrong thing during stalls. A tester that sends a request, waits for the reply and then sends the next, sends **fewer** requests while the server is stalled, so a 5-second stall produces one slow sample instead of the hundreds of requests that real users would have sent (and waited on) during those 5 seconds. The measured tail looks far better than users experience. This effect, named **coordinated omission** by Gil Tene, is avoided by load generators that send at a fixed rate regardless of responses and measure each request from when it **should** have been sent.

### Mechanism: tail amplification with fan-out

Chapter 23 (Section 23.5) and Chapter 24 (Section 24.2) met this for scatter–gather and parallel calls; the general rule belongs here. If a request waits for n independent backend calls, it is slow whenever **any** of them is slow:

```text
 P(request avoids every backend's slowest 1%) = 0.99^n
   n = 1:    99%      → 1% of requests see a backend p99-level delay
   n = 10:   90.4%    → 9.6% do
   n = 100:  36.6%    → 63.4% do: the backend's "rare" tail is the request's NORMAL case

 required backend percentile for a target: for the request's p99 to stay below the backends'
 threshold with n = 10 calls, each backend must meet it at p(0.99^(1/10)) = p99.9
```

The engineering responses are the ones from Chapters 23 and 24: fewer fan-out calls per request, hedged requests for the slowest calls, timeouts with partial results for optional data, and tight control of the backends' p99.9, not just their p99.

### Guarantees and non-guarantees

Percentiles from merged histograms guarantee an estimate of the service-wide distribution within bucket resolution. They do not show which requests were slow (traces do), cannot be averaged across instances or time windows, and can be distorted by measurement artefacts such as coordinated omission.

### Failure and diagnosis

- **Dashboards that look fine while users complain.** Evidence: average latency flat; p99 and p99.9 (or the share of requests over the target) rising. Fix: graph percentiles from histograms, and the fraction of requests slower than the SLO threshold.
- **Percentiles that do not add up.** Evidence: a service p99 lower than its slowest instance's p99, computed by averaging. Fix: aggregate buckets first.
- **Load tests better than production.** Evidence: load-test p99 far below production's at the same rate. Cause: coordinated omission, or test traffic without real data skew and cache behaviour. Fix: open-model load generators at fixed rates; production-like data.

### Backend connection

Never judge latency by its average. Record latency in histograms with buckets around your targets, compute percentiles from merged buckets, watch p99 and p99.9 per endpoint, and count the requests slower than your threshold. For any request that fans out, budget backends at higher percentiles than the request itself, and measure with load generators that do not hide stalls.

---

## 26.10 The Golden Signals, RED and USE

### Definitions

- The **four golden signals** (from Google's SRE book) are the metrics to watch for any user-facing system: **latency** (time to serve requests, separately for successful and failed ones), **traffic** (demand: requests per second, messages per second), **errors** (rate of failed requests, explicit or implicit such as wrong content or too slow), and **saturation** (how full the most constrained resource is).
- The **RED method** (Tom Wilkie) applies to every **service** and endpoint: **Rate** (requests per second), **Errors** (failed requests per second), **Duration** (latency distribution).
- The **USE method** (Brendan Gregg) applies to every **resource**: **Utilisation** (fraction of time busy, or fraction of capacity used), **Saturation** (work waiting because the resource is busy: queue length, waiters), **Errors** (error events of the resource).

### Why it exists

A large system offers thousands of metrics, and during an incident there is no time to look at all of them. These methods are checklists that guarantee coverage with few graphs: RED says whether a service is serving its users well, USE says which resource is the bottleneck. Used together, they turn "something is slow" into "this service's latency rose because this resource saturated".

### Mechanism: RED for services, USE for resources

```text
 RED, per service and per endpoint (and per dependency, from the caller's side)
   checkout POST /orders     Rate 420/s    Errors 0.4%    Duration p50 45 ms, p99 380 ms
   caller's view of payments  Rate 400/s   Errors 2.1%    Duration p99 1.6 s     ← the anomaly

 USE, per resource
   resource             utilisation               saturation                     errors
   CPU (per instance)   78% (cgroup quota used)   run-queue length; throttled    -
                                                  periods (Chapter 3, Section 3.12)
   memory               85% of limit              major page faults, GC time      OOM kills
   DB connection pool   10 / 10 in use            48 requests waiting to borrow   acquire timeouts
   worker threads       200 / 200 busy            accept and executor queue depth rejections
   disk                 %util 95                  I/O queue depth (Chapter 4)     I/O errors
   network interface    throughput vs capacity    drops, retransmits (Chapter 6)  errors
   downstream API       calls in flight vs limit  bulkhead waiters                timeouts
```

The two methods meet at the boundary: a dependency is a resource from the caller's point of view (USE: how many permits in use, how many waiting) and a service from its own (RED). Saturation is the signal people most often lack: utilisation says "busy", saturation says "busy and work is waiting", which is what raises latency (Chapter 11, Section 11.8).

### Concrete: the method as a diagnosis procedure

```text
 1. RED at the edge: which endpoints are affected? (one route, all routes, one region?)
 2. RED for each dependency, from the affected service's side: which call's duration or errors
    changed at the same time?
 3. follow the slowest dependency: repeat steps 1–2 inside it
 4. at the service where the slowness ORIGINATES (its own time grows, not its dependencies'):
    USE for each of its resources: which one is saturated?
 5. confirm with a trace (where the time goes) and a profile or thread dump (what the
    saturated resource is doing); then logs for the specific cause
 6. check "what changed?": deploys, configuration versions, traffic mix, data growth
```

### Guarantees and non-guarantees

RED and USE guarantee coverage of the most important service and resource indicators with a small number of metrics. They do not tell you why a resource is saturated (profiles, traces and logs do), and they only cover resources someone has instrumented: an unmeasured lock or an unmonitored pool remains invisible.

### Failure and diagnosis

- **Missing saturation metrics.** Evidence: latency high, every utilisation graph moderate, nothing explains it. Cause: an unmeasured queue (a lock, a pool, a semaphore, an executor). Fix: instrument waiters and queue time for every bounded resource.
- **Monitoring the wrong side.** Evidence: a dependency's own dashboards are green while callers see timeouts (network, DNS, connection pools on the caller's side). Fix: RED for each dependency measured by the caller.

### Backend connection

Give every service a RED dashboard per endpoint and per dependency, and every resource a USE row, with saturation measured explicitly. Walk them in order during incidents: RED to find where, USE to find what, then traces, profiles and logs to find why.

---

## 26.11 SLIs, SLOs, SLAs, Error Budgets and Alerting

### Definitions

- A **service-level indicator** (SLI) is a measured quantity that reflects what users experience, usually expressed as the proportion of good events: "the fraction of checkout requests that succeed in under 500 ms".
- A **service-level objective** (SLO) is a target for an SLI over a time window: "99.9% of checkout requests succeed in under 500 ms, measured over a rolling 30 days".
- A **service-level agreement** (SLA) is a contract with customers that states consequences (refunds, credits) if a level is not met. SLAs are usually looser than internal SLOs, so that the team reacts long before the contract is broken.
- An **error budget** is the amount of unreliability an SLO allows: for 99.9%, the budget is 0.1% of events in the window.
- The **burn rate** is how fast the budget is being consumed relative to an even spend: a burn rate of 1 uses the whole budget in exactly one window; a burn rate of 10 uses it in a tenth of the window.
- An alert on a **symptom** fires on something users experience (error rate, latency, SLO burn); an alert on a **cause** fires on something that might lead to it (high CPU, a full disk, a failed instance).

### Why it exists

"As reliable as possible" is not a goal: every nine costs more (Chapter 24, Section 24.11), and a team that spends all its time on reliability ships nothing, while a team that ignores it ships into outages. An SLO turns reliability into a number that product and engineering agree on, and the error budget turns that number into a decision rule: while budget remains, ship; when it is spent, slow down and fix reliability. It also defines what deserves a page: anything that threatens the SLO, and nothing else.

### Mechanism: choosing SLIs

Good SLIs measure the user's experience as directly as possible, at a point close to the user (the load balancer or the client, rather than deep inside a service):

| User expectation | SLI (good events / valid events) |
|---|---|
| the service answers | requests that did not fail with a 5xx or time out / all requests |
| it answers quickly | requests served faster than a threshold (for example 300 ms) / all requests |
| asynchronous work completes promptly | messages processed within 5 minutes of being produced / all messages |
| data is correct and fresh | read-model records less than 1 minute behind the source / sampled records |
| the business flow works | checkouts that reached "confirmed" / checkouts started (excluding user abandonment) |

Expressing latency as "the proportion of requests faster than a threshold" rather than "p99 below X" makes it countable, so it can have a budget and burn rates, and it is easy to compute from histograms (Section 26.9).

### Concrete: an SLO and its error budget, worked

```text
 SLI:  proportion of POST /checkout requests that return a non-5xx response in < 500 ms,
       measured at the load balancer
 SLO:  99.9% over a rolling 30 days
 traffic: an average of 20 requests per second (peaks of 40)

 requests per 30 days     = 20 × 86,400 × 30 = 51,840,000
 error budget (0.1%)      = 51,840 bad requests per 30 days
 as time at total outage  = 0.1% × 30 days = 43.2 minutes (Chapter 24's nines table)

 an incident: 25 minutes during which 4% of requests fail (at peak, 40 requests/s)
   bad requests = 40 × 1,500 s × 0.04 = 2,400  → 2,400 / 51,840 = 4.6% of the monthly budget
 the same incident at a total failure (100%):
   bad requests = 40 × 1,500 = 60,000           → 116% of the budget: SLO missed for the month

 burn rate during the 4% incident: actual bad ratio 4% / budget ratio 0.1% = 40
   → at that rate the whole 30-day budget would be gone in 30 days / 40 = 18 hours
```

### Mechanism: alerting on burn rate

An alert should page a human when the SLO is threatened soon enough to matter, and not otherwise. Alerting on "error rate above 0.1% for 5 minutes" pages for brief blips that spend almost no budget; alerting on "budget below 50%" pages too late. **Burn-rate alerts** with two windows, as described in Google's SRE workbook, balance the two:

```text
 page    if burn rate > 14.4 over the last 1 hour AND over the last 5 minutes
         (14.4 × 1 h / 720 h = 2% of a 30-day budget spent in one hour; the 5-minute window
          makes the alert stop quickly once the problem is fixed)
 page    if burn rate > 6 over 6 hours AND over 30 minutes       (5% of the budget in 6 hours)
 ticket  if burn rate > 1 over 3 days AND over 6 hours           (on track to miss the SLO)
```

### Mechanism: symptoms versus causes, and alert fatigue

Page on **symptoms**: SLO burn, error rates and latency users see, flows that stopped (no orders in 10 minutes). Do not page on **causes** that may or may not hurt users: a CPU at 90%, one instance down in a redundant group, a disk at 80%. Cause metrics belong on dashboards (for diagnosis) and in tickets (for capacity work), unless they predict certain user impact soon (a disk that will be full in two hours; a certificate expiring in three days).

**Alert fatigue** is what happens when pages are frequent and often not actionable: people learn to ignore them, and the real one is missed. Each paging alert should be urgent, actionable and linked to a **runbook** (the first steps to take). Review every page: if it required no action, change or delete the alert.

### Mechanism: the error-budget policy

The budget becomes useful when the team agrees in advance what happens as it is spent:

```text
 budget remaining > 50%     ship normally; run experiments; chaos tests (Chapter 24, Section 24.12)
 budget 0–50%               ship carefully; prioritise reliability items from recent postmortems
 budget exhausted           freeze risky releases (security fixes excepted); the team works on
                            the causes of the budget spend until the SLO is back on track
```

### Guarantees and non-guarantees

An SLO with burn-rate alerting guarantees that sustained threats to the user experience page someone, and that brief blips do not. It does not guarantee that the SLI measures what users actually care about (a perfectly met SLO on the wrong SLI is a silent failure), that the target is right, or that alerts catch failures the SLI does not cover.

### Failure and diagnosis

- **Pages nobody acts on.** Evidence: a high share of pages resolved with "no action". Fix: delete cause-based pages; move to burn-rate alerts; runbooks.
- **Users unhappy, SLO green.** Evidence: support tickets about failures the SLI does not count (wrong data, a broken flow returning 200). Fix: SLIs closer to the user and to business outcomes.
- **An SLO nobody uses.** Evidence: no error-budget policy; releases unaffected by budget state. Fix: agree the policy with product owners.

### Backend connection

Define a few SLIs per user-facing flow as proportions of good events measured near the user, set SLOs that product and engineering agree on, page on fast and slow burn rates, and keep cause metrics for dashboards. Write the error-budget policy down, so that reliability work is triggered by evidence rather than by the loudest voice after an outage.

---

## 26.12 Profiling in Production

### Definitions

- A **profiler** measures where a program spends a resource. A **sampling CPU profiler** interrupts the program at a fixed frequency (for example 100 times per second per thread), records the current call stack, and counts how often each stack appears; frequency of appearance estimates the share of CPU time.
- **On-CPU** profiling shows where threads use CPU. **Off-CPU** profiling shows where threads wait (on locks, I/O, sleep), which is where latency often hides.
- A **flame graph** (Brendan Gregg) visualises sampled stacks: each box is a function, its width proportional to the number of samples containing it, stacked by call depth.
- A **heap profile** shows which objects (and which allocation sites) hold memory now; an **allocation profile** shows which code allocates the most memory over time (and so drives garbage collection, Chapter 2, Section 2.8).
- **Continuous profiling** runs a low-overhead sampling profiler on production instances all the time and stores the profiles, so that "what was the CPU doing at 14:02 yesterday?" has an answer.

### Why it exists

Metrics say a service is CPU-bound; traces say a span is slow inside one service; neither says which code is responsible. A profile does, for the real workload, the real data and the real runtime: things that a laptop benchmark never reproduces. Sampling profilers are cheap enough (often around 1% overhead) to run in production, which is where the problem is.

### Concrete: reading a CPU flame graph

```text
 ┌───────────────────────────────────────────────────────────────────────────────────────────┐
 │                                     all samples (100%)                                    │
 ├─────────────────────────────────────────────────────────────────┬──────────────┬──────────┤
 │                        http_worker (82%)                         │ gc_threads   │ other    │
 ├──────────┬─────────────────────────────────────────────┬────────┤ (12%)        │ (6%)     │
 │ parse    │            OrderHandler.create (61%)        │ log    │              │          │
 │ (8%)     ├──────────┬───────────────────────┬──────────┤ (13%)  │              │          │
 │          │validate  │ PriceService.total    │ db.save  │        │              │          │
 │          │ (5%)     │ (44%)                 │ (12%)    │        │              │          │
 │          │          ├──────────────────────┬┤          │        │              │          │
 │          │          │ regex.compile (39%)  ││          │        │              │          │
 └──────────┴──────────┴──────────────────────┴┴──────────┴────────┴──────────────┴──────────┘
   (drawn root at the top here; many tools draw the root at the bottom and grow upwards)
```

How to read it:

1. **Width is the share of samples**, which estimates the share of CPU time spent in that function **and everything it called**. Here `OrderHandler.create` and its callees account for 61% of all CPU.
2. **Depth is the call stack**: a box's children are the functions it called.
3. **The horizontal order is not time.** Boxes at the same level are usually sorted alphabetically or merged; left-to-right says nothing about when things happened.
4. **Look for wide boxes with little or nothing above them**: that is **self time**, the code actually using the CPU. Here `regex.compile` is 39% of all CPU, inside the price calculation: a regular expression compiled on every call instead of once. Removing that one line would cut CPU by more than a third.
5. **Check the expected costs**: logging at 13% and garbage collection at 12% are worth questioning (excessive log formatting; allocation pressure, which an allocation profile would explain).
6. **Compare two profiles** (before and after a deploy, healthy versus slow period): a differential flame graph colours what grew and what shrank, which answers "what changed?" in code terms.

### Mechanism: the other profiles

- **Off-CPU and lock profiles** answer "why is this request slow if the CPU is idle?": time blocked on a lock, a connection pool, a socket read, a disk. Thread dumps taken a few seconds apart (Chapter 5, Section 5.14; Chapter 11, Section 11.10) are the poor man's version: a sample of what every thread is waiting on.
- **Heap profiles** find memory leaks: compare two heap snapshots an hour apart and look for object types and allocation sites that grow without bound (an unbounded cache, a listener list that is never cleaned).
- **Allocation profiles** find garbage-collection pressure: code that allocates gigabytes per minute of short-lived objects drives frequent collections and longer pauses, which shows in tail latency.
- **System-wide profiling** with kernel facilities (such as eBPF-based profilers on Linux) samples all processes, including the kernel, without changing the application, which helps when the time is spent in system calls, the network stack or another process.

> **Other runtimes:** The JVM's Flight Recorder and async-profiler, Go's built-in `pprof` endpoints, Python's sampling profilers such as py-spy, and Linux `perf` for native code all produce stack samples that can be drawn as flame graphs. Interpreted and JIT-compiled runtimes need profilers that understand their stacks; a generic native profiler may show only the interpreter's own functions.

### Guarantees and non-guarantees

A sampling profiler guarantees a statistically representative picture of where CPU (or allocation, or waiting) went during the sampling period, at low cost. It does not show rare, short events reliably (they may never be sampled), does not show time spent waiting unless it profiles off-CPU, and depends on symbols and stack information to name functions correctly.

### Failure and diagnosis

- **A CPU-bound service with no obvious cause.** Evidence: CPU saturated in USE metrics; traces show time inside the service's own span. Tool: a CPU flame graph during the busy period; look for wide self-time boxes. Diagnosis Lab 1 and Lab 3 use it.
- **Latency with idle CPU.** Tool: off-CPU profile or thread dumps; look for threads blocked on the same lock or pool.
- **Memory growing until OOM kills.** Tool: heap snapshots over time; growth by type and allocation site.

### Backend connection

Make profiling a routine production tool: continuous, low-overhead CPU and allocation profiling, plus the ability to take heap snapshots and thread dumps on demand. Read flame graphs by width and self time, compare profiles before and after changes, and reach for off-CPU views whenever latency rises without CPU.

---

## 26.13 Incident Response and Postmortems

### Definitions

- An **incident** is an unplanned event that degrades a service for its users (or threatens to), requiring a coordinated response.
- **Triage** is quickly assessing an incident's impact (who is affected, how badly) and urgency, to decide the response.
- **Mitigation** is reducing or stopping user impact, by any safe means, before the cause is understood. **Resolution** is restoring normal service. **Root-cause analysis** is understanding why it happened.
- An **incident commander** (IC) coordinates the response: assigns tasks, decides, and keeps the overview, while others investigate and communicate.
- A **postmortem** (incident review) is a written analysis after the incident: what happened, its impact, the timeline, the causes and contributing factors, and actions to prevent recurrence. A **blameless** postmortem examines systems and decisions, not individual fault, on the principle that people acted reasonably on the information they had.

### Why it exists

During an outage, every minute costs users, money and trust, and the natural instinct (everyone debugging the same thing, nobody deciding, nobody updating customers) wastes those minutes. After the outage, the natural instinct (find who made the mistake) guarantees that the next mistake is hidden rather than reported. Incident practice structures both.

### Mechanism: the response, step by step

```text
 1. DETECT     an SLO burn alert pages the on-call engineer (Section 26.11), or users report
 2. DECLARE    open an incident: a channel, an incident commander, a severity
               (for example SEV1: major user impact, all hands; SEV3: minor, business hours)
 3. TRIAGE     what is broken, for whom, since when? RED at the edge (Section 26.10);
               "what changed?": deploys, configuration versions, traffic, dependencies
 4. MITIGATE   stop the bleeding BEFORE understanding why:
                 roll back the last deploy or configuration change (Section 26.4)
                 turn off a feature with its kill switch (Chapter 24, Section 24.9)
                 shed load, fail over, add capacity, drain a bad zone or instance
               the question is "what is the fastest safe action that reduces impact?",
               not "what is the root cause?"
 5. COMMUNICATE status updates to stakeholders and users at a fixed rhythm (every 30 min),
               even when there is nothing new
 6. RESOLVE    confirm user-visible recovery (SLIs back to normal), then stand down
 7. REVIEW     blameless postmortem within days; action items with owners and dates
```

Mitigation before root cause is the core habit. A rollback of the deploy from 40 minutes ago takes two minutes and is correct in a large share of incidents, because most incidents are caused by changes; investigating first can take an hour of user impact. The evidence needed for root-cause analysis (logs, traces, a heap dump, the bad version's image) should be preserved while mitigating, but mitigation does not wait for it.

### Mechanism: diagnosing under pressure

The diagnosis procedure of this book, applied to production with the signals of this chapter:

```text
 symptom (alert, SLI)  →  WHERE: RED per service and dependency; which endpoints, which zone,
                          which version, which customers? (labels on metrics; Section 26.10)
                       →  WHAT CHANGED: deploys, configuration, traffic mix, data, dependencies
                       →  WHICH RESOURCE: USE for the service where time originates; traces to
                          see which span grew (Section 26.8)
                       →  WHY: profiles (CPU, off-CPU, heap), thread dumps, logs for the slow or
                          failed traces by trace ID; earlier chapters' diagnosis sections for the
                          layer involved (OS, network, database, cache, broker)
                       →  HYPOTHESIS → EVIDENCE that would confirm or refute it → act
```

Each hypothesis should come with the evidence that would separate it from the others, exactly as in the Diagnosis Labs throughout this book; the failure mode to avoid is changing things at random and watching graphs.

### Concrete: a postmortem outline

```text
 TITLE      Checkout errors after payment-client configuration change, 2026-10-24
 SUMMARY    For 38 minutes, 22% of checkouts failed. A configuration change reduced the
            payment-provider timeout from 2,000 ms to 200 ms (a unit error), which is below the
            provider's p50 latency.
 IMPACT     ~9,400 failed checkouts; 31% of the monthly error budget; no data loss; no duplicate
            charges (idempotency keys held).
 TIMELINE   14:02 config v413 rolled to all instances at once
            14:05 burn-rate page; 14:09 incident declared; 14:21 traces show payment calls
            failing at exactly 200 ms; 14:36 rollback to v412; 14:40 SLI recovered
 CAUSES     the value's unit was ambiguous ("timeout: 200", intended seconds × 1000?);
            no range validation for the key; configuration rolled to 100% at once
 WENT WELL  idempotency prevented duplicates; the burn-rate alert fired in 3 minutes
 WENT BADLY 27 minutes from page to diagnosis: no "config version" annotation on dashboards
 ACTIONS    unit-suffixed keys (timeout_ms) [owner, date]; range validation in CI [owner, date];
            staged configuration rollout [owner, date]; config version on dashboards [owner, date]
```

Note what is absent: the name of the engineer who made the change. The questions are why the system let an easy mistake reach every instance, and why it took 27 minutes to see it.

### Guarantees and non-guarantees

A practised incident process guarantees clear roles, fast mitigation attempts, regular communication and a written record. It does not guarantee a quick diagnosis (that depends on observability), that mitigation is possible without a deploy (that depends on kill switches and rollback paths built in advance), or that postmortem actions are completed unless they are tracked like any other work.

### Failure and diagnosis

- **Long time to mitigate.** Evidence: postmortem timelines with long gaps between detection and the first mitigating action. Causes: debugging before rolling back; no rollback path; unclear ownership. Fix: "roll back first" as the default; incident commanders.
- **Repeated incidents with the same cause.** Evidence: postmortems listing the same contributing factors. Fix: track action items to completion; review them in planning.
- **Hidden mistakes.** Evidence: incidents discovered late because nobody reported the change that caused them. Fix: blameless culture, made visible by how postmortems are written and discussed.

### Backend connection

Build the mitigation tools before you need them: fast rollback, kill switches, staged configuration, capacity controls. Make "what changed?" answerable in seconds with deploy and configuration annotations on dashboards. Run incidents with a commander and a rhythm of updates, mitigate before diagnosing, and write blameless postmortems whose action items change the system, not the people.

---

## 26.14 Placement-Round Topics: Intrusion Detection and Malware

> **Placement-round focus:** This section is revision material for information-security questions in campus placement written tests and university security courses. It is not part of the main flow. What carries into backend work: security monitoring is observability applied to attacks (Section 26.6), and a backend's own defences (Chapters 8, 9, 19 and 20) matter more than any perimeter device.

### Intrusion detection and prevention

An **intrusion detection system** (IDS) monitors network traffic or host activity for signs of attack and raises alerts. An **intrusion prevention system** (IPS) sits inline in the traffic path and can also **block** what it detects.

- A **network-based** IDS (NIDS) inspects traffic on a network segment; a **host-based** IDS (HIDS) watches one machine: system calls, file integrity, logins, processes.
- **Signature-based** detection matches known attack patterns (a byte sequence of a known exploit). It is precise for known attacks and blind to new ones. **Anomaly-based** detection learns normal behaviour and flags deviations. It can catch new attacks but produces more **false positives** (benign activity flagged) and needs tuning; a **false negative** is an attack that is missed.
- Encrypted traffic (Chapter 8) hides payloads from network IDS unless TLS is terminated before it, which is one reason host-based and application-level detection matter.
- A **honeypot** is a decoy system that no legitimate user should touch, so any access to it is suspicious.
- A **SIEM** collects and correlates logs and alerts from many sources (firewalls, IDS, servers, applications) to detect attacks that no single source reveals; a **security operations centre** (SOC) is the team that watches it.

| Control | Works at | Decides by | Blocks? |
|---|---|---|---|
| firewall (Chapter 6, Section 6.5) | network and transport layers | addresses, ports, connection state | yes |
| WAF (Chapter 10, Section 10.1) | HTTP | request patterns (injection, XSS signatures) | yes |
| IDS | network or host | signatures or anomalies | no, alerts |
| IPS | inline network or host | signatures or anomalies | yes |

### Malware taxonomy

**Malware** is software designed to harm or exploit systems. The categories tests ask for:

| Type | Defining behaviour |
|---|---|
| **virus** | attaches itself to a host program or file and spreads when that host is run or shared; needs a user action |
| **worm** | self-replicating program that spreads across networks by itself, exploiting vulnerabilities, without user action |
| **trojan** (trojan horse) | appears legitimate or useful, but performs hidden malicious actions; does not self-replicate |
| **ransomware** | encrypts the victim's data (or threatens to publish it) and demands payment for the key; immutable backups are the main defence (Chapter 15, Section 15.9) |
| **spyware** / **keylogger** | secretly collects information: browsing, credentials, keystrokes |
| **rootkit** | hides its own presence (and other malware) by modifying the operating system, often at kernel level |
| **backdoor** | a hidden way to bypass normal authentication |
| **botnet** | a network of compromised machines ("bots") controlled by an attacker through a command-and-control (C2) server; used for DDoS (Chapter 10, Section 10.1), spam and credential stuffing (Chapter 20, Section 20.2) |
| **logic bomb** | code that triggers a malicious action when a condition is met (a date, an event) |
| **adware** | displays unwanted advertising; often bundled with other software |

A **denial-of-service** (DoS) attack exhausts a target's resources from one source; a **distributed** DoS (DDoS) uses many sources, typically a botnet, so that blocking single addresses does not help (Chapter 6 covered SYN floods; Chapter 10 the edge defences).

### Drill for this section

**C1. Compare an IDS with an IPS, and signature-based with anomaly-based detection.**
A strong answer must mention:
- IDS monitors and alerts; IPS sits inline and blocks; network-based versus host-based.
- Signature-based: precise for known attacks, blind to new ones; anomaly-based: can detect new attacks, more false positives, needs tuning.
- Encrypted traffic limits network inspection; SIEM correlates alerts across sources.
*Common weak answer:* "An IDS is a firewall." A firewall filters by addresses and ports; an IDS analyses content or behaviour and, unlike an IPS, does not block.

**C2. Distinguish a virus, a worm and a trojan. What is a botnet used for?**
A strong answer must mention:
- Virus: attaches to a host program, spreads with user action; worm: self-replicates across networks without user action; trojan: disguised as legitimate, does not replicate.
- Botnet: compromised machines controlled through command-and-control; used for DDoS, spam, credential stuffing.
- Ransomware's main defence is restorable, immutable backups.
*Common weak answer:* using "virus" for all malware.

---

## Chapter Summary

Running a backend starts with a disciplined path from commit to production: one immutable image per commit, scanned, signed and described by an SBOM, **built once and promoted unchanged** through environments that differ only in configuration. Services scale **horizontally** when they are **stateless**, so that any instance can serve any request and be killed at any time; **autoscaling** follows a signal (concurrency or queue lag rather than CPU for I/O-bound services) with minutes of **lag**, so headroom and shedding cover the gap, and **capacity planning** with Little's Law runs down to the scarcest downstream resource. An **orchestrator** converges actual state to **desired state** through reconciliation loops, schedules instances by **requests**, enforces **limits** through Chapter 4's cgroups (CPU limits throttle; memory limits OOM-kill), keeps unready instances out of **service discovery**, and acts on probes that must stay local: a liveness probe that depends on the database restarts healthy instances in a loop.

**Deployments** limit the blast radius of change: **rolling** replaces batches, **blue–green** switches everything and can switch back in seconds, **canary** exposes a small share of traffic and rolls back automatically on SLO regression; feature flags separate deploy from release, and every pair of adjacent versions must work with the same database schema (**expand and contract**). Configuration is versioned, validated and staged like code, with secrets resolved at run time.

**Observability** answers new questions from evidence. **Logs** are structured, levelled, correlated by trace and request IDs, and masked for personal data and secrets; audit events go to a protected stream watched by security monitoring. **Metrics** (counters, gauges, histograms) are cheap aggregates whose labels must stay low in **cardinality**. **Traces** follow one request through every service by propagated context, sampled so that slow and failed requests are kept. **Averages lie**: latency is a distribution, judged by **percentiles** computed from merged histograms, and **fan-out** turns backends' rare tails into requests' common ones (1 − 0.99^n). The **golden signals**, **RED** for services and **USE** for resources structure dashboards and diagnosis. **SLIs** measure good events near the user, **SLOs** set targets, **SLAs** add contracts, and **error budgets** with **burn-rate alerts** decide both when to page and when to slow down. **Flame graphs** show where CPU goes by width and self time; off-CPU and heap profiles explain waiting and memory. **Incidents** are run with a commander, **mitigation before root cause**, regular communication, and **blameless postmortems** that change systems rather than blame people.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Build once, signed image | Every environment runs the same, verified bits | Same configuration, data or traffic; absence of unknown vulnerabilities |
| Stateless instances | Any instance serves any request; instances are disposable | Downstream capacity for more instances |
| Autoscaling | Capacity eventually follows the signal | Capacity during scaling lag; that the signal tracks the bottleneck |
| Orchestrator reconciliation | Continuous convergence to declared state | That the declared state is correct; enough cluster capacity |
| CPU request / limit | Fair share under contention / a hard ceiling | Latency without throttling under a limit |
| Memory limit | Containment of one instance's memory | Survival: exceeding it means an OOM kill |
| Readiness probe (local) | Traffic only to ready instances | Detection of gray failures |
| Canary with automated analysis | A regression in compared metrics affects only the canary share | Detection of unmeasured or slow-developing problems |
| Blue–green | Fast, complete switch back while blue is kept | Rollback after incompatible schema changes |
| Expand-and-contract migrations | Adjacent versions work with every schema state | Speed: changes take several deploys |
| Structured, correlated logs | A specific event can be found and linked to its trace | That it was logged, delivered, or free of personal data without redaction |
| Metrics | Cheap continuous aggregates | Individual cases; safety with unbounded labels |
| Traces | A timeline of a sampled request across instrumented services | Unsampled requests; uninstrumented components |
| Percentiles from merged histograms | Service-wide distribution within bucket resolution | Which requests were slow; validity of averaged percentiles |
| SLO + burn-rate alerts | Pages for sustained threats to users, not for blips | That the SLI measures what users care about |
| Sampling profiler | Representative picture of CPU or allocation at low overhead | Rare short events; waiting time unless profiled off-CPU |
| Incident process | Roles, fast mitigation attempts, communication, a record | Fast diagnosis without observability; completed actions without tracking |

---

## Interview Drill

**Q1. What happens between merging a change and it running in production?**
A strong answer must mention:
- CI builds and tests; static, dependency and secret scans.
- One immutable image per commit, scanned, signed, with an SBOM; promoted unchanged through environments; only configuration differs.
- Progressive rollout (canary or rolling) with automatic rollback on SLO regression.
- Recording what runs where (digest, config version) for incidents.
*Common weak answer:* "We build it for production and deploy." Rebuilding per environment means production runs untested bits.

**Q2. Horizontal versus vertical scaling? Why must services be stateless to scale?**
A strong answer must mention:
- Vertical: bigger machine, simple, has a ceiling and a single point of failure; horizontal: more instances, no ceiling, needs divisible work.
- Statelessness: any instance serves any request; instances can be added, removed and replaced without losing state; state moves to shared stores.
- Leftover state that breaks scaling: in-memory sessions, local files, per-instance jobs and limits.
- Downstream limits (connections) multiply with instances.
*Common weak answer:* "Add more servers." Without statelessness, new servers help only new users and lose state when removed.

**Q3. How does autoscaling work, and why does it lag?**
A strong answer must mention:
- desired = ceil(current × signal / target); signals: CPU, concurrency, RPS, queue lag, schedule.
- Lag: metric windows, evaluation interval, provisioning, image pull, warm-up, slow start: often minutes.
- Headroom, scheduled scaling and shedding cover spikes; slower scale-in with cooldowns.
- CPU is a poor signal for I/O-bound services.
*Common weak answer:* "It adds instances instantly when CPU is high." It takes minutes, and CPU may not reflect the bottleneck.

**Q4. What does a container orchestrator do? What are desired state, scheduling and service discovery?**
A strong answer must mention:
- Declared desired state; controllers reconcile actual to desired continuously (replace failed instances, maintain counts, roll out changes).
- Scheduling places instances on nodes by resource requests and constraints (zone spreading).
- Service discovery: stable names or registries listing only ready instances; server-side versus client-side.
- Probes and graceful shutdown integrate with discovery.
*Common weak answer:* "It runs Docker containers." The essential idea is declarative convergence, not starting processes.

**Q5. What is the difference between liveness and readiness probes, and what goes wrong with a bad liveness probe?**
A strong answer must mention:
- Liveness failure → restart; readiness failure → removed from traffic, no restart; startup probe protects slow starts.
- Probes must check only the instance itself.
- A liveness probe that depends on the database restarts every instance when the database is slow: cold starts, connection storms, a restart loop that turns a slowdown into an outage.
*Common weak answer:* "Both check /health." Their actions differ, so their checks must differ.

**Q6. What are resource requests and limits, and how do they relate to cgroups and CPU throttling?**
A strong answer must mention:
- Requests: scheduling and guaranteed share (cpu.weight); limits: ceilings (cpu.max, memory.max).
- CPU limit → throttling for the rest of each period once the quota is used: tail latency at low average CPU; evidence in cpu.stat.
- Memory limit → OOM kill, exit 137; limits include non-heap memory and page cache.
- Runtimes must size heaps and thread pools from cgroup limits.
*Common weak answer:* "Limits just cap usage harmlessly." CPU limits pause the process; memory limits kill it.

**Q7. Rolling versus blue–green versus canary deployments?**
A strong answer must mention:
- Rolling: batches, small surge, minutes to roll back, mixed versions during rollout.
- Blue–green: full second environment, instant switch and switch back, double capacity, 100% exposure at the switch.
- Canary: small traffic share compared with the old version, stepwise increase, automatic rollback; needs version-labelled metrics and enough traffic.
- All require compatibility between adjacent versions and with the database.
*Common weak answer:* "Blue–green is always safest." It exposes all users at once and cannot roll back an incompatible migration.

**Q8. How do you deploy a database schema change without downtime?**
A strong answer must mention:
- Old and new code run together during rollouts and after rollbacks.
- Expand and contract: add the new structure, write both, backfill, switch reads, stop writing the old, drop it later.
- Every schema state compatible with the current and previous code version.
- Lock-safe migration techniques (Chapter 18).
*Common weak answer:* "Run the migration, then deploy." Instances running the old code fail during the rollout, and rollback breaks.

**Q9. Logs versus metrics versus traces? How does distributed tracing work?**
A strong answer must mention:
- Logs: discrete events with details; metrics: cheap numeric aggregates for alerting and trends; traces: one request's spans across services.
- Diagnosis chain: metrics alert, traces locate, logs explain.
- Tracing: spans with trace ID, span ID and parent ID; context propagated in headers (traceparent) and message headers; collected and assembled; sampled (head versus tail).
- Correlation: trace IDs in every log line.
*Common weak answer:* "They are three kinds of logs." Each has a different cost model and answers a different question.

**Q10. What is metric cardinality, and why can a label take down your monitoring?**
A strong answer must mention:
- Each distinct label combination is a separate time series; cardinality is the product of label value counts.
- Unbounded labels (user ID, raw URL, request ID) create millions or billions of series.
- Symptoms: metrics backend memory exhaustion, slow queries, alerting failures for every service.
- High-cardinality detail belongs in logs and traces; labels use bounded values.
*Common weak answer:* "More labels give more insight." Unbounded labels destroy the system that provides the insight.

**Q11. Why use p99 and not averages? How are percentiles computed from histograms?**
A strong answer must mention:
- Latency is a skewed distribution; the average describes no real request and hides the tail.
- Users making many requests almost surely meet the tail (1 − 0.99^100 ≈ 63%); fan-out amplifies backend tails.
- Percentiles from summed histogram buckets with interpolation; never average per-instance percentiles.
- Buckets dense around targets; coordinated omission distorts load tests.
*Common weak answer:* "p99 is just a stricter average." It is a point on the distribution, and it cannot be averaged.

**Q12. What are the golden signals, and the RED and USE methods?**
A strong answer must mention:
- Golden signals: latency, traffic, errors, saturation.
- RED per service and dependency: rate, errors, duration.
- USE per resource: utilisation, saturation (waiting work), errors.
- Procedure: RED to find where, USE to find which resource, then traces, profiles and logs for why.
*Common weak answer:* listing the acronyms without using them as a diagnosis procedure.

**Q13. What are SLIs, SLOs, SLAs and error budgets? How should you alert?**
A strong answer must mention:
- SLI: proportion of good events measured near the user; SLO: target over a window; SLA: contract with consequences, looser than the SLO.
- Error budget = 1 − SLO of events; burn rate; worked example (a 99.9% SLO over 51.84 million requests allows 51,840 bad requests).
- Page on symptoms with multiwindow burn-rate alerts; causes on dashboards; every page actionable with a runbook.
- Error-budget policy: ship freely with budget, slow down when it is spent.
*Common weak answer:* "Alert when CPU is above 80%." It pages for things users may not feel and misses things they do.

**Q14. How do you read a CPU flame graph?**
A strong answer must mention:
- Width is the share of samples (function plus callees); depth is the call stack; horizontal order is not time.
- Wide boxes with little above them are self time, the actual hot code.
- Compare profiles before and after a change; differential flame graphs.
- Off-CPU and heap profiles for waiting and memory.
*Common weak answer:* "The left side is what ran first." The x-axis is not a timeline.

**Q15. How do you debug a production latency spike, and how should an incident be run?**
A strong answer must mention:
- Declare, assign an incident commander, triage impact; mitigate first (roll back, kill switch, shed, fail over) while preserving evidence.
- Diagnose: RED to find where, "what changed?", USE for the originating service, traces for which span grew, profiles and logs for why; hypotheses with separating evidence.
- Communicate regularly; resolve when SLIs recover.
- Blameless postmortem with owned, tracked actions.
*Common weak answer:* "SSH into servers and look at logs." Without structure and mitigation first, impact lasts longer and evidence is missed.

**Q16. What are operations, DevOps and SRE? What kinds of load test are there?**
A strong answer must mention:
- Operations: deploying, running, observing and repairing software; DevOps: the same teams build and run their software, with automation instead of hand-offs; SRE: software engineering applied to operations, with SLOs, error budgets, automation and blameless reviews.
- Infrastructure as code: declared, reviewed, versioned infrastructure applied by a tool, like configuration.
- Load (expected peak), stress (find the breaking point), soak (leaks over time), spike (sudden surge); open-model generators to avoid coordinated omission; production-like data.
*Common weak answer:* "DevOps is a job title for the person who runs the servers." It is a way of working, not a separate team.

---

## Scenario Exercises

1. You are asked to define SLOs for a public REST API whose clients include a mobile app and partners. Propose two SLIs, their SLOs, the error budget in requests for a month at 30 requests per second average, and the alerting rules.
2. A service autoscales on CPU with a 60% target. During a marketing push, latency rises to 4 seconds while CPU stays at 35%, and no instances are added. Explain, and propose a better signal and other protections.
3. A team plans a blue–green deployment that renames the `orders.status` values from strings to integers in the same release. Explain what will break, both on the switch and on a rollback, and plan the change safely.
4. Your log bill tripled in a month, and a security scan found email addresses and bearer tokens in application logs. Plan the clean-up and the controls that prevent both problems from returning.
5. A dashboard shows the service's "p99 latency" as the average of the 20 instances' p99 values; it reads 180 ms, while users report frequent multi-second waits. Explain how this can be, and what the dashboard should show instead.
6. A canary receives 1% of traffic (about 10 requests per second) for 10 minutes. The stable version's error rate is 0.2%; the canary's is 0.3%. Should the rollout proceed? Reason about the numbers and propose a better canary design.
7. Read this incident timeline and write the "causes" and "actions" sections of a blameless postmortem: a deploy at 09:00 added a new database index build; at 09:05 checkout latency rose; the on-call engineer spent 40 minutes examining the payments service; at 09:50 someone noticed the deploy; at 09:55 the index build was cancelled and latency recovered.

---

## Diagnosis Lab

**Lab 1: From an alert to a root cause with the three signals**
- *Symptom:* At 16:10, the burn-rate alert for "order history page served in under 800 ms" pages: burn rate 22 over the last hour. Error rates are normal.
- *Hypotheses:* (a) a slow database; (b) a slow downstream service; (c) a code change in the orders service; (d) a traffic change.
- *Evidence (metrics):* RED at the edge shows only `GET /orders` affected, all zones, p99 from 450 ms to 2.9 s, p50 almost unchanged: only some requests are slow. Traffic is normal, ruling out (d). RED for the orders service's dependencies: database query duration unchanged, ruling out (a); calls to the inventory service: rate up 9× while traffic is flat, each call fast (p99 40 ms). The deploy annotation shows orders version `8e21b0` at 14:35, with the canary passing (its traffic had few customers with large order histories).
- *Evidence (traces):* Tail-sampled slow traces show a staircase: for customers with many orders, the orders service makes one inventory call **per order line**, sequentially: 60 calls × 40 ms ≈ 2.4 s. Traces from before 14:35 show a single batch call.
- *Evidence (logs):* Logs for those trace IDs show `event=inventory_lookup` repeated per line, emitted by a new "show live stock per item" feature in `8e21b0`.
- *Root cause:* (c): the new version replaced a batch call with per-item sequential calls, invisible for small histories (and in the canary), slow for large ones.
- *Fix (now):* Turn off the feature's flag (the deploy shipped it enabled), and the burn rate returns to normal within minutes. *Fix (lasting):* restore the batch call (or parallel calls with a bound), add a fan-out metric (inventory calls per request) and a test with a large history; canary analysis that includes per-endpoint p99 for heavy users.
- *Prevention:* Per-request fan-out tracked as a metric; features released behind flags off by default.

**Lab 2: A restart loop with exit code 137**
- *Symptom:* After a traffic increase, instances of the reporting service restart every few minutes; restart counts climb into the dozens.
- *Hypotheses:* (a) liveness probe failures; (b) OOM kills by the memory limit; (c) a crash bug.
- *Evidence:* The container's last termination reason is "OOMKilled", exit code 137, ruling out (a) and pointing away from (c) (no stack trace in the logs; the process was killed). The node's kernel log shows "Memory cgroup out of memory" for the container (Chapter 4, Section 4.10). The memory limit is 2 GiB; the runtime's heap maximum is configured as 100% of the container's memory, so heap alone may reach 2 GiB, and thread stacks, native buffers and metaspace push the total over the limit when traffic adds threads and buffers. Heap metrics show the heap itself near its maximum at the moment of each kill.
- *Root cause:* (b): a heap sized to the whole container limit, leaving no room for non-heap memory.
- *Fix:* Size the heap to about 70–75% of the limit (leaving room for non-heap memory, Chapter 3, Section 3.18), bound per-request buffers in the report generator, and raise the limit if profiles show genuine need. Check the memory request matches typical use, to avoid eviction under node pressure.
- *Prevention:* Alerts on container memory near the limit and on OOM kill events; runtime memory settings derived from the cgroup limit.

**Lab 3: Tail latency with low CPU (CPU throttling)**
- *Symptom:* The search API's p99 rose from 120 ms to 700 ms after it was moved to the cluster. Average CPU utilisation is 30% of the limit; the database and dependencies are healthy.
- *Hypotheses:* (a) noisy neighbours on the node; (b) CPU throttling by the limit; (c) garbage-collection pauses.
- *Evidence:* The container's `cpu.stat` shows `nr_throttled` in 40% of periods, and `throttled_usec` rising steadily (Chapter 3, Section 3.12; Chapter 4, Section 4.10). The CPU limit is 2 CPUs. The runtime's worker pool and garbage-collector threads are sized for the node's 32 cores, read from the host: a burst runs 32 threads in parallel, uses the 200 ms per-100 ms quota in about 6 ms of wall time, and then the whole container is paused for the remaining ~94 ms of the period. GC logs show short pauses, not long enough to explain 700 ms on their own, but parallel GC threads also consume the quota.
- *Root cause:* (b), amplified by thread pools sized from the host's CPU count rather than the cgroup limit.
- *Fix:* Configure the runtime's processor count and pools from the cgroup limit (container-aware settings), raise the CPU limit or remove it in favour of an adequate request for this latency-sensitive service, and re-run a load test watching throttling counters.
- *Prevention:* Throttling as a standard dashboard and alert; runtime configuration checked for container awareness.

**Lab 4: The monitoring outage caused by one label**
- *Symptom:* At 11:20, dashboards for all services stop updating and several alerts fire "no data". Services themselves appear healthy to users.
- *Hypotheses:* (a) a network problem between services and the metrics system; (b) the metrics system overloaded; (c) a scrape configuration change.
- *Evidence:* The metrics server's memory climbed from 40% to its limit between 10:50 and 11:20, then it restarted repeatedly. The count of active series jumped from 4 million to 38 million at 10:50. The top metric by series count is `http_request_duration_seconds_bucket` from the catalogue service, whose version deployed at 10:48 labels requests with the raw path (`/products/81723`) instead of the route template (`/products/{id}`): 2.6 million product IDs × buckets.
- *Root cause:* (b), caused by an unbounded label introduced by a deploy: a cardinality explosion that took down monitoring for every service.
- *Fix (now):* Drop the label at ingestion with a relabelling rule (or roll back the catalogue deploy), restart the metrics server, and verify alerting is back. *Fix (lasting):* route templates in the instrumentation library by default; per-service series limits at ingestion; a CI check that rejects metrics with unbounded labels; monitoring of the monitoring system itself (series count, memory) from an independent place.
- *Prevention:* Alerting that pages when alerting data goes missing ("dead man's switch").

---

## Solution Sketches

1. SLIs: availability (proportion of requests without 5xx or timeout, measured at the edge) and latency (proportion served under 300 ms for reads, 800 ms for writes). SLOs: 99.9% and 99%, rolling 30 days. Budget at 30 requests/s: 30 × 86,400 × 30 = 77.76 million requests; 0.1% = 77,760 failed requests; 1% = 777,600 slow requests. Alerts: page on burn rate 14.4 over 1 h and 5 min, and 6 over 6 h and 30 min; ticket on 1 over 3 days. Partners may need an SLA, set looser than the SLO (for example 99.5%), and the mobile app's SLIs may be better measured client-side as well.
2. The service is I/O-bound: threads wait on a dependency or the database, latency rises, and CPU stays low, so the CPU-based autoscaler sees no reason to scale (and more instances might not help if the dependency is the bottleneck). Better signal: in-flight requests per instance (concurrency) or latency-aware scaling; scheduled scale-up before known pushes; plus the protections of Chapter 24: timeouts, bulkheads and admission control so that latency stays bounded while capacity is added, and checking the downstream's capacity before scaling callers.
3. At the switch, green writes integers while any remaining blue process (and any consumer, cache or report reading the column) expects strings; queries comparing to string values break; after a rollback, blue cannot read the integers green wrote. Safe plan (expand and contract): add a new `status_code` column; deploy code that writes both and reads the old; backfill; deploy code that reads the new and still writes both; verify; stop writing the old; drop it later. Each step's code works with each schema state, so blue–green switches and rollbacks are safe at every step.
4. Clean-up: find and remove or redact the affected logs per retention and privacy policy, rotate exposed tokens and invalidate sessions (Chapter 20, Section 20.11), and identify the code paths. Controls: redaction in the logging library for sensitive fields and patterns; a structured-logging standard that forbids logging request bodies; secret scanning on log samples; volume: sample high-volume success logs, lower levels on hot paths, counters instead of log lines for counts, shorter retention for debug logs, and per-service log volume budgets with alerts.
5. Averaging per-instance p99s is mathematically invalid: instances with few requests and fast responses pull the average down, and the true service-wide p99 depends on the merged distribution. Users also make many requests, so they meet the tail often. The dashboard should compute the p99 (and p99.9) from summed histogram buckets across instances, and show the proportion of requests slower than the latency target, per endpoint.
6. 10 requests/s × 600 s = 6,000 requests: expected errors at 0.2% are 12, observed at 0.3% are 18; with counts this small, the random variation (about √12 ≈ 3.5) makes a difference of 6 inconclusive (under two standard deviations). Proceeding or rolling back on this alone is guesswork. Better: run the step longer or at a larger share to gather enough requests, compare against a baseline group receiving the same traffic at the same time, use a statistical test with predefined thresholds, and include latency and key business metrics, not only errors.
7. Causes: the deploy combined a schema change (an index build that consumed I/O and locks) with application code, without being identified as risky; the index was built without the online technique (Chapter 18, Section 18.15); dashboards had no deploy annotations, so the change was not the first thing checked; the database's I/O saturation was not on the on-call dashboard. Actions: build indexes with non-blocking methods and in separate, scheduled migration steps; deploy and migration annotations on dashboards; "what changed?" as the first triage step in the runbook; USE metrics for the database on the service dashboard; each with an owner and date. No individual is named.

---

**END OF CHAPTER 26**
