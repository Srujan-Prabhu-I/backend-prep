# Chapter 23: Partitioning, Coordination, and Choosing Data Stores

## Where We Are

Chapter 22 copied the same data onto several machines. Copies help availability, read throughput and geography, but every replica still stores **all** the data and applies **every** write, so replication alone stops working when the data no longer fits on one machine or the writes exceed what one leader can absorb. Chapter 16 split a large table into partitions **inside one server** (Section 16.14); Chapter 10 met consistent hashing for spreading requests over backends (Section 10.3); Chapter 21 met hot keys in a cache (Section 21.6) and warned that a lock in Redis is subtler than it looks (Section 21.8); Chapter 22 used leases and epochs to fence an old leader (Section 22.5) and saw last-write-wins decided by clocks (Section 22.6).

This chapter takes the next step: splitting data **across** machines, and the coordination problems that follow. It covers partitioning (sharding) strategies, consistent hashing for data, hot spots, rebalancing and routing, secondary indexes and queries that span shards. Then it turns to the hard parts of distributed coordination: why clocks cannot order events and what can, consensus and the Raft algorithm, distributed locks and fencing tokens, and two-phase commit with its blocking problem. It ends with the non-relational data models and a method for choosing a data store from access patterns and guarantees, without slogans.

**After this chapter you can:**
- explain why and when to partition, and distinguish sharding from replication and from table partitioning;
- compare key-range, hash and directory partitioning, and predict hot spots for a given key;
- explain consistent hashing with virtual nodes, and compute how many keys move when a node joins, compared with mod-N hashing;
- mitigate hot keys and skew, choose a rebalancing strategy, and route requests to the right shard;
- design secondary indexes across shards (local versus global) and reason about cross-shard queries, joins and their tail latency;
- explain why wall clocks cannot order events, and use Lamport and vector clocks and monotonic clocks correctly;
- explain the consensus problem, why it needs majorities, and how Raft elects leaders and commits log entries;
- explain why distributed locks need leases and fencing tokens, and why a lock in Redis is subtle;
- trace two-phase commit, explain why it blocks when the coordinator fails, and why microservices avoid it;
- compare key–value, document, wide-column, graph, search and time-series stores, and BASE versus ACID;
- choose a data store from access patterns, consistency, integrity, scale and operations, and design polyglot persistence with a clear source of truth;
- diagnose a hot shard, a timing-out cross-shard query and a lock holder paused by garbage collection.

**Interview questions this chapter prepares you for:**
1. Why and how do you shard a database? How do you choose a shard key?
2. Range versus hash partitioning: what are the trade-offs?
3. What is consistent hashing, and why use virtual nodes?
4. How do you handle hot keys, skew and rebalancing?
5. How do you query across shards? Local versus global secondary indexes?
6. Why are clocks unreliable in distributed systems? What are Lamport and vector clocks?
7. What is consensus, and why is an algorithm such as Raft needed? Why majorities?
8. How does Raft elect a leader and replicate its log?
9. How do you build a correct distributed lock? What is a fencing token?
10. How does two-phase commit work, and why does it block? Why do microservices avoid it?
11. What are the main NoSQL data models, and what is each good for? What is BASE?
12. How do you choose between SQL and NoSQL? What is polyglot persistence?
13. What is scalability? Vertical versus horizontal scaling, and where does sharding fit?

**Scope.** Here: partitioning strategies, consistent hashing for data, hot spots, rebalancing and routing, cross-shard indexes and queries, clocks and logical time, consensus and Raft, distributed locks and fencing, two-phase commit, non-relational data models, choosing data stores, diagnosis, and a placement-round section on data-warehouse modelling and Byzantine faults (Section 23.13). Recalled, not re-taught: consistent hashing for load balancing (Chapter 10, Section 10.3), table partitioning inside one server (Chapter 16, Section 16.14), replication, failover, quorums and consistency models (Chapter 22), LSM-trees and column stores (Chapter 15, Sections 15.6–15.7), inverted indexes (Chapter 16, Section 16.6), transactions and isolation (Chapter 17). Elsewhere: sagas, the outbox and event-driven designs are Chapter 25; deploying and operating clusters is Chapter 26.

---

## 23.1 What Partitioning Is, and Why Systems Partition

### Definitions

**Partitioning** (in distributed databases usually called **sharding**) is splitting a dataset into disjoint pieces, called **partitions** or **shards**, stored on different nodes, so that each node holds and serves only part of the data. Every record belongs to exactly one partition, chosen by its **partition key** (shard key).

- **Sharding versus table partitioning**: table partitioning (Chapter 16, Section 16.14) splits a table into pieces **inside one database server**, for pruning and retention; sharding splits data **across servers**, for capacity.
- **Sharding versus replication**: replication copies **all** data to each replica (Chapter 22); sharding gives each node **part** of it. Real systems combine them: each shard is itself replicated, so each node holds replicas of several shards.
- **Scalability** is a system's ability to handle more load (more data, requests or writes) by adding resources, ideally with cost growing roughly in proportion to the load. **Vertical scaling** (scaling up) adds resources to one machine: more CPU cores, more memory, faster disks. **Horizontal scaling** (scaling out) adds machines and divides the work among them. Vertical scaling needs no change to the software, but it stops at the largest machine available and leaves one machine as a single point of failure. Horizontal scaling has no such ceiling, but only works if the work can be divided. Stateless application instances divide easily (Chapter 11, Section 11.6); data does not, and sharding is how it is divided. Scaling the application tier, autoscaling and capacity planning are Chapter 26's subject.

```text
            shard 1 (keys A–H)       shard 2 (keys I–Q)       shard 3 (keys R–Z)
 node 1     leader                   follower                 follower
 node 2     follower                 leader                   follower
 node 3     follower                 follower                 leader
 each node: one third of the leadership work, all three shards' data replicated for availability
```

### Why it exists

Three limits push a system past one machine:

1. **Data size**: the dataset (with indexes) exceeds what one machine's storage or memory can hold or back up and restore in an acceptable time (Chapter 15, Section 15.9).
2. **Write throughput**: one leader must apply every write (Chapter 22, Section 22.1); its CPU, WAL bandwidth or I/O saturates. Replicas do not help, because they apply the same writes.
3. **Hot working set**: the frequently read data no longer fits in one machine's memory, so reads fall to disk.

Partitioning divides all three by the number of shards, if the load spreads evenly. The cost is everything this chapter discusses: choosing keys, moving data, routing, queries and transactions that span shards.

### Mechanism: when not to shard yet

Sharding is a large, mostly irreversible complexity. Before it, the usual sequence is:

```text
 1. tune queries and indexes (Chapter 16)              often gives 10–100× on the worst paths
 2. a larger machine (vertical scaling)                simple; modern servers are very large
 3. read replicas and caching (Chapters 18, 21, 22)    removes read load from the leader
 4. table partitioning and archiving (Chapter 16)      keeps the hot set small
 5. split by function: move a whole domain             orders in one database, catalogue in another
    (a bounded context, Chapter 12, Section 12.11)     (each still unsharded)
 6. shard the data that still exceeds one node         the step this chapter is about
```

### Guarantees and non-guarantees

Partitioning guarantees that each node stores and processes only its share of data and writes, so capacity grows with the number of shards. It does not guarantee an even spread (skew, Section 23.4), cheap queries that need many shards (Section 23.5), transactions across shards (Section 23.9), or global uniqueness and ordering, which a single database gave for free.

### Failure and diagnosis

- **Sharding too early.** A team spends months on sharding infrastructure for data that would fit on one large server. Evidence: data size and write rates far below a single node's capacity. Fix: steps 1–5 above.
- **Sharding too late.** A single leader at the limit, with no time to redesign. Evidence: write latency and WAL rate rising steadily with no remaining headroom. Fix: capacity planning that projects growth, and choosing future shard keys early (put a tenant or customer ID on every table, Section 23.2).

### Backend connection

Delay sharding as long as simpler steps work, but design for it early: give every table that will grow a natural partition key (tenant, customer, account), keep transactions within one such key where possible, and avoid queries that need a global view of hot data. Those habits make a later split mechanical rather than a rewrite.

---

## 23.2 Partitioning Strategies

### Definitions

- **Key-range partitioning** assigns each shard a contiguous range of keys (`A–H`, `I–Q`, `R–Z`; or dates), in key order.
- **Hash partitioning** assigns each key to a shard by a hash of the key, so that keys spread uniformly regardless of their order.
- **Directory-based partitioning** (lookup-based) keeps an explicit mapping from keys (or groups of keys, such as tenants) to shards in a separate table or service.
- A **compound key** combines a partition key, which chooses the shard, with a **sort key** (clustering key), which orders records within it: Cassandra's partition key plus clustering columns, DynamoDB's partition key plus sort key.

### Why it matters

The partitioning strategy decides two things at once: how evenly load spreads, and which queries can be answered by one shard. These pull in opposite directions. Keeping related keys together makes range queries cheap and can concentrate load; scattering keys spreads load and makes range queries touch every shard.

### Concrete: range versus hash, with a hot spot

An events table, 3 shards, partitioned by `created_at`:

```text
 RANGE PARTITIONING by created_at
 shard 1: Jan–Apr        shard 2: May–Aug        shard 3: Sep–Dec
 query "events from 1–7 October"  → shard 3 only                ✔ efficient range scan
 writes today (all of October)    → shard 3 only                ✘ HOT SPOT: one shard takes 100% of writes,
                                                                    the others are idle

 HASH PARTITIONING by hash(event_id) mod 3
 writes today → spread evenly over shards 1, 2, 3               ✔ balanced
 query "events from 1–7 October"  → must ask ALL shards         ✘ scatter-gather (Section 23.5)
```

The same tension appears with any monotonically increasing key (timestamps, auto-increment IDs) and range partitioning. A common middle ground is a **compound key** that hashes a high-cardinality component and sorts by time inside it:

```text
 partition key = device_id (hashed: devices spread over shards)
 sort key      = created_at (within one device, events ordered by time)
 "events of device 81 in October" → one shard, one range scan    ✔
 "all events in October"          → all shards                   (still a scatter, but rare)
```

### Mechanism: the three strategies compared

| Strategy | How a key finds its shard | Range queries on the key | Load balance | Rebalancing | Typical systems |
|---|---|---|---|---|---|
| Key-range | binary search over range boundaries | efficient: one or a few shards | at risk of hot spots for sequential or skewed keys | split hot or large ranges, move ranges | HBase, Bigtable, CockroachDB, TiKV, MongoDB ranged shard keys |
| Hash | hash(key) → partition | scattered over all shards | even for well-distributed keys | needs a scheme that moves few keys (Section 23.3) | Cassandra, DynamoDB, Redis Cluster, MongoDB hashed shard keys |
| Directory | look the key (or its group) up in a mapping | depends on the grouping | can be tuned per group (move a heavy tenant to its own shard) | edit the mapping and move the group's data | multi-tenant SaaS (tenant → shard), Vitess-style lookup tables |

### Mechanism: choosing a shard key

Good shard keys share three properties:

1. **High cardinality and even distribution**: many distinct values with similar load. A boolean or a country code is a bad shard key (a few huge, uneven shards).
2. **Aligned with the dominant access pattern**: most requests include the key, so they go to one shard. For a SaaS product that is usually the **tenant ID**; for a consumer app, the **user ID**.
3. **Aligned with transactions**: data that must change together atomically shares a key (an order and its lines both keyed by customer), so transactions stay within one shard (Section 23.9).

A key chosen for writes but not reads (or the reverse) makes the other side expensive forever, which is why the choice deserves more design time than almost anything else in a sharded system.

### Guarantees and non-guarantees

Range partitioning guarantees that a key range lives on few shards; hash partitioning guarantees an even spread of distinct keys; directory partitioning guarantees placement by explicit choice. None guarantees even **load**, because load depends on how often each key is accessed, not on how many keys there are (Section 23.4).

### Failure and diagnosis

- **All writes on one shard.** Evidence: per-shard write rates, one shard far above the others; keys increasing with time under range partitioning. Fix: a compound key with a hashed prefix, or hash partitioning.
- **Every query hits every shard.** Evidence: query fan-out metrics (shards contacted per request) near the shard count. Cause: a shard key that the main queries do not include. Fix: rethink the key, or add a global index (Section 23.5).

### Backend connection

Pick the shard key from the requests that matter most, and include it in every one of them. Prefer tenant or user IDs for multi-tenant and consumer data, hashed when even spread matters, with a sort key for time-ordered access inside each partition. Write down which queries the choice makes expensive, and make sure they are rare.

---

## 23.3 Consistent Hashing for Data

### Definitions

Chapter 10 (Section 10.3) defined **consistent hashing** for load balancers: nodes and keys are hashed onto the same circular space (the **ring**), each key belongs to the first node clockwise from it, and **virtual nodes** (many positions per physical node) even out the spread. This section applies it to data, where the cost of moving a key is not a cache miss but **copying data** between machines.

A **fixed-partition** scheme is the main alternative: the key space is divided into many more partitions than nodes (thousands), keys hash to partitions, and whole partitions are assigned to nodes.

### Why it exists

The obvious mapping, `shard = hash(key) mod N`, moves almost every key when N changes. For a cache that means a cold cache; for a database it means copying almost the whole dataset across the network to add one machine. Consistent hashing and fixed partitions move only the fraction of data that the new node should take.

### Concrete: mod-N versus the ring

Adding a fifth node to four:

```text
 mod-N:   key k lives on node (hash(k) mod 4); after adding a node, on (hash(k) mod 5)
          a key stays only if hash(k) mod 4 == hash(k) mod 5: for uniformly distributed
          hashes that is 1 key in 5
          → 80% of keys MOVE  (in general N/(N+1) when going from N to N+1 nodes)

 consistent hashing, 4 nodes × 100 virtual nodes each on a ring of 2^32 positions:
      ring (clockwise)  … v(B) ── k1 ── v(A) ── k2 ── k3 ── v(C) ── k4 ── v(D) …
                        a key belongs to the next virtual node clockwise
      add node E with 100 virtual nodes, scattered around the ring:
      each new v(E) takes over the keys between its predecessor and itself, from ONE neighbour
      → about 1/5 = 20% of keys move, taken roughly evenly from A, B, C and D
      → removing a node moves only its own keys, spread over the remaining nodes
```

Virtual nodes matter twice for data. Without them, a node's single arc of the ring can be much larger than the average, so it stores more data; and when a node leaves, all its data would go to one neighbour, which then becomes overloaded. With many virtual nodes, both the stored share and the redistribution even out (and nodes of different capacity can be given different numbers of virtual nodes).

### Mechanism: fixed partitions, the practical variant

Many systems avoid moving individual keys altogether:

```text
 key → partition = hash(key) mod P, with P fixed and large (e.g. P = 1,024 or 16,384)
 partition → node: an assignment table, stored in a coordination service (Section 23.7)

 4 nodes:  each owns 256 partitions
 add node: move 1,024 / 5 ≈ 205 whole partitions to the new node (about 51 from each old node);
           no key changes partition, so nothing is rehashed; data moves partition by partition
```

Redis Cluster's 16,384 hash slots (Chapter 21, Section 21.7) and the fixed shard counts of systems such as Elasticsearch work this way. The constraint is that P must be chosen up front, larger than the largest number of nodes you expect, because changing it means rehashing everything.

### Guarantees and non-guarantees

Consistent hashing and fixed partitions guarantee that changing the number of nodes moves about 1/(N+1) of the data rather than nearly all of it, and spread keys evenly with enough virtual nodes or partitions. They do not guarantee even load for skewed access, do not remove the need to copy the moved data (which takes time and bandwidth), and do not preserve key order, so range queries still scatter.

### Failure and diagnosis

- **Uneven data per node.** Evidence: disk use differing widely across nodes. Cause: too few virtual nodes or partitions, or a few very large partitions (a huge tenant). Fix: more virtual nodes, splitting large partitions, directory placement for large tenants.
- **Cluster-wide slowdown while adding a node.** Evidence: network and disk saturated by data transfer; latency up on the old nodes. Fix: throttle rebalancing; add capacity before it is urgent.

### Backend connection

Never shard data with `mod N` over the current node count. Use consistent hashing or, more simply, a fixed large number of partitions mapped to nodes through a table, so that adding capacity is a matter of moving some partitions. Choose the partition count for the cluster you may have in five years.

---

## 23.4 Hot Spots, Rebalancing and Request Routing

### Definitions

- **Skew** is uneven load across partitions; a **hot spot** (hot shard) is a partition receiving a disproportionate share. A **hot key** is a single key responsible for much of that load (Chapter 21, Section 21.6, for caches).
- **Rebalancing** is moving partitions (or splitting and merging them) between nodes so that data and load are spread evenly, for example after adding nodes or when a partition grows.
- **Request routing** (service discovery for data) is how a request finds the node that holds its partition.

### Why it matters

Hashing spreads **keys** evenly, but load follows **activity**: one celebrity account, one viral product or one enormous tenant can concentrate thousands of requests per second on a single partition, which then runs at 100% while the rest idle. And clusters change: nodes join, fail and are replaced, and partitions grow, so data must move while the system keeps serving, without every client needing to know.

### Mechanism: mitigating hot keys and skew

```text
 HOT KEY SALTING (split one logical key across partitions)
   writes to "likes:post:777" → "likes:post:777#" + random(0..15)   → 16 keys on up to 16 shards
   reads sum the 16 counters (or a periodically aggregated total)
   cost: reads fan out; only works for data that can be split and recombined (counters, append-only)

 HEAVY TENANTS (directory placement)
   tenant 42 (40% of all traffic) → its own dedicated shard
   remaining tenants → hashed across the shared shards

 CACHING AND REPLICATION FOR HOT READS
   hot read-only keys → an in-process cache (Chapter 21, Section 21.9) or extra replicas of that shard
```

Load-aware systems can also **split** a hot range automatically (range partitioning makes this natural: split the range at the median key and move one half), but a single hot **key** cannot be split by range; it needs salting or caching.

### Mechanism: rebalancing strategies

| Strategy | How | Pros | Cons |
|---|---|---|---|
| Fixed number of partitions | many partitions from the start; move whole partitions | simple; no rehashing | partition count fixed; partitions grow with data |
| Dynamic splitting | split a partition when it exceeds a size (or load) threshold, merge small ones | adapts to data; works with range partitioning | starts with one partition unless pre-split; split bursts |
| Proportional to nodes | a fixed number of partitions per node; a new node splits some existing ones | partition size stays proportional | more complex bookkeeping |

**Automatic versus manual.** Fully automatic rebalancing can make a bad situation worse: a node that is slow (not dead) looks overloaded, the system moves data away from it, the transfer adds load, and the cluster spirals. Many operators therefore let the system **propose** moves and a human or a rate-limited controller **approve** them. Either way, moves must be throttled so that the copy does not starve normal traffic.

### Mechanism: request routing

```text
 (a) ANY NODE, FORWARDED          (b) ROUTING TIER                 (c) PARTITION-AWARE CLIENT
 client → any node                client → router/proxy            client keeps the partition map,
          node forwards to the            router looks up the map,   sends directly to the owner;
          owner (gossip-shared map)       forwards to the owner      "moved" replies refresh it
 e.g. Cassandra coordinators      e.g. Vitess vtgate, MongoDB       e.g. Redis Cluster clients
                                  mongos, Citus coordinator          (MOVED / ASK redirections)

 in all three, the authoritative map (partition → node) lives in a coordination store
 (ZooKeeper, etcd, or the cluster's own consensus group, Section 23.7), and changes are
 versioned so that a request routed with an old map is rejected or redirected
```

### Guarantees and non-guarantees

Rebalancing guarantees that data can move between nodes while the system serves, eventually balancing stored data. Routing guarantees that requests find the current owner, or are redirected. Neither guarantees balanced load under skewed access, and rebalancing does not happen instantly: during a move, a partition's data is being copied and both source and destination are busier.

### Failure and diagnosis

- **One shard at 100% CPU.** Evidence: per-shard CPU, request rate and the busiest keys on that shard; a single key or tenant dominating. Diagnosis Lab 1.
- **Rebalancing storms.** Evidence: continuous partition moves, high network use, latency up everywhere; a slow node triggering moves. Fix: rate-limit or pause automatic rebalancing; fix the slow node.
- **Requests to the wrong node.** Evidence: many redirect responses (`MOVED`), errors right after a topology change. Fix: clients that refresh the map on redirects; versioned maps.

### Backend connection

Assume skew: measure load per partition and per key, cache or salt hot keys, give the heaviest tenants their own placement, and rebalance in throttled, observable steps. Keep the partition map in a consensus-backed store and make every routing decision carry its version, so that stale routing is detected rather than silently wrong.

---

## 23.5 Secondary Indexes and Queries Across Shards

### Definitions

- A **local secondary index** (document-partitioned) is stored with each shard and indexes only that shard's records. A query by the indexed column must ask **every** shard: **scatter–gather**.
- A **global secondary index** (term-partitioned) is itself partitioned by the **indexed value**, so all entries for one value live on one index partition. A query by that value asks one index partition, but a write may need to update index partitions on other nodes, usually **asynchronously**.
- A **cross-shard query** needs data from several shards; a **cross-shard join** combines rows from different shards.
- **Co-location** stores related data on the same shard by giving it the same shard key; a **reference table** (broadcast table) is a small table copied to every shard.

### Why it matters

A sharded system answers queries by the shard key quickly and everything else expensively. The product will always need "everything else": find an order by its order number when sharded by customer, search users by email, list the top products across all tenants. How those queries are answered decides tail latency and whether results are consistent with recent writes.

### Concrete: local versus global index

Orders sharded by `customer_id`; we need lookups by `status`:

```text
 LOCAL INDEX on status (each shard indexes its own orders)
   write order 9001 (customer 81, status PAID) → shard 2: row + its local index, one shard, atomic
   query "orders WHERE status = 'REVIEW'" → send to ALL shards, merge the results

 GLOBAL INDEX on status (index partitioned by the status value)
   index partition for "REVIEW" lives on node 3, whatever shards the orders are on
   write order 9001 → shard 2 (row) + node 3 (index entry): two nodes; usually async → the index
                      can lag the data briefly (a just-written order may not appear yet)
   query "orders WHERE status = 'REVIEW'" → node 3 only, then fetch the rows from their shards
```

| | Local index | Global index |
|---|---|---|
| Writes | one shard, consistent with the row | two or more nodes; often asynchronous (eventually consistent) |
| Reads by the indexed value | scatter–gather to every shard | one index partition (plus fetching rows) |
| Fits | queries that also include the shard key, or rare scatter queries | frequent lookups by a non-shard-key value (email, order number) |
| Examples | Cassandra secondary indexes, Elasticsearch shards | DynamoDB global secondary indexes, explicit lookup tables |

A **lookup table** maintained by the application ("order number → customer ID", itself sharded by order number) is a hand-made global index: find the shard key first, then go to one shard.

### Mechanism: the tail-latency cost of scatter–gather

A scatter–gather request is as slow as its **slowest** shard. If each of 20 shards answers within 100 ms 99% of the time:

```text
 P(all 20 shards are fast) = 0.99^20 ≈ 0.82
 → about 18% of scatter–gather requests wait for at least one shard's 1-in-100 slow response:
   the shards' p99 becomes roughly the request's p82; with 100 shards, 0.99^100 ≈ 0.37,
   so most requests see a "rare" slow response
```

Fan-out turns rare tail latency into common latency (Chapter 26 covers percentiles). Mitigations: fewer shards per query (better keys, global indexes), hedged requests to replicas for the slowest shards (if a shard has not answered within its usual time, send the same read to a replica and use whichever reply arrives first; Chapter 24), per-shard timeouts with partial results where acceptable, and pagination with per-shard limits.

### Mechanism: joins across shards

```text
 1. CO-LOCATE   customers, orders and order_lines all sharded by customer_id → the join of
                a customer's orders and lines runs inside one shard, like a normal SQL join
 2. REFERENCE   small, rarely changing tables (countries, currencies, plans) copied to every
                shard → joins with them are local
 3. APPLICATION query shard(s) for one side, then fetch the matching rows of the other side by
    JOIN        key from their shards (a batched N+1, Chapter 18, Section 18.11)
 4. DENORMALISE store the needed fields of the other entity with the first (Chapter 14,
                Section 14.5), maintained by events (Chapter 25)
 5. ANALYTICS   for large ad-hoc joins, copy data to a warehouse or column store
    ELSEWHERE   (CDC, Chapter 22, Section 22.10) and query it there
```

Distributed SQL systems (CockroachDB, YugabyteDB, Spanner, Citus for PostgreSQL, Vitess for MySQL) execute cross-shard joins for you, but the network costs of moving rows between nodes remain, and `EXPLAIN` (Chapter 16) shows them as distributed operators.

### Guarantees and non-guarantees

Local indexes guarantee index consistency with their shard's data. Global indexes guarantee single-partition lookups by value, usually without immediate consistency. Co-location guarantees local joins and transactions for data sharing a key. Nothing guarantees cheap queries that ignore the shard key, or consistent snapshots across shards without a distributed transaction mechanism (Section 23.9).

### Failure and diagnosis

- **A cross-shard query timing out.** Evidence: traces showing one request fanning out to all shards, with one or two slow; latency growing with shard count. Diagnosis Lab 2.
- **"Missing" recent items in a lookup by a global index.** Evidence: the row exists on its shard but not yet in the index partition. Cause: asynchronous index maintenance. Fix: read the shard by key when freshness matters; accept and document the lag.
- **Joins that move huge intermediate results between nodes.** Evidence: distributed plans with large data transfers. Fix: co-location, reference tables, or moving the workload to analytics.

### Backend connection

Design the data model so that the frequent queries and the transactions include the shard key, co-locate what is joined, broadcast small reference tables, and add global indexes or lookup tables only for the few frequent queries by other values. Treat scatter–gather as a cost that multiplies tail latency, and keep large ad-hoc queries in an analytical copy.

---

## 23.6 Time and Order: Why Clocks Cannot Order Events

### Definitions

- A **time-of-day clock** (wall clock) reports the current date and time (`CLOCK_REALTIME`, `System.currentTimeMillis()`). It is set and corrected from outside, so it can **jump** forwards or backwards.
- A **monotonic clock** (`CLOCK_MONOTONIC`, `System.nanoTime()`) only moves forwards, at roughly the right rate, from an arbitrary starting point. Its absolute value means nothing; only differences between readings on the **same** machine do.
- **Clock skew** is the difference between two machines' wall clocks at the same instant; **drift** is a clock running slightly fast or slow.
- **NTP** (Network Time Protocol) synchronises wall clocks with time servers over the network; it reduces skew but cannot remove it.
- A **logical clock** orders events by counting, not by measuring time. A **Lamport clock** gives each event a number consistent with causality; a **vector clock** additionally detects whether two events are **concurrent**.
- The **happened-before** relation (Lamport, 1978): event a happened before event b if they occur in that order in one process, or a is the sending of a message that b receives, or by chaining these. Events where neither happened before the other are **concurrent**.

### Why it matters

Distributed systems constantly need order: which write is newer (Chapter 22, Section 22.6), whether a lease has expired (Section 23.8), which request came first. Wall clocks look like the answer and are not: each machine's clock differs from the others by milliseconds (sometimes much more), they jump when NTP corrects them or a leap second is applied, and a virtual machine paused for live migration resumes with a clock that has leapt ahead. Two events on different machines cannot be ordered by comparing their timestamps unless the timestamps differ by more than the worst-case skew, which nobody knows exactly.

### Concrete: clock skew and a wrong last-write-wins outcome

```text
 node A's clock is 120 ms AHEAD of real time; node B's clock is correct
 real time   event                                                  timestamp recorded
 10:00:00.000  client 1 → node A: SET status = "shipped"          10:00:00.120   (A's clock)
 10:00:00.050  client 2 → node B: SET status = "cancelled"        10:00:00.050   (B's clock)
               (client 2 acted 50 ms LATER, after seeing "shipped")
 last write wins compares timestamps: 10:00:00.120 > 10:00:00.050
 → final status = "shipped"; the later cancellation is silently discarded
```

The cancellation happened after the shipment and was meant to override it, but the clocks said otherwise. Neither node did anything wrong; the timestamps were simply not comparable at this resolution.

### Mechanism: what NTP gives you

NTP estimates the offset to a time server from round-trip timestamps and slews (gradually adjusts) or steps the local clock. Within one data centre with good time sources, skew is often around a millisecond or less; over the public internet, tens of milliseconds; and when a server cannot reach its time source, or its oscillator misbehaves, drift accumulates without bound. A clock can also be **stepped** backwards: a duration computed as "end wall time − start wall time" can then be negative. Hence the first rule:

- **Measure durations and timeouts with the monotonic clock**, never with wall-clock differences. Use wall clocks only for timestamps meant for humans, and with an explicit time zone (Chapter 14, Section 14.12).

Some systems bound clock uncertainty explicitly: Google's Spanner uses GPS and atomic clocks to know that true time lies within an interval [earliest, latest] (TrueTime) and **waits** out the uncertainty before committing, so that commit timestamps respect real-time order. Without such hardware, systems use logical clocks, or **hybrid logical clocks** (a wall-clock reading combined with a logical counter, used by CockroachDB among others) that stay close to physical time while respecting causality.

### Concrete: a Lamport clock, worked through

Each process keeps a counter `L`:

```text
 rules:  before each local event or send:  L = L + 1;  a message carries its sender's L
         on receiving a message with value m:  L = max(L, m) + 1

 process A           process B           process C
 a1: L=1  (local)
 a2: L=2  send ─────► b1: L=max(0,2)+1=3
                     b2: L=4  send ─────► c1: L=max(0,4)+1=5
 a3: L=3  (local)                         c2: L=6
                     b3: L=5 (local)

 if x happened before y, then L(x) < L(y):  a1(1) < a2(2) < b1(3) < b2(4) < c1(5) < c2(6)  ✔
 the converse is NOT true:  a3 = 3 < b2 = 4, yet a3 and b2 are concurrent (no message links them)
```

Breaking ties by process ID gives a **total order** consistent with causality, which is enough for many uses (ordering operations in a log, deciding a deterministic winner). What Lamport clocks cannot do is say whether two events were concurrent.

### Mechanism: vector clocks, at intuition level

A **vector clock** keeps one counter per process: `[A, B, C]`. A process increments its own entry for each event and, on receiving a message, takes the element-wise maximum of its vector and the message's, then increments its own entry.

```text
 a2 = [2,0,0]   b1 = [2,1,0] (received a2)   a3 = [3,0,0]
 compare a3 = [3,0,0] with b1 = [2,1,0]:
   a3 has more A, b1 has more B → neither ≤ the other → CONCURRENT
 compare a2 = [2,0,0] with b1 = [2,1,0]: a2 ≤ b1 in every entry → a2 happened before b1
```

That is exactly what a multi-leader or leaderless store needs (Chapter 22, Sections 22.6–22.7): if one version's vector dominates the other's, keep the newer; if they are concurrent, keep both as siblings or merge, instead of letting a clock pick a winner. The cost is one counter per writer, which is why practical systems use **version vectors** with one entry per replica rather than per client.

### Guarantees and non-guarantees

A monotonic clock guarantees non-decreasing readings on one machine, good for measuring durations. NTP-synchronised wall clocks guarantee approximate agreement, with unknown, usually small, but unbounded error. Lamport clocks guarantee that causally ordered events have increasing numbers; vector clocks additionally identify concurrency. None of them guarantees agreement with real time across machines, except systems with bounded-uncertainty clocks that explicitly wait.

### Failure and diagnosis

- **Lost updates under last-write-wins.** Evidence: a later write overwritten by an earlier one; the winning node's clock was ahead. Fix: version vectors, conditional writes, or a single leader per key.
- **Negative or huge durations, early timeouts.** Evidence: latency metrics with negative values, timeouts firing immediately after an NTP step. Cause: durations computed from wall clocks. Fix: monotonic clocks.
- **Lease or token validation failures on one host.** Evidence: that host's clock offset (`chronyc tracking` or `timedatectl`) far from its peers. Fix: time synchronisation monitoring; alarms on offset (Chapter 20's JWT clock-skew lab is the same failure).

### Backend connection

Use monotonic clocks for every timeout, deadline and duration; use wall clocks only for human-facing timestamps; never order events from different machines by comparing wall-clock times when correctness depends on it. Where order matters, get it from a single leader's log, a consensus system (Section 23.7) or logical clocks, and monitor clock offset on every host.

---

## 23.7 Consensus and Raft

### Definitions

**Consensus** is getting several nodes to agree on a single value (or a sequence of values), such that:

- **agreement**: no two nodes decide differently;
- **validity**: the decided value was proposed by some node;
- **termination**: every node that does not crash eventually decides.

Repeated consensus over a sequence of values gives a **replicated log**: every node applies the same commands in the same order, producing a **replicated state machine**, the general way to make any deterministic service fault-tolerant.

**Raft** (Ongaro and Ousterhout, 2014) is a consensus algorithm designed for understandability, widely used in practice. **Paxos** (Lamport) is the older, equivalent family; ZooKeeper uses a related protocol called ZAB.

- A **term** is a numbered period with at most one leader. A **majority** (quorum) of a cluster of N nodes is ⌊N/2⌋ + 1.

### Why it exists

Chapter 22 needed consensus several times without building it: to elect exactly one leader during failover (Section 22.5), to store the partition map that routing trusts (Section 23.4), and to decide who holds a lock (Section 23.8). Agreement is hard because nodes crash, messages are delayed, lost or reordered, and a node cannot tell a crashed peer from a slow one. The classic **two generals problem** shows the root of the difficulty with lost messages. Two generals must attack at the same time, and can communicate only by messengers who may be captured. General A sends "attack at dawn"; B must confirm, or A cannot know the message arrived; but then B cannot know whether the confirmation arrived, so A must confirm the confirmation, and so on. Whichever message is sent last might have been lost, so no finite exchange makes both sides certain. This is the same uncertainty as a request that times out with an unknown outcome (Chapter 7, Section 7.15). Practical systems do not try to remove it. They make operations safe to retry and decide by majorities, as below. The **FLP result** (Fischer, Lynch and Paterson, 1985) proves that in a fully asynchronous system, where messages can be delayed arbitrarily, no deterministic algorithm can guarantee consensus terminates if even one node may crash. Practical algorithms escape it by using timeouts: they are always **safe** (never two different decisions) and **live** (make progress) whenever the network behaves well enough for timeouts to be meaningful.

### Mechanism: why majorities

```text
 5 nodes: any majority is 3 nodes. Any two majorities share at least one node:
   {A, B, C} and {C, D, E} overlap in C;  {A, B, C} and {A, D, E} overlap in A
 → if one majority decided value X (or elected leader L in term t), any later majority
   contains at least one node that knows it, and that node prevents a contradictory decision

 a cluster of N tolerates f failures with N = 2f + 1:   3 nodes → 1 failure;  5 nodes → 2
 a 4-node cluster also tolerates only 1 failure (majority 3), so odd sizes are used
 during a partition, only the side with a majority can make decisions; the other side waits
 (this is the CP choice of Chapter 22, Section 22.9)
```

### Concrete: Raft leader election, with terms and votes

Each node is a **follower**, a **candidate** or the **leader**. The leader sends periodic heartbeats. A follower that hears nothing for its **election timeout** (randomised, for example 150–300 ms) becomes a candidate:

```text
 term 1: leader A sends heartbeats to B, C, D, E
 A crashes
 C's election timeout fires first (randomised timeouts make simultaneous candidates unlikely)
   C: term = 2, votes for itself, sends RequestVote(term 2, last log index/term) to all
   B: term 2 is new to it → grants its vote to C (it votes for at most ONE candidate per term)
   D: grants its vote
   E: its log is MORE up to date than C's → refuses (the election restriction: a candidate must
      have a log at least as up to date as the voter's, so a new leader has every committed entry)
   C has 3 votes of 5 (itself, B, D) = a majority → C is leader for term 2, sends heartbeats
 A recovers, still thinking it leads term 1; it sends an entry stamped term 1; nodes answer
   "my term is 2"; A sees the higher term and steps down to follower
 two candidates splitting the votes → nobody gets a majority → timeouts fire again with new
   random values → a new term, usually decided at the next attempt
```

The **term** acts as the epoch of Chapter 22 (Section 22.5): every message carries it, and any node that sees a higher term immediately accepts it and stops acting as leader, which is how a stale leader is fenced.

### Concrete: Raft log replication and commit on majority

```text
 leader C (term 2) receives command "SET x = 5" from a client
 1. append to its own log: index 7, term 2, "SET x = 5"
 2. send AppendEntries(prev index 6 / term 2, entry 7) to all followers
 3. B, D accept (their entry 6 matches); E is slow; A is down
 4. C has the entry on {C, B, D} = 3 of 5 → entry 7 is COMMITTED
 5. C applies it to its state machine, answers the client, and tells followers the new commit
    index in the next messages; they apply entry 7 too
 a follower whose log disagrees (it missed entries, or has entries from an old leader that were
 never committed) is repaired: the leader walks back to the last matching index and overwrites
 the rest with its own entries
```

Once an entry is stored on a majority, any future leader must have it (the election restriction plus overlapping majorities), so committed entries are never lost while a majority survives. Each committed write therefore costs a round trip to a majority, which is why consensus systems are used for small, critical data (configuration, leadership, locks, metadata), or are partitioned into many small consensus groups (one per data range, as in CockroachDB and TiKV).

### Mechanism: where consensus is used

| System | Uses consensus for |
|---|---|
| etcd (Raft) | Kubernetes cluster state; leader election and configuration for many systems |
| ZooKeeper (ZAB) | coordination for Hadoop-era systems, older Kafka versions, locks, configuration |
| Consul (Raft) | service registry and key–value configuration |
| Kafka (KRaft mode) | cluster metadata, replacing ZooKeeper |
| Patroni and similar (via etcd, Consul or ZooKeeper) | PostgreSQL leader election and failover (Chapter 22, Section 22.5) |
| CockroachDB, TiKV, YugabyteDB | one Raft group per data range, replicating the data itself |

### Guarantees and non-guarantees

Raft guarantees that committed entries are never lost or reordered while a majority of nodes survives, that at most one leader exists per term, and that all nodes apply the same commands in the same order. It does not guarantee progress without a majority, low latency (every write waits for a majority round trip, plus elections after failures), or tolerance of nodes that lie (Byzantine faults, Section 23.13).

### Failure and diagnosis

- **Unavailable during a partition.** Evidence: the minority side's nodes reject writes ("no leader"). Expected; place nodes in three or more failure domains.
- **Election storms.** Evidence: frequent term increases in logs; leadership changing every few seconds. Causes: election timeouts too short for network or disk latency (a slow disk delays log writes and heartbeats), overloaded nodes. Fix: faster disks for the log, longer timeouts.
- **Slow writes.** Evidence: commit latency equal to the slowest majority member's round trip. Fix: keep members in nearby zones; do not stretch one group across distant regions unless the latency is acceptable.

### Backend connection

Do not implement consensus yourself: use etcd, ZooKeeper, Consul, or a database built on Raft. Use them for small, critical, strongly consistent state: who the leader is, partition maps, locks, configuration. Run three or five members across failure domains, on fast disks, and remember that every write costs a majority round trip.

---

## 23.8 Distributed Locks, Leases and Fencing Tokens

### Definitions

- A **distributed lock** gives one process, among many on different machines, exclusive permission to do something: write to a resource, run a job, act as leader.
- A **lease** is a lock with an expiry time: the holder owns it until the lease expires unless it renews it first. Leases are necessary because a crashed holder would otherwise hold a lock forever.
- A **fencing token** is a number, increasing with every grant of the lock, that the holder sends with each operation on the protected resource; the **resource** rejects any operation carrying a token lower than the highest it has already seen.

### Why it exists

Chapter 11 (Section 11.7) showed that in-process locks protect nothing across instances. Chapter 17 (Section 17.8) used database locks inside one database. When the protected thing is outside any single database (a file in object storage, an external API, a batch job that must run once), applications reach for a distributed lock. The difficulty is not taking the lock but knowing that you **still** hold it at the moment you act: a process can be paused at any instant (garbage collection, Chapter 2, Section 2.8; a VM pause; a page fault on swapped memory, Chapter 3, Section 3.17) for longer than its lease.

### Concrete: the lease that expired during a GC pause

```text
 lock service grants leases of 10 s; storage holds file report.csv
 t=0     client 1 acquires the lease
 t=1     client 1 starts writing report.csv … then a stop-the-world GC pause begins
 t=10    client 1's lease EXPIRES (client 1 is frozen and does not know)
 t=11    client 2 acquires the lease, writes report.csv, finishes at t=14
 t=16    client 1's GC pause ends; it continues exactly where it stopped, still believing it holds
         the lock, and overwrites report.csv with its half-finished, older data
 result: two "exclusive" holders; the file is corrupted. Client 1 could even have checked
         "do I still hold the lease?" at t=9.9, just before the pause: the check passes and
         the pause comes after it.
```

No amount of checking by the client can close this gap, because the pause can happen between the check and the write. The protection has to be at the **resource**.

### Concrete: the fencing-token fix

```text
 the lock service returns an increasing token with each grant
 t=0     client 1 acquires the lease, token 33
 t=1     client 1 writes with token 33 → storage records "highest token seen: 33"
         … GC pause …
 t=11    client 2 acquires the lease, token 34; writes with token 34 → storage: highest = 34
 t=16    client 1 resumes and writes with token 33 → storage REJECTS it (33 < 34)
```

The token is the same idea as Raft's term and Chapter 22's epochs. Sources of suitable tokens: the revision number etcd returns for a key, ZooKeeper's transaction ID (zxid) or sequential node numbers, or a database sequence or row version incremented in the same transaction that grants the lock. The resource must support the check: a database `UPDATE … WHERE fence < $token` (Chapter 17, Section 17.11), object storage with conditional writes, or an API that accepts and enforces a fencing value.

### Mechanism: why "a Redis lock" is subtle

The common pattern (Chapter 21, Section 21.8) is `SET lock:job <random-value> NX PX 30000`, deleting the key only if it still holds your value. Its weak points:

1. **Pauses**, as above: a lease in Redis expires during a pause like any other, and `SET NX` provides no fencing token by itself.
2. **Asynchronous replication**: the lock key is written to the Redis primary, the primary fails before replicating it, a replica is promoted without the key, and a second client acquires the "same" lock (Chapter 22, Section 22.3).
3. **Clock assumptions**: expiry depends on the server's clock rate and the client's estimate of elapsed time.

The **Redlock** algorithm takes the lock on a majority of independent Redis nodes to address failover, and its safety under pauses and clock jumps has been publicly debated (notably in Martin Kleppmann's 2016 analysis and the Redis author's reply). The practical conclusion this book uses:

- a Redis lock is fine for **efficiency**: avoiding duplicate work most of the time, where an occasional double execution is harmless (two processes both warming a cache);
- for **correctness**, where two holders would corrupt data or double-charge, use a lock service built on consensus that issues fencing tokens (etcd, ZooKeeper), or the database's own locks and constraints, and make the resource check the token.

### Guarantees and non-guarantees

A lease guarantees that a crashed holder does not block others forever. A consensus-backed lock service guarantees that at most one client is **granted** the lease at a time. Only fencing at the resource guarantees that operations from a holder whose lease has expired are rejected. Nothing guarantees that the holder knows its lease is still valid at the moment of acting.

### Failure and diagnosis

- **Two holders.** Evidence: overlapping "lock acquired" and "work done" log intervals from two processes; a GC or pause log on one of them spanning the lease expiry. Diagnosis Lab 3.
- **Lock lost on failover.** Evidence: a second acquisition right after a cache failover. Fix: a consensus-backed store, or fencing.
- **Locks held too long.** Evidence: leases renewed for hours by a stuck process. Fix: bound the work done under a lock; monitor holders.

### Backend connection

Before using a distributed lock, ask whether the database can do the job (a unique constraint, a conditional update, `SELECT … FOR UPDATE SKIP LOCKED`, Chapter 17), because those are protected by the database's own transaction machinery. When a lock is needed for correctness, get it from a consensus system with fencing tokens and make the protected resource reject stale tokens; use cache-based locks only where a rare double execution is acceptable.

---

## 23.9 Distributed Transactions and Two-Phase Commit

### Definitions

A **distributed transaction** is one that changes data on several nodes (shards, databases, or other resources such as a message broker) and must commit on all or none. **Atomic commitment** is the problem of making all participants reach the same decision.

**Two-phase commit** (2PC) solves it with a **coordinator** and **participants**:

1. **Prepare phase**: the coordinator asks every participant to **prepare**: write everything needed to commit durably, take the locks, and vote **yes** (promising it can commit) or **no**.
2. **Commit phase**: if every vote is yes, the coordinator durably records **commit** and tells everyone to commit; otherwise it tells everyone to abort.

A participant that voted yes but has not yet heard the decision is **in doubt**: it may neither commit nor abort on its own. **XA** is a standard interface for 2PC across heterogeneous resources; PostgreSQL exposes participant-side 2PC as `PREPARE TRANSACTION` / `COMMIT PREPARED`, and MySQL as `XA PREPARE` / `XA COMMIT`.

### Why it exists

A single database makes a transaction atomic with its own log (Chapter 15, Section 15.4). When a transfer debits an account on shard 1 and credits one on shard 2, the two shards' logs are separate: one can commit while the other crashes. Some coordination protocol is needed so that both commit or both abort, and 2PC is the classic one.

### Concrete: the 2PC sequence

```text
 coordinator                       shard 1 (debit)                    shard 2 (credit)
 BEGIN distributed tx 77
 ── PREPARE 77 ──────────────────► write redo/undo durably, keep      ── PREPARE 77 ──────────────►
                                   locks on account 1, vote YES        same, locks on account 2, YES
 ◄── YES ─────────────────────────                                     ◄── YES ─────────────────────
 write "COMMIT 77" to the coordinator's own log (fsync)   ← the decision point
 ── COMMIT 77 ───────────────────► commit, release locks              ── COMMIT 77 ───────────────►
 ◄── done ────────────────────────                                     ◄── done ────────────────────
```

If any participant votes no (a constraint fails, a timeout), the coordinator logs **abort** and tells everyone to roll back.

### Concrete: the coordinator crashes after prepare (blocking)

```text
 shard 1 and shard 2 have voted YES and are holding locks on accounts 1 and 2
 coordinator crashes BEFORE telling anyone the decision (it may or may not have logged COMMIT)

 shard 1: in doubt. If it commits on its own, shard 2 may abort → inconsistent.
          If it aborts on its own, the coordinator may have decided COMMIT and told shard 2
          (which then committed) → inconsistent.
          → it must WAIT, holding its locks on account 1
 shard 2: the same, holding locks on account 2
 every other transaction touching accounts 1 or 2 waits behind those locks
 the system is BLOCKED until the coordinator recovers and reads its decision from its log
 (or an operator inspects the coordinator's log and resolves each in-doubt transaction by hand)
```

That is why 2PC is called a **blocking** protocol: one node's failure at the wrong moment stops progress for everyone holding related locks, for as long as that node is down. Participants cannot even ask each other, because if all of them voted yes, only the coordinator knows whether it decided commit.

### Mechanism: why microservices avoid it, and what replaces it

| Cost of 2PC | Consequence |
|---|---|
| blocking on coordinator failure | locks held for minutes or hours; outages spread to unrelated requests |
| locks held across the network round trips of both phases | lower throughput, more contention (Chapter 17, Section 17.5) |
| every participant must support prepare (XA) | most HTTP APIs, many brokers and SaaS services do not |
| coupling | one slow or unavailable service stalls transactions in all the others |
| operational burden | in-doubt transactions to monitor and resolve; in PostgreSQL, prepared transactions also hold back VACUUM (Chapter 17, Section 17.12) |

Systems that need cross-shard atomicity inside **one** database product (Spanner, CockroachDB and others) still use 2PC internally, but run the coordinator and participants as **replicated consensus groups** (Section 23.7), so that a coordinator "failure" is just a leader change within a group that still knows the decision: the protocol no longer blocks on a single machine. Across independently owned services, the usual answer is to avoid atomic commitment altogether: keep each transaction inside one service's database, and coordinate multi-step business operations with **sagas**, sequences of local transactions with compensating actions, triggered reliably through an outbox (Chapter 25). (Three-phase commit, an older proposal to avoid blocking, assumes bounded network delays and is rarely used.)

### Guarantees and non-guarantees

2PC guarantees atomic commitment: all participants commit or all abort, provided the coordinator's decision is durable and eventually delivered. It does not guarantee progress while the coordinator is unavailable after participants voted yes, low latency, or isolation beyond what each participant provides; and it guarantees nothing for resources that cannot prepare.

### Failure and diagnosis

- **Stuck in-doubt transactions.** Evidence: PostgreSQL `pg_prepared_xacts` entries hours old; MySQL `XA RECOVER` listing transactions; lock waits behind them; VACUUM unable to clean (old xmin). Fix: recover the coordinator, or resolve each one by hand with the coordinator's log as the authority.
- **Throughput collapse under contention.** Evidence: lock waits during prepare windows. Fix: keep transactions within one shard by co-locating data (Section 23.5).

### Backend connection

Design data placement so that transactions stay within one shard or one service. Use a database's own distributed transactions where it provides them on consensus-replicated coordinators, and avoid hand-rolled 2PC across services; for multi-service business processes, use sagas and the outbox pattern (Chapter 25). If you do run 2PC, monitor in-doubt transactions as an incident signal.

---

## 23.10 Beyond Relational: Data Models and Their Engines

### Definitions

**NoSQL** ("not only SQL") is an umbrella name for data stores that do not use the relational model as their primary model (Chapter 14, Section 14.1 named the families). The main ones:

- A **key–value store** maps opaque keys to values; operations are get, put and delete by key (Redis, DynamoDB in its simplest use, etcd, RocksDB as an embedded engine).
- A **document store** stores self-contained, nested documents (JSON or a binary form), queried by fields inside them (MongoDB, Couchbase; PostgreSQL's `jsonb` offers the model inside a relational database, Chapter 14, Section 14.12).
- A **wide-column store** (column-family store) stores rows grouped by a partition key, each holding many columns sorted by a clustering key, designed for very high write volumes (Cassandra, ScyllaDB, HBase, Bigtable).
- A **graph database** stores **nodes** and **edges** with properties and queries paths through them (Neo4j with the Cypher language; RDF stores with SPARQL).
- A **search engine** stores documents in **inverted indexes** (Chapter 16, Section 16.6) for full-text relevance search and aggregations (Elasticsearch, OpenSearch, Solr).
- A **time-series database** stores timestamped measurements per series, optimised for appends, time-range queries, downsampling and retention (TimescaleDB, InfluxDB, Prometheus's storage).

**BASE** ("basically available, soft state, eventually consistent") is a slogan, coined as a contrast to ACID, for systems that favour availability and accept that replicas converge over time (Chapter 22, Section 22.8). It describes a design attitude, not a precise guarantee: many "NoSQL" systems now offer transactions and strong consistency options, and many relational systems are deployed with asynchronous replicas.

### Why they exist

The relational model is general, and its engines are excellent at transactional workloads with flexible queries. Other models exist because some workloads have a shape that a different model serves more directly: lookups by key at enormous scale, aggregates read and written as one document, append-heavy writes partitioned by an entity, traversals of many-hop relationships, ranked text search, or billions of timestamped points. Each specialised store trades away generality (ad-hoc joins, multi-record transactions, rich constraints) for speed, scale or convenience on its shape.

### Mechanism: models, queries and engines

| Model | Data model | Query model | Typical storage engine | Scaling |
|---|---|---|---|---|
| Relational | tables, rows, keys, constraints | SQL: joins, aggregates, transactions | B+trees with WAL (Chapter 15, Section 15.6) | vertical, replicas; sharding by extension or distributed SQL |
| Key–value | key → opaque value | get/put/delete by key; sometimes ranges | hash tables in memory, LSM-trees (RocksDB) | hash partitioning (Section 23.2) |
| Document | nested documents, flexible schema | by key and by fields; secondary indexes; aggregation pipelines; limited joins | B-trees (MongoDB's WiredTiger) | sharding by a document key |
| Wide-column | partition key → sorted rows of columns | by partition key, ranges of the clustering key; no joins; queries designed per table | LSM-trees with SSTables and compaction | hash partitioning, leaderless replication (Chapter 22, Section 22.7) |
| Graph | nodes, edges, properties | traversals and pattern matching (Cypher: `MATCH (a)-[:FOLLOWS*1..3]->(b)`) | adjacency stored next to nodes ("index-free adjacency") or on top of key–value stores | hard to shard well (traversals cross shards) |
| Search | documents analysed into terms | relevance-ranked full text, filters, facets, aggregations | inverted indexes in immutable segments, merged in the background (like LSM compaction) | fixed number of shards per index, replicas |
| Time-series | series (metric + labels) → (timestamp, value) points | time ranges, rollups, rates, downsampling | append-optimised, compressed by time (delta and XOR encodings, Chapter 15, Section 15.7) | partitioned by time and series |

### Concrete: one domain, modelled three ways

An order with its lines:

```text
 RELATIONAL (normalised, Chapter 14)        orders(id, customer_id, status, …)
                                            order_lines(order_id, product_id, quantity, unit_cents)
                                            any query by any column; joins; constraints; transactions

 DOCUMENT (the aggregate as one document)   { "_id": "o_9001", "customer": "c_81", "status": "PAID",
                                              "lines": [ {"product": "p_17", "qty": 2, "unit": 1999},
                                                         {"product": "p_42", "qty": 1, "unit": 999} ] }
                                            read and write the whole order in one operation; product
                                            names copied in (denormalised) or referenced by ID

 WIDE-COLUMN (one table per query)          orders_by_customer: partition key customer_id,
                                                                clustering key created_at DESC
                                            → "latest orders of customer 81" is one partition scan;
                                            "orders by product" needs ANOTHER table, written too
```

The document model fits when the application reads and writes **aggregates** (Chapter 12, Section 12.3) as a unit and rarely queries across them. The wide-column model fits when a few known queries must scale to huge write volumes, at the cost of designing (and writing) one table per query. The relational model fits when queries are varied, relationships matter, and integrity must be enforced by the database.

### Guarantees and non-guarantees

Each model guarantees efficient operations on its own shape. None guarantees efficient operations outside it: a key–value store cannot answer "all orders over 100 euros", a wide-column store cannot join, a document store makes many-to-many relationships awkward, a search engine is not a system of record (its indexes are refreshed near-real-time and can lose acknowledged writes in some failure modes), and graph traversals that cross shards are slow. "Schemaless" stores still have a schema; it lives implicitly in the application code that reads the data.

### Failure and diagnosis

- **Joins rebuilt in application code.** A document store used for highly relational data; requests issue dozens of lookups to assemble one page. Evidence: N+1 patterns (Chapter 18, Section 18.11) against the store. Fix: a relational store for that data, or denormalised documents designed for the reads.
- **Search index used as the source of truth.** Data lost or inconsistent after an index rebuild or a cluster incident. Fix: the database as source of truth; the index derived from it by CDC (Chapter 22, Section 22.10).
- **Wide-column queries that scan the cluster.** Evidence: queries without the partition key, or with filtering that the store warns about. Fix: a table designed for that query.

### Backend connection

Learn each store's model by its **queries**, not its marketing: what can it answer in one operation, what does it make expensive, what does it guarantee about consistency and transactions? Model data for the reads you will run (aggregates for documents, one table per query for wide-column), and keep a relational system of record unless the workload clearly needs something else.

---

## 23.11 Choosing a Data Store

### Definitions

**Choosing a data store** is matching a workload's requirements (access patterns, consistency and integrity needs, scale, latency and operational constraints) to a store's data model, guarantees and costs. **Polyglot persistence** is using several kinds of stores in one system, each for the data it serves best. The **source of truth** (system of record, Chapter 1, Section 1.6) is the one store whose data wins when copies disagree.

### Why it matters

"SQL or NoSQL?" is usually asked as a matter of taste or fashion ("NoSQL scales, SQL doesn't"; "MongoDB is schemaless, so it's faster to develop"). Both slogans are wrong often enough to cause real harm: relational databases scale far beyond most companies' needs, and schemaless stores move the schema into every piece of code that reads the data. The decision should follow from requirements, and it is expensive to reverse.

### Mechanism: the questions, in order

```text
 1. ACCESS PATTERNS   what are the top 10 queries and writes, with rates and latency targets?
                      by key only? by many fields? ranges? joins? full text? graph traversals?
 2. INTEGRITY         which invariants must the store enforce (uniqueness, foreign keys, balances)?
                      which operations must be atomic across records?
 3. CONSISTENCY       which reads must be current (Chapter 22, Section 22.8)? what may be stale,
                      and for how long?
 4. SCALE             data size and write rate now and in 3 years; does it exceed one node
                      (Section 23.1)? read/write ratio?
 5. AVAILABILITY      behaviour required during node, zone and region failures (CAP choices per
                      operation, Chapter 22, Section 22.9)
 6. OPERATIONS        who runs it? backups and restore drills, upgrades, monitoring, team skills,
                      managed offerings, cost
```

Most transactional business data (orders, payments, accounts, inventory) answers question 2 with "many invariants, multi-record atomicity" and question 4 with "fits on one large node, perhaps with replicas", which points to a relational database. Specialised stores earn their place where one of the questions has an answer the relational engine serves poorly.

### Concrete: a selection table

| Store type | Model | Query pattern it serves | Consistency (typical) | Scaling | Typical use | Typical misuse |
|---|---|---|---|---|---|---|
| Relational (PostgreSQL, MySQL) | tables with constraints | varied queries, joins, aggregates, transactions | strong on the leader; ACID transactions | vertical + replicas; sharding with effort | orders, payments, accounts, most business data | huge append-only telemetry; full-text relevance search |
| Distributed SQL (CockroachDB, Spanner, YugabyteDB) | relational, sharded with consensus | SQL with transactions across shards | strong (serialisable), at higher write latency | horizontal | relational data beyond one node; multi-region with consistency | small apps that fit one node (pays latency and cost for nothing) |
| Key–value (Redis, DynamoDB, etcd) | key → value | get/put by key, small values | Redis: async replication; DynamoDB: eventual or strongly consistent reads; etcd: linearizable | partitioning by key | caches, sessions, counters, configuration, user state by ID | ad-hoc queries; data whose loss matters without durability settings |
| Document (MongoDB) | nested documents | by key and indexed fields, aggregate-at-a-time | configurable (write and read concerns) | sharding by key | content, catalogues, per-entity aggregates with varying shape | highly relational data with many-to-many joins |
| Wide-column (Cassandra, ScyllaDB) | partitioned sorted rows | known queries by partition key at very high write rates | tunable (quorums) | linear horizontal, multi-region | messaging, event logs, IoT, per-user timelines | ad-hoc queries, joins, frequent updates of the same row, invariants |
| Graph (Neo4j) | nodes and edges | multi-hop traversals, pattern matching | per product | mostly vertical | recommendations, fraud rings, network topology, permissions graphs (ReBAC, Chapter 20, Section 20.9) | simple lookups and tables modelled as graphs |
| Search (Elasticsearch, OpenSearch) | analysed documents | full-text relevance, filters, facets, log search | near-real-time, eventually consistent | shards + replicas | product search, log analytics | system of record; transactional data |
| Time-series (TimescaleDB, InfluxDB, Prometheus) | timestamped series | time ranges, rollups, rates | per product | partitioned by time | metrics, sensor data, financial ticks | entity data with frequent updates |
| Column store / warehouse (ClickHouse, BigQuery, Snowflake) | columnar tables (Chapter 15, Section 15.7) | large scans and aggregations | batch or near-real-time loads | massive parallel scans | analytics, reporting | OLTP point reads and updates |

### Mechanism: SQL versus NoSQL without slogans

| Slogan | What is actually true |
|---|---|
| "NoSQL scales, SQL doesn't." | Relational databases handle very large datasets and request rates on one node plus replicas, and distributed SQL scales horizontally. NoSQL stores scale specific access patterns by giving up general queries and multi-record transactions. |
| "NoSQL is schemaless, so development is faster." | The schema moves into application code and data migrations become code paths that handle every historical shape. Relational migrations are explicit (Chapter 18, Section 18.15). |
| "NoSQL is eventually consistent." | Many offer strong consistency options; many relational deployments read from asynchronous replicas. Consistency is a per-operation setting (Chapter 22). |
| "SQL is old." | Its model has decades of optimisers, tooling and operational knowledge behind it; that is an advantage. |

### Mechanism: polyglot persistence with one source of truth

```text
                    PostgreSQL (SOURCE OF TRUTH: orders, products, customers)
                         │ CDC / outbox (Chapter 22, Section 22.10; Chapter 25)
          ┌──────────────┼──────────────────────┬─────────────────────┐
          ▼              ▼                      ▼                     ▼
   Elasticsearch     Redis cache            ClickHouse             recommendations
   (product search)  (hot reads, sessions)  (analytics)            (graph, rebuilt nightly)
   derived, rebuildable from the source; may lag; never written directly by request paths
```

The rules that keep polyglot persistence sane: each piece of data has **exactly one** source of truth; derived stores are fed **from** it (CDC or outbox), never by dual writes from request handlers; every derived store can be **rebuilt** from the source; and each added store is justified by a workload the source cannot serve, because each costs operations, monitoring, backups and expertise.

### Guarantees and non-guarantees

A deliberate choice guarantees that the main access patterns are served by a store designed for them, with known consistency and integrity. Polyglot persistence with one source of truth guarantees that disagreements are resolved in the source's favour and derived stores can be rebuilt. Nothing guarantees that a store chosen for today's access patterns fits next year's; that is why the source of truth should be the most general store you can afford.

### Failure and diagnosis

- **The wrong store for the shape.** Symptoms: constant application-side joins, integrity bugs (duplicates, orphans) that a constraint would prevent, ad-hoc report requests the store cannot answer. Fix: move the system of record to a store matching its shape; keep the specialised store as a derived view.
- **Two sources of truth.** Symptoms: data that disagrees between stores with no rule for which is right. Fix: designate one source; derive the other.
- **Too many stores.** Symptoms: each store poorly operated, without backups or monitoring. Fix: consolidate; PostgreSQL alone (with `jsonb`, full-text search, table partitioning, extensions) covers more cases than teams expect.

### Backend connection

Start with a relational database as the system of record unless a requirement clearly rules it out, add specialised stores as derived views for specific workloads (search, caching, analytics, time series), and feed them from the source of truth. Write the decision down with its access patterns and guarantees, so that the next engineer knows why each store exists and what it may be trusted for.

---

## 23.12 Diagnosis Toolkit: Seeing a Sharded, Coordinated System

### Definition

The **distributed-data diagnosis toolkit** is the set of per-shard metrics, request traces, coordination-service views and clock checks that show where load concentrates, how requests fan out, who holds leadership and locks, and whether nodes agree on time.

### Why it matters

Averages hide sharded problems. A cluster at 30% CPU on average can have one shard at 100%; a p99 that looks fine on each shard can be poor for requests that touch twenty of them; a lock can be held by two processes without either logging an error. Each problem in this chapter needs its own signal.

### Concrete: the signals

| Question | Signal | Tools (examples) |
|---|---|---|
| Is load even across shards? | per-shard CPU, request rate, data size, top keys per shard | database per-node metrics; Cassandra `nodetool tablestats`; Redis Cluster per-node `INFO`, `--hotkeys`; MongoDB per-shard stats |
| How far do requests fan out? | shards contacted per request; per-shard spans in traces | distributed tracing (Chapter 26); query logs with shard IDs |
| Is rebalancing running? | partition moves in progress, bytes transferred, throttling | the cluster's balancer status |
| Who leads? Is consensus healthy? | current leader, term, commit index, election count | `etcdctl endpoint status` (shows leader, raft term and index), ZooKeeper `srvr`/`stat`, Raft metrics |
| Who holds a lock, with which token? | lock key, holder, lease expiry, revision | `etcdctl get --prefix /locks`, application logs that record tokens |
| Are clocks in sync? | offset from time servers per host | `chronyc tracking`, `timedatectl`, NTP exporters |
| Are distributed transactions stuck? | in-doubt prepared transactions and their age | `pg_prepared_xacts`, `XA RECOVER` |
| Did a process pause? | GC pause logs, safepoint times, VM steal time (Chapter 3, Section 3.12) | JVM GC logs, runtime metrics |

### Backend connection

Instrument per shard and per key, not only per cluster; record the fan-out and the shard of every request in traces; log lock acquisitions and fencing tokens; and alert on clock offset and in-doubt transactions. These are the signals the Diagnosis Lab below relies on.

---

## 23.13 Placement-Round Topics: Data-Warehouse Modelling, OLAP Operations and Byzantine Faults

> **Placement-round focus:** This section is revision material for DBMS and distributed-systems questions in placement rounds and university courses. It is not part of the main flow. What carries into backend work: star schemas are how analytical copies of OLTP data are commonly modelled (Chapter 15, Section 15.7), and the crash-fault assumption explains why Raft needs only a majority.

### Star and snowflake schemas

A **data warehouse** is a database built for analysis, loaded from operational systems. Its common model is **dimensional**:

- A **fact table** records measurable events at a chosen **grain** (one row per order line, per page view), with numeric **measures** (quantity, revenue) and foreign keys to dimensions.
- A **dimension table** describes the context of facts: date, product, customer, store, with descriptive attributes used for filtering and grouping.
- A **star schema** has one fact table joined directly to denormalised dimension tables. A **snowflake schema** normalises dimensions into sub-tables (product → category → department), saving space at the cost of more joins.

```text
 STAR                                              SNOWFLAKE (product dimension normalised)
              dim_date                                       dim_date
                 │                                              │
 dim_customer ─ fact_sales ─ dim_product            dim_customer ─ fact_sales ─ dim_product ─ dim_category
                 │          (name, category,                    │                              │
              dim_store      department)                     dim_store                   dim_department
 fact_sales(date_key, customer_key, product_key, store_key, quantity, revenue_cents)
```

A **slowly changing dimension** handles attributes that change over time: overwrite (type 1), or add a new row with validity dates so that history keeps the old value (type 2). Data reaches the warehouse by **ETL** (extract, transform, load) or **ELT** (load raw, transform inside the warehouse).

### OLAP operations

On a **data cube** (measures by several dimensions):

- **Roll-up**: aggregate to a coarser level (daily → monthly; city → country); SQL `GROUP BY ROLLUP` (Chapter 14, Section 14.8).
- **Drill-down**: the reverse, to finer detail.
- **Slice**: fix one dimension to one value (year = 2026).
- **Dice**: restrict several dimensions to subsets (year in 2025–2026 and region in EU, US).
- **Pivot**: rotate the view, turning rows into columns.

### Crash faults versus Byzantine faults

Raft and Paxos assume **crash faults**: nodes may stop, lose messages or be slow, but never lie. A **Byzantine fault** is a node behaving arbitrarily: sending contradictory or forged messages (a bug, corrupted memory, or an attacker). The **Byzantine generals problem** (Lamport, Shostak and Pease, 1982) shows that tolerating f Byzantine nodes requires at least **3f + 1** nodes (versus 2f + 1 for crash faults), and Byzantine fault-tolerant protocols (PBFT and successors, and the consensus of many blockchains) pay far more messages per decision. Systems inside one organisation's data centres normally assume crash faults and defend against corruption with checksums and authentication instead.

### Drill for this section

**C1. Design a star schema for an online shop's sales and explain the grain.**
A strong answer must mention:
- A fact table at a stated grain (one row per order line) with measures (quantity, revenue) and dimension keys.
- Dimensions: date, product, customer, store or channel, with descriptive attributes.
- Star versus snowflake (denormalised versus normalised dimensions) and slowly changing dimensions.
*Common weak answer:* copying the OLTP schema unchanged; the warehouse is modelled for analysis, not for transactions.

**C2. Name and explain the OLAP operations.**
A strong answer must mention:
- Roll-up and drill-down change the level of aggregation.
- Slice fixes one dimension; dice restricts several.
- Pivot rotates the presentation.
*Common weak answer:* treating slice and dice as synonyms.

**C3. Why does tolerating Byzantine faults need 3f + 1 nodes, while crash faults need 2f + 1?**
A strong answer must mention:
- Crash faults: a majority quorum overlaps any other majority in at least one correct node.
- Byzantine: quorums must overlap in enough nodes that correct ones outnumber faulty ones that can lie; with n = 3f + 1 and quorums of 2f + 1, any two quorums share at least f + 1 nodes, at least one of them correct.
- Crash-fault protocols (Raft) are used inside trusted environments; BFT protocols cost more messages.
*Common weak answer:* "Byzantine nodes are just slower crashed nodes"; they can send conflicting information.

---

## Chapter Summary

**Partitioning** (sharding) splits data across nodes when its size, write rate or hot working set exceed one machine, usually combined with replication of each shard; it should come after tuning, bigger machines, replicas, caching and functional splits. **Key-range** partitioning keeps ranges together and risks hot spots for sequential keys; **hash** partitioning spreads keys and scatters range queries; **directory** partitioning places groups explicitly; compound keys (hashed partition key plus sort key) combine them. The **shard key** should have high cardinality, appear in the dominant queries, and keep transactions on one shard. **Consistent hashing** with virtual nodes, or a fixed large number of partitions mapped to nodes, moves about 1/(N+1) of the data when a node joins instead of the N/(N+1) of mod-N. **Skew** concentrates load on hot keys and heavy tenants (mitigated by salting, caching, dedicated placement); **rebalancing** moves partitions in throttled steps; **routing** sends requests to owners using a versioned map kept in a consensus store. **Local** secondary indexes force scatter–gather queries, whose latency is set by the slowest shard; **global** indexes allow single-partition lookups but update asynchronously; joins are made local by co-location and reference tables.

**Clocks** cannot order events across machines: wall clocks are skewed and jump, so last-write-wins can discard a later write; durations must use **monotonic clocks**; **Lamport clocks** give a causal total order and **vector clocks** detect concurrency. **Consensus** makes nodes agree despite crashes; it needs **majorities**, because any two majorities overlap, so a cluster of 2f + 1 tolerates f failures and the minority side of a partition stops. **Raft** elects one leader per **term** by votes from a majority (with randomised timeouts and an up-to-date-log restriction) and commits a log entry once a majority stores it; etcd, ZooKeeper, Consul, KRaft and distributed SQL databases are built on such protocols.

A **distributed lock** needs a **lease** so crashed holders do not block forever, and a lease can expire while its holder is paused, so correctness needs **fencing tokens** checked by the resource; locks in Redis serve efficiency, consensus-backed locks with tokens serve correctness. **Two-phase commit** gives atomic commitment through a prepare vote and a durable decision, but **blocks** participants that voted yes while the coordinator is unreachable, holding locks; databases run it over consensus-replicated coordinators, and services avoid it in favour of local transactions and **sagas** (Chapter 25). Beyond relational, **key–value**, **document**, **wide-column**, **graph**, **search** and **time-series** stores each serve one shape of access efficiently and others poorly, and **BASE** names an attitude rather than a guarantee. A data store is chosen from access patterns, integrity, consistency, scale, availability and operations, usually with a relational **source of truth** and specialised stores derived from it by CDC.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Sharding | Data and writes divided across nodes | Even load; cheap cross-shard queries; cross-shard transactions; global uniqueness |
| Range partitioning | Ranges on few shards | Freedom from hot spots for sequential keys |
| Hash partitioning | Even spread of distinct keys | Efficient range queries; even load under skewed access |
| Consistent hashing / fixed partitions | About 1/(N+1) of data moves when a node joins | Free data movement; balanced load for hot keys |
| Routing with versioned maps | Requests reach the current owner or are redirected | Instant visibility of topology changes to every client |
| Local secondary index | Consistency with its shard's data | Queries without scatter–gather |
| Global secondary index | Single-partition lookup by value | Immediate consistency with the data |
| Monotonic clock | Non-decreasing readings on one machine | Meaningful comparison across machines |
| Lamport clock | Causally ordered events have increasing numbers | Detection of concurrency |
| Vector clock | Detection of concurrent versus ordered events | Small metadata with many writers |
| Raft | One leader per term; committed entries survive while a majority does | Progress without a majority; low latency; Byzantine tolerance |
| Lease | No permanent blocking by a crashed holder | That the holder still holds it when acting |
| Fencing token | Resource rejects operations from expired holders | Protection where the resource does not check tokens |
| 2PC | All participants commit or all abort | Progress while the coordinator is down after prepare |
| Specialised data stores | Efficiency on their own access shape | Efficiency, integrity or transactions outside it |
| Polyglot persistence with one source of truth | Clear resolution of disagreements; rebuildable derived stores | Freshness of derived stores |

---

## Interview Drill

**Q1. Why and how do you shard a database? How do you choose a shard key?**
A strong answer must mention:
- Shard when data size, write throughput or the hot set exceed one node, after tuning, scaling up, replicas, caching and functional splits.
- Each shard holds a disjoint part; shards are usually replicated.
- Shard key: high cardinality, even load, present in the dominant queries, keeping transactions on one shard (tenant or user ID).
- Costs: cross-shard queries, transactions, rebalancing, global uniqueness.
*Common weak answer:* "Split the table by ID ranges across servers." It ignores hot spots, query patterns and transactions.

**Q2. Range versus hash partitioning?**
A strong answer must mention:
- Range: efficient range scans; hot spots for sequential keys (timestamps, increasing IDs).
- Hash: even spread; range queries scatter to all shards.
- Compound keys: hashed partition key plus sort key within the partition.
- Example: events by date (one hot shard) versus by hashed device ID with time as the sort key.
*Common weak answer:* "Hash is always better because it balances." Not for range-heavy access.

**Q3. What is consistent hashing, and why use virtual nodes?**
A strong answer must mention:
- Keys and nodes hashed onto a ring; a key belongs to the next node clockwise.
- Adding a node moves about 1/(N+1) of keys, versus N/(N+1) with mod-N (80% from 4 to 5 nodes).
- Virtual nodes even out ownership and spread a leaving node's data over many nodes; weights for different capacities.
- Fixed partitions mapped to nodes as the practical variant (Redis Cluster's 16,384 slots).
*Common weak answer:* "It hashes keys consistently." It misses the movement arithmetic and virtual nodes.

**Q4. How do you handle hot keys, skew and rebalancing?**
A strong answer must mention:
- Measure per-shard and per-key load; skew comes from access, not key counts.
- Hot keys: caching, salting into several keys, replicas; heavy tenants on dedicated shards.
- Rebalancing: fixed partitions moved whole, dynamic splitting, throttled moves, human or rate-limited approval.
- Routing through a versioned map in a consensus store; clients refresh on redirects.
*Common weak answer:* "Add more nodes." A single hot key still lands on one node.

**Q5. How do you query across shards? Local versus global secondary indexes?**
A strong answer must mention:
- Local index: consistent writes on one shard; queries by the indexed value scatter to all shards.
- Global index: partitioned by value; one-partition lookups; asynchronous, possibly stale.
- Scatter–gather latency is set by the slowest shard: 0.99^20 ≈ 82% fast.
- Joins: co-location, reference tables, application joins, denormalisation, analytics elsewhere.
*Common weak answer:* "Query every shard and merge." It ignores tail latency and freshness trade-offs.

**Q6. Why are clocks unreliable? What are Lamport and vector clocks?**
A strong answer must mention:
- Skew, drift, NTP corrections and jumps, VM pauses; timestamps across machines are not comparable at small differences.
- Example: last-write-wins discarding a later write from a node with a slower clock.
- Monotonic clocks for durations and timeouts.
- Lamport: counters consistent with causality, total order with tie-breaks, no concurrency detection; vector clocks detect concurrency.
*Common weak answer:* "Use NTP." NTP reduces skew; it cannot eliminate it or prevent jumps.

**Q7. What is consensus, and why is an algorithm such as Raft needed? Why majorities?**
A strong answer must mention:
- Agreement on one value or a log despite crashes, delays and lost messages.
- Needed for leader election, partition maps, locks and configuration.
- Majorities overlap, so any decision is known to every later majority; 2f + 1 nodes tolerate f failures; the minority side cannot decide.
- FLP: no guaranteed termination in fully asynchronous systems; practical algorithms are always safe and live when timeouts work.
- The two generals problem: over a lossy channel, no finite exchange of messages makes both sides certain they agree.
*Common weak answer:* "Nodes vote and the majority wins." It misses why overlap prevents two leaders.

**Q8. How does Raft elect a leader and replicate its log?**
A strong answer must mention:
- Followers become candidates after a randomised election timeout; increment the term; request votes.
- One vote per node per term; the election restriction (candidate's log at least as up to date); majority wins; higher terms make stale leaders step down.
- The leader appends entries and sends them; an entry is committed once stored on a majority; followers with divergent logs are repaired.
- Every write costs a majority round trip.
*Common weak answer:* "The node with the lowest ID becomes leader." Raft elects by votes per term.

**Q9. How do you build a correct distributed lock? What is a fencing token?**
A strong answer must mention:
- Leases so crashed holders release; the lease can expire during a pause, so two holders can act.
- Fencing token: increasing per grant, sent with every operation; the resource rejects lower tokens.
- Redis `SET NX PX` locks: fine for efficiency; issues with pauses, async failover and clocks; Redlock is debated.
- For correctness: consensus-backed locks with tokens, or database constraints and locks.
*Common weak answer:* "Use Redis SETNX with a timeout." Safe only when an occasional double holder is acceptable.

**Q10. How does two-phase commit work, and why does it block? Why do microservices avoid it?**
A strong answer must mention:
- Prepare: participants make the transaction durable, keep locks, vote; commit: the coordinator logs and sends the decision.
- A participant that voted yes cannot decide alone; if the coordinator dies, it waits, holding locks.
- Microservices: blocking, held locks, coupling, many resources cannot prepare.
- Alternatives: keep transactions in one service; sagas with compensation and the outbox (Chapter 25); databases that run 2PC over consensus groups.
*Common weak answer:* "2PC is slow." The essential problem is blocking under coordinator failure.

**Q11. What are the main NoSQL data models? What is BASE?**
A strong answer must mention:
- Key–value, document, wide-column, graph, search, time-series: each with its data model, query model and typical engine.
- Each is efficient for one access shape and poor outside it.
- BASE: basically available, soft state, eventually consistent, a design attitude, not a precise guarantee; many stores offer stronger options.
*Common weak answer:* "NoSQL means no schema and eventual consistency." Both vary by product and configuration.

**Q12. How do you choose between SQL and NoSQL? What is polyglot persistence?**
A strong answer must mention:
- Decide from access patterns, integrity, consistency needs, scale, availability and operations, not slogans.
- Relational as the default system of record for transactional data with invariants and varied queries.
- Specialised stores for specific shapes (search, caching, time series, very high-volume key-based writes).
- Polyglot persistence: one source of truth, derived stores fed by CDC or outbox, rebuildable.
*Common weak answer:* "NoSQL scales, SQL doesn't." Relational and distributed SQL systems scale far; NoSQL trades generality for specific patterns.

**Q13. What is scalability? Vertical versus horizontal scaling, and where does sharding fit?**
A strong answer must mention:
- Scalability: handling more load by adding resources, with cost growing roughly in proportion.
- Vertical: a bigger machine; no software change, but a ceiling and a single point of failure.
- Horizontal: more machines sharing the work; no ceiling, but the work must be divisible.
- Stateless application instances scale out easily; data scales out by replication (reads) and sharding (data size and writes); scale up first while it suffices (Section 23.1).
*Common weak answer:* "Horizontal scaling is always better." It adds distribution problems (routing, cross-shard queries, coordination) that a larger machine avoids.

---

## Scenario Exercises

1. A SaaS product with 3,000 tenants stores everything in one PostgreSQL database that is reaching its write limits. One tenant generates 35% of all writes. Propose a sharding design: shard key, placement, routing, how to handle the big tenant, and which queries become harder.
2. A metrics store partitions by `(metric_name, timestamp_day)` with range partitioning; every day at midnight, write latency spikes and one node saturates. Explain and redesign the key.
3. Two services in different data centres update a user's profile; the store resolves conflicts by last-write-wins using each service's wall clock. Users report changes "reverting". Explain with a timeline and propose two fixes.
4. A team wants to run its own three-node consensus cluster, spread over two data centres (two nodes in one, one in the other). What happens when the link between the data centres fails, and when the larger data centre fails? Propose a better layout.
5. A nightly job takes a Redis lock with a 60-second TTL and writes the day's settlement file to object storage. Twice this year, two copies of the job ran concurrently and the file was corrupted. Explain and fix, naming what the storage must support.
6. An order service and a payment service each have their own database. A developer proposes XA two-phase commit across them so that "order placed" and "payment captured" are atomic. Explain the risks and propose an alternative.
7. A startup stores orders, users and products in a document database and now needs reports across them, uniqueness of emails and referential integrity between orders and products. Discuss whether to change the system of record, and how to migrate without downtime.

---

## Diagnosis Lab

**Lab 1: One shard at 100% CPU (hot key)**
- *Symptom:* In a 12-shard key–value cluster, shard 7 runs at 100% CPU and its p99 latency is 900 ms; the other shards average 25%. The problem started with a marketing campaign.
- *Hypotheses:* (a) uneven data distribution; (b) a hot key or tenant; (c) a hardware problem on shard 7's node.
- *Evidence:* Data size per shard is even (within 5%). Shard 7's request rate is 8× the others'. Its top-keys report shows `campaign:autumn:counter` receiving 60,000 increments per second and `campaign:autumn:config` 40,000 reads per second. The node's hardware metrics are normal, and moving shard 7's replica to a fresh node does not help.
- *Root cause:* (b): two hot keys concentrated on one partition by design (one counter, one config object).
- *Fix (now):* Cache `campaign:autumn:config` in process for a few seconds (Chapter 21, Section 21.9); salt the counter into 32 sub-keys spread across shards and sum them for display (Section 23.4). *Fix (lasting):* a review step for campaign features that identifies single keys with high request rates; per-key load metrics with alerts.
- *Prevention:* Load tests that model skewed traffic, not uniform keys.

**Lab 2: A cross-shard query timing out**
- *Symptom:* An admin search "orders by email" times out after 5 s for about 20% of requests since the cluster grew from 8 to 32 shards; each individual shard query takes about 40 ms on average.
- *Hypotheses:* (a) one slow shard; (b) fan-out tail latency; (c) the coordinator merging huge results.
- *Evidence:* Traces show every request queries all 32 shards (orders are sharded by `customer_id`; email is only in a local index). Per-shard p99 is 1.8 s, caused by occasional compactions and GC pauses on different nodes at different times; with 32 shards, 1 − 0.99^32 ≈ 28% of requests wait for at least one shard's p99, and those exceed the 5 s timeout when two slow shards coincide with retries. Results are small, so merging is cheap.
- *Root cause:* (b): scatter–gather over a growing number of shards turned each shard's rare slow responses into a common slow request.
- *Fix:* A global lookup table `email → customer_id` (sharded by email), maintained from the users table by CDC, so the search asks one lookup partition and then one order shard; hedged requests for remaining scatter queries; per-shard timeouts returning partial results with a warning for admin pages.
- *Prevention:* Track fan-out per endpoint; treat any frequent query without the shard key as a design issue.

**Lab 3: A lock holder paused by GC**
- *Symptom:* The monthly invoice-numbering job, which must assign consecutive invoice numbers exactly once, produced 214 duplicate invoice numbers this month. It runs on three instances and uses a distributed lock so that only one instance works at a time.
- *Hypotheses:* (a) the lock was not acquired correctly; (b) two instances held the lock at once; (c) a bug in number allocation.
- *Evidence:* The lock is a Redis key with a 15-second TTL, renewed every 5 seconds by a background thread. Instance A's GC log shows a 22-second stop-the-world pause at 02:14:03 (a full collection on a heap filled by the job's in-memory batch). Instance B's log shows it acquired the lock at 02:14:19, after A's key expired. A resumed at 02:14:25 and continued writing invoice numbers without re-checking the lock; both instances wrote numbers 10,412 to 10,625. The invoices table has no unique constraint on the invoice number.
- *Root cause:* (b): a lease expired during a GC pause, and nothing at the resource rejected the paused holder's writes (Section 23.8).
- *Fix (now):* Renumber the duplicates under a documented correction process. *Fix (lasting):* a unique constraint on `invoice_number`, so a second writer fails instead of duplicating; numbers allocated by the database (a sequence or an atomic counter row updated with `UPDATE … RETURNING`, Chapter 17, Section 17.11) instead of in the job's memory; if a lock is still needed, take it from etcd and pass its revision as a fencing token to a table that stores the highest token and rejects lower ones; reduce batch memory to avoid long GC pauses.
- *Prevention:* Treat "exactly once" requirements as database constraints, not lock discipline; record lock tokens in logs; alert on long GC pauses.

---

## Solution Sketches

1. Shard key: `tenant_id`, with directory-based placement (tenant → shard table in a consensus-backed store) so that tenants can be moved; most tenants hashed across a set of shared shards; the 35% tenant on its own dedicated shard (and, if it outgrows one node, sharded internally by its own customer or account key). Routing: a proxy or a library that looks up the tenant's shard (cached, versioned). Every table carries `tenant_id`; transactions stay within one tenant. Harder: cross-tenant reports (move to a warehouse fed by CDC), global uniqueness (per-tenant uniqueness instead, or a global lookup table), and moving a tenant (a copy, catch-up and cutover procedure).
2. Range partitioning by `(metric_name, day)` sends all writes for the current day of each metric to the partition for "today"; at midnight a new set of partitions becomes hot at once and lands on whichever node owns them. Redesign: hash the series identity (`metric_name + labels`) as the partition key so writes spread across nodes, with time as the sort key within each series; use time-based buckets only within the series (for retention and compaction), not as the leading partition component.
3. Timeline: service A (clock 200 ms ahead) writes "name = Ana B." at real time 10:00:00.000 with timestamp .200; service B writes "name = Ana Brown" at 10:00:00.100 with timestamp .100, after reading A's change; LWW keeps A's value, so B's later change reverts. Fixes: route each user's writes to one leader (conflict avoidance); or use version vectors (or a version column with conditional writes) so that a write based on version v replaces only v, and concurrent versions are detected and merged instead of decided by clocks.
4. Two-plus-one: if the link fails, the side with two nodes has a majority and continues; the single node stops (correct behaviour). If the larger data centre fails, the remaining single node has no majority, so the cluster stops entirely: the layout tolerates the loss of the smaller site only. Better: three sites (one node each, or 2–2–1 with five nodes), so that losing any one site leaves a majority; or, with only two sites, a small tie-breaker node in a third location.
5. The lock's TTL expired while the first job was paused (or slowed), the second job acquired it, and the first resumed and kept writing: two holders; possibly also a Redis failover losing the lock key. Fix: take the lock from a consensus store that returns a fencing token, and write the file with a conditional operation that the storage enforces (for example a conditional put that succeeds only if the object's version, or a token stored with it, is not newer), or write to a token-named object and commit by an atomic pointer update that rejects older tokens; also make the job idempotent (deterministic output per day), so a duplicate run produces the same file.
6. XA across two services couples their availability (either one's outage blocks the other's transactions), holds locks across network round trips, blocks on coordinator failure with in-doubt transactions (which also stall VACUUM on PostgreSQL), and requires XA support in both stacks and any broker involved. Alternative: a saga. The order service creates the order as `PENDING_PAYMENT` and records an outbox event in the same local transaction; the payment service captures the payment idempotently (keyed by order ID) and emits "payment captured" or "payment failed"; the order service then marks the order paid or cancels it (compensation), with timeouts and reconciliation (Chapter 25).
7. The new requirements (cross-entity reports, uniqueness, referential integrity) are exactly what a relational database enforces; keeping them in application code over a document store will keep producing integrity bugs. Migration: introduce PostgreSQL as the new system of record; backfill it from the document store; dual-read with comparison for a period; switch writes to PostgreSQL (with the document store updated from it by CDC while anything still reads it); then retire or keep the document store as a derived view for the access patterns it serves well. Add the unique and foreign-key constraints after cleaning existing duplicates and orphans.

---

**END OF CHAPTER 23**
