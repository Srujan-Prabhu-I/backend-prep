# Chapter 15: Storage Engines and Durability

## Where We Are

Chapter 14 treated the database as a perfect logical machine: tables of rows, constraints that always hold, queries that return the right answer. This chapter opens the machine. When you run `UPDATE orders SET status = 'PAID' WHERE id = 42` and the database answers `COMMIT`, something in a server process has to find the row on disk, change it, and make a promise: even if the power fails one millisecond from now, the change will survive. That promise, **durability**, is the subject of this chapter.

The pieces come from earlier chapters. Chapter 2 described the memory hierarchy and storage devices: RAM is fast and volatile, SSDs and HDDs are slow and persistent, and random access costs more than sequential access. Chapter 3 described processes, threads and page replacement. Chapter 4 described files, the operating system's page cache, and `fsync`, the only way to know data has reached stable storage. A storage engine is a careful arrangement of exactly these pieces.

The chapter follows a write from the client to the disk: the server's process model, how rows are laid out in pages, the buffer pool that caches pages in memory, the write-ahead log that makes commits durable, and crash recovery with checkpoints. Then it compares the two great designs for storage (update-in-place B-trees and log-structured merge trees), row and column layouts, in-memory databases, and finally backups, the last line of defence when everything else fails.

**After this chapter you can:**
- describe a database server's architecture, and explain PostgreSQL's process-per-connection model versus MySQL's thread-per-connection model;
- draw a slotted page and trace an insert, update and delete through it, in PostgreSQL and InnoDB;
- explain the buffer pool, its eviction policies, dirty pages and pin counts, and its interaction with the OS page cache;
- trace the write-ahead log path of a commit, and explain the WAL rule, LSNs, group commit and the cost of `fsync`;
- explain crash recovery (redo, and undo in InnoDB), the ARIES ideas, and how checkpoints trade normal-time I/O against recovery time;
- compare B-tree and LSM-tree storage by write, read and space amplification, and explain bloom filters and compaction;
- choose between row and column storage for a workload, and explain columnar compression and vectorised execution;
- reason about the durability options of an in-memory database such as Redis;
- design backups with logical and physical methods, point-in-time recovery, and explicit RPO and RTO;
- diagnose checkpoint spikes, full disks, runaway WAL growth and slow `fsync`.

**Interview questions this chapter prepares you for:**
1. How does a database store rows on disk?
2. Why does PostgreSQL use a process per connection, and what does that cost?
3. What is the buffer pool? How does it decide what to evict?
4. What is write-ahead logging? Why is it faster than writing the data pages at commit?
5. What happens, step by step, when the database crashes and restarts?
6. What does a checkpoint do, and what happens if checkpoints are too frequent or too rare?
7. How does PostgreSQL roll back a transaction, and how does InnoDB?
8. B-tree versus LSM-tree storage: how do they differ, and when would you choose each?
9. What are write, read and space amplification? What is a bloom filter for?
10. Row stores versus column stores: why is a column store faster for analytics?
11. How can an in-memory database be durable?
12. What is the difference between logical and physical backups? How does point-in-time recovery work? What are RPO and RTO?
13. Why is a replica not a backup?

**Scope.** Here: server architecture, page layout, the buffer pool, write-ahead logging, crash recovery and checkpoints, B-trees versus LSM trees as storage designs, row versus column storage, in-memory durability, backups, and storage-level diagnosis, plus a placement-round section on classic recovery theory (Section 15.11). Recalled, not re-taught: storage devices (Chapter 2, Section 2.4), process and thread models (Chapter 3, Sections 3.7–3.9), page replacement (Chapter 3, Section 3.16), the page cache and `fsync` (Chapter 4, Sections 4.3–4.4), RAID (Chapter 4, Section 4.13). Elsewhere: the B+tree as an index structure and how queries use it is Chapter 16; MVCC, isolation and locking are Chapter 17 (this chapter shows only what versions look like on a page); connection pools are Chapter 18; Redis as a cache is Chapter 21; replication is Chapter 22.

---

## 15.1 Inside a Database Server: Processes, Threads, Memory, Workers

### Definitions

A **database server** is a long-running program that owns the database's files, accepts client connections over a network socket, executes their SQL, and writes changes durably. It is built from layers:

- the **connection layer** accepts connections, authenticates clients and gives each one an execution context;
- the **query layer** parses SQL, plans it (the optimiser, Chapter 16) and executes the plan;
- the **storage engine** (or storage manager) stores rows and indexes in files, caches pages in memory, logs changes, and recovers after crashes.

A **background worker** is a process or thread inside the server that does maintenance work independently of any client: writing dirty pages, flushing the log, taking checkpoints, cleaning up old row versions.

### Why it matters

Every client connection costs the server memory and scheduling, and every background task competes with clients for CPU and I/O. Knowing the architecture explains facts that otherwise look arbitrary: why PostgreSQL needs a connection pooler at a few hundred connections while MySQL copes with more, why latency spikes every few minutes, and why "the database is slow" can mean a client query, a background writer, or the disk underneath them.

### Mechanism: the two process models

**PostgreSQL uses one operating-system process per client connection.** A supervisor process (historically called the **postmaster**) listens on the port. For each new connection it calls `fork` (Chapter 3, Section 3.5), and the child **backend process** serves that one client for the life of the connection. All processes share a region of **shared memory** holding the page cache (`shared_buffers`), WAL buffers, lock tables and per-transaction state.

```text
 PostgreSQL                                              MySQL (InnoDB)
 ┌─────────────────────────────────────────────┐         ┌──────────────────────────────────────────────┐
 │ postmaster (listens on :5432, forks)        │         │ mysqld: ONE process                          │
 │   ├─ backend  (client A)  private memory    │         │   ├─ connection thread (client A)            │
 │   ├─ backend  (client B)  private memory    │         │   ├─ connection thread (client B)            │
 │   ├─ backend  (client C)  private memory    │         │   ├─ connection thread (client C)            │
 │   ├─ checkpointer                           │         │   ├─ InnoDB page cleaner threads             │
 │   ├─ background writer                      │         │   ├─ InnoDB log writer / flusher threads     │
 │   ├─ WAL writer                             │         │   ├─ InnoDB purge threads                    │
 │   ├─ autovacuum launcher (+ workers)        │         │   ├─ InnoDB I/O threads                      │
 │   ├─ archiver, WAL senders (replication)    │         │   └─ ...                                     │
 │   └─ ...                                    │         │                                              │
 │  ┌──────────── shared memory ─────────────┐ │         │  shared by all threads, in one address space: │
 │  │ shared_buffers │ WAL buffers │ locks   │ │         │  buffer pool │ log buffer │ locks │ caches    │
 │  └────────────────────────────────────────┘ │         │                                              │
 └─────────────────────────────────────────────┘         └──────────────────────────────────────────────┘
```

**MySQL uses one thread per client connection inside a single `mysqld` process** (Chapter 3, Section 3.9). Threads share the process's address space directly; a thread cache reuses threads across connections, and some editions and forks offer a thread pool that serves many connections with fewer threads.

The process list on a PostgreSQL server shows the model directly (with the default setting that updates process titles):

```bash
$ ps -o pid,rss,cmd -u postgres
  PID   RSS CMD
 1021 31840 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main
 1023 10212 postgres: checkpointer
 1024  8840 postgres: background writer
 1026 12004 postgres: walwriter
 1027  9120 postgres: autovacuum launcher
 1028  7600 postgres: logical replication launcher
 4410 18920 postgres: app shop 10.0.1.5(52344) idle
 4412 21460 postgres: app shop 10.0.1.5(52346) SELECT
 4415 19880 postgres: app shop 10.0.1.7(40112) idle in transaction
```

Each client line shows the user, database, client address and what the backend is doing. `idle in transaction` is a backend holding an open transaction while doing nothing, the signature of the problem in Chapter 11's Lab 1.

### Mechanism: what each model costs

| | PostgreSQL (process per connection) | MySQL (thread per connection) |
|---|---|---|
| Creating a connection | `fork` a process, set up memory, authenticate: on the order of milliseconds | create or reuse a thread: cheaper |
| Memory per idle connection | a few MB of private memory per backend, more once it has run queries (caches, sort memory) | less, but still per-thread buffers |
| Isolation | a crash in one backend cannot corrupt another's private memory (the postmaster still restarts all backends if one dies abnormally, because shared memory may be damaged) | a crash in one thread takes down the whole process |
| Context switching with many connections | process switches; per-process state | thread switches; shared address space |
| Practical consequence | hundreds of connections are fine; thousands usually need a pooler such as PgBouncer (Chapter 18) | thousands of connections are more practical, though still costly |

Neither model makes a database do more useful work than it has CPU and disk for. A database with 8 cores runs at most 8 queries at any instant; extra connections only wait (Chapter 11, Section 11.5). The per-connection cost decides how cheaply those waiting connections can exist.

### Mechanism: the memory regions

```text
 shared (one copy for the server)                   per connection (one copy each)
 ─────────────────────────────────                  ──────────────────────────────
 page cache: shared_buffers / InnoDB buffer pool    sort and hash memory (PostgreSQL work_mem,
   (Section 15.3), usually the largest region         per sort or hash operation, possibly several
 WAL / redo log buffer (Section 15.4)                 per query: Chapter 16)
 lock tables, transaction state (Chapter 17)        parsed and planned statements
 catalog caches                                     temporary buffers, result buffers
```

The per-connection regions explain an out-of-memory pattern: a server that is stable with 50 connections can run out of memory with 500, because each connection may allocate sort memory independently. Sizing must be done as "shared regions + connections × per-connection peak", not shared regions alone.

### Mechanism: the background workers and what they do

| Work | PostgreSQL | InnoDB | Section |
|---|---|---|---|
| Write dirty pages to disk in the background | background writer, checkpointer | page cleaner threads | 15.3, 15.5 |
| Flush the log | WAL writer (plus each committing backend) | log writer and flusher threads | 15.4 |
| Take checkpoints | checkpointer | continuous ("fuzzy") checkpointing by page cleaners | 15.5 |
| Remove old row versions | autovacuum workers (VACUUM) | purge threads (old undo records) | 15.2; Chapter 17 |
| Ship the log to replicas or archives | WAL senders, archiver | binary log dump threads | 15.9; Chapter 22 |

### Guarantees and non-guarantees

The process model guarantees each connection an independent execution context; PostgreSQL's processes add memory isolation between client sessions. Neither model guarantees that more connections mean more throughput (they do not, beyond the cores and disks available), and neither makes background work free: checkpoints, vacuum and purge consume the same CPU and I/O as client queries.

### Failure and diagnosis

- **"FATAL: sorry, too many clients already"** (PostgreSQL) or **"Too many connections"** (MySQL). The connection limit was reached. Evidence: the process list or `pg_stat_activity` shows hundreds of idle connections, usually from application pools sized per instance (Chapter 11, Section 11.5). Fix: smaller application pools, a pooler (Chapter 18), not a higher limit by default.
- **Out of memory under load.** Evidence: memory grows with active connections running sorts or hashes; the OS kills a backend (Chapter 3, Section 3.17), and PostgreSQL then restarts all backends, dropping every connection. Fix: limit connections, size per-operation memory conservatively.
- **Latency spikes with no client cause.** Background work (a checkpoint, vacuum on a large table) competing for I/O. Evidence: the spikes line up with checkpoint or vacuum log lines (Section 15.10).

### Backend connection

Treat database connections as expensive, especially in PostgreSQL: keep them few, pooled and short-lived in transactions. Remember that the server is also doing its own work in the background, and that its memory use scales with connections, not only with data.

---

## 15.2 Pages and Heap Files

### Definitions

A **page** (or block) is the fixed-size unit in which a database reads and writes its files: 8 KB by default in PostgreSQL, 16 KB by default in InnoDB. Every table and index is a sequence of pages, and the database never reads or writes less than a page.

A **heap file** is a table stored as an unordered collection of pages, with each row placed wherever there is room. PostgreSQL stores tables as heap files. InnoDB instead stores each table as a **clustered index**: a B+tree ordered by the primary key whose leaf pages hold the rows themselves (the B+tree is taught in Chapter 16).

A **slotted page** is the standard page layout for variable-length rows: a header, an array of **slots** (line pointers) at the start of the page, row data packed from the end, and free space between them.

A **tuple** is one stored version of a row. A **row identifier** locates a tuple physically: in PostgreSQL, a **TID** (tuple identifier, shown as the system column `ctid`) is a pair (page number, slot number), such as `(0,3)`.

**TOAST** (The Oversized-Attribute Storage Technique) is how PostgreSQL stores values too large to fit comfortably in a page: compressed, and if still large, cut into chunks and stored in a separate TOAST table, with a pointer left in the row. InnoDB has the equivalent **overflow pages**.

### Why it exists

Disks and SSDs transfer data in blocks, and operating systems cache files in pages (Chapter 4, Section 4.3). A database that reads one 100-byte row still reads at least one block. Grouping many rows into a fixed-size page makes every read and write a predictable, cacheable unit, gives each row a stable address, and lets the database manage free space inside the page. Rows have variable length (text columns), so the layout must allow rows of different sizes to be added, grown, moved and removed without rewriting the page from scratch.

### Mechanism: the slotted page

```text
 PostgreSQL heap page, 8192 bytes
 offset 0
 ┌───────────────────────────────────────────────────────────────────────┐
 │ page header (24 bytes): page LSN (Section 15.4), checksum, flags,     │
 │   pd_lower (end of slot array), pd_upper (start of tuple data)        │
 ├───────────────────────────────────────────────────────────────────────┤
 │ slot array (line pointers, 4 bytes each): slot 1 │ slot 2 │ slot 3 ──┐│  grows downward →
 │   each slot = offset of its tuple in this page, length, status        ││
 ├───────────────────────────────────────────────── pd_lower ──────────┼┤
 │                                                                       ││
 │                         FREE SPACE                                    ││
 │                                                                       ││
 ├───────────────────────────────────────────────── pd_upper ──────────┼┤
 │ tuple 3 │ tuple 2 │ tuple 1                                    ◄─────┘│  ← grows upward
 │   each tuple = tuple header (~23 bytes: creating and deleting         │
 │   transaction IDs, flags, ctid of a newer version) + column values    │
 └───────────────────────────────────────────────────────────────────────┘
 offset 8191
```

The slot array is the key idea. A row's address is its **slot number**, not its byte offset. When the database rearranges tuples inside the page (to close gaps left by deleted tuples), it updates the offsets in the slots, and every index entry or TID pointing at `(page, slot)` remains valid.

With a header of 24 bytes and tuples of about 64 bytes plus a 4-byte slot each, one 8 KB page holds about (8192 − 24) / 68 ≈ 120 rows.

### Concrete: insert, update and delete on one page (PostgreSQL)

```sql
CREATE TABLE orders (id bigint, status text, total_cents integer);
INSERT INTO orders VALUES (1, 'PLACED', 3000), (2, 'PLACED', 2500), (3, 'PLACED', 9900);
SELECT ctid, * FROM orders;
```

```text
 ctid  │ id │ status │ total_cents
 (0,1) │ 1  │ PLACED │ 3000
 (0,2) │ 2  │ PLACED │ 2500
 (0,3) │ 3  │ PLACED │ 9900
```

**Insert.** Each new tuple is written at the top of the free space (`pd_upper` moves down), and a new slot is appended to the slot array (`pd_lower` moves up). Three inserts produced slots 1–3 on page 0.

**Update.** PostgreSQL never overwrites a tuple in place. It writes a **new version** of the row and marks the old version as superseded, because other transactions may still need to see the old one (multi-version concurrency control, Chapter 17):

```sql
UPDATE orders SET status = 'PAID' WHERE id = 2;
SELECT ctid, * FROM orders;
```

```text
 ctid  │ id │ status │ total_cents
 (0,1) │ 1  │ PLACED │ 3000
 (0,3) │ 3  │ PLACED │ 9900
 (0,4) │ 2  │ PAID   │ 2500          ← the row now lives in slot 4
```

```text
 page 0 after the UPDATE
 slots:  [1]→tuple1   [2]→tuple2 (old version)   [3]→tuple3   [4]→tuple2' (new version)
 tuple2 header: deleted by transaction 731 (xmax = 731); "newer version at (0,4)"
 tuple2' header: created by transaction 731 (xmin = 731)
```

The old version, now **dead** once no transaction can see it, still occupies space. If the update changes no indexed column and the new version fits on the same page, PostgreSQL makes it a **heap-only tuple (HOT)** update: indexes keep pointing at slot 2, and the chain from slot 2 leads to slot 4, so no index needs a new entry. Otherwise every index on the table gets a new entry pointing at `(0,4)`.

**Delete.** `DELETE FROM orders WHERE id = 1` does not remove tuple 1 either: it marks it deleted by the deleting transaction. The space is reclaimed later by **VACUUM**, which removes dead tuples that no running transaction can see, marks their slots reusable, and records the free space so that future inserts can use it. Chapter 17 explains how transactions decide which versions they see, and why a long-running transaction prevents VACUUM from reclaiming anything.

```text
 page 0 after DELETE id = 1 and a later VACUUM
 slots:  [1] unused   [2] redirect → 4 (HOT chain head)   [3]→tuple3   [4]→tuple2'
 free space now includes the bytes of tuple1 and the old tuple2; tuples were compacted inside the page
```

### Mechanism: the same operations in InnoDB

InnoDB stores rows in the leaf pages of the clustered primary-key index, in key order, and updates them **in place**:

- **Update**: InnoDB first writes the old column values to an **undo log** record (in a separate undo tablespace), then overwrites the row in its page, setting hidden fields on the row: the ID of the transaction that changed it and a **roll pointer** to the undo record. Readers that need the old version follow the roll pointer and rebuild it from undo (Chapter 17). If the new row no longer fits on the page, the page splits (Chapter 16).
- **Delete**: the row is **delete-marked** in place; **purge** threads remove it physically once no transaction needs it, and also discard undo records that are no longer needed.
- **Row address**: secondary indexes do not store a physical address; they store the row's **primary-key value**, so a lookup through a secondary index then searches the clustered index (Chapter 16 explains the consequences).

| | PostgreSQL heap | InnoDB clustered index |
|---|---|---|
| Row placement | wherever there is space | in primary-key order inside a B+tree |
| Update | new tuple version; old version left for VACUUM | in place; old values copied to undo |
| Old versions live in | the table itself (dead tuples) | the undo log |
| Cleanup | VACUUM (autovacuum) | purge threads |
| Index entries point to | TIDs `(page, slot)` | primary-key values |

### Mechanism: values that do not fit (TOAST and overflow)

A page must hold at least a few rows, so very large values cannot live inline. In PostgreSQL, when a row is larger than roughly 2 KB, the large variable-length values are first **compressed**; if the row is still too large, they are moved **out of line** into the table's TOAST table, split into chunks of roughly 2 KB, and the row keeps a small pointer. Reading the column fetches and reassembles the chunks; a query that does not select that column never touches them.

InnoDB does the same with **overflow pages**: with its default row format, long `TEXT`, `BLOB` and `VARCHAR` values that do not fit are stored entirely off-page, with a 20-byte pointer left in the row.

The consequence for application design: `SELECT *` on a table with large JSON or text columns reads the TOAST or overflow data for every row; selecting only the needed columns can be dramatically cheaper.

### Guarantees and non-guarantees

The page layout guarantees stable row addresses within the page despite movement, and fixed-size units for caching and I/O. It does not guarantee that updates are cheap: in PostgreSQL every update writes a whole new tuple (and possibly index entries), and dead tuples consume space until VACUUM runs; in InnoDB updates generate undo that must later be purged.

### Failure and diagnosis

- **Table bloat (PostgreSQL).** A table of 1 GB of live data occupies 10 GB on disk. Cause: dead tuples accumulate faster than autovacuum removes them, often because a long-running transaction prevents cleanup. Evidence: `pg_stat_user_tables.n_dead_tup` high and growing; old `xact_start` in `pg_stat_activity`. Chapter 17 covers the mechanism.
- **History list growth (InnoDB).** Undo records accumulate for the same reason. Evidence: "History list length" rising in `SHOW ENGINE INNODB STATUS`.
- **Slow wide-row queries.** `SELECT *` over large text or JSON columns. Evidence: high I/O on the TOAST table or overflow pages. Fix: select only needed columns; move rarely needed large data to a separate table.

### Backend connection

Every update in PostgreSQL is an insert plus a later cleanup, and every update in InnoDB is an in-place write plus undo. Heavy-update tables (counters, status fields, sessions) therefore need vacuum or purge to keep up, and long transactions are expensive for everyone. Keep rows narrow where possible, and select only the columns you need.

---

## 15.3 The Buffer Pool

### Definitions

The **buffer pool** (PostgreSQL calls it **shared buffers**) is the region of the database server's memory that caches pages of tables and indexes. Every read and write of a page goes through it.

- A **frame** is one page-sized slot in the pool.
- The **page table** (buffer mapping table) is a hash table that maps a page identifier (file, page number) to the frame holding it.
- A page in a frame is **dirty** if it has been modified in memory but not yet written back to disk.
- A page is **pinned** while some process or thread is using it; its **pin count** says by how many. A pinned page cannot be evicted.
- **Eviction** chooses a frame to reuse when a page not in the pool is needed and no frame is free.

### Why it exists

Reading a page from an SSD takes tens to hundreds of microseconds; from memory, well under a microsecond (Chapter 2, Section 2.2). Most workloads touch a small hot set of pages again and again: the top levels of every index, recent orders, popular products. Caching those pages in memory turns most page accesses into memory accesses. The database manages its own cache, rather than relying only on the operating system's page cache (Chapter 4, Section 4.3), because it knows things the OS does not: which pages are part of a large scan that will not be reread, which dirty pages must not be written before the log (Section 15.4), and which pages are pinned by running queries.

### Mechanism: the structures

```text
                          PAGE TABLE (hash)
                          (orders, page 0)   → frame 2
 request: "page 7 of      (orders, page 7)   → frame 0
  orders_pkey"            (customers, 3)     → frame 3
       │                  (orders_pkey, 7)   → frame 1
       ▼
 ┌─────────────┬─────────────┬─────────────┬─────────────┐
 │ frame 0     │ frame 1     │ frame 2     │ frame 3     │      BUFFER POOL
 │ orders p7   │ pkey p7     │ orders p0   │ customers p3│      (each frame = one 8 KB page)
 ├─────────────┼─────────────┼─────────────┼─────────────┤
 │ dirty: no   │ dirty: no   │ dirty: YES  │ dirty: no   │      frame descriptors
 │ pin: 0      │ pin: 2      │ pin: 0      │ pin: 0      │
 │ usage: 1    │ usage: 5    │ usage: 3    │ usage: 0    │
 │ page LSN: - │ page LSN: - │ LSN 0/3A10  │ page LSN: - │
 └─────────────┴─────────────┴─────────────┴─────────────┘
```

A page access, step by step:

```text
 1. hash (file, page number) → look up the page table
 2. hit:  pin the frame (pin count + 1), mark it recently used, return a pointer to the page in memory
    miss: choose a victim frame (eviction, below)
          if the victim is dirty: make sure the WAL is flushed up to its page LSN (Section 15.4),
                                  then write the page to the data file
          read the requested page from disk into the frame; update the page table; pin it
 3. the caller reads or modifies the page in memory (modification: set dirty, update page LSN)
 4. the caller unpins it when done (pin count − 1)
```

### Mechanism: eviction policies

Recall from Chapter 3 (Section 3.16) the LRU and Clock policies for page replacement. Databases use variants of them, adapted to two database-specific problems: **pinned** pages cannot be evicted, and **large scans** would otherwise flush the whole cache.

**PostgreSQL: clock sweep with usage counts.** Each frame has a usage count, incremented (up to a small maximum, 5) on each access. To find a victim, a "clock hand" walks around the frames: a pinned frame is skipped; a frame with usage count above zero has its count decremented and is skipped; the first unpinned frame with usage count zero is chosen.

**Eviction walkthrough.** Using the four frames above, a request for a page not in the pool arrives. The clock hand is at frame 0.

```text
 hand at  frame state (pin, usage)   action
 frame 0  pin 0, usage 1             usage → 0, move on
 frame 1  pin 2, usage 5             pinned: skip
 frame 2  pin 0, usage 3             usage → 2, move on
 frame 3  pin 0, usage 0             VICTIM: unpinned, usage 0, not dirty → reuse immediately
```

Had frame 3 been dirty, the database would first have written it to disk (after flushing the WAL up to its page LSN). A frequently used page survives several sweeps, because each access raises its count again; a page used once drops to zero after one pass.

**InnoDB: LRU with a midpoint.** InnoDB keeps an LRU list split into a "young" part (about five-eighths, by default) and an "old" part. A newly read page enters at the **midpoint**, the head of the old part, not at the head of the list. It moves to the young part only if it is accessed again after a short delay. A full-table scan therefore cycles its pages through the old part without displacing the hot pages in the young part.

**Scan resistance in PostgreSQL** comes from **ring buffers**: a large sequential scan, a `VACUUM` or a bulk load uses a small private ring of frames that it reuses, instead of spreading its pages across the whole pool.

### Mechanism: the buffer pool and the OS page cache

```text
 PostgreSQL (buffered I/O)                              InnoDB (commonly configured with O_DIRECT)
 ┌──────────────────┐                                   ┌──────────────────┐
 │ shared_buffers   │ ← page cached here                │ buffer pool      │ ← page cached here only
 └────────┬─────────┘                                   └────────┬─────────┘
          │ read()/write()                                       │ O_DIRECT read/write (bypasses page cache)
 ┌────────▼─────────┐                                            │
 │ OS page cache    │ ← and possibly here too                    │
 └────────┬─────────┘                                            │
          ▼                                                      ▼
        disk                                                   disk
```

PostgreSQL reads and writes its data files through the operating system's page cache (Chapter 4, Section 4.3). A page can therefore be cached twice: once in `shared_buffers` and once in the page cache. This is **double buffering**. It wastes some memory, but the page cache also acts as a second-level cache, and the OS performs readahead for sequential scans. This is why PostgreSQL's documentation suggests a modest share of RAM for `shared_buffers` as a starting point (commonly cited as about 25%) and leaves the rest to the OS.

InnoDB is commonly configured to bypass the page cache with `O_DIRECT` (Chapter 4, Section 4.4), so a page is cached only in the buffer pool. The buffer pool is then usually sized to take most of the machine's memory (a large majority of RAM on a dedicated database server is a common recommendation), because nothing else will cache the data.

Either way, the buffer pool decides which reads hit memory. A read that misses both caches goes to the device, and on an HDD a random read costs milliseconds (on the order of 100 times a sequential one, Chapter 2, Section 2.4); on an SSD it costs tens to hundreds of microseconds, with a much smaller gap between random and sequential.

### Concrete: measuring the hit ratio

```sql
-- PostgreSQL: blocks found in shared_buffers versus read from the OS (page cache or disk)
SELECT datname, blks_hit, blks_read,
       round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) AS hit_pct
FROM pg_stat_database WHERE datname = 'shop';
```

```text
 datname │ blks_hit    │ blks_read │ hit_pct
 shop    │ 9813402211  │ 41022910  │ 99.58
```

A hit ratio is a coarse signal: it is cumulative since the last statistics reset, and in PostgreSQL a "read" may still be served by the OS page cache. A sudden drop (a new query scanning a large table, a cache cold after a restart) is more informative than the absolute number. `EXPLAIN (ANALYZE, BUFFERS)` shows hits and reads for one query (Chapter 16).

### Guarantees and non-guarantees

The buffer pool guarantees that every process sees one consistent in-memory copy of each page, that pinned pages stay in memory while in use, and that no dirty page reaches disk before its log records (the WAL rule, Section 15.4). It does not guarantee that a given query's pages are cached, that a cold cache after a restart performs like a warm one, or that a single large scan cannot degrade other queries (scan resistance reduces this but does not eliminate it).

### Failure and diagnosis

- **Cold cache after restart or failover.** Latency is high for minutes after a restart while the hot set is read from disk. Evidence: low hit ratio and high read I/O just after startup. Mitigation: prewarming (PostgreSQL's `pg_prewarm` extension; InnoDB can dump and reload the list of buffer pool pages across restarts).
- **Working set larger than memory.** Constant evictions and reads; latency tracks disk latency. Evidence: persistently high read I/O, `iostat` showing busy devices (Chapter 4, Section 4.12). Fix: more memory, fewer or narrower rows read (indexes, Chapter 16), or archiving cold data.
- **Write stalls.** Clients wait for clean frames because dirty pages are written too slowly. Evidence: backends writing pages themselves (in PostgreSQL, buffers written by backends rather than the background writer or checkpointer); InnoDB reporting waits for free pages. Fix: faster storage, tuned background writing, smoother checkpoints (Section 15.5).

### Backend connection

The buffer pool is why "the database is fast" until the hot data no longer fits in memory, and then suddenly it is not. Size it deliberately for the engine (modest for PostgreSQL with the OS cache behind it; large for InnoDB with `O_DIRECT`), watch read I/O rather than only the hit ratio, and expect slower responses after restarts and failovers until the cache warms.

---

## 15.4 Write-Ahead Logging

### Definitions

A **write-ahead log** (WAL; InnoDB calls it the **redo log**) is an append-only file in which the database records every change to its pages **before** the changed pages themselves are written to the data files.

A **log record** describes one change: which transaction made it, which page it affects, and enough information to **redo** the change on that page (for example, "on page 0 of `orders`, insert this tuple at slot 4").

A **log sequence number** (LSN) is the position of a record in the log, a number that only increases. In PostgreSQL it is a byte offset into the WAL stream, shown as two hexadecimal halves, such as `0/3A1F2B8`. Every page header stores the LSN of the last record that modified it, the **page LSN**.

The **WAL rule** has two halves:

1. **Log before data.** A modified page may be written to the data file only after every log record describing its modifications has reached durable storage.
2. **Log before commit.** A transaction is committed, and may be acknowledged to the client, only after its **commit record** has reached durable storage.

**Group commit** is flushing the log once for many transactions that commit at about the same time, so that they share one `fsync`.

### Why it exists

To make a commit durable without a log, the database would have to write every modified page to its data file and `fsync` it before answering. A transaction that changes five rows on five different pages would cost five random page writes (40 KB with 8 KB pages, for perhaps 500 bytes of actual change) plus the synchronisation, on every commit. On an HDD each random write costs milliseconds; on an SSD the cost is lower but still far above a sequential append (Chapter 2, Section 2.4). Worse, the five page writes are not atomic: a crash after three of them leaves the transaction half-applied on disk.

The log solves both problems. Commit costs one **sequential append** of small records plus one `fsync` (shared with other transactions through group commit). The data pages can be written later, in the background, in any order, many changes at a time. And because the log holds a complete description of every committed change, a crash at any moment can be repaired by replaying it (Section 15.5).

### Mechanism: the write path of one commit

```text
 client                 backend / connection thread                          disk
   │ BEGIN
   │ UPDATE accounts SET balance = 400 WHERE id = 7
   │──────────────────► 1. find page 12 of accounts in the buffer pool (read it if needed), pin it
   │                    2. build a log record:
   │                         LSN 0/3A1F2B8  tx 731  "accounts page 12, slot 5: new tuple (7, 400)"
   │                       append it to the WAL buffer (shared memory)          (nothing on disk yet)
   │                    3. apply the change to the page in memory; page LSN := 0/3A1F2B8; mark dirty
   │◄── UPDATE 1 ───────
   │ COMMIT
   │──────────────────► 4. append the commit record   LSN 0/3A1F340  tx 731 COMMIT
   │                    5. write the WAL buffer to the WAL file up to 0/3A1F340 ─────────────► WAL file
   │                    6. fsync (or fdatasync) the WAL file                    ──── flush ──► device
   │                       ... waits for the device to confirm ...
   │◄── COMMIT ─────────7. only now: acknowledge
   │
   │                    later (seconds to minutes), in the background:
   │                    8. checkpointer / page cleaner: is the WAL flushed up to page 12's LSN?
   │                       yes → write page 12 to the data file ────────────────────────────► data file
```

At step 7, the change exists durably **only in the log**. The data file still holds the old balance. That is safe: if the server crashes now, recovery replays the log and rebuilds the change (Section 15.5). Step 8 is where the first half of the WAL rule bites: the page must not reach the data file before its log record, because if it did, and the transaction later turned out not to have committed, nothing in the log would describe what had been written.

In PostgreSQL the WAL lives in the `pg_wal` directory as a sequence of segment files, 16 MB each by default; the current position is visible with `SELECT pg_current_wal_lsn();`. InnoDB's redo log is a set of files of fixed total capacity, written circularly: space can be reused only after the changes it describes have reached the data files (Section 15.5).

### Mechanism: what the log looks like

PostgreSQL's `pg_waldump` tool decodes WAL files. An abridged excerpt for the transaction above:

```text
 rmgr: Heap   len (rec/tot): 72/72, tx: 731, lsn: 0/03A1F2B8, prev 0/03A1F280, desc: HOT_UPDATE ... blkref #0: rel 1663/16384/16402 blk 12
 rmgr: Transaction len (rec/tot): 34/34, tx: 731, lsn: 0/03A1F340, prev 0/03A1F2B8, desc: COMMIT 2026-10-21 10:15:03.112 UTC
```

Each record names its **resource manager** (heap, B-tree, transaction), its transaction, its LSN, the previous record's LSN, and the block it changes. The commit record is tiny; most of the log's volume is the descriptions of page changes.

### Mechanism: the cost of fsync, and group commit

The latency of a commit is dominated by step 6: waiting for the storage device to confirm that the log is durable (Chapter 4, Section 4.4). That latency varies enormously with the device:

| Storage (illustrative orders of magnitude; measure yours) | Typical `fsync` latency |
|---|---|
| Enterprise SSD with power-loss protection (it can acknowledge from protected cache) | tens of microseconds to well under a millisecond |
| Consumer SSD without power-loss protection | around a millisecond to several milliseconds |
| Cloud network block storage | around a millisecond or more, varying with load |
| HDD | several milliseconds (rotation and seek) |

If each commit needed its own `fsync`, and one `fsync` took 1 ms, a database could commit at most about 1,000 transactions per second, however many CPU cores it had. **Group commit** breaks that limit. While one flush is in progress, other transactions append their commit records to the WAL buffer and wait. When the flush finishes, the next flush covers all of their records at once:

```text
 time ───────────────────────────────────────────────────────────────────►
 flush #1 (1 ms): covers T1's commit record
                  meanwhile T2, T3 … T20 append commit records and wait
 flush #2 (1 ms): covers T2 … T20 in one fsync → 19 commits acknowledged together
 → throughput rises from ~1,000 to ~20,000 commits/s at the same 1 ms fsync latency;
   each commit still waits about one to two flush times
```

PostgreSQL does this naturally (any backend that flushes covers everyone's records up to its point), and can be told to wait briefly to gather more committers (`commit_delay`). InnoDB does the same for its redo log, and coordinates it with the binary log used for replication.

### Mechanism: choosing less durability, deliberately

Both engines can acknowledge a commit **before** the log is flushed, trading the most recent commits for latency:

| Setting | Behaviour | What a crash can lose |
|---|---|---|
| PostgreSQL `synchronous_commit = on` (default) | flush the WAL at each commit | nothing acknowledged |
| PostgreSQL `synchronous_commit = off` (can be set per transaction) | acknowledge, flush shortly after in the background | the last fraction of a second of acknowledged commits; never corruption |
| InnoDB `innodb_flush_log_at_trx_commit = 1` (default) | write and flush the redo log at each commit | nothing acknowledged |
| InnoDB `… = 2` | write to the OS at each commit, flush about once per second | about a second of commits if the **machine** crashes (a `mysqld` crash alone loses nothing) |
| InnoDB `… = 0` | write and flush about once per second | about a second of commits, even if only `mysqld` crashes |

The data stays consistent in every mode: the log is still written in order and replayed on recovery. What is lost is a short tail of commits the client was told succeeded. That can be acceptable for a click log or a metrics table, and is never acceptable for payments. PostgreSQL allows choosing per transaction, so the same database can be fully durable for orders and relaxed for analytics events. The client-side lesson is the one from Chapter 7 (Section 7.15) in a new place: an acknowledgement is only as strong as the durability setting behind it.

### Mechanism: torn pages, and why the log sometimes holds whole pages

A database page (8 or 16 KB) is larger than the unit the storage device writes atomically (often 4 KB, sometimes 512 bytes). If power fails in the middle of writing a page to the data file, the page on disk can be **torn**: part new, part old. Log records describe changes **relative to** a page's previous contents, so they cannot repair a page whose contents are garbage.

- **PostgreSQL** writes a **full-page image** into the WAL the first time each page is modified after a checkpoint (`full_page_writes`, on by default). Recovery restores the whole page from that image and replays later changes on top.
- **InnoDB** uses a **doublewrite buffer**: dirty pages are first written to a dedicated doublewrite area and flushed, and only then written to their real locations. After a crash, a torn page is restored from its doublewrite copy.

Both protections cost write volume. In PostgreSQL, the WAL rate jumps right after each checkpoint, while many pages are modified for the first time since it; Section 15.5 returns to this.

### Guarantees and non-guarantees

With durable settings and storage that honours flushes, WAL guarantees that every acknowledged commit survives a crash, and that a crash never leaves a half-applied transaction visible. It does not guarantee durability if the device or a virtualisation layer acknowledges flushes it has not performed (the `fsync` lie, Chapter 4, Section 4.4), if relaxed commit settings are in use, or if the disk itself is lost (that needs replication and backups, Section 15.9 and Chapter 22). In PostgreSQL and InnoDB alike, the WAL is a **redo** log: it describes how to reapply changes. How a transaction is undone differs between them, and Section 15.5 explains it.

### Failure and diagnosis

- **Commit latency equals storage latency.** Every write transaction takes a few milliseconds even though queries are fast. Evidence: PostgreSQL's `pg_stat_activity` showing backends in the `WalSync` or `WalWrite` I/O wait events; `pg_test_fsync` (a tool shipped with PostgreSQL) measuring the device's flush rate; `iostat` showing high write `await` on the WAL device (Chapter 4, Section 4.12). Fix: storage with power-loss protection, a separate fast WAL device, larger transactions where batching is acceptable, or relaxed commit for non-critical data.
- **Write throughput collapses on cheap storage.** Commits per second cap at roughly the inverse of `fsync` latency when concurrency is low. Fix: more concurrency (group commit), batching, better storage.
- **Acknowledged data missing after a crash.** Relaxed commit settings, or storage that lies about flushes. Evidence: configuration review; vendor documentation for write caches.

### Backend connection

Every committed write pays for one durable log flush, shared among concurrent commits. That makes the storage device's flush latency part of your API's latency, makes many tiny transactions expensive, and makes batching cheaper per row. Decide durability per kind of data rather than globally, and never accept "fast commits" from storage you cannot trust to flush.

---

## 15.5 Crash Recovery and Checkpoints

### Definitions

**Crash recovery** is the process that runs when a database starts after an unclean shutdown: it brings the data files to a state that contains every committed transaction and no effect of any uncommitted one.

**Redo** reapplies changes recorded in the log to pages that may not contain them. **Undo** removes changes made by transactions that did not commit.

A **checkpoint** is a point at which the database guarantees that all changes logged before a certain LSN, the **redo point**, have been written to the data files. Recovery can then start replaying from the redo point instead of from the beginning of the log.

Two buffer-management policies decide what recovery must do:

- **Steal**: the buffer pool may write a page containing **uncommitted** changes to disk (for example, to free a frame). With steal, the disk can hold uncommitted data after a crash, so some form of undo is needed.
- **No-force**: committed changes do **not** have to be written to the data files at commit. With no-force, the disk can lack committed data after a crash, so redo is needed.

Almost every production database uses **steal and no-force**, because the alternatives (keeping all uncommitted pages in memory, or writing every page at commit) are too expensive.

**ARIES** (Algorithms for Recovery and Isolation Exploiting Semantics, published by IBM researchers in 1992) is the classic recovery design for steal/no-force databases, and the conceptual basis of most engines' recovery.

### Why it exists

A crash stops the server at an arbitrary instant. Some committed changes are only in the log (no-force); some uncommitted changes may already be on disk (steal); some pages may be torn. Recovery has to resolve all three, automatically, before the database accepts connections. Checkpoints exist because without them recovery would have to replay the log from the day the database was created.

### Mechanism: ARIES in three phases

```text
 1. ANALYSIS   read the log from the last checkpoint; rebuild which transactions were active
               at the crash ("losers") and which pages might be dirty
 2. REDO       "repeat history": from the redo point, reapply EVERY logged change, committed or not,
               to every page that lacks it. A change is applied only if page LSN < record LSN,
               so redo is idempotent: running recovery twice gives the same result
 3. UNDO       roll back the losers' changes, newest first, following each transaction's chain of
               log records; each undo step is itself logged (a compensation log record) so that a
               crash during recovery does not undo something twice
```

The page LSN is what makes redo safe to repeat. If a page on disk has page LSN 160, then every record with LSN up to 160 is already reflected in it, and redo skips them.

### Mechanism: how PostgreSQL and InnoDB differ

Both engines are steal/no-force and both use the log for **redo**. They differ in how they deal with uncommitted changes:

- **InnoDB updates rows in place** (Section 15.2), so an uncommitted change that reached disk has overwritten the old value. The old value lives in the **undo log**. Rollback, at run time or during recovery, applies undo records to restore old values. Undo pages are themselves protected by the redo log, so after redo they are intact. InnoDB recovery therefore runs redo, then rolls back the transactions that were active at the crash using undo (much of this in the background after the server starts accepting connections).
- **PostgreSQL never overwrites tuples** (Section 15.2). An uncommitted transaction's changes are new tuple versions, marked with its transaction ID. Whether a transaction committed is recorded in a separate **commit log** (`pg_xact`), itself WAL-protected. Rollback simply records the transaction as **aborted**; its tuple versions become invisible to everyone (Chapter 17) and are removed later by VACUUM. PostgreSQL has no undo log and **no undo phase** in recovery: it replays the WAL, and any transaction without a commit record is treated as aborted.

| | PostgreSQL | InnoDB |
|---|---|---|
| Log used for redo | WAL | redo log |
| Where old row versions live | in the table, as dead tuples | in the undo log |
| Rollback | mark aborted in `pg_xact`; versions invisible; VACUUM cleans up | apply undo records |
| Recovery phases | redo from the checkpoint's redo point; no undo phase | redo, then roll back active transactions with undo |
| Cost paid later | VACUUM must remove dead tuples | purge must discard old undo |

### Concrete: a crash and its recovery

A sequence of log records, with LSNs written as small numbers for readability:

```text
 LSN  record                                            state on disk at the crash
 100  CHECKPOINT begins (redo point = 100)
 110  T1: update page A (account 7: 500 → 400)          page A on disk, page LSN 110 (written during the checkpoint)
 120  T2: update page B (account 9: 50 → 80)            page B on disk, page LSN 120 (stolen: T2 is uncommitted)
 130  T1: COMMIT                                        WAL flushed up to 130
 140  CHECKPOINT ends (record points back to redo point 100)
 150  T2: update page C (account 11: 10 → 0)            page C on disk is OLD (page LSN 40)
 160  T3: update page A (account 8: 0 → 25)             page A on disk is still LSN 110
 170  T3: COMMIT                                        WAL flushed up to 170
 175  T2: update page A (account 7: 400 → 390)          WAL flushed up to 175 by the background WAL writer
 ✕ CRASH (T2 never committed)
```

**Recovery.**

```text
 Analysis: start from the checkpoint (redo point 100). T1 committed (130), T3 committed (170),
           T2 has no commit record → T2 is a loser.

 Redo, from LSN 100:
   110 page A: page LSN 110 ≥ 110 → already applied, skip
   120 page B: page LSN 120 ≥ 120 → skip
   150 page C: page LSN 40 < 150   → apply (account 11 = 0)         ← uncommitted, reapplied anyway
   160 page A: page LSN 110 < 160  → apply (account 8 = 25)         ← committed, was missing from disk
   175 page A: page LSN 160 < 175  → apply (account 7 = 390)        ← uncommitted, reapplied anyway
   history is now exactly as at the crash

 Undo (InnoDB-style engine): T2 is a loser → undo its changes newest first, using undo records:
   175 → account 7 back to 400;  150 → account 11 back to 10;  120 → account 9 back to 50
   Result: T1's and T3's changes present, no trace of T2.

 PostgreSQL: no undo phase. T2's new tuple versions (accounts 9, 11 and 7) exist on the pages,
   but T2 has no commit record, so pg_xact treats it as aborted and every reader sees the
   previous versions (Chapter 17). VACUUM removes T2's versions later.
```

Note what recovery needed: the log from LSN 100 onward. Everything before the redo point could be discarded, because the checkpoint guaranteed it was already in the data files.

### Mechanism: what a checkpoint does

A PostgreSQL checkpoint:

```text
 1. note the current WAL position as the REDO POINT
 2. write every page that was dirty at that moment to the data files
    (spread over a large part of the interval to the next checkpoint, so as not to flood the disk)
 3. fsync the data files
 4. write a CHECKPOINT record to the WAL, and record its location in the control file (pg_control)
 5. WAL segments older than the redo point are no longer needed for crash recovery:
    recycle or remove them (unless still needed by archiving or replication, Section 15.9)
```

Checkpoints start when a time limit passes (`checkpoint_timeout`, 5 minutes by default in PostgreSQL at the time of writing) or when the WAL written since the last checkpoint reaches a size limit (`max_wal_size`, 1 GB by default at the time of writing), whichever comes first, and also at shutdown and on demand.

InnoDB uses **fuzzy checkpointing**: page cleaner threads write dirty pages continuously, oldest modification first, and the **checkpoint LSN** advances to the oldest change not yet written. The distance between the current LSN and the checkpoint LSN (the **checkpoint age**) must stay below the redo log's capacity, because redo space cannot be reused until the changes it describes are on disk. If writes outpace flushing and the checkpoint age nears capacity, InnoDB flushes aggressively and can stall writes until space is freed.

### Mechanism: the checkpoint trade-off, with numbers

Assume a database generating 20 MB of WAL per second at peak, and a recovery that replays WAL at about 100 MB per second (an assumption: replay speed depends on hardware and on how many replayed pages must be read from disk).

| | Checkpoint every 5 minutes | Checkpoint every 30 minutes |
|---|---|---|
| WAL to replay after a crash (worst case) | 20 MB/s × 300 s ≈ 6 GB → about 60 s of recovery | 20 MB/s × 1,800 s ≈ 36 GB → about 6 minutes |
| WAL kept on disk for recovery | about 6 GB | about 36 GB |
| Data-file writes | a hot page modified 1,000 times is written about every 5 minutes | the same page written once per 30 minutes: far fewer writes |
| Full-page images in the WAL (PostgreSQL) | one per page per 5 minutes: more WAL | one per page per 30 minutes |

Frequent checkpoints mean fast recovery but more total I/O; rare checkpoints mean less I/O but longer recovery and more WAL to store. A typical tuning keeps checkpoints driven by time rather than by WAL size (so that bursts do not trigger them too often), spreads each checkpoint's writes over most of the interval, and accepts a recovery time measured in a minute or two.

### Guarantees and non-guarantees

Crash recovery guarantees that after restart the database contains exactly the committed transactions (for those whose commit records were durable) and no uncommitted effects, provided the log and the storage behaved correctly. Checkpoints guarantee a bound on how much log recovery must replay. Neither guarantees a short recovery if checkpoints are rare and the write rate high; neither protects against losing the storage device itself; and neither helps against a logical mistake (a committed `DELETE` of the wrong rows is faithfully recovered), which needs backups and point-in-time recovery (Section 15.9).

### Failure and diagnosis

- **Periodic latency spikes.** Every few minutes, write latency and I/O jump. Cause: checkpoints writing many pages at once, or the burst of full-page images right after a checkpoint. Evidence: PostgreSQL logs each checkpoint (on by default in recent versions) with the number of buffers written and timing; spikes align with them. Fix: spread checkpoints (`checkpoint_completion_target`), raise `max_wal_size` so that time, not WAL volume, triggers them.
- **"checkpoints are occurring too frequently"**. PostgreSQL warns when WAL volume forces checkpoints closer together than intended, with a hint to increase `max_wal_size`.
- **Long restarts.** The database takes many minutes to come back after a crash. Cause: rare checkpoints with high write rates. Evidence: recovery log lines showing the redo range. Fix: more frequent checkpoints, or accept the trade-off with a standby for failover (Chapter 22).
- **InnoDB write stalls.** Writes pause under heavy load. Cause: checkpoint age approaching the redo log capacity. Evidence: `SHOW ENGINE INNODB STATUS` showing "Log sequence number" far ahead of "Last checkpoint at". Fix: larger redo capacity, faster storage, more page cleaning.

### Backend connection

Crash recovery is why a database can promise durability while writing data pages lazily; checkpoints decide how much that laziness costs at restart. When you size write-heavy workloads, expect periodic background I/O, plan for recovery time after a crash, and remember that recovery restores exactly what was committed, mistakes included.

---

## 15.6 Update-in-Place versus Log-Structured Storage: B-Trees and LSM-Trees

### Definitions

An **update-in-place** (page-oriented) storage engine keeps data in a structure of fixed-size pages, usually a **B+tree** (Chapter 16 teaches the structure itself), and changes a row by modifying the page that holds it, with a write-ahead log for durability. PostgreSQL, InnoDB, SQL Server, Oracle and SQLite work this way.

A **log-structured** storage engine never modifies data files in place. It buffers writes in memory and writes them out as new, immutable, sorted files, merging those files in the background. The dominant design is the **log-structured merge tree** (LSM-tree), introduced by Patrick O'Neil and colleagues in 1996, and used by RocksDB and LevelDB, by systems built on them (MyRocks, TiKV), by CockroachDB's Pebble engine, and by Cassandra, ScyllaDB and HBase.

LSM vocabulary:

- A **memtable** is an in-memory sorted structure (often a skip list) that receives all writes.
- An **SSTable** (sorted string table) is an immutable file of key–value pairs sorted by key, with an index block and usually a **bloom filter**.
- **Compaction** merges SSTables into new ones, discarding overwritten values and deleted keys.
- A **tombstone** is a record that marks a key as deleted; it hides older values until compaction removes both.
- A **bloom filter** is a compact probabilistic structure that answers "is this key possibly in this file?" with no false negatives and a tunable rate of false positives.

The costs are described as **amplification**:

- **write amplification**: bytes written to storage per byte of data the application wrote;
- **read amplification**: work (files or pages read) per logical read;
- **space amplification**: bytes on disk per byte of live data.

### Why it exists

Update-in-place engines turn each logical write into a modification of a page somewhere in the file: a random write, eventually. On HDDs, random writes cost on the order of 100 times more than sequential ones (Chapter 2, Section 2.4). Workloads that write far more than they read (event logs, metrics, messaging, time series, large key–value stores) were limited by random-write throughput. LSM-trees convert all writes into sequential appends and sequential file writes, at the cost of more work on reads and in background merging. On SSDs the random-versus-sequential gap is much smaller, but LSM-trees remain attractive for high ingest rates, compression, and reducing the number of small in-place writes that wear flash (Chapter 2, Section 2.4).

### Mechanism: the LSM write path

```text
 write(key, value)
   │
   ├─► 1. append to the LSM's own write-ahead log (sequential; fsync per its durability setting)
   └─► 2. insert into the MEMTABLE (sorted, in memory)
           │  memtable full (e.g. tens of MB)?
           ▼
        3. freeze it as an IMMUTABLE MEMTABLE; start a new memtable for new writes
           │
           ▼
        4. FLUSH: write it out as a new SSTable in level 0 (one sequential write); drop its WAL
           │
           ▼
 L0:  [sst 9] [sst 8] [sst 7]          ← L0 files may overlap in key range (each is one flushed memtable)
           │  compaction: merge L0 files with the overlapping L1 files
           ▼
 L1:  [a–f] [g–m] [n–s] [t–z]          ← from L1 down, files in a level have disjoint key ranges
           │  each level about 10× the size of the one above (leveled compaction)
           ▼
 L2:  [a–b][c–d][e–f] … [y–z]
 L3:  …
```

A write touches only memory and a sequential log append, so writes are fast and predictable. The cost moves into **compaction**, which repeatedly reads files and rewrites their contents into the next level. During a merge, for each key the newest version wins; overwritten values are dropped, and a tombstone removes older values of its key (the tombstone itself is dropped once it reaches the last level).

### Mechanism: the LSM read path, and bloom filters

```text
 get(key)
   1. memtable                    found? return (newest data)
   2. immutable memtables         found? return
   3. L0 files, newest first      for each file: bloom filter says "maybe"? → read its index, then the block
   4. L1: the ONE file whose range contains key → bloom filter → maybe read
   5. L2, L3, …: same
   the first level that has the key holds its newest version; a tombstone means "deleted"
```

Without help, a lookup for a key that exists only in the last level, or not at all, would read a block from every level. **Bloom filters** prevent most of that. A bloom filter for a file is an array of `m` bits and `k` hash functions:

```text
 adding a key:  set bits h1(key), h2(key), …, hk(key) to 1
 checking a key: if any of those bits is 0 → the key is DEFINITELY NOT in the file (skip the file)
                 if all are 1               → the key is POSSIBLY in the file (read it to find out)

 example, m = 16 bits, k = 3:
   add "order:42"  → bits 3, 9, 14       bits: 0001 0000 0100 0010
   add "order:77"  → bits 1, 9, 12       bits: 0101 0000 0100 1010
   check "order:99" → bits 3, 5, 12      bit 5 is 0 → definitely absent, no disk read
```

The false-positive rate depends on the number of bits per key. With the best number of hash functions:

| Bits per key | Hash functions | False-positive rate |
|---|---|---|
| 5 | 3 | about 9% |
| 10 | 7 | about 0.8% |
| 16 | 11 | about 0.05% |

At about 10 bits (1.25 bytes) per key, 99% of the files that do not contain a key are skipped without reading them. Bloom filters do not help range scans: a scan from `order:100` to `order:200` must open an iterator in every level whose range overlaps and merge their outputs.

### Mechanism: amplification compared

| | B-tree (update in place, with WAL) | LSM, leveled compaction | LSM, size-tiered compaction |
|---|---|---|---|
| Write amplification | each small change eventually rewrites a whole page (8–16 KB), plus WAL, plus occasional full-page images; high for random small updates, lower when many changes hit the same page between flushes | each key is rewritten roughly once per level per merge; commonly tens of times over its life | lower than leveled: files are merged less often |
| Read amplification (point read) | the tree's height (3–4 levels; upper levels cached): usually about one device read | memtable + L0 files + one file per level, mostly skipped by bloom filters: usually about one device read, more for L0 | several files per tier: higher |
| Read amplification (range scan) | a sequential walk of leaf pages | a merge of iterators across all levels: higher | higher |
| Space amplification | free space in pages (pages split by random inserts are commonly about two-thirds full), plus dead tuples in PostgreSQL | low (often cited around 10% extra) | high: obsolete copies until merged; temporarily up to about double during large merges |
| Write pattern on the device | random page writes plus sequential WAL | sequential writes only | sequential writes only |
| Latency predictability | stable; checkpoints cause periodic I/O | background compaction can fall behind and stall writes | larger, burstier merges |
| Deletes | remove (or mark) the row in its page | write a tombstone; space freed only by compaction | same |

The figures are qualitative orders of magnitude; the real numbers depend on row sizes, key distribution, configuration and the workload. The general rule, sometimes stated as the **RUM conjecture**, is that a storage design can optimise for at most two of read cost, update cost and memory (space) overhead at the expense of the third.

### Concrete: choosing between them

| Workload | Better fit | Reason |
|---|---|---|
| OLTP with mixed reads and updates, range queries, secondary indexes (orders, users, payments) | B-tree | predictable reads and range scans; mature transactional engines |
| Very high write rates of new keys (events, metrics, logs, time series, messaging) | LSM | sequential writes, compression, high ingest |
| Large key–value data on flash where storage cost dominates | LSM | better space amplification and compression |
| Read-mostly data with frequent range scans | B-tree | no merge of many levels on each scan |
| Heavy deletes followed by scans of the deleted range | B-tree (or careful LSM tuning) | LSM scans must step over tombstones until compaction |

### Guarantees and non-guarantees

Both designs, with a write-ahead log and correct `fsync` use, guarantee durability of acknowledged writes. A B-tree guarantees stable read performance and in-place updates; it does not guarantee cheap small random writes. An LSM-tree guarantees cheap, sequential writes; it does not guarantee that compaction keeps up (when it falls behind, read amplification grows and writes are throttled), and it does not free the space of deleted data until compaction reaches it.

### Failure and diagnosis

- **LSM write stalls.** Write latency suddenly rises or writes pause. Cause: compaction cannot keep up, so the number of L0 files or pending compaction bytes passes a threshold and the engine slows writers deliberately. Evidence: the engine's statistics (RocksDB reports stall counters and pending compaction bytes); I/O saturated by compaction. Fix: more I/O capacity, tuned compaction, or smoothing the write rate.
- **Slow reads after mass deletes (LSM).** Range scans read through thousands of tombstones. Evidence: tombstone counts in query traces (Cassandra, for example, logs warnings when a read scans many tombstones). Fix: data models that avoid deleting within hot ranges; time-based partitioning that drops whole files.
- **Disk space not freed after deletes.** In LSMs until compaction; in PostgreSQL until VACUUM (and returned to the OS only by rewriting the table). Evidence: on-disk size versus live row count.
- **B-tree write bottleneck.** Random-write-heavy ingestion saturating IOPS. Evidence: high write IOPS and `await` with low throughput. Fix: batching, append-friendly keys (Chapter 16), or an LSM-based store for that data.

### Backend connection

Most backend services store their system of record in a B-tree engine, and that is the right default for transactional data. Reach for an LSM-based store when a measured workload is dominated by high-volume writes of new keys, and budget for compaction I/O and for the different behaviour of deletes. When you meet a database you do not know, ask which of the two designs it uses: it predicts how writes, reads, deletes and disk usage will behave.

---

## 15.7 Row-Oriented versus Column-Oriented Storage

### Definitions

A **row-oriented** (row store) layout stores all the columns of a row together: page after page of complete rows. A **column-oriented** (column store) layout stores each column separately: all values of `status` together, all values of `total` together, with rows reassembled by position.

**OLTP** (online transaction processing) workloads run many small transactions that read and write a few rows each, by key: place an order, update a payment. **OLAP** (online analytical processing) workloads run fewer, large queries that read many rows but few columns, and aggregate them: revenue per category per month over two years.

**Vectorised execution** processes values in batches (vectors) of hundreds or thousands at a time, in tight loops, instead of one row at a time.

### Why it exists

An analytical query that sums one column of a 100-million-row table needs 4 bytes per row, but a row store must read whole rows to get them. If rows are 120 bytes wide, the query reads 30 times more data than it uses. For OLTP the opposite holds: placing an order writes one row with all its columns, and a column store would have to touch a separate file for every column. The two layouts exist because the two workloads read data in opposite directions.

### Concrete: one table, two layouts

```text
 orders
 id │ customer │ status    │ total │ created_at
 1  │ 81       │ PAID      │ 3000  │ 2026-10-01 10:00
 2  │ 17       │ PAID      │ 2500  │ 2026-10-01 10:02
 3  │ 81       │ CANCELLED │ 9900  │ 2026-10-01 10:05
 4  │ 42       │ PAID      │ 1200  │ 2026-10-01 10:09

 ROW STORE (one page)                              COLUMN STORE (one file or segment per column)
 ┌──────────────────────────────────────────────┐  id:          1, 2, 3, 4
 │ [1|81|PAID|3000|10:00] [2|17|PAID|2500|10:02]│  customer:    81, 17, 81, 42
 │ [3|81|CANCELLED|9900|10:05] [4|42|PAID|…]    │  status:      PAID, PAID, CANCELLED, PAID
 └──────────────────────────────────────────────┘  total:       3000, 2500, 9900, 1200
                                                   created_at:  10:00, 10:02, 10:05, 10:09
 SELECT * FROM orders WHERE id = 3;                → row store: one page read, one row
                                                     column store: one value from each of five columns
 SELECT status, SUM(total) FROM orders GROUP BY status;
                                                   → row store: read every column of every row
                                                     column store: read only status and total
```

With 100 million rows of 120 bytes, the analytical query reads about 12 GB from a row store. A column store reads the `status` and `total` columns only. Before compression, at 1 byte per dictionary-encoded status and 4 bytes per total, that is 500 MB, and compression typically shrinks it further.

### Mechanism: why columns compress well

Values in one column have the same type and often repeat or change slowly, so simple encodings shrink them dramatically:

| Encoding | Idea | Example |
|---|---|---|
| Dictionary | store each distinct value once; replace values by small codes | `status`: PAID → 0, CANCELLED → 1; four values in 4 bits |
| Run-length (RLE) | store (value, count) for runs of equal values; strongest when data is sorted | a column sorted by date: `2026-10-01 × 18,402` |
| Bit-packing | store small integers in as few bits as they need | quantities 1–100 in 7 bits instead of 32 |
| Delta | store differences between neighbours | timestamps a few seconds apart: small deltas instead of 8-byte values |

Compression helps twice: less data read from storage, and more data fitting in memory and CPU caches. Some engines compute directly on compressed data (comparing dictionary codes instead of strings).

### Mechanism: vectorised execution

A traditional row-at-a-time executor (the iterator model, Chapter 16) calls a function per operator per row: for 100 million rows, hundreds of millions of function calls, each doing a little work and branching unpredictably. A vectorised executor works on a batch of, say, 1,024 values from one column at a time:

```text
 row at a time:     for each row: next() → evaluate status = 'PAID' → if true: add total     (per row: calls, branches)
 vectorised:        for each batch of 1,024:
                       mask[i]  = (status_codes[i] == PAID_CODE)       one tight loop, SIMD-friendly
                       sum     += Σ total[i] where mask[i]             another tight loop
```

Tight loops over contiguous arrays use CPU caches well and let the compiler use SIMD instructions that process several values per instruction (Chapter 2, Sections 2.1 and 2.3). Column layout and vectorised execution reinforce each other: the column is already a contiguous array.

### Mechanism: skipping data with zone maps

Column stores split each column into **segments** (also called row groups or granules) of many thousands of rows, and keep small metadata per segment, the **zone map**: at least the minimum and maximum value. A query's filter is checked against the zone maps first, and segments that cannot contain a match are never read.

```text
 created_at column, 100 million rows in 1,000 segments of 100,000 rows, data loaded in time order

 segment │ min(created_at)    │ max(created_at)
 0       │ 2024-10-01 00:00   │ 2024-10-01 17:42
 1       │ 2024-10-01 17:42   │ 2024-10-02 10:05
 …       │ …                  │ …
 997     │ 2026-10-20 06:11   │ 2026-10-20 23:58
 998     │ 2026-10-20 23:58   │ 2026-10-21 14:30
 999     │ 2026-10-21 14:30   │ 2026-10-21 23:59

 WHERE created_at >= '2026-10-21'  → only segments 998 and 999 can match: 2 of 1,000 segments read
```

Zone maps work only when the data is **clustered** by the filtered column. Here the data was loaded in time order, so each segment covers a narrow time range. A filter on `customer`, whose values are scattered across every segment, matches every segment's min–max range and skips nothing. That is why column stores let you declare a **sort key** for each table: choosing the column that most queries filter on is the main physical design decision in a column store, playing the role that index design plays in a row store (Chapter 16).

### Mechanism: the cost of columns for writes

Inserting one row into a column store means appending a value to every column's segment, and updating a row means rewriting values in several compressed segments. Column stores therefore favour **batch** loading. Many buffer recent writes in a small row-oriented **delta store** and merge it into compressed column segments in the background, much like an LSM memtable (Section 15.6). They are fast at appending large batches and slow at many small updates.

| | Row store | Column store |
|---|---|---|
| Best for | OLTP: point reads, single-row inserts and updates, many concurrent transactions | OLAP: scans and aggregations over many rows and few columns |
| Reads of one whole row | one page | one value per column |
| Scans of a few columns | read every column | read only those columns |
| Compression | modest | high |
| Single-row writes | cheap | expensive; batch loads preferred |
| Examples | PostgreSQL, MySQL/InnoDB, SQL Server (default), Oracle (default) | ClickHouse, DuckDB, Snowflake, BigQuery, Amazon Redshift; Parquet and ORC as file formats |

### Guarantees and non-guarantees

A column layout guarantees that queries read only the columns they reference, and enables compression and vectorised execution. It does not guarantee fast point lookups or cheap updates, and analytical speed still depends on filtering (column stores keep min/max values per segment to skip segments that cannot match). A row layout guarantees cheap whole-row access and writes, not fast wide scans.

### Failure and diagnosis

- **Analytical queries slow the transactional database.** A report scanning two years of orders saturates I/O and evicts the hot set from the buffer pool (Section 15.3), and checkout latency rises. Evidence: long-running scans in `pg_stat_activity` coinciding with latency spikes. Fix: run analytics on a replica (Chapter 22) or move them to a column store fed by change data capture (Chapter 25).
- **Single-row writes to a column store crawl.** Evidence: insert latency per row high; many small parts or segments. Fix: batch writes, or buffer them upstream.

### Backend connection

Keep the system of record in a row store and send analytical workloads to a column store (a warehouse, or an embedded engine such as DuckDB for smaller data). Moving data between them is a pipeline problem (Chapter 25), and the latency of that pipeline decides how fresh the analytics are.

---

## 15.8 In-Memory Databases and Their Durability Options

### Definitions

An **in-memory database** keeps its primary copy of the data in RAM and serves reads and writes from memory. Durability, if any, comes from writing something to disk in the background or synchronously: a **snapshot** (a complete point-in-time copy), a **log of operations** (every write appended to a file), replication to other machines (Chapter 22), or a combination.

**Redis** is a widely used in-memory data store. Its two persistence mechanisms are **RDB** (periodic snapshots) and **AOF** (append-only file, a log of write commands). Redis as a cache and its data structures are Chapter 21's subject; here it serves as the standard example of in-memory durability choices.

### Why it exists

Memory is orders of magnitude faster than storage (Chapter 2, Section 2.2), and for data that fits in RAM an in-memory design avoids buffer management and page I/O entirely. But RAM is volatile: a restart or crash loses everything not written elsewhere. The design question is how much data you can afford to lose, and how much write latency and restart time you will pay to lose less.

### Mechanism: snapshots (RDB)

```text
 Redis process (parent)                                     child process
 │  serving clients in memory                               │
 │── fork() ─────────────────────────────────────────────► │ sees a copy-on-write image of memory
 │  keeps serving; each page it modifies is copied          │ (Chapter 3, Section 3.5)
 │  by the kernel before the write (copy-on-write)          │ writes the whole dataset to temp.rdb
 │                                                          │ fsync, rename temp.rdb → dump.rdb
 │◄────────────────────── child exits ─────────────────────│ (the write-temp-fsync-rename pattern,
 │                                                            Chapter 4, Section 4.4)
```

A snapshot is compact and fast to load at restart, and it costs the main process little: the child does the I/O. Its weaknesses are the loss window and `fork` itself. Everything written after the last completed snapshot is lost in a crash: with snapshots every few minutes, minutes of data. And `fork` on a large dataset must copy the process's page tables, which can pause the parent for a noticeable time, while heavy writes during the snapshot make the kernel copy many pages, raising memory use up to about double in the worst case.

### Mechanism: the append-only file (AOF)

With AOF enabled, Redis appends every write command to a log file. The `appendfsync` setting decides when that file is flushed:

| `appendfsync` | Behaviour | Loss window on a crash | Cost |
|---|---|---|---|
| `always` | `fsync` after every write | essentially none | every write waits for the device (Section 15.4) |
| `everysec` (the default) | `fsync` about once per second, in the background | about one second of writes | small |
| `no` | let the OS flush when it chooses | up to the OS's writeback delay (often tens of seconds, Chapter 4, Section 4.3) | smallest |

The log grows without bound, since it records every increment of every counter. Redis periodically **rewrites** it in the background (again by forking) into the minimal set of commands that recreates the current data. Modern versions can start the rewritten file with a compact RDB snapshot followed by the commands since, combining fast loading with a small loss window.

```text
 time ─────────────────────────────────────────────────────────────────────────►
 RDB only:   snapshot ··························· snapshot ··········✕ crash
                                                  └── writes since here are LOST (minutes)
 AOF every second:  writes logged, fsync each second ··············✕ crash
                                                          └── at most about 1 s LOST
```

### Mechanism: durability by replication

In-memory systems often rely on replicas for availability. Replication is not the same as durability on disk: Redis replication is **asynchronous** by default, so a primary can acknowledge a write, crash before sending it, and a replica promoted in its place never had it (Chapter 22 explains the general problem). The safest configurations combine an AOF on each node with replicas.

### Guarantees and non-guarantees

An in-memory database with `fsync`-per-write logging can be as durable as a disk-based one, at a similar per-write cost. With snapshots or per-second flushing, it guarantees durability only up to the last snapshot or flush. It does not guarantee that a promoted replica has every acknowledged write (asynchronous replication), and it does not guarantee bounded memory use during `fork`-based persistence.

### Failure and diagnosis

- **Latency spikes during persistence.** Evidence: spikes aligned with snapshot or AOF-rewrite start times; Redis reports the duration of its latest `fork` in its `INFO` output. Fix: smaller instances (sharding, Chapter 23), snapshots on replicas rather than the primary, adequate memory headroom.
- **Out-of-memory kills during snapshots.** Copy-on-write duplicating pages under write load, on a host with memory overcommit or a tight cgroup limit (Chapter 3, Section 3.17; Chapter 4, Section 4.10). Fix: memory headroom.
- **Data lost after a restart.** Persistence disabled or snapshot-only, discovered after the fact. Evidence: configuration; the age of the last successful save. Fix: choose the persistence mode deliberately and alert when saves fail.

### Backend connection

Decide for each in-memory store whether it is a cache (losing it is fine, Chapter 21) or a system of record (then it needs logged durability, replication and backups like any database). Put that decision in writing, because the default settings of many in-memory stores favour speed over durability.

---

## 15.9 Backups, Point-in-Time Recovery, RPO and RTO

### Definitions

A **backup** is a copy of the database, stored separately from it, from which the database can be rebuilt.

A **logical backup** contains the data as statements or portable records (`CREATE TABLE`, `INSERT` or a portable dump format). A **physical backup** contains the database's files (pages, WAL), in the engine's own format.

A **base backup** is a physical copy of the data files taken while the database runs, which becomes consistent when the WAL generated during the copy is replayed.

**WAL archiving** (continuous archiving) copies every completed WAL segment to separate storage as soon as it is written.

**Point-in-time recovery** (PITR) restores a base backup and replays archived WAL up to a chosen moment, such as just before an accidental `DELETE`.

The **recovery point objective** (RPO) is the maximum amount of data, measured in time, that the business accepts losing. The **recovery time objective** (RTO) is the maximum time the business accepts for restoring service.

### Why it exists

Crash recovery (Section 15.5) protects against crashes, with the storage intact. It does nothing against other causes of loss:

- the storage itself is lost (a failed volume, a deleted cloud disk, a lost data centre);
- a logical error is committed (`DELETE FROM orders` without a `WHERE`, a buggy migration, a corrupting bug in the application);
- an attacker encrypts or deletes the data (ransomware).

Replication does not protect against the second and third: a replica applies the bad `DELETE` within milliseconds. RAID does not protect against them either (Chapter 4, Section 4.13). Only an independent copy, from before the mistake, does.

### Mechanism: logical versus physical backups

| | Logical (`pg_dump`, `mysqldump`) | Physical (`pg_basebackup`, volume snapshots, Percona XtraBackup, MySQL's clone and enterprise backup tools) |
|---|---|---|
| Contents | schema and rows, as SQL or a portable format | data files and WAL, byte for byte |
| Consistency | `pg_dump` reads inside one transaction snapshot (Chapter 17); `mysqldump --single-transaction` does the same for InnoDB | consistent after replaying the WAL generated during the copy |
| Selectivity | one table, one schema, all | the whole server (cluster) |
| Portability | across major versions and even engines, with effort | same major version and platform |
| Backup speed on large databases | slow: reads every row | fast: copies files |
| Restore speed | slow: re-inserts every row and rebuilds every index | fast: copies files back, replays WAL |
| PITR | no | yes, with WAL archiving |
| Typical use | small databases, migrations, extracting one table | production disaster recovery |

### Mechanism: point-in-time recovery

```text
 Sunday 02:00      base backup taken (copy of data files + the WAL generated during the copy)
 Sunday → …        every completed WAL segment archived to object storage (archive_command or archive_library)
 Wednesday 14:03   a migration runs  DELETE FROM order_lines;  and commits
 Wednesday 14:10   the mistake is noticed

 Recovery to Wednesday 14:02:59:
   1. provision a new server; restore Sunday's base backup into its data directory
   2. configure where to fetch archived WAL from (restore_command) and the target:
        recovery_target_time = '2026-10-21 14:02:59+00'
   3. start in recovery mode (PostgreSQL 12+: a recovery.signal file in the data directory)
   4. the server replays archived WAL from Sunday 02:00 onward … stops at the target
   5. verify the data, then promote it (or copy the lost rows back into production)
```

MySQL's equivalent restores a physical or logical backup and then replays the **binary log** (the replication log, Chapter 22) with `mysqlbinlog` up to a time or position just before the bad statement.

### Concrete: estimating RPO and RTO for that plan

Assumptions: 500 GB database; base backups weekly; WAL generated at 70 GB per day; restore copies at 500 MB/s; WAL replay at 100 MB/s; WAL segments archived when full or at least every 60 seconds (PostgreSQL's `archive_timeout` forces a switch).

```text
 RPO (losing the primary server entirely):
   WAL not yet archived at the moment of loss: at most about 60 s of commits → RPO ≈ 1 minute
   (streaming the WAL to the archive continuously, e.g. with pg_receivewal, reduces this further)

 RTO for the Wednesday-afternoon recovery:
   restore base backup:   500 GB / 500 MB/s              ≈ 17 minutes
   replay WAL:            3.5 days × 70 GB ≈ 245 GB / 100 MB/s ≈ 41 minutes
   verify, repoint the application, warm the cache       ≈ 15–30 minutes
   total                                                 ≈ 75–90 minutes
   → daily base backups would cut the replay to about 12 minutes
```

The plan meets an RPO of a minute and an RTO of about an hour and a half. If the business needs a smaller RTO, take base backups more often, keep a standby (Chapter 22), or keep a **delayed replica**: a replica that applies WAL with a fixed delay, for example one hour, from which a mistake can be recovered by stopping it before it applies the bad change.

| Strategy | Typical RPO | Typical RTO | Protects against logical errors? |
|---|---|---|---|
| Nightly logical dump | up to 24 hours | hours (reload and rebuild indexes) | yes, back to the last dump |
| Physical base backup + WAL archiving (PITR) | about a minute (or less with streaming) | tens of minutes to hours | yes, to any moment |
| Synchronous replica | zero for loss of one node | seconds to minutes (failover) | no: the error replicates |
| Delayed replica | the delay | minutes | yes, within the delay |

### Mechanism: what makes a backup trustworthy

- **The 3-2-1 rule**: at least three copies of the data, on two different kinds of storage, one of them off-site (another region or account).
- **Immutability**: at least one copy that the production credentials cannot delete or overwrite (object storage with retention locks), so that an attacker who controls production cannot destroy the backups too.
- **Encryption** of backups at rest, with keys stored separately (Chapter 8, Section 8.3).
- **Monitoring**: alert when a backup or WAL archiving fails. PostgreSQL's `pg_stat_archiver` view counts failed archive attempts; a failing archive also makes WAL accumulate (Section 15.10).
- **Restore drills**: restore regularly into a separate environment, measure the time, and check the data. A backup that has never been restored is a hope, not a backup: the first test should not be the disaster.

### Guarantees and non-guarantees

A tested physical backup with continuous WAL archiving guarantees that the database can be rebuilt as of any moment between the oldest base backup and the last archived WAL, within a measured time. It does not guarantee anything about data written after the last archived segment, about backups that were never tested, or about backups stored with credentials an attacker also holds. Logical backups guarantee portability, not fast restores.

### Failure and diagnosis

- **Archiving silently failing.** Evidence: `failed_count` increasing in `pg_stat_archiver`; WAL accumulating in `pg_wal`. Consequence: PITR impossible past the last good segment, and eventually a full disk.
- **Restores slower than the RTO.** Discovered in a drill (or worse, in an incident). Fix: more frequent base backups, parallel restore, faster storage, or a standby.
- **Backups destroyed with production.** Same account, same credentials. Fix: separate account, immutable storage.

### Backend connection

Write down the RPO and RTO the business needs before choosing backup tools, then design to meet them and prove it with timed restore drills. Treat WAL archiving as part of the database: monitored, alerted, and tested. Remember that replicas give availability, and only backups give you the past.

---

## 15.10 Diagnosis Toolkit: Reading the Storage Layer's Evidence

### Definition

The **storage layer's evidence** is the set of statistics views, log lines and operating-system counters that show what the buffer pool, the log, checkpoints, archiving and the disks are doing. Most storage problems are visible in a handful of them.

### Why it matters

Storage problems surface as symptoms elsewhere: commit latency in an API, periodic spikes in a dashboard, a database that refuses writes. The evidence that explains them lives in places application developers rarely look. Knowing where to look turns "the database is slow sometimes" into a named cause within minutes.

### Concrete: the evidence and what it says

| Question | PostgreSQL | MySQL / InnoDB | Operating system |
|---|---|---|---|
| Are queries waiting on I/O or the log? | `pg_stat_activity.wait_event_type` / `wait_event` (e.g. `IO` / `WalSync`, `DataFileRead`) | `performance_schema` wait events | `iostat -x`: `await`, `%util` (Chapter 4, Section 4.12) |
| Are checkpoints frequent or heavy? | checkpoint log lines (`log_checkpoints`); `pg_stat_checkpointer` (PostgreSQL 17) or `pg_stat_bgwriter` (earlier) | `SHOW ENGINE INNODB STATUS`: "Log sequence number" versus "Last checkpoint at" | write bursts in `iostat` |
| How fast is WAL generated? | `pg_stat_wal` (PostgreSQL 14+); difference of `pg_current_wal_lsn()` over time | redo LSN growth in the status output | write throughput on the log device |
| Why is WAL accumulating? | `pg_replication_slots`, `pg_stat_archiver` | binary log retention settings | `du -sh` of the WAL directory |
| Is the cache effective? | `pg_stat_database` hits versus reads; `EXPLAIN (ANALYZE, BUFFERS)` | buffer pool hit rate and "pages read" in the status output | page-cache use in `free` |
| Is cleanup keeping up? | `pg_stat_user_tables.n_dead_tup`, last autovacuum time | "History list length" | |
| Is the disk filling? | errors in the server log | errors in the error log | `df -h`, `df -i` (inodes, Chapter 4, Section 4.2) |
| How fast is `fsync` here? | `pg_test_fsync` | | `fio` with sync writes |

A PostgreSQL checkpoint log line (representative; exact fields vary by version):

```text
LOG:  checkpoint complete: wrote 48213 buffers (18.4%); 0 WAL file(s) added, 12 removed, 31 recycled;
      write=269.912 s, sync=0.204 s, total=270.301 s; sync files=210, longest=0.041 s, average=0.001 s;
      distance=712345 kB, estimate=845210 kB
```

Reading it: 48,213 dirty pages (about 377 MB at 8 KB) were written, spread over 270 seconds, which is a well-spread checkpoint; the sync phase took 0.2 seconds, so the storage absorbed the writes; about 700 MB of WAL was generated since the previous checkpoint. A `checkpoint starting: wal` line (rather than `time`) would mean WAL volume, not the timer, triggered it.

Which replication slots are holding WAL back:

```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

```text
 slot_name        │ active │ retained_wal
 analytics_cdc    │ f      │ 212 GB
 standby_1        │ t      │ 48 MB
```

An inactive slot retaining 212 GB is a consumer that disconnected (Chapter 25 covers change-data-capture consumers) while the database keeps every WAL segment it has not yet received.

### Backend connection

Add these signals to the database's dashboards before you need them: WAL generation rate and retained WAL, checkpoint frequency and duration, archive failures, dead tuples, disk space and inodes, and `fsync` latency. The Diagnosis Lab at the end of this chapter uses them in four incidents.

---

## 15.11 Placement-Round Topics: File Organisation and Classic Recovery Theory

> **Placement-round topics.** This section is for campus placement rounds, university DBMS exams and written tests, where these topics are asked often. Product-company backend interviews rarely ask them in this form. **Give your main focus to Sections 15.1–15.10**; read this section before placement rounds and come back to it for revision.

### File organisation

How records are arranged in a file decides which operations are cheap:

| Organisation | Arrangement | Cheap | Expensive |
|---|---|---|---|
| **Heap (unordered)** | records wherever there is space | inserts; full scans | finding one record (without an index) |
| **Sequential (sorted)** | records ordered by a search key | range queries and ordered scans on that key; binary search | inserts in the middle (overflow chains or periodic reorganisation) |
| **Hashed** | a hash of the key chooses the bucket (page) | equality lookups on the key | range queries; growing beyond the bucket count (overflow chains) |
| **Clustered (index-organised)** | records stored in the leaves of an index on the key | key lookups and ranges | inserts that split pages (Chapter 16) |

Records are **fixed-length** (every field a fixed size: simple offsets, wasted space) or **variable-length** (needs a slot directory or length prefixes, as in the slotted page of Section 15.2). A record that does not fit in a page is **spanned** across pages (TOAST and overflow pages are the modern form).

### Log-based recovery, the textbook way

Exams describe recovery with simple log records `<T, X, old, new>` (transaction T changed item X from old to new), `<T start>` and `<T commit>`.

- **Deferred database modification**: a transaction writes nothing to the database until it commits; its changes are applied from the log afterwards. Recovery needs only **redo** for committed transactions; uncommitted ones never touched the database (this corresponds to no-steal).
- **Immediate database modification**: changes may reach the database before commit (steal), so recovery needs **undo** for uncommitted transactions (using `old`) and **redo** for committed ones (using `new`).

The recovery rule after a crash, with immediate modification: for each transaction with `<T start>` and `<T commit>` in the log, **redo** it; for each with `<T start>` but no commit, **undo** it, processing the log backwards. Undo before redo in the simplest textbooks; ARIES (Section 15.5) instead repeats history with redo first and then undoes losers, which handles more cases correctly.

**Worked example.** The log after a crash:

```text
 <T1 start> <T1, A, 1000, 900> <T1 commit> <T2 start> <T2, B, 2000, 2100> <T3 start> <T3, C, 700, 600> <T3 commit> ✕
 committed: T1, T3 → redo: A = 900, C = 600
 not committed: T2 → undo: B = 2000
```

### Checkpoints in the textbook model

Without checkpoints, recovery scans the whole log. A checkpoint record `<checkpoint L>` (where `L` lists active transactions) written after flushing all modified buffers lets recovery ignore every transaction that committed before the checkpoint: only transactions in `L` or started after the checkpoint need attention. Real systems spread the flushing ("fuzzy" checkpoints, Section 15.5) instead of pausing all work.

### Shadow paging

**Shadow paging** avoids a log altogether. The database keeps a page table that maps logical pages to disk blocks. A transaction works on a **copy** of the page table: each page it modifies is written to a **new** block, and its copy of the table points there, while the original (**shadow**) table still points to the old blocks. Commit is a single atomic switch of the pointer from the shadow table to the new table; abort simply discards the new table and blocks.

```text
 before commit:  current page table → new blocks for modified pages;  shadow page table → old blocks
 commit:         atomically replace the root pointer: current table becomes the database
 crash before:   the shadow table is still the database → nothing to undo or redo
```

Its weaknesses explain why mainstream databases use WAL instead: modified pages move on disk (data loses its physical clustering), every commit must write all modified pages and the page table (force), old blocks need garbage collection, and concurrent transactions are hard to support. Copy-on-write filesystems and some embedded databases (LMDB, for example) use the same idea.

### Buffer replacement in exam questions

Beyond LRU and Clock (Section 15.3), exams ask about **MRU** (most recently used), which evicts the page just used. It is the right choice for a repeated sequential scan of a file larger than the buffer: LRU would evict each page just before it is needed again, while MRU keeps the first part of the file cached. This is the same insight that led to ring buffers and midpoint insertion in real engines.

### Drill for this section

**C1. Deferred versus immediate database modification: what does recovery need in each?**
A strong answer must mention:
- Deferred: no database changes before commit; recovery redoes committed transactions only.
- Immediate: changes may reach the database before commit; recovery must undo uncommitted and redo committed transactions.
- The log must be written before the data in the immediate scheme (the WAL rule).
- Real engines are immediate (steal) with no-force, hence ARIES-style redo plus undo, or MVCC as in PostgreSQL.
*Common weak answer:* "Deferred is safer." It avoids undo but forces all of a transaction's changes to be held until commit.

**C2. Given a log ending in a crash, which transactions are redone and which undone?**
A strong answer must mention:
- Redo every transaction with a commit record; undo every transaction with a start but no commit.
- Undo uses old values, processing the log backwards; redo uses new values, forwards.
- With checkpoints, transactions committed before the checkpoint can be ignored.
*Common weak answer:* undoing every transaction after the checkpoint, including committed ones.

**C3. What is shadow paging, and why is it rarely used by databases?**
A strong answer must mention:
- Copy-on-write pages with an atomic switch of the page-table root at commit; no undo or redo needed.
- Costs: lost data clustering, force-at-commit writes, garbage collection, poor concurrency.
- WAL with in-place updates avoids these, which is why mainstream engines use it.
*Common weak answer:* "It is a kind of backup." It is an atomic-commit mechanism, not a backup.

---

## Chapter Summary

A database server is a set of client execution contexts and background workers around shared memory. **PostgreSQL forks one process per connection**; **MySQL runs one thread per connection** inside a single process. Either way, connections cost memory and only the CPU cores and disks do work, so connections should be few and pooled. Background workers write dirty pages, flush the log, take checkpoints and clean up old row versions, competing with clients for the same I/O.

Data lives in fixed-size **pages** (8 KB in PostgreSQL, 16 KB in InnoDB). A **slotted page** keeps a slot array at the front and tuples at the back, so rows keep a stable address (`(page, slot)`, PostgreSQL's TID) while they move inside the page. PostgreSQL stores tables as heaps and updates by writing **new tuple versions**, leaving dead ones for VACUUM; InnoDB stores rows in a clustered primary-key B+tree and updates **in place**, copying old values to the **undo log**. Large values go out of line (TOAST, overflow pages).

The **buffer pool** caches pages in memory. A page table maps page IDs to frames; frames carry dirty flags, pin counts and usage information; eviction uses clock sweep (PostgreSQL) or a midpoint LRU (InnoDB), with scan resistance. PostgreSQL also relies on the OS page cache (double buffering); InnoDB usually bypasses it with `O_DIRECT`.

**Write-ahead logging** makes commits durable cheaply: every change is described in the log first, a commit is acknowledged only after its commit record is flushed, and pages are written later, never before their log records. **Group commit** shares one `fsync` among many commits, and the storage device's flush latency becomes part of every write's latency. Full-page images (PostgreSQL) and the doublewrite buffer (InnoDB) protect against torn pages. The WAL is a **redo** log in both engines.

**Crash recovery** follows ARIES's logic for steal/no-force buffer management: analyse from the last checkpoint, **redo** every logged change missing from its page (idempotent thanks to page LSNs), then remove uncommitted effects. InnoDB rolls back with its **undo log**; PostgreSQL has no undo phase, because uncommitted versions are simply invisible once the transaction is marked aborted. **Checkpoints** bound the log to replay, trading normal-time I/O and WAL volume against recovery time.

Two storage philosophies compete. **B-trees** update pages in place: stable reads and range scans, random writes. **LSM-trees** append to a memtable and write immutable, sorted **SSTables**, merged by **compaction**, with **bloom filters** to skip files on reads: sequential writes and good compression, at the cost of compaction work, tombstones and higher read amplification for scans. **Row stores** suit OLTP; **column stores**, with compression and vectorised execution, suit analytics. **In-memory databases** such as Redis choose their durability: snapshots (losing minutes), logs flushed per second (losing about a second), or per-write flushes (losing nothing, costing latency), plus replication that is asynchronous by default.

Finally, **backups** are the only protection against lost storage, logical mistakes and attackers: logical dumps for portability, physical base backups with **WAL archiving** for **point-in-time recovery**, designed against an explicit **RPO** and **RTO**, kept off-site and immutable, monitored, and proven by timed restore drills.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Process / thread per connection | An independent execution context per client (process isolation in PostgreSQL) | More throughput from more connections |
| Slotted page | Stable row addresses despite movement inside the page | Cheap updates; space reuse without VACUUM / purge |
| Buffer pool | One consistent cached copy per page; pinned pages stay; no page written before its WAL | That a query's pages are cached; warm performance after restart |
| WAL with flush at commit | Acknowledged commits survive a crash; no half-applied transactions | Durability on storage that lies about flushes; durability with relaxed commit settings; survival of a lost disk |
| Group commit | Many commits share one flush | Lower latency for a single commit |
| Full-page writes / doublewrite | Recovery from torn pages | Low WAL volume right after checkpoints |
| Crash recovery | Exactly the durably committed transactions, no uncommitted effects | Protection from committed mistakes; fast recovery with rare checkpoints |
| Checkpoint | A bound on WAL to replay | Smooth I/O unless spread |
| B-tree storage | Stable point and range reads; in-place updates | Cheap random small writes |
| LSM storage | Sequential writes, high ingest, compression | Compaction keeping up; space freed promptly after deletes; cheap range scans |
| Bloom filter | No false negatives; few false positives at ~10 bits/key | Help for range scans |
| Column store | Reads only referenced columns; strong compression | Fast point lookups or single-row updates |
| Redis RDB / AOF | Durability up to the last snapshot / flush interval (or every write with `always`) | Acknowledged writes on a promoted async replica; bounded memory during `fork` |
| Physical backup + WAL archiving | Restore to any moment in the retained window | Data after the last archived segment; untested restores |
| Replica | Availability after a node failure | Protection from logical errors (they replicate) |

---

## Interview Drill

**Q1. How does a database store rows on disk?**
A strong answer must mention:
- Fixed-size pages (8 KB PostgreSQL, 16 KB InnoDB) as the unit of I/O and caching.
- Slotted pages: header, slot array, free space, tuples from the end; TIDs `(page, slot)` stay valid as tuples move.
- PostgreSQL heap with new versions per update and VACUUM; InnoDB clustered primary-key B+tree with in-place updates and undo.
- Large values out of line (TOAST, overflow pages).
*Common weak answer:* "Rows are written to a file one after another." Misses pages, slots, versions and why updates are not overwrites in PostgreSQL.

**Q2. Why does PostgreSQL use a process per connection, and what does it cost?**
A strong answer must mention:
- The postmaster forks a backend per connection; shared memory holds buffers, WAL buffers and locks.
- Costs: connection setup in milliseconds, several MB per backend, many processes to schedule; hence poolers for thousands of connections.
- Benefit: memory isolation between sessions.
- MySQL uses threads in one process; cheaper connections, no isolation; neither adds CPU.
*Common weak answer:* "Because PostgreSQL is old." The trade-off is isolation and simplicity versus per-connection cost.

**Q3. What is the buffer pool, and how does it evict pages?**
A strong answer must mention:
- Database-managed cache of pages; page table → frames; dirty flag, pin count, usage information.
- Eviction: PostgreSQL clock sweep with usage counts; InnoDB LRU with midpoint insertion; pinned pages never evicted; dirty victims written only after their WAL is flushed.
- Scan resistance (ring buffers, midpoint insertion).
- Interaction with the OS page cache: double buffering in PostgreSQL, `O_DIRECT` commonly in InnoDB.
*Common weak answer:* "It's an LRU cache." Pure LRU would let one scan flush the hot set, and it ignores pinning and the WAL rule.

**Q4. What is WAL, and why is it faster than writing pages at commit?**
A strong answer must mention:
- Log records describing changes, appended sequentially; LSNs; page LSNs.
- The WAL rule: log before data, commit record durable before acknowledging.
- One sequential append and one (shared) `fsync` per commit instead of random page writes; pages written later in the background.
- Group commit; torn-page protection; WAL is redo in both PostgreSQL and InnoDB.
*Common weak answer:* "WAL is a backup of the data." It is a redo log for crash recovery (and the source for replication and PITR).

**Q5. Walk through crash recovery.**
A strong answer must mention:
- Start from the last checkpoint's redo point; analysis finds committed and loser transactions.
- Redo every logged change whose LSN is newer than the page's LSN (idempotent).
- InnoDB then rolls back losers using undo; PostgreSQL has no undo phase, since uncommitted versions are invisible once marked aborted.
- Steal/no-force explains why both redo and some form of undo are needed.
*Common weak answer:* "It undoes everything after the last checkpoint." Committed work after the checkpoint must be redone, not undone.

**Q6. What does a checkpoint do? What if checkpoints are too frequent or too rare?**
A strong answer must mention:
- Writes all pages dirty as of a redo point, syncs them, records the checkpoint, allowing older WAL to be recycled.
- Too frequent: more I/O, more full-page images, periodic spikes. Too rare: long recovery, more WAL kept.
- Spreading checkpoints; time-driven rather than WAL-size-driven; InnoDB's fuzzy checkpointing and redo capacity.
- Numbers: WAL rate × interval ≈ WAL to replay.
*Common weak answer:* "A checkpoint saves the database." It bounds recovery; durability already came from the WAL.

**Q7. How do PostgreSQL and InnoDB roll back a transaction?**
A strong answer must mention:
- PostgreSQL: mark the transaction aborted in `pg_xact`; its tuple versions become invisible; VACUUM removes them later; no undo log.
- InnoDB: apply undo log records to restore the old values written over in place.
- In both, the WAL/redo log is for redo; rollback does not read it backwards.
- Consequences: PostgreSQL bloat and VACUUM; InnoDB undo growth and purge.
*Common weak answer:* "Both undo using the WAL." The WAL is redo-only in both; the mechanisms differ.

**Q8. B-tree versus LSM-tree storage: how do they differ, and when would you choose each?**
A strong answer must mention:
- B-tree: in-place page updates with WAL; stable point and range reads; random writes.
- LSM: memtable → SSTables → compaction; sequential writes; bloom filters; tombstones.
- Amplification: LSM lower random-write cost but compaction write amplification and higher scan cost; B-tree page-sized writes for small changes.
- Fit: OLTP with ranges and secondary indexes → B-tree; high-ingest, write-heavy key–value → LSM.
*Common weak answer:* "LSM is faster." Faster at writes, with compaction cost and slower scans.

**Q9. What are write, read and space amplification? What is a bloom filter for?**
A strong answer must mention:
- Definitions: bytes written per byte of data; reads per logical read; bytes on disk per byte of live data.
- The trade-off between them (RUM conjecture).
- Bloom filter: bit array plus k hashes; "definitely not" or "maybe"; ~10 bits per key gives ~1% false positives; used per SSTable to skip files on point reads; useless for range scans.
*Common weak answer:* "A bloom filter tells you whether a key exists." It can only say "definitely not" for sure.

**Q10. Row stores versus column stores: why is a column store faster for analytics?**
A strong answer must mention:
- Reads only the referenced columns.
- Compresses well (dictionary, run-length, bit-packing, delta).
- Vectorised execution over contiguous column arrays (cache-friendly, SIMD).
- Expensive single-row writes and point lookups; batch loading with delta stores; OLTP stays in row stores.
*Common weak answer:* "Column stores have better indexes." The gain comes from layout, compression and execution, not indexes.

**Q11. How can an in-memory database be durable?**
A strong answer must mention:
- Snapshots (Redis RDB via `fork` and copy-on-write): loss window since the last snapshot.
- Operation logs (AOF) with flush per write, per second or by the OS: loss window from zero to tens of seconds.
- Replication helps availability but is asynchronous by default.
- Costs: `fork` pauses and memory growth; log rewrites.
*Common weak answer:* "It writes to disk sometimes." The loss window must be stated and chosen.

**Q12. Logical versus physical backups; how does point-in-time recovery work; what are RPO and RTO?**
A strong answer must mention:
- Logical: portable, selective, slow to restore; physical: fast, same version, supports PITR.
- PITR: base backup + continuously archived WAL; replay to a target time just before the mistake.
- RPO: tolerable data loss in time; RTO: tolerable restoration time; numbers drive the design.
- Off-site, immutable copies and timed restore drills.
*Common weak answer:* "Take a nightly `pg_dump`." Gives a 24-hour RPO and an hours-long RTO for large databases.

**Q13. Why is a replica not a backup?**
A strong answer must mention:
- A replica applies every committed change, including mistakes and malicious deletions, within moments.
- It protects against node loss, not logical errors or ransomware with shared credentials.
- Delayed replicas and PITR cover logical errors.
*Common weak answer:* "Replicas can fail too." True, but the decisive reason is that errors replicate.

**Q14. Why does commit latency depend on the storage device?**
A strong answer must mention:
- Commit waits for the WAL flush (`fsync`), whose latency ranges from microseconds (enterprise SSD with power-loss protection) to milliseconds (consumer SSD, network storage, HDD).
- Without group commit, commits per second ≈ 1 / flush latency for serial commits.
- Relaxed commit settings trade the last moments of acknowledged commits for latency.
- Storage that lies about flushes breaks durability.
*Common weak answer:* "Commits write all the data, so big transactions are slow." Commit flushes only the log.

**Q15. What is a torn page, and how do databases protect against it?**
A strong answer must mention:
- A page larger than the device's atomic write unit can be half-written at power loss.
- Redo records describe deltas and cannot fix a corrupted base page.
- PostgreSQL writes a full-page image on the first change after each checkpoint; InnoDB writes through a doublewrite buffer.
- Cost: extra WAL or write volume.
*Common weak answer:* "Checksums fix it." Checksums detect corruption; they cannot restore the page.

**Q16. Why does an update in PostgreSQL create a new row version, and what does that cost?**
A strong answer must mention:
- MVCC: readers may still need the old version, so updates write new tuples (Chapter 17).
- HOT updates avoid new index entries when no indexed column changes and the page has room.
- Cost: dead tuples, VACUUM work, bloat if VACUUM is blocked by long transactions.
- InnoDB instead keeps old versions in undo, cleaned by purge.
*Common weak answer:* "It's a design mistake." It is the design that lets readers never block writers.

---

## Scenario Exercises

1. A team moves its PostgreSQL database from a server with enterprise NVMe drives to cloud network block storage with the same capacity. Read queries are unchanged, but write-heavy endpoints are three times slower. Explain the mechanism, give the evidence you would collect, and three options.
2. Your application opens 2,000 connections to PostgreSQL (20 instances × 100-connection pools), and the server's memory alarms fire during traffic peaks although `shared_buffers` is only 25% of RAM. Explain why, and what you would change.
3. A product manager proposes setting PostgreSQL `synchronous_commit = off` globally to halve API latency. The database stores orders, payments and page-view analytics. Explain what would be risked, and propose a narrower change.
4. A metrics service writes 200,000 data points per second and almost never updates or deletes them; queries read time ranges per series. It runs on PostgreSQL and is struggling with write I/O. Argue for or against an LSM-based store, naming the amplification trade-offs and any new operational work.
5. A finance analyst runs a two-year revenue report on the production database every hour. Each run takes 8 minutes, and checkout latency doubles while it runs. Explain the storage-level mechanism and propose two architectures.
6. Design a backup strategy for a 2 TB PostgreSQL database with an RPO of 5 minutes and an RTO of 1 hour, including how you would prove it works.
7. Redis is used both as a cache and as the only store of users' shopping carts, with snapshot persistence every 15 minutes. A node restarts. What do users experience, and what would you change?

---

## Diagnosis Lab

**Lab 1: Latency spikes every five minutes**
- *Symptom:* Write latency at p99 jumps from 15 ms to 400 ms for about 30 seconds, roughly every five minutes, on a PostgreSQL server. Reads are mostly unaffected.
- *Hypotheses:* (a) checkpoints; (b) autovacuum; (c) a scheduled job.
- *Evidence:* The server log shows `checkpoint starting: time` every five minutes, each completing with `write=27.4 s` and about 300,000 buffers written. `checkpoint_completion_target` was set to 0.1 in an old configuration, so each checkpoint writes its pages in the first tenth of the interval. `iostat -x` shows write `await` rising to 40 ms during those 30 seconds. The WAL rate also jumps right after each checkpoint (full-page images).
- *Root cause:* (a): checkpoint writes compressed into a short burst saturate the device, and commits wait behind them.
- *Fix:* Spread checkpoint writes over most of the interval (`checkpoint_completion_target` near 0.9), consider a longer `checkpoint_timeout` with a matching `max_wal_size` (accepting longer recovery), and verify the device's write capacity.
- *Prevention:* Dashboards of checkpoint duration, buffers written and `await`; configuration reviews against current defaults.

**Lab 2: The database stops accepting writes**
- *Symptom:* All writes fail; then the PostgreSQL server shuts down. The log ends with `PANIC:  could not write to file "pg_wal/xlogtemp.31877": No space left on device`.
- *Hypotheses:* (a) data growth filled the disk; (b) WAL accumulated; (c) logs or temporary files filled it.
- *Evidence:* `df -h` shows the volume at 100%. `du -sh pg_wal` shows 380 GB of WAL, against a normal few GB. `pg_stat_archiver` shows `failed_count` climbing for six days: the archive command's credentials to object storage had expired. PostgreSQL keeps every segment that has not been archived successfully.
- *Root cause:* (b), caused by failing WAL archiving. Without space to write WAL, PostgreSQL cannot guarantee durability and stops (a PANIC rather than accepting writes it cannot log).
- *Fix:* Free space safely (expand the volume; never delete WAL files by hand), fix the archive credentials, let the archiver catch up, then restart. Take a new base backup, since PITR across the gap is impossible.
- *Prevention:* Alert on `failed_count` and on WAL directory size; WAL on a volume with headroom; disk-usage alerts well before 100%.

**Lab 3: WAL growth with archiving healthy**
- *Symptom:* The WAL directory grows by about 40 GB per day; archiving reports no failures; disk-usage alerts are firing.
- *Hypotheses:* (a) higher write volume; (b) a replication slot retaining WAL; (c) a very long checkpoint interval.
- *Evidence:* WAL generation rate is unchanged. `pg_replication_slots` shows a logical slot `analytics_cdc` inactive for nine days, retaining 360 GB. The change-data-capture consumer was decommissioned without dropping its slot.
- *Root cause:* (b). A replication slot guarantees that WAL is kept until its consumer receives it, even if the consumer never returns.
- *Fix:* Confirm the consumer is truly gone, then drop the slot (`SELECT pg_drop_replication_slot('analytics_cdc')`); the next checkpoints recycle the WAL.
- *Prevention:* Alert on inactive slots and retained WAL per slot; where supported, cap retention per slot (PostgreSQL's `max_slot_wal_keep_size`), accepting that a lagging consumer then has to resynchronise; include slot cleanup in decommissioning checklists.

**Lab 4: Commits slow after a storage migration**
- *Symptom:* After moving MySQL to new virtual machines, single-row `INSERT` statements take 6–10 ms instead of under 1 ms; CPU is idle; queries that only read are as fast as before.
- *Hypotheses:* (a) network latency to the database; (b) slow flushes on the new storage; (c) a lock or contention problem.
- *Evidence:* The latency is identical for a client on the database host itself, which rules out (a). `innodb_flush_log_at_trx_commit = 1` and `sync_binlog = 1`, so every commit flushes the redo log and the binary log. A synchronous write test on the new volume (`fio` with `fsync` after each write) measures about 4 ms per flush; the old servers had local SSDs with power-loss protection at about 0.1 ms. Performance-schema waits show time in log-flush waits, not locks.
- *Root cause:* (b): commit latency is bounded by flush latency, now two flushes of about 4 ms each.
- *Fix:* Move the redo and binary logs to storage with low flush latency (a provisioned-IOPS or local NVMe volume, where available), batch writes in the application where possible, and rely on group commit under concurrency. Do not relax the flush settings for financial data; consider it only for data whose loss on a crash is acceptable.
- *Prevention:* Measure `fsync` latency (`fio`, `pg_test_fsync`) as part of any storage selection, not only throughput and IOPS.

---

## Solution Sketches

1. Commits wait for the WAL flush; network block storage typically has much higher flush latency than local NVMe with power-loss protection, so every write transaction's latency rises while reads (served from the buffer pool) do not. Evidence: wait events showing `WalSync`; `pg_test_fsync` on the old and new volumes; write `await` in `iostat`. Options: a faster or provisioned-latency volume for `pg_wal`; batching writes per transaction and relying on group commit; relaxed `synchronous_commit` only for data whose loss is acceptable.
2. Each backend process has its own memory, and each sort or hash in a query can use up to `work_mem`, several per query; 2,000 backends × their private memory at peak can exceed what is left after shared buffers. Change: put a pooler (Chapter 18) in front so the database sees a few dozen active connections, size application pools from the database's capacity (Chapter 11, Section 11.5), and set per-operation memory with the concurrency in mind.
3. With `synchronous_commit = off`, the database acknowledges commits before their WAL is flushed; a crash loses the last fraction of a second of acknowledged transactions (no corruption). For orders and payments that means customers told "paid" whose payment record disappears. Narrower: keep `on` globally and set `off` only for the analytics transactions (`SET LOCAL synchronous_commit = off` in those transactions), or batch analytics writes.
4. For: the workload is append-heavy with new keys and range reads per series, which suits LSM storage (sequential writes, strong compression of time-ordered data, no in-place page updates); a time-series engine built on LSM ideas would also partition by time and drop whole files for retention. Trade-offs: compaction I/O and write amplification, read amplification for scans across many SSTables (mitigated by time-partitioned layouts), different backup and operational tooling, and loss of SQL joins and transactions with the rest of the data. Alternatives within PostgreSQL: batching inserts, time-partitioned tables, a columnar or time-series extension.
5. The report scans years of rows, reading many pages from disk and pushing the hot pages of the checkout path out of the buffer pool, while competing for I/O; checkout queries then miss the cache and wait for the device. Architectures: run reports on a read replica (Chapter 22) so the primary's cache and I/O are untouched; or stream changes into a column store (Chapter 25) and run analytics there, where the report reads only a few compressed columns.
6. Continuous WAL archiving with streaming (`pg_receivewal`) or `archive_timeout` well under 5 minutes gives an RPO of a few minutes or less; daily physical base backups (2 TB at, say, 1 GB/s restore ≈ 35 minutes) plus at most one day of WAL replay must fit in an hour, so measure the replay rate and take base backups more often if needed, or keep a warm standby for node loss. Store backups in another account and region with immutability; encrypt them. Prove it with monthly timed restores to a scratch environment, PITR to a chosen timestamp, data checks, and alerts on archive failures.
7. A 15-minute snapshot interval means a restart loses up to 15 minutes of cart changes: users see items disappear. As a cache that is acceptable; as the only store of carts it is not. Either move carts to the primary database (with Redis as a cache in front), or enable AOF with `appendfsync everysec` (losing about a second) plus replicas and backups, and treat that Redis as a system of record with monitoring of persistence failures.

---

**END OF CHAPTER 15**
