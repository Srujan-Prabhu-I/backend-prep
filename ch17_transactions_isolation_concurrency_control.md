# Chapter 17: Transactions, Isolation, and Concurrency Control

## Where We Are

Chapter 5 taught concurrency inside one process: threads that share memory, the races they cause (lost updates, check-then-act), and the tools that prevent them (locks, atomic compare-and-swap, condition variables). Chapter 11 moved the same problems to servers with many instances and showed that an in-process lock protects nothing across them; the fix was to push the decision into the database with an atomic conditional `UPDATE`, a constraint or a row lock. Chapter 14 defined what queries mean, Chapter 15 showed how a database makes a commit durable with the write-ahead log, and Chapter 16 showed how it finds rows quickly.

This chapter is where those threads meet. A database serves thousands of concurrent requests, each running a sequence of reads and writes, often against the same rows. **Transactions** are the database's promise about how those sequences behave: all or nothing, and, to a degree you choose, as if each ran alone. Getting this right is the difference between a shop that never oversells and one that sells the last keyboard three times; between an account balance that is always correct and one that is occasionally overdrawn by a race nobody can reproduce.

The chapter defines transactions and ACID, then shows precisely which **anomalies** concurrent transactions can produce, with a timeline for each. It explains the **isolation levels** as "which anomalies can still happen", engine by engine, because PostgreSQL and MySQL's InnoDB implement the same names differently. Then it opens the two mechanisms behind them, **locking** and **multi-version concurrency control** (MVCC), followed by serialisable snapshot isolation, explicit locks, optimistic and pessimistic patterns, deadlocks, atomic in-database operations, and the damage long transactions do. A worked case solves the same oversell problem four ways and compares them.

**After this chapter you can:**
- define a transaction and use commit, rollback, autocommit and savepoints correctly;
- explain each ACID property precisely, with the mechanism that implements it in PostgreSQL and InnoDB;
- draw a two-transaction timeline for dirty reads, non-repeatable reads, phantoms, lost updates, read skew and write skew;
- state which anomalies each isolation level allows in the SQL standard, in PostgreSQL and in InnoDB, and why;
- explain two-phase locking, lock modes, intention locks and lock granularity;
- explain MVCC in PostgreSQL (heap versions, `xmin`/`xmax`, VACUUM, wraparound) and InnoDB (undo chains, read views, purge), and work through a visibility check;
- explain serialisable snapshot isolation and handle serialisation failures with retries;
- use `SELECT … FOR UPDATE`, `SKIP LOCKED` and advisory locks, and choose between optimistic and pessimistic concurrency;
- explain, detect and prevent database deadlocks;
- solve oversell and double-spend problems four ways and choose between them;
- diagnose lock waits, deadlocks, serialisation failures and idle-in-transaction sessions.

**Interview questions this chapter prepares you for:**
1. What is a transaction? What do commit, rollback, autocommit and savepoints do?
2. Explain ACID precisely. What implements each property?
3. What are a dirty read, a non-repeatable read and a phantom?
4. What are a lost update, read skew and write skew? Give an example of write skew.
5. What are the isolation levels, and what does each prevent? What do PostgreSQL and MySQL actually do at each?
6. What is two-phase locking? What are shared, exclusive and intention locks?
7. How does MVCC work? How do PostgreSQL and InnoDB differ?
8. What is VACUUM, and what is transaction ID wraparound?
9. What is serialisable snapshot isolation, and why must applications retry?
10. What do `SELECT … FOR UPDATE` and `SKIP LOCKED` do? How would you build a job queue in a database?
11. Optimistic versus pessimistic locking: how do they work, and when would you use each?
12. How do database deadlocks happen, and how are they detected and prevented?
13. How do you prevent two customers from buying the last item?
14. Why are long-running transactions dangerous?

**Scope.** Here: transactions, ACID, anomalies, isolation levels per engine, locking, MVCC, SSI, explicit locks, optimistic and pessimistic concurrency, deadlocks, atomic operations, long transactions, and a placement-round section on schedule theory and textbook protocols (Section 17.15). Recalled, not re-taught: races, locks and deadlock theory in threads (Chapter 5, Sections 5.2, 5.5 and 5.8), the WAL and crash recovery (Chapter 15, Sections 15.4–15.5), page layout and row versions (Chapter 15, Section 15.2), constraints (Chapter 14, Section 14.3), conditional requests and idempotency keys (Chapter 13, Sections 13.5 and 13.7). Elsewhere: transaction boundaries in application code, ORMs and connection pools are Chapter 18; transactions across services (two-phase commit) are Chapter 23; sagas and the outbox are Chapter 25; consistency in replicated systems is Chapter 22.

---

## 17.1 What a Transaction Is

### Definitions

A **transaction** is a sequence of database operations (reads and writes) that the database treats as a single unit of work: either **all** of its changes take effect, or **none** of them do, and, depending on the isolation level, it is shielded from the effects of other transactions running at the same time.

- **`COMMIT`** ends a transaction successfully: its changes become permanent and visible to others.
- **`ROLLBACK`** (abort) ends it unsuccessfully: all its changes are discarded, as if it never ran.
- **Autocommit** is the mode in which every statement that is not inside an explicit transaction runs as its own one-statement transaction, committed immediately.
- A **savepoint** is a named point inside a transaction to which the transaction can roll back partially, undoing only the work done after it.

### Why it exists

Business operations are rarely one write. Placing an order inserts the order, inserts its lines, decrements stock for each product and records a payment intent. If the server crashes after decrementing stock but before inserting the order, stock is lost; if two customers run the same sequence at once, they can both see "one left" and both buy it. Without transactions, every application would have to detect and repair half-finished work after crashes, and coordinate concurrent users by hand: the file-based problems of Chapter 14 (Section 14.1). A transaction moves both problems into the database, which solves them once with the write-ahead log (Chapter 15) and concurrency control (this chapter).

### Mechanism: the lifecycle of a transaction

```sql
BEGIN;                                                         -- start (START TRANSACTION in MySQL)
UPDATE products SET stock = stock - 1 WHERE id = 17;           -- part of the unit
INSERT INTO orders (id, customer_id, status) VALUES (9001, 81, 'PLACED');
INSERT INTO order_lines (order_id, product_id, quantity, unit_cents) VALUES (9001, 17, 1, 1999);
COMMIT;                                                        -- all three take effect together
```

```text
          BEGIN                                   COMMIT
 ───────────┼──── stmt 1 ── stmt 2 ── stmt 3 ──────┼──────────► changes durable and visible
            │                                     │
            │       any statement fails, the client sends ROLLBACK,
            │       or the server crashes before COMMIT
            └────────────────► ROLLBACK: no change from stmts 1–3 survives
```

The database makes this true even across a crash: a transaction is committed exactly when its commit record is durable in the log (Chapter 15, Section 15.4), and recovery removes or hides every change of a transaction without one (Chapter 15, Section 15.5).

### Mechanism: autocommit, and the "transaction" you did not start

Most drivers and database consoles start in **autocommit** mode: each statement is its own transaction. Two consequences surprise people:

```sql
-- autocommit ON (the default in most drivers)
UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- committed immediately
-- the application crashes here
UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- never runs: 100 has vanished
```

The two updates were two transactions, so the first one's effect survives on its own. Wrapping them in `BEGIN … COMMIT` makes the transfer atomic. The opposite mistake is also common: a framework or driver silently turns autocommit **off** and opens a transaction on the first statement, which then stays open, holding locks and blocking cleanup, until something commits (the "idle in transaction" sessions of Section 17.12). Chapter 18 covers how drivers and frameworks manage transaction boundaries.

### Mechanism: errors inside a transaction, and savepoints

What happens when one statement in a transaction fails differs between engines:

- In **PostgreSQL**, any error puts the whole transaction into an **aborted** state. Every later statement fails with `current transaction is aborted, commands ignored until end of transaction block`, and the only way forward is `ROLLBACK`.
- In **MySQL**, most statement errors roll back only that statement; the transaction stays open and the earlier statements remain part of it (some errors, such as deadlocks, roll back the whole transaction).

**Savepoints** give explicit control in both:

```sql
BEGIN;
INSERT INTO orders (id, customer_id, status) VALUES (9002, 81, 'PLACED');
SAVEPOINT before_coupon;
INSERT INTO coupon_redemptions (order_id, code) VALUES (9002, 'AUTUMN');   -- fails: coupon already used
ROLLBACK TO SAVEPOINT before_coupon;            -- undo only the failed part; the order insert survives
UPDATE orders SET note = 'coupon rejected' WHERE id = 9002;
COMMIT;                                          -- the order is committed without the coupon
```

A savepoint is not a nested commit: nothing after `BEGIN` is durable or visible to others until the outer `COMMIT`. Some frameworks implement "nested transactions" with savepoints, which is exactly what they are.

### Guarantees and non-guarantees

A transaction guarantees that its writes are applied all together or not at all, and that once `COMMIT` returns successfully they are durable (with durable commit settings, Chapter 15, Section 15.4). It does not guarantee anything about effects outside the database (an email sent inside a transaction that later rolls back is still sent; Chapter 25 handles this with the outbox), that the client learns the outcome (a connection lost during `COMMIT` leaves it unknown, Chapter 7, Section 7.15), or, by itself, full protection from concurrent transactions (that depends on the isolation level, Section 17.4).

### Failure and diagnosis

- **Half-applied operations.** A transfer debits but never credits. Evidence: related statements committed separately in the database log; autocommit on. Fix: one explicit transaction per business operation.
- **"current transaction is aborted" errors** (PostgreSQL). The application catches an error and keeps issuing statements in the same transaction. Fix: roll back (or use a savepoint deliberately) after any error.
- **Commit outcome unknown.** The connection drops after `COMMIT` is sent. Evidence: the client logs a network error at commit. Fix: idempotency keys and a status check (Chapter 13, Section 13.5), never a blind retry.

### Backend connection

Make each business operation one transaction, explicitly, and keep everything that is not a database write (HTTP calls, emails) outside it (Chapter 11, Section 11.5; Chapter 12, Section 12.10). Know your driver's autocommit default, and treat an error inside a transaction as the end of that transaction unless you deliberately used a savepoint.

---

## 17.2 ACID, Property by Property

### Definitions

**ACID** names four properties of transactions:

- **Atomicity**: a transaction's changes are applied all or nothing. ("Atomic" here means indivisible with respect to **failures**, not, as in Chapter 5, with respect to concurrent observers.)
- **Consistency**: a transaction takes the database from one state that satisfies its invariants to another that also satisfies them. Invariants are rules such as "stock is never negative" or "debits equal credits".
- **Isolation**: concurrent transactions do not interfere with each other; in its strongest form, **serialisability**, the result is the same as if they had run one at a time in some order. (This is the database meaning of isolation; Chapter 4, Section 4.10 used the word for separating containers, a different idea.)
- **Durability**: once a transaction commits, its changes survive crashes and power loss.

**Concurrency control** is the set of mechanisms a database uses to provide isolation: locks (Section 17.5), multi-version concurrency control (Section 17.6), or both.

### Why it exists

"Transactions are ACID" is recited in every interview, but each letter is a separate guarantee with a separate mechanism, a separate cost, and a separate way to be weakened. Knowing which mechanism implements which letter is what lets you answer the questions that matter: is my data safe if the server crashes now, can two requests interfere, and which rules is the database actually enforcing for me?

### Mechanism: what implements each property

| Property | Implemented by | PostgreSQL | InnoDB | Weakened by |
|---|---|---|---|---|
| **Atomicity** | a way to remove or hide an unfinished transaction's changes | new row versions marked with the transaction ID; aborting marks the transaction aborted in `pg_xact` and its versions become invisible (Chapter 15, Section 15.5) | in-place changes with old values saved in the **undo log**; rollback applies undo | autocommit splitting one operation into several transactions |
| **Consistency** | constraints in the database **plus** correct application logic inside transactions | `NOT NULL`, `UNIQUE`, `CHECK`, foreign keys, exclusion constraints (Chapter 14, Section 14.3) | the same, except exclusion constraints | rules that exist only in application code, combined with weak isolation (write skew, Section 17.3) |
| **Isolation** | concurrency control | MVCC snapshots, row locks, serialisable snapshot isolation | MVCC read views, two-phase locking with record, gap and next-key locks | choosing a weaker isolation level (Section 17.4) |
| **Durability** | the write-ahead (redo) log, flushed at commit | WAL + `fsync` at commit (Chapter 15, Section 15.4) | redo log + flush at commit | relaxed commit settings; storage that lies about flushes; losing the disk (needs replicas and backups) |

Three points that interviewers probe:

1. **Consistency is partly the application's job.** The database guarantees the invariants it has been told about, as constraints. An invariant that lives only in code ("at least one doctor must be on call") is preserved only if every transaction's logic is correct **and** isolation is strong enough that concurrent transactions cannot jointly break it (Section 17.3, write skew). The "C" is the odd one out: atomicity, isolation and durability are mechanisms that help the application keep its own invariants.
2. **Atomicity is not isolation.** An atomic transaction can still read data that another transaction changes halfway through its work; whether it can depends on isolation.
3. **ACID consistency is not CAP consistency.** CAP's "consistency" (Chapter 22) means linearisability in a replicated system, an unrelated idea that shares the word.

### Concrete: the same transfer, failing each letter

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```text
 without atomicity:   crash between the updates → 100 debited, never credited
 without consistency: no CHECK (balance >= 0) and no check in code → account 1 goes to −40
 without isolation:   a concurrent report sums all balances between the two updates → total is 100 short
 without durability:  COMMIT returns, power fails, data was only in memory → the transfer is gone
```

### Guarantees and non-guarantees

ACID guarantees, for each transaction: all-or-nothing application, durability after commit, preservation of declared constraints, and the isolation promised by the chosen level. ACID does not guarantee that undeclared invariants hold, that the default isolation level prevents every anomaly (it does not, in PostgreSQL or MySQL), that anything outside the database is part of the transaction, or anything about multiple databases or services (Chapters 23 and 25).

### Failure and diagnosis

- **An invariant broken despite "using transactions".** Evidence: data violating a rule that only application code enforces, created by two concurrent transactions. Cause: weak isolation plus an undeclared invariant (Section 17.3). Fix: declare it as a constraint if possible; otherwise stronger isolation or explicit locks (Sections 17.7–17.8).
- **Lost acknowledged commits after a crash.** Evidence: durability settings relaxed; storage without power-loss protection (Chapter 15, Section 15.4).

### Backend connection

When someone says "the database is ACID", ask which isolation level, which constraints and which commit setting. Declare every invariant you can as a constraint, so that consistency does not depend on every code path being perfect, and choose isolation knowing which anomalies remain.

---

## 17.3 Concurrency Anomalies

### Definitions

An **anomaly** (phenomenon) is an outcome of concurrent transactions that could not happen if they ran one after another. The standard names, from the SQL standard and from the 1995 paper "A Critique of ANSI SQL Isolation Levels" by Berenson and colleagues:

- **Dirty write**: a transaction overwrites a value written by another transaction that has not yet committed.
- **Dirty read**: a transaction reads a value written by another transaction that has not yet committed.
- **Non-repeatable read** (fuzzy read): a transaction reads the same row twice and gets different values, because another transaction committed a change in between.
- **Phantom**: a transaction runs the same **query** twice and gets a different **set** of rows, because another transaction committed an insert or delete that matches the query's condition.
- **Lost update**: two transactions read the same value, each computes a new value from it, and both write; one write overwrites the other.
- **Read skew**: a transaction reads two related values at different moments and sees an inconsistent combination, because another transaction changed both in between.
- **Write skew**: two transactions read an overlapping set of data, each makes a decision based on it, and each writes a **different** row; together they break an invariant that each alone preserved.

### Why it matters

Isolation levels (Section 17.4) are defined by which of these anomalies they allow. Memorising level names is useless without knowing what each anomaly looks like in a real system. Each anomaly below is shown as a timeline of two transactions, T1 and T2, with time flowing downward.

### Concrete: dirty write and dirty read

```text
 DIRTY WRITE                                       DIRTY READ
 accounts: id 1 balance 500                        accounts: id 1 balance 500
 T1                        T2                      T1                          T2
 UPDATE bal=400 (id 1)                             UPDATE bal=400 (id 1)
                           UPDATE bal=300 (id 1)                               SELECT bal → 400   ← uncommitted
 ROLLBACK                                          ROLLBACK
                           COMMIT                                              (T2 acted on 400, a value
 → which value is restored? T1's rollback would                                 that never existed)
   erase T2's committed write, or T2's write
   survives over T1's undone one: undefined.
   Every isolation level forbids dirty writes.
```

Dirty reads let a transaction act on data that may be rolled back: a report showing a payment that is later cancelled, a decision based on stock that was never decremented.

### Concrete: non-repeatable read and phantom

```text
 NON-REPEATABLE READ                               PHANTOM
 T1                          T2                    T1                                T2
 SELECT price (id 17) → 1999                       SELECT count(*) FROM orders
                             UPDATE price=2499       WHERE status='PENDING' → 3
                             (id 17)                                                  INSERT order
                             COMMIT                                                   (status 'PENDING')
 SELECT price (id 17) → 2499                                                          COMMIT
 ← same row, different value                       SELECT * FROM orders
                                                     WHERE status='PENDING' → 4 rows
                                                   ← same query, different set of rows
```

The non-repeatable read concerns **one existing row** changing; the phantom concerns **which rows** match a condition. The distinction matters because preventing phantoms requires protecting rows that do not exist yet, which locking implementations do with range (gap) locks and snapshot implementations do by reading from a fixed snapshot.

### Concrete: lost update

```text
 products: id 17 stock 5. Each request adds stock it received, using read-modify-write in application code.
 T1                                             T2
 SELECT stock (id 17) → 5
                                                SELECT stock (id 17) → 5
 app computes 5 + 10 = 15
 UPDATE stock = 15 (id 17)
 COMMIT
                                                app computes 5 + 3 = 8
                                                UPDATE stock = 8 (id 17)
                                                COMMIT
 final stock = 8: T1's 10 units are lost (correct: 18)
```

This is Chapter 5's lost update (Section 5.2) with transactions instead of threads. It happens whenever the new value is computed **outside** the database from a value read earlier, unless the isolation level detects it or the code locks the row. `UPDATE products SET stock = stock + 10` computes inside the database and does not suffer it (Section 17.11).

### Concrete: read skew

```text
 accounts: id 1 balance 500, id 2 balance 500 (total 1000). T2 transfers 100 from 1 to 2.
 T1 (report)                                    T2 (transfer)
 SELECT balance (id 1) → 500
                                                UPDATE id 1: 500 → 400
                                                UPDATE id 2: 500 → 600
                                                COMMIT
 SELECT balance (id 2) → 600
 report: total = 500 + 600 = 1100   ← money appeared from nowhere
```

Each read returned committed data; neither value was "dirty". But the two values come from different moments, before and after the transfer, so their combination never existed. Read skew breaks backups taken by reading tables one after another, integrity checks, and any report that adds up related rows.

### Concrete: write skew, the on-call doctors

A hospital rule: **at least one doctor must be on call** for each shift. Alice and Bob are both on call and both feel unwell; each, at the same moment, asks the system to take them off call. The application checks the rule before writing:

```sql
-- what each request runs
BEGIN;
SELECT count(*) FROM on_call WHERE shift = 'night' AND on_duty;    -- "is anyone else on call?"
-- if count >= 2: allowed to leave
UPDATE on_call SET on_duty = false WHERE shift = 'night' AND doctor = $me;
COMMIT;
```

```text
 on_call: (night, Alice, true), (night, Bob, true)
 T1 (Alice)                                     T2 (Bob)
 SELECT count(on duty) → 2   ✔ OK to leave
                                                SELECT count(on duty) → 2   ✔ OK to leave
 UPDATE Alice → off duty
                                                UPDATE Bob → off duty
 COMMIT                                         COMMIT
 result: nobody is on call. Each transaction kept the rule on its own; together they broke it.
```

Write skew is subtle because the two transactions write **different rows**. No row is overwritten, so nothing that detects conflicting writes to the same row (row locks, the lost-update check of snapshot isolation) notices anything. The conflict is between what one transaction **read** and what the other **wrote**. The same pattern appears in backend systems as: two bookings of the same room for overlapping times (each checks "no overlapping booking" and inserts a different row), two users claiming the same username in a system without a unique constraint, a balance spread over several rows (two withdrawals each check the total and each insert a debit), and the double-spend of Section 17.13.

### Guarantees and non-guarantees

The anomaly catalogue guarantees a shared vocabulary for what can go wrong. It is not complete (researchers have defined others), and the absence of the named anomalies is not by itself a proof of serialisability, except where the level is defined as serialisable.

### Failure and diagnosis

Anomalies leave no error; they leave wrong data. Typical evidence: totals that do not reconcile (read skew in reports, lost updates in counters), invariants broken by pairs of transactions committed within milliseconds of each other (write skew), values that "revert" (lost update). Reproducing them requires concurrency: two sessions stepping through the timeline by hand, or a load test with assertions on the invariant (Chapter 5, Section 5.14).

### Backend connection

For every read-then-write in your transactions, ask which anomaly would break it: is a value computed in application code from an earlier read (lost update)? Does a decision depend on rows the transaction does not write (write skew)? Does a report combine several reads (read skew)? The answer decides the isolation level or locking you need.

---

## 17.4 Isolation Levels: the Standard and the Engines

### Definitions

An **isolation level** is a setting, chosen per transaction or per session, that determines which anomalies the database allows in exchange for concurrency and performance. The SQL standard defines four, from weakest to strongest: **Read Uncommitted**, **Read Committed**, **Repeatable Read** and **Serializable**. Only Serializable promises **serialisability**: every execution is equivalent to some one-at-a-time order of the transactions.

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;                     -- PostgreSQL
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ; START TRANSACTION;   -- MySQL
```

The default is **Read Committed** in PostgreSQL and **Repeatable Read** in MySQL's InnoDB.

### Why it exists

Full serialisability costs concurrency: some transactions must wait or be aborted. Many workloads tolerate some anomalies (a dashboard does not need a perfectly consistent snapshot), so databases offer weaker levels that allow more concurrency. The trap is that the standard defines the levels by three anomalies only (dirty read, non-repeatable read, phantom), in terms that fit lock-based implementations. Modern engines implement them with snapshots and locks in different combinations, so the same name means different guarantees in different databases.

### Mechanism: how each level is implemented

| Level | SQL standard says | PostgreSQL implements | InnoDB implements |
|---|---|---|---|
| Read Uncommitted | dirty reads allowed | identical to Read Committed (PostgreSQL never shows uncommitted data) | reads the latest version of each row, committed or not |
| Read Committed | no dirty reads | each **statement** reads from a new snapshot of committed data; an `UPDATE`/`DELETE` that finds a row changed by a concurrent committed transaction re-evaluates its `WHERE` on the new version and proceeds | each consistent read (plain `SELECT`) reads from a new snapshot per statement; writes and locking reads lock the latest committed version; gap locks mostly not used |
| Repeatable Read | no non-repeatable reads (phantoms allowed) | **snapshot isolation**: the whole **transaction** reads from one snapshot taken at its first statement; updating a row that a concurrent transaction changed and committed after the snapshot fails with a serialisation error | plain `SELECT`s read from one snapshot (a "read view") for the whole transaction; `UPDATE`, `DELETE` and locking reads operate on the **latest committed** version and take **next-key locks** (record + gap) on the ranges they scan |
| Serializable | serialisable | **serialisable snapshot isolation** (SSI, Section 17.7): snapshot isolation plus detection of dangerous read–write patterns; aborts one transaction with a serialisation error | Repeatable Read, but plain `SELECT`s in a transaction become locking reads (`FOR SHARE`): strict two-phase locking with next-key locks; conflicts block and can deadlock |

### Concrete: the anomaly × level × engine matrix

"Possible" means the anomaly can occur at that level; "Prevented" means it cannot. Footnotes explain the cases where the answer depends on how the application reads.

| Anomaly | Standard RU | Standard RC | Standard RR | Standard SER | PG RC (default) | PG RR | PG SER | InnoDB RU | InnoDB RC | InnoDB RR (default) | InnoDB SER |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Dirty write | Prevented | Prevented | Prevented | Prevented | Prevented | Prevented | Prevented | Prevented | Prevented | Prevented | Prevented |
| Dirty read | Possible | Prevented | Prevented | Prevented | Prevented | Prevented | Prevented | **Possible** | Prevented | Prevented | Prevented |
| Non-repeatable read | Possible | Possible | Prevented | Prevented | Possible | Prevented | Prevented | Possible | Possible | Prevented ¹ | Prevented |
| Phantom | Possible | Possible | Possible | Prevented | Possible | Prevented ² | Prevented | Possible | Possible | Prevented ³ | Prevented |
| Lost update (read-modify-write in the app) | Possible | Possible | Possible ⁴ | Prevented | Possible | Prevented ⁵ | Prevented | Possible | Possible | **Possible** ⁶ | Prevented |
| Read skew | Possible | Possible | Possible ⁴ | Prevented | Possible | Prevented | Prevented | Possible | Possible | Prevented ¹ | Prevented |
| Write skew | Possible | Possible | Possible | Prevented | Possible | **Possible** | Prevented | Possible | Possible | **Possible** ⁶ | Prevented |

1. For plain (consistent) `SELECT`s, which read the transaction's snapshot. A locking read (`FOR UPDATE`) or an `UPDATE` in the same transaction sees the latest committed version instead, so mixing the two can show different values.
2. In practice: PostgreSQL's Repeatable Read reads from one snapshot, so rows committed by others after it began never appear (accuracy rule 4: no phantoms in practice; write skew still possible).
3. Plain `SELECT`s see a fixed snapshot; locking reads and `UPDATE`/`DELETE` lock the scanned ranges with next-key locks, which blocks other transactions from inserting into them.
4. The standard's three-phenomenon definitions do not address these anomalies directly; implementations differ.
5. The second writer's `UPDATE` of a row changed after its snapshot fails with `could not serialize access due to concurrent update`; the application must retry.
6. InnoDB does **not** detect lost updates or write skew at Repeatable Read when values are read with plain `SELECT`s: the later `UPDATE` simply writes over the latest committed version. Using `SELECT … FOR UPDATE` for the read prevents both (Section 17.8).

The two defaults behave differently on the most common bug:

```text
 lost update with read-modify-write, both engines at their DEFAULT level

 PostgreSQL, Read Committed:  T2's UPDATE waits for T1, then overwrites → lost update (no error)
 PostgreSQL, Repeatable Read: T2's UPDATE fails: ERROR: could not serialize access due to concurrent update
 MySQL InnoDB, Repeatable Read (default): T2's UPDATE waits for T1, then overwrites → lost update (no error)
```

### Mechanism: why snapshots prevent some anomalies and not others

A snapshot gives a transaction a consistent view of the committed database as of one moment. That prevents every anomaly that consists of **reading** inconsistent data: dirty reads, non-repeatable reads, phantoms and read skew. It does not, by itself, coordinate **writes based on those reads**. Snapshot isolation adds one write rule ("first committer wins": a transaction may not update a row that another transaction changed after its snapshot), which prevents lost updates on the **same** row. Write skew writes **different** rows, so it passes both checks. Serialisability needs something more: either locks on what was read (InnoDB's Serializable, two-phase locking) or tracking read–write dependencies (PostgreSQL's SSI).

### Guarantees and non-guarantees

Each level guarantees the absence of the anomalies marked "Prevented" for its engine. Only Serializable guarantees serialisability, and it does so by blocking (InnoDB) or by aborting transactions that must be retried (PostgreSQL). No level guarantees that the application's own invariants hold if they depend on reads at a weaker level, and the same level name does not guarantee the same behaviour across engines.

### Failure and diagnosis

- **Bugs that appear after moving between databases.** Code that relied on PostgreSQL Repeatable Read's lost-update error silently loses updates on MySQL's Repeatable Read. Evidence: the engine change; read-modify-write code paths.
- **Serialisation failures in logs.** `could not serialize access …` (SQLSTATE `40001`) at Repeatable Read or Serializable in PostgreSQL. These are expected at those levels and must be retried (Section 17.7).
- **Reports that do not reconcile at Read Committed.** Read skew across statements. Fix: run the report as one Repeatable Read transaction (a single snapshot).

### Backend connection

Know the default level of your database and its exact meaning there. Use Read Committed with explicit atomic updates or row locks for most OLTP code, Repeatable Read for consistent multi-statement reads (reports, exports), and Serializable (with a retry loop) when correctness depends on invariants across rows that you cannot express as constraints. Never assume a level name means the same thing on another engine.

---

## 17.5 Two-Phase Locking and Lock Granularity

### Definitions

A **database lock** is a marker, managed by the database, that a transaction holds on a data item (a row, a range of index keys, a page, a table) to restrict what other transactions may do to it until the lock is released. It is the database form of Chapter 5's lock (Section 5.5), with two differences: transactions usually hold locks until they end, and the database itself detects deadlocks (Section 17.10).

- A **shared lock** (S, read lock) allows other shared locks on the same item but not exclusive ones: many readers, no writer.
- An **exclusive lock** (X, write lock) allows no other lock on the item: one writer, nobody else.

**Two-phase locking** (2PL) is the rule that each transaction acquires locks in a **growing phase** and releases them in a **shrinking phase**, and never acquires a new lock after releasing any lock. **Strict 2PL** additionally holds all **exclusive** locks until the transaction commits or aborts. (**Rigorous 2PL**, which holds all locks to the end, appears in Section 17.15.)

**Lock granularity** is the size of the item a lock covers: a row, a page, a table, a whole database. **Intention locks** (IS, IX, SIX) are locks on a larger item (a table) that announce that the transaction holds or will take finer locks inside it. **Lock escalation** is replacing many fine-grained locks with one coarse lock.

### Why it exists

Locks are the oldest way to prevent anomalies: a transaction that holds a shared lock on everything it read and an exclusive lock on everything it wrote, until it ends, cannot see or cause the anomalies of Section 17.3. The two-phase rule is what makes this provably correct. If a transaction could release a lock and later take another, another transaction could slip in between and observe or change an intermediate state.

### Mechanism: the compatibility matrix and the two phases

```text
 compatibility                 requested
 held                    S            X
 S                       ✔ grant      ✘ wait
 X                       ✘ wait       ✘ wait

 number of locks held
 ▲                     lock point (all locks acquired)
 │            ┌──────────●─────────┐
 │       ┌────┘   growing          │   shrinking
 │   ┌───┘                         │   (strict 2PL: X locks released only here, at COMMIT/ROLLBACK)
 │ ──┘                             └──────────
 └────────────────────────────────────────────────► time
   BEGIN   acquire, acquire, …           COMMIT: release
```

Why 2PL produces serialisable executions, in one sentence: order the transactions by their **lock points** (the moment each holds all its locks). Any two transactions that conflict on an item must hold their conflicting locks at different times, so the one whose lock point comes first finished using the item before the other started, and the whole execution is equivalent to running them in lock-point order.

**Strict** 2PL matters for recovery: if exclusive locks were released before commit, another transaction could read the value and commit, and then the first could abort. Holding write locks to the end prevents those cascading aborts (Section 17.15 names this a "strict schedule").

### Concrete: 2PL preventing a lost update

```text
 stock of product 17 = 5; both transactions read with a lock intended for writing (FOR UPDATE, Section 17.8)
 T1                                       T2
 SELECT stock … FOR UPDATE  → X lock, 5
                                          SELECT stock … FOR UPDATE  → waits for T1's X lock …
 UPDATE stock = 15
 COMMIT (releases X)
                                          … granted, reads 15
                                          UPDATE stock = 18
                                          COMMIT
 final stock 18: correct
```

### Mechanism: granularity and intention locks

Locking every row a large scan reads would create millions of lock records; locking the whole table for every update would serialise all writers. Databases lock at several granularities and use **intention locks** so that a coarse lock can be checked against fine ones cheaply:

```text
 table orders ─ IX (T1 intends to write some rows)      a transaction wanting to lock the WHOLE table in X
   ├─ row 101 ─ X (T1)                                  sees T1's IX on the table and waits, without
   ├─ row 102                                           scanning every row lock
   └─ row 103 ─ S (T2) ─ table also has IS (T2)
```

| held \ requested | IS | IX | S | SIX | X |
|---|---|---|---|---|---|
| IS | ✔ | ✔ | ✔ | ✔ | ✘ |
| IX | ✔ | ✔ | ✘ | ✘ | ✘ |
| S | ✔ | ✘ | ✔ | ✘ | ✘ |
| SIX | ✔ | ✘ | ✘ | ✘ | ✘ |
| X | ✘ | ✘ | ✘ | ✘ | ✘ |

(SIX means "shared on the whole table, intending exclusive on some rows": a transaction that reads everything and updates a few rows.)

**InnoDB** uses exactly this scheme: table-level IS/IX locks plus row-level locks in its indexes. **PostgreSQL** has eight table-level lock modes, from `ACCESS SHARE` (taken by every `SELECT`) to `ACCESS EXCLUSIVE` (taken by most `ALTER TABLE` forms, `DROP` and `TRUNCATE`), and stores row locks in the row versions themselves rather than in a lock table, so it never needs to escalate row locks. Some databases, such as SQL Server, **escalate** many row locks to a table lock to save memory, which can suddenly block unrelated transactions; neither PostgreSQL nor InnoDB escalates row locks.

### Mechanism: InnoDB's record, gap and next-key locks

To prevent phantoms with locks, a transaction must lock not only the rows it read but the **gaps** where matching rows could be inserted. InnoDB locks entries in indexes:

```text
 index on orders(amount): existing values 10, 20, 30
 gaps:   (−∞,10)  [10]  (10,20)  [20]  (20,30)  [30]  (30,+∞)

 T1 (Repeatable Read): SELECT * FROM orders WHERE amount BETWEEN 15 AND 25 FOR UPDATE;
   locks:  next-key lock on 20 = record 20 + gap (10,20)
           next-key lock on 30 = record 30 + gap (20,30)       (the scan stops at the first value past 25)
 T2: INSERT … amount = 18  → falls in gap (10,20) → waits until T1 ends
 T2: INSERT … amount = 26  → falls in gap (20,30) → waits
 T2: INSERT … amount = 35  → gap (30,+∞) not locked → proceeds
```

- A **record lock** locks one index entry; a **gap lock** locks the interval between two entries; a **next-key lock** is a record lock plus the gap before it.
- Gap locks only block **inserts** into the gap; they do not conflict with each other.
- They are taken by locking reads, `UPDATE` and `DELETE` at Repeatable Read and Serializable, but mostly not at Read Committed.
- Without a suitable index, a locking scan locks every row and gap it passes: an `UPDATE … WHERE unindexed_column = …` can lock the whole table.

### Mechanism: the lock queue, and the migration that took the site down

Lock requests on a table wait in a **queue**, and a waiting exclusive request blocks requests behind it even when they would be compatible with the current holders:

```text
 PostgreSQL
 10:00:00  T_report: SELECT … FROM orders (long-running report)  holds ACCESS SHARE on orders
 10:00:05  migration: ALTER TABLE orders ADD COLUMN note text      wants ACCESS EXCLUSIVE → queues behind T_report
 10:00:06  every new SELECT/INSERT on orders                       wants ACCESS SHARE / ROW EXCLUSIVE:
                                                                   compatible with T_report, but NOT with the
                                                                   queued ALTER → queues behind the ALTER
 → the whole application stops touching orders until the report finishes, minutes later
```

The `ALTER TABLE` itself would take milliseconds; the outage comes from waiting. Migrations must set a short lock timeout (`SET lock_timeout = '2s'` in PostgreSQL) and retry, so that a blocked DDL statement gives up instead of stalling all traffic (Chapter 18 covers migrations).

### Guarantees and non-guarantees

Strict 2PL guarantees serialisable executions when every read and write takes its lock, and prevents cascading aborts. With range (next-key) locks it also prevents phantoms. It does not guarantee freedom from deadlocks (they are a direct consequence of waiting for locks, Section 17.10), good concurrency for read-heavy workloads (readers block writers and vice versa), or short waits.

### Failure and diagnosis

- **Lock waits.** Requests slow, CPU idle. Evidence: sessions waiting on locks in `pg_locks` / `pg_stat_activity.wait_event_type = 'Lock'`, or `performance_schema.data_lock_waits` in MySQL (Section 17.14).
- **Whole-table blocking from DDL.** Evidence: many sessions waiting behind one `ALTER TABLE` waiting behind one long query. Fix: `lock_timeout` on migrations; online schema-change methods.
- **Unexpectedly broad InnoDB locks.** Inserts blocked into "unrelated" ranges. Cause: gap locks from locking scans, often over missing indexes. Evidence: `SHOW ENGINE INNODB STATUS` lock details. Fix: indexes for locking queries, or Read Committed where phantom protection is not needed.

### Backend connection

Locks are how a database turns concurrent requests into a safe order, and every lock is also a queue. Keep transactions short so locks are held briefly, index the columns your updates and locking reads filter on, and never run schema changes without a lock timeout.

---

## 17.6 Multi-Version Concurrency Control (MVCC)

### Definitions

**Multi-version concurrency control** (MVCC) keeps several **versions** of each row, so that a transaction can read a consistent **snapshot** of the database (the committed state as of one moment) while other transactions write new versions. **Readers never block writers, and writers never block readers.** Writers still conflict with writers on the same row, and use row locks for that.

- A **snapshot** records which transactions' effects are visible: those committed before it was taken.
- A **visibility rule** decides, for each row version, whether it belongs to a given snapshot.
- Old versions that no snapshot can see any more are **garbage**, removed by **VACUUM** (PostgreSQL) or **purge** (InnoDB).

### Why it exists

With pure locking, a long report holding shared locks blocks every update of the rows it read, and every update blocks the report. Most applications read far more than they write, and readers mostly want "a consistent view", not "the latest value under a lock". Keeping old versions lets every reader see a consistent snapshot without taking locks. The cost is storage for old versions and background work to remove them.

### Mechanism: PostgreSQL, versions in the heap

Every PostgreSQL row version (tuple) carries two transaction IDs in its header (Chapter 15, Section 15.2):

- **`xmin`**: the transaction that created this version;
- **`xmax`**: the transaction that deleted or superseded it (0 if none).

An `UPDATE` sets `xmax` on the old version and writes a new version whose `xmin` is the updating transaction; the old version points to the new one. Transaction IDs (XIDs) are assigned in increasing order.

A **snapshot** is three things, which PostgreSQL shows as `xmin:xmax:xip_list`:

```sql
SELECT pg_current_snapshot();          -- PostgreSQL 13+
--  100:104:101,103
```

```text
 xmin = 100      every transaction with XID < 100 had finished when the snapshot was taken
 xmax = 104      every transaction with XID ≥ 104 had not started: invisible
 xip  = 101,103  transactions in progress at the time: invisible, even if they commit later
 → visible transactions: XID < 104, not in {101, 103}, and COMMITTED (checked in pg_xact)
```

**The visibility rule** for a tuple, simplified: the tuple is visible to a snapshot if its `xmin` is visible (committed and in the snapshot's past) **and** its `xmax` is either empty, or not visible (aborted, still in progress for this snapshot, or in its future).

**A visibility check, worked through.** The row for product 17 has three versions; the reader's snapshot is `100:104:101,103`.

```text
 version  stock  xmin  xmax   check
 v1       5      90    101    xmin 90: < 100, committed → visible creator
                              xmax 101: in xip (in progress for this snapshot) → the delete is NOT visible
                              → v1 IS VISIBLE: this reader sees stock = 5
 v2       15     101   103    xmin 101: in xip → creator not visible → v2 invisible
 v3       18     103   0      xmin 103: in xip → invisible

 a newer snapshot 104:106: (none in progress), after 101 and 103 committed:
 v1  xmax 101 committed and visible → deleted for this snapshot → invisible
 v2  xmin 101 visible; xmax 103 committed and visible → superseded → invisible
 v3  xmin 103 visible; xmax 0 → VISIBLE: stock = 18
```

Two readers see different, internally consistent values of the same row at the same moment, and neither took a lock.

When a snapshot is taken depends on the isolation level: at Read Committed, a new one for every statement; at Repeatable Read and Serializable, one for the whole transaction (Section 17.4).

### Mechanism: VACUUM, bloat and wraparound

Old versions stay in the table until **VACUUM** (run automatically by autovacuum, Chapter 15, Section 15.1) removes those that **no running transaction's snapshot can see**. The boundary is the **xmin horizon**: the oldest snapshot still in use anywhere. A version deleted by a transaction that committed before the horizon is dead and can be removed; anything newer must stay, because some transaction might still need it.

```text
 xmin horizon set by the oldest open transaction (started 3 hours ago)
 ├──────────────────────────────────────────────────────────────►
 every version superseded in the last 3 hours must be KEPT, in every table, however often rows are updated
 → a hot table updated 1,000 times per second keeps ~10 million dead versions → BLOAT
```

**Bloat** is space occupied by dead versions (and the index entries that point to them). It makes scans read more pages, indexes larger and the buffer pool less effective. VACUUM makes the space reusable inside the table but does not shrink the file; returning space to the operating system needs a rewrite (`VACUUM FULL`, which locks the table, or an online rebuild tool).

**Transaction ID wraparound.** PostgreSQL's XIDs are 32-bit numbers compared in a circular space: from any XID, about 2 billion are "the past" and 2 billion "the future". A row whose `xmin` is more than about 2 billion transactions old would suddenly appear to be from the future and vanish. VACUUM prevents this by **freezing** old tuples (marking them visible to everyone, independent of their XID). Autovacuum forces an anti-wraparound vacuum on any table whose oldest unfrozen XID passes a threshold (200 million transactions by default at the time of writing). If freezing cannot keep up (because vacuum is blocked by a long transaction or keeps failing), PostgreSQL eventually stops assigning new transaction IDs and refuses writes with an error saying it is avoiding wraparound data loss, until a vacuum completes. Monitor `age(datfrozenxid)` per database.

### Mechanism: InnoDB, versions in the undo log

InnoDB keeps only the **newest** version of each row in the table (the clustered index, Chapter 16, Section 16.3). Every row carries hidden fields: **`DB_TRX_ID`**, the transaction that last changed it, and **`DB_ROLL_PTR`**, a pointer to the **undo record** holding the previous version, which points to the one before, forming a version chain.

A transaction's snapshot is a **read view**, created at its first consistent read (Repeatable Read) or at each statement (Read Committed), holding:

```text
 up_limit_id   = the lowest transaction ID active when the view was created
 low_limit_id  = the next transaction ID to be assigned at that moment
 active list   = the transactions active at that moment

 visible(trx_id):  trx_id < up_limit_id                       → visible (committed before the view)
                   trx_id ≥ low_limit_id                      → invisible (started after)
                   trx_id in the active list                  → invisible
                   otherwise                                  → visible
```

**The same check, InnoDB style.** Read view: up_limit 101, low_limit 104, active {101, 103}.

```text
 clustered index (newest)                undo chain (older versions)
 [id 17 | stock 18 | trx 103] ──roll ptr──► [stock 15 | trx 101] ──roll ptr──► [stock 5 | trx 90]

 1. newest: trx 103 is in the active list → invisible → follow the roll pointer
 2. trx 101: in the active list → invisible → follow
 3. trx 90: < up_limit 101 → visible → the reader sees stock = 5
```

**Purge** threads delete undo records once no read view can need them. A long-running transaction keeps an old read view alive, so undo cannot be purged: the **history list length** reported by `SHOW ENGINE INNODB STATUS` grows, and every consistent read that must walk long undo chains gets slower.

### Concrete: the two designs compared

| | PostgreSQL | InnoDB |
|---|---|---|
| Where old versions live | in the table (heap) | in the undo log |
| Newest version | wherever the latest tuple is | in the clustered index |
| A reader finds its version by | checking each version's `xmin`/`xmax` | starting at the newest and walking back through undo |
| Snapshot | `xmin:xmax:in-progress list` | read view: up limit, low limit, active list |
| Cleanup | VACUUM (and freezing) | purge |
| Damage from long transactions | table and index bloat, frozen-XID age grows | undo growth, long version chains, slow reads |
| Update cost | new tuple, possibly new index entries (unless HOT) | in-place write + undo record |
| Rollback | mark aborted; versions become garbage | apply undo |

### Guarantees and non-guarantees

MVCC guarantees each transaction a consistent snapshot of committed data, without read locks. It does not, by itself, prevent write conflicts (row locks and the first-committer-wins rule do that), lost updates at Read Committed, or write skew below Serializable. It guarantees nothing about space: old versions accumulate as long as any transaction might need them.

### Failure and diagnosis

- **Bloat and slow scans** (PostgreSQL). Evidence: `n_dead_tup` high in `pg_stat_user_tables`; table size far above live data; an old `backend_xmin` or `xact_start` in `pg_stat_activity`. Fix: end the long transaction, tune autovacuum for hot tables.
- **Wraparound warnings.** Log warnings that a database must be vacuumed within some number of transactions. Evidence: `age(datfrozenxid)` approaching the limit. Fix: find what blocks vacuum (long transactions, abandoned replication slots or prepared transactions), then vacuum.
- **History list growth** (InnoDB). Evidence: history list length in the millions; undo tablespace growing; slower reads. Fix: end long transactions.

### Backend connection

MVCC is why your reports do not block checkouts. Its price is paid by cleanup, and cleanup is held hostage by the oldest open transaction anywhere in the database. Keep transactions short, monitor dead tuples, frozen-XID age and history list length, and treat any session idle in a transaction for minutes as an incident (Section 17.12).

---

## 17.7 Serializable Snapshot Isolation

### Definitions

**Serializable snapshot isolation** (SSI) is a concurrency-control method that runs transactions with snapshot isolation (Section 17.4) and additionally tracks which transactions read data that others wrote. When it detects a pattern of dependencies that could make the execution non-serialisable, it aborts one of the transactions with a **serialisation failure**, which the application must **retry**. It was published by Cahill, Röhm and Fekete in 2008, and PostgreSQL has implemented it as its Serializable level since version 9.1.

A **read–write antidependency** (rw-conflict) from T1 to T2 exists when T1 reads a version of some data and T2, concurrently, writes a newer version of it: T1 did not see T2's write, so in any equivalent serial order T1 must come before T2.

### Why it exists

Write skew (Section 17.3) slips through snapshot isolation because it involves no write–write conflict. Making snapshot isolation serialisable with locks (as InnoDB's Serializable does) brings back readers blocking writers. SSI keeps the non-blocking reads of snapshots and catches the dangerous cases by bookkeeping, paying with occasional aborts instead of waits.

### Mechanism: what SSI tracks

1. Each Serializable transaction records what it **read**, with special non-blocking **SIREAD** locks (predicate locks): on individual tuples, on index ranges, or, when it scanned a whole table, on the table.
2. When a transaction **writes** something that a concurrent transaction recorded as read, SSI records an rw-conflict from the reader to the writer.
3. Every non-serialisable execution under snapshot isolation contains a "dangerous structure": a transaction with an rw-conflict **in** and an rw-conflict **out** (a **pivot**), among transactions that overlap in time. When SSI sees one forming, it aborts one of the transactions involved.

**The on-call doctors at Serializable** (PostgreSQL):

```text
 T1 (Alice)                                                T2 (Bob)
 BEGIN ISOLATION LEVEL SERIALIZABLE                        BEGIN ISOLATION LEVEL SERIALIZABLE
 SELECT count(*) … → 2  (SIREAD on Alice's and Bob's rows)
                                                           SELECT count(*) … → 2  (SIREAD on both rows)
 UPDATE Alice → off duty   (T2 read Alice's row: rw T2 → T1)
                                                           UPDATE Bob → off duty   (T1 read Bob's row: rw T1 → T2)
 COMMIT  ✔
                                                           COMMIT ✘
 ERROR:  could not serialize access due to read/write dependencies among transactions
 DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
 HINT:  The transaction might succeed if retried.
```

The cycle T1 → T2 → T1 cannot correspond to any serial order. T2 is aborted; on retry, its `SELECT` sees that Alice is already off duty, counts 1, and refuses to let Bob leave. The invariant holds.

### Mechanism: the retry loop

At Serializable (and at Repeatable Read in PostgreSQL), serialisation failures are a **normal outcome** that the application must handle, identified by SQLSTATE `40001`:

```pseudo
function runSerializable(work, maxAttempts = 5):
    for attempt in 1..maxAttempts:
        tx = db.begin(isolation = SERIALIZABLE)
        try:
            result = work(tx)                  // reads and writes only; NO emails, HTTP calls, or other side effects
            tx.commit()
            return result
        catch SerializationFailure (SQLSTATE 40001) or DeadlockDetected (40P01):
            tx.rollback()
            sleep(randomBetween(0, 10ms × 2^attempt))      // backoff with jitter (Chapter 24)
    throw TooMuchContention()                              // surface as 503 / retry later
```

The body must be safe to run several times: everything it does must be inside the database transaction, because only that is undone on abort.

### Mechanism: the costs

- **False positives.** SSI is conservative: it aborts on dangerous structures that might not have produced an anomaly. Abort rates rise with concurrency and with how much data transactions read.
- **Coarse read tracking.** SIREAD locks are promoted from rows to pages to whole tables when a transaction reads a lot, and a sequential scan locks the whole table, so a transaction that scans a table conflicts with any concurrent write to it. Good indexes (Chapter 16) reduce aborts.
- **Read-only transactions** can declare themselves `SERIALIZABLE READ ONLY DEFERRABLE`: PostgreSQL waits for a snapshot that is guaranteed safe, after which the transaction can never be aborted. This suits long reports.

### Guarantees and non-guarantees

SSI guarantees serialisability for all transactions running at Serializable, without read locks. It does not guarantee that a transaction commits (it may be aborted and must be retried), protect transactions running at weaker levels at the same time, or make non-database side effects safe to retry.

### Failure and diagnosis

- **Serialisation-failure storms.** Many retries under load, throughput falling. Evidence: rising `40001` counts; transactions that scan large tables (table-level SIREAD locks). Fix: indexes so reads lock less, smaller transactions, or a targeted lock for the hot invariant instead of global Serializable.
- **Side effects repeated.** An email sent twice because the transaction body sent it before an abort. Fix: side effects after commit, through an outbox (Chapter 25).

### Backend connection

Serializable is the simplest way to make complex invariants correct, because you write each transaction as if it ran alone. In PostgreSQL it costs aborts, not waits. Pair it with a disciplined retry wrapper, keep side effects out of the transaction, and measure the abort rate before making it the default.

---

## 17.8 Explicit Locking

### Definitions

**Explicit locking** is an application asking the database for a lock directly, instead of relying on the isolation level:

- **`SELECT … FOR UPDATE`** reads rows and takes exclusive row locks on them, held until the transaction ends. Other transactions that try to update, delete or lock those rows wait.
- **`SELECT … FOR SHARE`** takes shared row locks: others may also lock for share but may not update the rows.
- **`NOWAIT`** makes a locking read fail immediately instead of waiting if a row is already locked.
- **`SKIP LOCKED`** makes a locking read silently skip rows that are already locked.
- An **advisory lock** is a lock on an application-chosen number or name, not on any row, used to coordinate application work through the database.

(PostgreSQL adds two weaker modes, `FOR NO KEY UPDATE`, which plain `UPDATE`s that do not change key columns take, and `FOR KEY SHARE`, which foreign-key checks take, so that ordinary updates do not block inserts that reference the row.)

### Why it exists

Isolation levels protect what they protect implicitly; often a transaction needs to say "I am about to change these rows based on what I read; nobody else touch them until I am done". That is the read-modify-write case of the lost update and many write-skew cases, and `FOR UPDATE` turns them into an ordered queue regardless of isolation level. `SKIP LOCKED` exists for a different need: many workers taking **different** rows from the same table at once, without waiting on each other.

### Mechanism: FOR UPDATE, NOWAIT

```sql
BEGIN;
SELECT stock FROM products WHERE id = 17 FOR UPDATE;      -- row locked; concurrent FOR UPDATE / UPDATE waits
-- application logic decides, using the value it read
UPDATE products SET stock = 4 WHERE id = 17;
COMMIT;                                                   -- lock released
```

```sql
SELECT … FROM seats WHERE id = 'A7' FOR UPDATE NOWAIT;
-- if another transaction holds it:
-- ERROR:  could not obtain lock on row in relation "seats"     (PostgreSQL; the request fails at once)
```

`NOWAIT` suits interactive flows ("this seat is being booked by someone else, pick another") where waiting would be worse than failing fast. A middle ground is a lock timeout (`SET lock_timeout` in PostgreSQL, `innodb_lock_wait_timeout` in MySQL).

### Concrete: a job queue with SKIP LOCKED

A `jobs` table holds work for a pool of workers (Chapter 25 compares this with message brokers). Each worker must claim jobs no other worker has claimed, without blocking:

```sql
-- PostgreSQL: claim up to 10 due jobs atomically, skipping any another worker has locked
WITH next AS (
    SELECT id FROM jobs
    WHERE status = 'PENDING' AND run_at <= now()
    ORDER BY run_at
    LIMIT 10
    FOR UPDATE SKIP LOCKED
)
UPDATE jobs j
SET status = 'RUNNING', locked_by = $1, locked_at = now(), attempts = attempts + 1
FROM next
WHERE j.id = next.id
RETURNING j.id, j.kind, j.payload;
```

```text
 jobs (PENDING, due): 1 2 3 4 5 6 7 8 9 10 11 12 …
 worker A: locks 1–10 (in its transaction), marks them RUNNING, commits
 worker B (same instant): rows 1–10 are locked → SKIPPED → claims 11–20
 worker C: claims 21–30
 no worker waits for another; no job is claimed twice
```

The claim transaction commits immediately, so no locks are held while the work runs. Two design points follow:

- **Crash recovery.** A worker that dies after claiming leaves jobs `RUNNING` forever. A sweeper resets jobs whose `locked_at` is older than a lease (for example 5 minutes) back to `PENDING`, so the work happens **at least once** (Chapter 25), and job handlers must be idempotent.
- **Indexing.** A partial index on `(run_at) WHERE status = 'PENDING'` keeps the claim query cheap as completed jobs accumulate (Chapter 16, Section 16.5).

`SKIP LOCKED` is supported by PostgreSQL (9.5+) and MySQL (8.0+). It deliberately gives an **inconsistent** view (it hides locked rows), which is right for queues and wrong for almost anything else.

### Mechanism: advisory locks

Advisory locks lock an application-defined key, using the database's lock manager (and its deadlock detection) to coordinate work that is not about any particular row:

```sql
-- PostgreSQL: run the monthly statements job on exactly one instance (Chapter 11, Section 11.7)
SELECT pg_try_advisory_xact_lock(hashtext('monthly-statements-2026-10'));   -- true: we hold it; false: someone else does
-- … if true, do the work in this transaction; the lock is released automatically at commit or rollback
```

- **Transaction-scoped** advisory locks (`pg_advisory_xact_lock`, `pg_try_advisory_xact_lock`) release at the end of the transaction: the safe default.
- **Session-scoped** ones (`pg_advisory_lock`) last until explicitly released or the connection closes. Behind a connection pool or a transaction-mode pooler (Chapter 18), "the session" may be shared with other requests or moved between them, so session-level advisory locks are easy to leak or misuse.
- MySQL's equivalent is `GET_LOCK('name', timeout)` / `RELEASE_LOCK('name')`, which are session-scoped.

Advisory locks are only as good as the code that respects them: they lock nothing unless every participant asks for them. They also protect only within one database, and a lock held by a crashed or partitioned client is released when its connection dies, which other processes must allow for (distributed locks and fencing are Chapter 23's subject).

### Guarantees and non-guarantees

`FOR UPDATE` guarantees that no other transaction updates, deletes or locks the selected rows until the transaction ends, at any isolation level. `SKIP LOCKED` guarantees that concurrent claimers never wait for or receive the same locked rows. Advisory locks guarantee mutual exclusion among participants that use them. None of them protects rows that the query did not select (a `FOR UPDATE` on existing rows does not stop inserts of new matching rows, except through InnoDB's gap locks), and all of them can deadlock (Section 17.10).

### Failure and diagnosis

- **Requests queued behind one row.** Every checkout waits on `FOR UPDATE` of a popular product. Evidence: lock waits concentrated on one row (Section 17.14). Fix: shorter transactions; atomic conditional updates (Section 17.11), which hold the lock for one statement.
- **Jobs stuck in `RUNNING`.** Evidence: old `locked_at` values. Fix: lease-based sweeper; idempotent jobs.
- **Advisory locks never released.** Evidence: `pg_locks` entries of type `advisory` held by idle connections. Fix: transaction-scoped locks.

### Backend connection

Use `FOR UPDATE` when a transaction must decide based on a row and then change it, and keep that transaction short. Build simple durable job queues with `SKIP LOCKED` and a lease sweeper. Use transaction-scoped advisory locks for "only one instance does this" work, and remember they are a convention that every participant must follow.

---

## 17.9 Optimistic versus Pessimistic Concurrency

### Definitions

**Pessimistic concurrency control** assumes conflicts are likely and prevents them up front: a transaction locks what it will change before changing it (`SELECT … FOR UPDATE`, Section 17.8), and others wait.

**Optimistic concurrency control** assumes conflicts are rare and detects them at write time: a transaction reads without locking, remembers what it read (usually a **version** number), and makes its write conditional on that version being unchanged. If it changed, the write affects nothing and the transaction retries or reports a conflict. Chapter 5 (Section 5.4) introduced the idea as compare-and-swap; Chapter 13 (Section 13.7) applied it to APIs with ETags.

### Why both exist

Locks are cheap when held briefly and conflicts are frequent, but they cannot be held across slow steps: a user editing a form for two minutes cannot hold a row lock for two minutes (Chapter 11, Section 11.5). Optimistic control holds nothing while the user or the application thinks, but it wastes the work of every attempt that loses a conflict. Which one wins depends on how often conflicts happen and how long the read-to-write gap is.

### Mechanism: optimistic control with a version column

```sql
-- read (no locks)
SELECT id, stock, version FROM products WHERE id = 17;          -- stock 5, version 12
-- the application computes the new value
UPDATE products
SET stock = 4, version = version + 1
WHERE id = 17 AND version = 12;                                  -- compare-and-set
-- rows affected: 1 → success; 0 → someone changed it since we read: retry or report a conflict
```

```pseudo
function decrementStockOptimistic(productId, maxAttempts = 5):
    for attempt in 1..maxAttempts:
        (stock, version) = db.query("SELECT stock, version FROM products WHERE id = $1", productId)
        if stock < 1: return OutOfStock
        rows = db.execute("UPDATE products SET stock = $1, version = version + 1
                           WHERE id = $2 AND version = $3", stock - 1, productId, version)
        if rows == 1: return Ok
        sleep(randomJitter(attempt))                          // someone else won; try again
    return TooMuchContention
```

The comparison happens inside the `UPDATE`, atomically, so two concurrent writers cannot both succeed against the same version: the second's `UPDATE` re-checks the row after the first commits (Section 17.11) and finds `version = 13`.

### Mechanism: what contention does to each

If each attempt conflicts with probability p, an optimistic operation needs on average 1 / (1 − p) attempts:

```text
 conflict probability p      0.1      0.5      0.9
 average attempts            1.1      2        10      ← each failed attempt is wasted reads and work
```

| | Pessimistic (lock first) | Optimistic (check at write) |
|---|---|---|
| Holds locks during | the whole transaction | nothing until the final `UPDATE` (one statement) |
| Under low contention | lock overhead for nothing | nearly free |
| Under high contention | requests queue; throughput limited by lock hold time | many retries; work wasted; possible starvation of unlucky clients |
| Long read-to-write gaps (user think time) | impossible (locks would be held for minutes) | natural fit |
| Deadlocks | possible (Section 17.10) | no lock waits, so no deadlocks from the read phase |
| Failure the caller sees | a wait (or a lock timeout) | a conflict to retry or show (`409`/`412` in APIs) |

### Guarantees and non-guarantees

Both guarantee that no update is applied on top of a state the writer did not see (no lost update). Neither alone prevents write skew across different rows (Section 17.3) unless the version or lock covers the data the decision depended on. Optimistic control does not guarantee progress under heavy contention; pessimistic control does not guarantee short waits.

### Failure and diagnosis

- **Retry storms** (optimistic): conflicts rise with load, retries add load. Evidence: a rising ratio of zero-row updates to successful ones. Fix: switch hot paths to pessimistic or atomic operations; backoff with jitter.
- **Lock queues** (pessimistic): one hot row serialises everything. Evidence: lock waits on the same row (Section 17.14). Fix: shorten the locked section; use an atomic `UPDATE` (Section 17.11).

### Backend connection

Use optimistic control when conflicts are rare or the gap between read and write is long (human edits, API resources with ETags), and pessimistic locking or atomic statements when conflicts are frequent and the work between read and write is short and inside one transaction.

---

## 17.10 Database Deadlocks

### Definitions

A **database deadlock** is a cycle of transactions, each waiting for a lock held by the next (Chapter 5, Section 5.8 taught the theory: the four Coffman conditions and the wait-for graph). The database **detects** deadlocks by finding cycles in its **wait-for graph**, chooses a **victim**, and aborts it, releasing its locks so the others can proceed.

### Why it exists

Two-phase locking (Section 17.5) holds locks until commit and makes transactions wait for each other, which satisfies all four deadlock conditions: mutual exclusion, hold and wait, no preemption and circular wait. Deadlocks are therefore not a bug in the database but a normal consequence of locking, and every application that updates more than one row per transaction must expect them.

### Concrete: two transfers in opposite directions

```text
 T1: transfer from account 1 to account 2             T2: transfer from account 2 to account 1
 T1                                                   T2
 UPDATE accounts … WHERE id = 1   → X lock on row 1
                                                      UPDATE accounts … WHERE id = 2   → X lock on row 2
 UPDATE accounts … WHERE id = 2   → waits for T2
                                                      UPDATE accounts … WHERE id = 1   → waits for T1
                                                                                          DEADLOCK

 wait-for graph:      T1 ──waits for──► T2
                       ▲                │
                       └──waits for─────┘          a cycle: neither can ever proceed
```

PostgreSQL resolves it like this (in the server log and the victim's error):

```text
ERROR:  deadlock detected
DETAIL:  Process 4410 waits for ShareLock on transaction 7302; blocked by process 4412.
         Process 4412 waits for ShareLock on transaction 7301; blocked by process 4410.
HINT:  See server log for query details.
```

MySQL's InnoDB reports `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction`, and `SHOW ENGINE INNODB STATUS` shows both transactions and their locks in a "LATEST DETECTED DEADLOCK" section.

### Concrete: two more deadlock shapes

Opposite lock order is the textbook case. Two other shapes cause many deadlocks in production, and neither needs two different rows in opposite order.

**A lock upgrade on one row.** Both transactions read a row with a shared lock, then both try to update it:

```text
 T1                                              T2
 SELECT … FROM seats WHERE id = 'A7' FOR SHARE   → S lock
                                                 SELECT … FROM seats WHERE id = 'A7' FOR SHARE   → S lock (compatible)
 UPDATE seats … WHERE id = 'A7'                  → needs X; waits for T2's S
                                                 UPDATE seats … WHERE id = 'A7'   → needs X; waits for T1's S
                                                 DEADLOCK on a single row
```

The same happens implicitly in InnoDB at Serializable, where plain `SELECT`s take shared locks (the on-call doctors of Section 17.3 end this way at that level). The fix is to take the exclusive lock first (`FOR UPDATE`) when you intend to write.

**"Lock it if absent, then insert" in InnoDB.** A common pattern checks for a row with a locking read and inserts it if missing. At Repeatable Read, a locking read that finds **no** row locks the **gap** where it would be, and gap locks do not conflict with each other, but an insert into a locked gap waits for every gap lock there (Section 17.5):

```text
 unique index on users(username): existing values 'ana', 'chen'   gap ('ana','chen') contains 'ben'
 T1                                                     T2
 SELECT … WHERE username = 'ben' FOR UPDATE  → no row; gap lock ('ana','chen')
                                                        SELECT … WHERE username = 'ben' FOR UPDATE  → no row;
                                                          gap lock ('ana','chen') (gap locks are compatible)
 INSERT username 'ben'  → insert into the gap: waits for T2's gap lock
                                                        INSERT username 'ben'  → waits for T1's gap lock
                                                        DEADLOCK: InnoDB rolls one back (error 1213)
```

The fix is to let the unique constraint decide, in one statement: `INSERT … ON DUPLICATE KEY UPDATE` (MySQL) or `INSERT … ON CONFLICT` (PostgreSQL), or a plain `INSERT` whose duplicate-key error the application handles (Section 17.11). Running such code at Read Committed also avoids the gap locks.

### Mechanism: detection and victim selection

- **PostgreSQL** does not check every wait immediately. A transaction that has waited for a lock longer than `deadlock_timeout` (1 second by default at the time of writing) triggers a search of the wait-for graph; if it finds a cycle, it aborts the transaction that ran the check. Short waits, the common case, cost no graph search.
- **InnoDB** checks for a cycle immediately whenever a transaction must wait, and rolls back the transaction it judges cheaper to undo (the one that has modified or locked fewer rows). At very high concurrency, this check itself can become expensive; it can be disabled, leaving `innodb_lock_wait_timeout` to break deadlocks slowly.
- In both, the victim's **whole transaction** is rolled back, so it is safe to retry from the start: SQLSTATE `40P01` in PostgreSQL and `40001` in InnoDB, the same retry wrapper as Section 17.7.

### Mechanism: prevention

Detection makes deadlocks survivable; prevention makes them rare:

1. **Lock in a consistent order.** If every transaction locks accounts in increasing ID order, a cycle is impossible (this breaks circular wait, Chapter 5, Section 5.8). For transfers: lock `min(from, to)` first. For batches: sort the IDs before updating, or `SELECT … WHERE id = ANY($1) ORDER BY id FOR UPDATE`, because `UPDATE … WHERE id IN (…)` may touch rows in any order.
2. **Keep transactions short**, so the window for overlapping waits is small.
3. **Avoid lock upgrades.** Two transactions that each take a shared lock on a row (`FOR SHARE`, or reads at InnoDB's Serializable) and then both try to update it each wait for the other's shared lock: a deadlock from a single row. Take the exclusive lock first (`FOR UPDATE`) if you intend to write.
4. **Mind hidden locks.** Foreign-key checks lock referenced rows; InnoDB gap locks from concurrent inserts into the same range can deadlock each other; unindexed locking scans lock far more than intended.

### Guarantees and non-guarantees

The database guarantees that a deadlock is eventually broken by aborting one participant, and that the victim's work is fully rolled back. It does not guarantee which transaction is chosen, that retries succeed, or that deadlocks are rare: that depends on access order.

### Failure and diagnosis

- **Sporadic deadlock errors under load.** Evidence: the deadlock reports name the statements and rows. Look for opposite access orders, lock upgrades and gap locks. Fix: consistent ordering, shorter transactions.
- **Long stalls without deadlock errors.** Lock waits rather than cycles, or InnoDB with deadlock detection disabled waiting for `innodb_lock_wait_timeout`. Evidence: lock-wait views (Section 17.14).

### Backend connection

Every service that updates several rows in one transaction needs a retry wrapper for deadlocks and a consistent lock order. Log every deadlock with both statements (PostgreSQL logs them; InnoDB with `innodb_print_all_deadlocks`), and treat a rising rate as a design problem, not noise.

---

## 17.11 Atomic In-Database Operations and Constraints as Guards

### Definition

An **atomic in-database operation** computes and applies a change within a single SQL statement, so that the read and the write happen under the same row lock, with no gap in which another transaction can intervene. A **constraint used as a concurrency guard** makes an invalid concurrent outcome impossible, whatever the application code does.

### Why it exists

Most anomalies in Section 17.3 come from reading a value into the application, deciding there, and writing back later. Moving the decision **into** the statement removes the gap. It is usually the simplest and fastest correct solution, and it works at the default isolation level of both engines.

### Mechanism: the conditional update

```sql
UPDATE products SET stock = stock - 1
WHERE id = 17 AND stock >= 1
RETURNING stock;                                  -- PostgreSQL; in MySQL check the affected-row count
```

```text
 stock of product 17 = 1; both transactions at Read Committed (PostgreSQL)
 T1                                              T2
 UPDATE … WHERE id = 17 AND stock >= 1
   → row locked; stock 1 → 0
                                                 UPDATE … WHERE id = 17 AND stock >= 1
                                                   → row is locked by T1: wait
 COMMIT
                                                   → T1 committed: re-evaluate the WHERE on the NEW version:
                                                     stock = 0, condition false → 0 rows updated
                                                 the application sees 0 rows → "out of stock"
```

Why this is safe even at Read Committed: an `UPDATE` always operates on the **latest committed** version of the row, under an exclusive row lock, and PostgreSQL re-checks the `WHERE` clause on that version after waiting. InnoDB likewise reads the latest committed version for `UPDATE` (a "current read") under a row lock. The read and the write are one step.

The application must check the result: `0 rows` is the business answer "not available", the same affected-row-count pattern as Chapter 11 (Section 11.7).

### Mechanism: other atomic forms

```sql
-- counters: computed in the database, never read-then-written
UPDATE posts SET like_count = like_count + 1 WHERE id = $1;

-- insert-if-absent, for uniqueness and idempotency (Chapter 13, Section 13.5; Chapter 14, Section 14.11)
INSERT INTO idempotency_keys (account_id, idem_key, request_hash, status)
VALUES ($1, $2, $3, 'in_progress')
ON CONFLICT (account_id, idem_key) DO NOTHING
RETURNING idem_key;                                -- a row back: we claimed it; nothing: someone else did

-- a balance that cannot go negative: conditional update plus a constraint as a backstop
UPDATE wallets SET balance_cents = balance_cents - $2
WHERE id = $1 AND balance_cents >= $2;
ALTER TABLE wallets ADD CONSTRAINT balance_non_negative CHECK (balance_cents >= 0);
```

### Mechanism: constraints as guards

A constraint is checked against the **committed state plus the current change**, for every writer, at every isolation level (immediately per statement unless declared deferrable, Chapter 14, Section 14.3). That makes constraints the strongest concurrency guards a database offers:

| Invariant | Guard | Concurrent violation becomes |
|---|---|---|
| no duplicate email | `UNIQUE` (or a unique index on `lower(email)`) | the second insert fails (unique violation) |
| one successful payment per order | partial unique index (Chapter 16, Section 16.5) | the second success fails |
| stock never negative | `CHECK (stock >= 0)` | the update that would go negative fails |
| no overlapping bookings of a room | exclusion constraint (PostgreSQL, Chapter 16, Section 16.6) | the second overlapping insert fails |
| each order belongs to an existing customer | foreign key | the insert or the delete fails |

Unique and exclusion constraints are the general tool for **write skew** of the "two transactions insert conflicting rows" kind: the two transactions wrote different rows, but the index sees both, so the database refuses the second. Where a rule can be expressed as a constraint, it is safer than any isolation level, because it does not depend on every transaction choosing the right one.

### Guarantees and non-guarantees

A single-statement conditional update guarantees that its condition and its change are evaluated together on the latest committed version under a lock. Constraints guarantee their invariant across all concurrent transactions. Neither covers invariants that span rows in ways a condition or constraint cannot express ("at least one doctor on call", "total of these debits ≤ this limit"), which need locks on a common row or Serializable.

### Failure and diagnosis

- **Silent "success" when nothing changed.** The code does not check the affected-row count. Evidence: orders confirmed with stock unchanged. Fix: always check rows affected or use `RETURNING`.
- **A hot row.** Millions of increments on one counter row serialise on its lock. Evidence: lock waits on one row. Fix: striped counters (one row per shard of the counter, summed on read; Chapter 5, Section 5.9 for the in-memory version), or batching.

### Backend connection

Before reaching for isolation levels or explicit locks, ask whether the decision can be one statement (`UPDATE … WHERE condition`, `INSERT … ON CONFLICT`) and whether the invariant can be a constraint. Most oversell, double-booking, duplicate and double-spend bugs disappear with those two tools.

---

## 17.12 Long Transactions

### Definition

A **long transaction** is one that stays open for much longer than the work inside it needs: seconds to hours. **Idle in transaction** is the state of a session that has an open transaction but is not running a statement: the application began a transaction and has not finished it.

### Why it matters

A transaction's cost is not its own running time. While it is open, it holds its locks, it pins the oldest snapshot that cleanup must respect, and on replicas it can block or be cancelled by replication. One forgotten transaction can degrade an entire database.

### Mechanism: what an open transaction holds back

```text
 a session: BEGIN … SELECT … (then the application waits on a slow HTTP call, or forgets to commit) …

 1. LOCKS          every row it updated or locked stays locked → other writers queue (Section 17.5)
                   any table lock (even the ACCESS SHARE of a SELECT) blocks DDL → and the DDL blocks everyone
 2. CLEANUP        PostgreSQL: its snapshot is the xmin horizon → VACUUM cannot remove any version superseded
                   since it began, in ANY table → bloat, slower scans, rising frozen-XID age (Section 17.6)
                   InnoDB: its read view prevents purge → history list and undo grow; reads walk longer chains
 3. REPLICATION    PostgreSQL hot standby: a long query on a replica conflicts with WAL replay that removes
                   rows it may need → the replica either delays replay (lag grows) or cancels the query
                   ("canceling statement due to conflict with recovery"); hot_standby_feedback avoids the
                   cancellations by holding back cleanup ON THE PRIMARY instead, causing bloat there
 4. RECOVERY       InnoDB: a huge uncommitted transaction must be rolled back after a crash, which can take a long time
```

### Concrete: finding them

```sql
-- PostgreSQL: transactions open for more than a minute
SELECT pid, usename, state, now() - xact_start AS tx_age, wait_event_type, wait_event,
       left(query, 60) AS last_query
FROM pg_stat_activity
WHERE xact_start < now() - interval '1 minute'
ORDER BY xact_start;
```

```text
 pid  │ usename │ state               │ tx_age   │ wait_event_type │ wait_event │ last_query
 4415 │ app     │ idle in transaction │ 02:41:07 │ Client          │ ClientRead    │ SELECT * FROM carts WHERE id = $1 FOR UPDATE
 4502 │ app     │ active              │ 00:03:12 │ Lock            │ transactionid │ UPDATE carts SET … WHERE id = $1
```

Session 4415 started a transaction 2 hours 41 minutes ago, locked a cart row with `SELECT … FOR UPDATE`, and has been waiting for its client ever since; session 4502 wants to update that row and is blocked behind it (it waits on 4415's transaction ID, which is how PostgreSQL represents a row-lock wait).

### Mechanism: defences

- **Application structure:** one short transaction per business operation; no network calls, user input or sleeps inside a transaction (Chapter 11, Section 11.5; Chapter 12, Section 12.10).
- **Timeouts in the database:** PostgreSQL's `idle_in_transaction_session_timeout` ends sessions idle inside a transaction; `statement_timeout` bounds single statements; recent versions also offer a `transaction_timeout`. MySQL offers `wait_timeout` for idle connections and statement execution limits.
- **Chunked batch work:** a data fix touching 50 million rows runs as many transactions of, say, 10,000 rows each, so that no transaction holds locks or the cleanup horizon for long.
- **Reports on replicas** with snapshots that are allowed to be cancelled, or on a separate analytical store (Chapter 15, Section 15.7).

### Guarantees and non-guarantees

A long transaction still has all its ACID guarantees; nothing is incorrect. What it does not guarantee is anyone else's performance: locks, cleanup and replication all degrade around it.

### Failure and diagnosis

- **Bloat, slow queries, lock waits, rising replication lag, all at once.** Evidence: the query above; `backend_xmin` of the oldest session; InnoDB `information_schema.innodb_trx` ordered by `trx_started`. Fix: terminate the session (after understanding it), then fix the code path that left it open.
- **Recurring idle-in-transaction sessions.** Evidence: the same query text each time. Cause: a code path that begins a transaction and returns early, or makes an HTTP call inside it. Fix: the code; plus the timeout as a safety net.

### Backend connection

Measure transaction duration, not just query duration. Alert on any transaction open longer than a minute or so, set idle-in-transaction timeouts, and run large data changes in small batches.

---

## 17.13 Worked Case: Oversell and Double-Spend, Solved Four Ways

### Definition

**Oversell** is selling more units of an item than exist, because concurrent purchases each saw enough stock. **Double-spend** is spending the same balance twice, because concurrent withdrawals each saw enough funds. Both are lost updates or write skews at the business level.

### Why it matters

These are the concurrency bugs that cost money directly, and they are the standard interview question for this chapter ("two customers buy the last item at the same moment"). There are four correct solutions with different trade-offs, and one incorrect one that is surprisingly common.

### Concrete: the bug

```text
 products: id 17, stock 1. Two checkouts, Read Committed (PostgreSQL) or Repeatable Read (InnoDB default).
 T1                                           T2
 SELECT stock (id 17) → 1   ✔ enough
                                              SELECT stock (id 17) → 1   ✔ enough
 UPDATE stock = 0 (computed: 1 − 1)
 INSERT order A
 COMMIT
                                              UPDATE stock = 0 (computed: 1 − 1)      ← waits, then overwrites
                                              INSERT order B
                                              COMMIT
 two orders, one keyboard. (At PostgreSQL Repeatable Read, T2 would instead get a serialisation error.)
```

### Concrete: four solutions

**A. Atomic conditional update** (Section 17.11)

```sql
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 17 AND stock >= 1;   -- check rows affected
-- 1 row: continue; 0 rows: ROLLBACK, "sold out"
INSERT INTO orders …;
COMMIT;
```

T2's `UPDATE` waits for T1's row lock, re-evaluates `stock >= 1` on the committed version (0), updates nothing, and T2 reports "sold out".

**B. Pessimistic lock: `SELECT … FOR UPDATE`** (Section 17.8)

```sql
BEGIN;
SELECT stock FROM products WHERE id = 17 FOR UPDATE;   -- T2 waits here until T1 commits, then reads 0
-- application: if stock < 1 → ROLLBACK, "sold out"; else
UPDATE products SET stock = stock - 1 WHERE id = 17;
INSERT INTO orders …;
COMMIT;
```

**C. Optimistic version check** (Section 17.9)

```sql
SELECT stock, version FROM products WHERE id = 17;              -- 1, v12 (both transactions)
BEGIN;
UPDATE products SET stock = 0, version = 13 WHERE id = 17 AND version = 12;   -- T1: 1 row; T2: 0 rows
-- T2 retries: re-reads stock 0, v13 → "sold out"
INSERT INTO orders …;
COMMIT;
```

**D. Serializable with retry** (Section 17.7)

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT stock FROM products WHERE id = 17;          -- plain read
UPDATE products SET stock = 0 WHERE id = 17;       -- PostgreSQL: T2 gets a serialisation failure (40001)
INSERT INTO orders …;
COMMIT;                                            -- the retry wrapper re-runs T2, which now reads 0
```

In InnoDB, Serializable turns the plain `SELECT` into a shared lock: both transactions hold shared locks and then both need exclusive ones, so one is chosen as a deadlock victim (Section 17.10) and retried. The outcome is correct, by a different mechanism.

### Concrete: comparing them

Assume a flash sale on one hot product, and (as illustrative assumptions) a statement round trip of 0.2 ms inside the data centre and 3 ms of application work between the read and the write in solutions B and C.

| | A. Conditional `UPDATE` | B. `FOR UPDATE` | C. Optimistic version | D. Serializable + retry |
|---|---|---|---|---|
| Correct? | yes | yes | yes | yes (both engines) |
| Row lock held for | one statement, until commit (keep the transaction short) | read → app logic → write → commit | only the final statement | engine-dependent: no read locks (PostgreSQL), shared locks (InnoDB) |
| Throughput on one hot row (illustrative) | highest: the lock is held for the shortest time | lower: lock held through app work (~3 ms → ~300 purchases/s on that row) | collapses under heavy contention: most attempts lose and retry | aborts or deadlocks rise with contention |
| What the losing buyer experiences | immediate "sold out" (after a short wait) | a wait, then "sold out" | a retry, then "sold out" | an abort, a retry, then "sold out" |
| Code complexity | lowest | low | retry loop | retry wrapper; everything in the transaction must be retry-safe |
| Generalises to | single-row conditions | any logic over rows you can lock | resources edited over long gaps (APIs, forms) | complex invariants across many rows |

**The double-spend variant.** A wallet whose balance is a **sum of ledger rows** (deposits and withdrawals) has no single row to update conditionally: two withdrawals each sum the ledger, each see enough funds, and each **insert** a different withdrawal row. That is write skew (Section 17.3), and neither plain row locks on the new rows nor snapshot isolation stop it. Correct solutions: keep a **balance row** per wallet and use A (`UPDATE wallets SET balance = balance - $2 WHERE id = $1 AND balance >= $2`) in the same transaction as the ledger insert; or lock the wallet row with B before summing; or run at Serializable (D), which detects the read–write conflict on the ledger.

### Guarantees and non-guarantees

All four guarantee no oversell for the stock row they protect. None guarantees fairness (who gets the last item), protection of invariants on rows they do not touch, or correctness when part of the purchase happens outside the database (a payment call after the stock decrement, which needs the compensation patterns of Chapter 25).

### Failure and diagnosis

- **Oversell despite "using transactions".** Evidence: plain `SELECT` then computed `UPDATE` at Read Committed (or InnoDB Repeatable Read). Fix: A or B.
- **Hot-item meltdown.** A flash sale where checkouts time out. Evidence: lock waits or retry rates concentrated on one product. Fix: A with very short transactions; or allocate stock in batches (reservation rows) to spread contention.

### Backend connection

Default to A for counters and stock, B when the decision needs logic that cannot be one statement, C for long-lived edits, and D for invariants spanning many rows. Whatever you choose, add the constraint (`CHECK (stock >= 0)`) as a last line of defence.

---

## 17.14 Diagnosis Toolkit: Locks, Deadlocks, Serialisation Failures, Idle Transactions

### Definition

The **concurrency evidence** of a database is the set of views and logs that show who holds which locks, who waits for whom, which transactions are open, and which were aborted by deadlocks or serialisation failures.

### Why it matters

Concurrency problems show up as latency (waiting), errors (deadlocks, serialisation failures) or wrong data (anomalies). The first two leave precise evidence; reading it turns "the database is slow" into "session 4502 is waiting for a row locked by session 4415, which has been idle in a transaction for 2 hours".

### Concrete: who is blocking whom (PostgreSQL)

```sql
SELECT blocked.pid  AS blocked_pid,  left(blocked.query, 50)  AS blocked_query,
       blocking.pid AS blocking_pid, left(blocking.query, 50) AS blocking_query,
       blocking.state AS blocking_state, now() - blocking.xact_start AS blocking_tx_age
FROM pg_stat_activity blocked
JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS b(pid) ON true
JOIN pg_stat_activity blocking ON blocking.pid = b.pid;
```

```text
 blocked_pid │ blocked_query                 │ blocking_pid │ blocking_query                 │ blocking_state      │ blocking_tx_age
 4502        │ UPDATE carts SET … WHERE id = │ 4415         │ SELECT * FROM carts WHERE id = $1 FOR UPDATE │ idle in transaction │ 02:41:07
 4519        │ UPDATE carts SET … WHERE id = │ 4502         │ UPDATE carts SET … WHERE id =  │ active              │ 00:03:12
```

`pg_blocking_pids` (PostgreSQL 9.6+) returns the sessions blocking a given one. `pg_locks` shows every lock in detail; row-lock waits appear there as waits for another transaction's ID, because PostgreSQL records row locks in the rows themselves. With `log_lock_waits = on`, PostgreSQL logs any lock wait longer than `deadlock_timeout`, which catches waits that happen while nobody is watching.

### Concrete: the MySQL equivalents

```sql
-- MySQL 8.0
SELECT * FROM sys.innodb_lock_waits\G                       -- waiting and blocking transactions, their queries
SELECT trx_id, trx_state, trx_started, trx_rows_locked, trx_query
FROM information_schema.innodb_trx ORDER BY trx_started;    -- open transactions, oldest first
SHOW ENGINE INNODB STATUS\G                                  -- LATEST DETECTED DEADLOCK, history list length
```

Setting `innodb_print_all_deadlocks = ON` writes every deadlock to the error log, not only the latest.

### Concrete: reading the signals

| Signal | Where | Means | First action |
|---|---|---|---|
| Sessions in `wait_event_type = 'Lock'` | `pg_stat_activity` | lock waits | find the blocker with `pg_blocking_pids` |
| `idle in transaction` for minutes | `pg_stat_activity` | an open transaction holding locks and the cleanup horizon | identify the code path; terminate if harmful |
| `deadlock detected` / error 1213 | server log; client errors | a lock cycle | read both statements; fix lock order |
| SQLSTATE `40001` | client errors | serialisation failure (or InnoDB deadlock) | confirm the retry wrapper; measure the rate |
| `Lock wait timeout exceeded` (MySQL 1205) | client errors | waited longer than `innodb_lock_wait_timeout` | find the blocker in `sys.innodb_lock_waits` |
| History list length growing | InnoDB status | purge blocked by an old read view | find the oldest transaction in `innodb_trx` |
| `n_dead_tup` growing; old `backend_xmin` | PostgreSQL statistics | VACUUM blocked | find the oldest snapshot holder |

### Backend connection

Put lock waits, transaction age, deadlock and serialisation-failure rates on the database dashboard, and log lock waits. Most concurrency incidents are found within minutes by the blocking-sessions query; most are caused by one long or idle transaction.

---

## 17.15 Placement-Round Topics: Schedules, Serialisability and Textbook Protocols

> **Placement-round topics.** This section is for campus placement rounds, university DBMS exams and written tests, where these topics are asked often. Product-company backend interviews rarely ask them in this form. **Give your main focus to Sections 17.1–17.14**; read this section before placement rounds and come back to it for revision.

### Transaction states

Textbooks draw a transaction's life as a state machine:

```text
              last statement                commit record
              executed                      durable (Ch 15, 15.4)
 ┌────────┐ ─────────────► ┌───────────────────┐ ──────────────► ┌───────────┐
 │ ACTIVE │                │ PARTIALLY         │                 │ COMMITTED │
 └────────┘                │ COMMITTED         │                 └───────────┘
      │                    └───────────────────┘
      │ error, ROLLBACK,           │ failure before the commit is durable
      │ deadlock victim            │ (crash, constraint check at commit)
      ▼                            ▼
 ┌────────┐   rollback complete   ┌─────────┐
 │ FAILED │ ────────────────────► │ ABORTED │  → restart (if the cause was transient) or kill
 └────────┘                       └─────────┘
```

- **Active**: executing statements.
- **Partially committed**: the last statement has run, but the commit is not yet durable; the transaction can still fail here (a deferred constraint checked at commit, a crash before the log flush).
- **Committed**: the commit record is durable; the effects can no longer be undone by the database.
- **Failed**: normal execution cannot continue (PostgreSQL's aborted-transaction state of Section 17.1 is this state).
- **Aborted**: rolled back; the database is as if the transaction never ran. The system may **restart** it (a deadlock victim, Section 17.10) or **kill** it (a logic error).

### Schedules and conflicts

A **schedule** is an ordering of the operations of several transactions that keeps each transaction's own operations in order. Notation: r1(A) means "T1 reads A", w2(B) "T2 writes B", c1 "T1 commits". A **serial** schedule runs transactions one after another; a **serialisable** schedule is equivalent to some serial schedule.

Two operations **conflict** if they belong to different transactions, access the same item, and at least one is a write: read–write, write–read and write–write pairs. Swapping two adjacent non-conflicting operations does not change the result.

### Conflict serialisability and the precedence graph

A schedule is **conflict serialisable** if non-conflicting swaps can turn it into a serial schedule. The test: build a **precedence graph** with a node per transaction and an edge Ti → Tj whenever an operation of Ti conflicts with a **later** operation of Tj. The schedule is conflict serialisable if and only if the graph has **no cycle**; a topological order of the graph gives the equivalent serial order.

```text
 S1:  r1(A) w1(A) r2(A) w2(A) r1(B) w1(B)
      conflicts: w1(A) before r2(A), w1(A) before w2(A)  → T1 → T2   (no edge back)
      acyclic → conflict serialisable, equivalent to T1, T2

 S2:  r1(X) r2(X) w1(X) w2(X)          (the lost update of Section 17.3)
      r1(X) before w2(X) → T1 → T2;   r2(X) before w1(X) → T2 → T1
      cycle T1 ⇄ T2 → NOT conflict serialisable
```

### View serialisability

Two schedules are **view equivalent** if every read reads the value written by the same transaction (or the initial value) in both, and the final write of each item is by the same transaction. A schedule is **view serialisable** if it is view equivalent to a serial schedule. Every conflict-serialisable schedule is view serialisable; the converse fails only with **blind writes** (writes without a preceding read):

```text
 S3:  r1(X) w2(X) w1(X) w3(X)
      precedence graph: r1(X)→w2(X): T1→T2;  w2(X)→w1(X): T2→T1  → cycle: not conflict serialisable
      view: in S3, T1 reads the initial X (as in serial T1,T2,T3); the final write is T3's (as in T1,T2,T3)
      → view serialisable, equivalent to T1, T2, T3 (w2 and w3 are blind writes)
```

Testing view serialisability is NP-complete, which is one reason real systems use conflict-based mechanisms.

### Recoverable, cascadeless and strict schedules

- **Recoverable**: if Tj reads a value written by Ti, then Ti commits before Tj commits. Otherwise Tj could commit having read data that Ti later rolls back, which can never be undone.
- **Cascadeless** (avoids cascading aborts): transactions read only values written by **committed** transactions, so one abort never forces others to abort.
- **Strict**: no transaction reads **or overwrites** a value written by an uncommitted transaction. Strict schedules make undo simple (restore the old value). Strict 2PL produces strict schedules.

```text
 non-recoverable:  w1(A) r2(A) c2 … a1        T2 committed after reading T1's value; T1 aborts → T2 cannot be undone
 recoverable but cascading:  w1(A) r2(A) a1 → T2 must abort too
```

Every strict schedule is cascadeless, and every cascadeless schedule is recoverable.

### Variants of two-phase locking

| Variant | Rule | Gives |
|---|---|---|
| Basic 2PL | no lock acquired after any lock is released | conflict serialisability; deadlocks and cascading aborts possible |
| Conservative (static) 2PL | acquire **all** locks before the transaction starts (declare its read and write sets) | no deadlocks; needs the sets in advance; less concurrency |
| Strict 2PL | basic 2PL + hold **exclusive** locks until commit or abort | strict, cascadeless schedules (what databases implement) |
| Rigorous 2PL | basic 2PL + hold **all** locks until commit or abort | strict schedules; transactions serialise in commit order |

### Deadlock prevention with timestamps: wait-die and wound-wait

Real databases detect deadlocks (Section 17.10); textbooks also teach two schemes that **prevent** them, using each transaction's start timestamp (a smaller timestamp means **older**). When Ti requests a lock held by Tj:

| Scheme | Ti older than Tj | Ti younger than Tj | Kind |
|---|---|---|---|
| **Wait-die** | Ti waits | Ti **dies** (aborts, restarts later with its **original** timestamp) | non-preemptive: only the requester is ever aborted |
| **Wound-wait** | Ti **wounds** Tj (Tj is aborted) | Ti waits | preemptive: an older transaction aborts the younger holder |

In both, waits only go in one direction of age (wait-die: old waits for young; wound-wait: young waits for old), so no cycle can form. Keeping the original timestamp on restart means a transaction eventually becomes the oldest and cannot be aborted again, which prevents starvation. Wait-die tends to cause more restarts (young requesters die repeatedly); wound-wait fewer. A related simpler scheme is the **timeout**: wait at most a fixed time, then abort, which prevents permanent deadlock at the cost of aborting some transactions that were not deadlocked.

### Timestamp ordering and Thomas's write rule

The **timestamp-ordering protocol** gives each transaction T a timestamp TS(T) when it starts, and each item X a read timestamp RTS(X) and a write timestamp WTS(X) (the largest timestamps of transactions that read or wrote it). It forces conflicting operations to happen in timestamp order:

```text
 T reads X:   if TS(T) < WTS(X): a younger transaction already wrote X → abort T (restart with a new timestamp)
              else read X; RTS(X) = max(RTS(X), TS(T))
 T writes X:  if TS(T) < RTS(X): a younger transaction already read the old value → abort T
              if TS(T) < WTS(X): a younger transaction already wrote X → abort T
              else write X; WTS(X) = TS(T)
```

**Thomas's write rule** relaxes the last write check: if TS(T) < WTS(X) (but TS(T) ≥ RTS(X)), T's write is **obsolete**, because a younger transaction's write already supersedes it and nobody read in between, so it is **ignored** instead of aborting T. This admits some view-serialisable schedules that are not conflict serialisable. Timestamp ordering causes no deadlocks (nobody waits) but can starve long transactions with repeated restarts.

### Validation-based (optimistic) protocol

The textbook optimistic protocol runs each transaction in three phases: **read** (work on private copies), **validate** (check that no committed transaction conflicted with what it read), and **write** (install its changes if validation passed; otherwise restart). It is the formal ancestor of Section 17.9's version checks and of SSI's conflict detection.

### Drill for this section

**C1. Is the schedule r1(A) r2(A) w1(A) w2(A) c1 c2 conflict serialisable? Draw its precedence graph.**
A strong answer must mention:
- Conflicts: r1(A)→w2(A) gives T1→T2; r2(A)→w1(A) gives T2→T1; w1(A)→w2(A) gives T1→T2.
- The graph has the cycle T1 ⇄ T2, so it is not conflict serialisable.
- It is the lost-update anomaly; 2PL or snapshot isolation's first-committer-wins rule would prevent it.
*Common weak answer:* checking only write–write conflicts, which gives T1→T2 alone and the wrong conclusion.

**C2. Define recoverable, cascadeless and strict schedules, and give a non-recoverable example.**
A strong answer must mention:
- Recoverable: a reader commits only after the writer it read from commits.
- Cascadeless: reads only committed data; strict: neither reads nor overwrites uncommitted data.
- Strict ⊂ cascadeless ⊂ recoverable.
- Example: w1(A) r2(A) c2 a1.
*Common weak answer:* confusing recoverable with serialisable; they are independent properties.

**C3. Compare basic, conservative, strict and rigorous 2PL.**
A strong answer must mention:
- All guarantee conflict serialisability.
- Conservative avoids deadlocks by locking everything up front.
- Strict holds exclusive locks to commit (no cascading aborts); rigorous holds all locks to commit.
- Databases implement strict 2PL (InnoDB at Serializable, and for writes generally).
*Common weak answer:* "strict 2PL has no deadlocks." Only conservative 2PL avoids them.

**C4. Explain the timestamp-ordering protocol and Thomas's write rule.**
A strong answer must mention:
- Timestamps per transaction; read and write timestamps per item; abort when an operation arrives too late.
- Thomas's rule ignores obsolete writes instead of aborting, admitting some view-serialisable schedules.
- No waiting, so no deadlocks; possible starvation through restarts.
*Common weak answer:* "Thomas's rule lets transactions write in any order." It only skips writes already superseded by a younger write.

**C5. Draw the transaction state diagram. Compare wait-die and wound-wait.**
A strong answer must mention:
- Active → partially committed → committed; active or partially committed → failed → aborted; aborted transactions are restarted or killed.
- Partially committed: last statement done, commit not yet durable; it can still fail.
- Wait-die: an older requester waits, a younger one dies; wound-wait: an older requester wounds (aborts) the younger holder, a younger requester waits.
- Waits go one way in age, so no cycles; restarted transactions keep their timestamps, preventing starvation.
*Common weak answer:* swapping the two schemes, or saying the aborted transaction gets a new timestamp, which reintroduces starvation.

---

## Chapter Summary

A **transaction** groups reads and writes into one unit: `COMMIT` makes all its changes durable and visible, `ROLLBACK` discards them all, **autocommit** makes every statement its own transaction, and **savepoints** allow partial rollback inside one. **ACID** names four separate guarantees with separate mechanisms: **atomicity** (PostgreSQL hides an aborted transaction's row versions; InnoDB applies undo), **consistency** (declared constraints plus correct application logic, the one property the database cannot provide alone), **isolation** (concurrency control: locks and MVCC), and **durability** (the write-ahead log flushed at commit).

Concurrent transactions can produce **anomalies**: dirty writes and reads, **non-repeatable reads**, **phantoms**, **lost updates**, **read skew** and **write skew**, the last being two transactions that each read overlapping data and write different rows, together breaking an invariant (the on-call doctors). **Isolation levels** are defined by which anomalies remain possible, and engines differ: PostgreSQL's Read Committed (default) takes a snapshot per statement; its Repeatable Read is **snapshot isolation** (no phantoms in practice, lost updates raise serialisation errors, write skew possible); its Serializable is **SSI**. InnoDB's Repeatable Read (default) uses a transaction snapshot for plain reads and **next-key locks** for locking reads and writes, but does not detect lost updates or write skew with plain reads; its Serializable converts reads into shared locks.

Two mechanisms implement isolation. **Two-phase locking** holds shared and exclusive locks (with **intention locks** for granularity, and gap and next-key locks for ranges) until commit, giving serialisable executions at the cost of waiting and deadlocks. **MVCC** keeps row versions so readers see snapshots without locks: PostgreSQL stores versions in the heap with `xmin`/`xmax` and cleans them with **VACUUM** (and must freeze them to prevent **XID wraparound**); InnoDB keeps the newest version in place and older ones in **undo chains**, cleaned by **purge**. **SSI** adds tracking of read–write dependencies to snapshot isolation and aborts transactions that could make the outcome non-serialisable, so applications must **retry**.

Applications choose their own tools on top: **explicit locks** (`FOR UPDATE`, `NOWAIT`, `SKIP LOCKED` for job queues, advisory locks), **pessimistic** locking for short, high-contention decisions and **optimistic** version checks for rare conflicts and long gaps, and above all **atomic in-database operations** (`UPDATE … WHERE stock >= 1`, `INSERT … ON CONFLICT`) and **constraints** as concurrency guards. **Deadlocks** are a normal consequence of locking: databases detect cycles and abort a victim, and applications prevent them with consistent lock ordering and short transactions, and retry when they happen. **Long transactions** hold locks, block VACUUM and purge, and disturb replicas, so transactions must be short and monitored. The oversell problem has four correct solutions (conditional update, `FOR UPDATE`, version check, Serializable with retry) with different throughput and failure behaviour, and the evidence for every concurrency incident lives in the blocking-sessions, open-transaction and deadlock views.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Transaction | All-or-nothing writes; durable after commit | Effects outside the database; that the client learns the outcome; isolation beyond its level |
| Savepoint | Partial rollback inside a transaction | Durability or visibility before the outer commit |
| ACID consistency | Declared constraints always hold | Undeclared invariants under weak isolation |
| Read Committed (PG) | No dirty reads; each statement sees committed data | Repeatable reads; protection from lost updates or skew |
| Repeatable Read (PG) | One snapshot; no non-repeatable reads, phantoms or lost updates (errors instead) | Protection from write skew |
| Repeatable Read (InnoDB) | Snapshot for plain reads; next-key locks for locking reads and writes | Detection of lost updates or write skew with plain reads |
| Serializable | Serialisable outcomes (PG by aborts, InnoDB by locks) | That a transaction commits without retry or waiting |
| Strict 2PL | Serialisable, strict schedules | Freedom from deadlocks; short waits |
| MVCC | Lock-free consistent reads | Prevention of write conflicts by itself; bounded storage of old versions |
| VACUUM / purge | Removal of versions no snapshot needs | Progress while a long transaction holds the horizon |
| SSI | Serialisability without read locks | Commit without retry; safety of side effects in retried code |
| `FOR UPDATE` | No other writer on the selected rows until commit | Protection from inserts of new matching rows (except InnoDB gap locks) |
| `SKIP LOCKED` | Concurrent claimers never get the same locked rows | A consistent view (it hides rows by design) |
| Optimistic version check | No write on top of an unseen version | Progress under heavy contention |
| Conditional `UPDATE` / constraint | Correctness across all concurrent writers for that row or rule | Invariants spanning rows in ways a condition or constraint cannot express |
| Deadlock detection | Cycles broken by aborting a victim | Which transaction is chosen; rarity of deadlocks |

---

## Interview Drill

**Q1. What is a transaction? What do commit, rollback, autocommit and savepoints do?**
A strong answer must mention:
- A unit of work whose writes take effect all or nothing, isolated from concurrent transactions to the chosen level.
- Commit makes changes durable and visible; rollback discards them; a crash before commit equals rollback.
- Autocommit runs each statement as its own transaction, so a multi-statement operation needs explicit `BEGIN … COMMIT`.
- Savepoints allow partial rollback inside a transaction; PostgreSQL aborts the whole transaction on any error unless a savepoint is used.
*Common weak answer:* "A transaction is a group of SQL statements." Misses the all-or-nothing and isolation guarantees, and autocommit.

**Q2. Explain ACID precisely. What implements each property?**
A strong answer must mention:
- Atomicity: PostgreSQL marks the transaction aborted and its versions become invisible; InnoDB applies undo.
- Consistency: constraints plus correct application logic; the database guarantees only declared invariants.
- Isolation: concurrency control (MVCC snapshots, locks, SSI), to the chosen level.
- Durability: the WAL/redo log flushed at commit.
- ACID consistency is not CAP consistency.
*Common weak answer:* reciting the four words with one-line definitions and no mechanisms.

**Q3. What are a dirty read, a non-repeatable read and a phantom?**
A strong answer must mention:
- Dirty read: reading another transaction's uncommitted write.
- Non-repeatable read: the same row read twice gives different values after another transaction commits a change.
- Phantom: the same query gives a different set of rows after another transaction inserts or deletes matching rows.
- Preventing phantoms requires snapshots or range (gap/next-key) locks, because the rows do not exist yet.
*Common weak answer:* treating non-repeatable reads and phantoms as the same thing.

**Q4. What are a lost update, read skew and write skew? Give a write-skew example.**
A strong answer must mention:
- Lost update: two read-modify-write cycles on the same value; one overwrites the other.
- Read skew: two related values read at different moments, giving a combination that never existed.
- Write skew: two transactions read overlapping data, write different rows, and jointly break an invariant (on-call doctors; double booking; double spend from a ledger).
- Snapshot isolation prevents lost updates on the same row but not write skew.
*Common weak answer:* describing write skew as "two transactions write the same row", which is a lost update.

**Q5. What are the isolation levels, and what does each prevent? What do PostgreSQL and MySQL actually do?**
A strong answer must mention:
- Standard levels defined by dirty reads, non-repeatable reads and phantoms; only Serializable is serialisable.
- PostgreSQL: RC default (snapshot per statement); RR = snapshot isolation (no phantoms in practice, lost updates raise errors, write skew possible); Serializable = SSI with retries.
- InnoDB: RR default (snapshot for plain reads, next-key locks for locking reads and writes, no lost-update detection with plain reads); Serializable = locking reads.
- Teach by "which anomalies remain", grounded in snapshots versus locks.
*Common weak answer:* reciting the standard table and assuming every engine follows it.

**Q6. What is two-phase locking? What are shared, exclusive and intention locks?**
A strong answer must mention:
- Growing then shrinking phases; no lock acquired after any release; serialisable by lock-point order.
- Strict 2PL holds exclusive locks until commit, avoiding cascading aborts.
- S and X compatibility; intention locks (IS, IX, SIX) at table level announce finer locks for cheap conflict checks.
- Gap and next-key locks for ranges (InnoDB); deadlocks as a consequence.
*Common weak answer:* "Locks are acquired in two steps." The phases describe the order of acquiring and releasing all locks.

**Q7. How does MVCC work? How do PostgreSQL and InnoDB differ?**
A strong answer must mention:
- Multiple row versions; each transaction reads a snapshot; readers and writers do not block each other.
- PostgreSQL: versions in the heap with `xmin`/`xmax`; snapshot `xmin:xmax:in-progress`; visibility rule; VACUUM removes dead versions.
- InnoDB: newest version in place with trx id and roll pointer; older versions in undo; read views; purge.
- Writers still lock rows; long transactions block cleanup.
*Common weak answer:* "MVCC means no locks." Writes still take row locks.

**Q8. What is VACUUM, and what is transaction ID wraparound?**
A strong answer must mention:
- VACUUM removes versions no snapshot can see (below the xmin horizon), records free space, sets visibility-map bits, freezes old tuples.
- Bloat when VACUUM falls behind or is blocked by long transactions.
- 32-bit XIDs compared circularly; tuples must be frozen before ~2 billion transactions pass; otherwise PostgreSQL stops assigning XIDs to protect data.
- Monitor dead tuples and `age(datfrozenxid)`.
*Common weak answer:* "VACUUM defragments the disk." It removes dead versions and prevents wraparound.

**Q9. What is serialisable snapshot isolation, and why must applications retry?**
A strong answer must mention:
- Snapshot isolation plus tracking of read–write dependencies (SIREAD locks).
- Aborts a transaction in a dangerous structure (a pivot with rw-conflicts in and out), which catches write skew.
- Aborts are normal (SQLSTATE 40001); the transaction body must be retried, so it must have no external side effects.
- Costs: false positives, coarse read tracking on scans.
*Common weak answer:* "Serializable runs transactions one at a time." It runs them concurrently and aborts the dangerous ones.

**Q10. What do `SELECT … FOR UPDATE` and `SKIP LOCKED` do? How would you build a job queue in a database?**
A strong answer must mention:
- `FOR UPDATE` locks the selected rows until commit; others wait (or fail with `NOWAIT`).
- `SKIP LOCKED` skips rows locked by others, so many workers claim different jobs without waiting.
- Claim in a short transaction (`UPDATE … FROM (SELECT … FOR UPDATE SKIP LOCKED) RETURNING`), process outside it, lease timeout and sweeper for crashed workers; jobs must be idempotent (at-least-once).
- A partial index on pending jobs.
*Common weak answer:* `SELECT` the next pending job, then `UPDATE` it: two workers get the same job.

**Q11. Optimistic versus pessimistic locking: how do they work, and when would you use each?**
A strong answer must mention:
- Pessimistic: lock before changing (`FOR UPDATE`); others wait; cannot span user think time.
- Optimistic: read a version, write `WHERE version = $v`, check rows affected, retry on conflict.
- Expected attempts 1/(1−p): optimistic degrades under contention; pessimistic queues on hot rows.
- Optimistic for rare conflicts and long gaps (forms, APIs with ETags); pessimistic or atomic statements for hot, short decisions.
*Common weak answer:* "Optimistic is faster." Only when conflicts are rare.

**Q12. How do database deadlocks happen, and how are they detected and prevented?**
A strong answer must mention:
- Transactions holding locks while waiting for each other's locks form a cycle (opposite lock orders; lock upgrades from shared to exclusive; gap locks).
- Detection: wait-for graph cycles; PostgreSQL after `deadlock_timeout`, InnoDB immediately; a victim is rolled back.
- Prevention: consistent lock order (sorted IDs), short transactions, exclusive locks up front.
- Retry the victim's whole transaction.
*Common weak answer:* "Use a timeout." Timeouts break deadlocks slowly; detection, ordering and retries are the answer.

**Q13. How do you prevent two customers from buying the last item?**
A strong answer must mention:
- The bug: plain read, then computed write, at Read Committed (or InnoDB Repeatable Read).
- Four fixes: conditional `UPDATE … WHERE stock >= 1` with a row-count check; `SELECT … FOR UPDATE`; optimistic version column; Serializable with retry.
- Trade-offs: lock duration, throughput on a hot row, retries, complexity.
- A `CHECK (stock >= 0)` backstop; the double-spend variant needs a balance row or Serializable.
*Common weak answer:* "Wrap it in a transaction." A transaction at the default level does not prevent it.

**Q14. Why are long-running transactions dangerous?**
A strong answer must mention:
- They hold locks (and table locks that block DDL, which blocks everyone).
- They pin the oldest snapshot: VACUUM (PostgreSQL) or purge (InnoDB) cannot clean, causing bloat, undo growth and wraparound risk.
- They disturb replicas (query cancellations or replication lag; hot-standby feedback moves the bloat to the primary).
- Defences: short transactions, no network calls inside, idle-in-transaction and statement timeouts, batched large updates, monitoring transaction age.
*Common weak answer:* "They use resources." The specific damage is locks held and cleanup blocked database-wide.

**Q15. Why is `UPDATE … SET stock = stock - 1 WHERE stock >= 1` safe at Read Committed?**
A strong answer must mention:
- An `UPDATE` locks the row and operates on the latest committed version.
- A concurrent `UPDATE` waits for the lock, then re-evaluates the `WHERE` on the new version (PostgreSQL) or reads the current version (InnoDB).
- The read and the write are one step: no gap for another transaction.
- The application must check the affected-row count.
*Common weak answer:* "Because the database is ACID." Atomicity alone does not explain it; the row lock and the re-check do.

**Q16. When would you use a constraint instead of an isolation level to protect an invariant?**
A strong answer must mention:
- Whenever the invariant can be expressed as `UNIQUE`, `CHECK`, foreign key, partial unique index or exclusion constraint.
- Constraints are enforced for every writer at every level, immediately per statement.
- They turn write skew of the "two conflicting inserts" kind into an error.
- Isolation, locks or Serializable for invariants a constraint cannot express.
*Common weak answer:* "Use Serializable for everything." Costs aborts everywhere for rules a constraint expresses for free.

---

## Scenario Exercises

1. A ticketing site lets users pick a seat; two users sometimes get the same seat. The code runs `SELECT status FROM seats WHERE id = $1`, checks it is free, then `UPDATE seats SET status = 'SOLD'`. The database is MySQL at its default level. Explain the anomaly and give two fixes, saying which you prefer for an interactive seat map.
2. A hospital scheduling service enforces "at least two nurses on every ward" in application code, at PostgreSQL Read Committed. A ward ended up with one nurse after two nurses swapped shifts at the same moment. Name the anomaly, show the timeline, and give three ways to prevent it.
3. A nightly reconciliation report sums account balances table by table at Read Committed and occasionally finds money missing or extra, although transfers are correct. Explain and fix.
4. After enabling Serializable globally in PostgreSQL, a service sees 5% of transactions fail with SQLSTATE `40001` at peak, and some customers get duplicate confirmation emails. Explain both observations and how to fix them.
5. A batch job updates the prices of 2,000 products in one transaction, in the order the CSV file lists them; at the same time, a second job adjusts stock for overlapping products in ID order. Deadlocks appear nightly. Explain and fix.
6. A table of user sessions in PostgreSQL has grown from 2 GB to 60 GB over a week although the number of active sessions is stable. Give the most likely mechanism, the evidence you would collect, and the fix.
7. Design a durable email-sending queue in PostgreSQL for 20 worker processes, guaranteeing that each email is sent at least once, preferably once, and that a crashed worker's emails are retried.

---

## Diagnosis Lab

**Lab 1: Checkouts stall behind one idle transaction (lock waits, idle in transaction)**
- *Symptom:* Checkout latency jumps from 80 ms to timeouts for a subset of carts; CPU is low; no deploy happened.
- *Hypotheses:* (a) a slow query; (b) lock waits; (c) the connection pool is exhausted.
- *Evidence:* The blocking-sessions query (Section 17.14) shows 37 sessions waiting with `wait_event_type = 'Lock'`, all blocked (directly or in a chain) by pid 4415: `idle in transaction` for 2 h 41 min, last query `SELECT * FROM carts WHERE id = $1 FOR UPDATE`. The application log shows that request path calls a recommendation service after locking the cart; a timeout exception there skips the code that commits, and the connection returns to the pool with the transaction still open.
- *Root cause:* (b): a transaction left open while holding a row lock, caused by an exception path that never commits or rolls back, plus a network call inside the transaction.
- *Fix (now):* `SELECT pg_terminate_backend(4415);` (its transaction rolls back, releasing the locks). *Fix (lasting):* move the network call out of the transaction; make transaction scope exception-safe (always commit or roll back in `finally`, or the framework's transaction template, Chapter 18); set `idle_in_transaction_session_timeout`.
- *Prevention:* Alert on transactions older than 60 seconds; `log_lock_waits = on`.

**Lab 2: Nightly deadlocks (deadlock log)**
- *Symptom:* Every night, the price-import job fails a few batches with `ERROR: deadlock detected`.
- *Evidence:* The server log shows both statements: process A runs `UPDATE products SET price = … WHERE id = 8812` while holding a lock on product 1307; process B runs `UPDATE products SET stock = … WHERE id = 1307` while holding 8812. The import updates products in CSV order; the stock-adjustment job updates them in ascending ID order.
- *Root cause:* two transactions acquiring row locks on the same products in different orders.
- *Fix:* Sort the IDs in the import before updating (ascending, like the other job); make batches smaller (500 products per transaction instead of 2,000); retry the deadlock victim.
- *Prevention:* A team rule that multi-row writes lock in primary-key order; deadlock rate on the database dashboard.

**Lab 3: Serialisation failures and double emails**
- *Symptom:* After switching the booking service to Serializable, 4% of bookings fail with `could not serialize access due to read/write dependencies among transactions`, and some customers receive two confirmation emails.
- *Evidence:* Failures cluster on bookings for the same venue and hour; `EXPLAIN` of the availability check shows a sequential scan of `bookings` (no index on `(venue_id, starts_at)`), so every transaction takes a table-level SIREAD lock and conflicts with any concurrent booking anywhere. The retry wrapper re-runs the whole booking function, which sends the confirmation email before committing.
- *Root cause:* coarse read tracking from a missing index, inflating conflicts; and a side effect inside the retried transaction.
- *Fix:* Add the index (now only bookings for the same venue and hour conflict); send emails after commit via an outbox (Chapter 25); keep the retry wrapper. Consider replacing the check with an exclusion constraint on `(venue_id, during)`, which removes the need for Serializable for this rule (Chapter 16, Section 16.6).
- *Prevention:* Monitor the 40001 rate; review that retried code has no side effects.

**Lab 4: Bloat from a forgotten replica query (vacuum blocked)**
- *Symptom:* The `sessions` table grows from 2 GB to 60 GB in a week; queries on it slow down; autovacuum runs constantly but `n_dead_tup` stays in the hundreds of millions.
- *Evidence:* On the primary, no session is older than a few minutes. `pg_stat_replication` shows the analytics replica's `backend_xmin` far in the past; `hot_standby_feedback = on` on that replica, and a dashboard query there has been running for six days.
- *Root cause:* the replica's long-running query holds back the xmin horizon of the primary through hot-standby feedback, so VACUUM on the primary cannot remove any dead version created since it began.
- *Fix:* Cancel the replica query; VACUUM can then clean up (the file returns to a normal size only after a rewrite with an online tool or `VACUUM FULL` in a maintenance window). Set a statement timeout for analytics users on the replica.
- *Prevention:* Alert on transaction age on replicas too, and on the gap between the newest XID and each replica's `backend_xmin`; run long analytics outside the replication chain (Chapter 15, Section 15.7).

---

## Solution Sketches

1. A lost update / write skew on the seat row: at InnoDB Repeatable Read, both plain `SELECT`s see "free", and both `UPDATE`s succeed (the second overwrites the first, no error). Fixes: a conditional update `UPDATE seats SET status = 'SOLD', buyer = $2 WHERE id = $1 AND status = 'FREE'` with a row-count check; or `SELECT … FOR UPDATE NOWAIT` to show "being booked by someone else" immediately. For an interactive seat map, the conditional update (fast, no lock held across user think time), optionally with a short "hold" row that expires, is preferable.
2. Write skew: both transactions count nurses on the ward (2), each removes a different nurse's assignment, and both commit, leaving one. Timeline as the on-call doctors. Preventions: run the swap at Serializable with retry; lock a common row (`SELECT … FROM wards WHERE id = $1 FOR UPDATE`) before counting, which serialises changes to that ward; or maintain a `nurse_count` column on the ward with `CHECK (nurse_count >= 2)` updated in the same transaction.
3. Read skew: each table, or each statement, is read at a different moment at Read Committed, so a transfer that commits between two reads is counted on one side only. Fix: run the whole report in one Repeatable Read transaction (one snapshot), or on a replica with a consistent snapshot; PostgreSQL's `SERIALIZABLE READ ONLY DEFERRABLE` also gives a safe snapshot for long reports.
4. Serialisation failures are expected at Serializable under contention; 5% suggests coarse SIREAD locks (scans without indexes) or genuinely hot data, so add indexes for the checks and narrow transactions. Duplicate emails mean the email is sent inside the retried transaction body: the first attempt sent it, then aborted, and the retry sent it again. Move side effects after commit, ideally through an outbox table written in the same transaction (Chapter 25).
5. The two jobs lock overlapping product rows in different orders (CSV order versus ID order), forming cycles. Fix: sort IDs in both jobs; reduce batch size so each transaction holds fewer locks for less time; retry deadlock victims.
6. Dead versions are not being removed: VACUUM is blocked by an old snapshot (a long transaction, an idle-in-transaction session, a replica with hot-standby feedback running a long query, an abandoned replication slot or prepared transaction). Evidence: `n_dead_tup`, last autovacuum, the oldest `xact_start`/`backend_xmin` in `pg_stat_activity` and `pg_stat_replication`, `pg_replication_slots`, `pg_prepared_xacts`. Fix: remove the blocker, let VACUUM clean, then rewrite the table online to return space; set timeouts and alerts.
7. An `emails` table with status, `run_at`, attempts, `locked_by`, `locked_at`, and a partial index on pending rows. Workers claim batches with `UPDATE … FROM (SELECT … FOR UPDATE SKIP LOCKED LIMIT n) RETURNING`, commit, send, then mark sent in a separate short transaction. A sweeper resets rows whose lease expired. Sending is at-least-once (a crash after sending but before marking sent resends), so include an idempotency key the email provider de-duplicates on (Chapter 13, Section 13.5), and cap attempts with a dead-letter status (Chapter 25).

---

**END OF CHAPTER 17**
