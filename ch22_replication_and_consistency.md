# Chapter 22: Replication and Consistency

## Where We Are

Up to now, the database has mostly been one server. Chapter 15 showed how that server makes a commit durable with its write-ahead log (Section 15.4), and noted in passing that the same log feeds replicas, and that a replica is not a backup (Section 15.9). Chapter 17 showed what long transactions do to hot standbys (Section 17.12). Chapter 18 split reads from writes, met replica lag and the read-your-writes problem for the first time, and pointed drivers at "the primary" after a failover (Section 18.16). Chapter 21 used asynchronous replication in Redis and warned that acknowledged writes can vanish on failover (Section 21.7).

This chapter is the home of **replication**: keeping copies of the same data on several machines, and of the **consistency** questions that copies raise. It explains why systems replicate, the three architectures (single-leader, multi-leader, leaderless), the anomalies that replication lag produces and how to prevent each, what goes wrong during failover (including the dreaded **split-brain**), how quorums work, the ladder of consistency models from linearizability down to eventual consistency, and the CAP theorem stated precisely, with PACELC for the trade-off that exists even when nothing is broken. The chapter ends with change data capture, replication into systems other than databases.

**After this chapter you can:**
- explain why systems replicate, and why replication is not backup;
- explain physical (log-shipping) and logical replication, and compare synchronous, semi-synchronous and asynchronous replication by durability, latency and availability;
- draw the read-your-writes, monotonic-reads and consistent-prefix anomalies and apply a fix for each;
- explain failure detection, election, promotion and fencing, and recognise split-brain;
- choose among single-leader, multi-leader and leaderless replication, and resolve write conflicts with last-write-wins, merging or CRDTs;
- reason about quorums (W + R > N), sloppy quorums, hinted handoff, read repair and anti-entropy;
- place a system on the consistency ladder (linearizable, sequential, causal, session guarantees, eventual) and state what each costs;
- state the CAP theorem correctly, debunk its common misreadings, and use PACELC;
- explain change data capture as replication into other systems;
- diagnose "I saved but don't see it", replica lag spikes and data lost in a failover.

**Interview questions this chapter prepares you for:**
1. Why replicate? Why is a replica not a backup?
2. How does database replication work? Physical versus logical replication?
3. Synchronous versus asynchronous replication: what are the trade-offs?
4. What is replication lag, and which anomalies does it cause? How do you fix each?
5. How does failover work? What can go wrong?
6. What is split-brain, and how is it prevented?
7. Leader–follower versus multi-leader versus leaderless replication: when would you use each?
8. How are write conflicts resolved in multi-leader systems? What is a CRDT?
9. How do quorums work? Does W + R > N guarantee strong consistency?
10. What is linearizability? How does it differ from eventual consistency?
11. What is causal consistency?
12. Explain the CAP theorem properly. What does it not say?
13. What is PACELC?
14. What is change data capture?

**Scope.** Here: reasons to replicate, single-leader replication and its synchrony modes, replication-lag anomalies, failover and split-brain, multi-leader and leaderless replication, conflict resolution, quorums and anti-entropy, consistency models, CAP and PACELC, change data capture as replication, replication diagnosis, and a placement-round section on distributed databases as textbooks present them (Section 22.12). Recalled, not re-taught: the WAL and crash recovery (Chapter 15, Sections 15.4–15.5), backups and PITR (Chapter 15, Section 15.9), transactions and isolation (Chapter 17), read/write splitting in applications (Chapter 18, Section 18.16), timeouts as unknown outcomes (Chapter 7, Section 7.15), memory models and sequential consistency (Chapter 5, Section 5.3). Elsewhere: partitioning, consistent hashing, clocks, consensus (Raft), distributed locks with fencing tokens and two-phase commit are Chapter 23; change data capture pipelines, the outbox and event-driven design are Chapter 25; deployment across regions is Chapter 26.

---

## 22.1 What Replication Is, and Why Systems Replicate

### Definitions

**Replication** is keeping a copy of the same data on several machines, called **replicas**, connected by a network, and propagating changes from one copy to the others. A **node** is one machine (or process) taking part.

- The replica that accepts writes in a single-leader design is the **leader** (primary, master); the others are **followers** (secondaries, replicas, standbys).
- **Replication lag** is the delay between a change being applied on one replica and on another.
- **Consistency**, in this chapter, means the guarantees a replicated system gives about what readers see across replicas and over time: whether a read reflects the latest write, writes in order, or eventually all writes. This is a different meaning from the "C" of ACID (Chapter 17, Section 17.2), which is about invariants.
- A **network partition** is a failure in which some nodes cannot communicate with others, while each group continues to run.

### Why it exists

One database server is a single point of failure, a single limit on read throughput, and a single location. Replication addresses four separate needs:

| Need | How replication helps | What it does not fix |
|---|---|---|
| **Availability** | if the leader fails, a follower can take over in seconds instead of restoring from backup in hours | failures that replicate (bad writes, corruption by the application) |
| **Read scaling** | read-only queries spread over followers (Chapter 18, Section 18.16) | write throughput (every replica applies every write) |
| **Latency and geography** | a replica in each region serves nearby users' reads | write latency for users far from the leader (single-leader) |
| **Durability** | a committed write exists on more than one machine before (synchronous) or soon after (asynchronous) it is acknowledged | logical mistakes, which are durably copied too |

### Concrete: replication is not backup

Chapter 15 (Section 15.9) stated the rule; here is the mechanism on one timeline:

```text
 10:00:00.000  operator runs on the primary:   DELETE FROM orders;    (meant for a test database)
 10:00:00.004  COMMIT                           primary: 0 orders
 10:00:00.010  replica 1 receives and applies the WAL   replica 1: 0 orders
 10:00:00.012  replica 2 (other region)                 replica 2: 0 orders
 10:00:30      someone notices. Every copy is faithfully, durably, identically empty.
 recovery: point-in-time restore from a backup + archived WAL to 09:59:59 (Chapter 15, Section 15.9),
           or a DELAYED replica that applies changes one hour late, stopped before 10:00:00
```

Replication copies **every** committed change, good or bad, within milliseconds; that is its purpose. A backup is a copy **from the past** that changes do not reach. They protect against different failures, and a system needs both.

### Guarantees and non-guarantees

Replication guarantees, depending on its mode, that committed data exists on several machines (immediately, or after a lag), and that the system can keep serving when one machine fails. It does not guarantee that replicas agree at any given instant, that a failover loses nothing (with asynchronous replication it can), that writes scale, or any protection against mistakes and corruption that are themselves replicated.

### Failure and diagnosis

- **"We have replicas, so we don't need backups."** Discovered after a destructive change or ransomware. Fix: backups with PITR, delayed replicas, restore drills (Chapter 15, Section 15.9).
- **Write bottleneck despite replicas.** Evidence: the leader's write I/O and CPU saturated while followers are idle. Cause: replication does not add write capacity. Fix: reduce write volume, or partition (Chapter 23).

### Backend connection

Decide which of the four needs you are replicating for, because each leads to different settings: availability needs fast, automated failover; read scaling needs lag-aware routing; geography needs regional replicas and care with write paths; durability needs synchronous (or quorum) acknowledgement. And keep backups regardless.

---

## 22.2 Single-Leader Replication: How Changes Flow

### Definitions

In **single-leader replication** (leader–follower, primary–replica), all writes go to one leader, which records them in a **replication log** and sends that log to followers, which apply it in the same order.

- **Physical replication** (log shipping, WAL shipping) sends the storage engine's own redo log (Chapter 15, Section 15.4): byte-level page changes. Followers become byte-for-byte copies of the leader. PostgreSQL's **streaming replication** is the standard example.
- **Logical replication** sends changes at the level of rows ("insert into `orders` this row", "update row 42 of `products`: price 1999 → 2499"), independent of storage layout. PostgreSQL's logical replication (publications and subscriptions) and MySQL's row-based **binary log** (binlog) are examples.
- **Statement-based replication** sends the SQL statements themselves; replaying them must produce the same result on every replica, which non-deterministic statements (`now()`, `random()`, auto-increment interplay, triggers) can break. MySQL defaults to row-based logging for this reason.
- A **replication slot** (PostgreSQL) records how far a consumer has received the log, so that the leader keeps the log it still needs.

### Why it exists

A single leader gives every write one order: the order of the leader's log. Followers that apply the same log in the same order converge to the same state, without any conflict resolution. That simplicity is why single-leader replication is the default for relational databases.

### Mechanism: the flow

```text
 client ── INSERT … COMMIT ──► LEADER
                                1. write WAL records, flush (Chapter 15, Section 15.4)
                                2. a WAL sender process streams new WAL to each follower
                                        │                              │
                                        ▼                              ▼
                                 FOLLOWER 1 (same zone)          FOLLOWER 2 (other region)
                                 receive → write → flush → replay  (same steps, later)
                                 reports back: written up to LSN X, flushed up to Y, replayed up to Z
```

A new follower starts from a **base backup** of the leader (a copy of its data files, Chapter 15, Section 15.9) plus the log from the backup's starting point, then streams continuously. Followers can usually serve read-only queries while they apply the log (PostgreSQL's hot standby).

### Mechanism: physical versus logical

| | Physical (WAL streaming) | Logical (row changes) |
|---|---|---|
| What is sent | page-level redo records | row inserts, updates, deletes per table |
| Replica state | an exact copy: same version, same indexes, same bloat | its own tables; can have different indexes, extra tables, a different major version |
| Subsets | whole database cluster only | per table or publication |
| Uses | high availability, read replicas, the basis of failover | migrations between versions, partial copies, feeding other systems (change data capture, Section 22.10) |
| DDL | replicated automatically | often not replicated (PostgreSQL logical replication does not replicate schema changes) |
| Cost and caveats | must match versions and platforms | sequences, large objects and DDL need separate handling; conflicts if the target is also written |

MySQL's replication is logical at the row level in its usual configuration (`binlog_format=ROW`, the default at the time of writing), with **global transaction identifiers** (GTIDs) naming each transaction so that a replica knows exactly which transactions it has applied, which simplifies failover.

### Guarantees and non-guarantees

Single-leader replication guarantees that followers apply writes in the leader's commit order, so a follower's state is always some **past** state of the leader (a consistent prefix of its history). It does not guarantee how far in the past (lag is unbounded with asynchronous replication), does not scale writes, and, for logical replication, does not carry everything (DDL, sequences) unless configured.

### Failure and diagnosis

- **Disk filling on the leader.** A replication slot of a disconnected follower or consumer retains WAL indefinitely (Chapter 15, Section 15.10). Evidence: `pg_replication_slots` with an inactive slot and large retained WAL.
- **Replica broken by non-determinism.** Statement-based replication producing different rows on replicas. Evidence: data differing between leader and replica. Fix: row-based logging.
- **Logical replication stops after a schema change.** Evidence: apply errors on the subscriber naming a missing column. Fix: apply DDL to subscribers first (expand/contract, Chapter 18, Section 18.15).

### Backend connection

Use physical replication for high availability and read replicas, and logical replication for upgrades, partial copies and feeding other systems. Monitor every slot and every follower's position, because a stuck consumer silently fills the leader's disk.

---

## 22.3 Synchronous, Semi-Synchronous and Asynchronous Replication

### Definitions

- **Asynchronous replication**: the leader acknowledges a commit to the client as soon as it is durable **locally**; followers receive it afterwards.
- **Synchronous replication**: the leader acknowledges a commit only after one or more followers have confirmed receiving (or durably storing, or applying) it.
- **Semi-synchronous replication**: a middle form; in MySQL, the leader waits until at least one replica has **received** the transaction (written it to its relay log), not until it has applied it.
- **Quorum commit**: the leader waits for a number of followers out of a set (for example any 1 of 2), so that one slow follower does not stall commits.
- The **data-loss window** is the set of acknowledged commits that exist only on the leader at a given moment, and would be lost if the leader died then.

### Why it exists

A commit can be "done" at several points: on the leader's disk, on a follower's network buffer, on a follower's disk, visible on a follower. Each later point protects against more failures and costs more latency. Choosing where the acknowledgement happens is choosing between write latency, durability and availability.

### Concrete: the commit-ack point and the loss window

```text
 ASYNCHRONOUS                                        SYNCHRONOUS (wait for follower flush)
 client   leader            follower                 client   leader            follower
   │ COMMIT ─►│                                        │ COMMIT ─►│
   │          │ flush WAL (local)                      │          │ flush WAL (local)
   │◄── OK ───│ ◄─ ACK POINT                            │          │── WAL ──────►│ receive, flush
   │          │── WAL ──────►│ (ms … minutes later)     │          │◄─ flushed ───│
   │          │              │ apply                    │◄── OK ───│ ◄─ ACK POINT
   │                                                    │
 leader dies between OK and the WAL arriving:         leader dies after OK: the follower
 the client was told "committed", the follower        already has the commit; promotion
 never got it → LOST after failover                   loses nothing acknowledged
 loss window: everything not yet received             loss window: none (for confirmed followers)
 commit latency: local flush only                     commit latency: local flush + network RTT
                                                       + follower flush
```

With a follower in the same zone (round trip 0.5 ms, flush 0.2 ms), synchronous commit adds about a millisecond. With a follower in another region (round trip 70 ms), it adds 70 ms or more to **every** commit, which is why cross-region replicas are usually asynchronous.

### Mechanism: the settings in PostgreSQL and MySQL

PostgreSQL (`synchronous_standby_names` lists which standbys are synchronous, for example `ANY 1 (replica_a, replica_b)` for a quorum; `synchronous_commit` chooses how far to wait):

| `synchronous_commit` | Acknowledge when | Survives |
|---|---|---|
| `off` | before even the local WAL flush (Chapter 15, Section 15.4) | nothing for the last moments |
| `local` | local WAL flushed | leader process crash, not leader loss |
| `remote_write` | synchronous standby has written the WAL to its OS (not flushed) | leader loss, unless both machines lose power |
| `on` (with synchronous standbys) | synchronous standby has **flushed** the WAL | leader loss |
| `remote_apply` | synchronous standby has **applied** it, so reads there see it | leader loss; also gives read-your-writes on that standby |

MySQL's semi-synchronous replication, in its lossless mode (wait point `AFTER_SYNC`), makes the commit visible to other clients only after a replica has acknowledged receipt, so a failover to that replica loses no acknowledged transaction. If no replica acknowledges within a timeout, MySQL falls back to asynchronous replication, quietly reopening the loss window: monitor for that fallback.

### Mechanism: synchronous replication and availability

If the only synchronous follower is down, a leader that insists on waiting cannot commit anything: the database becomes unavailable for writes because a **replica** failed. Remedies: quorum commit over several followers (`ANY 1 of 2`), automatic downgrade to asynchronous (which trades the guarantee for availability, and should alert), or three or more replicas so that one failure leaves a quorum. This is the first appearance of the trade-off that CAP and PACELC formalise (Section 22.9).

### Guarantees and non-guarantees

Synchronous (or lossless semi-synchronous) replication guarantees that every acknowledged commit exists on at least one confirmed follower, so promoting that follower loses nothing acknowledged. Asynchronous replication guarantees low commit latency and that followers apply writes in order eventually. None of them guarantees that reads on followers are current (except `remote_apply` on the synchronous standby), and synchronous replication does not guarantee write availability when synchronous followers are unreachable.

### Failure and diagnosis

- **Commit latency jumps.** Evidence: commit times equal to the round trip to a distant synchronous standby; PostgreSQL backends waiting on the `SyncRep` wait event. Fix: a nearer synchronous standby, quorum commit.
- **Writes hang.** Evidence: all commits waiting on `SyncRep`; the synchronous standby is down. Fix: quorum over several standbys; documented downgrade procedure.
- **Silent loss window.** MySQL semi-sync fell back to async after a timeout. Evidence: semi-sync status variables showing it off. Fix: alert on the fallback.

### Backend connection

Choose synchrony per data value: synchronous (in-zone or quorum) for data you cannot lose, asynchronous for remote replicas and read scaling, and per-transaction relaxation (PostgreSQL lets a transaction set `synchronous_commit`) for data that tolerates loss. Know your loss window in seconds, because after an asynchronous failover it is exactly what disappears.

---

## 22.4 Replication Lag and Its Anomalies

### Definitions

With asynchronous followers, readers of a follower see a state that is **behind** the leader. Three named anomalies result, each with a matching guarantee (the **session guarantees** of Section 22.8):

- **Read-your-writes** (read-after-write consistency): a user always sees their own earlier writes. Violated when a user writes to the leader and then reads from a lagging follower.
- **Monotonic reads**: a user who has seen a value never later sees an **older** one. Violated when consecutive reads go to followers with different lag.
- **Consistent prefix reads**: a reader sees writes in an order that respects causality; if write B was made because of write A, nobody sees B without A. Violated in partitioned or multi-leader systems where related writes travel through different paths.

### Why it matters

Lag is usually milliseconds, so these anomalies are rare in testing and common in production: under load, during a large transaction, or after a replica restarts. Users experience them as "my change didn't save", "the comment appeared and disappeared", or "the reply showed up before the question", and they generate support tickets and duplicate submissions.

### Concrete: read-your-writes and its fix

Chapter 18 (Section 18.16) met this anomaly in an application routing reads to replicas. The timeline, and the fixes in full:

```text
 user          leader                      follower (lag 1.2 s)
 save bio ───► UPDATE users SET bio='…' COMMIT (t=0)
 reload  ─────────────────────────────────► SELECT bio → old bio      ✘ (t=0.3 s)
                                             applies the update at t=1.2 s

 FIXES
 a) read from the leader for data the user may have modified (their own profile), or for N seconds
    after any write by this user (tracked in the session, Chapter 19, Section 19.3)
 b) remember the leader's log position after the write (pg_current_wal_lsn() = X, or the GTID);
    route the read to a follower that has replayed ≥ X, or wait briefly for it, or use the leader
 c) synchronous_commit = remote_apply on the replica that serves this user's reads (Section 22.3)
```

Across devices (a phone and a laptop), the "remember the position" approach needs the position stored server-side with the user, not in one device's cookie.

### Concrete: monotonic reads and its fix

```text
 a comment is posted at t=0
 user refreshes at t=0.5 → follower A (lag 0.1 s) → sees the comment
 user refreshes at t=0.8 → follower B (lag 2 s)   → the comment is GONE   ✘ (time went backwards)

 FIX: pin each user (session) to one follower, for example by hashing the user ID to a replica,
      or track the highest position the user has seen and read only from replicas at or beyond it
```

If the pinned replica fails, the user moves to another and may see an older state once; the position-tracking fix avoids even that.

### Concrete: consistent prefix and its fix

The anomaly needs data that travels through different paths, typically different partitions (Chapter 23) replicated independently:

```text
 partition P1 holds Ana's message, partition P2 holds Ben's reply; each has its own leader and lag
 t=0.0  Ana:  "Is the store open on Sunday?"     → P1 leader
 t=0.4  Ben:  "Yes, 10 to 4."                    → P2 leader (Ben read Ana's message first)
 observer reads P1's follower (lag 1 s) and P2's follower (lag 0.1 s) at t=0.6:
        sees "Yes, 10 to 4."  but not the question    ✘ (effect before cause)

 FIXES: keep causally related writes in the same partition (one conversation, one partition), so
        one log orders them; or carry causal dependencies (Ben's write records that it depends on
        Ana's), and hold back a write until its dependencies are visible (causal consistency, 22.8)
```

### Mechanism: why lag happens and how large it gets

Lag is the time a change spends being sent, written and applied. It grows when:

- the leader produces WAL faster than a follower can apply it (a bulk update, Chapter 18, Section 18.14; an index build);
- the follower is busy (heavy read queries competing for I/O and CPU) or applies the log on one thread while the leader wrote it from many;
- the follower **pauses replay** to avoid cancelling a long query that needs rows the replay would remove (PostgreSQL's recovery conflicts, Chapter 17, Section 17.12);
- the network between them is slow or saturated, or the follower was offline and is catching up.

With asynchronous replication there is no upper bound: a follower can be hours behind and still report itself healthy unless something checks.

### Guarantees and non-guarantees

The techniques above guarantee their specific session property for the user or session that uses them. Plain asynchronous followers guarantee only that each follower shows a consistent prefix of the leader's history (single-leader, one log), at an unknown delay; they guarantee none of the three properties across followers or across partitions.

### Failure and diagnosis

- **"I saved but don't see it."** Evidence: the write committed on the leader; the read hit a follower whose replay position was behind the write's position. Diagnosis Lab 1.
- **Values that flicker.** Evidence: consecutive reads of the same user served by different replicas with different lag. Fix: session pinning.
- **Lag spikes.** Evidence: `replay_lag` or `Seconds_Behind_Source` rising during batch jobs or long replica queries. Diagnosis Lab 2.

### Backend connection

For each read path served by followers, decide which session guarantee it needs: read-your-writes for a user's own data, monotonic reads for anything users refresh, consistent prefix for conversations and ordered histories. Implement them with leader reads, position tracking, session pinning or co-located partitions, and put replica lag on a dashboard with an alert, because nothing else bounds it.

---

## 22.5 Failover, Split-Brain and Fencing

### Definitions

- **Failover** is replacing a failed leader with a follower: detecting the failure, choosing a new leader, **promoting** it (letting it accept writes), and redirecting clients and the other followers to it.
- **Failure detection** decides that a node is dead, almost always by a **timeout**: no heartbeat or response within a set time.
- **Leader election** is how the remaining nodes agree on which one becomes leader. Doing this safely requires a **consensus** algorithm or an external coordinator that uses one (Raft is Chapter 23's subject).
- **Split-brain** is a state in which two nodes both believe they are the leader and both accept writes, so the data diverges.
- **Fencing** is preventing a node that is no longer leader from acting as one: cutting its power or network access (**STONITH**, "shoot the other node in the head"), revoking its access to storage, or making every write carry an **epoch** (term, generation) number that receivers check, so that writes from an older leader are rejected.

### Why it matters

Failover turns hours of restore into seconds of unavailability, but it is also where replicated databases lose data and corrupt themselves. A timeout cannot distinguish "the leader is dead" from "the leader is slow" or "the network between us is broken" (the unknown outcome of Chapter 7, Section 7.15, applied to nodes). Act too eagerly and you promote a second leader while the first is still running; act too cautiously and every hiccup becomes a long outage.

### Mechanism: the steps of a failover

```text
 1. DETECT     heartbeats from the leader stop for longer than the timeout (e.g. 10–30 s)
 2. DECIDE     a majority of observers (a coordinator cluster such as etcd, Consul or ZooKeeper,
               or the replicas themselves via consensus) agrees that the leader is gone,
               and which follower is most up to date (highest replicated log position)
 3. FENCE      make sure the old leader cannot accept writes: power off, revoke its storage or
               network, or take away its leader lease (it must stop writing when the lease expires)
 4. PROMOTE    the chosen follower stops replaying and starts accepting writes (new epoch / timeline)
 5. REDIRECT   other followers replicate from the new leader; clients find it (DNS, a virtual IP,
               a proxy, or drivers given several hosts with "primary only", Chapter 18, Section 18.16)
 6. REJOIN     the old leader, when it returns, becomes a follower; any writes it took that were
               never replicated must be discarded or reconciled (PostgreSQL's pg_rewind rewinds it
               to the point where the histories diverged)
```

Tools such as Patroni (PostgreSQL) and Orchestrator (MySQL), and managed cloud databases, automate these steps. Patroni, for example, keeps a **leader key with a time-to-live** in a consensus store: the leader must keep renewing it, and a node that cannot renew it demotes itself.

### Concrete: split-brain, then fencing

```text
 WITHOUT FENCING
 t=0    leader L (epoch 7) serves writes; follower F replicates asynchronously
 t=1    network partition: the monitor and F cannot reach L; clients in L's zone still can
 t=31   timeout: the monitor promotes F to leader (epoch 8); some clients now write to F
 t=31…  L does not know it was replaced; clients in its zone keep writing to it
        L: order 501 → customer 81          F: order 501 → customer 44     (same id, two histories)
 t=300  partition heals: two leaders, diverged data; one side's writes must be thrown away
        or merged by hand

 WITH FENCING (lease + epochs)
 t=0    L holds a leader lease in the coordinator, renewed every 3 s, valid 10 s
 t=1    partition: L can no longer renew
 t=11   L's lease expires: L DEMOTES ITSELF to read-only before any new leader can exist
 t=31   the coordinator grants the lease to F, epoch 8, only after L's lease is certainly over
        storage and replicas reject any write tagged with epoch 7
 t=300  partition heals: L rejoins as a follower; no second history exists
```

The lease approach needs clocks that run at about the same **rate** on both sides (not the same time), and a safety margin for pauses: a leader that is frozen for 15 seconds (a long GC pause, a VM migration) can wake up still "believing" it holds a lease. That is why the strongest designs also put the **epoch** on every write and make the storage or the followers reject old epochs, the same fencing-token idea that Chapter 23 develops for distributed locks.

### Mechanism: what an asynchronous failover loses

If the old leader had acknowledged commits that no follower had received (Section 22.3), promoting a follower loses them. Worse, they can come back in confusing ways: the new leader reuses identifiers that the lost transactions had used (an auto-increment value, an order number already emailed to a customer), and external systems that already saw the lost writes (a cache, a message broker, a payment provider) now disagree with the database. Diagnosis Lab 3 is this failure. Remedies: synchronous or quorum replication for data you cannot lose; identifiers that do not depend on the database's counters (UUIDs, Chapter 14, Section 14.2); and reconciliation with external systems after a failover.

### Guarantees and non-guarantees

A failover with consensus-based election and fencing guarantees at most one leader accepting writes at any time, and that the new leader is the most up-to-date available follower. It does not guarantee zero data loss unless replication to that follower was synchronous, does not guarantee a short outage (detection timeouts plus promotion take seconds to minutes), and does not guarantee that clients find the new leader promptly (DNS caching, connection pools holding old connections).

### Failure and diagnosis

- **Two primaries.** Evidence: both nodes report `pg_is_in_recovery() = false` (PostgreSQL), or writes appearing on two nodes; conflicting rows with the same keys after the partition heals. Cause: failover without fencing. Fix: lease-based or consensus-based tooling, fencing at the storage or network level.
- **Flapping failovers.** Evidence: repeated promotions during short network blips or load spikes. Cause: timeouts too short. Fix: longer timeouts, require several missed heartbeats, rate-limit failovers.
- **Clients writing to a demoted primary.** Evidence: `cannot execute UPDATE in a read-only transaction` (Chapter 18, Section 18.16). Fix: drivers with primary-only targeting, pool eviction on that error.

### Backend connection

Never build failover by hand with a script that pings the leader and promotes a follower: use tooling that elects through consensus and fences the old leader. Choose detection timeouts knowing that every second of timeout is a second of write unavailability, and every second too short risks a false failover. Design identifiers and external side effects to survive the loss of the last few seconds of writes, because asynchronous failover will eventually lose them.

---

## 22.6 Multi-Leader Replication and Conflict Resolution

### Definitions

In **multi-leader replication** (multi-primary, active–active), several nodes accept writes, and each replicates its changes to the others asynchronously.

- A **write conflict** is two leaders accepting concurrent changes to the same data, neither having seen the other's change.
- **Conflict resolution** decides the final value: automatically (**last write wins**, merging) or by the application or the user.
- **Last write wins** (LWW) keeps the change with the highest timestamp (or another total order) and discards the others.
- A **CRDT** (conflict-free replicated data type) is a data type whose concurrent updates can always be merged automatically, in any order, to the same result, because its merge operation is commutative, associative and idempotent.

### Why it exists

Single-leader replication sends every write to one place. That is a problem when users are spread across continents (every write pays the round trip to the leader's region), when a region must keep accepting writes while cut off from the others, or when clients work offline and synchronise later (mobile apps, collaborative editors, calendars). Multi-leader replication lets each region, device or editor write locally, and pays for it with conflicts.

### Mechanism: how conflicts arise

```text
 leader EU                                   leader US
 user edits the title of document 7:          another user edits the same title, at the same moment:
   "Q3 plan" → "Q3 plan (draft)"                "Q3 plan" → "Q3 roadmap"
 commits locally; replicates to US ──────┐  ┌── commits locally; replicates to EU
                                         ▼  ▼
                              each leader receives a change to a value it has ALSO changed
                              → which title wins? both? neither?
```

In a single-leader system, the second write would have waited behind the first, or failed. Here both have already been accepted and acknowledged.

### Mechanism: resolving conflicts

| Approach | How | Cost |
|---|---|---|
| **Avoid** | route all writes for a record to one "home" leader (a user's region), so conflicts happen only during failover | the original latency problem for users far from their home region |
| **Last write wins** | attach a timestamp (or a unique, ordered ID) to each write; keep the highest | **silently discards** the other writes; clock skew decides the winner (Chapter 23) |
| **Merge** | combine values: union of sets, concatenation, field-by-field (title from one, body from the other) | only works where a meaningful merge exists |
| **Record both** | keep all concurrent values ("siblings") and let the application or user resolve on read | application complexity; a UI for conflicts |
| **CRDTs** | data types designed so concurrent updates always merge to the same result | only certain types; metadata overhead |

**CRDTs at intuition level:**

```text
 G-COUNTER (grow-only counter, e.g. page views), replicas A and B
 state = one counter per replica: {A: 0, B: 0}; value = sum
 A increments 3×: {A: 3, B: 0}     B increments 2× concurrently: {A: 0, B: 2}
 merge = element-wise MAX: {A: 3, B: 2} → value 5, whichever order the merges happen, however
         often they are repeated
 (a PN-counter pairs two G-counters, increments and decrements, for counters that go down)

 OR-SET (observed-remove set, e.g. a shopping cart)
 every add gets a unique tag: add(milk#a1); a remove deletes only the tags it has SEEN
 concurrent: replica A removes milk (it saw #a1); replica B adds milk again (#b7)
 merge: #a1 removed, #b7 kept → milk is in the cart ("add wins" over a concurrent remove)
```

Collaborative editors use sequence CRDTs or operational transformation to merge concurrent text edits character by character. The general lesson: automatic resolution is safe only for data whose merge rule matches the business meaning, and LWW is a merge rule that means "lose data".

### Guarantees and non-guarantees

Multi-leader replication guarantees local write latency and continued writes in each region during partitions. With CRDTs it guarantees **convergence**: replicas that have received the same updates reach the same state. It does not guarantee that a write is final when acknowledged (it may lose an LWW contest later), does not preserve invariants that span records ("at most one booking per seat" cannot be enforced by two leaders that cannot talk, Chapter 17, Section 17.3), and does not order writes across leaders.

### Failure and diagnosis

- **Lost updates under LWW.** Evidence: a user's change "reverts" after a few seconds; the replication log shows two concurrent writes with close timestamps. Fix: avoid conflicts by home-region routing, or use merges or CRDTs.
- **Duplicate unique values.** Two leaders both accept the same username or order number. Fix: allocate unique values per leader (ID ranges, prefixes), or route uniqueness-critical writes to one leader.
- **Clock skew choosing winners.** Evidence: writes from one node always win; its clock is ahead. Fix: logical or hybrid clocks (Chapter 23).

### Backend connection

Prefer single-leader replication, with conflict avoidance by home region, unless local writes across regions or offline operation are real requirements. When you need multi-leader, decide the conflict rule per data type, use CRDTs where they fit (counters, sets, collaborative documents), never use last-write-wins for data whose loss matters, and keep invariants that need coordination on a single leader.

---

## 22.7 Leaderless Replication and Quorums

### Definitions

In **leaderless replication** (Dynamo-style, after Amazon's Dynamo paper of 2007; used by Cassandra and ScyllaDB, among others), there is no leader: a client (or a coordinator node acting for it) sends each write to several replicas and reads from several, and the system treats an operation as successful when enough of them respond.

- **N** is the number of replicas that store each key; **W** the number that must acknowledge a write; **R** the number that must answer a read.
- A **quorum** is a set of replicas large enough that any write quorum and any read quorum overlap: **W + R > N**.
- **Read repair**: when a read finds that some replicas returned an older version, the client or coordinator writes the newest version back to them.
- **Anti-entropy**: a background process that compares replicas and copies missing data. **Merkle trees** (hash trees) make the comparison cheap.
- A **sloppy quorum** accepts writes on reachable nodes that are not among the key's N home replicas when the home replicas are unreachable; **hinted handoff** later forwards those writes to the home replicas.
- A **gossip protocol** (epidemic protocol) spreads information through a cluster by having each node periodically exchange what it knows with a few randomly chosen peers. Leaderless stores use it to share cluster membership, which node owns which key range, and which nodes appear to be down.

### Why it exists

A leader is a bottleneck and a failover risk. Without one, any replica can serve any request, a slow or failed node is simply not waited for, and the system keeps working as long as enough replicas respond. This suits workloads that need very high write availability across machines and data centres and can tolerate weaker guarantees than a single leader gives.

### Concrete: N = 3, W = 2, R = 2

```text
 key "cart:81" lives on replicas A, B, C (N = 3)

 WRITE v2 (W = 2)                               READ (R = 2)
 client ──► A ✔ ack                              client ──► B: v2 (version 2)
        ──► B ✔ ack     → success after 2 acks          ──► C: v1 (version 1)   → return the newest, v2
        ──► C ✘ (down; misses v2)                        and READ-REPAIR: write v2 to C

 why it works: the 2 replicas that acknowledged the write and the 2 that answered the read
 must share at least one node, because 2 + 2 > 3:

      write set {A, B}        read set {B, C}       overlap {B} has the newest version
      ┌───┐ ┌───┐ ┌───┐
      │ A │ │ B │ │ C │       any 2-of-3 write set and any 2-of-3 read set intersect
      └───┘ └───┘ └───┘
```

Versions must be comparable to know which value is newest: a version number per key, or a **version vector** (one counter per replica) that can also detect that two versions are **concurrent**, in which case both are kept as siblings (Section 22.6).

Choices of W and R trade latency and availability:

| Setting (N = 3) | Write needs | Read needs | Character |
|---|---|---|---|
| W = 3, R = 1 | all replicas | one | fast reads; writes fail if any replica is down |
| W = 2, R = 2 | majority | majority | balanced; tolerates one failure for both |
| W = 1, R = 3 | one | all | fast writes; reads fail if any replica is down |
| W = 1, R = 1 | one | one | fastest, most available; **W + R = 2 ≤ 3: reads may miss writes** |

Cassandra exposes this per request as a **consistency level**: `ONE`, `QUORUM`, `LOCAL_QUORUM` (a majority within the local data centre), `ALL`, and others.

### Mechanism: why W + R > N is not "strong consistency"

The overlap guarantees that a read **can** see the latest acknowledged write, not that the system behaves like a single copy. Several edge cases break that intuition:

- **Concurrent writes** to the same key are both "latest"; resolution (LWW or siblings) decides, and LWW can drop one.
- **A failed write is not rolled back.** If a write reached only 1 of the 2 replicas it needed, the client sees a failure, but the one replica keeps the value, and later reads may or may not return it.
- **A read concurrent with a write** may return the new value from one replica and, a moment later, another read the old value: reads are not linearizable (Section 22.8) without extra steps (for example, read repair completing before the read returns).
- **Sloppy quorums** break the overlap: during a partition, writes are accepted by nodes outside the key's home set, so a read quorum of home replicas can miss them until hinted handoff delivers them.
- **Node replacement** from a stale replica can reduce the number of copies holding the latest value below W.

So quorums give a tunable, probabilistic freshness, plus availability; systems that need linearizable operations use consensus (Chapter 23) or offer special modes (Cassandra's lightweight transactions use a Paxos-based protocol).

### Mechanism: read repair and anti-entropy with Merkle trees

Read repair fixes only keys that are read. For the rest, replicas periodically compare their data. Comparing every key would be expensive, so each replica builds a **Merkle tree** over a key range: leaves are hashes of key ranges, each parent the hash of its children.

```text
                    root  H(H12, H34)              compare roots: equal → ranges identical, done
                   /                \              different → compare children, descend only
             H12 = H(H1,H2)    H34 = H(H3,H4)              into subtrees whose hashes differ
             /      \           /      \
           H1       H2        H3       H4          only the differing leaf ranges' keys are
         keys a–f  g–m      n–s      t–z           exchanged and repaired
```

With a tree of depth d, two replicas that differ in one key range find it after comparing about d hashes on the path, instead of every key.

### Mechanism: membership and failure detection by gossip

Without a leader, every node must still know which other nodes exist, which key ranges they own, and which seem to be down, so that it can choose the N replicas for a key and skip dead ones. A central registry would be a single point of failure, so leaderless stores such as Cassandra spread this state by gossip:

```text
 every second, each node:
   1. increments its own heartbeat counter
   2. picks one to three random peers and exchanges its table {node → (heartbeat counter, status, key ranges)}
   3. keeps, for every node, whichever entry has the higher heartbeat counter
 a change known to one node reaches all N nodes in about log₂ N rounds, because each round roughly
 doubles the number of nodes that know it: 1,000 nodes → about 10 rounds
 a node whose heartbeat counter has stopped increasing is SUSPECTED to be down
```

Instead of a fixed timeout, Cassandra uses a **phi accrual failure detector**: for each peer it records the usual interval between heartbeat updates, and outputs a suspicion level (phi) that rises the longer the current silence lasts compared with that usual interval. An operator chooses the threshold at which a node is treated as down, trading fast detection against false alarms on a slow network. Gossip guarantees that membership information eventually reaches every node and keeps working when any node fails. It does not give all nodes the same view at the same instant: two nodes can briefly disagree about whether a third is up, which is one reason coordinators write to other nodes with hinted handoff rather than refusing writes.

### Guarantees and non-guarantees

Leaderless replication with W + R > N guarantees that a successful read quorum includes at least one replica that acknowledged the latest **successful** write, under normal (strict) quorums. It guarantees high availability as long as W (or R) replicas respond. It does not guarantee linearizability, a single order of concurrent writes, rollback of partially failed writes, or overlap under sloppy quorums. Anti-entropy guarantees eventual convergence of replicas, not when.

### Failure and diagnosis

- **Resurrected deletes.** A deleted value reappears. Cause: a replica missed the delete (a **tombstone**, Chapter 15, Section 15.6) and the tombstone was garbage-collected before repair reached that replica, so anti-entropy copied the old value back. Fix: run repairs more often than the tombstone grace period.
- **Reads returning old data at `ONE`.** Evidence: consistency level `ONE` for reads and writes. Fix: `QUORUM` (or `LOCAL_QUORUM`) for data that must be fresh.
- **Latency set by the slowest of R or W replicas.** Evidence: p99 tracks the slowest node. Fix: speculative retries to another replica, lower levels where acceptable.

### Backend connection

Choose N, W and R per operation from the freshness and availability each needs, prefer `LOCAL_QUORUM` in multi-region clusters to avoid cross-region latency on every request, and schedule anti-entropy repairs. Do not build invariants that need a single order (uniqueness, balances) on quorum reads and writes alone.

---

## 22.8 The Consistency Models Ladder

### Definitions

A **consistency model** is a contract that states which results reads may return, given the history of operations in a replicated (or concurrent) system. From strongest to weakest:

- **Linearizability** (atomic consistency, Herlihy and Wing, 1990): the system behaves as if there were **one copy** of the data and every operation took effect atomically at some instant between its start and its end. Once a write completes, every later read, by anyone, returns it or something newer.
- **Sequential consistency** (Lamport, 1979; Chapter 5, Section 5.3 used it for memory): all operations appear in **some** single order that respects each client's own order, but that order need not match real time across clients.
- **Causal consistency**: operations that are causally related (one could have influenced the other: a reply after the message it answers, a read before a write that depended on it) are seen in that order by everyone; concurrent operations may be seen in different orders by different readers.
- **Session guarantees** (read-your-writes, monotonic reads, monotonic writes, writes-follow-reads): per-client promises that make one user's view sensible (Section 22.4).
- **Eventual consistency**: if writes stop, all replicas eventually return the same value. It promises nothing about what reads return in the meantime.

### Why it matters

"Strongly consistent" and "eventually consistent" are often used as the only two options, and loosely. The models in between are what most real systems provide, and the differences decide concrete bugs: whether a username reserved by one request is visible to the next, whether a reply can appear before its question, whether a user sees their own edit. Stronger models need more coordination between replicas, which costs latency and availability (Section 22.9).

### Concrete: the ladder, with one history

```text
 STRONGER, more coordination, higher latency, less available during partitions
   │  linearizable      one copy, real-time order
   │  sequential        one order for everyone, consistent with each client's order (not real time)
   │  causal            cause before effect for everyone; concurrent ops in any order
   │  session guarantees each client's own view is sensible (RYW, monotonic reads)
   ▼  eventual          converges when writes stop; anything goes meanwhile
 WEAKER, less coordination, lower latency, available during partitions

 history (x starts at 0):
   client A:  write x = 1  [10:00:00.000 – 10:00:00.050, completes]
   client B:  read x       [10:00:00.100 – 10:00:00.120]  → returns 0
   client C:  read x       [10:00:00.200] → 1, then read x [10:00:00.300] → 0

                                        linearizable  sequential  causal   eventual
 B reads 0 AFTER A's write completed       ✘ forbidden   ✔ allowed   ✔        ✔
   (real time says A finished first; a sequential order "B read, then A wrote" is still valid)
 C reads 1 then 0 (goes back in time)      ✘             ✘           ✘ (it saw the write,      ✔
                                                                        must not un-see it)
```

A second example separates causal from eventual:

```text
 Ana posts "Is the store open?" (write m1); Ben reads m1 and replies "Yes" (write m2, caused by m1)
 Chen reads and sees m2 without m1          causal: ✘ forbidden      eventual: ✔ allowed
 Dee sees two unrelated posts in either order: allowed by both (they are concurrent)
```

### Mechanism: what each model costs

| Model | What it needs | Cost | Typical systems (single-key or general) |
|---|---|---|---|
| Linearizable | coordination on every write and on reads (a single leader read, or a quorum with consensus) | latency of a round trip to a majority, often across regions; unavailable on the minority side of a partition | consensus stores (etcd, ZooKeeper reads with sync), single-leader databases reading from the leader, Spanner-style systems |
| Sequential | a single total order of operations | similar to linearizable in practice for replicated stores | rarely offered as a distinct product guarantee |
| Causal | tracking dependencies (version vectors, causal metadata) and delaying visibility until dependencies are present | metadata; some delay; **available** during partitions | several geo-distributed databases offer causal sessions (for example MongoDB's causally consistent sessions) |
| Session guarantees | client-side tokens (positions seen) or sticky routing | small | applications on async replicas (Section 22.4); DynamoDB-style "consistent read" options |
| Eventual | none beyond eventual propagation | lowest | DNS caches, CDNs, async replicas, many NoSQL defaults |

Linearizability matters when correctness depends on a **single, current** answer: acquiring a lock or leadership (Section 22.5), enforcing uniqueness (two users registering the same name), a balance check before spending. Causal consistency is enough for most social and collaborative features and survives partitions. Eventual consistency suits caches, counters, recommendations and anything whose staleness is harmless.

### Guarantees and non-guarantees

Each model guarantees exactly the histories it allows and nothing more; a system's documentation should state which model each operation provides. No model implies durability (a linearizable system can still lose data in a disaster), and "eventual" gives no bound on how long convergence takes. Note that linearizability is about **single operations on single objects** in real time; serialisability (Chapter 17, Section 17.2) is about **transactions** over many objects in some serial order. A system can offer one without the other; offering both is called strict serialisability.

### Failure and diagnosis

- **Uniqueness violated in an eventually consistent store.** Two accounts with the same username, registered through different replicas. Fix: a linearizable operation for the claim (a single leader, a conditional write with consensus, or a database unique constraint, Chapter 14, Section 14.3).
- **Effects before causes.** Replies before questions, deletions before creations. Fix: causal consistency (co-location, dependency tracking).
- **Confusing linearizable with serialisable.** A design that assumes linearizable single-key operations give multi-row transactional safety. Fix: transactions where invariants span keys (Chapter 17).

### Backend connection

For each piece of data, state the weakest model that is still correct, because each step up the ladder costs latency and availability. Use linearizable operations only for the decisions that need them (locks, uniqueness, leader election, money movements), causal or session guarantees for user-facing views, and eventual consistency for caches and aggregates.

---

## 22.9 The CAP Theorem, Precisely, and PACELC

### Definitions

The **CAP theorem** (conjectured by Eric Brewer in 2000, proved by Seth Gilbert and Nancy Lynch in 2002) concerns three properties of a replicated data store:

- **C, consistency**, in CAP's sense, means **linearizability** (Section 22.8): every read returns the most recent completed write.
- **A, availability**, in CAP's sense, means **every request received by a non-failed node gets a non-error response**, without waiting indefinitely.
- **P, partition tolerance**, means the system continues to operate while the network loses messages between nodes.

The theorem: **when a network partition occurs, a system must choose between consistency and availability** for the requests it receives. It cannot provide both linearizability and availability during a partition.

**PACELC** (Daniel Abadi, 2010) extends this: if there is a **P**artition, choose between **A**vailability and **C**onsistency; **E**lse (normal operation), choose between **L**atency and **C**onsistency.

### Why it exists

Partitions are not optional: networks drop packets, links fail, switches reboot, and cloud zones lose connectivity. When two groups of replicas cannot talk, a request arriving on one side faces a dilemma. If it answers, it may return data that the other side has already changed (not linearizable). If it refuses or waits, it is not available. CAP names that dilemma. PACELC adds the everyday version: even without a partition, keeping replicas linearizable requires waiting for coordination, which costs latency.

### Concrete: the same request during a partition

```text
 3 replicas of an account balance: A and B in zone 1, C in zone 2; the network between the zones fails
 a client in zone 2 sends to C:  "withdraw 100 if balance ≥ 100"   (balance was 150; a withdrawal
                                                                     of 120 just committed in zone 1)

 CP behaviour (prefer consistency)                AP behaviour (prefer availability)
 C cannot reach a majority (A, B)                 C answers from its local copy
 → C refuses: 503 / "try later" or waits          → balance looks like 150 → withdraw 100 → OK
   until the partition heals                       → after healing: balance 150 − 120 − 100 = −70
 correct, but unavailable to zone-2 clients          available, but the invariant broke: a conflict
                                                     to resolve (Section 22.6) or compensate
 zone-1 clients (majority side) keep working       both zones kept working
```

For an account balance, CP behaviour is the obvious choice. For a shopping cart or a "likes" counter, AP behaviour (accept, merge later) is often better, which is why Dynamo was designed that way. The choice can be made per operation and per data type, not per system.

### Mechanism: what CAP does not say

| Misconception | What is actually true |
|---|---|
| "Pick any two of three." | P is not optional for a distributed system; the choice C-or-A arises **only during a partition**. Without one, a system can be both consistent and available. |
| "CA systems exist." | A single-node database is "CA" only because it has no partitions to tolerate; replicate it and the question returns. |
| "This database is AP" (or CP) as a fixed label. | Most systems offer per-operation choices (quorum levels, read-from-leader options, consistency flags), and behave differently in different failures. |
| "CP systems are never available." | CP systems stay available on the **majority** side of a partition; only the minority side refuses. |
| "C in CAP is ACID's C." | CAP's C is linearizability; ACID's C is invariant preservation (Chapter 17, Section 17.2). |
| "CAP covers latency." | It does not. PACELC adds the latency–consistency trade-off in normal operation. |
| "Eventual consistency is the AP choice." | Eventual consistency is the weakest of many AP-compatible models; causal consistency is also achievable with availability. |

### Mechanism: PACELC, the trade-off without a partition

```text
 normal operation, a write in region EU, replicas in EU and US (round trip 80 ms)

 EC (consistency)                                   EL (latency)
 wait for the US replica (or a majority) before     acknowledge after the local write; replicate
 acknowledging: +80 ms on every write;               asynchronously: fast writes; US readers may
 every read anywhere sees it                         see old data for a while
```

Systems are often summarised as PA/EL (favour availability in partitions and latency otherwise: Dynamo-style stores at low consistency levels), PC/EC (favour consistency in both: consensus-based databases), or mixed (PA/EC, PC/EL), but as with CAP, the honest description is per operation and per configuration.

### Guarantees and non-guarantees

CAP guarantees an impossibility: no system provides linearizability and total availability during a partition. It says nothing about what a system does when there is no partition, about latency, about weaker consistency models, or about durability. PACELC states that consistency costs latency even in good times; it does not quantify it.

### Failure and diagnosis

- **Unavailability on the minority side.** Evidence: during a zone partition, a CP store's nodes in the smaller zone return errors or time out. Expected behaviour; fix with clients that fail over to the majority side, and with a replica layout where a majority survives the likely failure (three zones, not two).
- **Inconsistency after a partition heals.** Evidence: conflicting values or broken invariants created on both sides. Expected for AP choices; needs conflict resolution or compensation designed in advance (Section 22.6; sagas in Chapter 25).

### Backend connection

State, per operation, what it should do when replicas cannot communicate: refuse (CP) or accept and reconcile later (AP), and what consistency it needs in normal operation at what latency (PACELC). Put replicas in at least three failure domains so that a single failure leaves a majority, and never describe a system as simply "CP" or "AP" without saying for which operations and settings.

---

## 22.10 Change Data Capture as Replication into Other Systems

### Definitions

**Change data capture** (CDC) is reading a database's committed changes (inserts, updates, deletes) from its replication log, as they happen, and delivering them to other systems: a search index, a cache invalidator (Chapter 21, Section 21.5), a data warehouse, a message broker, another service. It is logical replication (Section 22.2) whose consumer is not a database replica.

### Why it exists

Many systems need to follow the data in the system of record: search must index new products, caches must drop changed entries, analytics must see orders. The tempting solution, **dual writes** (the application writes to the database and then to the search index), is unreliable: one write can succeed and the other fail, and two concurrent requests can apply their second writes in the opposite order. CDC derives every downstream copy from the one log the database already keeps, in commit order, including changes made by scripts, migrations and other services.

### Mechanism: the flow

```text
 application ──► DATABASE (system of record) ── commits written to WAL / binlog
                                                     │ logical decoding (PostgreSQL output plugins
                                                     │ such as pgoutput) / binlog reading (MySQL)
                                                     ▼
                                              CDC connector (e.g. Debezium), position stored
                                              in a replication slot or as a binlog position/GTID
                                                     │ change events, in commit order per table/key:
                                                     │ {op: "u", table: "products", key: 17,
                                                     │  before: {price: 1999}, after: {price: 2499},
                                                     │  source: {lsn: "0/3A1F340", ts: …}}
                                                     ▼
                                              message broker (a log, Chapter 25)
                                        ┌────────────┼─────────────────┐
                                        ▼            ▼                 ▼
                                  search index   cache invalidator   warehouse
```

A new consumer usually starts with a **snapshot** of the current table contents, then continues with the change stream from the snapshot's log position, the same pattern as a new replica (base backup plus log).

### Guarantees and non-guarantees

CDC guarantees that downstream systems receive every committed change, in commit order (per table or key, depending on the pipeline), including changes made outside the application, and nothing that was rolled back. It does not guarantee timeliness (downstream copies lag like asynchronous replicas), exactly-once delivery (consumers must be idempotent, Chapter 25), or that downstream schemas keep up with upstream schema changes.

### Failure and diagnosis

- **WAL accumulating on the source.** A stopped connector's slot retains WAL (Chapter 15, Section 15.10). Evidence: retained WAL per slot. Fix: alert on slot lag; cap retention where acceptable.
- **Downstream drift after dual writes.** Evidence: search results missing items the database has. Fix: replace dual writes with CDC (or the outbox pattern, Chapter 25).

### Backend connection

When another system must follow the database, derive it from the database's log with CDC instead of writing to both from application code. Treat each consumer as an asynchronous replica: monitor its lag, make it idempotent, and plan its initial snapshot. Chapter 25 builds on this with brokers, the outbox pattern and event-driven designs.

---

## 22.11 Diagnosis Toolkit: Reading Replication

### Definition

The **replication diagnosis toolkit** is the set of views and status commands that show each replica's position relative to its leader, the lag at each step of the pipeline, the state of slots and connectors, and the history of failovers.

### Why it matters

Replication problems look like application bugs ("my data disappeared", "the page shows old values") or like disk problems ("the primary's disk is full"). The replication views turn them into positions and delays.

### Concrete: PostgreSQL

```sql
-- on the primary: one row per connected standby or logical consumer
SELECT application_name, state, sync_state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
--  application_name │   state   │ sync_state │ … │ replay_lag
--  replica_a        │ streaming │ sync       │ … │ 00:00:00.0012
--  replica_us       │ streaming │ async      │ … │ 00:00:04.83

-- on a standby
SELECT pg_is_in_recovery(), pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(),
       now() - pg_last_xact_replay_timestamp() AS approx_lag;  -- misleading when the primary is idle

-- slots that retain WAL (Chapter 15, Section 15.10)
SELECT slot_name, slot_type, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
```

`write_lag`, `flush_lag` and `replay_lag` separate where a delay happens: in the network and the standby's writes, in its disk flushes, or in applying the changes.

### Concrete: MySQL

```sql
SHOW REPLICA STATUS\G
--  Replica_IO_Running: Yes            ← receiving the binlog
--  Replica_SQL_Running: Yes           ← applying it
--  Seconds_Behind_Source: 4           ← estimated from event timestamps; can read 0 while the
--                                        receiving side is behind, and is NULL when a thread stopped
--  Retrieved_Gtid_Set / Executed_Gtid_Set   ← what has been received versus applied
--  Last_SQL_Error                      ← why applying stopped
```

### Concrete: reading the signals

| Symptom | Signal | Typical cause |
|---|---|---|
| users see old data | `replay_lag` / `Seconds_Behind_Source` high at that time | lag during batch writes or long replica queries |
| lag growing steadily | replay position falling behind receive position | the replica cannot apply as fast as the leader writes (single-threaded apply, I/O) |
| lag jumps then recovers | lag correlated with bulk jobs or recovery conflicts | large transactions, long queries on the replica |
| primary disk filling | inactive slot with large retained WAL | dead consumer |
| writes failing after failover | `pg_is_in_recovery()` true on the node clients use | stale routing to the old primary |

### Backend connection

Graph replay lag per replica and alert on it in seconds, not only on replication being "up". Measure lag with a heartbeat row written on the leader every second and read on replicas when the leader may be idle. Record every failover with its log position, so that lost transactions can be identified afterwards.

---

## 22.12 Placement-Round Topics: Distributed Databases in Textbooks

> **Placement-round focus:** This section is revision material for the distributed-database chapter of university DBMS courses and placement tests. It is not part of the main flow. What carries into backend work: fragmentation is partitioning (Chapter 23), and the transparency goals describe what a good distributed database hides from applications.

A **distributed database** is a single logical database whose data is stored at several sites connected by a network, managed by a **distributed DBMS** (DDBMS) that makes the distribution transparent to users. A **homogeneous** DDBMS runs the same software at every site; a **heterogeneous** one connects different DBMSs (often through gateways), and a **federated** one keeps each site autonomous.

**Fragmentation** divides a relation into pieces stored at different sites:

- **Horizontal fragmentation**: subsets of rows, defined by selection (σ<sub>region='EU'</sub>(customers) at the EU site). Reconstructed by union.
- **Vertical fragmentation**: subsets of columns, each including the key (π<sub>id, name, email</sub> at one site, π<sub>id, preferences</sub> at another). Reconstructed by join on the key.
- **Mixed (hybrid) fragmentation**: both.
- Correctness rules: **completeness** (every row or column appears in some fragment), **reconstruction** (the relation can be rebuilt), **disjointness** (no row in two horizontal fragments; vertical fragments share only the key).

**Replication** and **allocation**: each fragment may be stored at one site (no replication), several (partial replication) or all (full replication); allocation decides where, trading read locality against update cost.

**Transparency** levels: **location** transparency (users need not know where data is), **fragmentation** transparency (users query the relation, not its fragments), **replication** transparency (users do not see copies), and **failure** transparency (the system hides site failures). Textbooks also distinguish **primary-copy** replication (one site owns updates for each item, as in Section 22.2) from **update-anywhere** (as in Section 22.6). Distributed transactions and two-phase commit are Chapter 23's subject.

### Drill for this section

**C1. Explain horizontal, vertical and mixed fragmentation, and the correctness rules.**
A strong answer must mention:
- Horizontal: rows by a selection predicate, rebuilt by union; vertical: columns with the key, rebuilt by join; mixed: both.
- Completeness, reconstruction, disjointness.
- Allocation and replication of fragments trade read locality against update cost.
*Common weak answer:* forgetting that vertical fragments must each carry the key, without which reconstruction is impossible.

**C2. What kinds of transparency should a distributed DBMS provide?**
A strong answer must mention:
- Location, fragmentation, replication and failure transparency.
- Homogeneous versus heterogeneous (and federated) DDBMSs.
- Primary-copy versus update-anywhere replication.
*Common weak answer:* listing "transparency" without saying what each one hides.

---

## Chapter Summary

**Replication** keeps copies of data on several machines for **availability**, **read scaling**, **latency across regions** and **durability**, but every committed change, including a disastrous one, is copied in milliseconds, so replication is never a backup. In **single-leader** replication, one leader orders all writes in its log and followers apply it, either **physically** (WAL streaming: exact copies for failover and read replicas) or **logically** (row changes: upgrades, partial copies, other systems). The acknowledgement point decides durability: **asynchronous** replication acknowledges after the local flush and can lose recent commits on failover; **synchronous** (or lossless **semi-synchronous** and **quorum**) replication acknowledges after followers confirm, adding a round trip and risking write unavailability when synchronous followers are down.

Asynchronous followers lag, without bound, producing three anomalies with three fixes: **read-your-writes** (leader reads for a user's own data, waiting for a log position, `remote_apply`), **monotonic reads** (pinning a session to one replica or tracking positions) and **consistent prefix** (keeping causally related writes in one log, or tracking dependencies). **Failover** detects a dead leader by timeout, elects a successor by consensus, **fences** the old leader (leases, STONITH, epochs checked on every write) and redirects clients; without fencing it risks **split-brain**, two leaders with diverging histories, and with asynchronous replication it loses the last acknowledged writes.

**Multi-leader** replication accepts writes in several places for geography and offline work, and must resolve **conflicts**: avoid them with a home leader per record, or resolve them by last-write-wins (which loses data), merging, siblings, or **CRDTs** whose merges always converge. **Leaderless** replication writes to W and reads from R of N replicas, with **W + R > N** making read and write sets overlap, plus **read repair**, **hinted handoff** and **Merkle-tree anti-entropy**; quorums give tunable freshness and high availability, not linearizability.

The **consistency ladder** runs from **linearizable** (one copy, real time) through **sequential**, **causal** (cause before effect, achievable with availability), **session guarantees** and **eventual** consistency, each step weaker and cheaper. **CAP**, stated precisely, says that during a partition a system must give up either linearizability or availability for the requests it receives; **PACELC** adds that even without partitions, consistency costs latency. Both choices are best made per operation. **Change data capture** applies replication to other systems, deriving search indexes, caches and warehouses from the database's own log instead of unreliable dual writes, and every one of these copies is monitored, like a replica, by its position and its lag.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Replication | Copies on several machines; survival of node loss | Protection from replicated mistakes (not a backup); write scaling |
| Single-leader log | Followers apply writes in commit order (a prefix of the leader's history) | Bounded lag; current reads on followers |
| Asynchronous replication | Low commit latency | Acknowledged commits surviving failover |
| Synchronous / lossless semi-sync / quorum commit | Acknowledged commits exist on a confirmed follower | Write availability when confirmations cannot be obtained; current reads elsewhere |
| `remote_apply` | Reads on that standby see the commit | Reads on other standbys |
| Read-your-writes techniques | A user sees their own writes | Other users' views; cross-device without server-side tracking |
| Session pinning | Monotonic reads for that session | Freshness |
| Consensus-based failover with fencing | At most one leader accepting writes | Zero data loss with asynchronous followers; short outages |
| Multi-leader | Local writes; writes during partitions | Freedom from conflicts; cross-record invariants |
| CRDTs | Convergence of replicas that received the same updates | That the merged result is what each user intended for every data type |
| Quorum W + R > N | Read set overlaps the latest successful write set (strict quorums) | Linearizability; rollback of failed writes; overlap under sloppy quorums |
| Linearizability | One-copy, real-time behaviour per object | Availability during partitions; multi-object transactions |
| Causal consistency | Effects after causes for everyone | A single order of concurrent operations |
| Eventual consistency | Convergence once writes stop | Anything about reads in the meantime; a time bound |
| CDC | Every committed change delivered, in commit order | Timeliness; exactly-once delivery |

---

## Interview Drill

**Q1. Why replicate? Why is a replica not a backup?**
A strong answer must mention:
- Availability (failover), read scaling, latency across regions, durability on several machines.
- Replication copies every committed change, including mistakes and corruption, within milliseconds.
- Backups (with PITR) and delayed replicas preserve the past; both are needed.
- Replication does not scale writes.
*Common weak answer:* "Replicas are copies, so they are backups." A `DELETE` replicates too.

**Q2. How does database replication work? Physical versus logical?**
A strong answer must mention:
- Single leader orders writes in its log; followers apply the same log in the same order.
- Physical: WAL/page-level streaming; exact copies; same version; whole cluster.
- Logical: row-level changes; subsets, different versions and indexes; DDL often not replicated; also feeds CDC.
- Statement-based replication's non-determinism problem; new followers start from a base backup plus the log.
*Common weak answer:* "The leader sends SQL statements to the followers." Mostly not; non-deterministic statements would diverge.

**Q3. Synchronous versus asynchronous replication: what are the trade-offs?**
A strong answer must mention:
- Async: acknowledge after local flush; low latency; acknowledged commits can be lost on failover (the loss window).
- Sync: acknowledge after follower confirmation; no loss for confirmed followers; adds a round trip; writes stall if synchronous followers are down.
- Semi-sync and quorum commit as middle grounds; MySQL semi-sync can silently fall back to async.
- In-zone synchronous plus cross-region asynchronous as a common layout.
*Common weak answer:* "Synchronous is always better because it is safer." It costs latency on every write and can make the database unavailable.

**Q4. What is replication lag, and which anomalies does it cause?**
A strong answer must mention:
- The delay between commit on the leader and application on a follower; unbounded with async replication.
- Read-your-writes violations: fixed by leader reads, waiting for a log position, `remote_apply`.
- Monotonic reads violations: fixed by session pinning or position tracking.
- Consistent prefix violations across partitions: fixed by co-locating related writes or tracking causal dependencies.
*Common weak answer:* "Lag means data is a bit old." It misses the specific user-visible anomalies and their fixes.

**Q5. How does failover work, and what can go wrong?**
A strong answer must mention:
- Detect by timeout, decide by consensus, fence the old leader, promote the most up-to-date follower, redirect clients and followers, rejoin the old leader as a follower.
- Timeouts cannot distinguish slow from dead: false failovers versus long outages.
- Async failover loses acknowledged writes; reused IDs and external systems that saw lost writes.
- Clients still connected to the old leader.
*Common weak answer:* "The replica becomes the new primary automatically." It skips fencing, data loss and client redirection.

**Q6. What is split-brain, and how is it prevented?**
A strong answer must mention:
- Two nodes both acting as leader and accepting writes, so histories diverge.
- Caused by failover during a partition without fencing.
- Prevention: consensus-based election (majority), leader leases that expire before a new leader is chosen, STONITH, epochs or fencing tokens checked by storage and followers.
- Three failure domains so a majority exists.
*Common weak answer:* "Use a heartbeat." Heartbeats detect absence; they do not stop the old leader from writing.

**Q7. Leader–follower versus multi-leader versus leaderless?**
A strong answer must mention:
- Single leader: one write order, no conflicts, writes limited to and routed through one node; the default for relational databases.
- Multi-leader: local writes per region or device and writes during partitions, at the cost of conflict resolution.
- Leaderless: quorums of N replicas, high availability, tunable consistency, read repair and anti-entropy; membership and failure suspicion shared by gossip.
- Choice by write locality, availability needs and tolerance for conflicts.
*Common weak answer:* "Leaderless is the most scalable, so it is best." It gives up a single order and makes invariants hard.

**Q8. How are write conflicts resolved? What is a CRDT?**
A strong answer must mention:
- Avoidance by home leader; last-write-wins (loses data, depends on clocks); merging; siblings resolved by the application.
- CRDTs: types with commutative, associative, idempotent merges that always converge (G-counter with element-wise max, OR-set with unique tags).
- Invariants needing coordination cannot be enforced by independent leaders.
*Common weak answer:* "The latest timestamp wins." Clock skew decides "latest", and the other write is silently lost.

**Q9. How do quorums work? Does W + R > N guarantee strong consistency?**
A strong answer must mention:
- N replicas per key; writes need W acknowledgements, reads R responses; W + R > N makes the sets overlap.
- Example N = 3, W = 2, R = 2 tolerates one failure for reads and writes.
- Not linearizable: concurrent writes, partially failed writes not rolled back, reads concurrent with writes, sloppy quorums.
- Read repair, hinted handoff, Merkle-tree anti-entropy.
*Common weak answer:* "Yes, quorums are strongly consistent." Overlap is necessary, not sufficient.

**Q10. What is linearizability? How does it differ from eventual consistency?**
A strong answer must mention:
- Linearizable: behaves as one copy; every operation takes effect at an instant between start and end; later reads see completed writes.
- Eventual: replicas converge when writes stop; reads may return anything meanwhile.
- Costs: coordination, latency, unavailability on a partition's minority side, versus none.
- Uses: locks, leader election, uniqueness, money for linearizable; caches and aggregates for eventual.
- Linearizability is not serialisability.
*Common weak answer:* "Linearizable means all replicas are updated at the same time." It is about observable order of operations, not simultaneous updates.

**Q11. What is causal consistency?**
A strong answer must mention:
- Causally related operations are seen in that order by everyone; concurrent ones in any order.
- Example: a reply never visible before the message it answers.
- Implemented with dependency tracking (version vectors, causal metadata) or co-location.
- Achievable while remaining available during partitions; stronger than eventual, weaker than linearizable.
*Common weak answer:* "Everything is seen in the same order." That is sequential consistency.

**Q12. Explain the CAP theorem properly. What does it not say?**
A strong answer must mention:
- C is linearizability, A is a non-error response from every non-failed node, P is operating despite lost messages.
- During a partition, a system must give up C or A for the requests it receives.
- Not "pick two": partitions are not optional; without one, both C and A are possible; choices are per operation; CP systems serve the majority side.
- CAP's C is not ACID's C; CAP says nothing about latency.
*Common weak answer:* "You can only have two of consistency, availability and partition tolerance." It misses that the choice exists only during partitions.

**Q13. What is PACELC?**
A strong answer must mention:
- If partitioned, choose availability or consistency; else, choose latency or consistency.
- Consistency in normal operation needs coordination (waiting for remote replicas or a majority), which costs latency.
- Systems described as PA/EL, PC/EC and mixes, per operation and configuration.
*Common weak answer:* treating it as a renamed CAP; its contribution is the latency trade-off without partitions.

**Q14. What is change data capture?**
A strong answer must mention:
- Reading committed changes from the database's replication log and delivering them to other systems.
- Replaces unreliable dual writes; captures changes from every writer, in commit order.
- Snapshot plus stream for new consumers; slots retain WAL until consumed.
- Consumers lag and must be idempotent (Chapter 25).
*Common weak answer:* "Triggers that write to another table." That is a different, in-database mechanism; CDC reads the log.

---

## Scenario Exercises

1. A team runs PostgreSQL with one asynchronous replica in another region and proposes making it synchronous "for safety". Write latency today is 4 ms; the inter-region round trip is 85 ms. Estimate the effect, explain what happens when the replica is down, and propose a layout that gives zero data loss for zone failures without an 85 ms penalty.
2. Users of a social app report that their own comments sometimes disappear after a refresh and reappear a few seconds later. Reads go to a pool of three replicas behind a load balancer. Name the anomalies, show the timelines, and fix both.
3. During a network incident, both database nodes of a two-node cluster accepted writes for 4 minutes. Explain how this happened, what data is now at risk, how to reconcile it, and how to prevent it.
4. A global note-taking app must let users edit offline on two devices and merge later. Choose a replication model and a conflict-resolution approach for (a) the note's title, (b) its tag set, (c) its body text, and (d) a per-user storage quota.
5. A Cassandra-like store uses N = 3, with writes at `ONE` and reads at `ONE` for a user-settings table. Users occasionally see old settings after changing them. Explain, and give two configurations that fix it with different latency and availability trade-offs.
6. An architect says: "Our database is CP, so we cannot be available during a zone failure." The cluster has five nodes across three zones. Correct the statement and explain what happens to requests in each zone when one zone is cut off.
7. A service keeps an Elasticsearch index in sync by writing to PostgreSQL and then to the index in the same request handler. The index occasionally misses updates or contains changes that were rolled back. Explain both, and redesign.

---

## Diagnosis Lab

**Lab 1: "I saved but don't see it"**
- *Symptom:* After editing their shipping address, about 3% of users see the old address on the confirmation page; reloading a few seconds later shows the new one.
- *Hypotheses:* (a) a cache serving stale data; (b) replica lag; (c) the write failing silently.
- *Evidence:* The write committed (the leader's row has the new address; the audit log shows it). The confirmation page's query runs on a read replica; request logs show the read 150–400 ms after the write. `pg_stat_replication.replay_lag` for the replicas fluctuates between 50 ms and 900 ms, spiking during a nightly import that is still running at peak times. No cache is involved on this path.
- *Root cause:* (b): read-your-writes violation on a lagging replica.
- *Fix:* After a write, store the leader's LSN (`pg_current_wal_lsn()`) in the user's session; for that user's reads in the next 30 seconds, use a replica only if its `pg_last_wal_replay_lsn()` is at or beyond it, otherwise read from the leader. Move the import out of peak hours and chunk it (Chapter 18, Section 18.14).
- *Prevention:* An alert on replay lag above one second; a test that reads immediately after writes against a replica with injected lag.

**Lab 2: A replica lag spike**
- *Symptom:* The analytics replica's lag grows from seconds to 40 minutes every afternoon, then catches up overnight; dashboards show stale data.
- *Hypotheses:* (a) network saturation; (b) replay blocked by queries on the replica; (c) the replica's disk too slow.
- *Evidence:* `pg_stat_replication` shows `write_lag` and `flush_lag` small but `replay_lag` large: WAL arrives and is stored, but is not applied. On the replica, `pg_stat_activity` shows a 45-minute analytics query; the replica's settings delay replay (rather than cancelling queries) for conflicts with running queries (`max_standby_streaming_delay = -1`). The log shows replay waiting for that query.
- *Root cause:* (b): recovery conflicts (Chapter 17, Section 17.12): replaying cleanup records that remove rows the query may need is postponed until the query ends, so lag grows for as long as such queries run.
- *Fix:* Bound the delay (a finite `max_standby_streaming_delay`, accepting that very long queries will be cancelled), shorten analytics queries or move them to a separate replica or warehouse (via CDC, Section 22.10), and keep the replica that serves user-facing reads free of long queries.
- *Prevention:* Separate replicas by purpose; alerts on replay lag and on long queries on replicas.

**Lab 3: Data lost in a failover**
- *Symptom:* After an automatic failover at 14:02, customers complain that orders confirmed by email between 14:01:50 and 14:02:00 do not exist; new orders since then reuse some of the same order numbers.
- *Hypotheses:* (a) the application failed to commit those orders; (b) the failover lost acknowledged commits; (c) the new primary is missing data for another reason.
- *Evidence:* The old primary's WAL (recovered from its disk after the incident) contains 214 committed orders after the last LSN the promoted replica had received. Replication was asynchronous. Order numbers come from a database sequence, which on the new primary continued from its own, older value, so it issued numbers already used by the lost orders and already emailed to customers.
- *Root cause:* (b): an asynchronous failover's loss window (Section 22.3), amplified by identifiers allocated by the database and side effects (emails) already sent.
- *Fix (now):* Extract the lost transactions from the old primary's WAL (or a logical decoding of it), re-insert them under new order numbers after checking for conflicts, and contact affected customers; move the sequences past the highest number issued. *Fix (lasting):* a synchronous or quorum standby in another zone for the order database (Section 22.3); order numbers not tied to the sequence of one node (UUIDs or a dedicated allocator); confirmation emails sent from an outbox after the commit is replicated (Chapter 25).
- *Prevention:* Know and accept, in writing, the loss window per database; failover drills that measure it; post-failover reconciliation procedures.

---

## Solution Sketches

1. Synchronous commit to the remote replica would add at least the 85 ms round trip (plus the remote flush) to every commit, so writes go from about 4 ms to about 90 ms, and with a single synchronous replica, an outage of that replica (or of the link) blocks all writes. Layout: a synchronous (or quorum `ANY 1 (a, b)`) standby in another zone of the same region (round trip around 1 ms), giving zero loss for zone failures at small cost, plus the cross-region replica kept asynchronous for disaster recovery, with its loss window accepted and monitored.
2. Read-your-writes (the user's write is not yet on the replica that serves the read) and monotonic reads (consecutive refreshes hit replicas with different lag, so the comment appears and disappears). Fixes: route the user's own reads to the leader or to replicas that have reached the position of their last write (track it in the session); pin each user's reads to one replica (hash of user ID) for monotonicity, or track the highest position seen and never read from a replica behind it.
3. Split-brain: with two nodes and no external arbiter, each side of the partition concluded the other was dead and promoted itself (or the old primary kept writing while the standby was promoted); there was no majority to decide and no fencing. At risk: rows written on both sides with conflicting keys, and side effects triggered from both. Reconcile: freeze writes, compare both histories from their logs (logical decoding), choose a base, replay the other side's non-conflicting changes, resolve conflicts by business rules, re-number duplicates, notify affected customers. Prevent: three voting members (or a witness) in three failure domains, consensus-based failover with leases and fencing (STONITH or storage-level epochs).
4. Multi-leader or offline-first replication with per-field rules: (a) title: last-write-wins is acceptable for a small field, or keep both and ask the user if they differ; (b) tags: an OR-set CRDT, so concurrent additions and removals merge; (c) body text: a sequence CRDT or operational transformation for character-level merging; (d) quota: an invariant across all notes; enforce it on the server with a single authority (reject or flag when syncing), not by independent devices.
5. W = 1 and R = 1 give W + R = 2 ≤ N = 3, so a read can go to a replica that has not received the write. Fix A: `QUORUM` writes and reads (2 + 2 > 3): fresher, tolerates one replica failure, higher latency. Fix B: `ALL` writes and `ONE` reads (3 + 1 > 3): fast reads, but writes fail when any replica is down; or `LOCAL_QUORUM` in multi-region setups to keep latency local. (Neither makes reads linearizable, Section 22.7.)
6. CAP's trade-off applies only during a partition, and a CP system remains available on the majority side. With five nodes across three zones (for example 2, 2, 1), cutting off one zone leaves a majority (at least three nodes) in the other two: requests there succeed. Requests reaching nodes in the isolated zone cannot reach a majority and are refused (or redirected by clients to the healthy zones). "CP" describes the minority side's behaviour, not the whole system's availability.
7. Dual writes: if the index write fails or the process crashes after the database commit, the index misses the update; if the index is written before (or regardless of) a transaction that later rolls back, it holds a change that never committed; and concurrent requests can apply their index writes in a different order from their commits. Redesign: write only to PostgreSQL, and feed the index from CDC (logical decoding through a connector to a broker, consumed by an idempotent indexer that applies changes in commit order per document), or use the outbox pattern (Chapter 25); add a periodic reconciliation job.

---

**END OF CHAPTER 22**
