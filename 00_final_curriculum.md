# The Backend Engineering Handbook: Foundations & Internals
## Final Curriculum v2 (27 chapters): LOCKED

> **Status:** LOCKED (approved 2026-10-14). This file is the single source of truth for chapter scope.
> The superseded 20-chapter v1 and its drafts are in `archive/v1/`.

---

## 0. What changed from v1 and why

| # | Gap in v1 | Why it matters for the goal | Fix in v2 |
|---|---|---|---|
| 1 | **The front matter assumed Core Java mastery** (JMM, locks, thread pools, GC, proxies), and so did `SYSTEM_PROMPT.md` and the Ch 8 prompt | The goal says *do not assume Core Java*. Concurrency primitives, memory visibility, GC and proxies are interview CS topics in their own right, so the book must teach them | Drop the "Contract with Core Java". Add **Ch 2 (Hardware & Runtimes)** and **Ch 5 (Concurrency Fundamentals)**, taught without tying them to one language |
| 2 | **Java/Spring-centric prompts** (Servlet API, `@Async`, `SecurityContextHolder`, MDC, HikariCP as definitions) | The goal says concepts must be language- and framework-independent | Mechanism first, then pseudocode. Java appears in a **"Java lens"** box and Spring only in a **"Framework lens"** box, with a Python/Node contrast where it helps |
| 3 | **One OS chapter (25–30 pp)** carrying processes, threads, scheduling, VM, files, page cache, I/O models, sockets, limits, sync and deadlock | Too much for one chapter if the reader starts from zero. Classic interview items were missing: PCB & process states, IPC, page replacement (LRU/FIFO/Belady), TLB, multi-level page tables, copy-on-write, mmap, user vs kernel threads, and deadlock prevention/avoidance/detection (Banker's) | Split into **Ch 3 (Processes, Threads, Scheduling, Memory)** and **Ch 4 (Files, I/O, IPC, Isolation)**. Synchronization moves to Ch 5 |
| 4 | **VMs vs containers, namespaces and images were out of scope** | "VM vs container" is a standard interview question, and you can't *operate* a backend without knowing what a container is at OS level | Ch 4 covers namespaces + cgroups + images as OS mechanisms. Ch 26 covers deployment and scaling **concepts**. Docker/K8s product tutorials stay out |
| 5 | **Ch 5 HTTPS had no cryptography foundation** | You can't explain TLS, JWT signatures, password hashing or HMAC webhooks from understanding without symmetric/asymmetric crypto, hashes, MACs, signatures and key exchange | **Ch 8: Cryptography Essentials + TLS** |
| 6 | **Ch 12 overloaded**: 22 required items covering architecture, storage, WAL, ACID, isolation, MVCC, locking, scans and EXPLAIN in one chapter, and its scan/EXPLAIN material overlapped Ch 13 | Isolation levels and MVCC are the single most-asked DB interview area and need room. LSM trees and column stores were missing entirely | Split into **Ch 15 Storage & Durability**, **Ch 16 Indexing & Query Execution** and **Ch 17 Transactions & Concurrency Control** |
| 7 | **Distributed data was thin**: Ch 17 had 18–22 pp for replication + CAP + all NoSQL selection. **No sharding, consistent hashing, hot keys, clocks, consensus/leader election or 2PC** | These are core interview and system-design foundations, and "multiple users / multiple nodes" failures come from them | **Ch 22 Replication & Consistency** and **Ch 23 Partitioning, Coordination & Choosing Data Stores** |
| 8 | **Ch 19 overloaded**: resilience + messaging + observability in 22–28 pp (v1 itself warned it might overflow) | Each is its own body of mechanisms | **Ch 24 Failure & Resilience**, **Ch 25 Async Work & Messaging**, **Ch 26 Production: Deployment, Scaling, Observability** |
| 9 | **Security was scattered** with no threat-model spine. Rate-limiting algorithms, security headers/CSP, secrets management and mTLS were thin | Interviewers ask "how do you secure an API?" and expect a structured answer | Each attack keeps its v1 home. Ch 20 ends with an **"Application Security Synthesis"** that maps the OWASP Top 10 to chapters. Rate-limit algorithms go in Ch 24 |
| 10 | **No interview-drill structure.** The prompts asked only for "review questions" | The goal is to say definitions *through understanding* and to reason about unfamiliar scenarios | Every chapter ends with an **Interview Drill** (question + what a strong answer must contain + common wrong answers) and **Scenario Exercises** |
| 11 | **Drafts are far below target length.** Ch 3 Networking ≈ 4.3k words and Ch 4 HTTP ≈ 4.6k words against a 25–32 page target (≈ 11–14k words). Ch 2 OS ≈ 7.6k | At that density the "mechanism" step gets skipped | Targets are now set in **words** (1 page ≈ 450 words) and enforced per chapter |
| 12 | Prompts reference an "old PDF" and Gemini/AI Studio specifics | That ties quality to a source you're replacing | The new prompts are self-contained. The curriculum is the source of truth |

**Size:** 27 chapters, ≈ 650–720 pages (≈ 290–320k words). The growth is the honest cost of "assume I know nothing outside basic programming".

---

## 1. Reader contract (replaces v1 front matter)

- **Assumed:** you can program in *some* language (variables, functions, loops, objects, basic collections) and you know basic data structures (array, list, hash map, tree) at the level of what they are.
- **Not assumed:** OS, networking, databases, security, distributed systems, web architecture, concurrency, JVM internals or any framework.
- **Language policy:** concepts are defined without reference to a language. Examples use pseudocode first. **Java** is the primary illustration language. **Python/Node** contrasts appear where execution models differ (GIL, event loop). **Spring Boot** appears only to show how a framework implements a mechanism the chapter has already explained.
- **Out of scope:** DSA problem solving, framework tutorials, Docker/Kubernetes/CI tooling as products, frontend frameworks, cloud-vendor specifics, and full system-design interviews. The book builds the foundations those rely on.

---

## 2. How every chapter is built (mandatory template)

```
Chapter opener
  - Where we are: what previous chapters established, what this one adds
  - Learning outcomes: "After this chapter you can explain/trace/diagnose..."
  - Interview questions this chapter prepares you for (8–15)
  - Scope: what is here, and what is elsewhere (with chapter numbers)

Each major concept, in this order:
  1. Definition: interview-ready, 1–3 sentences
  2. Why it exists: the problem it solves, and what breaks without it
  3. Mechanism: state, steps, data structures, interactions (diagram/trace required)
  4. Concrete: worked example, timeline, SQL, pseudocode, wire bytes, or code
  5. Guarantees & non-guarantees: what it promises and what it explicitly does not
  6. Failure & diagnosis: what goes wrong, the symptom, the evidence, the tool
  7. Backend connection: how it shapes design, performance, correctness, production
  [Java lens] / [Framework lens] / [Other languages] boxes, only where useful

Chapter close
  - Summary: the mental model in one page
  - Guarantees/Non-guarantees table
  - Interview Drill: 10–20 Qs, each with "a strong answer must mention..." and "common wrong answer"
  - Scenario Exercises: 4–8 unfamiliar situations to reason through (not recall)
  - Diagnosis Lab: 2–4 symptom → hypothesis → evidence → fix walkthroughs
```

---

# PART I: THE MACHINE AND THE OPERATING SYSTEM

## Chapter 1: The Web Application: A Structural Overview
*10–12 pp (≈5k words) | Prereq: none | Draft exists: `ch01_web_application_overview.md`*

1. Client/server as a distributed system: why the boundary exists, and why a server is a program, not a machine
2. The request–response interaction and the trust boundary (never trust the client)
3. Frontend, API, backend, data stores, caches, queues, third parties: the role of each box
4. State and failure boundaries: where data lives, and what can fail independently
5. Logical architecture vs physical deployment (one box can be many machines, and the reverse)
6. Monolith vs services: a first look, with no recommendations yet
7. "What happens when you click Buy": a 1-page trace used as the book's roadmap, every hop labelled with its chapter number
8. How to read this book: the chapter template and how to use the Interview Drills

*Status: revised to v2 on 2026-10-14 (Core Java assumptions removed, roadmap remapped, v2 chapter close added).*

---

## Chapter 2: How a Program Runs: Hardware, Memory Hierarchy, and Language Runtimes
*15–18 pp (≈7.5k words) | Prereq: Ch 1 | NEW*

**You'll be able to answer:** Why is RAM access slower than cache? Compiled vs interpreted vs JIT? Stack vs heap? What is garbage collection, and why does it cause latency spikes? Why do "latency numbers" matter for design?

1. The CPU: instructions, registers, clock, cores, hardware threads (SMT)
2. The memory hierarchy: registers → L1/L2/L3 → RAM → SSD → HDD → network. Latency numbers worth memorising, and why
3. CPU caches: cache lines, locality (temporal/spatial), cache misses, and false sharing (a preview of Ch 5)
4. Storage devices: HDD mechanics (seek, rotation) vs SSD (pages, blocks, erase, write amplification). Sequential vs random I/O, qualified by medium
5. From source to execution: compilation, interpretation, bytecode + VM, JIT; AOT vs JIT trade-offs
6. A program's memory layout: code, static data, stack (frames, recursion, stack overflow), heap (allocation)
7. Memory management models: manual (malloc/free; leaks, use-after-free), reference counting (cycles), tracing GC
8. Tracing GC mechanics: roots, mark, sweep/compact, generational hypothesis, stop-the-world pauses, concurrent collectors. Why GC pauses show up as p99 latency
9. Runtime comparison as execution environments: JVM, CPython (GIL, ref counting), Node/V8 (single-threaded event loop + JIT), Go (goroutines + GC)
10. Backend connection: why the runtime's memory use ≠ the configured heap (preview of Ch 3 RSS), startup/warm-up, choosing heap size

*Not here:* the OS view of memory (Ch 3), concurrency (Ch 5).

---

## Chapter 3: Operating Systems I: Processes, Threads, Scheduling, and Memory
*28–32 pp (≈13.5k words) | Prereq: Ch 2 | Seed: `archive/v1/ch02_os_foundations_v1.md`*

**You'll be able to answer:** Process vs thread? What happens in a context switch? What is a system call and why is it expensive? How does virtual memory work? What is a page fault, TLB, thrashing? Zombie vs orphan? fork/exec? User vs kernel threads?

1. What an OS is: resource manager + abstraction + isolation. Kernel vs user space
2. CPU privilege modes, system calls (step-by-step trap → kernel → return), interrupts vs traps vs exceptions, polling vs interrupts. Cost of a syscall
3. Program vs process. Process address space. **Process Control Block (PCB)**. PIDs
4. **Process lifecycle & states** (new, ready, running, blocked/waiting, terminated) with transition diagram
5. Process creation: fork, exec, copy-on-write, wait. Exit status. Zombie and orphan processes, PID 1 / init
6. Signals: SIGTERM vs SIGKILL vs SIGINT vs SIGHUP, handlers, graceful shutdown (backend link)
7. Threads: what a thread shares vs owns (stack, registers, TLS), why threads are cheaper than processes, and why they're riskier
8. **Threading models:** kernel threads, user-level (green) threads, M:N mapping, coroutines. Where virtual threads/goroutines/async fit (preview of Ch 5/11)
9. Multi-process vs multi-threaded servers (e.g. PostgreSQL process-per-connection vs MySQL thread-per-connection, Nginx workers)
10. **CPU scheduling:** goals (throughput, latency, fairness). Preemptive vs cooperative. FCFS, SJF, SRTF, Round Robin, Priority, MLFQ (interview set). Linux CFS idea. Starvation & aging
11. **Context switch** mechanics and cost: registers, TLB/cache effects, voluntary vs involuntary switches
12. CPU-bound vs I/O-bound work, load average, run queue, CPU throttling under quotas
13. **Virtual memory:** why it exists (isolation, abstraction, overcommit), virtual vs physical addresses, MMU
14. Paging: pages/frames, page tables, **multi-level page tables**, **TLB** and TLB misses, huge pages. Segmentation (interview contrast). Internal vs external fragmentation
15. Page faults (minor vs major), demand paging, copy-on-write revisited, **mmap**
16. **Page replacement:** FIFO, Optimal, LRU, Clock/second-chance, Belady's anomaly. Working set, **thrashing**
17. Swap and why backends avoid it. The OOM killer. Memory overcommit
18. What the OS measures: VSZ vs RSS vs a runtime heap; why a process with a 2 GB heap uses 3 GB of RSS
19. Diagnosis lab: `top/htop`, `ps`, `vmstat`, `/proc/<pid>`, high CPU vs high load vs high iowait; a process "vanished" (OOM)
20. Placement-round topics (added 2026-10-21; campus rounds and exams): batch vs multiprogramming vs multitasking vs multiprocessing vs real-time, the boot process, contiguous memory allocation (first/best/worst fit, compaction)

*Not here:* files/I/O (Ch 4), locks and synchronization (Ch 5).

---

## Chapter 4: Operating Systems II: Files, I/O, IPC, and Isolation
*22–26 pp (≈11k words) | Prereq: Ch 3 | Seed: `archive/v1/ch02_os_foundations_v1.md`*

**You'll be able to answer:** What is a file descriptor? Blocking vs non-blocking vs async I/O? How does epoll work, and why does it scale? What is the page cache? Why does fsync matter? VM vs container? What are cgroups and namespaces? IPC mechanisms?

1. "Everything is a file": file descriptors, the per-process FD table, the open-file table, inodes
2. Filesystems: inodes, directories, links, journaling (what it protects and what it doesn't)
3. The read/write path: user buffer → kernel → **page cache** → disk. Write-back, dirty pages
4. **Durability:** write vs fsync vs O_DIRECT, the "fsync lie" risks, and why databases care (preview of Ch 15)
5. **I/O models:** blocking, non-blocking, I/O multiplexing (select → poll → **epoll**/kqueue; level vs edge-triggered), signal-driven, async I/O (io_uring). The C10K problem
6. Sockets as file descriptors: socket(), bind(), listen(), accept(), connect(), the accept/backlog queue (bridge to Ch 6 and Ch 10)
7. Zero-copy (sendfile) and buffering: why it matters for proxies and file serving
8. **IPC:** pipes, named pipes, Unix domain sockets, shared memory, message queues, signals. Trade-offs, and why services use network IPC
9. Resource limits: ulimit/rlimits (nofile, nproc), FD exhaustion ("Too many open files"), port exhaustion preview
10. **Isolation:** VMs (hypervisor types 1/2, hardware virtualisation) vs **containers** (shared kernel; namespaces: pid, net, mnt, uts, ipc, user; **cgroups**: CPU quota/shares, memory limits). Container images as layered filesystems. Security implications of a shared kernel. **VM vs container comparison table**
11. Running a server process well: PID 1 in containers, signal forwarding, graceful shutdown sequence
12. Diagnosis lab: `lsof`, `strace`, `iostat`, `ss`, `dmesg`; FD leak; cgroup CPU throttling; disk-full; slow fsync
13. Placement-round topics (added 2026-10-21; campus rounds and exams): disk scheduling (FCFS, SSTF, SCAN, LOOK, C-SCAN, C-LOOK), file allocation (contiguous, linked, indexed; inode pointers), free-space management, RAID 0/1/5/6/10

---

## Chapter 5: Concurrency Fundamentals
*30–35 pp (≈15k words) | Prereq: Ch 3 | NEW (replaces the Core Java assumption)*

**You'll be able to answer:** Concurrency vs parallelism? What is a race condition, and how do you prevent one? Mutex vs semaphore? What is a memory model / visibility problem? What is CAS and lock-free? The four deadlock conditions and how to prevent/avoid/detect deadlock? Producer–consumer? Async/await vs threads? What is the GIL?

1. Concurrency vs parallelism; why software is concurrent (I/O overlap, multicore, responsiveness)
2. Shared state and the **race condition**: read-modify-write, check-then-act. Interleaving diagrams. The **critical section** problem and its requirements (mutual exclusion, progress, bounded waiting)
3. **Atomicity, visibility, ordering:** CPU caches, store buffers, compiler/CPU reordering. What a **memory model** is. **Happens-before**. Memory barriers. Language-neutral, with a Java lens (`volatile`, `synchronized`) and a C++/Go note
4. **Hardware atomics:** test-and-set, compare-and-swap (CAS), fetch-and-add. Building a spinlock from CAS. The ABA problem
5. **Locks:** mutex, spinlock vs blocking lock (futex idea), reentrancy, fairness, read–write locks, lock granularity, lock contention cost
6. **Semaphores** (counting/binary) vs mutex. **Condition variables** & monitors (wait/signal, spurious wake-ups, the "while not if" rule)
7. **Classic problems:** producer–consumer (bounded buffer), readers–writers, dining philosophers. Each solved with pseudocode
8. **Liveness failures:** deadlock (4 Coffman conditions; prevention via lock ordering, avoidance via Banker's algorithm intuition, detection via the wait-for graph, recovery), livelock, starvation, priority inversion
9. Lock-free and wait-free structures (concept level), atomic counters, why lock-free isn't automatically faster
10. Thread-safety strategies: immutability, confinement (thread-local), synchronisation, concurrent data structures, message passing
11. **Thread pools & executors:** work queues, sizing for CPU-bound vs I/O-bound work, rejection policies
12. **Futures/promises and async/await:** what they are, continuations, callback hell
13. **Execution models:** thread-per-task, event loop (Node; single thread + epoll), coroutines/green threads/virtual threads/goroutines (M:N), actors (brief). **GIL** (CPython) and its consequences
14. Testing and diagnosing concurrency bugs: why they're non-deterministic, thread dumps, lock contention profiling, stress testing

*Not here:* server-specific capacity and request-scoped state (Ch 11), database concurrency control (Ch 17).

---

# PART II: NETWORKS AND WEB PROTOCOLS

## Chapter 6: Networking: How Computers Communicate
*28–32 pp (≈13.5k words) | Prereq: Ch 4 | Seed: `archive/v1/ch03_networking_v1.md` (≈4.3k words, expand ~3×)*

**You'll be able to answer:** What happens at each layer when you send a packet? TCP vs UDP? Explain the 3-way handshake and TIME_WAIT. How does DNS resolution work? What is NAT? Flow vs congestion control? Why does connect time out vs get refused?

1. Why layering: the TCP/IP model vs OSI (as interview vocabulary), encapsulation/decapsulation, headers at each layer
2. Link layer: Ethernet frames, MAC addresses, switches, **ARP**, MTU
3. Network layer: IPv4 header, addressing, **subnets/CIDR**, routing tables, default gateway, hop-by-hop routing, TTL, **ICMP** (ping, traceroute), fragmentation & Path MTU
4. **DHCP**: how a host gets its address
5. Private vs public addressing, **NAT** (SNAT/DNAT, port translation, conntrack), IPv6 basics, firewalls & security groups (stateful vs stateless)
6. **DNS** in depth: hierarchy, root/TLD/authoritative, recursive resolvers, stub resolver, caching & TTL, record types (A, AAAA, CNAME, MX, TXT, NS, SRV), negative caching, DNS-based load balancing, failure modes and propagation myths
7. Transport layer: ports, multiplexing, the 4-tuple
8. **TCP** in depth: segment header, 3-way handshake, sequence/ack numbers, retransmission & RTO, the sliding window, **flow control** (rwnd), **congestion control** (slow start, AIMD, cwnd, fast retransmit, CUBIC/BBR names), Nagle & delayed ACK, keepalive, 4-way teardown, **TCP state machine**, **TIME_WAIT** (why it exists, port exhaustion), RST, SYN backlog & SYN floods
9. What TCP does **not** guarantee: message boundaries, application-level delivery, exactly-once
10. **UDP**: when and why (DNS, video, QUIC). Head-of-line blocking in TCP, and QUIC's answer
11. Socket API from the network view: client vs server lifecycle, ephemeral ports, SO_REUSEADDR/REUSEPORT
12. Latency, bandwidth, throughput, BDP, RTT, packet loss and its effect on throughput. Tail latency
13. Load balancing primer: L4 vs L7 (detail in Ch 10)
14. Diagnosis lab: refused vs connect timeout vs read timeout vs reset vs unknown host vs no route. Tools: `ping`, `traceroute`, `dig`, `ss`, `tcpdump`/Wireshark reading
15. Placement-round topics (added 2026-10-21; campus rounds and exams): hub/bridge/switch/router and collision vs broadcast domains, topologies, circuit vs packet switching, unicast/broadcast/multicast/anycast, classful addressing and subnetting practice, parity/checksum/CRC/Hamming, stop-and-wait/Go-Back-N/selective repeat, ALOHA and CSMA/CD vs CSMA/CA, distance-vector vs link-state routing (RIP, OSPF), BGP

---

## Chapter 7: HTTP: The Language of Client–Server Communication
*28–32 pp (≈13.5k words) | Prereq: Ch 6 | Seed: `archive/v1/ch04_http_v1.md` (≈4.6k words, expand ~3×)*

**You'll be able to answer:** Structure of an HTTP request/response? Safe vs idempotent methods? PUT vs PATCH vs POST? 401 vs 403? How does HTTP caching work (ETag, Cache-Control)? HTTP/1.1 vs 2 vs 3? What does keep-alive do?

1. Why HTTP exists. Stateless request/response. Where it sits in the stack
2. URIs/URLs: components, encoding, the request target, query vs path vs fragment
3. Message format on the wire: start line, headers, body. **Framing** (Content-Length vs chunked). Request smuggling as a framing failure
4. **Methods:** semantics, **safety, idempotency, cacheability**. Why GET must not mutate. PUT/PATCH/POST/DELETE semantics
5. **Status codes:** the classes, and the ones to know (200, 201, 202, 204, 301/302/307/308, 304, 400, 401, 403, 404, 409, 412, 422, 429, 500, 502, 503, 504) and what each does *not* tell you
6. Headers: general, request, response, representation. Host, User-Agent, Accept-*, Content-Type, Authorization, custom headers
7. Bodies & representations: media types, JSON, form encoding, multipart, character encoding/UTF-8
8. Content negotiation & compression (gzip/br). Range requests
9. **Connections:** keep-alive/persistent connections, pipelining, head-of-line blocking. Connection ≠ request ≠ thread
10. **HTTP/2:** binary framing, streams, multiplexing, HPACK, flow control, server push (deprecated). **HTTP/3** over QUIC
11. Redirects and their semantics
12. **HTTP caching:** freshness (max-age, Expires), validation (ETag/If-None-Match, Last-Modified), Cache-Control directives (no-cache vs no-store, private/public, s-maxage), Vary. Browser vs shared/CDN caches
13. Conditional requests & optimistic concurrency (If-Match → 412), a preview of Ch 13/17
14. Cookies as a protocol mechanism (Set-Cookie syntax only; behaviour in Ch 9/19)
15. Limits, timeouts, malformed messages, lost responses: why a retry after a timeout is ambiguous
16. End-to-end trace: raw bytes of a real request/response, with `curl -v`

---

## Chapter 8: Cryptography Essentials and HTTPS/TLS
*20–24 pp (≈10k words) | Prereq: Ch 7 | v1 Ch 5: expanded with a crypto foundation*

**You'll be able to answer:** Symmetric vs asymmetric encryption? Hashing vs encryption vs encoding? What is a MAC/HMAC and a digital signature? How does the TLS handshake work? How does certificate validation work? What is forward secrecy? What does HTTPS *not* protect?

1. Security goals: confidentiality, integrity, authenticity, non-repudiation. Threat model: passive vs active attacker (MITM)
2. Encoding vs encryption vs hashing (Base64 is not encryption)
3. **Symmetric encryption:** block/stream ciphers concept, AES, modes (why ECB is bad), **AEAD** (AES-GCM, ChaCha20-Poly1305), nonces
4. **Cryptographic hashes:** properties (preimage, collision resistance), SHA-256. Why fast hashes are wrong for passwords (preview of Ch 20)
5. **MAC / HMAC:** integrity + authenticity with a shared key (webhooks, cookies, JWT HS256)
6. **Asymmetric crypto:** key pairs, RSA/ECC intuition, encryption vs **digital signatures**
7. **Key exchange:** Diffie–Hellman/ECDHE intuition, **forward secrecy**
8. **Certificates & PKI:** X.509 fields, CAs, chain of trust, validation steps, hostname verification, revocation (CRL/OCSP), Certificate Transparency, Let's Encrypt/ACME
9. **TLS 1.3 handshake** step by step (vs 1.2 at a high level), session resumption, 0-RTT and its replay risk, SNI, ALPN
10. Where TLS terminates: at a proxy/LB vs end-to-end, re-encryption, **mTLS** for service-to-service
11. HSTS, mixed content, certificate expiry outages
12. What HTTPS guarantees and what it doesn't (endpoint compromise, metadata, application bugs)
13. Diagnosis lab: expired cert, wrong chain, hostname mismatch, untrusted CA, protocol mismatch; `openssl s_client`

---

## Chapter 9: How the Browser Works, and Browser-Side Security
*20–24 pp (≈10k words) | Prereq: Ch 6–8*

**You'll be able to answer:** What happens when you type a URL? Same-Origin Policy? How does CORS work (preflight)? What is XSS and how do you prevent it? What does CSP do? SSR vs CSR vs SSG, and what they change for the backend?

1. Browser architecture: processes, networking stack, connection pooling per origin, DNS/HTTP caches
2. **Navigation end to end:** URL → DNS → TCP → TLS → HTTP → parse → render (the full interview answer)
3. Rendering pipeline: HTML parsing → DOM, CSS → CSSOM, render tree, layout, paint, composite. Blocking scripts, async/defer
4. JavaScript in the browser: event loop, fetch/XHR, where API calls come from
5. Browser storage: cookies (browser behaviour: SameSite, Secure, HttpOnly, scope), localStorage/sessionStorage, IndexedDB, and their security implications
6. **Origins & the Same-Origin Policy:** what is and isn't restricted
7. **CORS:** simple vs preflighted requests, the headers, credentials, common misconfigurations, why CORS is not server-side protection
8. **XSS:** stored, reflected, DOM-based. Output encoding, sanitisation, **CSP**, HttpOnly
9. Clickjacking (frame-ancestors/X-Frame-Options) and other security headers (referrer policy, nosniff)
10. **SSR vs CSR vs SSG/ISR**, only as they change what the backend serves, caches and secures
11. Diagnosis lab: CORS error decoded, blocked mixed content, using DevTools network/timing panels

---

# PART III: THE SERVER AND THE APPLICATION

## Chapter 10: Inside the Server: From Network Bytes to Application Code
*25–30 pp (≈12.5k words) | Prereq: Ch 4, 6, 7*

**You'll be able to answer:** What does a reverse proxy do? L4 vs L7 load balancing? Load-balancing algorithms? What happens between accept() and your handler? What is a CDN? How do WebSockets work?

1. The edge: CDN (edge caching, origin shielding), WAF, DDoS protection, at concept level
2. **Forward vs reverse proxy**; what a reverse proxy does (TLS termination, routing, buffering, compression, rate limits, header rewriting, X-Forwarded-For/Forwarded)
3. **Load balancers:** L4 vs L7, algorithms (round robin, least connections, least response time, consistent hashing, power-of-two-choices), health checks (active/passive), sticky sessions and their costs, connection draining
4. API gateways vs load balancers vs service mesh (concept level)
5. The server process: listening socket, accept queue, acceptor/worker design, keep-alive handling, request parsing and limits
6. Dispatch: routing, the middleware/filter chain pattern, handler invocation, response writing, flushing, streaming
7. Language-neutral server architectures: thread-per-connection (classic Java servlet containers), worker processes (Gunicorn), event loop (Node, Nginx), async runtimes. [Java lens: Tomcat & Servlet API] [Framework lens: Spring DispatcherServlet]
8. Error handling across layers: who produces a 502 vs 503 vs 504 vs 500, and why
9. Long-lived connections: **WebSockets** (the 101 upgrade, frames), **SSE**, long polling. Effects on proxies and capacity
10. Timeouts at each hop and how they must nest
11. Diagnosis lab: 502 from the proxy, 504 gateway timeout, request body too large, client IP missing behind proxy

---

## Chapter 11: Concurrency in a Server: Execution Models, Capacity, and Overload
*22–26 pp (≈11k words) | Prereq: Ch 5, Ch 10*

**You'll be able to answer:** How many requests can a server handle concurrently? Little's Law? Why does adding threads make it worse? Thread-per-request vs event loop vs virtual threads? What is backpressure? Why is shared mutable state in a singleton service a bug?

1. Concurrency at the request level: overlapping requests, shared vs request-scoped state
2. Execution models compared on one workload: thread pool, event loop, virtual threads/goroutines, multi-process. What each does with blocking calls
3. **Little's Law** and capacity math: concurrency = throughput × latency. Pool sizing for CPU vs I/O work. Worked examples
4. Queues everywhere: accept queue, executor queue, pool wait queue. Why unbounded queues hide overload
5. Resource pools as separate bottlenecks (DB connections, HTTP client pools): why more threads → more DB contention
6. Thread safety in application code: singleton services, static/global state, caches, request-local context (thread-locals, context propagation in async code). [Framework lens: request scope, `@Async` context loss]
7. Cross-instance coordination: why in-process locks fail with N replicas, a preview of DB locks (Ch 17) and distributed locks (Ch 23)
8. **Overload:** saturation, latency collapse, retry storms, **backpressure**, bounded queues, admission control, load shedding (mechanisms detailed in Ch 24)
9. Blocking vs non-blocking APIs vs reactive streams: API style ≠ execution model. When reactive is worth it
10. Diagnosis lab: thread dump reading, pool exhaustion, a slow downstream causing total stall, CPU saturation vs lock contention

---

## Chapter 12: Application Architecture: Structure, Dependencies, and Data Flow
*25–30 pp (≈12.5k words) | Prereq: Ch 10, 11*

**You'll be able to answer:** What is dependency injection and why use it? What is IoC? Layered vs hexagonal architecture? How do frameworks add behaviour (transactions, security) without your code calling it? Monolith vs microservices trade-offs?

1. Why structure matters: coupling, cohesion, change, testability
2. Layered architecture (controller/service/repository), and the responsibility of each layer
3. Domain modelling basics: entities, value objects, DTOs vs domain objects, validation placement
4. **Dependency injection & Inversion of Control:** the problem, constructor injection, containers, object lifecycles/scopes. Building a tiny DI container in pseudocode. [Framework lens: Spring IoC]
5. **Interception & proxies:** decorator pattern, dynamic proxies, AOP. How "annotations" add transactions/security/caching. The self-invocation pitfall
6. Request data flow: deserialisation → validation → mapping → business logic → persistence → serialisation
7. **Serialisation:** JSON mapping, unknown fields, polymorphism, **insecure deserialisation**
8. Error handling strategy: exceptions vs result types, mapping errors to HTTP responses
9. Configuration, profiles, secrets injection (12-factor config)
10. Hexagonal/ports & adapters, clean architecture: what problem they solve and what they cost
11. **Monolith vs modular monolith vs microservices:** boundaries, data ownership, operational cost, distributed-monolith failure
12. Testing seams the architecture creates (unit vs integration), at concept level

---

## Chapter 13: Designing Web APIs
*25–30 pp (≈12.5k words) | Prereq: Ch 7, 12*

**You'll be able to answer:** What makes an API RESTful? How do you design pagination? How do idempotency keys work? How do you version an API? REST vs gRPC vs GraphQL? How do you handle errors consistently?

1. What an API contract is. Consumers, compatibility, Postel's law and its limits
2. **REST:** resources, representations, uniform interface, statelessness, HATEOAS (and its real-world use). Resource modelling exercises
3. Method/status usage in practice, and partial updates (PUT vs PATCH, JSON Patch/Merge Patch)
4. **Idempotency:** idempotent design, **idempotency keys** (storage, concurrency, expiry). Retries from the client view
5. **Pagination:** offset vs cursor/keyset (and why offset breaks at scale), filtering, sorting
6. Concurrency control in APIs: ETags + If-Match, version fields, 409/412
7. Errors: RFC 9457 Problem Details, validation errors, error codes
8. **Versioning & evolution:** additive changes, breaking changes, URL vs header versioning, deprecation
9. Long-running operations: 202 + status resource, webhooks (signing with HMAC, retries)
10. Data formats & schema evolution: JSON, Protocol Buffers, Avro. Forward/backward compatibility
11. **gRPC** (HTTP/2, protobuf, streaming) and **GraphQL** (resolvers, N+1, query cost): mechanisms and trade-offs vs REST
12. API security surface: authN/authZ placement (detailed in Ch 20), input validation, mass assignment, rate limiting headers (429, Retry-After)
13. Outbound calls: HTTP clients, connection pools, timeouts, **SSRF**
14. Documentation and contracts: OpenAPI, contract testing (concept level)

---

# PART IV: DATA PERSISTENCE

## Chapter 14: Relational Data Modelling and SQL
*28–32 pp (≈13.5k words) | Prereq: Ch 1*

**You'll be able to answer:** What is normalisation, and 1NF/2NF/3NF/BCNF? Primary vs foreign vs candidate keys? Types of joins? WHERE vs HAVING? What are window functions? How does NULL behave?

1. The relational model: relations, tuples, attributes, domains. Why it won (data independence, declarative queries)
2. Keys: super, candidate, primary, alternate, foreign, surrogate vs natural
3. Constraints: NOT NULL, UNIQUE, CHECK, FK actions, when constraints are checked (immediate vs deferred)
4. **ER modelling:** entities, relationships, cardinality (1:1, 1:N, M:N, junction tables), mapping to tables
5. **Normalisation:** update/insert/delete anomalies, functional dependencies, 1NF → 2NF → 3NF → BCNF with worked decomposition. **Denormalisation** trade-offs
6. SQL fundamentals: DDL/DML/DCL/TCL. SELECT evaluation order (FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT)
7. **Joins:** inner, left/right/full outer, cross, self, anti/semi joins (EXISTS/NOT EXISTS), with result-set diagrams
8. Aggregation, GROUP BY, HAVING. Subqueries (correlated vs not), CTEs, recursive CTEs
9. **Window functions:** ROW_NUMBER, RANK, DENSE_RANK, LAG/LEAD, running totals, top-N per group
10. **NULL** & three-valued logic, and the pitfalls (NOT IN with NULLs)
11. Set operations, UPSERT (ON CONFLICT / MERGE), views and materialised views
12. Data types that matter: numeric vs float for money, timestamps & time zones, text/collation, JSON columns
13. Classic interview queries: nth highest salary, duplicates, gaps, top-N per group, running totals

*Not here:* how SQL executes (Ch 16), transactions (Ch 17).

---

## Chapter 15: Storage Engines and Durability
*22–26 pp (≈11k words) | Prereq: Ch 4, Ch 14 | NEW split from v1 Ch 12*

**You'll be able to answer:** How does a database store rows on disk? What is the buffer pool? What is WAL and how does crash recovery work? B-tree vs LSM-tree storage? Row vs column stores? What does a checkpoint do?

1. Database server architecture: client connections, process/thread model (PostgreSQL processes vs MySQL threads), shared memory, background workers
2. **Pages & heap files:** slotted page layout, tuples, row IDs/TIDs, TOAST/overflow
3. **Buffer pool:** page caching, eviction (LRU variants/clock), dirty pages, and its interaction with the OS page cache (double buffering)
4. **Write-ahead logging:** the WAL rule, log records, LSNs, group commit, fsync on commit
5. **Crash recovery:** redo (and undo in InnoDB-style engines), ARIES intuition, checkpoints and the recovery-time trade-off
6. Two persistence philosophies: **update-in-place (B-tree based)** vs **log-structured (LSM-tree:** memtable, SSTables, compaction, bloom filters; write vs read vs space amplification)
7. **Row-oriented vs column-oriented** storage: OLTP vs OLAP, compression, vectorised execution
8. In-memory databases & durability options (Redis RDB/AOF as an example)
9. Backups: logical vs physical, PITR with WAL archiving, RPO/RTO
10. Diagnosis lab: checkpoint spikes, disk full, WAL growth, slow fsync storage

---

## Chapter 16: Indexing and Query Execution
*25–30 pp (≈12.5k words) | Prereq: Ch 15*

**You'll be able to answer:** How does a B+tree index work? Why not index every column? Composite index column order? Covering index? Clustered vs non-clustered? How does the query planner choose a plan? Nested-loop vs hash vs merge join? How do you read EXPLAIN?

1. Why indexes exist: from a full scan to a logarithmic lookup
2. **B+tree** structure: nodes, fan-out, height, search/insert/split/delete, range scans via leaf links
3. Clustered vs secondary indexes (InnoDB clustered PK vs PostgreSQL heap + index). Index-only scans & visibility
4. **Composite indexes:** leftmost-prefix rule, column ordering, equality-then-range rule
5. Covering indexes, partial indexes, expression indexes, unique indexes
6. Other index types: hash, GIN/inverted (full text, JSON), GiST/spatial, BRIN, bitmap
7. Write cost, storage cost, and index bloat: why more indexes slow writes
8. **Query processing:** parse → rewrite → plan → execute. Logical vs physical plans
9. **The optimiser:** statistics, selectivity, cardinality estimation, cost model. Why stale stats produce bad plans
10. Scan types: seq, index, index-only, bitmap
11. **Join algorithms:** nested loop, hash join, sort-merge join, when each wins. Join ordering
12. **Reading EXPLAIN / EXPLAIN ANALYZE** with worked plans
13. Query anti-patterns: functions on indexed columns, implicit casts, leading wildcards, OR conditions, SELECT *, offset pagination
14. Partitioning tables (range/list/hash) within one database, and partition pruning
15. Diagnosis lab: slow-query log, pg_stat_statements, a plan that flipped, a missing index found from evidence

---

## Chapter 17: Transactions, Isolation, and Concurrency Control
*28–32 pp (≈13.5k words) | Prereq: Ch 5, 15, 16 | NEW split from v1 Ch 12*

**You'll be able to answer:** Explain ACID precisely. What are dirty read, non-repeatable read, phantom, lost update, write skew? What are the isolation levels and what does each prevent? How does MVCC work? Optimistic vs pessimistic locking? How do database deadlocks happen?

1. What a transaction is. Commit/rollback, autocommit, savepoints
2. **ACID**, each property with its implementing mechanism (atomicity via undo/MVCC, durability via WAL, consistency via constraints + application, isolation via concurrency control)
3. **Anomalies** with timelines: dirty write, dirty read, non-repeatable read, phantom, **lost update**, **read skew**, **write skew** (on-call doctors example)
4. **Isolation levels** per the SQL standard vs reality: Read Uncommitted, Read Committed, Repeatable Read (**PostgreSQL = snapshot isolation; InnoDB = MVCC + next-key/gap locks**), Serializable. A matrix of anomaly × level × engine
5. **Two-phase locking (2PL)**, strict 2PL, shared/exclusive locks, intention locks, lock granularity, lock escalation
6. **MVCC:** row versions, snapshots, visibility rules (xmin/xmax), undo-log MVCC (InnoDB) vs heap-version MVCC (PostgreSQL), VACUUM, bloat, wraparound
7. **Serializable Snapshot Isolation (SSI)** intuition, serialisation failures and retry
8. Explicit locking: SELECT … FOR UPDATE / FOR SHARE, NOWAIT, **SKIP LOCKED** (job queues), advisory locks
9. **Optimistic vs pessimistic concurrency:** version columns, compare-and-set UPDATE, retry loops
10. **Database deadlocks:** how they form, detection, victim selection, prevention by ordering
11. Atomic in-database operations: `UPDATE … SET x = x - 1 WHERE x > 0`, constraints as concurrency guards, uniqueness for idempotency
12. Long transactions: lock holding, vacuum blocking, replication effects
13. Worked case: inventory oversell and double-spend, solved four ways and compared
14. Diagnosis lab: lock waits (pg_locks), deadlock log, serialization failures, idle-in-transaction

---

## Chapter 18: Application–Database Integration
*25–30 pp (≈12.5k words) | Prereq: Ch 11, 12, 14, 17*

**You'll be able to answer:** How does a connection pool work, and how do you size it? What is SQL injection and why do prepared statements prevent it? What is the N+1 problem? How does an ORM track changes? Where should transaction boundaries go?

1. Drivers and the wire protocol: connect, auth, query, result streaming
2. **Connection cost and connection pooling:** pool mechanics, sizing (Little's Law again; DB cores), wait timeouts, leaks, validation, server-side poolers (PgBouncer modes)
3. **Prepared statements:** parse/plan reuse, parameter binding, and why they stop **SQL injection**. Injection variants (ORDER BY, LIKE, second-order)
4. **ORM internals:** mapping, identity map/persistence context, unit of work, dirty checking, flush timing, lazy vs eager loading, proxies. [Framework lens: JPA/Hibernate] [Other languages: SQLAlchemy, Prisma]
5. **N+1 queries:** cause, detection, fixes (joins, batch fetching, DTO projections)
6. **Transaction boundaries in application code:** where to begin/commit, propagation, read-only transactions, never calling remote services inside a transaction, the self-invocation pitfall
7. Lost updates at the application level, and choosing among the Ch 17 fixes
8. Batching, bulk operations, streaming large results
9. **Schema migrations:** versioned migrations, zero-downtime changes (expand/contract), locking DDL hazards
10. Read/write splitting and replica lag (a preview of Ch 22)
11. Diagnosis lab: pool exhaustion ("connection is not available"), slow endpoint caused by N+1, leaked transaction, migration lock outage

---

# PART V: STATE, IDENTITY, AND SECURITY

## Chapter 19: Web State Management: Cookies, Sessions, and CSRF
*20–24 pp (≈10k words) | Prereq: Ch 7, 9, 10*

**You'll be able to answer:** Why is HTTP stateless and how do sessions add state? Cookie attributes and what each protects? Server-side sessions vs tokens? Sticky sessions vs a shared session store? What is CSRF and how do you prevent it? Session fixation?

1. Statelessness and where state can live (client, server memory, shared store, DB)
2. Cookies end to end: attributes (Domain, Path, Expires/Max-Age, Secure, HttpOnly, **SameSite** Lax/Strict/None), third-party cookie restrictions
3. **Server-side sessions:** session ID generation (entropy), storage, lookup, expiry (idle vs absolute), invalidation
4. Scaling sessions: in-memory + sticky sessions vs a shared store (Redis) vs stateless signed/encrypted cookies. Trade-offs
5. Session security: hijacking, **fixation** (regenerate on login), logout semantics
6. **CSRF:** the mechanism, why cookies enable it, defences (SameSite, synchronizer token, double-submit, Origin checks), when token-based APIs are immune and when they aren't
7. Stateful vs stateless auth preview: sessions vs bearer tokens (detail in Ch 20)
8. Diagnosis lab: users logged out randomly behind a load balancer, cookie not sent cross-site, CSRF token mismatch

---

## Chapter 20: Authentication, Authorization, and Identity
*30–35 pp (≈15k words) | Prereq: Ch 8, 19*

**You'll be able to answer:** AuthN vs AuthZ? How should passwords be stored? Session vs JWT trade-offs? How does JWT validation work, and what are its pitfalls? Explain the OAuth 2.0 authorization code flow with PKCE. OAuth vs OIDC? RBAC vs ABAC? How do services authenticate to each other?

1. Identity, authentication, authorization: precise distinctions
2. **Password storage:** salting, slow KDFs (bcrypt, scrypt, Argon2id), peppers, why SHA-256 is wrong. Credential stuffing, brute force, rate limiting, lockout trade-offs, account enumeration
3. Login, logout, password reset, and email-verification flows done safely
4. **MFA:** TOTP mechanism, WebAuthn/passkeys (public-key challenge–response)
5. **Tokens:** opaque vs self-contained. **JWT** structure, signing (HS256 vs RS256/ES256), validation steps, `alg:none` / key confusion, expiry, revocation problem, refresh tokens & rotation, where to store tokens in a browser
6. **OAuth 2.0:** roles, the authorization code flow + **PKCE** step by step, client credentials, scopes, why the implicit flow died
7. **OpenID Connect:** ID token vs access token, discovery, JWKS, SSO
8. **Authorization models:** ACLs, RBAC, ABAC, ReBAC. Object-level authorization and **IDOR/BOLA**. Enforcing it in code and in queries (multi-tenancy, row-level security)
9. **Service-to-service identity:** API keys, mTLS, workload identity, token exchange. **SSRF** against metadata endpoints
10. **Secrets management:** where secrets live, rotation, not in code/logs
11. **Application Security Synthesis:** threat modelling (STRIDE at a glance), defence in depth, OWASP Top 10 mapped to the chapters where each is taught, security logging & audit trails
12. Diagnosis lab: 401 vs 403 debugging, clock-skew JWT failures, token-expiry storms, privilege escalation via IDOR

---

# PART VI: PERFORMANCE AND DISTRIBUTED DATA

## Chapter 21: Caching
*22–26 pp (≈11k words) | Prereq: Ch 7, 11, 17*

**You'll be able to answer:** Where can you cache, and what does each layer cost? Cache-aside vs read-through vs write-through vs write-back? How do you invalidate? What are stampede, penetration and avalanche, and how do you prevent them? LRU vs LFU? How do you keep a cache consistent with the DB?

1. Why caching works (locality, skew) and the cost model: hit ratio, latency, staleness
2. **Cache layers:** browser, CDN, reverse proxy, in-process, distributed (Redis/Memcached), DB buffer pool. What each is good at
3. **Patterns:** cache-aside, read-through, write-through, write-behind/back, refresh-ahead
4. **Eviction:** LRU (implementation: hash map + doubly linked list), LFU, FIFO, TTL, approximated LRU (Redis)
5. **Invalidation & consistency:** TTL vs explicit invalidation vs versioned keys. The delete-vs-update race. Double-delete. Why a cache gives no transactional guarantees
6. **Failure modes:** stampede/thundering herd (locking, request coalescing, jitter, early refresh), penetration (negative caching, bloom filters), avalanche (TTL jitter), hot keys
7. Distributed cache internals: data structures, single-threaded execution (Redis), persistence, replication and eviction policies, sharding preview (consistent hashing, Ch 23)
8. Caching beyond reads: rate-limit counters, sessions, idempotency results, distributed locks (preview of their caveats, Ch 23)
9. Multi-level caching and memory sizing
10. Diagnosis lab: low hit rate, stale data bug, latency spike from a hot key, cache outage taking down the DB

---

## Chapter 22: Replication and Consistency
*22–26 pp (≈11k words) | Prereq: Ch 15, 17, 18 | v1 Ch 17: expanded*

**You'll be able to answer:** Why replicate? Sync vs async replication trade-offs? Leader–follower vs multi-leader vs leaderless? What is replication lag, and which anomalies does it cause? Linearizability vs eventual consistency? Explain CAP properly, and PACELC? What is split-brain?

1. Why replicate: availability, read scaling, latency/geography, durability. Replication vs backup
2. **Single-leader replication:** log shipping (physical/WAL) vs logical replication, sync/semi-sync/async, durability vs latency
3. **Replication lag anomalies:** read-your-writes, monotonic reads, consistent prefix, and the techniques for each
4. **Failover:** failure detection, timeouts, leader election, promotion, **split-brain**, fencing
5. **Multi-leader replication:** use cases, write conflicts, conflict resolution (LWW, merge, CRDTs at intuition level)
6. **Leaderless replication:** quorums (W + R > N), sloppy quorums, hinted handoff, read repair, anti-entropy (Merkle trees)
7. **Consistency models ladder:** linearizability, sequential, causal, read-your-writes, eventual. What each costs
8. **CAP theorem** precisely (a partition forces a choice), the misconceptions, **PACELC**
9. Change Data Capture as replication to other systems (preview of Ch 25)
10. Diagnosis lab: "I saved but don't see it", replica lag spike, failover data loss

---

## Chapter 23: Partitioning, Coordination, and Choosing Data Stores
*24–28 pp (≈12k words) | Prereq: Ch 22 | NEW*

**You'll be able to answer:** How do you shard a database? Range vs hash partitioning? What is consistent hashing? How do you handle hot keys and rebalancing? Why are clocks unreliable? What is consensus and why is Raft needed? How does 2PC work and why is it avoided? How do you pick SQL vs NoSQL?

1. Why partition (shard): data size and write throughput beyond one node
2. **Partitioning strategies:** key-range, hash, directory-based. **Consistent hashing** (ring, virtual nodes)
3. Hot spots/hot keys and skew mitigation. **Rebalancing** strategies. Request routing
4. Secondary indexes across shards (local vs global). Cross-shard queries and joins
5. **Time & order:** clock skew, NTP, why wall clocks can't order events. Lamport clocks, vector clocks (intuition). Monotonic clocks for timeouts
6. **Consensus:** the problem, why it's hard (FLP intuition), **Raft** (leader election, log replication, terms, majority quorums), and where it's used (etcd, ZooKeeper, DB failover)
7. **Distributed locks & leases:** lease expiry hazards, **fencing tokens**, why "Redis lock" is subtle
8. **Distributed transactions:** **two-phase commit** (protocol, coordinator failure, blocking), why microservices avoid it, the preview of sagas (Ch 25)
9. **Beyond relational:** key-value, document, wide-column, graph, search (inverted index), time-series. Their data models, query models and storage engines
10. **Choosing a data store:** access patterns, consistency needs, integrity, scale, operations. SQL vs NoSQL without slogans. Polyglot persistence and source of truth
11. Diagnosis lab: one shard at 100% CPU (hot key), a cross-shard query timing out, a lock holder paused by GC

---

# PART VII: RELIABILITY AND OPERATIONS

## Chapter 24: Failure and Resilience
*24–28 pp (≈12k words) | Prereq: Ch 11, 13, 22 | split from v1 Ch 19*

**You'll be able to answer:** How do you set timeouts across a call chain? When is a retry safe? What are exponential backoff and jitter? How does a circuit breaker work? Bulkheads? How do rate limiters work (token bucket vs sliding window)? What is a cascading failure, and how do you stop one?

1. Failure taxonomy: crash, omission, timing, Byzantine (named only). Partial failure and the "unknown outcome" problem
2. **Timeouts & deadlines:** connect vs read vs total, deadline propagation, nesting rules, latency budgets
3. **Retries:** when safe (idempotency, Ch 13), exponential backoff, **jitter**, retry budgets, retry amplification across layers
4. **Circuit breakers:** closed/open/half-open state machine, thresholds, fallbacks and their dangers
5. **Bulkheads & concurrency limits**, adaptive concurrency
6. **Rate limiting algorithms:** fixed window, sliding log, sliding window counter, **token bucket**, leaky bucket. Distributed rate limiting
7. **Load shedding, backpressure, admission control**, priority-based shedding
8. **Cascading failures & metastable failures:** retry storms, thundering herds, the death spiral, and how to recover
9. Graceful degradation, feature flags/kill switches
10. Health checks (liveness vs readiness), graceful shutdown & connection draining (OS link to Ch 4)
11. Redundancy, failover and availability math (nines, serial vs parallel availability)
12. Chaos/failure-injection testing (concept)
13. Diagnosis lab: one slow dependency taking the whole system down, a retry storm after an outage, mis-nested timeouts

---

## Chapter 25: Asynchronous Work and Messaging
*24–28 pp (≈12k words) | Prereq: Ch 17, 22, 24 | split from v1 Ch 19*

**You'll be able to answer:** When should work be async? Queue vs pub/sub vs log (RabbitMQ vs Kafka)? At-most/at-least/exactly-once? How do you make consumers idempotent? What is the dual-write problem and the outbox pattern? Saga choreography vs orchestration? How does Kafka guarantee ordering?

1. Sync vs async: latency, coupling, failure isolation, user-visible consistency
2. Background jobs & task queues: workers, scheduling, cron, delayed jobs, DB-backed queues (SKIP LOCKED)
3. **Messaging models:** point-to-point queue, pub/sub, **partitioned log**. Broker internals at concept level (RabbitMQ-style vs Kafka-style)
4. **Kafka-style log mechanics:** topics, partitions, offsets, consumer groups, rebalancing, retention, compaction, ordering per partition
5. **Delivery semantics:** acknowledgements, at-most-once, at-least-once, "exactly-once" as an end-to-end business property (idempotent consumers, transactional processing)
6. **Idempotent consumers & deduplication** (inbox pattern, unique constraints)
7. **Ordering**, poison messages, retries with backoff, **dead-letter queues**, replay
8. **The dual-write problem → transactional outbox**, CDC (Debezium-style) as an alternative
9. **Sagas:** choreography vs orchestration, compensations, the failure matrix
10. Event-driven architecture: events vs commands, event schemas & evolution, event sourcing and CQRS (concept and costs)
11. Backpressure and consumer lag
12. Worked workflow: order → payment → inventory → notification, with failures injected at each step
13. Diagnosis lab: growing consumer lag, duplicate emails sent, a message stuck in a retry loop, a lost event from a dual write

---

## Chapter 26: Running Backends in Production: Deployment, Scaling, and Observability
*24–28 pp (≈12k words) | Prereq: Ch 4, 10, 24 | NEW (concept level, no product tutorials)*

**You'll be able to answer:** Horizontal vs vertical scaling? Why must services be stateless to scale? Blue–green vs canary vs rolling deployments? Logs vs metrics vs traces? Why use p99 and not averages? What are SLI/SLO/SLA and error budgets? How do you debug a production latency spike?

1. From code to running service: build artifact, image, configuration, environment
2. **Scaling:** vertical vs horizontal, statelessness, autoscaling signals and lag, capacity planning with Little's Law
3. Orchestration concepts: desired state, scheduling, service discovery, liveness/readiness probes, resource requests/limits (and their cgroup mapping from Ch 4)
4. **Deployment strategies:** rolling, blue–green, canary, feature flags. Backward-compatible deploys with DB migrations (links to Ch 18)
5. Configuration & secret delivery in production
6. **Observability:** logs (structured, levels, correlation IDs, PII hygiene), metrics (counters/gauges/histograms, cardinality), traces (spans, context propagation, sampling), OpenTelemetry concept
7. **Percentiles & tail latency:** why averages lie, histogram percentiles, coordinated omission (brief), fan-out amplification of tail latency
8. **Golden signals, RED & USE methods**
9. **SLI, SLO, SLA, error budgets**, alerting on symptoms vs causes, alert fatigue
10. Profiling in production: CPU flame graphs, heap/allocation profiles, thread dumps (generic)
11. Incident response: triage, mitigation before root cause, postmortems (blameless)
12. Diagnosis lab: walking from an alert to a root cause with the three signals

---

## Chapter 27: Capstone: End-to-End Request Lifecycle and Systematic Diagnosis
*20–25 pp (≈10k words) | Prereq: all*

1. **"What happens when you type a URL and press Enter?"**: the complete answer, every layer, with chapter cross-references (a 5-minute interview version and a 30-minute deep version)
2. Full lifecycle trace of a write request (place order): browser → DNS → TCP/TLS → CDN/LB → proxy → server → thread/event loop → middleware/auth → service → cache → DB transaction → outbox → response → async consumers
3. **A reusable diagnosis method:** symptom → scope → hypothesis → evidence → isolate layer → fix → verify → prevent
4. Case studies (each traced across layers):
   - p99 latency spike (GC / N+1 / pool exhaustion: telling them apart)
   - Inventory oversell under concurrency
   - Login works locally, fails in production (cookies/CORS/proxy headers)
   - Partial failure in an async payment workflow
   - Cascading outage from one slow dependency
   - Stale reads after failover
5. Final concept map: how every chapter connects
6. **Readiness self-assessment:** a checklist of 100 interview questions mapped to chapters
7. Where to go next: implementation practice, projects, DSA, framework depth, system design

---

## Topic homes (single source of truth; no orphans, no duplicates)

| Topic | Primary home | Also used in |
|---|---|---|
| Memory hierarchy, GC | Ch 2 | Ch 3, 26 |
| Processes/threads/scheduling/virtual memory | Ch 3 | Ch 5, 11 |
| I/O models, epoll, FDs, page cache, fsync | Ch 4 | Ch 10, 15 |
| Containers vs VMs, cgroups, namespaces | Ch 4 | Ch 26 |
| Locks, CAS, memory model, deadlock theory | Ch 5 | Ch 11, 17 |
| Little's Law, pool sizing | Ch 11 | Ch 18, 26 |
| DNS, TCP, UDP, NAT | Ch 6 | Ch 10, 27 |
| HTTP caching semantics | Ch 7 | Ch 21 |
| Crypto primitives, TLS, mTLS | Ch 8 | Ch 19, 20 |
| SOP, CORS, XSS, CSP | Ch 9 | Ch 19 |
| L4/L7 LB, reverse proxy, CDN, WebSockets | Ch 10 | Ch 21, 26 |
| DI, proxies/AOP, insecure deserialisation | Ch 12 | Ch 18 |
| REST, idempotency keys, pagination, gRPC, GraphQL, SSRF | Ch 13 | Ch 20, 24 |
| Normalisation, SQL, window functions | Ch 14 | |
| WAL, buffer pool, B-tree vs LSM, column stores | Ch 15 | Ch 16, 23 |
| B+tree indexes, planner, join algorithms, EXPLAIN | Ch 16 | Ch 18 |
| ACID, isolation, MVCC, locking | Ch 17 | Ch 18, 22 |
| Connection pools, SQL injection, ORM, N+1, migrations | Ch 18 | |
| Cookies (behaviour), sessions, CSRF | Ch 19 | |
| Password hashing, JWT, OAuth/OIDC, RBAC, IDOR, OWASP map | Ch 20 | |
| Caching patterns, eviction, stampede | Ch 21 | |
| Replication, consistency models, CAP/PACELC | Ch 22 | Ch 23 |
| Sharding, consistent hashing, clocks, Raft, 2PC, NoSQL | Ch 23 | Ch 21, 25 |
| Timeouts, retries, circuit breakers, rate-limit algorithms | Ch 24 | Ch 13 |
| Queues, Kafka, outbox, sagas, delivery semantics | Ch 25 | |
| Deployment, scaling, logs/metrics/traces, SLOs | Ch 26 | Ch 27 |

---

## Size summary

| Part | Chapters | Pages |
|---|---|---|
| I: Machine & OS | 1–5 | ≈105–125 |
| II: Networks & Protocols | 6–9 | ≈95–110 |
| III: Server & Application | 10–13 | ≈95–115 |
| IV: Data Persistence | 14–18 | ≈125–145 |
| V: State, Identity, Security | 19–20 | ≈50–60 |
| VI: Performance & Distributed Data | 21–23 | ≈70–80 |
| VII: Reliability & Operations | 24–27 | ≈90–110 |
| **Total** | **27** | **≈630–745 (≈290–330k words)** |
