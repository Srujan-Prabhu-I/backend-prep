# Chapter 1: The Web Application: A Structural Overview

## Where We Are

This is the first chapter. It assumes only that you can program in some language. Its job is to give you the **map**: the components of a web application, what each one is responsible for, where state lives, where failures happen, and which chapter of this book explains each mechanism in depth. Mechanisms here are kept shallow on purpose. Every later chapter zooms into one region of this map.

**After this chapter you can:**
- explain why a web application is a distributed system and why the client/server boundary exists;
- name each component of a typical backend (edge, application process, application code, database, cache, queue, external services) and state its responsibility;
- explain the trust boundary and why the backend must re-validate everything;
- distinguish logical architecture from physical deployment;
- trace a request from click to rendered page at the level of "which component, which chapter".

**Interview questions this chapter prepares you for:**
1. What is the client–server model, and why not run everything on the client?
2. Why is client-side validation not enough?
3. What is an API? What is serialization?
4. What is the difference between a reverse proxy, a load balancer and an application server?
5. Why can't you keep important data in the application process's memory?
6. How is a call to another service different from a function call?
7. What does "client and server are roles, not machines" mean?
8. What is the difference between logical architecture and physical deployment?
9. Monolith vs microservices: what is the basic trade-off?
10. At a high level, what happens when you click "Buy" on a shopping site?
11. What is a trust boundary?
12. Why does a cache appear in so many architectures, and what does it cost?

**Scope.** This chapter names components and assigns them to chapters. It does not explain how any of them work internally: OS (Chapters 3–4), networking (Chapter 6), HTTP (Chapter 7), TLS (Chapter 8), browsers (Chapter 9), servers (Chapter 10), databases (Chapters 14–18), caches (Chapter 21), queues (Chapter 25).

---

## 1.1 The Distributed Nature of Web Applications

A **web application** is a software application that users access over a network (typically the internet) through a web browser or mobile client, rather than installing and running it entirely on their own device. Gmail, Amazon, and any banking portal you log into through a browser are web applications.

To see why web applications are built the way they are, consider the alternative. A calculator program you write and run on your laptop exists entirely within a single **process** on a single machine. (A process is a running program; Chapter 3 defines it precisely.) The user interface, the logic and the data all share one memory and one execution environment. A function call is instant and cannot "partially fail".

Web applications do not work this way. A web application is fundamentally a **distributed system**: a system whose components run on different networked computers and coordinate by passing messages. It is one *logical* application whose parts are *physically* separated, and every interaction between those parts crosses a network.

This separation follows the **client–server model**, an architecture that divides software into two roles separated by a network boundary:

1. **The client** is the program that requests services or data. It handles the user interface and runs on the user's device (a web browser on a laptop, a native app on a phone).
2. **The server** is the program that provides services. It enforces business rules and security on infrastructure the operator controls. Persistent data is usually *not* kept inside the server process itself. It lives in dedicated stores (databases, caches) that the server talks to over the network.

### Why the Boundary Exists

If phones are as powerful as older servers, why not run the whole application on the user's device? The boundary exists to solve three problems:

- **Authoritative, shared state.** Millions of bank customers cannot each hold their own copy of the account balances. The system needs one authoritative source of truth that every user's actions modify.
- **Security and trust.** You cannot trust an environment you do not control. If the logic "deduct $100 from this account" ran on the client, a malicious user could modify it to deduct $0. Critical operations must run behind a boundary the user cannot tamper with.
- **Resource access.** The backend needs resources a client device must never hold: database credentials, third-party API keys (secret strings that identify your company to another company's service), other users' data, and computing capacity beyond a single device.

### A Server Is a Program, Not a Machine

Engineers often point at a machine in a rack and call it "a server". In software engineering this is a misleading habit.

A **server** is a *process*: a running program that asks the operating system to let it receive network connections on a particular **port** (a number, such as 443, that identifies which program on a machine should receive incoming traffic; Chapter 6). When we say "the server received a request", we mean the operating system handed incoming bytes on that port to that process.

You can run fifty server processes on one laptop as long as they use different ports. Conversely, one logical "server" application can run as thousands of identical processes on thousands of machines. In this book, **"server" means the software process handling requests** unless stated otherwise. When we mean the physical computer, we say *machine* or *host*.

---

## 1.2 The Request–Response Interaction

A **protocol** is an agreed set of rules governing how data is formatted, transmitted and interpreted between two systems. Without a shared protocol, a client and a server could exchange bytes but could not understand each other.

On the web, the client–server conversation is almost always governed by the **Hypertext Transfer Protocol (HTTP)**, an application-level protocol that defines the format of request and response messages, the meaning of operations (GET, POST, DELETE…) and the meaning of status codes (200 OK, 404 Not Found…). HTTP is covered in full in Chapter 7.

The basic interaction is the **request–response cycle**:

```text
┌────────────────┐                     ┌────────────────┐
│                │     1. REQUEST      │                │
│     CLIENT     │ ──────────────────► │     SERVER     │
│   (initiates)  │                     │  (processes)   │
│                │    3. RESPONSE      │                │
│                │ ◄────────────────── │  2. does work  │
└────────────────┘                     └────────────────┘
```

1. **Request:** the client sends a structured message stating what it wants ("give me the profile of user 42").
2. **Processing:** the server interprets the request, does the work (which may involve other systems) and produces a result.
3. **Response:** the server sends back a structured message with the result or an error indication.

Two consequences matter immediately:

- **"Client" and "server" are roles, not machines.** When your backend receives a request from a browser, it is a server. When it then queries a database or calls a payment provider, *it* is the client in a new request–response cycle. A single request from a user often fans out into many such cycles behind the scenes.
- **The client cannot see what happened in step 2.** If the response never arrives, the client cannot know whether the server did nothing, did half the work, or did everything and the response was lost. This "unknown outcome" problem drives much of the reliability engineering in Chapters 7, 13 and 24.

> **Try it.** You can watch a real request and response now. Open a terminal and run `curl -v https://example.com` (in Windows PowerShell, type `curl.exe`, because plain `curl` there is a different command). Lines starting with `>` are the request your machine sent. Lines starting with `<` are the response: a status line such as `HTTP/2 200`, then headers, then the page itself. Lines starting with `*` are curl's own notes about connecting. In a browser, press F12, open the Network tab and reload any page to see the same thing for every file the page loads.

---

## 1.3 The Frontend: Presentation and the Trust Boundary

The **frontend** is the part of the application that runs on the user's device, typically JavaScript, HTML and CSS in a browser, or a mobile app. It is responsible for presentation, user interaction and client-side navigation. The frontend's files reach the device by download, like any other response: the browser requests them from the backend or from a **CDN** (content delivery network: servers around the world that keep copies of files close to users). How the browser actually loads and runs them is the subject of Chapter 9.

The **backend** is the part of the application that runs on servers the operator controls: the processes that receive requests from frontends and other systems, apply the business rules, store and retrieve data, and talk to other services. Users never see it directly; they see only its responses. Frontend and backend meet at an API (Section 1.4), and everything behind that API (application servers, databases, caches, queues and the infrastructure in front of them) is the backend. That territory is the subject of this book. **Full-stack** engineering means working on both sides.

### The Rule of Client Trust

Because frontend code runs on hardware the user controls, **the frontend is an untrusted environment**. You must assume the "client" can be anyone, including an attacker sending hand-crafted requests with a command-line tool such as `curl` (Section 1.2).

The **trust boundary** is the line between code and data you control (the backend and its infrastructure) and code and data you do not (anything on the user's side, or anything arriving over the network). Everything that crosses it inward must be treated as potentially hostile.

Example: a frontend checks that a password has at least 8 characters before enabling the Submit button. This **client-side validation** exists purely for user experience: it gives instant feedback without a round trip. But an attacker can skip the page entirely and send this directly:

```http
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"email": "x@example.com", "password": "ab", "role": "admin"}
```

This is a raw HTTP request. The first line gives the method (`POST`), the path (`/api/users`) and the protocol version. The next lines are **headers**, `Name: value` pairs carrying information about the request: `Host` names the site, and `Content-Type` says the body is JSON. A blank line ends the headers, and the body follows. Chapter 7 covers the format in full.

The JavaScript check never ran. Notice the extra `"role": "admin"` field too: the attacker can send *any* data, not just the fields your form shows.

Therefore, **client-side validation cannot enforce anything.** The backend must independently:
- **validate** every input (types, lengths, ranges, allowed fields),
- **authenticate** every request (establish who is calling; Chapter 20),
- **authorize** every action (decide whether this caller may do this to this object; Chapter 20).

**What the trust boundary does not mean:** it does not mean the backend's internal network is automatically safe. Requests from other internal services also cross boundaries and can be forged or compromised (service-to-service identity is in Chapter 20).

---

## 1.4 The API: The Contract Between Systems

If frontend and backend are separate programs, possibly written by different teams in different languages, how do they agree on what to send? Through an API.

An **API (Application Programming Interface)** is a defined interface that specifies how one piece of software can request services from another. A **web API** is that contract expressed over HTTP: which operations exist, what the client must send, and what the server will return. The client relies only on the contract. It knows nothing about the classes, functions or SQL running behind it, which is what lets each side change its internals independently.

The vocabulary you will meet constantly:

- An **endpoint** is a URL identifying a resource or operation, e.g. `https://api.example.com/users/42`.
- An **HTTP method** states intent: `GET` to read, `POST` to create, `PUT`/`PATCH` to update, `DELETE` to remove (precise semantics in Chapter 7).
- **JSON (JavaScript Object Notation)** is a text format for structured data built from objects (key–value pairs) and arrays. It is the most common body format for web APIs.
- **Serialization** converts an in-memory data structure into a sequence of bytes that can be stored or transmitted (for example, a user object becomes the JSON text `{"id":42,"name":"Asha"}`). **Deserialization** is the reverse: parsing received bytes back into an in-memory structure.

Why serialization matters: memory addresses, object references and language-specific types are meaningless on another machine. Only bytes cross the network. The backend serializes its data into JSON; the frontend deserializes that JSON into JavaScript objects; a different client (a mobile app in Kotlin or Swift) deserializes the *same* bytes into its own types. Serialization formats and their security pitfalls are covered in Chapter 12. Designing APIs is Chapter 13.

**What a contract does not guarantee.** An API states what each side *should* send. Nothing forces either side to comply. Suppose the server renames the field `price` to `unitPrice` while an older mobile app still reads `price`. Depending on how strict its deserializer is, the app either fails with a parsing error or silently fills the missing field with a default value, and users see blank or zero prices. The symptom appears only on older client versions. The evidence is a difference between the bytes actually sent and the bytes the client expects, visible when you capture the response (for example with `curl`) and compare it with the documented contract. Changing a contract without breaking existing clients (versioning and schema evolution) is Chapter 13.

---

## 1.5 The Anatomy of the Backend: From Proxies to Code

A request to `api.example.com` does not land directly in the function you wrote. "The server" is usually a stack of specialised layers, each handling a distinct phase. A common backend can be understood through **three logical layers**. Not every system has all three as separate pieces, and some platforms (serverless functions, edge runtimes) collapse them, but the *roles* always exist somewhere:

1. **Edge layer (load balancer / reverse proxy).** The outer perimeter.
   - A **load balancer** distributes incoming requests across multiple identical backend **instances** so no single one is overwhelmed. (An instance here is one running copy of the backend process, not an object.) It stops sending traffic to instances that fail **health checks**: small requests, such as `GET /health`, that it sends every few seconds to ask each instance "are you working?".
   - A **reverse proxy** is a server that sits in front of application processes and receives requests on their behalf. It hides the internal layout from the internet and commonly performs **TLS termination**, request buffering, compression and routing. HTTPS is HTTP carried inside an encrypted channel called **TLS**; terminating TLS means decrypting it here, so the inner hops need not (Chapter 8).
   - The difference is the job, not the box. A load balancer *spreads* traffic across many copies; a reverse proxy *stands in* for the processes behind it. One program (Nginx, for example) often does both, and a request may pass through both: a cloud load balancer picks a machine, and a reverse proxy on that machine hands the request to the right process.
   - Mechanisms: Chapter 10. Common implementations include Nginx, HAProxy, Envoy and cloud load balancers.
2. **Application process (runtime / HTTP server).** The process that listens on a port, accepts network connections, reads raw HTTP bytes, parses them into a structured request, and hands that request to your code. You will also hear it called the **application server**, and in Java the *servlet container*: all three names mean this one role. It also decides *how* many requests run at once: with a pool of **threads** (independent sequences of execution inside one process, each able to handle one request), an event loop (one thread that switches between many requests while each waits), or several worker processes (Chapters 3, 10 and 11).
3. **Application code.** The logic you write: routing a request to the right handler, validating input, enforcing business rules, talking to databases and other services, and building the response (Chapter 12).

The generic mechanism is always: **accept connection → parse HTTP → dispatch to a handler → run logic → write response.**

> **Java lens.** In Java web applications the application-process role is often filled by a *servlet container* such as Apache Tomcat, which implements the Java Servlet specification. Spring Boot embeds Tomcat inside your application, so `java -jar app.jar` starts both. Your `@RestController` classes are the application-code layer.
>
> **Other runtimes.** Python apps typically use Gunicorn or Uvicorn as the application process, with Django/Flask/FastAPI code inside. Node.js has an HTTP server built into its runtime, with Express or Fastify handlers on top. Different names, same three roles.

**Deployment arrangements vary.** These layers used to be separate products on separate machines. Today the application process is often embedded in the application itself, and the edge may be a managed cloud service. The logical layers remain even when they are physically bundled. That distinction is the subject of Section 1.8.

---

## 1.6 Supporting Infrastructure

Your application code runs inside a process, and a process has memory: you can keep data in variables, lists and maps. But process memory has two properties that make it unsuitable as the home of important data:

- It is **ephemeral**: it is lost when the process crashes, restarts or is redeployed.
- It is **isolated**: it is not shared between the multiple instances behind a load balancer. User A's request may hit instance 1 and user A's next request instance 2.

Process memory is still useful for data you can afford to lose and rebuild, such as a local cached copy of something the database holds. It must never be the only copy.

So backends rely on additional components. Each solves a specific problem. They are named here so you can recognise them; their mechanisms are covered in the chapters indicated.

- **Relational database** (e.g. PostgreSQL, MySQL). It stores data in tables of rows and columns and is queried in the language SQL. It is the **system of record**: the authoritative store of application state. Work is grouped into **transactions**: a transaction is a set of reads and writes that the database applies all together or not at all, so a failure halfway leaves no half-done change behind. The database provides **durability** (committed data survives crashes and, when configured correctly, power loss) and **isolation** of concurrent modifications. This is a different sense from "isolated" above: it means two transactions running at the same moment do not see each other's unfinished work. Durability is not magic. It is implemented with logs, buffers and careful use of the operating system's file APIs (Chapter 4 and Chapter 15). SQL and modelling: Chapter 14. Indexes: Chapter 16. Transactions: Chapter 17. Integration with application code: Chapter 18.
- **Cache** (e.g. Redis, Memcached). A fast, usually in-memory store holding copies of data that is expensive to compute or fetch. It trades freshness for speed: a cache may serve data that is out of date. Chapter 21.
- **Message queue / log** (e.g. RabbitMQ, Kafka). Lets the application hand work to background workers (send an email, process a video) so the user does not wait, and lets services exchange events without calling each other directly. Here a **log** means an append-only list of messages. The word has other meanings in this book: the record of changes a database writes for durability, and the text lines an application prints about what it did. Context tells you which. Queues commonly deliver each message **at least once**: a message is never silently lost, but it may arrive twice, so workers must tolerate duplicates. Chapter 25.
- **External services** (e.g. payment providers, email services, object storage). Third-party APIs your backend calls over the network. You control neither their availability nor their latency. Chapters 13 and 24.

The following table is worth committing to memory, because every design and incident discussion comes back to it:

| Component | State it holds | Primary guarantee | When it fails or misbehaves |
|---|---|---|---|
| Browser / client | UI state, cookies, local storage | None you can rely on (untrusted) | Stale screens, tampered requests, offline |
| Edge (LB / proxy) | Little: connections, routing config | Distributes traffic, terminates TLS | All traffic stops, or 502/503/504 errors (Chapter 10) |
| Application instance | Ephemeral, per-instance memory | Runs your logic | Crash loses in-flight work; overload causes timeouts |
| Database | Durable system of record | Committed data persists; transactions are isolated | Writes fail, queries slow, connections exhausted |
| Cache | Copies of data, possibly stale | Speed, not correctness | Stale reads; on outage, load floods the DB |
| Queue / log | Messages awaiting processing | Delivery, commonly configured as at-least-once (never exactly-once processing by itself; Chapter 25) | Backlog grows, duplicates, out-of-order processing |
| External service | Their data, not yours | Whatever their contract says | Slow, down, or succeeds while you see a timeout |

> **Why this matters:** interviews and outages start with the same question: *what does each piece guarantee, and what happens when it fails?* This book exists to let you answer that for every row.

---

## 1.7 State Boundaries and Failure Boundaries

A web application is not one program with one memory. It is **several independently running components, each with its own state, talking across network boundaries.**

```text
 Browser memory (DOM, JS objects, cookies)   ← client state; untrusted
        │  NETWORK: HTTP bytes, round-trip time, loss, timeouts
        ▼
 Application process (memory, threads)       ← ephemeral; one of N instances
        │  NETWORK: connections to other processes
        ├──────────────► Database process     ← durable system of record
        ├──────────────► Cache process        ← fast, deliberately allowed to be stale
        └──────────────► External API         ← not yours; fails independently
```

A **state boundary** is a point where one component's memory ends and another's begins: data on one side is not automatically visible or consistent on the other. A **failure boundary** is a point where one side can fail while the other keeps running.

**A function call inside one process is a fundamentally different operation from a call across a network.** Inside a process you share memory, errors propagate up the call stack, and a crash takes everything down together, so there is no half-state between caller and callee. Across a boundary you get four new properties:

1. **Serialization:** data becomes bytes (Section 1.4). The receiver gets a copy, not a shared object.
2. **Latency:** at least one network round trip, often several (Chapter 6). A local call takes nanoseconds; a network call takes from hundreds of microseconds inside a data centre to hundreds of milliseconds across continents.
3. **Independent failure:** the other side can be down, slow, or overloaded, or can *succeed while your response is lost*. A timeout does not tell you what happened (Chapter 24).
4. **Independent state:** the database can have committed while the cache still holds the old value. The browser can show a confirmation the server never stored.

This is why later chapters spend so much effort on transactions (Chapters 17 and 18), cache invalidation (Chapter 21), retries and idempotency (Chapters 13 and 24), and consistency between copies of data (Chapter 22). If you remember one sentence from this chapter besides the trust boundary, make it this: **every box in the diagram is a different process, and the arrows are where systems actually break.**

---

## 1.8 Logical Architecture vs. Physical Deployment

**Logical architecture** describes roles and the flow of data and responsibility: a request reaches a load balancer, is routed to an application instance, which consults a cache, falls back to the database, and perhaps publishes an event to a queue.

**Physical deployment** describes where those roles actually run: which machines, virtual machines or containers, in which data centres.

The same logical architecture can be deployed very differently:

- **Local development:** the application, database, cache and queue run as separate processes on one laptop.
- **Small production system:** the application runs on two virtual machines behind a managed load balancer, and the database is a managed service with one standby copy.
- **Large production system:** hundreds of application instances in containers across several data centres, a database cluster with replicas in multiple regions, and a global edge network in front.

```text
                         ┌──────────────────── Edge (Ch 10) ───────────────────┐
  Users ──DNS(Ch 6)──►   │  CDN (Ch 10, 21) ──► Load balancer ──► Reverse proxy │
  Browser (Ch 9)         └──────────────────────────────┬──────────────────────┘
  HTTP (Ch 7), TLS (Ch 8)                               │
                                 ┌──────────────────────┼───────────────────────┐
                                 ▼                      ▼                       ▼
                         App instance 1          App instance 2   ...   App instance N
                         (process: Ch 3–5, server internals: Ch 10–11, code: Ch 12–13)
                                 │   │   │                 auth: Ch 19–20
              ┌──────────────────┘   │   └───────────────────────┐
              ▼                      ▼                           ▼
        Cache (Ch 21)        Database primary (Ch 14–18)     Queue/log (Ch 25)
                                     │ replication (Ch 22)       │
                                     ▼                           ▼
                             Database replicas            Background workers (Ch 25)
                             partitioning (Ch 23)         External APIs (Ch 13, 24)

     Cross-cutting: failure & resilience (Ch 24), deployment & observability (Ch 26)
```

The logical boundaries (for example, the network hop between application and database) exist in every deployment. Physical deployment determines their **latency, failure independence and capacity**. The hop costs tens of microseconds over the loopback interface on a laptop, and roughly 0.1–1 ms across a data-centre network. The two ends fail together on a laptop and independently in production. That is why a system that works perfectly on a laptop can fail in production: the boundaries were always there, but on the laptop they were too fast and too reliable to notice.

Deployment, containers and scaling are covered conceptually in Chapters 4 and 26.

---

## 1.9 Monolith vs. Services: A First Look

As an application grows, an organisation must decide how to divide the *application code* itself.

- A **monolith** is an application whose business capabilities (users, orders, payments, catalogue…) are built and deployed as **one deployable unit**, usually sharing one database. To **deploy** is to package a new version of the code, copy it to the servers and start it there in place of the old version. Calls between capabilities are in-process function calls.
- A **microservice architecture** splits capabilities into **separately deployable services**, each typically owning its own data, communicating over the network (HTTP/gRPC calls or messages).

The basic trade-off follows directly from Section 1.7:

| | Monolith | Microservices |
|---|---|---|
| Calls between capabilities | In-process: fast; fail together | Network: slow; fail independently; unknown outcomes |
| Data consistency across capabilities | One database transaction can cover them | Requires distributed patterns (sagas, outbox; Chapter 25) |
| Deployment and scaling | Deployed as one unit; scaled by running more copies of the whole application | Each service deployed and scaled on its own |
| Team autonomy at large scale | Harder | Easier |
| Operational complexity | Lower | Much higher (discovery, tracing, versioning) |

A monolith scales well: you run many identical instances behind a load balancer. What it cannot do is scale or release *one part* alone. The team row matters more than it looks. With 50 engineers in one deployable unit, every change waits for one shared release and one shared test run. Separate services let each team ship on its own schedule, and in practice that, rather than speed, is the usual reason to split.

Splitting a monolith converts function calls into network calls, and therefore takes on *every* failure mode of Section 1.7. The full discussion, including the "modular monolith" middle ground and the "distributed monolith" failure, is in Chapter 12. For now, recognise both terms and the trade-off.

---

## 1.10 What Happens When You Click "Buy": Your Roadmap

This is the most important section of the chapter, but not because you should understand it now. **You do not need to understand these steps yet.** Read it once for the shape of the journey and expect most terms to be new. Each step maps to a mechanism explained in the later chapter named in its right-hand column.

The machinery is the same on every stack: **browser → DNS → TCP → TLS → HTTP → edge → application process → application code → database → external services and queues → response → browser.** The steps below are described generically. A Framework lens box afterwards shows the names a Java/Spring Boot backend gives the same steps.

Scenario: a signed-in user on a shop's product page clicks **Buy** for product 42. The page's JavaScript sends this request:

```http
POST /api/orders HTTP/1.1
Host: api.example.com
Cookie: session=7f3a...
Idempotency-Key: 9b1c2d4e-...
Content-Type: application/json

{"productId": 42, "quantity": 1}
```

Two headers are new. `Cookie` carries a **session** identifier: a random token the server gave this browser at sign-in, so it can recognise later requests as coming from the same user. The browser sends it back automatically on every request to the site (Chapter 19). `Idempotency-Key` is explained in step 1.

Notice what the body does *not* contain: the price. The client says what it wants to buy; the server decides what it costs (Section 1.3).

```text
 Browser ──(1–6: DNS, TCP, TLS, HTTP request)──► Edge: load balancer + reverse proxy (7–8)
                                                      │
                                                      ▼
                                     App instance: parse, middleware, route,
                                     validate, check idempotency key (9–14)
                                                      │
                     ┌────────────────────────────────┼──────────────────────────────┐
                     ▼                                ▼                              ▼
    Database: BEGIN; lock the stock      Payment API: charge, with          Cache: drop the stale
    row; insert order + outbox row;      a timeout and the                  stock entry (24)
    COMMIT (15–22); record the           idempotency key (23)
    payment result (24)
                     │
                     ▼  (after the response, asynchronously)
    Outbox relay ──► Queue/log ──► Worker sends the "order received" email (31–32)

 Response: App ──201 Created + order status (25–28)──► Edge ──(29)──► Browser shows that status (30)
```

### Phase 1: The client and the network (Chapters 6–9)

| Step | What happens | Chapter |
|---|---|---|
| 1 | The user clicks Buy. The page's JavaScript builds the request: a JSON body, the session cookie (attached by the browser), and a freshly generated **idempotency key**, a unique ID for this purchase attempt so that a retry of the same attempt cannot create a second order. | 9, 13 |
| 2 | The browser knows (from a previous HSTS header, the site's instruction "always use HTTPS with me") that this site requires HTTPS, so it never tries plain HTTP. | 8 |
| 3 | The browser needs an **IP address** for `api.example.com`: the numeric network address of the machine, such as `93.184.215.14`. **DNS**, the internet's directory of names, provides it. The browser checks its own cache, then the OS asks a recursive DNS resolver, which may query the root, top-level-domain and authoritative servers. | 6 |
| 4 | A **TCP connection** (a reliable two-way stream of bytes between two programs) to the server's IP on port 443 is established: SYN → SYN-ACK → ACK. One round trip before any data flows. | 6 |
| 5 | A TLS 1.3 handshake (TLS is the encryption under HTTPS) verifies the server's certificate and agrees encryption keys. Typically one more round trip. | 8 |
| 6 | The HTTP request is sent over the encrypted connection. | 7 |

If the browser already has an open connection to this host, steps 3–5 are skipped entirely. **Connection reuse** is a major latency optimisation (Chapters 6 and 7).

### Phase 2: Entering the server (Chapters 4, 5, 10, 11)

| Step | What happens | Chapter |
|---|---|---|
| 7 | The load balancer chooses a healthy application instance. | 10 |
| 8 | The reverse proxy (if TLS was terminated here) forwards plain HTTP to the application process, adding headers such as the client's original IP. | 10 |
| 9 | The operating system's network stack receives the bytes and wakes the application process. The server parses the HTTP and assigns the request to a thread (or to an event loop). | 4, 5, 10, 11 |
| 10 | **Middleware** (code the server runs before and after every handler) performs the cross-cutting checks: CORS, session lookup from the cookie, a CSRF check (this is a cookie-authenticated POST), authentication and authorization. | 9, 10, 19, 20 |
| 11 | The router matches `POST /api/orders` to a handler function. | 10 |

### Phase 3: Application logic (Chapters 12–13)

| Step | What happens | Chapter |
|---|---|---|
| 12 | The handler deserializes the JSON and validates it: the quantity is a positive integer, the product exists, and there are no unexpected fields. | 12 |
| 13 | The idempotency key is looked up. If this key was already used, the stored result of the first attempt is returned and nothing new is created. | 13 |
| 14 | A database transaction is started, either explicitly or by a framework wrapper around the service function. | 17, 18 |

### Phase 4: Data persistence (Chapters 14–18, 25)

| Step | What happens | Chapter |
|---|---|---|
| 15 | An already-open database connection is borrowed from a **connection pool**, avoiding a new TCP + authentication handshake. | 18 |
| 16 | The statements are sent with values as separate **parameters**, never pasted into the SQL text (this is what prevents SQL injection). The price is read from the database, not from the request. | 14, 18 |
| 17 | The database parses each statement, and the optimizer chooses a plan (here, primary-key index lookups). | 16 |
| 18 | The executor walks the indexes and fetches pages through the **buffer pool** (in memory if cached, from disk if not). | 15, 16 |
| 19 | The stock update takes a **row lock**. If two users buy the last unit at the same moment, the second waits, then finds `stock = 0` and updates nothing. Plain reads elsewhere are not blocked (**MVCC**). | 17 |
| 20 | An `OrderPlaced` event row is inserted into an **outbox** table in the *same* transaction, so the event exists if and only if the order does. Publishing straight to the queue after the commit would be unsafe: a crash between the two leaves an order whose event, and therefore whose email, is lost. | 25 |
| 21 | `COMMIT`: the database writes its log record to durable storage before it reports success. | 4, 15 |
| 22 | The connection is returned to the pool for the next request. | 18 |

This is SQL, the database's query language (Chapter 14). Data lives in **tables** of rows. `SELECT` reads rows, `UPDATE` changes them and `INSERT` adds new ones; `WHERE` chooses which rows. `$1`, `$2` and `$3` are the separate parameters of step 16.

```sql
-- step 16, in outline ($1 = product id, $2 = quantity, $3 = user id)
SELECT price FROM products WHERE id = $1;
UPDATE inventory SET stock = stock - $2 WHERE product_id = $1 AND stock >= $2;  -- 0 rows → out of stock
INSERT INTO orders (user_id, product_id, quantity, unit_price, status)
     VALUES ($3, $1, $2, <price read above>, 'PENDING');  -- order exists; payment not yet confirmed
INSERT INTO outbox (event_type, payload) VALUES ('OrderPlaced', '{"orderId": ...}');
```

### Phase 5: The payment and the response (Chapters 7, 9–13, 21, 24)

| Step | What happens | Chapter |
|---|---|---|
| 23 | The backend, now acting as a *client*, calls the payment provider over HTTPS with a timeout and the same idempotency key. If the call times out, the outcome is unknown (Section 1.2): the card may or may not have been charged. | 13, 24 |
| 24 | A second, short transaction records what step 23 learned (`PAID`, `PAYMENT_FAILED`, or no change if the outcome is unknown; see the table below), and the cached stock count for product 42 is invalidated. | 17, 21 |
| 25 | The internal order object is mapped to a response shape (a **DTO**, data transfer object) so the API contract stays independent of the database schema, then serialized to JSON. | 12 |
| 26 | The server writes `201 Created`, a `Location` header naming the new order, and the JSON body, which includes the order's `status`. `201` means only that the **order resource now exists**; whether it is paid is in `status`. | 7, 10 |
| 27 | The response passes back through middleware (e.g. adding headers, logging). | 10 |
| 28 | The bytes are written to the connection. The thread (or event-loop slot) is free for the next request. | 11 |
| 29 | The proxy and load balancer relay the response to the client. | 10 |
| 30 | The browser parses the JSON, and JavaScript shows a message that matches `status` (table below), not a blanket "Order placed". | 9 |

"Order placed" hides three separate facts, which become true at different times and are recorded separately:

- **Order created:** the order row exists, with the price the server chose (step 21's commit). The `201` reports this.
- **Payment confirmed:** the provider has said the charge succeeded (steps 23–24). Only then is the status `PAID`.
- **Order fulfilled:** the goods have been shipped. A different process records this later; nothing in this request does.

What the client receives depends on how step 23 ended:

| Payment call (step 23) | Status recorded (step 24) | Response body | What the user sees |
|---|---|---|---|
| Provider confirms the charge | `PAID` | `201`, `"status": "PAID"` | "Order confirmed" |
| Provider declines the card | `PAYMENT_FAILED`; the reserved stock is returned | `201`, `"status": "PAYMENT_FAILED"` | "Payment declined; try another card" |
| Timeout or lost connection: **outcome unknown** | Stays `PENDING`. A background job later asks the provider about this idempotency key and moves the order to `PAID` or `PAYMENT_FAILED` | `201`, `"status": "PENDING"` | "Order received; we are confirming your payment" |

The third row is the honest answer to an unknown outcome. Telling the user "paid" could be false, and telling them "failed, try again" could lead to a second charge. Which status codes and response shapes an API should use for each case is a design decision covered in Chapter 13; resolving unknown outcomes is covered in Chapters 24 and 25.

### Phase 6: After the response (Chapter 25)

| Step | What happens | Chapter |
|---|---|---|
| 31 | A relay process reads the outbox row and publishes `OrderPlaced` to a message queue or log. | 25 |
| 32 | A background worker consumes the event and sends an "order received" email. (`OrderPlaced` means the order was created, not that it was paid; a "payment confirmed" email would be triggered by a separate event written when the order becomes `PAID`.) The worker can receive the same event twice (at-least-once delivery), so the send must be safe to repeat. | 25 |

Why is the card charged *after* the commit rather than inside the transaction? Holding a transaction, and its row lock, open across a slow call to a third party would make every other buyer of that product wait. Ordering these steps safely, and recovering when step 23 times out, is the subject of Chapters 13, 17, 24 and 25.

> **Framework lens: the same trace in Java/Spring Boot.** Step 9: embedded Tomcat takes a worker thread from its pool. Steps 10–11: Spring Security's filter chain runs, then Spring's `DispatcherServlet` routes to a `@RestController` method. Step 12: the Jackson library deserializes the JSON and Bean Validation annotations check it. Step 14: the `@Transactional` annotation causes a proxy object to begin a transaction (Chapter 12 explains proxies). Steps 15–16: JPA/Hibernate generates SQL, HikariCP provides the pooled connection, and JDBC sends a `PreparedStatement`. Step 25: Jackson serializes the DTO. These are *names for the steps*, not definitions of them. A Python/FastAPI or Node/Express backend performs the same steps under different names.

**This trace is your map.** Every term in it (DNS, TCP, TLS, middleware, idempotency key, connection pool, buffer pool, row lock, MVCC, outbox) is explained at mechanism level in the chapter shown. When you finish the book, return to this page. Every step should be something you could explain and debug. Chapter 27 repeats this trace with full depth and uses it for systematic diagnosis.

---

## 1.11 How to Read This Book

**Order.** The 27 chapters build on each other and should be read in order. The book starts at the bottom of the stack, with how a program runs on hardware and the operating system (Chapters 2–5). It then moves up through the network and protocols (6–9), the server and application (10–13), data (14–18), identity and security (19–20), distributed data and performance (21–23), and reliability and operations (24–26), ending with a synthesis (27).

**How each concept is taught.** Every major concept follows the same seven steps:
1. a precise **definition** you could say in an interview;
2. **why it exists**: the problem that forced it into being;
3. **how it works internally**: state, steps, interactions;
4. a **concrete** diagram, trace or worked example;
5. its **guarantees and non-guarantees**;
6. how it **fails** and how you would **diagnose** it;
7. how it shapes **real backend engineering**.

**Languages and frameworks.** Concepts are defined independently of any language. Examples use pseudocode, SQL and protocol traces. Where it helps, **Java lens**, **Framework lens** and **Other runtimes** boxes show how Java, Spring Boot, Python or Node.js implement the mechanism. The goal is that, after this book, a framework feels like a convenient way of doing things you already understand.

**How to use the end-of-chapter material.**
- **Interview Drill:** answer each question *aloud* before reading the "strong answer" points. If you can only recite a definition, you have not finished the chapter.
- **Scenario Exercises:** unfamiliar situations. Reason them out from mechanisms before reading the solution sketch.
- **Diagnosis Lab:** practice going from symptom to evidence to cause, which is the core skill of operating backends.

**What this book is not.** It is not a framework tutorial, a DSA book or a system-design interview guide. It builds the foundations those depend on. Implementation practice, projects and framework study should follow it and will go much faster because of it.

The next chapter starts at the very bottom: what actually happens when a program runs on a CPU, and how memory, storage and language runtimes behave.

---

## Chapter Summary

A web application is a **distributed system** built on the **client–server model**. The client (browser or app) handles presentation and interaction but runs in an **untrusted** environment, so the backend must validate, authenticate and authorize everything independently. Client and server communicate through an **API**: a contract of endpoints, methods and data formats (usually JSON over HTTP), with data **serialized** into bytes to cross the network.

The backend is a composite of logical layers: an **edge** (load balancer, reverse proxy) that distributes and protects traffic; an **application process** that accepts connections, parses HTTP and schedules requests onto threads or an event loop; and **application code** that implements the business logic. Because process memory is **ephemeral and per-instance**, durable state lives in a **database**. **Caches** add speed at the cost of possible staleness, **queues** move slow work off the request path, and **external services** add dependencies you do not control.

Every component is a separate process with its own state. Every arrow between them is a **network boundary** that adds serialization, latency, independent failure and independent state. The logical architecture stays constant while physical deployment changes the latency and failure characteristics of those boundaries. Splitting an application into microservices turns function calls into network calls and inherits all of these properties.

A single click on Buy travels through DNS, TCP, TLS and HTTP, the edge, the server and its middleware, application logic and an idempotency check, a pooled database connection, a transaction that locks a stock row and is made durable by the database's log, a call to a payment provider whose timeout leaves the outcome unknown, and a queue that hands follow-up work to a background worker. The rest of this book explains each of those steps.

## Guarantees vs. Non-Guarantees

| Component / idea | Guarantees | Does NOT guarantee |
|---|---|---|
| Client-side validation | Faster feedback for honest users | Anything about what the server receives |
| Process memory | Fast access within one instance | Survival across restarts, or visibility to other instances |
| Database (committed data) | Durability and transactional isolation (when configured correctly) | That the cache, client or other services see the change immediately |
| Cache | Lower latency for repeated reads | Freshness or correctness |
| A network call | That *if* a response arrives, you learn what the component that sent it decided (which may be a proxy rather than the service; see Lab 2) | That a missing response means the operation failed: the outcome is unknown |
| Load balancer | Spreads requests over healthy instances | That consecutive requests from one user hit the same instance |

## Interview Drill

**Q1. What is the client–server model, and why don't we run everything on the client?**
A strong answer must mention:
- Two roles separated by a network: the client requests, the server provides.
- The need for authoritative shared state (one source of truth for all users).
- The client is untrusted: logic and rules there can be modified.
- Secrets and resources (DB credentials, API keys, other users' data) cannot live on clients.
*Common weak answer:* "Because servers are more powerful." Power isn't the reason; trust and shared state are.

**Q2. Why isn't client-side validation sufficient?**
A strong answer must mention:
- The client runs on hardware the user controls. Requests can be crafted without the UI.
- Client validation is a UX feature (instant feedback).
- The server must re-validate inputs, authenticate and authorize every request, including fields the form never shows.
*Common weak answer:* "Because JavaScript can be disabled." True but shallow: the real point is that any request can be forged.

**Q3. What is an API? What are serialization and deserialization, and why are they needed?**
A strong answer must mention:
- An API is a contract: operations, inputs, outputs, errors. It decouples client from implementation.
- Only bytes cross a network. In-memory objects and references are meaningless elsewhere.
- Serialization: in-memory structure → bytes (e.g. JSON). Deserialization: bytes → structure in the receiver's language.
*Common weak answer:* "Serialization converts to JSON." JSON is one format; the concept is structure ↔ bytes.

**Q4. Distinguish a load balancer, a reverse proxy, an application server process and application code.**
A strong answer must mention:
- LB: distributes requests across instances and uses health checks.
- Reverse proxy: fronts application processes; does TLS termination, buffering, routing, header handling.
- Application process: accepts connections, parses HTTP, schedules work onto threads/event loop, dispatches to handlers.
- Application code: routing to handlers, validation, business rules, persistence.
- The roles may be combined in one product or process.
*Common weak answer:* naming products ("Nginx, Tomcat") without stating the roles.

**Q5. Why shouldn't important data live in the application process's memory?**
A strong answer must mention:
- Memory is ephemeral: lost on crash, restart or deploy.
- Memory is per-instance: other instances behind the LB cannot see it.
- A database gives durability and controlled concurrent access.
- In-memory data is acceptable only as a cache, i.e. a copy that can be lost.
*Common weak answer:* "Because RAM is limited." Even infinite RAM would not fix ephemerality or isolation.

**Q6. How is a call to another service different from a function call in the same process?**
A strong answer must mention:
- Serialization: copies of data, not shared references.
- Latency: orders of magnitude slower.
- Independent failure, including the *unknown outcome* after a timeout.
- Independent state: the two sides can disagree.
*Common weak answer:* "It's just slower." Speed is the least important difference.

**Q7. What does "client and server are roles, not machines" mean? Give an example.**
A strong answer must mention:
- A role is defined by who initiates a request–response.
- A backend is a server to browsers and a client to its database, cache and third-party APIs.
- One user request often triggers a tree of client calls.
*Common weak answer:* "The server is the big machine in the data centre." That confuses a host with a role: the same process is a client and a server within one request.

**Q8. What is the difference between logical architecture and physical deployment? Why does it matter?**
A strong answer must mention:
- Logical: roles and data flow. Physical: machines/containers/regions running them.
- The same logical design can run on a laptop or across regions.
- Physical deployment determines the latency, failure independence and capacity of each boundary.
- "Works on my laptop" bugs often come from boundaries that were too fast or too reliable locally to notice.
*Common weak answer:* treating them as the same thing ("our architecture is three servers").

**Q9. What is the basic trade-off between a monolith and microservices?**
A strong answer must mention:
- Monolith: one deployable unit, in-process calls, a single database transaction can span capabilities, simpler to operate.
- Microservices: independent deployment/scaling/ownership, but calls become network calls with every failure mode, and consistency across services needs distributed patterns.
- The choice is driven by team scale and boundaries, not fashion.
*Common weak answer:* "Microservices are more scalable." A monolith can also scale horizontally; the main gain is organisational and deployment independence.

**Q10. At a high level, what happens when a user clicks "Buy"?**
A strong answer must mention:
- DNS → TCP → TLS → HTTP request (and that these are skipped on connection reuse). The request says *what* to buy; the server looks up the price.
- Edge (LB/proxy) → application process → middleware (session, CSRF, authentication, authorization) → handler → validation → idempotency-key check.
- One transaction: a conditional stock update under a row lock, the order insert and an outbox event, then a commit made durable by the database's log.
- The payment provider is called after the commit, with a timeout and the idempotency key; a timeout means the outcome is unknown.
- `201 Created` means the order exists; its `status` says whether payment is confirmed (`PAID`), declined, or still `PENDING` because the outcome is unknown. The email is sent later by a worker that must tolerate duplicate events.
*Common weak answer:* "The browser sends the order to the server, which saves it in the database." It skips the network setup, the trust boundary (who decides the price), duplicate protection and the unknown outcome of the payment call.

**Q11. What is a trust boundary?**
A strong answer must mention:
- The line between components/data you control and those you do not.
- Everything crossing inward is validated, authenticated and authorized.
- Internal networks are not automatically trusted.
*Common weak answer:* "The firewall." A firewall is one control at one network edge; the trust boundary is wherever data you do not control enters, including requests from inside your own network.

**Q12. Why does a cache appear in so many architectures, and what is its cost?**
A strong answer must mention:
- It serves repeated reads faster and reduces DB load.
- It holds copies, which can be stale. It is not the source of truth.
- It adds failure modes (stale data; the DB is flooded if the cache goes down).
*Common weak answer:* "A cache makes everything faster" with no mention of staleness.

**Q13. What is the backend, and how does it differ from the frontend?**
A strong answer must mention:
- Frontend: runs on the user's device (browser or mobile app); presentation and interaction; untrusted.
- Backend: runs on servers the operator controls; business rules, data storage, integration with other services; reached through an API.
- The trust boundary between them: the backend re-validates everything it receives (Section 1.3).
- Backend concerns are concurrency, durability, security, scaling and failure handling: the subject of this book.
*Common weak answer:* "The backend is the database." It is the whole server side: application processes, data stores, caches, queues and the infrastructure in front of them.

## Scenario Exercises

1. A team stores logged-in users' shopping carts in a map inside the application process. With one instance it works. After scaling to three instances behind a load balancer, users report items "randomly disappearing" from carts. Explain exactly why, using Sections 1.5–1.7.
2. A payment request from your backend to a payment provider times out after 10 seconds. Your code shows the user "Payment failed, please try again." Why might this be dangerous? What extra information would you need?
3. A mobile app and a web frontend both use your API. The mobile team hides the "discount" field in their UI for non-premium users. A security review flags this. What is the problem, and where must the fix live?
4. An application works perfectly on a developer laptop (app, DB and cache all local) but, in production, a page that makes 200 small database queries one after another takes more than a second. Using Section 1.8, explain why.

### Solution Sketches

1. Each instance has its own memory (state boundary). The LB sends a user's requests to different instances, and each sees a different (or empty) cart. Fix: store carts in a shared store (database or shared cache); detailed options in Chapter 19.
2. A timeout means *unknown outcome* (Section 1.2): the provider may have charged the card and the response was lost. Retrying could double-charge. You need the operation's actual status (query the provider) and an idempotency key so a retry cannot apply twice (Chapters 13 and 24).
3. Hiding a field is a client-side control. Anyone can send the request with a discount. The server must authorize whether this caller may apply a discount (Chapter 20).
4. Locally, each DB round trip is a loopback call costing tens of microseconds, and a small development data set keeps each query fast: 200 queries finish in a few milliseconds. In production, assume each query costs about 1 ms of network round trip plus 4 ms of execution against production-sized tables: 200 × 5 ms ≈ 1 s, before any other work. The boundary always existed; physical deployment made it expensive. This pattern (the "N+1 query" problem) is in Chapter 18.

## Diagnosis Lab

**Lab 1: "The site shows an old price after we updated it."**
- *Symptom:* an admin updates a product's price. The database shows the new price, but users see the old one for several minutes.
- *Hypotheses:* (a) the browser cached the response; (b) a CDN/edge cache cached it; (c) the application cache (e.g. Redis) holds the old value; (d) some instances read from a lagging database replica.
- *Evidence:* compare a request with caching bypassed (a hard refresh or `curl`) against a normal one. Inspect response headers (`Cache-Control`, `Age`; Chapter 7). Query the app cache key directly. Check which DB node served the read.
- *Root cause pattern:* a copy of the data exists at some boundary and nothing invalidated it.
- *Fix/prevention:* invalidate or version the cached copy on update (Chapter 21), set explicit HTTP caching headers (Chapter 7), and route read-after-write to the primary (Chapter 22).

**Lab 2: "Every request returns 502 Bad Gateway."**
- *Symptom:* all API calls return 502. The application team says "our app isn't logging any errors."
- *Hypotheses:* the 502 is generated by the edge (proxy/LB), not the app. The proxy cannot get a valid response from any instance: instances crashed, are listening on a different port, or are failing health checks.
- *Evidence:* the error page or headers identify which component produced the 502. The proxy's error log records why the upstream failed (in Nginx, for example, a line containing `connect() failed (111: Connection refused) while connecting to upstream`). Check whether application processes are running and listening (Chapters 4 and 10).
- *Root cause pattern:* "no errors in the app logs" often means the request never reached the app. Locate *which box* produced the error before debugging code.
- *Fix/prevention:* restore or replace the instances, correct the upstream configuration, and add health-check alerting (Chapters 24 and 26).

---

**END OF CHAPTER 1**
