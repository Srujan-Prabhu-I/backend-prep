# Chapter 18: Application–Database Integration

## Where We Are

Chapters 14 to 17 looked at the database from the inside. Chapter 14 defined what a query means, Chapter 15 showed how a commit becomes durable, Chapter 16 showed how the database finds rows quickly, and Chapter 17 showed how concurrent transactions are kept apart. In all four chapters the client was a person typing SQL into a console. In production it almost never is. The client is your application: a process with a pool of worker threads (Chapter 11), layered code (Chapter 12), and a request deadline measured in milliseconds.

This chapter covers the space between the two. Every query an application runs crosses a stack of machinery that it rarely sees: a **driver** that speaks the database's wire protocol, a **connection pool** that lends out a small number of expensive connections, perhaps a **server-side pooler** in between, **prepared statements** that separate SQL from data, and often an **ORM** that turns objects into rows and decides on its own when to send SQL. Most production database incidents are born here rather than inside the database: the pool runs dry, an endpoint issues 300 queries instead of 3, a transaction stays open across a slow HTTP call, a migration locks a table, or a value typed by a user becomes part of a SQL statement.

The chapter starts at the bottom, with the bytes a driver sends to connect and query, and builds up: the cost of a connection, pools and how to size them, poolers such as PgBouncer, prepared statements and SQL injection, ORM internals (identity map, unit of work, dirty checking, lazy loading), the N+1 problem, transaction boundaries in application code, application-level lost updates, batching and streaming, schema migrations without downtime, and read/write splitting. It ends with the evidence that ties each failure to its symptom in metrics and logs.

**After this chapter you can:**
- explain what a database driver does and trace a connection and a query through the PostgreSQL and MySQL wire protocols;
- break down the cost of opening a database connection and compare it with borrowing one from a pool;
- explain how a connection pool works (borrow, return, reset, validation, timeouts, leak detection) and size it with Little's Law and the database's core count;
- choose between session, transaction and statement pooling in a server-side pooler, and name what each mode breaks;
- explain how prepared statements and parameter binding work, and why they stop SQL injection, including the ORDER BY, LIKE and second-order variants;
- explain how an ORM maps objects, tracks changes (identity map, unit of work, dirty checking), decides when to flush, and loads associations lazily through proxies;
- detect and fix N+1 queries with joins, batch fetching and projections;
- decide where transactions begin and end in application code, use propagation and read-only transactions correctly, and keep remote calls out of transactions;
- choose among the Chapter 17 fixes for lost updates in ORM-based code;
- batch writes, run bulk operations and stream large results without exhausting memory;
- write versioned migrations, change a schema with zero downtime using expand/contract, and avoid locking DDL outages;
- explain read/write splitting and the read-your-writes problem caused by replica lag;
- diagnose pool exhaustion, N+1 slowdowns, leaked transactions and migration lock outages from their symptoms.

**Interview questions this chapter prepares you for:**
1. What does a database driver do? What happens when an application connects to PostgreSQL?
2. Why is opening a database connection expensive? Why does PostgreSQL in particular need pooling?
3. How does a connection pool work, and how do you size it?
4. What is PgBouncer? What are session, transaction and statement pooling, and what breaks in transaction mode?
5. What is a prepared statement? What does the database reuse, and what can go wrong with plan reuse?
6. What is SQL injection, and why do prepared statements prevent it? Can you inject through ORDER BY or LIKE? What is second-order injection?
7. What is an ORM? What is the object–relational impedance mismatch?
8. How does an ORM track changes? What are the identity map, the unit of work, dirty checking and flushing?
9. What is lazy loading, how is it implemented, and what is a LazyInitializationException?
10. What is the N+1 problem? How do you detect it, and what are three ways to fix it?
11. Where should transaction boundaries go? What do propagation levels such as REQUIRED and REQUIRES_NEW do?
12. Why should you never call a remote service inside a database transaction?
13. Why does `@Transactional` sometimes do nothing?
14. How do you prevent lost updates when using an ORM?
15. How do you insert a million rows efficiently? How do you read a table too large to fit in memory?
16. How do you rename a column with zero downtime? Why can a one-line `ALTER TABLE` take a site down?
17. What is read/write splitting, and what is the read-your-writes problem?

**Scope.** Here: drivers and wire protocols, connection cost, client-side pools and their sizing, server-side poolers, prepared statements, SQL injection, ORM internals, lazy loading, N+1, transaction boundaries and propagation in code, application-level lost updates, batching and streaming, schema migrations, read/write splitting, and a placement-round section on JDBC, cursors and embedded SQL (Section 18.18). Recalled, not re-taught: the database's process model (Chapter 3, Section 3.9; Chapter 15, Section 15.1), Little's Law and pools as bottlenecks (Chapter 11, Sections 11.3 and 11.5), proxies, AOP and the self-invocation pitfall (Chapter 12, Section 12.5), repositories and layers (Chapter 12, Section 12.2), query processing and plan caching (Chapter 16, Section 16.8), transactions, locks and the four oversell fixes (Chapter 17), API expand-and-contract (Chapter 13, Section 13.9). Elsewhere: replication, lag and failover in depth are Chapter 22; caching query results is Chapter 21; the outbox pattern and change data capture are Chapter 25; timeouts, retries and bulkheads as resilience patterns are Chapter 24; deployment strategies are Chapter 26.

---

## 18.1 The Driver and the Wire Protocol

### Definitions

**Application–database integration** is everything that happens between application code deciding "I need this data" and the database executing SQL: opening and reusing connections, encoding queries and parameters, decoding results, mapping rows to objects, and deciding where transactions begin and end. The code that does this is usually called the application's **persistence layer** (or data-access layer, Chapter 12, Section 12.2). **Persistence** itself means keeping data beyond the life of the process that created it, which for a backend almost always means a database.

A **database driver** (or client library) is the library inside the application process that implements one database's client protocol. It opens network connections to the database server, authenticates, sends SQL and parameters, receives results, and converts them into the language's values (integers, strings, dates). Above the driver there is usually a **standard interface** for the language: JDBC in Java, DB-API in Python, `database/sql` in Go. Code written against the interface can switch drivers without changing.

A **wire protocol** is the exact format of the messages a database client and server exchange over the connection: which bytes mean "here is a query", "here is a row", "an error happened" or "I am ready for the next command". PostgreSQL calls its protocol the **frontend/backend protocol**. MySQL has its own **client/server protocol**. They are different, so a PostgreSQL driver cannot talk to MySQL.

A **database connection** is one authenticated session between a client and the database server, carried over one TCP connection (or a Unix domain socket, Chapter 4, Section 4.8). On the server it has a dedicated worker: in PostgreSQL a whole OS process, in MySQL a thread (Chapter 3, Section 3.9). It also carries **session state**: the current transaction, settings changed with `SET`, temporary tables, prepared statements and session-level locks.

### Why it exists

A database is a separate server process, often on another machine. The application cannot call its functions directly; it can only send bytes over a socket (Chapter 4, Section 4.6). Something must agree on what those bytes mean, handle authentication and TLS (Chapter 8), split a stream of bytes into messages (TCP has no message boundaries, Chapter 6, Section 6.9), and turn the database's binary or text representation of a `numeric` or a `timestamptz` into a value the program can use. Writing that by hand in every application would be slow and error-prone, so each database ships or blesses drivers, and each language defines one interface for all of them.

### Mechanism: connecting to PostgreSQL

Every PostgreSQL message after the first has the same frame: a one-byte **type**, a four-byte **length** (which counts itself but not the type byte), and a body. This is the length-prefix framing of Chapter 6, Section 6.9. Here is what a driver and server exchange when an application connects with TLS and password authentication (SCRAM-SHA-256, the default password method in recent PostgreSQL versions):

```text
 application (driver)                                  PostgreSQL
 ─────────────────────────────────────────────────────────────────────────────
 TCP connect to db.internal:5432 ───── SYN / SYN-ACK / ACK ──────►  postmaster accepts
 SSLRequest (8 bytes: length 8, code 80877103) ─────────────────►
                                ◄──────────────────────── 'S'   (one byte: "yes, TLS")
 TLS 1.3 handshake (Chapter 8, Section 8.9) ◄──────────────────►
                                                         postmaster forks a backend
                                                         process for this connection
 StartupMessage (protocol 3.0, user=app, database=shop,
                 application_name=orders-api) ──────────────────►
                                ◄──── R  AuthenticationSASL (mechanisms: SCRAM-SHA-256)
 p  SASLInitialResponse (client nonce) ─────────────────────────►
                                ◄──── R  AuthenticationSASLContinue (salt, iterations)
 p  SASLResponse (proof computed from the password) ────────────►
                                ◄──── R  AuthenticationSASLFinal (server signature)
                                ◄──── R  AuthenticationOk
                                ◄──── S  ParameterStatus  server_version=17.2
                                ◄──── S  ParameterStatus  client_encoding=UTF8
                                ◄──── S  ParameterStatus  TimeZone=UTC   (… several more)
                                ◄──── K  BackendKeyData  (backend PID, secret key)
                                ◄──── Z  ReadyForQuery   status 'I' (idle)
```

Several details here matter later in the chapter:

- **The backend is a process.** PostgreSQL's postmaster forks a new backend process for each connection (Chapter 15, Section 15.1). That process lives exactly as long as the connection.
- **SCRAM is deliberately slow.** The client derives its proof from the password with an iterated hash (Chapter 8, Section 8.4 explains why password hashing is slow on purpose), so authentication costs real CPU on both sides, not just round trips.
- **`BackendKeyData`** gives the client the backend's PID and a secret. To cancel a running query, a driver opens a **second** connection and sends a `CancelRequest` with that pair; the postmaster signals the backend. This is how a driver's "cancel" or query timeout works in PostgreSQL.
- **`ReadyForQuery` carries the transaction status**: `I` (idle, no transaction), `T` (inside a transaction block) or `E` (inside a failed transaction, Chapter 17, Section 17.1). Drivers and poolers read this byte to know whether a connection is mid-transaction.
- **`application_name`** is a free-text label the client sends. It appears in `pg_stat_activity`, so setting it to your service's name is the cheapest diagnostic investment in this chapter.

### Mechanism: running a query

PostgreSQL has two ways to run SQL. The **simple query protocol** sends the whole SQL text in one `Query` message:

```text
 Q  Query "SELECT id, status FROM orders WHERE customer_id = 81"  ──────►
                         ◄──── T  RowDescription  (2 columns: id int8, status text)
                         ◄──── D  DataRow         ("9001", "PLACED")
                         ◄──── D  DataRow         ("9002", "SHIPPED")
                         ◄──── C  CommandComplete "SELECT 2"
                         ◄──── Z  ReadyForQuery   'I'
```

The **extended query protocol** splits the same work into steps, which is what makes prepared statements and parameter binding possible (Section 18.6):

```text
 P  Parse    name="" sql="SELECT id, status FROM orders WHERE customer_id = $1"
 B  Bind     portal="" statement="" parameters: [ "81" ]   ← the value travels separately
 D  Describe portal ""
 E  Execute  portal="" max_rows=0 (all rows)
 S  Sync                                                          ──────►
             ◄──── 1 ParseComplete   ◄──── 2 BindComplete
             ◄──── T RowDescription  ◄──── D DataRow ×2
             ◄──── C CommandComplete "SELECT 2"   ◄──── Z ReadyForQuery 'I'
```

A **portal** is PostgreSQL's name for a statement bound to specific parameter values and ready to run: in effect, an open cursor. `Execute` with `max_rows = 500` returns 500 rows and then `PortalSuspended`, and the client can ask for the next 500. That is how drivers **stream** large results (Section 18.14).

When something fails, the server sends an `ErrorResponse` (type `E`) whose fields include a severity, a five-character **SQLSTATE** code and a message, for example `C 23505` and `M duplicate key value violates unique constraint "orders_pkey"`. Drivers turn these into exceptions; application code should branch on the SQLSTATE, not the message text (Chapter 12, Section 12.8).

### Mechanism: the MySQL protocol, for contrast

MySQL frames packets with a three-byte length and a one-byte **sequence number**. The **server** speaks first:

```text
 TCP connect to db.internal:3306
                        ◄──── Initial Handshake (protocol 10, server version, connection id,
                              random salt, capability flags, auth plugin
                              "caching_sha2_password")
 HandshakeResponse (user, auth response, database, capabilities incl. TLS) ──►
   (with TLS: the client first sends a short SSL request, then the TLS handshake)
                        ◄──── OK packet
 COM_QUERY "SELECT id, status FROM orders WHERE customer_id = 81" ──►
                        ◄──── column count, column definitions, rows, end-of-result packet
```

On the server, the connection is served by a **thread**, often reused from a thread cache, not a forked process. That makes a new MySQL connection cheaper than a new PostgreSQL connection, though it still costs TCP, TLS and authentication round trips. Prepared statements use separate commands (`COM_STMT_PREPARE`, then `COM_STMT_EXECUTE` with binary-encoded parameters).

### Mechanism: what the driver buffers

When a query returns rows, where do they go? Many drivers read the **entire result** into application memory before returning the first row to your code. PostgreSQL's JDBC driver does so unless you set a fetch size inside a transaction; MySQL Connector/J does so unless you ask it to stream. `SELECT * FROM events` against a 40-million-row table then means 40 million rows in the heap, and an out-of-memory crash. Section 18.14 shows how to stream instead.

```text
 default (buffered)                          streaming (fetch size 500)
 DB ──all rows──► driver buffer ──► app      DB ──500──► driver ──► app processes 500
                  (memory = whole result)    DB ──500──► driver ──► app processes 500 …
                                             (memory = one batch; connection busy until done)
```

> **Java lens:** JDBC is Java's standard database interface: `DriverManager` or a `DataSource` gives a `Connection`; a `PreparedStatement` sends SQL with `?` placeholders; a `ResultSet` iterates rows. The PostgreSQL JDBC driver (pgJDBC) and MySQL Connector/J implement it. Application code should depend on `DataSource`, which is what connection pools implement (Section 18.3).

> **Other runtimes:** Python drivers follow DB-API 2.0 (`psycopg` for PostgreSQL, `mysqlclient` or `PyMySQL` for MySQL): `connect()`, `cursor()`, `execute(sql, params)`, `fetchmany()`. Go's `database/sql` defines the interface and drivers such as `pgx` implement it; notably, `sql.DB` is already a pool, not a single connection. Node.js has `pg` (node-postgres) and `mysql2`, both asynchronous.

### Guarantees and non-guarantees

A driver guarantees that messages are framed and decoded correctly, that errors surface with the server's SQLSTATE, and, with TLS and certificate verification on, that you are talking to the intended server (Chapter 8, Section 8.12). It does not guarantee that a lost connection's last command did or did not run: if the connection drops after `COMMIT` is sent, the outcome is **unknown** (Chapter 7, Section 7.15). It does not limit memory for large results unless you ask it to stream, and it does not make a cancelled query free: the server may already have done the work.

### Failure and diagnosis

- **Authentication and access errors.** `FATAL: password authentication failed for user "app"` (wrong secret, often after a rotation, Chapter 12, Section 12.9); `FATAL: no pg_hba.conf entry for host "10.0.3.17", user "app", database "shop", no encryption` (the server's access rules do not allow this client or require TLS); MySQL's `ERROR 1045 (28000): Access denied for user`. Evidence: the database server log shows the same line with the client address.
- **Connection limit reached.** PostgreSQL: `FATAL: sorry, too many clients already` (SQLSTATE `53300`), or a message that the remaining slots are reserved for superusers; MySQL: `ERROR 1040: Too many connections`. Evidence: `SELECT count(*) FROM pg_stat_activity` near `max_connections`. Cause is usually too many pools times too many instances (Section 18.4).
- **TLS problems.** Certificate verification errors or a server that refuses non-TLS clients. Evidence and tools as in Chapter 8, Section 8.13.
- **Out of memory on a big query.** Heap grows by the size of the result before the first row is processed. Evidence: a heap dump dominated by the driver's row buffers. Fix: streaming (Section 18.14) or pagination.

### Backend connection

Treat the connection as a remote, stateful, expensive object. Always set `application_name` (or MySQL's connection attributes) so the database can tell you which service and instance holds which connection. Always enable TLS with verification across untrusted networks. Know whether your driver buffers whole results. And branch on SQLSTATE codes, which are stable, rather than on error strings, which change between versions.

---

## 18.2 What a Connection Costs

### Definition

The **connection cost** is the time and resources needed to go from "no connection" to "ready to run a query": network round trips for TCP, TLS and authentication, CPU for cryptography and password hashing, and on the server the creation and initialisation of a worker (a process in PostgreSQL, a thread in MySQL) with its own memory. A **pooled borrow** is the cost of taking an already-open connection from a pool: usually a few microseconds of in-process work.

### Why it matters

A typical query on a warm, indexed table takes well under a millisecond on the database server. If every request opened its own connection, the setup would cost many times more than the query, and the database would spend a large share of its CPU on forking processes and hashing passwords instead of answering queries. That gap is the whole reason connection pools exist.

### Concrete: the cost breakdown

Assume the application and a PostgreSQL server are in the same data centre, with a **round-trip time (RTT) of 0.5 ms** (Chapter 6, Section 6.12), TLS 1.3, and SCRAM authentication. The numbers below are illustrative orders of magnitude, not measurements of a particular system; measure your own with the method at the end of this section.

```text
 step                                         round trips   time (RTT 0.5 ms)
 ───────────────────────────────────────────────────────────────────────────
 TCP three-way handshake (Chapter 6, 6.8)          1            0.5 ms
 SSLRequest → 'S' (PostgreSQL's TLS negotiation)   1            0.5 ms
 TLS 1.3 handshake (Chapter 8, 8.9)                1            0.5 ms
   + key exchange and certificate signature check  -           ~0.5–1 ms CPU
 StartupMessage → AuthenticationSASL               1            0.5 ms
 SCRAM exchange (two more messages each way)       2            1.0 ms
   + iterated password hashing, both sides         -           ~1–3 ms CPU
 postmaster fork + backend initialisation          -           ~1–2 ms server
 ParameterStatus … ReadyForQuery                   (same trip)  -
 ───────────────────────────────────────────────────────────────────────────
 total before the first query                      ~6         ≈ 5–8 ms
 first queries on the new backend                  -           slower: its catalog
                                                                caches start empty
 server memory for the backend                     -           a few MB, growing with
                                                                caches and work_mem use
 ───────────────────────────────────────────────────────────────────────────
 pooled borrow: take an idle connection from an
 in-process queue (maybe a liveness check)          0 (or 1)    ~microseconds (≈ 0.5 ms
                                                                with a validation query)
```

Now change one assumption. If the application runs in one region and the database in another, with an **RTT of 60 ms**, the six round trips alone cost **360 ms** before the first query. Even a perfectly tuned query cannot hide that.

Two more effects do not show up in the table:

- **Per-connection memory and scheduling.** Each PostgreSQL backend is a process. A thousand mostly idle connections are a thousand processes with their own memory and their own entries in the OS scheduler and in the database's shared structures (for example, the list of running transactions every snapshot must scan, Chapter 17, Section 17.6). Idle connections are not free.
- **Connection storms.** When the application restarts or the database fails over, every instance reconnects at once. Thousands of simultaneous forks and SCRAM computations can saturate the database's CPU for seconds, which is exactly when it is needed most.

MySQL's thread-per-connection model avoids the fork, and a thread cache can reuse an existing thread, so a new MySQL connection is cheaper on the server. The network and authentication round trips remain the same, so pooling is still standard practice there too.

### Concrete: measuring it yourself

```bash
# PostgreSQL ships pgbench; -C opens a NEW connection for every transaction.
pgbench -h db.internal -U app -S -T 30 shop        # reuse one connection per client
pgbench -h db.internal -U app -S -T 30 -C shop     # reconnect for every transaction
# Compare the reported transactions per second and average latency of the two runs.
```

The `-C` run typically shows several times lower throughput and much higher latency for the same read-only workload, and much higher CPU on the database host. The exact ratio depends on TLS, authentication method and network distance, which is why it is worth measuring on your own setup.

### Guarantees and non-guarantees

A freshly opened connection guarantees a clean session: no leftover transaction, settings or temporary tables. A pooled connection is cheaper but guarantees cleanliness only if the pool resets it on return (Section 18.3). Neither guarantees that the connection is still alive at the moment you use it: a pooled connection may have been cut by the network while idle.

### Failure and diagnosis

- **Per-request connections.** Symptom: every request is slow by a near-constant few milliseconds, and the database shows a high rate of new connections and short-lived backends. Evidence: the database log with `log_connections = on` shows a connection per request; the application creates connections with `DriverManager` (or the language equivalent) instead of a pool.
- **Reconnect storm after failover or deploy.** Symptom: database CPU at 100% with little query work right after a restart; many processes in authentication. Evidence: connection rate spike in the database's metrics. Fix: pools with a bounded size and gradual warm-up, jittered reconnects (Chapter 24), and a server-side pooler (Section 18.5).

### Backend connection

Never open a database connection per request in a server. Open a small, bounded number per instance and reuse them. Put the database close to the application (same region, ideally same zone), and remember that the gap between "connect" and "borrow" grows with network distance.

---

## 18.3 Connection Pooling: How a Pool Works

### Definitions

A **connection pool** is a component inside the application process that keeps a bounded set of open database connections and lends them to callers. A caller **borrows** (acquires, checks out) a connection, uses it for one unit of work, and **returns** (releases) it. If none is free, the caller waits in the pool's queue, up to a timeout. Chapter 11, Section 11.5 introduced pools as separate bottlenecks; this section opens one up.

The vocabulary used by most pools:

- **Maximum pool size**: the most connections the pool will ever hold open, borrowed or idle.
- **Minimum idle**: how many idle connections the pool tries to keep ready.
- **Acquisition timeout** (connection timeout): how long a caller waits for a connection before the pool gives up and throws an error.
- **Idle timeout**: how long an unused connection may sit idle before the pool closes it (only above the minimum).
- **Maximum lifetime**: the age after which the pool retires a connection, even if healthy, once it is returned.
- **Validation** (liveness check): confirming that a connection still works before handing it out, or periodically while idle.
- **Leak**: a borrowed connection that is never returned.
- **Leak detection threshold**: a time after which the pool logs a warning that a connection has been borrowed "too long", with the stack trace of the code that borrowed it.

### Why it exists

Section 18.2 showed that a connection costs milliseconds to create and megabytes on the server, while a query costs well under a millisecond. A pool pays the creation cost rarely and spreads one connection across thousands of requests. It also enforces a **ceiling**: whatever happens to traffic, this instance never opens more than its maximum, which protects the database from the "more concurrency, less throughput" collapse shown in Chapter 11, Section 11.5.

### Mechanism: the pool's state

```text
                         ┌──────────────── connection pool (max 10) ────────────────┐
  request threads        │                                                          │
                         │   idle connections        borrowed connections           │
  T1 ── borrow() ───────►│   ┌───┬───┬───┐           ┌───┬───┬───┬───┬───┬───┬───┐  │
  T2 ── borrow() ───────►│   │ c8│ c9│c10│           │ c1│ c2│ c3│ c4│ c5│ c6│ c7│  │
  T3 ── return(c4) ─────►│   └───┴───┴───┘           └───┴───┴───┴───┴───┴───┴───┘  │
                         │                                                          │
  T4 … T40 waiting ─────►│   wait queue: T4, T5, … (each with its own deadline)    │
                         │                                                          │
                         │   housekeeper (every few seconds): close idle above      │
                         │   minimum, retire old connections, top up to minimum,    │
                         │   check for leaks                                        │
                         └──────────────────────────────────────────────────────────┘
                                         │  real TCP connections  │
                                         ▼                        ▼
                                     database server: one backend per connection
```

The pool is built from the primitives of Chapter 5: a counting semaphore or lock plus condition variable to bound how many connections exist and to park waiters (Section 5.6), and a queue of idle connections. Simplified pseudocode:

```pseudo
class ConnectionPool:
    idle      = Deque()           // ready connections, most recently used first
    total     = 0                 // open connections (idle + borrowed)
    lock, notEmpty = Mutex(), Condition()

    function borrow(timeout):
        deadline = now() + timeout
        lock.acquire()
        try:
            loop:
                if idle.notEmpty():
                    c = idle.popFront()
                    if c.idleFor() > SHORT_WINDOW and not c.isAlive():   // validation
                        c.closeQuietly(); total -= 1; continue
                    c.borrowedAt = now(); c.borrowedBy = currentStackTrace()  // leak detection
                    return c
                if total < MAX_SIZE:
                    total += 1
                    lock.release()
                    try:   return openNewConnection()        // slow: done outside the lock
                    catch: lock.acquire(); total -= 1; notEmpty.signal(); throw
                remaining = deadline - now()
                if remaining <= 0:
                    throw PoolTimeout("no connection available within " + timeout)
                notEmpty.await(remaining)                     // park until a return
        finally:
            lock.releaseIfHeld()

    function giveBack(c):
        reset(c)                                              // see below
        lock.acquire()
        if c.age() > MAX_LIFETIME or c.isBroken():
            c.closeQuietly(); total -= 1                      // housekeeper will top up
        else:
            idle.pushFront(c)
        notEmpty.signal()
        lock.release()
```

Two details are worth noticing. The pool opens new connections **outside** its lock, because connecting takes milliseconds and must not block every borrower. And the most recently returned connection is handed out first (**LIFO**), which keeps a small set of "hot" connections busy and lets the rest age out when traffic drops.

### Mechanism: the life of one pooled connection

```text
          open (TCP + TLS + auth)
  ┌──────────┐  ok   ┌──────┐  borrow  ┌──────────┐   return   ┌───────────┐
  │ creating │──────►│ idle │─────────►│ borrowed │───────────►│ resetting │
  └──────────┘       └──────┘◄─────────┴──────────┘            └─────┬─────┘
       │ fails          ▲  │ idle too long,           │ held longer       │ clean
       ▼                │  │ too old, or failed        │ than leak          │
    (error to           │  │ validation                │ threshold:         │
     borrower)          │  ▼                           ▼ warning + stack    │
                        │ ┌────────┐                                       │
                        │ │ closed │◄──── broken during use / reset failed ─┤
                        │ └────────┘                                       │
                        └──────────────────────────────────────────────────┘
```

### Mechanism: reset on return

A connection carries session state (Section 18.1). If one request leaves it in a strange state, the next borrower inherits it. A careful pool **resets** each connection on return:

- **Roll back any open transaction.** A request that threw an exception after `BEGIN` but before `COMMIT` must not hand an open transaction (with its locks) to the next request. Pools usually roll back if the connection is not in autocommit mode or reports an open transaction.
- **Restore connection properties** the borrower changed: autocommit, isolation level, read-only flag, default schema, network timeout.
- **Optionally clear session state** such as `SET` parameters, temporary tables and session advisory locks. PostgreSQL's `DISCARD ALL` does this in one statement, but it also drops the session's prepared statements, so many client-side pools skip it and rely on the application not to change session state.

### Mechanism: validation, lifetimes and the network

A pooled connection can be dead without knowing it. The database may have restarted or failed over; a firewall or NAT device between the two may have silently dropped an idle connection after its idle timeout (Chapter 6, Section 6.5: the ghost connection). The next query on it then hangs until a TCP timeout or fails with a reset.

Pools defend in three ways:

1. **Validate before lending.** Check the connection with a driver-level liveness call or a trivial query (`SELECT 1`) if it has been idle for more than a short window.
2. **Keepalive while idle.** Periodically touch idle connections so middleboxes see traffic.
3. **Maximum lifetime.** Retire connections after a fixed age (often 30 minutes), set a little **shorter** than any idle or connection-age limit enforced by the network or the database, so the pool always closes connections before something else silently does. Lifetimes are usually randomised slightly so that connections do not all expire at once.

### Mechanism: leaks

A **connection leak** is a code path that borrows and never returns:

```pseudo
function exportReport(id):
    c = pool.borrow()
    rows = c.query("SELECT ... WHERE report_id = $1", id)
    if rows.isEmpty():
        return NOT_FOUND            // ✘ returns without giving c back
    write(rows)
    c.close()                       // "close" on a pooled connection means "return it"
```

Each call to the early-return path permanently removes one connection from the pool. With a pool of 10, the tenth "not found" turns the pool into a wall: every later request waits for the full acquisition timeout and fails. The fix is structural, not careful: borrow in a construct that always returns, such as try-with-resources in Java, `with` in Python, `defer rows.Close()` in Go, or, better, never borrow by hand and let the framework's transaction or repository layer do it.

### Concrete: what it looks like from outside

A pool exposes a few metrics, and the same small set tells almost every pool story:

```text
 metric (per instance)             healthy            exhausted           leaking
 ─────────────────────────────────────────────────────────────────────────────────
 active (borrowed) connections     2–4 of 10          10 of 10            climbs to 10
                                                                          and stays,
                                                                          even at night
 idle connections                  6–8                0                   0
 pending (waiting) borrowers       0                  dozens              dozens
 acquisition time p99              < 1 ms             = timeout (30 s)    = timeout
 usage time (held per borrow) p99  5 ms               2 s (slow queries   normal for
                                                       or slow work in     most; a few
                                                       a transaction)      never return
 timeouts per minute               0                  hundreds            hundreds
```

The difference between **exhausted** and **leaking** is visible in the database: when the pool is exhausted, `pg_stat_activity` shows the instance's 10 connections `active` (running queries) or `idle in transaction` (holding a transaction while the application does something else). When it is leaking, the connections are `idle` (no transaction, nothing running) but never returned: the database sees idle sessions, while the pool believes they are borrowed.

> **Java lens:** HikariCP is the most widely used JDBC pool and Spring Boot's default. Its main settings are `maximumPoolSize` (10 by default at the time of writing), `minimumIdle` (equal to the maximum unless set), `connectionTimeout` (30 s by default: the acquisition timeout), `idleTimeout`, `maxLifetime` (30 minutes by default), `keepaliveTime` and `leakDetectionThreshold` (off by default). An acquisition timeout throws `SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30000ms`; recent versions append the pool's total, active, idle and waiting counts to that message. With leak detection on, HikariCP logs a warning containing "Apparent connection leak detected" and the borrower's stack trace. Through Micrometer it publishes `hikaricp.connections.active`, `.idle`, `.pending`, `.acquire`, `.usage` and `.timeout`. A "connection is not available" error therefore means "no connection was free within the timeout", never "the database is down".

> **Other runtimes:** SQLAlchemy's default `QueuePool` holds `pool_size` connections plus `max_overflow` extra ones (5 and 10 by default), waits `pool_timeout` seconds (30), and raises `QueuePool limit of size 5 overflow 10 reached, connection timed out, timeout 30.00`; `pool_pre_ping` validates and `pool_recycle` sets a lifetime. Go's `sql.DB` is a pool whose maximum open connections is **unlimited** by default, so `SetMaxOpenConns`, `SetMaxIdleConns` and `SetConnMaxLifetime` must be set explicitly; forgetting to close `rows` is the classic Go leak. node-postgres's `Pool` defaults to 10 connections and, by default at the time of writing, waits for a connection **without a timeout** unless `connectionTimeoutMillis` is set. Prisma manages its own pool (`connection_limit`, `pool_timeout` in the connection string) and reports pool timeouts as error `P2024`.

### Guarantees and non-guarantees

A pool guarantees that the instance never holds more than its maximum number of connections, that a waiting caller gets a connection or an error within the acquisition timeout, and that connections are reused instead of recreated. It does not guarantee that a borrowed connection is alive (validation narrows the window but cannot close it), that a connection comes back clean unless reset is implemented, that waiting is fair, or that the database can serve the total number of connections all instances together are allowed to open.

### Failure and diagnosis

- **Pool exhaustion.** Symptom: "connection is not available" or "pool timeout" errors across all endpoints, latency equal to the acquisition timeout, low database CPU. Evidence: pool metrics (active = max, pending > 0, usage time high), thread dumps with many threads parked in the pool's borrow method (Chapter 11, Section 11.10), and in the database `pg_stat_activity` for that `application_name`. Diagnosis Lab 1 walks through it.
- **Leak.** Symptom: as above, but it builds up over hours and resets on restart. Evidence: the pool's leak-detection warning with the stack trace; database sessions `idle` while the pool counts them as borrowed.
- **Stale connections.** Symptom: the first request after a quiet period hangs and then fails with a connection reset or "I/O error"; it happens at night or after a failover. Evidence: failures cluster after idle periods; an idle timeout on a firewall or NAT gateway shorter than the pool's lifetime. Fix: validation, keepalive, and a maximum lifetime shorter than the network's idle timeout.
- **Dirty session state.** Symptom: one request occasionally sees another request's `search_path`, time zone or open transaction. Evidence: code that calls `SET` without `SET LOCAL`, or returns connections mid-transaction. Fix: reset on return; use `SET LOCAL` inside transactions.

### Backend connection

Use exactly one pool per database per instance, created at startup. Give it a **short** acquisition timeout (hundreds of milliseconds to a couple of seconds, below the request deadline) so that callers fail fast instead of piling up behind a long default; set a lifetime below the network's idle limits; turn on leak detection in non-production environments at least; and export the pool's metrics. Most of the incident timelines in this chapter can be read straight off those six numbers.

---

## 18.4 Sizing a Connection Pool

### Definition

**Pool sizing** is choosing each instance's maximum pool size so that, across all instances, the database receives about as much concurrency as it can process efficiently, and no more. It is driven by two numbers: the **demand** (how many connections are busy at once at peak, from Little's Law) and the **supply** (how many concurrent queries the database handles well, which depends mainly on its CPU cores and storage).

### Why it matters

Chapter 11, Section 11.5 showed the shape: database throughput rises with concurrency up to a point, then flattens and falls as processes, locks and caches contend. Too small a pool makes requests wait in the application while the database idles. Too large a pool moves the queue into the database, where it is more expensive: every waiting query is a running process competing for CPU, holding snapshots and sometimes locks. The usual instinct when a pool times out ("raise the pool to 100") is often exactly wrong.

### Mechanism: demand from Little's Law

Recall Little's Law from Chapter 11, Section 11.3: the average number of items in a system **L** equals the arrival rate **λ** times the average time each spends in it **W**. For a pool, the "items" are borrowed connections:

```text
 busy connections (L) = borrows per second (λ) × time each borrow holds a connection (W)
```

The **hold time** W is not the query time. It is everything between borrow and return: all the queries of the unit of work, the round trips between them, and any application work done while the connection is held. That is why the anti-pattern of Section 18.12 (remote calls inside a transaction) is so damaging: it inflates W without doing any database work.

### Mechanism: supply from the database

The database can only execute as many queries in parallel as it has cores, plus whatever overlap I/O waits allow. A commonly cited starting point for PostgreSQL, from the PostgreSQL wiki and HikariCP's documentation, and already met in Chapter 11, Section 11.5:

```text
 connections ≈ (database cores × 2) + effective number of independent disks
```

It is a starting point for a load test, not a law. A workload whose data fits in memory is CPU-bound and benefits little beyond the core count; a workload that waits on storage or on locks can use somewhat more. The important idea is that the answer is in the tens for a typical server, not the hundreds, and that it is a budget for **all** clients of the database together.

### Concrete: a worked sizing example

Assumptions:

- The database server has **16 cores** and SSD storage; data mostly fits in its buffer pool.
- At peak the service handles **2,000 requests per second**, all of which touch the database.
- Each request borrows a connection once and holds it for **6 ms** on average (three queries of about 1 ms each, plus round trips and a little application work between them).
- The service runs on **8 instances** behind a load balancer. Two other services (an admin backend and a reporting job) also use this database.

**Step 1: demand.** L = 2,000 × 0.006 = **12 connections busy on average**, across the whole fleet. Per instance: 250 requests per second × 0.006 s = **1.5** busy connections on average.

**Step 2: supply.** Starting point: 16 × 2 + 1 ≈ **33 concurrent queries** for the whole database. Reserve some for the other two services (say 4 each) and for administrators and migrations (say 3, matching superuser-reserved slots): about **22 connections** for this service.

**Step 3: divide.** 22 / 8 instances ≈ **3**. Round to **4** per instance, since averages hide bursts. Check headroom: a pool of 4 with a 6 ms hold time can serve 4 / 0.006 ≈ **660 borrows per second** per instance, against 250 at peak: 2.6× headroom for bursts and uneven load balancing.

**Step 4: check the edges.**

```text
 situation                                  connections opened by this service
 ───────────────────────────────────────────────────────────────────────────────
 steady state                               8 × 4 = 32 at most (12 busy on average)
 rolling deploy (old + new overlap)         16 × 4 = 64 for a few minutes
 autoscaler doubles instances to 16         16 × 4 = 64, and 128 during a deploy
 database max_connections                   e.g. 200 (each connection a process)
```

The rolling-deploy and autoscaling rows show why the budget must be set for the maximum instance count, not today's: the autoscaler should cap instances, or per-instance pools should shrink, or a server-side pooler (Section 18.5) should hold the line at the database.

**Step 5: what if hold time grows?** Suppose a new feature makes each request call a payment service while holding its connection, raising W from 6 ms to **60 ms**. Demand becomes L = 2,000 × 0.06 = **120 busy connections**, against a fleet-wide pool of 32. Requests queue for connections, acquisition times hit the timeout, and the service fails, although the database is nearly idle. Raising the pool to 15 per instance (120 total) would "fix" it only by pushing 120 mostly idle-in-transaction sessions onto a database designed for about 33 active ones. The real fix is to shrink W (Section 18.12).

### Mechanism: why the queue belongs in the application

With a pool of 4 and 200 worker threads, most threads wait in the pool's queue during a burst. That is correct. A thread waiting in the application costs a parked thread (or a parked virtual thread, which costs even less; Chapter 5, Section 5.13). A query waiting inside the database costs a running backend process, a snapshot that holds back VACUUM, and contention on shared structures. Queueing at the cheaper place, with a short acquisition timeout as a release valve, is the design. Virtual threads make it tempting to remove the limit, but they reduce the cost of waiting, not the database's capacity (Chapter 5, Section 5.13).

### Guarantees and non-guarantees

A correctly sized pool guarantees that the database never receives more concurrent work from this service than its share. It does not guarantee low latency if the hold time grows, if queries are slow, or if other clients ignore their budgets. Sizing rules give a starting point; only a load test against a production-like database shows the actual peak of the throughput curve.

### Failure and diagnosis

- **Pool too small for the demand.** Evidence: pending borrowers > 0 and acquisition time rising **while the database is under-used** (CPU low, few active sessions) and hold times are normal. Fix: a slightly larger pool, or more instances, within the database's budget.
- **Pool too large for the database.** Evidence: many `active` sessions in `pg_stat_activity`, high database CPU or lock waits, query latency rising with concurrency, and throughput no better than with a smaller pool. Fix: smaller pools; usually latency falls.
- **Hold time inflation.** Evidence: the pool's usage-time metric rises while query times in `pg_stat_statements` stay flat; sessions in `idle in transaction`. Fix: move non-database work out of the transaction.
- **Connection budget overrun.** Evidence: `too many clients` errors during deploys or scale-out. Fix: cap instances, shrink per-instance pools, or add a server-side pooler.

### Backend connection

Write the sizing down as arithmetic in the service's runbook: database cores, budget per service, instances (maximum, including deploys), pool per instance, measured hold time. Recompute it when any of those change. Alert on pool pending count and on hold time, not just on errors: they move minutes before the first timeout.

---

## 18.5 Server-Side Poolers: PgBouncer and Its Modes

### Definitions

A **server-side connection pooler** (or connection proxy) is a separate process that sits between applications and the database, speaks the database's wire protocol on both sides, accepts many client connections, and multiplexes them over a small number of real database connections. **PgBouncer** is the most widely used one for PostgreSQL. Others include Pgpool-II, PgCat and Odyssey for PostgreSQL, ProxySQL for MySQL, and managed proxies offered by cloud providers.

A pooler's **pooling mode** decides how long a client keeps a real server connection:

- **Session pooling**: a server connection is assigned when the client connects and is returned only when the client disconnects. Safe for everything, but saves little.
- **Transaction pooling**: a server connection is assigned when the client starts a transaction (or sends a single autocommit statement) and returned when the transaction ends. Many clients can share one server connection over time.
- **Statement pooling**: a server connection is returned after every statement; multi-statement transactions are not allowed.

### Why it exists

Client-side pools solve the cost of connecting, but each instance keeps its own pool. With 60 instances × 10 connections, the database holds 600 connections, most of them idle at any moment, each a PostgreSQL process. Short-lived processes (serverless functions, cron jobs, scripts) cannot keep a pool at all and would connect per invocation. A server-side pooler collapses all of this to a few dozen real connections, because at any instant only a small fraction of client connections is inside a transaction.

### Mechanism: transaction pooling

```text
  600 client connections                PgBouncer                 PostgreSQL
  (60 instances × pool of 10)     ┌───────────────────────┐
  ─────────────────────────────►  │ client side: accepts  │       30 server connections
  app-1  c1  idle                 │ 600 cheap connections │       (default_pool_size per
  app-1  c2  BEGIN … ───────────► │                       │ ───►   user+database pair)
  app-7  c5  BEGIN … ───────────► │ assign a free server  │ ───►  s1 s2 … s30
  app-9  c3  idle                 │ connection at BEGIN,  │
  app-12 c8  SELECT (autocommit)─►│ release it when       │ ───►
   …                              │ ReadyForQuery says    │
                                  │ status 'I' again      │
                                  │ queue clients while   │
                                  │ all 30 are busy       │
                                  └───────────────────────┘
```

The pooler watches the protocol. When a client sends the first message of a transaction, it takes a free server connection (or queues the client). It forwards messages both ways. When the server's `ReadyForQuery` reports status `I` (no transaction open, Section 18.1), the transaction is over and the server connection goes back to the pooler's free list, ready for any other client. A client may therefore run consecutive transactions on **different** server backends.

### Concrete: what each mode breaks

Because consecutive transactions can land on different backends, anything that lives in the **session** rather than in a transaction stops working reliably in transaction mode:

| Session feature | What goes wrong in transaction pooling | Workaround |
|---|---|---|
| `SET search_path` / `SET timezone` (not `LOCAL`) | The setting stays on the server connection and leaks to other clients; your next transaction may not see it | `SET LOCAL` inside the transaction, or set per role/database |
| Session advisory locks (`pg_advisory_lock`) | Lock taken on one backend; unlock attempted on another; or held forever by a backend another client now uses | Transaction-level `pg_advisory_xact_lock` (Chapter 17, Section 17.8) |
| `LISTEN` / `NOTIFY` listeners | The listening session is not stable | A direct (non-pooled) connection for listeners |
| Temporary tables, `WITH HOLD` cursors | Disappear or appear on another client's session | Use within one transaction, or session mode |
| Named prepared statements (protocol level) | Prepared on one backend, executed on another: `prepared statement "S_1" does not exist` | Recent PgBouncer versions (1.21 and later) can track protocol-level prepared statements when `max_prepared_statements` is set; otherwise disable server-side prepares in the driver |

In **session** mode none of these break, but the pooler only helps by absorbing connection churn, not by sharing backends. **Statement** mode breaks multi-statement transactions entirely, so it suits only autocommit workloads.

### Concrete: PgBouncer configuration and evidence

```ini
; pgbouncer.ini (illustrative)
[databases]
shop = host=10.0.2.10 port=5432 dbname=shop

[pgbouncer]
listen_port = 6432
pool_mode = transaction
default_pool_size = 30          ; server connections per (user, database) pair
max_client_conn = 2000          ; client connections PgBouncer will accept
reserve_pool_size = 5           ; extra server connections when clients wait too long
query_wait_timeout = 10         ; seconds a client may wait for a server connection
server_idle_timeout = 600
```

PgBouncer has an admin console (connect to the special `pgbouncer` database) whose `SHOW POOLS` command reports, per pool: `cl_active` (clients linked to a server connection), `cl_waiting` (clients waiting for one), `sv_active`, `sv_idle` and `maxwait` (how long the oldest waiting client has waited). Rising `cl_waiting` and `maxwait` is the pooler-level equivalent of the client pool's "pending" metric: the shared pool is too small for the transaction rate, or transactions are too long.

### Mechanism: two levels of pooling

With a pooler, applications usually keep their own client-side pool too: the client pool saves the TCP and TLS setup to the pooler, and the pooler limits the database's backends. The sizing logic of Section 18.4 now applies to the **pooler's** server pool (about the database's efficient concurrency), while client pools can be larger because client connections to PgBouncer are cheap. The pooler becomes one more hop (a fraction of a millisecond inside a data centre) and one more component that must be highly available: run more than one, behind a load balancer or on each application host.

### Guarantees and non-guarantees

A pooler in transaction mode guarantees that the number of database backends stays bounded no matter how many clients connect, and that each transaction runs entirely on one backend. It does not guarantee session continuity between transactions, does not make long transactions cheaper (a long transaction holds its server connection for its whole duration, exactly as before), and does not add database capacity.

### Failure and diagnosis

- **Session state surprises after introducing a pooler.** Symptoms: `prepared statement "S_3" does not exist` or `... already exists`; wrong time zones or schemas; advisory locks never released. Evidence: errors began when `pool_mode = transaction` was enabled. Fix: as in the table.
- **Clients waiting at the pooler.** Symptom: latency rises; the database looks fine; clients eventually get `query_wait_timeout` errors. Evidence: `SHOW POOLS` with high `cl_waiting` and `maxwait`, all `sv_active`. Fix: find long transactions holding server connections (`pg_stat_activity` on the database: they appear as normal sessions owned by PgBouncer's user, so `application_name` passed through matters); enlarge the pool only within the database's budget.
- **Pooler as a single point of failure.** Symptom: every service loses the database at once. Fix: redundant poolers and health checks.

### Backend connection

Introduce a pooler when the number of instances or short-lived clients makes the total connection count too high for the database, which happens early with PostgreSQL's process-per-connection design. Choose transaction mode for typical web workloads, audit the code for session-level features first, and keep transactions short: the pooler can only share a backend between transactions, never during one.

---

## 18.6 Prepared Statements and Parameter Binding

### Definitions

A **prepared statement** is a SQL statement sent to the database once as a **template** with placeholders for values, then executed one or more times with different values supplied separately. The placeholder syntax depends on the database and driver: `$1, $2` in PostgreSQL's protocol, `?` in MySQL and JDBC, `:name` in many libraries.

**Parameter binding** is the act of supplying values for the placeholders as **separate data**, outside the SQL text. The database never parses a bound value as SQL; it receives it already marked as "the value of parameter 1, of type bigint".

A **server-side prepared statement** is one the database itself parses (and possibly plans) once and keeps for the life of the session. A **client-side (emulated) prepared statement** is one the driver handles itself: it escapes each value and substitutes it into the text before sending one ordinary query. Some drivers emulate by default (MySQL Connector/J and PHP's PDO for MySQL, at the time of writing).

### Why it exists

Prepared statements solve two separate problems:

1. **Repeated work.** Chapter 16, Section 16.8 showed that the database parses, analyses, rewrites and plans every SQL text before executing it. For a query that runs thousands of times per second with different values, repeating the parse and plan each time wastes CPU.
2. **Code mixed with data.** If values are pasted into SQL text, the database cannot tell where the programmer's SQL ends and the user's data begins. Any value can change the meaning of the statement. That is SQL injection (Section 18.7).

### Mechanism: what happens on the wire

Recall the extended query protocol from Section 18.1. A **named** prepared statement separates preparation from execution:

```text
 once per connection:
   P  Parse  name="get_orders"  sql="SELECT id, status FROM orders WHERE customer_id = $1"
             parameter types: [int8]
   S  Sync                        ◄── 1 ParseComplete, Z ReadyForQuery

 each execution:
   B  Bind     statement="get_orders" portal="" format: binary
               param 1: length 8, bytes 00 00 00 00 00 00 00 51   (the integer 81)
   E  Execute  portal=""  max_rows=0
   S  Sync                        ◄── 2 BindComplete, D DataRow …, C "SELECT 2", Z 'I'
```

The SQL text crosses the network once. The value `81` is sent as a typed, length-prefixed field of the `Bind` message: eight bytes that are an integer, not characters that a SQL parser will read.

### Mechanism: what the database reuses

When a `Parse` message arrives, PostgreSQL parses and analyses the statement and stores the result in the backend's memory under the statement name. What happens on later executions is the plan-caching behaviour of Chapter 16, Section 16.8, recalled briefly:

```text
 execution 1–5    custom plan: planned with the actual parameter values each time
 execution 6+     if a generic plan (planned once, ignoring values) is estimated to cost
                  about the same, PostgreSQL switches to it and stops re-planning
```

The saving is the parse and analysis on every execution, plus planning once the generic plan is chosen. The risk is a **generic plan that is bad for some values**: for skewed data, a plan chosen for "a typical customer" can be very slow for the one customer with millions of rows (Chapter 16, Section 16.8 covers diagnosis and `plan_cache_mode`).

Prepared statements live in **one session**. A different connection does not know `get_orders`. That is why drivers keep a per-connection statement cache, why pools that drop session state on return (`DISCARD ALL`) lose them, and why transaction-mode poolers needed special support (Section 18.5).

### Mechanism: what can and cannot be a parameter

A parameter stands for a **value**: a literal that could appear where you would otherwise write `81` or `'PLACED'`. It cannot stand for any other part of the SQL:

```sql
SELECT * FROM orders WHERE status = $1;                 -- ✔ a value
SELECT * FROM orders WHERE created_at > $1 LIMIT $2;     -- ✔ values (LIMIT accepts a parameter in PostgreSQL)
SELECT * FROM $1 WHERE id = 7;                           -- ✘ a table name is not a value
SELECT * FROM orders ORDER BY $1;                        -- ✘ binds a constant: "ORDER BY 'created_at'"
                                                         --   sorts by the same constant for every row
SELECT * FROM orders ORDER BY created_at $1;             -- ✘ ASC/DESC are keywords, not values
SELECT * FROM orders WHERE id IN ($1);                   -- ✘ one parameter is one value, not a list
SELECT * FROM orders WHERE id = ANY($1);                 -- ✔ PostgreSQL: bind one array value
```

Identifiers (tables, columns), keywords (`ASC`, `DESC`), operators and lists of unknown length need a different technique: an **allowlist** that maps permitted inputs to fixed SQL fragments written by the programmer (Section 18.7). For lists, either generate one placeholder per element (`IN ($1, $2, $3)`), which creates a different statement text per list length, or bind an array.

### Concrete: preparing in plain SQL

PostgreSQL also exposes prepared statements as SQL commands, handy for experiments in a console:

```sql
PREPARE get_orders (bigint) AS
  SELECT id, status FROM orders WHERE customer_id = $1;

EXECUTE get_orders(81);
EXECUTE get_orders(82);

SELECT name, statement, generic_plans, custom_plans FROM pg_prepared_statements;
--     name    |                    statement                     | generic_plans | custom_plans
-- ------------+--------------------------------------------------+---------------+-------------
--  get_orders | PREPARE get_orders (bigint) AS SELECT id, ...    |             0 |            2

DEALLOCATE get_orders;
```

> **Java lens:** `connection.prepareStatement("SELECT … WHERE customer_id = ?")` followed by `ps.setLong(1, 81)` and `ps.executeQuery()`. pgJDBC uses the extended protocol for every `PreparedStatement`, but by default it switches to a named server-side statement only after the same statement has been executed a few times on a connection (`prepareThreshold`, 5 by default at the time of writing). MySQL Connector/J emulates prepares on the client unless `useServerPrepStmts=true`; its escaping is still safe when the connection's character set is configured correctly. A plain `Statement` with string concatenation offers neither benefit.

> **Other runtimes:** Python DB-API drivers take parameters as a separate argument: `cur.execute("SELECT … WHERE customer_id = %s", (81,))`. The `%s` is a placeholder for the driver, not Python string formatting; writing `"… = %s" % value` reintroduces injection. Go: `db.Query("SELECT … WHERE customer_id = $1", 81)`. Node: `client.query("SELECT … WHERE customer_id = $1", [81])`.

### Guarantees and non-guarantees

Parameter binding guarantees that a bound value is treated as a value of the declared type and never as SQL syntax. A server-side prepared statement guarantees that parsing is done once per session. Neither guarantees a good plan for every value, survival of the statement across connections or across `DISCARD ALL`, or any protection for parts of the SQL that were built from strings.

### Failure and diagnosis

- **Parameter-sensitive slowness.** The same statement is fast in a console (literal values, custom plan) and slow from the application after a few executions (generic plan). Evidence and fix: Chapter 16, Section 16.8.
- **`prepared statement "S_1" does not exist`** or **`already exists`**. The session changed under the driver: a transaction-mode pooler, or `DISCARD ALL` on return. Fix: pooler prepared-statement support, or disable server-side prepares (pgJDBC `prepareThreshold=0`).
- **`cached plan must not change result type`** (PostgreSQL). A table's columns changed (a migration) while a session held a prepared `SELECT *`. Fix: name columns explicitly; such errors after a deploy are a hint to reconnect pools or avoid `SELECT *`.
- **Statement cache memory.** Thousands of distinct statement texts (generated `IN` lists of every length) fill driver and server statement caches. Evidence: memory of long-lived backends grows; `pg_prepared_statements` is huge. Fix: bind arrays instead of variable-length `IN` lists.

### Backend connection

Use parameter binding for **every** value in **every** query, whether or not you care about plan reuse. Know whether your driver prepares on the server or emulates, because that decides how it interacts with poolers and migrations. And keep the number of distinct statement texts small and stable: it makes statement caches effective and statistics views such as `pg_stat_statements` readable.

---

## 18.7 SQL Injection

### Definitions

**SQL injection** is a vulnerability in which data supplied from outside the program becomes part of the **text** of a SQL statement in a way that changes the statement's structure, so an attacker can make the database run SQL the programmer did not write. It belongs to the wider class of **injection** flaws (OWASP Top 10 2021, category A03: Injection), which also includes command injection and NoSQL query injection.

Three variants matter for interviews and for real code:

- **Classic (first-order) injection**: the input arrives in the request and is concatenated into SQL in the same request.
- **Identifier and keyword injection**: input is used for a part of the SQL that **cannot** be a bound parameter, such as an `ORDER BY` column or a sort direction, so developers concatenate it even when they bind everything else.
- **Second-order (stored) injection**: the input is stored safely, then **later** read back from the database and concatenated into SQL by another piece of code that trusts it because "it came from our own database".

### Why it exists

A SQL statement is a program in a language with its own grammar. When code builds that program by gluing strings together, any string can contain grammar: a quote ends a literal, `--` starts a comment, `OR` adds a condition, `;` ends a statement. The database's parser sees only the final text and has no way to know which characters were meant as data. The vulnerability exists whenever the boundary between code and data is drawn only by quoting characters inside one string.

### Mechanism: the classic attack, step by step

A login check written with concatenation:

```pseudo
function login(email, password):
    sql = "SELECT id, password_hash FROM users WHERE email = '" + email + "'"
    row = db.queryOne(sql)
    ...
```

A normal user sends `email = alice@example.com`, and the database receives:

```sql
SELECT id, password_hash FROM users WHERE email = 'alice@example.com'
```

An attacker sends `email = ' OR '1'='1' --`. The database receives:

```sql
SELECT id, password_hash FROM users WHERE email = '' OR '1'='1' --'
```

What the parser sees, token by token:

```text
 SELECT id , password_hash FROM users WHERE
 email = ''          ← the attacker's first quote CLOSED the programmer's literal
 OR '1' = '1'        ← a condition the attacker wrote, true for every row
 --'                 ← a comment: swallows the programmer's closing quote
```

The query returns every user, and code that takes "the first row" may log the attacker in as the first account in the table, often an administrator. Variations do worse:

```sql
-- UNION-based: append a second query whose rows come back as if they were the first's
' UNION SELECT id, password_hash FROM users WHERE is_admin --

-- Boolean or time-based "blind" injection: no data is returned, but the attacker learns one
-- bit at a time from whether the page changes, or how long it takes
' OR (SELECT substr(password_hash, 1, 1) FROM users WHERE id = 1) = 'a' --
' OR (SELECT CASE WHEN substr(password_hash,1,1) = 'a' THEN pg_sleep(2) END) IS NULL --

-- Stacked queries: a second statement, where the protocol allows several in one message
'; DROP TABLE users; --
```

Stacked queries depend on the protocol path. PostgreSQL's simple query protocol accepts several statements in one message; its extended protocol accepts exactly one per `Parse`. MySQL drivers reject multiple statements unless an option enables them. That changes the damage, never the existence of the hole: a single injected `SELECT` can read any table the database user can read.

### Mechanism: why binding defeats it

With a parameter, the SQL text is fixed by the programmer and the value travels separately:

```text
 P  Parse  sql="SELECT id, password_hash FROM users WHERE email = $1"
           → parsed NOW, before any value exists; the tree has exactly one
             comparison whose right side is "parameter 1, type text"
 B  Bind   param 1: length 15, bytes "' OR '1'='1' --"
           → copied into the parameter slot; never tokenised by the SQL parser
 E  Execute
           → compares email with the 15-character string "' OR '1'='1' --"; no user has it
```

The statement's structure is decided before the attacker's bytes arrive, and nothing the attacker sends can reach the parser. That is a stronger property than "escaping": escaping tries to neutralise dangerous characters inside one string, and fails if the escaping function, the character set or the context is wrong. Binding removes the string altogether. (Drivers that emulate prepares do escape internally, which is safe when the driver knows the connection's character set; historically, mismatched multi-byte character sets were one way escaping went wrong.)

### Concrete: ORDER BY injection and the allowlist fix

An API lets clients sort a list: `GET /orders?sort=created_at&dir=desc`. Values in `WHERE` are bound, but a column name cannot be a parameter (Section 18.6), so the developer concatenates:

```pseudo
sql = "SELECT id, status, total_cents FROM orders WHERE customer_id = $1 ORDER BY "
      + request.sort + " " + request.dir + " LIMIT 50"
db.query(sql, [customerId])
```

The attacker sends `sort = (CASE WHEN (SELECT substr(password_hash,1,1) FROM users WHERE id=1)='a' THEN id ELSE total_cents END)`. The query is valid SQL. Its rows come back in one order or the other depending on a secret, so by comparing the two orders the attacker extracts the hash one character at a time: a blind injection through a sort parameter. A `dir` of `desc; DELETE FROM orders` is worse wherever stacked queries are allowed.

The fix is an **allowlist**: the client chooses among names the programmer defined, and the SQL fragment comes from the programmer's map, never from the request.

```pseudo
SORTABLE = { "created": "created_at", "total": "total_cents", "status": "status" }
DIRECTIONS = { "asc": "ASC", "desc": "DESC" }

column = SORTABLE.get(request.sort)     or reject 400 "unknown sort field"
dir    = DIRECTIONS.get(request.dir)    or reject 400
sql = "SELECT id, status, total_cents FROM orders WHERE customer_id = $1 " +
      "ORDER BY " + column + " " + dir + ", id " + dir + " LIMIT 50"   // id: tie-breaker (Ch 13, 13.6)
db.query(sql, [customerId])
```

Concatenation remains, but every concatenated fragment is a constant chosen from the program's own text. The same technique handles dynamic table names (map tenants or report types to fixed names), optional filters (append fixed fragments with bound values), and operators. When an identifier truly must come from data (an admin tool that names a table), quote it with the database's identifier-quoting function (PostgreSQL's `format('%I', name)` or `quote_ident`) and still check it against the catalog.

### Concrete: LIKE, where binding is necessary but not sufficient

A search box binds its value correctly:

```sql
SELECT id, name FROM products WHERE name LIKE '%' || $1 || '%';
```

No injection is possible: `$1` is a value. But `LIKE` has its own small language inside that value: `%` matches any sequence and `_` any single character. A user who searches for `%` or `_` gets every product; a search for `a%b%c%d%e%f%g%...` makes the matcher do a lot of work on every row; a leading `%` already forces a full scan (Chapter 16, Section 16.13). In an authorisation-sensitive query (`WHERE email LIKE $1` meant as "exact match"), a `%` turns "equals" into "matches anything".

The fix is to **escape the pattern characters** in the value and declare the escape character, or not to use `LIKE` for exact matches:

```pseudo
function likeEscape(s): return s.replace("\\", "\\\\").replace("%", "\\%").replace("_", "\\_")
db.query("SELECT id, name FROM products WHERE name LIKE $1 ESCAPE '\\'",
         ["%" + likeEscape(userInput) + "%"])
```

For real search, use a full-text or trigram index (Chapter 16, Section 16.6) with bound values, plus a minimum length and a rate limit.

### Concrete: second-order injection

A sign-up form inserts the username with a bound parameter. Safe so far:

```sql
INSERT INTO users (username, email) VALUES ($1, $2);     -- $1 = "admin'--"
```

Months later, a password-reset job written by someone else builds SQL from the stored row, reasoning that "data from our own table is trusted":

```pseudo
user = db.queryOne("SELECT username FROM users WHERE id = $1", [userId])  // "admin'--"
sql = "UPDATE users SET password_hash = '" + newHash +
      "' WHERE username = '" + user.username + "'"
db.execute(sql)
```

The statement the database receives:

```sql
UPDATE users SET password_hash = '$argon2id$...' WHERE username = 'admin'--'
```

The attacker's stored quote closes the literal; the comment removes the rest. The job resets the **administrator's** password to a value the attacker knows. The input was harmless when it entered and harmful when it was reused: input validation at the edge would not have helped unless it rejected every quote in every username forever.

The defence is the same rule applied without exceptions: **bind every value in every query, wherever the value came from**: the request, the database, a file, a message queue, another service. "Trusted source" is not a property of a string.

### Mechanism: defence in depth

Binding is the fix. Several layers limit the damage when a hole slips through:

- **Least privilege.** The application's database user owns no tables, cannot run DDL, and has only the `SELECT`/`INSERT`/`UPDATE`/`DELETE` rights it needs; migrations run as a different user (Section 18.15). Then `DROP TABLE` fails even if injected.
- **No raw error messages to clients.** Database errors in responses (`syntax error at or near "OR"`) help attackers map the query; return a generic problem response (Chapter 13, Section 13.8) and log the details.
- **ORMs and query builders**, which bind values by default. They do not protect raw SQL strings, string-built ORM queries (JPQL/HQL concatenation is injectable in the same way), or sort and filter fields passed straight through.
- **Detection.** Spikes of syntax errors in the database log, WAF alerts, and authorised testing with tools such as sqlmap. A WAF is a net, not a fix: it matches known patterns and misses encodings it does not expect (Chapter 10, Section 10.1).

The same idea applies outside SQL: a document database queried with a JSON object built from request fields can receive `{"password": {"$ne": null}}` instead of a string (NoSQL operator injection); the fix is again to separate structure from values and validate types.

### Guarantees and non-guarantees

Parameter binding guarantees that bound values cannot change a statement's structure. It does not protect identifiers, keywords or fragments built from strings; it does not stop `LIKE` wildcards or other mini-languages inside values; and it does nothing for code paths that build SQL from data read back from the database unless those paths bind too.

### Failure and diagnosis

- **Symptom of probing:** bursts of `ERROR: syntax error at or near` or `unterminated quoted string at or near` in the PostgreSQL log (MySQL: `You have an error in your SQL syntax`) from one endpoint, often with quotes and `UNION`/`SLEEP`/`pg_sleep` in parameters of the access log. Evidence: correlate the database log's statement with the request ID (Chapter 10, Section 10.11).
- **Symptom of a successful attack:** responses larger than usual from a list or search endpoint, unusual queries in `pg_stat_statements` (a `UNION` against `users` from the orders service), slow responses matching `pg_sleep` values.
- **Code review signals:** string concatenation or interpolation next to `query(`/`execute(`; `ORDER BY` built from request fields; "raw" or "native" query methods; queries built from database values in batch jobs.

### Backend connection

Make binding the only way the codebase can build a query: repository methods that take values, query builders, and a lint or review rule against string-built SQL. Treat every non-value part of SQL (sort fields, directions, table names, operators) as a closed allowlist defined in code. Run the application with a least-privilege database user. And remember that second-order injection is a code path problem, not an input problem: it is found by reading the code that reuses data, not the code that accepts it.

---

## 18.8 Object–Relational Mapping

### Definitions

An **object–relational mapper (ORM)** is a library that maps classes in the application to tables in a relational database, so that code can load, create, modify and delete objects while the library generates the SQL. The mapping is described by **metadata**: which class goes with which table, which field with which column, how the identity (primary key) is generated, and how references between objects correspond to foreign keys.

The **object–relational impedance mismatch** is the set of differences between the object model and the relational model that make the mapping imperfect:

- **Identity**: objects are identified by memory address; rows by primary key. Two loaded copies of the same row are two objects unless the ORM prevents it (Section 18.9).
- **Associations**: an object holds a reference or a collection (`order.lines`); a table holds a foreign key on the "many" side (`order_lines.order_id`). Navigation goes both ways in objects, one way in rows.
- **Navigation versus queries**: objects are traversed one reference at a time; relational data is retrieved by declarative set queries. Traversing references naively means one query per step (Section 18.11).
- **Inheritance**: classes inherit; tables do not, so a class hierarchy must be mapped to one table, a table per class, or a table per concrete class.
- **Granularity**: an `Address` value object may live inside an entity's table as columns, not as its own table.

Two architectural styles are common. In the **Active Record** pattern, each object knows how to save itself (`order.save()`), and a class corresponds closely to a table. In the **Data Mapper** pattern, objects know nothing about the database, and a separate mapper (the ORM's session or entity manager) moves data between them and the tables. Hibernate/JPA and SQLAlchemy's ORM follow Data Mapper; Ruby on Rails' ActiveRecord and Django's ORM follow Active Record.

### Why it exists

Chapter 12, Section 12.6 showed that every request maps data several times: request DTO to domain object, domain object to rows, rows back to objects. Writing that SQL and mapping code by hand for every entity is repetitive and easy to get subtly wrong (forgetting a column in one of five queries). An ORM writes the mechanical SQL, tracks which objects changed, and handles relationships. The price is that SQL becomes **implicit**: the library decides how many queries to run and when, and much of this chapter's remaining failure modes come from that.

### Mechanism: mapping metadata to SQL

The mapping for an order with lines, in language-neutral pseudocode:

```pseudo
entity Order -> table "orders":
    id          : Long     -> column "id", primary key, generated by sequence "orders_seq"
    customerId  : Long     -> column "customer_id"
    status      : Status   -> column "status" (stored as text)
    version     : Int      -> column "version", used for optimistic locking (Section 18.13)
    customer    : Customer -> many-to-one via column "customer_id"  (join to customers.id)
    lines       : List<OrderLine> -> one-to-many, mapped by OrderLine.order

entity OrderLine -> table "order_lines":
    id          : Long     -> column "id", primary key
    order       : Order    -> many-to-one via column "order_id"   (the owning side: holds the FK)
    productId   : Long     -> column "product_id"
    quantity    : Int      -> column "quantity"
```

From this metadata the ORM generates statements such as:

```sql
-- load by id
SELECT o.id, o.customer_id, o.status, o.version FROM orders o WHERE o.id = $1;
-- insert a new order (id taken from the sequence first, see below)
SELECT nextval('orders_seq');
INSERT INTO orders (customer_id, status, version, id) VALUES ($1, $2, $3, $4);
-- load the lines when first needed
SELECT l.id, l.order_id, l.product_id, l.quantity FROM order_lines l WHERE l.order_id = $1;
```

**Identifier generation** affects performance more than it seems. With a **sequence**, the ORM can fetch ids in advance (often in blocks, so one `nextval` covers 50 new objects) and send inserts later in batches. With an **identity column** (`AUTO_INCREMENT`, `GENERATED AS IDENTITY`, Chapter 14, Section 14.2), the id exists only after the `INSERT` runs, so the ORM must execute each insert immediately to learn it, which defeats insert batching (Section 18.14).

**Associations have an owning side.** In a bidirectional one-to-many, the foreign key lives in `order_lines`, so the `OrderLine.order` field is the **owning side**: setting it is what changes the column. Adding a line to `order.lines` without setting `line.order` changes nothing in the database in most ORMs, a frequent "my child row has a null foreign key" bug.

> **Framework lens:** JPA is Java's standard ORM interface and Hibernate its most common implementation. The mapping above is written with annotations: `@Entity`, `@Table(name = "orders")`, `@Id`, `@GeneratedValue(strategy = SEQUENCE)`, `@Version`, `@ManyToOne`, `@JoinColumn(name = "customer_id")` and `@OneToMany(mappedBy = "order")`, where `mappedBy` marks the non-owning side. Spring Data JPA adds repository interfaces whose method names (`findByCustomerIdAndStatus`) are turned into queries.

> **Other runtimes:** SQLAlchemy (Python) has a Core layer (a SQL expression builder) and an ORM layer in Data Mapper style, with mappings declared as classes. Prisma (TypeScript) works differently: a schema file generates a typed client, and queries are explicit calls (`prisma.order.findMany({ include: { lines: true } })`) that return plain objects. Prisma has no persistent identity map or dirty checking: nothing is saved unless you call an update. Django's and Rails' ORMs are Active Record style.

### Guarantees and non-guarantees

An ORM guarantees that the SQL it generates matches the mapping metadata, binds values as parameters, and keeps objects and rows consistent within its rules. It does not guarantee efficient SQL, a predictable number of queries, or that its in-memory objects reflect what other transactions have committed since they were loaded. It does not remove the need to understand transactions, isolation and indexes; it hides where they are used.

### Failure and diagnosis

- **Mapping mistakes.** Null foreign keys (only the inverse side set), wrong column types (enums stored by ordinal so that reordering an enum silently corrupts meaning), or missing columns. Evidence: the generated SQL in the ORM's SQL log; a schema validation step at startup.
- **Surprise SQL.** An innocent-looking getter triggers queries. Evidence: SQL logging with bound parameters, or a per-request query count (Section 18.11).
- **Schema drift.** The ORM's mapping and the real schema disagree after a manual change. Evidence: errors like `column "delivery_address" does not exist`. Fix: migrations as the single source of schema changes (Section 18.15), and mapping validation in tests.

### Backend connection

Use an ORM where it pays: loading and saving aggregates (Chapter 12, Section 12.3) with straightforward relationships. Drop to SQL for reports, bulk operations and complex reads. Either way, make the SQL visible: log it in development and tests, and count queries per request.

---

## 18.9 Inside the ORM: Identity Map, Unit of Work, Dirty Checking and Flush

### Definitions

- A **persistence context** (Hibernate/JPA term; SQLAlchemy calls it the **session**) is the ORM's in-memory workspace for one unit of work, usually one transaction or one request. It tracks every object it has loaded or been given.
- The **identity map** is the persistence context's table from (entity type, primary key) to the one in-memory object representing that row. Loading the same row twice returns the same object.
- A **managed** (persistent) entity is an object currently tracked by a persistence context. A **transient** (new) object is not yet known to any context; a **detached** object was managed by a context that has since closed; a **removed** object is scheduled for deletion.
- The **unit of work** is the pattern of recording all changes made during a business operation and writing them to the database together, in one go, at the end.
- **Dirty checking** is how the ORM discovers which managed objects changed: by comparing each object's current field values with a **snapshot** of the values it had when loaded.
- **Flushing** is the moment the ORM turns the recorded changes into SQL (`INSERT`, `UPDATE`, `DELETE`) and sends them to the database, inside the current transaction. Flushing is not committing.

### Why it exists

Without these mechanisms, code would have to remember to save every object it changed, in the right order, and could hold two different objects for the same row whose changes overwrite each other. The identity map gives one object per row per unit of work; the unit of work collects all changes so they can be sent together, ordered correctly and batched; dirty checking frees code from calling "save" after every modification.

### Mechanism: the identity map

```pseudo
tx.begin()
a = session.find(Order, 42)     // SELECT … FROM orders WHERE id = 42  → stored in identity map
b = session.find(Order, 42)     // no SQL: identity map hit
assert a is b                   // the same object, not just equal
a.status = SHIPPED
print(b.status)                 // SHIPPED: there is only one object
```

The identity map is a per-context cache, often called the **first-level cache**. It is not shared between requests, and it does not see changes other transactions commit after the object was loaded: within the context, `find(Order, 42)` keeps returning the object loaded first. Queries (as opposed to `find` by id) always go to the database, but rows they return for objects already in the map are mapped to the **existing** objects, and the existing objects' in-memory state wins.

### Mechanism: the life of an entity

```text
                 new Order()
                     │
                     ▼
               ┌───────────┐  persist / save   ┌───────────┐   remove     ┌─────────┐
               │ transient │──────────────────►│  managed  │─────────────►│ removed │
               └───────────┘                   └───────────┘              └────┬────┘
                                                 ▲     │ context closes,       │ flush:
                       find / query ─────────────┘     │ clear, or evict       │ DELETE
                                                       ▼                       ▼
                                                 ┌───────────┐            (row gone
                                                 │ detached  │             at commit)
                                                 └─────┬─────┘
                                                       │ merge: copy state onto a
                                                       └──► managed instance (SELECT first)
```

Changes to a **managed** object are saved automatically at the next flush. Changes to a **detached** object are not saved at all unless it is merged back. Many "my update was not saved" and "my update was saved although I never called save" bugs are simply about which state the object was in.

### Mechanism: dirty checking and the flush, as a flow

```text
 load:   SELECT … WHERE id = 42
         row ──► build object O42 ──► identity map[(Order,42)] = O42
                                 └──► snapshot[(Order,42)]   = copy of loaded values
                                      {status: PLACED, customer_id: 81, version: 3}

 work:   O42.status = SHIPPED          (plain field assignment: no SQL, nothing recorded)
         session.persist(new OrderLine(...))   → action queue: INSERT pending

 flush (before commit, before some queries, or on explicit flush()):
   1. for every managed object, compare current values with its snapshot
         O42: status PLACED → SHIPPED  ⇒ dirty
   2. build the SQL for all pending work, in a fixed order:
         inserts  →  updates  →  collection changes  →  deletes   (Hibernate's order)
   3. send it (batched where possible) on the transaction's connection:
         INSERT INTO order_lines (…) VALUES (…)
         UPDATE orders SET customer_id=$1, status=$2, version=$3 WHERE id=$4 AND version=$5
   4. refresh the snapshots to the flushed values

 commit: COMMIT   (the transaction ends; the context usually closes with it)
```

Some consequences of this design are not obvious until they bite:

- **Changes are saved without a save call.** Any modification of a managed object inside a transaction is written at flush. Code that "only wanted to format the name for display" by modifying the entity will update the row.
- **SQL runs later than the code that caused it.** A unique-constraint violation or a deadlock caused by line 12 surfaces at the flush, maybe at commit, in a different stack frame. Error handling around line 12 never sees it.
- **The fixed flush order can violate constraints.** Deleting a row and inserting a new one with the same unique key in one unit of work fails, because inserts are flushed **before** deletes. The fix is an explicit flush after the delete.
- **The default UPDATE writes all mapped columns**, not only the changed one (Hibernate's default; it lets one prepared statement serve every update of that entity). Two concurrent requests that change different fields of the same row then overwrite each other's field (Section 18.13). Hibernate's `@DynamicUpdate` generates per-change column lists.
- **Dirty checking costs CPU and memory** proportional to the number of managed objects: every flush compares every field of every managed object. A batch job that loads 200,000 entities into one context holds 200,000 objects and snapshots and spends seconds per flush (Section 18.14).

### Mechanism: when the ORM flushes

Most ORMs default to an **automatic** flush mode: flush at commit, and also **before running a query whose results could be affected by pending changes**, so that the query sees them.

```pseudo
tx.begin()
o = session.find(Order, 42)
o.status = CANCELLED                       // pending, not yet in the database
n = session.query("SELECT count(*) FROM Order o WHERE o.status = 'CANCELLED'")
// the ORM flushes the UPDATE first, so the count includes order 42
tx.commit()
```

A **manual** flush mode (no flushing until explicitly asked) is used for read-only work, where it also lets the ORM skip dirty checking. A native SQL query may or may not trigger an automatic flush depending on the ORM and how much it can tell about the tables involved, which is a classic source of "my query did not see my change".

### Concrete: watching it happen

Turning on SQL logging makes the deferred behaviour visible:

```text
12:00:01.101  orders-api  POST /orders/42/ship  request start
12:00:01.103  SQL  select o.id,o.customer_id,o.status,o.version from orders o where o.id=?   [42]
12:00:01.104  app  order 42 loaded, status=PLACED
12:00:01.104  app  status set to SHIPPED; shipment row created
12:00:01.105  app  calling notifyWarehouse()                      ← no SQL has run for the change
12:00:01.180  SQL  insert into shipments (order_id,carrier,id) values (?,?,?)   [42,'DHL',5512]
12:00:01.181  SQL  update orders set customer_id=?,status=?,version=? where id=? and version=?
                   [81,'SHIPPED',4,42,3]
12:00:01.183  SQL  commit
```

The `UPDATE` appears 77 ms after the code that caused it, at commit, after the warehouse call; and the update includes `customer_id`, which nobody changed.

> **Framework lens:** In JPA, the persistence context belongs to an `EntityManager`; in Spring, one is bound to each transaction (`@Transactional`) and closed at its end. `EntityManager.persist`, `find`, `merge`, `remove`, `flush`, `clear` and `detach` correspond directly to the transitions above; the flush mode is `AUTO` by default and Hibernate also offers `MANUAL`. Spring Boot enables **open session in view** by default at the time of writing (`spring.jpa.open-in-view=true`, with a startup warning): the persistence context then stays open for the whole web request, including view rendering and serialisation, so lazy loading works anywhere, at the cost of database access from the presentation layer and a connection potentially held for longer. Many teams disable it.

> **Other runtimes:** SQLAlchemy's `Session` is a unit of work with an identity map; `session.add()` makes an object pending, `autoflush` (on by default) flushes before queries, and `expire_on_commit` (on by default) marks all objects stale after commit so the next attribute access reloads them. Prisma has no unit of work: each call such as `prisma.order.update(...)` is its own statement, and `prisma.$transaction([...])` or an interactive transaction groups them.

### Guarantees and non-guarantees

The persistence context guarantees one object per row within a context, that all changes to managed objects are written at flush in a consistent order, and that the context's own queries see its own pending changes under automatic flush. It does not guarantee that objects reflect other transactions' later commits, that SQL errors surface where the change was made, that only changed columns are written, or that a large context remains fast.

### Failure and diagnosis

- **Unexpected updates.** Rows change although no code "saved" them. Evidence: `UPDATE` statements in the SQL log at commit; an entity modified for display or conversion. Fix: map to DTOs for presentation work; use read-only transactions (Section 18.12).
- **Errors at commit instead of at the operation.** A constraint violation stack trace that points at the transaction interceptor, not the business code. Fix: explicit `flush()` at points where you need errors to surface, inside the `try` that handles them.
- **Stale reads within a long context.** A request reads an entity, another transaction updates it, the first re-reads and still sees old values. Evidence: identity-map hits in the SQL log (no `SELECT` for the second read). Fix: shorter contexts, `refresh`, or optimistic locking to detect the conflict at write time.
- **Slow flushes in batches.** Evidence: time spent in the ORM's flush or dirty-check code in a profiler; memory proportional to rows processed. Fix: flush and clear every N objects (Section 18.14).

### Backend connection

Keep the persistence context's lifetime equal to one short transaction. Treat loaded entities as "live" objects whose modifications will be written, and do presentation work on DTOs. Expect SQL at flush time, not at the line of code, and design error handling accordingly.

---

## 18.10 Lazy Loading, Eager Loading and Proxies

### Definitions

- **Lazy loading** is deferring the loading of an association (a referenced object or a collection) until the code first accesses it.
- **Eager loading** is loading an association at the same time as its owner, either with a join in the same query or with an immediate second query.
- A **lazy proxy** is a placeholder object that stands in for an unloaded entity. It looks like the real class (it is a run-time generated subclass, Chapter 12, Section 12.5) and knows only the entity's id; calling any other method on it triggers a `SELECT`. A **lazy collection wrapper** plays the same role for collections.
- The **fetch plan** (fetch strategy) of a query is the decision, per association, of what to load now and what to defer.

### Why it exists

Object graphs are connected: an order references a customer, which has addresses and other orders, each with lines referencing products. Loading everything reachable from one object would load most of the database. Lazy loading lets a request load only what it actually touches. The cost is that the number of queries now depends on which code runs, and that touching an association after the persistence context has closed is impossible.

### Mechanism: how a lazy proxy works

```text
 session.find(Order, 42)
   SELECT o.id, o.customer_id, o.status FROM orders o WHERE o.id = 42
   → Order#42 { id: 42, status: PLACED,
                customer: Customer$Proxy{ id: 81, target: null, session: S } ,  ← no query yet
                lines: LazyList{ owner: 42, loaded: false, session: S } }        ← no query yet

 order.customer.getId()     → 81 (the proxy already knows the id; no SQL in most ORMs)
 order.customer.getName()   → proxy intercepts the call:
                                if target == null:
                                    if session S is closed: throw "could not initialize proxy"
                                    target = S.load("SELECT … FROM customers WHERE id = 81")
                                return target.getName()
 for line in order.lines    → LazyList loads: SELECT … FROM order_lines WHERE order_id = 42
```

Because the proxy is a subclass generated at run time, it shares the proxy pitfalls of Chapter 12, Section 12.5: final classes or final methods cannot be intercepted, `getClass()` returns the proxy class, `equals` implementations that compare classes or read fields directly (instead of through getters) see the uninitialised proxy, and `instanceof` checks against subclasses in an inheritance hierarchy can fail because the proxy extends the base class.

### Mechanism: eager loading and its two forms

Eager loading by **join** fetches owner and association in one query:

```sql
SELECT o.id, o.status, c.id, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.id = $1;
```

Eager loading by **separate select** runs the owner query and then an immediate query per association, which, applied to a list of owners, becomes the N+1 problem of Section 18.11. An association marked eager in the **mapping** is loaded on every load of the owner, whether or not this particular use case needs it, and the mapping cannot be overridden to "lazy" for one query. That is why the usual advice is to map associations lazy and choose eager loading **per query**.

### Concrete: the closed-context failure

```pseudo
@Transactional
function getOrder(id): return orderRepository.findById(id)     // context closes on return

// controller, outside the transaction:
order = service.getOrder(42)
return toJson(order)          // serialiser walks order.lines → lazy load → context closed
```

```text
org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role:
com.shop.order.Order.lines: could not initialize proxy - no Session
```

There are three possible "fixes", and only one is good:

1. **Keep the context open for the whole request** (open session in view, Section 18.9). The error disappears, the serialiser triggers one query per association it walks, and the connection is held during serialisation. This hides N+1 problems instead of fixing them.
2. **Mark the association eager in the mapping.** Every load of every order now loads its lines, even where they are not needed.
3. **Decide the fetch plan in the use case.** The service method that returns an order for the API loads exactly what the response needs (a fetch join or batch fetch, Section 18.11) and maps it to a DTO **inside** the transaction. The controller receives a plain object with no proxies.

The third option is a consequence of Chapter 12's layering: the service defines the use case, so it also defines what data the use case needs.

> **Framework lens:** In JPA, `@ManyToOne` and `@OneToOne` are **eager** by default and `@OneToMany` and `@ManyToMany` lazy, a historical default that many teams override with `fetch = FetchType.LAZY` on every to-one association. Hibernate generates proxies with a bytecode library at run time; `Hibernate.initialize(x)` forces loading, and `Hibernate.isInitialized(x)` tests it.

> **Other runtimes:** SQLAlchemy's relationship loading is configured per relationship (`lazy="select"` by default) and overridden per query with options such as `joinedload()`, `selectinload()` and `raiseload()`; `raiseload` makes any unplanned lazy load raise an error, which turns hidden N+1 problems into test failures. Prisma loads relations only when requested with `include` or `select`, so there is no implicit lazy loading.

### Guarantees and non-guarantees

Lazy loading guarantees that associations are loaded at most once per context and only if touched. It does not guarantee a bounded number of queries, that access outside the context works, or that a proxy behaves exactly like the real class with respect to type checks and equality. Eager mapping guarantees the association is present, but not that it is cheap or needed.

### Failure and diagnosis

- **`LazyInitializationException`** (or the equivalent "detached instance" error in other ORMs). Evidence: the stack trace shows serialisation or view code outside the transaction. Fix: fetch in the use case and return DTOs.
- **Eager graphs that load half the database.** Evidence: a simple `findById` producing a multi-way join or many queries; large result sizes. Fix: lazy mappings, explicit fetch plans.
- **Proxy-related bugs.** `equals` returning false for the same entity; `ClassCastException` with inheritance. Fix: id-based `equals`/`hashCode` through getters, no final entity classes, avoid type checks on lazily loaded references.

### Backend connection

Map associations lazily and decide fetching per use case. Return DTOs from transactional service methods so that nothing outside the transaction can trigger SQL. Lazy loading is a convenience for code that touches a handful of objects; for lists, it is the mechanism behind the next section's most common performance bug.

---

## 18.11 The N+1 Query Problem

### Definition

The **N+1 query problem** is a pattern in which code runs **one** query to fetch a list of N items and then **one more query per item** to fetch related data, for N+1 queries in total where one or two would do. It is the most common performance bug in ORM-based code, but it is not specific to ORMs: any loop that calls the database once per element produces it.

### Why it happens

Lazy loading (Section 18.10) makes every association access look like a field read. A loop over 100 orders that prints each customer's name looks like 100 field reads; it is 100 queries. With eager to-one mappings loaded by separate selects, the same happens even without touching the association. The code is correct and fast in development, where lists have five items and the database is on the same laptop, and slow in production, where lists have hundreds of items and every query costs a network round trip.

### Mechanism: where the time goes

```pseudo
@Transactional(readOnly = true)
function recentOrders():
    orders = orderRepository.findTop100ByOrderByCreatedAtDesc()   // 1 query
    return orders.map(o -> OrderSummary(
        o.id,
        o.customer.name,               // lazy proxy: 1 query per distinct customer
        o.lines.size()))               // lazy collection: 1 query per order
```

For 100 orders with 100 different customers, that is 1 + 100 + 100 = **201 queries**. Assume each costs **0.5 ms** of network round trip plus **0.2 ms** of execution on the database:

```text
 N+1 version:     201 × (0.5 + 0.2) ms  ≈ 141 ms, and 201 round trips on one connection
 two-query fix:     2 × (0.5 + ~1.5) ms ≈   4 ms   (each query does more work, once)
```

The database itself is barely busy: each query is a primary-key lookup. The time is spent in **round trips**, in the driver and ORM handling 201 result sets, and in the connection being held for the whole loop. By Little's Law (Section 18.4), a 141 ms hold time instead of 4 ms needs 35 times as many connections for the same request rate.

### Concrete: what it looks like in a SQL log

```text
14:02:11.210 SQL select o.id,o.created_at,o.customer_id,o.status from orders o order by o.created_at desc limit ?   [100]
14:02:11.213 SQL select c.id,c.name,c.email from customers c where c.id=?          [5513]
14:02:11.214 SQL select l.id,l.order_id,l.product_id,l.quantity from order_lines l where l.order_id=?   [90211]
14:02:11.215 SQL select c.id,c.name,c.email from customers c where c.id=?          [882]
14:02:11.216 SQL select l.id,l.order_id,l.product_id,l.quantity from order_lines l where l.order_id=?   [90210]
14:02:11.217 SQL select c.id,c.name,c.email from customers c where c.id=?          [41007]
   … 195 more lines of the same two statements with different parameters …
14:02:11.352 SQL select l.id,l.order_id,l.product_id,l.quantity from order_lines l where l.order_id=?   [90112]
```

The signature: **the same statement text repeated many times in one request, with only the parameter changing**, right after one list query.

### Mechanism: detecting it

N+1 problems hide in plain sight because each query is fast. Evidence comes from counting:

- **Per-request query count.** Log or measure how many statements each request runs. Most ORMs can report it (Hibernate's statistics, SQLAlchemy event hooks, a JDBC proxy driver that counts statements). A list endpoint running hundreds of queries is a finding.
- **Tracing.** Distributed traces (Chapter 26) record one request's path through every service, and each timed step in a trace is a **span**. They show a request span with a long "comb" of identical, tiny database spans.
- **Database statistics.** In `pg_stat_statements`, a primary-key lookup with an enormous `calls` count and a tiny `mean_exec_time`, whose call rate is a multiple of the request rate of a list endpoint:

```sql
SELECT calls, round(mean_exec_time::numeric, 3) AS mean_ms, left(query, 70) AS query
FROM pg_stat_statements ORDER BY calls DESC LIMIT 3;
--    calls    | mean_ms |                                 query
-- ------------+---------+------------------------------------------------------------------------
--  418 220 571|   0.021 | select c.id,c.name,c.email from customers c where c.id=$1
--  401 932 006|   0.034 | select l.id,l.order_id,l.product_id,l.quantity from order_lines l where
--    2 011 405|   1.102 | select o.id,o.created_at,o.customer_id,o.status from orders o order by
```

A ratio of about 200 calls of the child queries for every call of the list query is the N+1 fingerprint.

- **Tests that assert query counts.** A test that loads the endpoint with 50 items and asserts "at most 3 statements" catches regressions before production. SQLAlchemy's `raiseload` (Section 18.10) is a stricter variant: any unplanned lazy load fails the test.

### Concrete: fix 1, a join (fetch join)

Load the association in the same query:

```sql
SELECT o.id, o.created_at, o.status, c.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
ORDER BY o.created_at DESC
LIMIT 100;
```

For **to-one** associations (each order has one customer), this is ideal: one query, no extra rows. For **to-many** associations (each order has many lines), a join multiplies rows (the fan-out of Chapter 14, Section 14.7): 100 orders with 5 lines each return 500 rows, with each order's columns repeated five times. Two consequences:

- **Pagination breaks.** `LIMIT 100` now limits joined rows, not orders. ORMs that detect this either refuse or fetch **everything** and paginate in memory. Hibernate warns with a message such as `firstResult/maxResults specified with collection fetch; applying in memory`, a warning that should be treated as an error.
- **Two collections multiply each other.** Joining lines (5 per order) and payments (3 per order) in one query returns 5 × 3 = 15 rows per order: a Cartesian product. Hibernate refuses some of these combinations with `MultipleBagFetchException: cannot simultaneously fetch multiple bags`.

### Concrete: fix 2, batch fetching

Load the association for **all** owners with one query using their ids:

```sql
-- query 1: the page of orders
SELECT o.id, o.created_at, o.customer_id, o.status FROM orders o ORDER BY o.created_at DESC LIMIT 100;
-- query 2: all their lines at once
SELECT l.id, l.order_id, l.product_id, l.quantity
FROM order_lines l WHERE l.order_id = ANY($1);          -- $1 = array of the 100 order ids
-- query 3: all their customers at once
SELECT c.id, c.name FROM customers c WHERE c.id = ANY($1);   -- $1 = distinct customer ids
```

Three queries regardless of N; pagination stays correct because query 1 has no join; no row multiplication. The ORM groups the children by `order_id` in memory. ORMs offer this as a setting (Hibernate's batch size for associations, which loads lazy proxies in groups of N ids at a time; SQLAlchemy's `selectinload`; Prisma's `include`, which issues a query per relation with an `IN` list). It is usually the best default for collections. This is also the idea behind GraphQL's DataLoader (Chapter 13, Section 13.12): collect keys during one tick, then load them in one query.

### Concrete: fix 3, a projection

Often the endpoint does not need entities at all. It needs a few columns and an aggregate. Ask the database for exactly that:

```sql
SELECT o.id, o.created_at, c.name AS customer_name, count(l.id) AS line_count
FROM orders o
JOIN customers c ON c.id = o.customer_id
LEFT JOIN order_lines l ON l.order_id = o.id
GROUP BY o.id, o.created_at, c.name
ORDER BY o.created_at DESC
LIMIT 100;
```

One query, a page of 100 rows, no entity objects, no snapshots, no dirty checking, no lazy proxies, and the counting done where the data is. Map the rows straight to a DTO (`OrderSummary`). ORMs support this as **DTO projections** (a query that selects into a non-entity class or an interface). For read-only list endpoints this is usually the fastest and simplest option.

### Concrete: choosing among the fixes

| Situation | Best fix | Why |
|---|---|---|
| To-one association needed for each row | Join (fetch join) | One query, no fan-out |
| To-many association, paged list | Batch fetch (`= ANY($1)` / `IN`) | Pagination stays correct; no Cartesian product |
| Several to-many associations | Batch fetch, one query per association | Avoids multiplying rows |
| Read-only list or report | DTO projection with joins and aggregates | No entity overhead; the database aggregates |
| Need to modify the loaded entities | Fetch join or batch fetch of entities | Projections are not managed, so changes are not saved |

### Mechanism: N+1 beyond ORMs

The same shape appears wherever a loop calls something remote:

```pseudo
for id in cart.productIds:  price = pricingService.get(id)     // N HTTP calls: N+1 over HTTP
for user in users:          cache.get("prefs:" + user.id)       // N cache round trips
```

The fixes are the same: a batch interface (`POST /prices:batchGet` with a list of ids, a multi-get in the cache), or a join at the source.

### Guarantees and non-guarantees

Each fix guarantees a bounded number of queries independent of N. None guarantees that the remaining queries are fast: a batch fetch with 10,000 ids or a projection without a suitable index can be slow in its own right (Chapter 16), and a fetch join on a large collection can return enormous result sets.

### Failure and diagnosis

- **Symptom:** a list endpoint's latency grows linearly with page size or with the number of items a user has; database CPU low; connection usage time per request high. Diagnosis Lab 2 walks through it.
- **Evidence:** per-request query count; repeated statements in the SQL log or trace; `pg_stat_statements` call ratios.
- **Regression after a "harmless" change:** a new field in a response DTO (`customer.loyaltyTier`) triggers a lazy load per row. Evidence: query count per request jumped at the deploy. Prevention: query-count assertions in tests.

### Backend connection

Make query count per request a first-class metric for list endpoints. Decide fetch plans in the use case, defaulting to batch fetching for collections and projections for read-only views. And recall Little's Law: an N+1 problem is not only a latency bug but a **capacity** bug, because it multiplies how long each request holds a connection.

---

## 18.12 Transaction Boundaries in Application Code

### Definitions

A **transaction boundary** is the point in the code where a transaction begins and the point where it ends (commit or rollback). Chapter 17 defined transactions inside the database; this section is about **who** in the application decides to start and end them, and **how**.

- **Programmatic transaction management**: the code calls begin, commit and rollback explicitly (directly or through a helper that takes a function).
- **Declarative transaction management**: the code marks a method as transactional (an annotation or decorator), and a proxy or wrapper begins a transaction before the method and commits or rolls back after it (Chapter 12, Section 12.5).
- **Transaction propagation**: the rule that decides what happens when a transactional method is called while a transaction is **already** active: join it, start a new one, run without one, or fail.
- A **read-only transaction** is one declared in advance never to write, which lets the database and the ORM skip work and lets routing send it to a replica (Section 18.16).

### Why it exists

A transaction lives on **one database connection** (Chapter 17, Section 17.1): every statement of the transaction must be sent on the same connection, between `BEGIN` and `COMMIT`. In application code, a business operation calls several repositories, each of which needs "the connection". Something must make sure they all get the **same** connection, that the transaction ends exactly once, that an exception rolls it back, and that the connection returns to the pool afterwards. Getting any of this wrong produces half-applied operations, leaked transactions or exhausted pools.

### Mechanism: how the transaction finds the connection

Frameworks bind the current transaction (and its connection) to the **current execution context**: a thread-local variable for thread-per-request servers, or an explicit context object passed down the call chain (Chapter 11, Section 11.6).

```text
 request thread T7
 ────────────────────────────────────────────────────────────────────────────────
 OrderService.placeOrder()  [transactional proxy]
   ├─ pool.borrow() → conn c3;  c3: autocommit off (BEGIN)
   ├─ context[T7].tx = { connection: c3, status: active, rollbackOnly: false }
   │
   │   inventoryRepo.reserve(…)  → looks up context[T7].tx → runs on c3
   │   orderRepo.insert(…)       → looks up context[T7].tx → runs on c3
   │   paymentRepo.recordIntent  → looks up context[T7].tx → runs on c3
   │
   ├─ method returns normally → COMMIT on c3          (exception → ROLLBACK on c3)
   ├─ context[T7].tx = none
   └─ pool.giveBack(c3)
```

Because the binding is per thread (or per context), work handed to **another thread** inside the method (an async task, a parallel stream) does not see the transaction: it borrows a different connection and runs in autocommit or in its own transaction (the context-loss problem of Chapter 11, Section 11.6).

### Mechanism: where to put the boundary

The rule from Chapter 12, Section 12.2: **the transaction boundary belongs to the use case**, which in a layered design is the application service method.

- **Not in the controller.** The controller deals with HTTP; a transaction opened there spans request parsing and response serialisation and holds a connection for all of it.
- **Not in each repository method.** If every repository call commits on its own, a use case that calls three repositories is three transactions, and a failure in the third leaves the first two committed.
- **In the service method that implements one business operation**, so that "place an order" is one transaction containing exactly the writes that must succeed or fail together.

A use case's transaction should contain **only database work** and the in-memory logic that decides it. Network calls, file uploads, emails and message publishing belong **outside** it.

### Concrete: the anti-pattern, a remote call inside a transaction

```pseudo
@Transactional
function checkout(cartId):
    cart = carts.lockForUpdate(cartId)            // SELECT … FOR UPDATE: row lock taken
    order = orders.create(cart)                   // INSERT
    inventory.reserve(cart.items)                 // UPDATE products SET stock = stock - …
    charge = paymentGateway.charge(cart.total)    // ✘ HTTP call to a third party, 300 ms–30 s
    orders.markPaid(order, charge.id)             // UPDATE
    // commit on return
```

The timeline under normal conditions, and on a bad day:

```text
 normal day                                    bad day: payment gateway slow (8 s)
 ─────────────────────────────────────────     ──────────────────────────────────────────────
 t=0      borrow conn, BEGIN                   t=0      borrow conn, BEGIN
 t=2ms    cart row locked (FOR UPDATE)         t=2ms    cart row locked
 t=5ms    INSERT order, UPDATE stock           t=5ms    INSERT order; product rows locked
          (product rows now locked too)
 t=6ms    → HTTP POST /charges                 t=6ms    → HTTP POST /charges
            connection held, idle in tx                   connection held, idle in tx,
            locks held                                    locks on cart and products held
 t=306ms  ← 200 OK, charge ch_91               t=8.0s   ← timeout in the client (or 200 OK late)
 t=308ms  UPDATE orders SET paid …             t=8.0s   ROLLBACK (on timeout) … but did the
 t=310ms  COMMIT, return conn                           charge happen? UNKNOWN (Chapter 7, 7.15)
                                               meanwhile: every other checkout of the same
 hold time: 310 ms instead of ~8 ms                     products waits on the product row locks;
                                                        every request on this instance waits for
                                                        a connection
```

Four separate failures come from one design:

1. **Pool exhaustion by arithmetic.** By Little's Law, at 50 checkouts per second, a hold time of 8 ms needs 0.4 connections; 310 ms needs 15.5; 8 s needs **400**. A pool of 10 is exhausted at the first slowdown of a dependency the database has nothing to do with.
2. **Lock waits spread the slowdown.** The transaction holds row locks on popular products while waiting for HTTP. Other checkouts of those products queue behind it in the database (Chapter 17, Section 17.5), even on instances whose pools are fine.
3. **Inconsistent outcomes.** If the charge succeeds and the commit then fails (a deadlock, a constraint violation, a lost connection), the customer is charged for an order that does not exist. If the HTTP call times out, the transaction rolls back but the charge may have happened. The database transaction cannot roll back a third party.
4. **Long transactions** hold back VACUUM and purge (Chapter 17, Section 17.12), and appear as `idle in transaction` in `pg_stat_activity`.

### Concrete: the fix, splitting the work

```pseudo
function checkout(cartId, idempotencyKey):
    // transaction 1: short, database only
    order = tx(() ->
        cart = carts.lockForUpdate(cartId)
        o = orders.create(cart, status = PENDING_PAYMENT, key = idempotencyKey)
        inventory.reserve(cart.items)             // conditional UPDATE (Chapter 17, 17.11)
        return o)                                  // COMMIT: locks released, conn returned

    // outside any transaction: the slow, uncertain call, made safe to retry
    result = paymentGateway.charge(order.total, idempotencyKey = "order-" + order.id)

    // transaction 2: short, records the outcome
    tx(() ->
        if result.succeeded: orders.markPaid(order.id, result.chargeId)
        else:                orders.markPaymentFailed(order.id); inventory.release(order))
```

```text
 t=0      tx1 BEGIN … COMMIT (8 ms): order PENDING_PAYMENT, stock reserved, no locks after
 t=8ms    → payment call (connection NOT held; 300 ms or 8 s, it no longer matters to the pool)
 t=308ms  tx2 BEGIN … COMMIT (3 ms): order PAID
```

The order now has explicit **intermediate states**. A crash between steps leaves an order in `PENDING_PAYMENT`, which a reconciliation job resolves by asking the payment provider about the idempotency key (Chapter 13, Section 13.5). Publishing events or sending emails after the commit is handled the same way: either after commit in code (with a small risk of losing the message if the process dies first), or reliably through an **outbox** table written inside the transaction and relayed afterwards (Chapter 25). When a multi-step workflow spans services, this becomes a **saga** (Chapter 25).

### Mechanism: propagation

When a transactional method calls another transactional method, the inner method's **propagation** setting decides what happens. The names below are Spring's, but other frameworks offer the same choices:

| Propagation | If a transaction is active | If none is active | Typical use |
|---|---|---|---|
| `REQUIRED` (default) | Join it | Start one | Almost everything |
| `REQUIRES_NEW` | **Suspend** it and start an independent one on **another connection** | Start one | An audit record that must commit even if the caller rolls back |
| `NESTED` | Create a **savepoint** inside it (Chapter 17, Section 17.1) | Start one | Partial rollback of a step |
| `SUPPORTS` | Join it | Run without one | Reads that can go either way |
| `MANDATORY` | Join it | Throw | Methods that must never run outside a caller's transaction |
| `NOT_SUPPORTED` | Suspend it, run without one | Run without one | Long non-database work called from a transactional method |
| `NEVER` | Throw | Run without one | Guarding code that must not be in a transaction |

Two details catch people:

- **Joining means sharing the fate.** With `REQUIRED`, if the inner method throws an exception that the framework's rules treat as a rollback (Chapter 12, Section 12.5), the shared transaction is marked **rollback-only**, even if the outer method catches the exception. The outer method's commit then fails with an "unexpected rollback" error. Catching an exception does not undo the mark.
- **`REQUIRES_NEW` needs a second connection.** The outer transaction's connection stays borrowed and suspended while the inner one borrows another. If every request does this, a pool of size N deadlocks with N concurrent requests: each holds one connection and waits for a second, which nobody returns until the acquisition timeout fires.

```text
 pool max = 4, four concurrent requests, each: outer tx (REQUIRED) → inner tx (REQUIRES_NEW)
 R1 holds c1, waits for a 2nd   R2 holds c2, waits   R3 holds c3, waits   R4 holds c4, waits
 → nobody can proceed; all four fail after connectionTimeout. Pool-induced deadlock.
```

The fix is to avoid nested independent transactions on hot paths, or to size the pool for it (at least threads × 2 for that path, which usually means: redesign).

### Mechanism: read-only transactions

Declaring a transaction read-only does three useful things:

1. **The database can skip work.** `START TRANSACTION READ ONLY` in MySQL avoids assigning a transaction ID, and in PostgreSQL `BEGIN READ ONLY` rejects any write with `cannot execute INSERT in a read-only transaction`, which catches bugs.
2. **The ORM can skip work.** With read-only, an ORM can turn off flushing and skip dirty checking and snapshot copies, which saves memory and CPU on large reads.
3. **Routing can send it to a replica** (Section 18.16).

### Mechanism: the self-invocation pitfall, again

Chapter 12, Section 12.5 showed that a proxy intercepts only calls that go **through** it: when one method of a service calls another method of the **same** object via `this`, the transactional advice on the second method never runs. In transaction terms the consequences are specific: a method annotated `REQUIRES_NEW` that is called internally simply joins the caller's transaction (or runs in autocommit if there is none), so the "independent audit record" rolls back with its caller; a method annotated `readOnly` runs read-write; a method with a timeout runs without one. The fixes are those of Section 12.5: move the method to another component, or use programmatic transactions. The symptom is always the same: the framework's transaction debug log shows no "creating new transaction" line where you expected one.

### Concrete: programmatic form

When the boundary must be visible or dynamic, a helper that takes a function makes it explicit and exception-safe:

```pseudo
function tx(work, readOnly = false, timeoutSeconds = 5):
    c = pool.borrow(timeout = 500ms)
    try:
        c.execute(readOnly ? "BEGIN READ ONLY" : "BEGIN")
        c.execute("SET LOCAL statement_timeout = '" + timeoutSeconds + "s'")
        bindToContext(c)
        result = work()
        c.execute("COMMIT")
        return result
    catch e:
        c.executeQuietly("ROLLBACK")
        throw e
    finally:
        unbindFromContext()
        pool.giveBack(c)            // always, on every path: no leak possible
```

> **Framework lens:** Spring's `@Transactional` implements all of this through a proxy and a `PlatformTransactionManager` (`DataSourceTransactionManager` for plain JDBC, `JpaTransactionManager` for JPA), binding the connection to the thread via `TransactionSynchronizationManager`. Attributes: `propagation`, `isolation`, `readOnly`, `timeout`, `rollbackFor`. With `readOnly = true`, Spring marks the JDBC connection read-only (which some drivers translate into a read-only transaction) and sets Hibernate's flush mode to manual. `TransactionTemplate` is the programmatic form. `@TransactionalEventListener` runs a listener after the surrounding transaction commits by default, the in-process way to "do this only if the commit succeeded".

> **Other runtimes:** SQLAlchemy: `with Session(engine) as session, session.begin(): …` commits at the end of the block and rolls back on an exception. Go: `tx, err := db.BeginTx(ctx, opts)`, then `defer tx.Rollback()` (harmless after commit) and `tx.Commit()`; the transaction is a value passed explicitly, so there is no thread binding to lose. Node/Prisma: `prisma.$transaction(async (tx) => { … })`.

### Guarantees and non-guarantees

A well-placed boundary guarantees that a business operation's database writes commit or roll back together, on one connection, and that the connection returns to the pool on every path. It does not make side effects outside the database transactional, does not protect against concurrent transactions beyond the isolation level (Chapter 17), and, with declarative management, applies only to calls that go through the proxy.

### Failure and diagnosis

- **Half-applied operations.** Evidence: the operation's statements committed separately (autocommit), visible in the database log with `log_statement` or in the SQL log without a surrounding `BEGIN`. Causes: the boundary on repositories, self-invocation, work moved to another thread.
- **Idle-in-transaction sessions.** Evidence: `pg_stat_activity` rows with `state = 'idle in transaction'` and old `xact_start`, owned by your `application_name`. Cause: remote calls or slow work inside the transaction, or a leaked transaction (an exception path that skips commit and rollback, Diagnosis Lab 3).
- **"Transaction silently rolled back because it has been marked as rollback-only."** Cause: an inner joined method threw; the outer method caught it. Fix: decide which exceptions should doom the transaction; use a savepoint (`NESTED`) or a separate component for optional steps.
- **Pool deadlock with `REQUIRES_NEW`.** Evidence: all connections borrowed, each by a thread whose stack shows it waiting for a second connection inside a transactional method.

### Backend connection

Put boundaries on use cases. Keep transactions short and free of I/O other than the database: as a rule of thumb, a transaction should take milliseconds, not seconds. Give transactions a timeout. Make every intermediate state of a multi-step operation explicit in the data, so that work done outside transactions can be resumed or reconciled. And verify boundaries from the outside, with the database's own view of open transactions, rather than trusting annotations.

---

## 18.13 Lost Updates at the Application Level

### Definition

An **application-level lost update** is the lost-update anomaly of Chapter 17, Section 17.3, produced not by two SQL statements in a console but by the read-modify-write cycle of application code: load an object, change it in memory, write it back. When two requests do this concurrently on the same row, the second write overwrites the first, and one change disappears without an error.

### Why it is so common

ORMs encourage exactly this cycle: `find`, set fields, let the flush write them. The `UPDATE` the ORM sends sets **absolute values** (`SET stock = 4`), computed from what the code read earlier, not relative ones (`SET stock = stock - 1`). Two more features of application code widen the window:

- **The edit gap.** A user opens an edit form at 10:00 (read), thinks for five minutes, and submits at 10:05 (write). Another user's change at 10:02 is overwritten. No database transaction spans those five minutes, so no isolation level can help.
- **Full-row updates.** Section 18.9 noted that an ORM may write every mapped column. Two requests that change **different** fields (one changes the email, the other the phone number) each write both columns, and one change is lost.

### Concrete: the bug in ORM form

```pseudo
@Transactional
function decrementStock(productId, qty):
    p = products.findById(productId)      // SELECT … stock = 5
    if p.stock < qty: throw OutOfStock
    p.stock = p.stock - qty               // in memory: 4
    // flush: UPDATE products SET name=$1, price=$2, stock=$3 WHERE id=$4   [.., .., 4, 17]
```

```text
 request A                                   request B
 ───────────────────────────────────────     ───────────────────────────────────────
 SELECT … WHERE id=17   → stock = 5
                                             SELECT … WHERE id=17   → stock = 5
 stock = 4 in memory                         stock = 4 in memory
 UPDATE … SET stock = 4; COMMIT
                                             UPDATE … SET stock = 4; COMMIT
 two units sold; stock shows 4 instead of 3. No error anywhere.
```

At PostgreSQL's and InnoDB's default isolation levels (Read Committed and Repeatable Read respectively, Chapter 17, Section 17.4), both transactions succeed. At PostgreSQL Repeatable Read, the second `UPDATE` fails with a serialisation error, which the application must retry.

### Mechanism: the four fixes, in ORM terms

Chapter 17, Section 17.13 compared four solutions inside the database. Here is how each looks in application code, and what changes when an ORM is involved.

**1. Atomic update in the database.** Skip the read-modify-write entirely:

```sql
UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1;   -- check the row count
```

In an ORM this is a **bulk** or **modifying** query, not an entity change. It bypasses the persistence context: an entity of product 17 already loaded in the same context still shows the old stock, and a later flush of that entity could write the old value back. Run such queries before loading the entity, or clear or refresh it afterwards.

**2. Pessimistic lock.** Read with `SELECT … FOR UPDATE` so the second request waits until the first commits, then reads the new value:

```sql
SELECT id, stock FROM products WHERE id = $1 FOR UPDATE;
```

ORMs expose this as a lock mode on the find or query. The lock lasts until commit, so the transaction must be short and must not contain remote calls (Section 18.12).

**3. Optimistic lock with a version column.** Each row carries a `version`; the ORM adds it to every update's `WHERE` clause and increments it:

```sql
UPDATE products SET name=$1, price=$2, stock=$3, version = 8 WHERE id = $4 AND version = 7;
-- 1 row: success.   0 rows: someone else changed it since we read version 7.
```

The ORM checks the affected-row count and throws an optimistic-locking exception on 0. This is the only one of the four that works across the **edit gap**: carry the version to the client (in a hidden form field or an `ETag`, Chapter 13, Section 13.7) and send it back with the write. A request that submits version 7 after someone saved version 8 gets a conflict (HTTP 409 or 412) instead of silently overwriting.

**4. Serializable isolation with retry.** Run the transaction at Serializable (or PostgreSQL Repeatable Read for this case) and retry on serialisation failure (Chapter 17, Section 17.7).

### Concrete: choosing

| Situation | Fix | Reason |
|---|---|---|
| Counter, balance, stock decrement | Atomic `UPDATE … SET x = x - $1 WHERE … AND x >= $1` | No read needed; one statement; highest throughput |
| User edits a record in a form | Optimistic version + `ETag`/`If-Match` | Works across the edit gap; no lock held while a human thinks |
| Short read-decide-write with complex logic, high contention on one row | `SELECT … FOR UPDATE` | Waiting is cheaper than repeated optimistic retries (Chapter 17, Section 17.9) |
| Invariant across several rows (write skew) | Serializable + retry, or lock a common parent row | Single-row fixes cannot see other rows |
| Different fields edited concurrently | Partial updates (only changed columns) or per-field merge, plus a version | Full-row writes overwrite unrelated changes |

### Mechanism: where the retry goes

Optimistic failures, serialisation failures and deadlocks (Chapter 17, Section 17.10) all call for a **retry of the whole transaction**, and the retry must sit **outside** the transaction boundary:

```text
 retry wrapper (attempts = 3, backoff with jitter, Chapter 24)
   └── transaction boundary (BEGIN … COMMIT)
         └── use case logic: read, decide, write
```

A retry inside the transaction is useless: in PostgreSQL the transaction is already aborted after the error, and in any database the data it read is stale. A retry outside it re-reads fresh data. Only retry errors that mean "try again" (SQLSTATE `40001` serialisation failure, `40P01` deadlock, optimistic-lock exceptions), and only retry code with no side effects outside the database, or with idempotent ones (Chapter 17, Section 17.7).

### Guarantees and non-guarantees

Each fix guarantees that no write is silently lost for the row or rows it covers. Atomic updates guarantee it without retries; optimistic locking guarantees detection, not success, so callers must handle conflicts; pessimistic locking guarantees success after a wait, but not a bounded wait. None of them guarantees correctness for invariants spanning rows the mechanism does not touch, and none works if some code path writes the row without using it (a bulk `UPDATE` that does not increment the version defeats optimistic locking for everyone).

### Failure and diagnosis

- **Silent lost updates.** Symptom: counts that drift (stock, likes, balances) under load; customer reports that "my change disappeared". Evidence: audit or history tables showing two updates computed from the same old value; no errors in logs. Fix: one of the four, by the table above.
- **Optimistic-lock exceptions surfacing as 500 errors.** Evidence: `OptimisticLockException` or "Row was updated or deleted by another transaction" in logs. Fix: map them to 409 with a clear message for user edits, or retry automatically for system updates.
- **Versions bypassed.** Evidence: a batch job or a SQL script updating rows without touching `version`; conflicts not detected after it runs. Fix: every writer increments the version, enforced in review (or with a trigger).

### Backend connection

Look for read-modify-write cycles in every ORM write path, and pick the fix by the table above. For anything a human edits, carry the version end to end, from the database row to the HTTP `ETag` and back. For counters and stock, prefer atomic SQL over entity updates. Put retries outside transaction boundaries, and only for errors that mean "try again".

---

## 18.14 Batching, Bulk Operations and Streaming Large Results

### Definitions

- **Statement batching** is sending many executions of the same prepared statement, each with its own parameters, in one round trip (or a few), instead of one round trip per row.
- A **multi-row statement** is one SQL statement that affects many rows at once, such as `INSERT … VALUES (…), (…), (…)` or a set-based `UPDATE … WHERE …`.
- **Bulk loading** is using the database's dedicated high-speed path for large imports: `COPY` in PostgreSQL, `LOAD DATA` in MySQL.
- **Streaming** a result means processing rows as they arrive in batches, with bounded memory, instead of loading the whole result first (Section 18.1). It uses a **cursor**: a database-side position in a result that the client advances by fetching the next batch.
- The **fetch size** is how many rows the driver asks for per batch while streaming.

### Why it exists

Two costs dominate large data operations from an application: **round trips** and **memory**. With a round trip of 0.5 ms, inserting 100,000 rows one statement at a time spends **50 seconds** just waiting on the network, even if each insert takes microseconds on the server. Reading 5 million rows of about 200 bytes each into memory needs at least **1 GB** of heap, before the language's per-object overhead (often several times the raw size). Batching removes the round trips; streaming bounds the memory.

### Concrete: the insert ladder

Inserting **100,000** rows, round trip 0.5 ms (illustrative orders of magnitude):

```text
 method                                         round trips      typical total time
 ───────────────────────────────────────────────────────────────────────────────────
 one INSERT per row, autocommit                 100,000 + a       ~1–2 minutes (round trips
                                                commit (WAL                    + 100,000 commit
                                                flush) per row                 flushes, Ch 15, 15.4)
 one INSERT per row, one transaction            100,000           ~50–60 s (round trips)
 batched prepared INSERT, 1,000 rows per batch  ~100              ~1–3 s
 multi-row INSERT … VALUES (…) × 1,000          ~100              ~1–2 s
 COPY / LOAD DATA (streamed file or rows)       a handful         well under a second to ~1 s
```

Each step up the ladder removes a cost: autocommit's per-row commit flush, then per-row round trips, then per-statement parsing and execution overhead. `COPY` moves rows in a compact format through a single command with minimal per-row work.

### Mechanism: batching in drivers and ORMs

```pseudo
ps = conn.prepare("INSERT INTO events (id, user_id, kind, at) VALUES ($1, $2, $3, $4)")
for chunk in rows.chunked(1000):
    for r in chunk: ps.addBatch(r.id, r.userId, r.kind, r.at)
    ps.executeBatch()              // sent together; the driver may rewrite into multi-row INSERTs
conn.commit()
```

Several details decide whether batching really happens:

- **Driver rewriting.** Some drivers send batched executions as separate statements pipelined in one round trip; others can rewrite them into multi-row `INSERT`s when an option is enabled (pgJDBC's `reWriteBatchedInserts`, MySQL Connector/J's `rewriteBatchedStatements`, both off by default at the time of writing).
- **ORM batching is opt-in and fragile.** The ORM must be told a batch size, must group statements of the same shape (ordering inserts and updates by entity type helps), and cannot batch inserts for entities whose ids come from an **identity column**, because it must run each insert immediately to learn the id (Section 18.8). Sequence-based ids with block allocation keep batching possible.
- **The persistence context grows.** Inserting 100,000 entities through an ORM keeps 100,000 managed objects and snapshots in memory and makes every flush slower (Section 18.9). Flush and **clear** the context every batch:

```pseudo
for i, row in enumerate(rows):
    session.persist(Event.from(row))
    if i % 1000 == 999:
        session.flush()            // send the batch
        session.clear()            // forget managed objects: memory stays flat
```

### Mechanism: bulk updates and deletes, in chunks

Updating or deleting many rows is best done **set-based** in SQL, not by loading entities:

```sql
-- one statement instead of loading and saving 2 million entities
UPDATE orders SET status = 'ARCHIVED' WHERE status = 'DELIVERED' AND delivered_at < now() - interval '2 years';
```

But one statement over millions of rows is one long transaction: it holds locks on every changed row, generates a large burst of WAL (which replicas must replay, Chapter 22), and holds back VACUUM (Chapter 17, Section 17.12). The standard answer is **chunking** by key range, each chunk its own short transaction:

```pseudo
lastId = 0
loop:
    ids = tx(() -> db.query(
        "UPDATE orders SET status = 'ARCHIVED'
         WHERE id IN (SELECT id FROM orders
                      WHERE id > $1 AND status = 'DELIVERED' AND delivered_at < now() - interval '2 years'
                      ORDER BY id LIMIT 5000)
         RETURNING id", [lastId]))
    if ids.isEmpty(): break
    lastId = max(ids)
    sleep(50ms)                    // leave room for normal traffic and replication
```

Each chunk is a keyset step (Chapter 13, Section 13.6), so the job never re-scans finished rows, can stop and resume from `lastId`, and keeps every transaction short.

### Mechanism: streaming large results

To read a result too large for memory, the client must ask for it in batches. There are three ways, from the protocol up:

1. **Driver-level fetch size.** The driver asks the server for N rows at a time (in PostgreSQL, `Execute` with `max_rows` on a portal, Section 18.1). In pgJDBC this works only inside a transaction (autocommit off) with a forward-only result and a positive fetch size; otherwise the driver silently buffers everything. MySQL Connector/J streams row by row when the fetch size is set to `Integer.MIN_VALUE`, or uses server cursors with `useCursorFetch=true`.
2. **Explicit SQL cursors.** `DECLARE c CURSOR FOR SELECT …; FETCH 1000 FROM c; … CLOSE c;` inside a transaction (PostgreSQL), which works with any driver.
3. **Keyset pagination for jobs.** Read `WHERE id > $last ORDER BY id LIMIT 5000` repeatedly, each query a short statement. Unlike a cursor, it holds no transaction or snapshot open between batches, at the cost of not seeing one consistent snapshot of the whole table.

```text
 approach               memory      connection held     snapshot             survives restarts
 ────────────────────────────────────────────────────────────────────────────────────────────
 buffered result        whole set   for the query       one, short           no
 fetch size / cursor    one batch   whole duration      one, LONG            no
 keyset batches         one batch   per batch           one per batch        yes (resume from key)
```

The middle row hides a cost: streaming a 2-hour export through one cursor is a **2-hour transaction**, with all the effects of Chapter 17, Section 17.12 (VACUUM held back, `idle in transaction` between fetches if the consumer is slow). For long jobs, keyset batches are usually better; for an export that must be a consistent snapshot, run it on a replica or with an exported snapshot.

When streaming into an HTTP response (a CSV download), the database cursor, the application and the client are a pipeline: a slow client fills the socket buffer, the application stops fetching, and the cursor's transaction stays open (Chapter 10, Section 10.6 on streaming with backpressure). Give such endpoints their own connection budget and a maximum duration.

### Guarantees and non-guarantees

Batching guarantees fewer round trips; it does not make a batch atomic unless it runs in one transaction, and a failure in the middle of `executeBatch` may leave some rows applied (in autocommit) or the whole transaction aborted (in PostgreSQL). Streaming guarantees bounded client memory; it does not bound how long the connection and snapshot are held. Chunking guarantees short transactions; it does not give one consistent view of the data across chunks.

### Failure and diagnosis

- **Slow imports.** Evidence: database CPU and disk barely used while the job runs for an hour; one statement per row in the SQL log; one commit per row (high WAL flush counts in `pg_stat_wal`). Fix: batching or `COPY`, one transaction per chunk.
- **Batching silently disabled.** Evidence: the ORM's batch statistics show batch size 1; identity-column ids. Fix: sequence ids, insert ordering, driver rewrite options.
- **Out-of-memory in exports.** Evidence: heap dumps full of driver row buffers or managed entities. Fix: real streaming (check autocommit and fetch size), or keyset batches with a cleared context.
- **Replication lag and bloat after a bulk change.** Evidence: replica lag spike (Section 18.16) and dead tuples after a single huge `UPDATE`. Fix: chunk with pauses.

### Backend connection

Never process large data row by row over the network. Insert with batches or `COPY`; update and delete set-based, in key-ordered chunks with short transactions; read with streaming or keyset batches, and know which one holds a transaction open. Run long jobs with their own pool and connection budget so they cannot starve request traffic.

---

## 18.15 Schema Migrations

### Definitions

A **schema migration** is a script that changes the database schema (tables, columns, indexes, constraints) or transforms existing data from one version to the next. **Versioned migrations** are an ordered series of such scripts, each with a unique, increasing version (`V1__create_orders.sql`, `V2__add_status_index.sql`), applied exactly once per database in order, and recorded in a **schema history table** in the database itself. A **migration tool** (Flyway, Liquibase, Alembic, the migration systems of Rails and Django, Prisma Migrate) reads the history table, finds which scripts have not run yet, and runs them.

A **zero-downtime migration** is a schema change performed while the application keeps serving traffic, with no period in which requests fail because the code and the schema disagree or because a lock blocks them.

**Expand/contract** (also called parallel change) is the technique that makes most zero-downtime changes possible: first **expand** the schema so that it supports both the old and the new code, then migrate code and data, and only then **contract** it by removing what the old code needed. Chapter 13, Section 13.9 used the same idea for API fields.

### Why it exists

Schema changes made by hand ("run this `ALTER` on production before the deploy") drift between environments, get forgotten, and cannot be reviewed or repeated. Versioned migrations make the schema part of the codebase: every environment reaches the same schema by running the same scripts in the same order, and the history table records what was applied.

Zero-downtime techniques exist because two things are true during every deploy of a running service:

1. **Old and new code run at the same time.** A rolling deploy (Chapter 26) replaces instances one by one, so for minutes both versions serve traffic against **one** schema. Any schema change must be compatible with both.
2. **DDL takes locks.** Many schema changes need a strong table lock, and a lock request waits in a queue behind long-running queries while blocking everything that arrives after it (Chapter 17, Section 17.5). A change that takes milliseconds can stop a table for minutes.

### Mechanism: how a migration tool works

```text
 application startup (or a separate migration job, preferred)
   1. connect as the migration user (DDL rights, unlike the app user: Section 18.7)
   2. take a lock so that only ONE migrator runs (concurrent instances wait or skip)
        (a lock table row, or an advisory lock on PostgreSQL)
   3. read the history table:
        version | description         | checksum   | installed_on        | success
        1       | create orders       | 9f2a…      | 2026-03-02 10:11:05 | true
        2       | add status index    | 41bc…      | 2026-04-17 08:30:41 | true
   4. compare with scripts on the classpath / in the repo:  V1, V2, V3, V4
        - checksum of V1, V2 must match what was recorded (edited history ⇒ fail)
        - V3, V4 are pending
   5. run V3 (in a transaction where the database allows), record it; then V4
   6. release the lock
```

Rules that follow:

- **Never edit an applied migration.** The checksum check fails, and other environments have already run the old version. Fix forward with a new migration.
- **Prefer forward-only.** "Down" migrations are rarely tested and often cannot restore data (a dropped column's contents are gone). Recovery is usually a new forward migration.
- **Know your database's DDL transactions.** In PostgreSQL most DDL is transactional, so a failed migration rolls back cleanly (except statements that cannot run in a transaction, such as `CREATE INDEX CONCURRENTLY`, which need a migration marked to run outside one). In MySQL, DDL statements cause an **implicit commit**, so a migration that fails halfway leaves the earlier statements applied and the history table must be repaired by hand.
- **Run migrations as a separate step,** before the new code rolls out, rather than from every starting instance. Otherwise a slow migration delays every instance's startup, and health checks may kill instances in the middle of it.

### Concrete: renaming a column with zero downtime

Goal: rename `orders.shipping_address` to `delivery_address`. A plain rename is instant in the database but **breaks every running old instance** the moment it commits: their queries name a column that no longer exists. The expand/contract sequence:

```text
 step  change                                          code running          schema has
 ────────────────────────────────────────────────────────────────────────────────────────
 0     (start)                                         v1: reads/writes old  old
 1     MIGRATION (expand):                             v1                    old + new (empty)
         ALTER TABLE orders ADD COLUMN delivery_address text;
         (nullable, no default: metadata-only)
 2     DEPLOY v2: writes BOTH columns;                 v1 and v2 mixed,      old + new
         reads old (new may be empty for old rows)     then v2 only
 3     BACKFILL in chunks (Section 18.14):             v2                    old + new (filled)
         UPDATE orders SET delivery_address = shipping_address
         WHERE id BETWEEN $1 AND $2 AND delivery_address IS NULL;
 4     DEPLOY v3: reads NEW, still writes both         v2 and v3 mixed,      old + new
         (so a rollback to v2 still finds old data)    then v3 only
 5     DEPLOY v4: reads and writes NEW only            v3 and v4 mixed,      old + new
         (v3 still writes both, so nothing is lost)    then v4 only          (old unused)
 6     verify: no code reads or writes the old column (query logs, pg_stat_statements,
         grep); wait at least one release cycle
 7     MIGRATION (contract):                           v4                    new only
         ALTER TABLE orders DROP COLUMN shipping_address;
```

Why each step is safe:

- At every moment, **every running version** finds the columns it uses. v1 never sees the new column (it ignores it); v4 never needs the old one.
- **Writes are duplicated** for as long as any running version, or any version you might roll back to, reads the old column. That is why v3 writes both even though it reads only the new one.
- The **backfill** runs after all writers write both columns, so no row can be written to the old column only after the backfill passes it.
- The **contract** step is irreversible, so it waits until nothing could need the old column, including a rollback.

A variation replaces steps 2 to 4 with a **database trigger** that copies writes from one column to the other, so the application changes once; it moves logic into the database and must be removed during the contract step. Either way, a rename costs several deploys. Teams that deploy often consider that normal; teams that do not are tempted into the one-step rename and its outage.

### Mechanism: locking DDL hazards

Even an "expand" step can cause an outage if its DDL locks a table for long or waits in the lock queue. The main hazards, with PostgreSQL behaviour (MySQL notes follow):

| Change | Hazard | Safer method |
|---|---|---|
| Any `ALTER TABLE` | Needs an `ACCESS EXCLUSIVE` lock (for most forms); waits behind a long query and **blocks every later query** on the table meanwhile (Chapter 17, Section 17.5) | `SET lock_timeout = '2s'` in the migration, then retry; run when no long queries are active |
| `ADD COLUMN` with a constant default | Metadata-only since PostgreSQL 11; older versions rewrote the table | Fine on current versions; volatile defaults (`DEFAULT random()`) still rewrite |
| `ALTER COLUMN … TYPE` | Usually **rewrites the whole table** under an exclusive lock (exceptions: some widenings such as a larger `varchar` limit) | Expand/contract with a new column and a backfill |
| `SET NOT NULL` | Scans the whole table under the exclusive lock | Add `CHECK (col IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT` (weaker lock), then `SET NOT NULL`, which PostgreSQL 12+ can prove from the validated check |
| `ADD FOREIGN KEY` / `ADD CHECK` | Validates every row while holding locks | `NOT VALID`, then `VALIDATE CONSTRAINT` separately (Chapter 14, Section 14.3) |
| `CREATE INDEX` | Blocks writes for the whole build | `CREATE INDEX CONCURRENTLY` (outside a transaction; check for an invalid index on failure; Chapter 16, Section 16.7) |
| `ADD UNIQUE` / `PRIMARY KEY` constraint | Builds a unique index while blocking writes | Build the unique index concurrently, then attach it as a constraint (`… USING INDEX`) |
| Large data `UPDATE` in the migration | One long transaction: locks, WAL burst, replica lag | Chunked backfill outside the migration (Section 18.14) |
| `RENAME`, `DROP COLUMN` | Fast, but break old code instantly | Expand/contract |

The lock-queue mechanism deserves one more look, because it turns a fast statement into an outage:

```text
 10:00:00  analytics query on orders, running 4 min       holds ACCESS SHARE
 10:01:00  migration: ALTER TABLE orders ADD COLUMN …      wants ACCESS EXCLUSIVE → WAITS
 10:01:00+ every SELECT/INSERT/UPDATE on orders            queue BEHIND the ALTER → WAIT
           → connection pools exhaust; the site is down for 3 minutes
 10:04:00  analytics query ends → ALTER runs (5 ms) → queue drains
 with lock_timeout = 2s: at 10:01:02 the ALTER gives up; traffic resumes; the
 migration retries later.
```

**MySQL notes.** InnoDB's online DDL can perform many changes without blocking writes (`ALGORITHM=INSTANT` for adding columns in MySQL 8.0, `ALGORITHM=INPLACE, LOCK=NONE` for others), but every DDL statement still needs a brief exclusive **metadata lock** at its start and end. A long-running transaction on the table makes the DDL wait, and everything behind it waits too: the process list shows `Waiting for table metadata lock`. Tools such as gh-ost and pt-online-schema-change perform large changes by copying into a shadow table and swapping it in.

### Guarantees and non-guarantees

A migration tool guarantees that each migration runs once per database, in order, and that applied scripts were not edited. Expand/contract guarantees that every running code version finds the schema it expects. Neither guarantees that a DDL statement takes no locks, that a long data change does not cause replica lag, or that a migration can be undone; and in MySQL, the tool cannot make a multi-statement DDL migration atomic.

### Failure and diagnosis

- **Migration lock outage.** Symptom: during a deploy, all requests touching one table hang, pools exhaust, then everything recovers at once. Evidence: `pg_stat_activity` / `pg_locks` show the `ALTER TABLE` waiting for an `AccessExclusiveLock` and dozens of sessions waiting behind it (`pg_blocking_pids`, Chapter 17, Section 17.14); MySQL shows `Waiting for table metadata lock`. Diagnosis Lab 4 walks through it.
- **Old code breaks after a migration.** Errors such as `column "shipping_address" does not exist` from instances still on the previous version, or `cached plan must not change result type` (Section 18.6). Fix: expand/contract; avoid `SELECT *` in application queries.
- **Failed half-applied migration (MySQL).** Evidence: the history table records a failure; some of the script's DDL is present. Fix: repair manually, then make migrations one DDL statement each.
- **Startup timeouts.** Evidence: instances killed by health checks while running a long migration at startup. Fix: separate migration step.

### Backend connection

Treat migrations as code: versioned, reviewed, tested against a production-sized copy, and run as a separate deploy step by a privileged user the application does not have. Design every schema change for two code versions at once. Put `lock_timeout` (or MySQL's `lock_wait_timeout`) in every migration, avoid table rewrites on large tables, and split dangerous changes into expand, migrate and contract steps, accepting several deploys as the price of no downtime.

---

## 18.16 Read/Write Splitting and Replica Lag

### Definitions

A **replica** (read replica, standby) is a copy of the database that continuously applies the changes made on the **primary**, the one server that accepts writes. Chapter 22 covers replication in depth; this section needs only its working definition and one property.

**Read/write splitting** is routing an application's writes (and reads that must be current) to the primary, and other reads to one or more replicas, so that read traffic scales beyond one server.

**Replication lag** is the delay between a change committing on the primary and becoming visible on a replica. With the usual **asynchronous** replication, it is normally milliseconds but has no upper bound: a large transaction, a busy replica or a network problem can stretch it to seconds or minutes.

The **read-your-writes** (read-after-write) problem is the anomaly in which a user makes a change and then, reading from a lagging replica, does not see it.

### Why it exists

Many applications read far more than they write. One primary has a fixed number of cores; replicas add read capacity, isolate heavy reporting queries from transactional traffic, and can serve users in other regions. The price is that a replica is a **slightly old** copy, and the application must decide, query by query, whether slightly old is acceptable.

### Mechanism: routing in the application

```text
                          ┌───────────── application ─────────────┐
   POST /profile ───────► │ write path → primary pool             │──► primary  (reads + writes)
   GET  /profile ───────► │ read path  → replica pool             │──► replica 1 ┐ apply the
   GET  /reports ───────► │ read-only tx → replica pool           │──► replica 2 ┘ primary's WAL
                          └───────────────────────────────────────┘     (lag: ms … minutes)
```

Common implementations:

- **Two pools** (one per server role), chosen explicitly by repository or use case.
- **A routing data source** that inspects the current transaction: read-only transactions (Section 18.12) go to a replica pool, read-write ones to the primary pool. This makes the read-only flag decide correctness, so it must be set deliberately.
- **A proxy** (ProxySQL for MySQL, some PostgreSQL poolers and cloud proxies) that routes by statement type. It cannot know which reads need fresh data, so it needs hints.

### Concrete: the read-your-writes anomaly

```text
 user            application                  primary                   replica (lag 800 ms)
 ─────────────────────────────────────────────────────────────────────────────────────────
 POST /profile   UPDATE users SET name='Ana'
 name = "Ana"    WHERE id=81; COMMIT  ───────► name = Ana (t=0)
                 303 See Other → /profile
 GET /profile    read-only → replica  ──────────────────────────────────►  name = Anna (old)
                 renders "Anna"                                           (applies the change
 "my change                                                                at t=800 ms)
  did not save!"  user edits again …
```

No component misbehaved: the replica is doing what asynchronous replication promises. The anomaly is a consequence of routing this particular read to it.

### Mechanism: mitigations

| Technique | How | Cost |
|---|---|---|
| Read from the primary after a write | For N seconds after a user writes (track in the session or a cookie), route that user's reads to the primary | N must exceed realistic lag; more primary load |
| Route by need | Reads that drive decisions or show the user's own data go to the primary; listings, search and other users' data to replicas | Requires classifying queries |
| Wait for the replica to catch up (LSN-based) | After the write, record the primary's WAL position (`pg_current_wal_lsn()`); before reading on a replica, check `pg_last_wal_replay_lsn()` ≥ that position, else wait briefly or use the primary | Extra queries; precise |
| Synchronous replication for the relevant replica | The primary waits for the replica before confirming commit (Chapter 22) | Higher commit latency; availability depends on the replica |

Never use replica reads for a **read-then-write decision** (checking stock or a balance before writing): the decision can be based on data the primary has already changed. Decisions read from the primary, inside the writing transaction (Chapter 17).

### Mechanism: when the primary changes

After a **failover** (a replica promoted to primary, Chapter 22), pooled connections to the old primary are broken or, worse, connected to a server that is now read-only, where writes fail with `cannot execute UPDATE in a read-only transaction` (SQLSTATE `25006`). Drivers can be given several hosts and told to connect only to the one that accepts writes (libpq's `target_session_attrs=read-write` or `primary`, pgJDBC's `targetServerType=primary`), and pools must retire connections that hit such errors. DNS-based failover is limited by DNS caching in the application runtime (Chapter 6, Section 6.6), and a pool's maximum lifetime bounds how long stale connections survive.

### Guarantees and non-guarantees

Read/write splitting guarantees more read capacity and isolation of read load. It does not guarantee that a replica read reflects any particular recent write, that lag is bounded, or that replica reads are consistent with each other across different replicas (two consecutive reads may go to replicas with different lag, so data can appear to go backwards in time; Chapter 22 names this a monotonic-reads violation).

### Failure and diagnosis

- **"My change did not save" reports.** Evidence: the write committed on the primary; the next read was routed to a replica whose lag at that moment exceeded the gap. Measure lag on PostgreSQL with `pg_stat_replication.replay_lag` on the primary, or `now() - pg_last_xact_replay_timestamp()` on the replica (meaningful only while writes are flowing); on MySQL, `Seconds_Behind_Source` in `SHOW REPLICA STATUS`.
- **Lag spikes** after bulk operations or during long queries on the replica (PostgreSQL may delay replay for conflicting queries, Chapter 17, Section 17.12). Evidence: lag graph correlated with job schedules. Fix: chunked bulk changes (Section 18.14).
- **Writes failing after failover.** Evidence: `read-only transaction` errors from one instance group; connection strings pointing at a fixed host. Fix: multi-host connection strings with a primary target, pool eviction on that error.

### Backend connection

Split reads only where staleness is acceptable, and decide it per use case, not per statement type. Implement read-your-writes for a user's own data, monitor lag as a first-class metric with an alert, and route decision-making reads to the primary. Chapter 22 explains why lag exists, the consistency models that describe it, and how failover works.

---

## 18.17 Diagnosis Toolkit: Seeing the Application from the Database

### Definition

The **application–database diagnosis toolkit** is the set of views and metrics that let you see application behaviour from both sides of the connection: the pool's own metrics on the application side, and the database's per-session and per-statement views on the other, joined by `application_name`, timing and request IDs.

### Why it matters

Each failure in this chapter has a matching signature on both sides. Pool exhaustion is visible as pending borrowers in the application and as sessions in a specific state in the database; N+1 as many statements per request and as huge call counts; leaked transactions as idle-in-transaction sessions; migration outages as lock queues. Reading both sides is what turns "the database is slow" into a specific root cause.

### Concrete: who holds the connections

```sql
-- PostgreSQL: connections per service and state
SELECT application_name, state, count(*),
       max(now() - xact_start)  AS oldest_tx,
       max(now() - state_change) AS longest_in_state
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY application_name, state
ORDER BY count(*) DESC;
--  application_name |        state        | count | oldest_tx | longest_in_state
-- ------------------+---------------------+-------+-----------+------------------
--  orders-api       | idle                |    41 |           | 00:12:03
--  orders-api       | idle in transaction |     9 | 00:04:51  | 00:04:50
--  orders-api       | active              |     4 | 00:00:01  | 00:00:01
--  reports          | active              |     2 | 00:03:10  | 00:03:10
```

How to read the states:

| State | Meaning | If many |
|---|---|---|
| `active` | Running a statement right now | Slow queries or too much concurrency: look at the queries and wait events |
| `idle in transaction` | Inside a transaction, not running anything: the application holds the connection and is doing something else | Remote calls or slow code inside transactions, or leaked transactions |
| `idle in transaction (aborted)` | As above, after an error; must roll back | Error handling that does not roll back |
| `idle` | Connected, no transaction | Normal pool idles; too many means oversized pools |

MySQL's equivalents are `SHOW PROCESSLIST` (or `performance_schema.threads`), with the command `Sleep` for idle connections, and `information_schema.innodb_trx` for open transactions and their start times (Chapter 17, Section 17.14).

### Concrete: which statements cost the most

```sql
SELECT calls, round(total_exec_time) AS total_ms, round(mean_exec_time::numeric, 2) AS mean_ms,
       rows / greatest(calls, 1) AS rows_per_call, left(query, 60) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 5;
```

Sorted by **total** time, it finds the statements that dominate database load; sorted by **calls**, it finds N+1 fingerprints (Section 18.11); a high `rows_per_call` with a small page size points at missing `LIMIT`s or fan-out joins. Because `pg_stat_statements` normalises parameters, a codebase that binds values produces a short, readable list, while one that concatenates them produces thousands of near-duplicates: one more reason to bind (Section 18.6).

### Concrete: the signature table

| Problem | Application side | Database side |
|---|---|---|
| Pool exhaustion (slow queries) | active = max, pending > 0, acquisition time = timeout, usage time high | Many `active` sessions for this app; long-running queries; maybe lock waits |
| Pool exhaustion (work inside transactions) | As above | Many `idle in transaction` sessions with growing `oldest_tx` |
| Connection leak | active = max over hours, never drops at night; leak warnings | Sessions `idle` (not in a transaction) for a long time, matching the "borrowed" count |
| N+1 | High statements per request; latency grows with page size | Huge `calls` of a PK lookup, tiny `mean_exec_time`, call ratio ≈ items per page |
| Leaked transaction | Pool slowly loses connections; lock-wait errors on specific rows | One or more `idle in transaction` sessions hours old, holding locks (`pg_blocking_pids`) |
| Migration lock outage | Every request touching one table times out at once during a deploy | DDL session waiting for `AccessExclusiveLock`, a queue of sessions behind it |
| Connection limit | Errors on connect, during deploys or scale-out | Connection count at `max_connections`; `too many clients` in the log |

### Concrete: settings that make the evidence exist

```text
 application:  application_name per service (and instance) in the connection string
               pool metrics exported; leak detection threshold set (e.g. 2× the longest legit hold)
               per-request statement count in logs or traces; SQL logging in tests
 PostgreSQL:   pg_stat_statements enabled; log_min_duration_statement (e.g. 500 ms)
               log_lock_waits = on; idle_in_transaction_session_timeout; statement_timeout per role
               log_connections / log_disconnections when chasing connection churn
 MySQL:        slow query log with long_query_time; performance_schema; innodb_print_all_deadlocks
```

### Backend connection

Set `application_name`, export pool metrics, and enable `pg_stat_statements` before you need them. In an incident, look at both sides: the pool tells you that requests are waiting, and the database tells you what the borrowed connections are doing.

---

## 18.18 Placement-Round Topics: JDBC, Cursors and Embedded SQL

> **Placement-round focus:** This section is revision material for campus placement interviews and written tests, which often ask JDBC and DBMS-course questions in a specific form. It is not part of the main flow. What carries into backend work: the difference between `Statement` and `PreparedStatement` (Section 18.7), and that a cursor is how large results are streamed (Section 18.14).

### ODBC and JDBC

**ODBC** (Open Database Connectivity) is a language-independent C API, originally from Microsoft, for connecting programs to databases through drivers. **JDBC** (Java Database Connectivity) is Java's equivalent: a set of interfaces in `java.sql` that every Java database driver implements. Both let code use one API against many databases, with a database-specific driver underneath.

### The four JDBC driver types

| Type | Name | How it works | Today |
|---|---|---|---|
| 1 | JDBC–ODBC bridge | Translates JDBC calls into ODBC calls | Removed from the JDK (Java 8) |
| 2 | Native-API (partly Java) | Calls the database's native client library (C) through JNI | Rare; needs native libraries on every machine |
| 3 | Network protocol (pure Java, middleware) | Talks a generic protocol to a middleware server, which talks to the database | Rare |
| 4 | Thin driver (pure Java, native protocol) | Implements the database's wire protocol directly in Java (Section 18.1) | Standard: pgJDBC, MySQL Connector/J |

### The JDBC steps and the three statement kinds

This subsection is a Java lens by nature, because JDBC is Java's API. The classic sequence asked in tests:

```java
// 1. load/register driver (automatic since JDBC 4.0 via the service loader)
// 2. open a connection (in real servers: from a pooled DataSource, Section 18.3)
try (Connection con = dataSource.getConnection();
     // 3. create a statement
     PreparedStatement ps = con.prepareStatement(
         "SELECT id, status FROM orders WHERE customer_id = ?")) {
    ps.setLong(1, 81);                                   // bind
    // 4. execute
    try (ResultSet rs = ps.executeQuery()) {
        // 5. process results
        while (rs.next()) { long id = rs.getLong("id"); String s = rs.getString("status"); }
    }
}   // 6. close: try-with-resources closes rs, ps, con (con returns to the pool)
```

| Interface | Use | Notes |
|---|---|---|
| `Statement` | Static SQL with no parameters | Concatenating values into it causes SQL injection (Section 18.7) |
| `PreparedStatement` | Parameterised SQL, executed one or many times | Binding prevents injection; may reuse plans (Section 18.6) |
| `CallableStatement` | Calling stored procedures: `{call transfer(?, ?, ?)}` | Supports OUT parameters (Chapter 14, Section 14.14 defines procedures) |

`executeQuery` returns a `ResultSet` (for `SELECT`), `executeUpdate` returns an affected-row count (for `INSERT`/`UPDATE`/`DELETE`/DDL), and `execute` returns a boolean saying whether a result set is available. A `ResultSet` has a **type** (`TYPE_FORWARD_ONLY`, `TYPE_SCROLL_INSENSITIVE`, `TYPE_SCROLL_SENSITIVE`) and a **concurrency** (`CONCUR_READ_ONLY`, `CONCUR_UPDATABLE`). Transactions: `con.setAutoCommit(false)`, then `con.commit()` or `con.rollback()`; `con.setSavepoint()` and `con.setTransactionIsolation(...)` map to Chapter 17's concepts. `DriverManager.getConnection` opens a new physical connection each time; a `DataSource` is the factory interface that pools implement.

### Cursors

A **cursor** is a named, database-side pointer into the result of a query, used to process the result row by row or in batches. In SQL and in procedural extensions (PL/pgSQL, PL/SQL) it has a fixed lifecycle:

```sql
BEGIN;
DECLARE order_cur CURSOR FOR
    SELECT id, total_cents FROM orders WHERE status = 'PLACED' ORDER BY id;
FETCH 100 FROM order_cur;      -- rows 1–100
FETCH 100 FROM order_cur;      -- rows 101–200
CLOSE order_cur;
COMMIT;
```

Textbook terms: an **implicit cursor** is the one the database opens automatically for each single SQL statement (in Oracle PL/SQL, inspected through `SQL%ROWCOUNT`, `SQL%FOUND`); an **explicit cursor** is declared by the programmer with the DECLARE / OPEN / FETCH / CLOSE lifecycle. Cursors can be forward-only or **scrollable** (`SCROLL`, allowing `FETCH PRIOR`, `FETCH ABSOLUTE n`), and **insensitive** (a snapshot of the result) or **sensitive** to concurrent changes. A cursor `WITH HOLD` survives the transaction's commit in PostgreSQL, at the cost of materialising its result. The backend lesson is Section 18.14's: a cursor holds a transaction, a snapshot and a connection for as long as it is open.

### Embedded SQL and dynamic SQL

**Embedded SQL** is SQL written directly inside a host-language program (C, COBOL, older Java with SQLJ), marked with `EXEC SQL … ;`, and translated by a **precompiler** into ordinary library calls before compilation. Host variables are referenced with a colon (`:custId`), and status is reported through `SQLCODE`/`SQLSTATE`. **Static SQL** is fixed at compile time (the precompiler can check and even plan it); **dynamic SQL** is built as a string at run time and prepared then (`EXEC SQL PREPARE …`, `EXECUTE …`), which is more flexible and, if values are concatenated rather than bound, open to injection. Modern drivers and ORMs replaced precompilers, but the static/dynamic distinction is the same one as prepared statements versus string-built SQL.

### Drill for this section

**C1. What are the four types of JDBC drivers? Which one is used today and why?**
A strong answer must mention:
- Type 1 JDBC–ODBC bridge, Type 2 native API, Type 3 network/middleware, Type 4 thin pure-Java native protocol.
- Type 4 is standard: no native libraries or middleware, implements the wire protocol directly.
- Type 1 was removed from the JDK in Java 8.
*Common weak answer:* listing the names without saying what each translates to, or claiming Type 1 is still common.

**C2. Compare Statement, PreparedStatement and CallableStatement.**
A strong answer must mention:
- `Statement`: static SQL, values concatenated, injection risk, parsed every time.
- `PreparedStatement`: placeholders and binding, no injection through values, possible plan reuse, batching.
- `CallableStatement`: stored procedures with IN/OUT parameters.
*Common weak answer:* "PreparedStatement is faster", without the security reason, which is the more important one.

**C3. What is a cursor? Distinguish implicit and explicit cursors, and explain the lifecycle.**
A strong answer must mention:
- A database-side pointer into a query result, processed in rows or batches.
- Implicit: opened automatically per statement; explicit: declared, opened, fetched, closed by the programmer.
- Scrollable versus forward-only; sensitive versus insensitive.
- A cursor holds a transaction and a connection while open.
*Common weak answer:* "a cursor is a loop over rows", missing that it is server-side state with a cost.

**C4. What is embedded SQL? How do static and dynamic SQL differ?**
A strong answer must mention:
- SQL inside a host program, translated by a precompiler, host variables with `:`.
- Static SQL is fixed at compile time; dynamic SQL is built and prepared at run time.
- Dynamic SQL is injectable if values are concatenated instead of bound.
*Common weak answer:* treating dynamic SQL as unsafe by definition; it is safe when values are bound.

---

## Chapter Summary

An application reaches its database through a **driver**, a library that speaks one database's **wire protocol**. Connecting to PostgreSQL costs a TCP handshake, a TLS negotiation and handshake, a SCRAM authentication exchange and the **fork of a backend process**, several milliseconds in one data centre and hundreds across regions, while a query often costs less than a millisecond. Queries run either as one text message (simple protocol) or as **Parse, Bind, Execute** steps (extended protocol), and drivers often **buffer whole results** unless asked to stream. That cost gap is why applications use a **connection pool**: a bounded set of open connections, lent out and returned, with an **acquisition timeout**, validation and a maximum lifetime against dead connections, a **reset** on return so session state does not leak, and **leak detection** for code that never returns what it borrowed.

A pool is sized from both sides with **Little's Law**: demand is borrows per second times **hold time**, and supply is what the database runs efficiently, roughly twice its cores plus its disks, divided among all instances (including during deploys and autoscaling). Queues belong in the application, where waiting is cheap, not in the database. When instances or short-lived clients multiply connections, a **server-side pooler** such as PgBouncer shares a few backends among many clients; **transaction pooling** shares them between transactions, which breaks session-level features such as `SET`, session advisory locks and (without support) prepared statements.

A **prepared statement** sends SQL once as a template and its values separately as bound **parameters**. The database reuses the parse (and possibly the plan, with the generic-plan risk of Chapter 16), and, more importantly, a bound value can never change the statement's structure. That is why binding stops **SQL injection**, the flaw in which input becomes SQL syntax. Binding covers values only: **ORDER BY** columns, directions and table names need **allowlists**; **LIKE** patterns need their wildcards escaped; and **second-order injection**, where stored data is later concatenated into SQL, is prevented only by binding everywhere, whatever the data's source.

An **ORM** maps classes to tables and generates SQL, bridging an **impedance mismatch** of identity, associations, navigation and inheritance. Inside it, the **persistence context** keeps an **identity map** (one object per row), records work as a **unit of work**, discovers changes by **dirty checking** against snapshots, and sends SQL at **flush** time, in a fixed order, often writing every column. **Lazy loading** defers associations behind run-time **proxies**, which fail outside the context and, inside loops, produce the **N+1 problem**: one list query plus one query per item. It shows up as repeated statements per request and enormous call counts in `pg_stat_statements`, and is fixed by **joins** for to-one associations, **batch fetching** for collections, and **DTO projections** for read-only views.

**Transaction boundaries** belong on use-case methods, contain only database work, and are found through the execution context, so other threads do not share them. **Propagation** decides whether nested calls join (`REQUIRED`), suspend and start anew on another connection (`REQUIRES_NEW`, a pool-deadlock risk), or use savepoints (`NESTED`); **read-only** transactions save work and enable replica routing; proxy-based declarations are skipped on **self-invocation**. A **remote call inside a transaction** multiplies hold time, exhausts pools, spreads lock waits and cannot be rolled back; the fix is short transactions around the call, explicit intermediate states and idempotency keys. ORM read-modify-write cycles cause **lost updates**, fixed by atomic SQL for counters, **optimistic versions** carried through HTTP for human edits, pessimistic locks for hot short decisions and Serializable with retry for multi-row invariants, with retries outside the transaction.

Large data needs **batching** or `COPY` for writes, **chunked set-based** updates for bulk changes, and **streaming** or keyset batches for reads, each with a clear rule on how long it holds a connection and a snapshot. **Versioned migrations** keep schemas in code; **expand/contract** keeps every running code version compatible during rolling deploys; and **lock timeouts**, concurrent index builds, `NOT VALID` constraints and backfills keep DDL from queuing the whole site behind one long query. **Read/write splitting** adds read capacity at the price of **replica lag** and the **read-your-writes** anomaly, handled by routing a user's own and decision-making reads to the primary. Every one of these failures has a two-sided signature: pool metrics in the application, and session states and statement statistics in the database.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Driver / wire protocol | Correct framing and decoding; errors with SQLSTATE; server identity with verified TLS | The outcome of a command when the connection drops; bounded memory for big results |
| New connection | A clean session | Low cost (milliseconds, a backend process) |
| Connection pool | At most N connections per instance; a connection or an error within the timeout; reuse | A live connection; a clean session without reset; capacity at the database; fairness |
| Pool sizing by Little's Law | Concurrency matched to demand and to the database's budget | Low latency when hold times or queries grow |
| PgBouncer transaction mode | Bounded backends; each transaction on one backend | Session state across transactions; cheaper long transactions |
| Prepared statement | Parse once per session; values never parsed as SQL | Good plans for all values; survival across connections or `DISCARD ALL` |
| Parameter binding | No injection through values | Safety of identifiers, keywords, LIKE patterns or string-built fragments |
| ORM mapping | SQL matching the metadata; bound parameters | Efficient SQL or a predictable query count |
| Identity map | One object per row per context | Freshness against other transactions' commits |
| Unit of work / dirty checking | All managed changes written at flush, in a consistent order | Errors at the line that caused them; only changed columns written; speed for huge contexts |
| Lazy loading | Associations loaded only if touched, once per context | A bounded number of queries; access after the context closes |
| N+1 fixes (join, batch fetch, projection) | Query count independent of N | Fast remaining queries; correct pagination with collection fetch joins |
| Transaction boundary on the use case | All-or-nothing database writes; connection returned on every path | Atomicity of side effects outside the database; interception on self-invocation |
| `REQUIRES_NEW` | An independent commit | Freedom from pool deadlock (needs a second connection) |
| Read-only transaction | No writes; less work for database and ORM | Fresh data if routed to a replica |
| Optimistic version | Detection of conflicting writes, including across the edit gap | Success without retries; protection if some writer skips the version |
| Atomic `UPDATE` | No lost update on that row without retries | Invariants spanning several rows |
| Batching / `COPY` | Far fewer round trips | Atomicity without a transaction |
| Streaming / cursor | Bounded client memory | Short transactions or snapshots |
| Versioned migrations | Each script once, in order, unedited | Lock-free DDL; reversibility; atomic DDL in MySQL |
| Expand/contract | Every running code version finds its columns | A single-deploy change |
| Read/write splitting | More read capacity | Read-your-writes; bounded lag; monotonic reads |

---

## Interview Drill

**Q1. What does a database driver do? What happens when an application connects to PostgreSQL?**
A strong answer must mention:
- A driver implements one database's wire protocol behind a standard interface (JDBC, DB-API, `database/sql`): connect, authenticate, send SQL and parameters, decode results and errors.
- PostgreSQL connection: TCP handshake, SSLRequest and TLS handshake, StartupMessage, SCRAM authentication exchange, ParameterStatus, BackendKeyData, ReadyForQuery.
- The postmaster forks a dedicated backend process per connection.
- Queries run via the simple protocol (one `Query` message) or the extended protocol (Parse, Bind, Execute, Sync).
*Common weak answer:* "the driver sends SQL to the database." It misses authentication, the backend process and the protocol steps that explain cost and prepared statements.

**Q2. Why is opening a database connection expensive, and why does PostgreSQL in particular need pooling?**
A strong answer must mention:
- Several round trips (TCP, TLS negotiation and handshake, authentication), cryptographic and password-hashing CPU on both sides.
- In PostgreSQL, a process fork and backend initialisation per connection, with cold catalog caches and megabytes of memory.
- Thousands of connections mean thousands of processes, contention in shared structures, and reconnect storms after failures.
- MySQL threads are cheaper, but the network and authentication costs remain.
*Common weak answer:* "because of the TCP handshake." That is the smallest part of the cost.

**Q3. How does a connection pool work?**
A strong answer must mention:
- A bounded set of open connections; borrow, use, return; callers wait in a queue up to an acquisition timeout when none is free.
- New connections opened outside the pool's lock; validation and keepalive for idle connections; a maximum lifetime below network idle limits.
- Reset on return: roll back open transactions, restore connection properties.
- Leak detection: warn with the borrower's stack trace when a connection is held too long.
- Metrics: active, idle, pending, acquisition time, usage time, timeouts.
*Common weak answer:* "it caches connections so they can be reused," with no mention of bounds, waiting, timeouts or reset.

**Q4. How do you size a connection pool?**
A strong answer must mention:
- Demand by Little's Law: busy connections = borrows per second × hold time (the whole borrow, not just query time).
- Supply: the database's efficient concurrency, roughly 2 × cores + disks as a starting point, shared by all clients.
- Divide the budget among instances, using the maximum instance count including deploys and autoscaling; verify headroom.
- Bigger pools past the database's peak reduce throughput; queue in the application, with a short acquisition timeout.
- Confirm with a load test; virtual threads do not change the database's capacity.
*Common weak answer:* "set it equal to the number of threads" or "make it large to avoid timeouts." Both push the queue into the database.

**Q5. What is PgBouncer, and what do session, transaction and statement pooling mean?**
A strong answer must mention:
- A server-side pooler that accepts many client connections and multiplexes them over few database backends.
- Session mode: a backend per client connection; transaction mode: a backend per transaction, released when ReadyForQuery shows idle; statement mode: per statement, no multi-statement transactions.
- Transaction mode breaks session state: `SET` without `LOCAL`, session advisory locks, `LISTEN`, temporary tables, protocol-level prepared statements without pooler support.
- It does not make long transactions cheaper or add database capacity.
*Common weak answer:* "PgBouncer makes PostgreSQL faster." It bounds connections; it does not speed up queries.

**Q6. What is a prepared statement, and what does the database reuse?**
A strong answer must mention:
- A SQL template with placeholders, parsed once per session; values sent separately in a Bind step.
- PostgreSQL reuses the parse and analysis, and after several custom plans may switch to a generic plan.
- Generic plans can be bad for skewed values; prepared statements are per session (poolers, `DISCARD ALL`).
- Only values can be parameters, not identifiers, keywords or lists (use arrays).
*Common weak answer:* "a precompiled query for speed," missing per-session scope, the plan-reuse risk and the security role.

**Q7. What is SQL injection, and why do prepared statements prevent it?**
A strong answer must mention:
- Input becomes part of the SQL text and changes its structure (a quote ends a literal, `OR` adds a condition, `--` comments out the rest).
- With binding, the statement is parsed before the value arrives; the value travels as typed data and is never tokenised.
- Binding beats escaping because there is no string to escape correctly.
- Defence in depth: least-privilege database users, generic error responses, no string-built queries in ORMs.
*Common weak answer:* "prepared statements escape special characters." Binding does not escape anything; it keeps data out of the parser.

**Q8. Can you inject through ORDER BY or LIKE? What is second-order injection?**
A strong answer must mention:
- ORDER BY columns and directions cannot be bound, so they are often concatenated; fix with an allowlist mapping API names to fixed SQL fragments.
- LIKE with a bound value is not injectable, but `%` and `_` are wildcards inside the value; escape them with an `ESCAPE` clause.
- Second-order: input stored safely, later read back and concatenated into SQL by another code path.
- The fix for second-order is binding everywhere regardless of the value's source.
*Common weak answer:* "binding everything solves all injection." It does not cover identifiers, keywords or LIKE semantics.

**Q9. What is an ORM, and what is the object–relational impedance mismatch?**
A strong answer must mention:
- A library mapping classes to tables and generating SQL from mapping metadata.
- Mismatches: identity (address versus key), associations (references versus foreign keys), navigation versus set queries, inheritance, granularity.
- Data Mapper versus Active Record.
- Identifier generation and owning sides affect batching and correctness.
*Common weak answer:* "an ORM means you don't write SQL," which hides that it writes SQL whose number and timing you must control.

**Q10. How does an ORM track changes? Explain the identity map, unit of work, dirty checking and flushing.**
A strong answer must mention:
- The persistence context keeps an identity map: one object per (type, id) per context; repeated finds return the same object.
- On load it stores a snapshot; at flush it compares fields with snapshots to find dirty objects.
- The unit of work queues inserts, updates and deletes and sends them at flush in a fixed order, often updating all columns.
- Flush happens before commit, before affected queries (automatic mode) or on demand; flushing is not committing.
- Consequences: changes saved without save calls; errors appear at flush; large contexts are slow.
*Common weak answer:* "the ORM listens to setters." Most ORMs compare snapshots at flush time.

**Q11. What is lazy loading, how is it implemented, and what is a LazyInitializationException?**
A strong answer must mention:
- Deferring an association's load until first access.
- A run-time generated proxy subclass (or collection wrapper) holding the id and a reference to the context; first access runs a SELECT.
- Outside a closed context there is no connection, so access throws.
- Fixes: fetch what the use case needs inside the transaction and return DTOs; not open-session-in-view or eager mappings.
*Common weak answer:* "make it eager." That loads the association everywhere and often creates N+1 or huge joins.

**Q12. What is the N+1 problem? How do you detect and fix it?**
A strong answer must mention:
- One query for N items, then one per item for related data, usually through lazy loading in a loop.
- Cost is round trips and connection hold time, not database CPU; it multiplies pool demand by Little's Law.
- Detection: repeated identical statements per request in logs or traces, per-request query counts, `pg_stat_statements` call ratios, query-count assertions in tests.
- Fixes: join fetch for to-one, batch fetching (`IN` / `= ANY`) for collections, DTO projections for read-only views.
- Collection fetch joins break pagination and multiply rows.
*Common weak answer:* "add an index." Each query is already a fast primary-key lookup; the problem is their number.

**Q13. Where should transaction boundaries go?**
A strong answer must mention:
- On the use-case (application service) method, so one business operation is one transaction.
- Not on controllers (too long) or individual repositories (not atomic across them).
- Only database work inside; no remote calls, emails or publishing.
- The transaction is bound to the execution context; work on other threads does not share it.
- Verify with the database's view of open transactions.
*Common weak answer:* "put @Transactional on everything." Boundaries must match business operations, and annotations can be bypassed.

**Q14. Why should you never call a remote service inside a database transaction?**
A strong answer must mention:
- The connection and locks are held for the call's duration; by Little's Law, hold time multiplies required connections, so a slow dependency exhausts the pool.
- Row locks held during the call make other transactions wait, spreading the slowdown.
- A rollback cannot undo the remote effect; a timeout leaves it unknown; a failed commit after a successful call creates inconsistency.
- Fix: short transactions before and after, explicit intermediate states, idempotency keys, outbox for messages.
*Common weak answer:* "it is slow." It is a capacity and correctness problem, not only latency.

**Q15. What do propagation settings do? Why can `REQUIRES_NEW` deadlock a pool, and why does `@Transactional` sometimes do nothing?**
A strong answer must mention:
- `REQUIRED` joins or starts; `REQUIRES_NEW` suspends and starts an independent transaction on a second connection; `NESTED` uses a savepoint.
- A joined inner failure marks the shared transaction rollback-only even if caught.
- `REQUIRES_NEW` on a hot path: N requests each holding one connection and waiting for another exhaust a pool of N.
- Proxy-based transactions are skipped on self-invocation, private methods and objects created with `new` (Chapter 12, Section 12.5).
*Common weak answer:* "nested transactions commit independently," which is true only of `REQUIRES_NEW`, and ignores its connection cost.

**Q16. How do you prevent lost updates when using an ORM?**
A strong answer must mention:
- ORM read-modify-write writes absolute values computed from stale reads; full-row updates can lose changes to other fields.
- Atomic `UPDATE … SET x = x - $1 WHERE … AND x >= $1` for counters, bypassing (and then refreshing) the persistence context.
- Optimistic version column: `WHERE id = $ AND version = $`, 0 rows means conflict; carry the version through `ETag`/`If-Match` for human edits.
- Pessimistic `FOR UPDATE` for short hot decisions; Serializable plus retry for multi-row invariants.
- Retries go outside the transaction.
*Common weak answer:* "use a transaction." At default isolation levels, a transaction alone does not prevent it.

**Q17. How do you insert a million rows efficiently, and how do you read a table that does not fit in memory?**
A strong answer must mention:
- Avoid per-row round trips and per-row commits: batched prepared statements, multi-row `INSERT`, or `COPY`/`LOAD DATA`, in chunked transactions.
- ORM batching needs a batch size, sequence ids (identity ids defeat it) and flush+clear every batch.
- Reading: driver fetch size or cursors (holding a transaction and snapshot) versus keyset batches (short statements, resumable).
- Watch long transactions, replica lag and memory.
*Common weak answer:* "use a loop with a transaction." It still pays a round trip per row.

**Q18. How do you rename a column with zero downtime, and why can a one-line ALTER TABLE take a site down?**
A strong answer must mention:
- Old and new code run together during rolling deploys, so the schema must suit both.
- Expand (add new column), dual-write, backfill in chunks, switch reads, stop old writes, contract (drop old column) after rollback is no longer possible.
- ALTER TABLE needs a strong lock; waiting behind a long query, it blocks every later query on the table (lock queue).
- Use `lock_timeout` and retry; avoid rewrites; `CREATE INDEX CONCURRENTLY`; `NOT VALID` then `VALIDATE`.
*Common weak answer:* "run the rename at night." Old instances still break, and the lock queue can still form.

**Q19. What is read/write splitting, and what is the read-your-writes problem?**
A strong answer must mention:
- Writes and fresh reads to the primary; staleness-tolerant reads to replicas.
- Asynchronous replication lag is unbounded; a user's read after their own write may hit a lagging replica.
- Mitigations: read from the primary for a period after a write, route by need, LSN-based waiting, synchronous replicas.
- Never base write decisions on replica reads.
*Common weak answer:* "replicas are copies, so any read can go there."

---

## Scenario Exercises

1. A service runs 12 instances with a pool of 20 connections each against a PostgreSQL server with 8 cores and `max_connections = 200`. During a rolling deploy, new instances fail to start with `FATAL: sorry, too many clients already`, and during peaks, query latency on the database doubles although traffic rose only 20%. Explain both observations with numbers and propose a configuration.
2. After moving a Python service from direct connections to PgBouncer in transaction mode, users occasionally see timestamps in the wrong time zone, and a background job that uses `pg_advisory_lock` sometimes runs twice concurrently. Explain both and fix them.
3. An admin page lists 50 tenants with their user count and the date of their last invoice. It takes 1.8 s in production and 40 ms in development. `pg_stat_statements` shows a query `select … from users where tenant_id=$1` with about 50 times as many calls as the tenant list query. Explain, and give two different fixes with their trade-offs.
4. A reporting endpoint accepts `?groupBy=region&sort=revenue desc` and builds SQL from them. A penetration test reports that the endpoint leaks data. The developer says "impossible, we use prepared statements". Explain how, and how to fix it.
5. A "transfer credits" use case calls `ledger.debit()` and `ledger.credit()`, both annotated `@Transactional(propagation = REQUIRES_NEW)` "to make each step safe". Under load, the service freezes for exactly 30 s at a time; in other incidents, a debit commits while the credit fails. Explain both problems and redesign it.
6. A team adds a `NOT NULL` column with a default to a 400 GB table on PostgreSQL 10 during business hours, and the site goes down for 25 minutes. The same migration took 50 ms in staging. Explain why, and how you would perform the change on that version and on a current version.
7. A user changes their shipping address, is redirected to the order confirmation page, and sees the old address; the order is then shipped to the old address. The service reads from replicas for all `GET` requests. Explain both failures (display and shipping) and fix them.

---

## Diagnosis Lab

**Lab 1: "Connection is not available" (pool exhaustion by slow queries)**
- *Symptom:* At 09:05 on the first working day of the month, every endpoint of `orders-api` starts returning 500s after exactly 30 s, with `HikariPool-1 - Connection is not available, request timed out after 30000ms` in the logs. Database CPU is 55% and disk reads are high. Nothing was deployed today; last week's release added an "export my orders as CSV" button for business customers.
- *Hypotheses:* (a) the database is overloaded for everyone; (b) slow queries hold the connections; (c) connections are held across non-database work (Chapter 11's Lab 1 had this cause); (d) a leak.
- *Evidence:* Pool metrics on every instance: active = 10 of 10, pending ≈ 120, usage time p99 jumped from 15 ms to 28 s at 09:03. The database's `pg_stat_activity` grouped by `application_name` (Section 18.17) shows the opposite of hypothesis (c): `orders-api` has 76 sessions `active` and almost none `idle in transaction`, and 71 of the active ones run the same statement, `SELECT … FROM orders o JOIN order_lines l … WHERE o.customer_id = $1 AND o.created_at >= $2`, each for 10–30 s, with wait events `DataFileRead`. `pg_stat_statements` shows that statement with a mean time of 21 s and a call count that jumped this morning, when business customers ran their monthly exports (each click also retried automatically by the front end after 15 s). `EXPLAIN` shows the export reads a year of orders for large accounts with no suitable index and buffers the whole result in the driver (Section 18.1).
- *Root cause:* (b): a slow query on a shared pool. A burst of exports, each holding a connection for about 20 s, took every connection in the fleet's pools (by Little's Law, even 4 exports per second × 20 s = 80 connections, the whole fleet's 8 × 10); all other endpoints then waited for connections and timed out. The front-end retry doubled the load. There was no statement timeout to bound the hold time.
- *Fix (now):* turn the export button off with its feature flag and cancel the running export queries (`pg_cancel_backend` on their PIDs); traffic recovers within seconds. *Fix (lasting):* give exports their own small pool (2 connections per instance) so they can never take the request path's connections (a bulkhead, Chapter 24); a `statement_timeout` for the request-path role (for example 2 s) and a longer one for the export role; an index on `orders (customer_id, created_at)`; stream the export with keyset batches instead of one buffered query (Section 18.14), or generate it asynchronously as a job (Chapter 13, Section 13.10); remove the front-end auto-retry for non-idempotent long operations; acquisition timeout down to 1 s.
- *Prevention:* Alert on pool pending count and usage-time p99; review new endpoints for their worst-case query time on the largest customer, not the average one; a load test with a large account before release.

**Lab 2: A list endpoint gets slower every month (N+1)**
- *Symptom:* `GET /api/accounts/{id}/invoices` p99 rose from 120 ms to 2.4 s over three months; slow mainly for large customers. Database CPU is low; nothing obvious in the slow query log (no statement over 500 ms).
- *Hypotheses:* (a) a missing index that hurts as data grows; (b) N+1 queries growing with the number of invoices; (c) a large response payload.
- *Evidence:* A trace of one slow request shows 1 query for invoices and then 1,214 short spans, alternating `select … from invoice_lines where invoice_id=?` and `select … from payments where invoice_id=?`, each 0.6–1 ms. The request returns 600 invoices (no pagination). `pg_stat_statements`: the two child queries have about 600 times as many calls as the parent. The connection usage-time histogram for this endpoint matches the request latency.
- *Root cause:* (b): the DTO mapper reads `invoice.lines` and `invoice.payments` for each invoice, triggering two lazy loads per invoice: 1 + 2N queries, growing with the customer's invoice count.
- *Fix:* Paginate (50 invoices per page, keyset by `(issued_at, id)`); load lines and payments with two batch queries (`invoice_id = ANY($1)`) for the page; for the totals the list actually shows, use a DTO projection with `sum()` and `count()` instead of loading lines at all. Query count per request drops to 2–3; p99 to 35 ms.
- *Prevention:* A test that loads an account with 100 invoices and asserts at most 4 statements; per-request query count in traces with an alert on outliers; lazy loads disallowed in list endpoints (`raiseload`-style settings or code review).

**Lab 3: Locks that never go away (leaked transaction)**
- *Symptom:* Since a release on Monday, the pool of `billing-api` loses a few connections per day; by Thursday, requests to update certain customer records hang until lock timeouts (`ERROR: canceling statement due to lock timeout`). Restarting the service fixes everything for a day.
- *Hypotheses:* (a) a connection leak; (b) a transaction left open on a pooled connection; (c) a long-running batch job.
- *Evidence:* `pg_stat_activity` shows 6 `billing-api` sessions in `idle in transaction` with `xact_start` between 1 and 3 days ago; their last statement is `UPDATE customers SET credit_hold = true WHERE id = $1`. `pg_blocking_pids` shows the hanging updates blocked by these sessions. The pool's leak detector reports nothing (the connections were returned). The release added a code path that sets autocommit off on a borrowed connection for a "manual transaction" and returns early on a validation error without commit or rollback; the pool in use does not roll back on return in this configuration.
- *Root cause:* (b): an exception path that returned the connection mid-transaction. The connection went back to the pool **with an open transaction holding row locks**. The next borrowers of that connection ran their statements inside the stale transaction (and their changes were never committed either), and other connections queued on the locks.
- *Fix (now):* terminate the idle-in-transaction backends (`pg_terminate_backend`), which rolls them back; check for lost writes from requests that ran inside the leaked transactions. *Fix (lasting):* replace manual begin/commit with the framework's transaction template (Section 18.12); configure the pool to roll back uncommitted work on return; set `idle_in_transaction_session_timeout` (for example 60 s) for the application role.
- *Prevention:* Alert on transaction age; a lint rule against manual `setAutoCommit(false)` outside the transaction helper.

**Lab 4: The deploy that froze the orders table (migration lock outage)**
- *Symptom:* During a routine deploy at 14:05, every request that touches orders hangs; pools exhaust across three services; at 14:11 everything recovers by itself. The migration in the deploy was `ALTER TABLE orders ADD COLUMN gift_message text;`, which took 4 ms in staging.
- *Hypotheses:* (a) a table rewrite; (b) the DDL waited for a lock and blocked others behind it; (c) unrelated database overload.
- *Evidence:* The database log (`log_lock_waits = on`) shows the migration session `still waiting for AccessExclusiveLock on relation orders after 1000.123 ms`, and dozens of sessions waiting for `AccessShareLock` and `RowExclusiveLock` on the same table. `pg_blocking_pids` (captured by an incident script) shows the queue rooted at an analytics session that had been running a 6-minute `SELECT` over `orders` since 14:04:40. The `ALTER` itself completed in 3 ms at 14:11:02, right after the analytics query finished.
- *Root cause:* (b): the lock queue. A metadata-only DDL statement needed an `ACCESS EXCLUSIVE` lock, waited behind a long `ACCESS SHARE` holder, and every later query on `orders` queued behind the waiting DDL (Chapter 17, Section 17.5).
- *Fix:* Every migration begins with `SET lock_timeout = '2s'`, and the migration runner retries a few times with backoff; long analytics queries move to a replica (Section 18.16) with a `statement_timeout` for the analytics role.
- *Prevention:* A migration linter that rejects DDL without a lock timeout and flags rewrites, non-concurrent index builds and constraint validation on large tables; run migrations as a separate deploy step, with a pre-check for long-running queries on the target tables.

---

## Solution Sketches

1. Connections: 12 × 20 = 240 configured, over `max_connections = 200`; during a rolling deploy, old and new instances overlap (up to 24 × 20 = 480), so new instances cannot connect. The latency doubling is the throughput curve: an 8-core database runs best with roughly 2 × 8 + disks ≈ 17–20 active queries, and peaks push far more concurrent queries in, adding contention. Configuration: budget about 20 connections for this service (leaving room for others and admins), pools of 2 per instance (or put PgBouncer in transaction mode with a server pool of about 20 and let client pools be larger), cap the autoscaler, and keep an acquisition timeout of around 1 s. Measure hold time to confirm demand (Little's Law) fits.
2. Transaction pooling moves consecutive transactions to different backends. A session-level `SET timezone` made on one backend applies to whichever client later gets that backend, and not to the client's next transaction; fix with `SET LOCAL` in each transaction or setting the time zone per role or database (or in the application, using `timestamptz` and formatting in code). The advisory lock is session-scoped: taken on one backend, it stays held there after the transaction, while the job's next statements run elsewhere, and unlocking may happen on a different backend; meanwhile another instance can acquire it. Fix with transaction-scoped `pg_advisory_xact_lock` inside one transaction, or a direct (non-pooled) connection for the job.
3. N+1: the page runs the tenant query, then one `users` count query per tenant and probably one invoice query per tenant, each a round trip; development has a local database and few tenants. Fix A: one projection query with joins and aggregates (`count(u.id)`, `max(i.issued_at)` grouped by tenant), the fastest and simplest for a read-only page; check that it does not fan out (aggregate users and invoices in separate subqueries or lateral joins to avoid multiplying rows). Fix B: batch the child queries (`WHERE tenant_id = ANY($1) GROUP BY tenant_id`), keeping entity-based code; three queries in total. Add a query-count test.
4. Prepared statements bind values only; `groupBy` and `sort` are identifiers and keywords, so they were concatenated into the SQL. An attacker can put a subquery or `CASE` expression into the sort or group expression and extract data by observing ordering or errors (blind injection), or worse if stacked queries are allowed. Fix: allowlists mapping each permitted API value to a fixed column expression and direction; reject everything else with 400; keep binding all values; run the report as a least-privilege, read-only database user.
5. Each `REQUIRES_NEW` suspends the caller's transaction and borrows another connection; if the caller is itself transactional, every transfer holds two or three connections, and under load all pool connections are held by requests waiting for another: a pool deadlock until the 30 s acquisition timeout. And because debit and credit commit independently, a failure in the credit leaves a committed debit. Redesign: one `REQUIRED` transaction for the transfer containing both ledger writes (a conditional debit `UPDATE … WHERE balance >= $1` and the credit), with consistent lock ordering by account id to avoid deadlocks (Chapter 17, Section 17.10), retried as a whole on deadlock or serialisation failures; if an audit record must survive rollback, write it after the outcome or through a separate, non-nested path.
6. Before PostgreSQL 11, `ADD COLUMN … DEFAULT …` rewrote the whole table under an `ACCESS EXCLUSIVE` lock; 400 GB took 25 minutes, blocking all access (staging had a small table). On PostgreSQL 10: add the column without a default (metadata-only), set the default for new rows separately (`ALTER COLUMN … SET DEFAULT`, metadata-only), backfill existing rows in chunks, then add `CHECK (col IS NOT NULL) NOT VALID`, `VALIDATE CONSTRAINT`, and keep the check (on version 10 `SET NOT NULL` would still scan; on 12+ it can use the validated check). On a current version, `ADD COLUMN … NOT NULL DEFAULT 'x'` with a constant default is metadata-only; still run it with `lock_timeout` to avoid the lock queue.
7. Display: the confirmation page `GET` went to a lagging replica: read-your-writes violation. Shipping: if the fulfilment step read the order's address from a replica (or from a cache filled from one) to decide where to ship, it made a write-side decision on stale data. Fix: route a user's reads to the primary for a window after their writes (or use LSN-based waiting) for their own data; read decision-making data (the address used to create a shipment) from the primary inside the transaction that creates the shipment; better still, copy the address into the shipment at the moment the order is confirmed, so later changes cannot race with fulfilment.

---

**END OF CHAPTER 18**
