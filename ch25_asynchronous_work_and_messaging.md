# Chapter 25: Asynchronous Work and Messaging

## Where We Are

Most of this book has followed one synchronous request: the client waits while the server parses, authenticates, queries and answers. Several chapters have pointed past that model. Chapter 1 named the message queue as one of a backend's building blocks (Section 1.6). Chapter 6 showed that TCP delivers a byte stream, not messages, and that "delivered" and "processed" are different things (Section 6.9). Chapter 13 answered long operations with `202 Accepted` and sent webhooks with at-least-once delivery (Section 13.10), and made retries safe with idempotency keys (Section 13.5). Chapter 17 built a job queue in a database table with `SKIP LOCKED` (Section 17.8). Chapter 22 explained change data capture and why writing to a database and a second system from the same request is unreliable (Section 22.10). Chapter 23 showed why two-phase commit is avoided across services (Section 23.9). Chapter 24 gave retries their backoff, jitter and budgets.

This chapter is the home of **asynchronous work and messaging**. It explains when work should leave the request path, how background jobs and task queues work, the three messaging models (queues, publish/subscribe and partitioned logs) and the brokers that implement them, the mechanics of a Kafka-style log, what delivery guarantees really mean, and how to build consumers that are correct despite duplicates, reordering, poison messages and replays. It then solves the dual-write problem with the transactional outbox, coordinates multi-service business processes with sagas, introduces event-driven architecture, event sourcing and CQRS with their costs, and ends with a complete order workflow with failures injected at every step.

**After this chapter you can:**
- decide when work should be asynchronous, and state what it costs in latency, coupling, failure isolation and user-visible consistency;
- design background jobs with workers, scheduling, delays, retries and database-backed queues;
- distinguish point-to-point queues, publish/subscribe and partitioned logs, and explain how RabbitMQ-style and Kafka-style brokers store and deliver messages;
- explain topics, partitions, offsets, consumer groups, rebalancing, retention and compaction, and what ordering a log guarantees;
- trace at-most-once and at-least-once delivery through acknowledgement timelines, and explain why exactly-once is an end-to-end application property;
- build idempotent consumers with an inbox table and unique constraints;
- handle ordering, poison messages, retries with backoff, dead-letter queues and replay;
- explain the dual-write problem and implement the transactional outbox with a polling relay or CDC;
- design sagas with choreography or orchestration, compensations and a failure matrix;
- distinguish events from commands, evolve event schemas, and weigh event sourcing and CQRS;
- diagnose consumer lag, duplicate side effects, messages stuck in retry loops and events lost by dual writes.

**Interview questions this chapter prepares you for:**
1. When should work be asynchronous? What does it cost?
2. How does a background job queue work? A database-backed queue versus a broker?
3. Queue versus pub/sub versus log: what is the difference between RabbitMQ and Kafka?
4. How does Kafka store and deliver messages? What are partitions, offsets and consumer groups?
5. How does Kafka guarantee ordering, and how can retries break it?
6. What do at-most-once, at-least-once and exactly-once delivery mean?
7. How do you make a consumer idempotent?
8. What is a poison message? How do retries and dead-letter queues work?
9. What is the dual-write problem? What is the transactional outbox pattern?
10. Outbox polling relay versus CDC: which and why?
11. What is a saga? Choreography versus orchestration? What is a compensating action?
12. What is the difference between an event and a command? How do you evolve event schemas?
13. What are event sourcing and CQRS, and what do they cost?
14. What is consumer lag, and how do you diagnose and fix it?

**Scope.** Here: synchronous versus asynchronous design, background jobs and task queues, messaging models and broker internals at concept level, Kafka-style log mechanics, delivery semantics, idempotent consumers, ordering, poison messages, retries and dead-letter queues, the transactional outbox and CDC for messaging, sagas, event-driven architecture, event sourcing and CQRS, backpressure and consumer lag, a worked workflow, messaging diagnosis, and a placement-round section on JMS and messaging protocols (Section 25.14). Recalled, not re-taught: TCP's delivery limits (Chapter 6, Section 6.9), idempotency keys (Chapter 13, Section 13.5), webhooks and long-running operations (Chapter 13, Section 13.10), the `SKIP LOCKED` job queue (Chapter 17, Section 17.8), replication and leader election (Chapters 22–23), change data capture and dual writes (Chapter 22, Section 22.10), two-phase commit (Chapter 23, Section 23.9), retries, backoff, jitter and backpressure (Chapter 24; Chapter 11, Section 11.8). Elsewhere: running brokers in production, monitoring and autoscaling consumers are Chapter 26; the full request lifecycle including asynchronous paths is Chapter 27.

---

## 25.1 Synchronous versus Asynchronous Work

### Definitions

- **Synchronous** work is done while the caller waits: the caller sends a request and cannot proceed until the response arrives.
- **Asynchronous** work is handed off and done later, by someone else: the caller records that the work must happen and continues without waiting for the result.
- A **message** is a self-contained unit of data sent from a **producer** to one or more **consumers** through an intermediary, without the producer and consumer talking directly.
- A **message broker** is the intermediary: a server that receives, stores and delivers messages. A **queue** is a broker's ordered buffer of messages waiting to be consumed.
- **Temporal coupling** is the requirement that two components be available at the same time for an interaction to succeed. Synchronous calls have it; messaging removes it.

### Why it exists

Chapter 24 showed what synchronous chains cost: latency adds up along the chain, availability multiplies down it (Chapter 12, Section 12.11), and one slow dependency can hold every caller's threads. Many steps of a user action do not need to finish before the user gets an answer. When a customer places an order, the order must be recorded and the payment authorised before the page says "order placed", but the confirmation email, the warehouse notification, the loyalty points and the analytics event do not. Doing them synchronously makes the order slower and makes it fail when the email provider fails.

### Mechanism: what changes when work becomes asynchronous

```text
 SYNCHRONOUS                                          ASYNCHRONOUS
 client → orders → payments → email → warehouse       client → orders → payments
          (waits for each, in sequence)                        │ records "order placed" message
 latency = sum of all steps                                    ▼
 availability = product of all steps                  ◄── 201 Created (fast)
 email provider down → order fails                    broker ─┬─► email worker      (later)
                                                              ├─► warehouse worker  (later)
                                                              └─► analytics         (later)
                                                      email provider down → emails wait in the
                                                      queue; orders keep working
```

| Concern | Synchronous | Asynchronous |
|---|---|---|
| Latency seen by the user | sum of all steps | only the steps before the hand-off |
| Coupling | caller must know and reach every callee now | producer knows only the message; consumers can be added without changing it |
| Failure isolation | a slow or failed callee fails the caller | a failed consumer delays its own work; the producer is unaffected |
| Load | spikes hit every component at once | the queue absorbs spikes; consumers work at their own pace |
| Consistency seen by the user | the result is complete when the response arrives | the result arrives later: **eventual consistency** (Chapter 22, Section 22.8) |
| Error handling | the caller gets the error and can tell the user | errors happen after the user has gone; they must be retried, parked or compensated |
| Debugging | one call stack, one trace | causes and effects separated in time and process; needs correlation IDs |

### Concrete: deciding what goes async

For each step of a user action, ask three questions:

```text
 step                         must the user see its     can it fail without        verdict
                              result in the response?   failing the action?
 ─────────────────────────────────────────────────────────────────────────────────────────────
 validate cart, create order  yes                       no                          SYNC
 authorise payment            yes (accepted or not)     no                          SYNC (or async
                                                                                    with "pending")
 reserve inventory            usually                   depends on the business     SYNC or saga
 send confirmation email      no                        yes (retry later)           ASYNC
 notify warehouse             no                        yes                         ASYNC
 update recommendations       no                        yes                         ASYNC
 resize uploaded images       no (show a placeholder)   yes                         ASYNC job
 generate a 2 GB export       no (link by email)        yes                         ASYNC job + 202
```

The third question is hidden in the second: if a step happens later, **someone must make sure it eventually happens**, exactly the problem of the rest of this chapter.

### Guarantees and non-guarantees

Moving work to a broker guarantees that the producer is not slowed or failed by the consumer, and that work waits in the queue while the consumer is down (within the broker's retention). It does not guarantee when the work happens, that it happens only once, that related messages are processed in order, or that the user sees a consistent state in the meantime. And it does not remove failures; it moves them to where no user is waiting, which means they must be handled by retries, dead-letter queues and alerts instead of error pages.

### Failure and diagnosis

- **Users see stale or missing results.** "I placed the order but it is not in my list." Evidence: the order-list view is built by a consumer that lags. Fix: read-your-writes for the user's own data (show the order from the write model, or optimistically in the client), and keep lag low (Section 25.11).
- **Silent failures.** Emails not sent for days, with no error anywhere a human looks. Evidence: a growing dead-letter queue nobody monitors. Fix: alerts on dead-letter queues and consumer lag.
- **Async used to hide a slow dependency that is actually required.** The order is accepted, but payment fails later, and the business has no process for that. Fix: keep required steps synchronous, or design the pending and failed states explicitly (Section 25.9).

### Backend connection

Keep on the request path only what the user must see in the response and what must succeed for the action to count; move everything else behind a queue. Then design for what asynchrony brings: eventual consistency in the UI, durable hand-off, idempotent consumers, retries and dead-letter handling, monitoring of lag, and correlation IDs that tie a message back to the request that caused it.

---

## 25.2 Background Jobs and Task Queues

### Definitions

- A **background job** (task) is a unit of work executed outside a request: send an email, resize an image, generate a report, call a slow partner API.
- A **task queue** (job queue) stores jobs until a **worker**, a process that repeatedly takes jobs and executes them, is free. A task queue is a point-to-point queue (Section 25.3) specialised for jobs, usually with retries, delays and scheduling built in.
- A **delayed job** is a job that must not run before a given time. A **scheduled** (recurring, **cron**) job runs on a timetable ("every day at 02:00"). The name comes from `cron`, the Unix daemon that runs commands on schedules written as five time fields.
- A **lease** (visibility timeout) is the time a worker may hold a claimed job before it is considered abandoned and given to another worker.

### Why it exists

Any work that is slow, unreliable or not needed for the response should not run on a request thread: it holds a worker that could serve other requests (Chapter 11), it fails the user's request when it fails, and it dies with the request if the instance restarts. A task queue moves that work to dedicated workers, durably, with retries.

### Mechanism: a job's life

```text
 enqueue ──► PENDING ──(worker claims, lease starts)──► RUNNING ──(success)──► DONE
               ▲   ▲                                      │  │
               │   └──── lease expired (worker crashed) ──┘  │
               │                                             │ failure
               └──── retry: run_at = now + backoff ◄─────────┤ (attempts < max)
                                                             │
                                                             └─(attempts = max)──► DEAD (parked for a human)
```

Every transition follows from a failure the queue must survive. The worker can crash mid-job, so a claimed job carries a lease, and a job whose lease expires returns to `PENDING`. That means a job can run **more than once**, so job handlers must be idempotent (Section 25.6). Failures are retried with exponential backoff and jitter (Chapter 24, Section 24.3), expressed simply as a later `run_at`. Jobs that keep failing stop at a maximum and wait for a human in a dead state, rather than retrying forever (Section 25.7).

### Mechanism: a database-backed queue

Chapter 17 (Section 17.8) built the core: workers claim due jobs with `FOR UPDATE SKIP LOCKED` in a short transaction, process outside it, and a sweeper resets expired leases. Adding retries, delays and dead jobs is a few columns:

```sql
CREATE TABLE jobs (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  kind        text        NOT NULL,                 -- 'send_email', 'resize_image', …
  payload     jsonb       NOT NULL,
  status      text        NOT NULL DEFAULT 'PENDING',  -- PENDING, RUNNING, DONE, DEAD
  run_at      timestamptz NOT NULL DEFAULT now(),   -- delayed jobs and retries
  attempts    int         NOT NULL DEFAULT 0,
  max_attempts int        NOT NULL DEFAULT 8,
  locked_until timestamptz,                         -- the lease
  last_error  text,
  dedupe_key  text UNIQUE                           -- optional: refuse duplicate enqueues
);
CREATE INDEX jobs_due ON jobs (run_at) WHERE status = 'PENDING';

-- claim (one short transaction)
UPDATE jobs SET status = 'RUNNING', attempts = attempts + 1,
                locked_until = now() + interval '5 minutes'
WHERE id IN (SELECT id FROM jobs
             WHERE status = 'PENDING' AND run_at <= now()
             ORDER BY run_at
             FOR UPDATE SKIP LOCKED
             LIMIT 10)
RETURNING id, kind, payload, attempts;

-- on failure: schedule a retry with exponential backoff and jitter, or park it
UPDATE jobs
SET status   = CASE WHEN attempts >= max_attempts THEN 'DEAD' ELSE 'PENDING' END,
    run_at   = now() + least(interval '1 hour',
                             interval '10 seconds' * power(2, attempts)) * random(),
    last_error = $2, locked_until = NULL
WHERE id = $1;
```

Polling every second is fine at modest volume. PostgreSQL's `LISTEN`/`NOTIFY` can wake workers when a job is inserted, so they need not poll often. The great advantage of a database queue is that **enqueuing a job can be part of the business transaction**: the order row and the "send confirmation email" job commit together or not at all, which removes the dual-write problem for jobs (Section 25.8). Its limits are throughput (every claim and completion is a write; tens of thousands of jobs per second strain a general-purpose database) and the bloat of a table with constant updates and deletes (Chapter 17, Section 17.6).

Dedicated task-queue libraries (Sidekiq and BullMQ on Redis, Celery on RabbitMQ or Redis, and many database-backed libraries) provide the same lifecycle with workers, retries, scheduling and dashboards. Redis-based queues are fast but inherit Redis's durability settings (Chapter 21, Section 21.7): a job enqueued just before a failover can be lost unless persistence and replication are configured for it.

### Mechanism: scheduled jobs across many instances

A recurring job defined in application code runs once **per instance**: with 12 instances, the 02:00 billing job runs 12 times. Three fixes, in increasing robustness (Chapter 11, Section 11.7 introduced the problem):

```text
 1. a separate scheduler process (one replica) that only ENQUEUES jobs; workers execute them
 2. a claim per period: INSERT INTO job_runs (job, period) VALUES ('billing', '2026-10-24')
                        ON CONFLICT DO NOTHING RETURNING …   → only the instance that inserted runs it
 3. leader election through a consensus store (Chapter 23, Section 23.7): only the leader schedules
 and in all cases: the job itself is idempotent for its period, because claims and leaders can
 still fail over in the middle of a run
```

Scheduled jobs also need answers to "what if a run is missed?" (the scheduler was down at 02:00: run it late, or skip it?) and "what if a run is still going when the next one is due?" (overlap, skip, or queue?). Write them down per job.

### Guarantees and non-guarantees

A durable task queue with leases guarantees that an enqueued job is executed **at least once** while its retries last, survives worker crashes, and is parked for inspection after repeated failure. It does not guarantee exactly-once execution, execution at an exact time (only "not before"), ordering between jobs, or throughput beyond what its storage can sustain.

### Failure and diagnosis

- **Jobs run twice.** A lease shorter than the job's duration lets another worker claim it while the first is still running. Evidence: two "started" logs for one job ID, minutes apart, without a crash. Fix: leases longer than the worst-case run, lease renewal (heartbeats) for long jobs, idempotent handlers.
- **Jobs stuck in `RUNNING`.** No sweeper, or a sweeper that stopped. Evidence: old `locked_until` values. Fix: the sweeper; alert on age of the oldest running job.
- **Queue table bloat.** Evidence: the jobs table and its indexes growing despite few pending jobs; dead tuples (Chapter 17, Section 17.6). Fix: delete or archive done jobs in batches; partition by time (Chapter 16, Section 16.14).
- **A recurring job running on every instance.** Evidence: duplicate runs at the same minute from different hosts. Fix: one scheduler, period claims, leader election.

### Backend connection

Move slow and failure-prone work into jobs, enqueue them in the same database transaction as the business change when you can, and treat every job as something that may run twice or late. Give each job kind a lease, a retry policy with backoff and jitter, a maximum, and a dead state that someone monitors. A database-backed queue is often the simplest correct choice until the job rate demands a broker.

---

## 25.3 Messaging Models and Broker Internals

### Definitions

Three models describe who receives a message and what happens to it afterwards:

- **Point-to-point queue**: each message is delivered to **one** of the consumers reading the queue (**competing consumers**), and is removed once that consumer acknowledges it. It distributes work.
- **Publish/subscribe** (pub/sub): a message published to a **topic** is delivered to **every** subscriber. It distributes information.
- **Partitioned log**: messages are appended to an ordered, durable log split into **partitions**; consumers **read** at their own position (offset) and the log keeps messages for a retention period whether or not they have been read. Many independent consumer groups can each read everything, and each group splits the partitions among its members.

An **acknowledgement** (ack) is a consumer's signal to the broker that it has finished with a message. A **negative acknowledgement** (nack) says it failed and should be redelivered or dead-lettered.

### Why it matters

The three models answer different questions. "Who does this job?" is a queue. "Who needs to know this happened?" is pub/sub. "What happened, in order, and can I read it again?" is a log. Brokers are built around one model and emulate the others, and their storage designs differ accordingly. Choosing a broker without knowing which model a workload needs produces systems that fight their tools: work queues built on logs that cannot skip a slow message, or event histories built on queues that delete what they deliver.

### Concrete: the three models

```text
 POINT-TO-POINT QUEUE (work distribution)
 producer ─► [ m5 m4 m3 m2 m1 ] ─┬─► worker A gets m1   (each message to ONE worker;
                                 ├─► worker B gets m2    acked messages are deleted)
                                 └─► worker C gets m3

 PUBLISH/SUBSCRIBE (fan-out)
 producer ─► topic "order.placed" ─┬─► email service       (EVERY subscriber gets a copy)
                                   ├─► warehouse service
                                   └─► analytics service

 PARTITIONED LOG (durable, replayable, ordered per partition)
 topic "orders", 3 partitions; each message has an OFFSET in its partition
 partition 0: [0][1][2][3][4][5][6]        group "email":     A reads p0 at 5, p1 at 3
 partition 1: [0][1][2][3][4]              (members split     B reads p2 at 7
 partition 2: [0][1][2][3][4][5][6][7][8]   the partitions)
                                           group "analytics": C reads p0, p1, p2 at offsets 2, 1, 4
                                           (independent position; messages stay until retention ends)
```

### Mechanism: a queue broker (RabbitMQ-style)

A RabbitMQ-style broker routes and holds messages until they are consumed:

```text
 producer ──publish(routing key "order.placed.eu")──► EXCHANGE ──bindings──► QUEUES ──► consumers
   exchange types:  direct  (route to queues bound with exactly this key)
                    fanout  (route to every bound queue: pub/sub)
                    topic   (route by pattern: "order.*.eu", "order.#")
                    headers (route by message header values)
 each queue: messages stored (in memory, on disk, or replicated); delivered to one consumer at a time
 consumer: receives up to PREFETCH unacked messages; acks each → broker DELETES it;
           nacks or disconnects → the message is REQUEUED (redelivered, flagged "redelivered")
 queue → dead-letter exchange: messages rejected, expired, or over a length limit go there
```

Pub/sub in this model is one queue **per subscriber**, each bound to the same exchange: every subscriber gets its own copy. The broker tracks per-message state (delivered, unacked) and deletes messages once acked, so it is excellent for work distribution, per-message routing, priorities and per-message retry, and poor at replay: once acked, a message is gone. Replicated **quorum queues** keep queue contents on several nodes using Raft (Chapter 23, Section 23.7). Amazon SQS follows the same queue model with a **visibility timeout** (a received message is hidden from other consumers for a period and reappears if not deleted) instead of connection-based acknowledgement.

### Mechanism: a log broker (Kafka-style)

A Kafka-style broker stores each partition as an append-only file (a sequence of segment files) on disk, replicated to other brokers, and serves reads by offset:

```text
 producer ─► partition leader appends to its log ─► followers replicate ─► acknowledged
 consumers fetch "partition 2 from offset 7, up to 1 MB" → sequential disk reads, often from
 the OS page cache (Chapter 4, Section 4.3), sent with zero-copy where possible (Chapter 4)
 the broker does NOT track per-message delivery: each consumer group records ONE number per
 partition, its committed offset ("everything before 7 is done")
 messages are deleted by RETENTION (time or size), not by consumption
```

That design trades per-message features for throughput, ordering and replay: appends and sequential reads are cheap, one offset per partition is a tiny amount of state, and any consumer can rewind. What it gives up is per-message acknowledgement: a consumer cannot skip one slow or failing message and move past it without moving its offset past it (Section 25.7).

### Concrete: choosing between them

| Need | Queue broker (RabbitMQ, SQS) | Log broker (Kafka, Pulsar, Kinesis; Redis Streams at smaller scale) |
|---|---|---|
| Distribute jobs to workers | natural: competing consumers, per-message acks | possible: one partition per consumer at a time limits parallelism to the partition count |
| Fan-out to many independent consumers | one queue per subscriber | natural: each group reads the same log |
| Replay history, add a new consumer later | not possible after ack | natural, within retention |
| Ordering | per queue with one consumer; lost with competing consumers | per partition, with keys choosing partitions |
| Per-message retry, delay, priority, TTL | built in | built by the application (retry topics) |
| Throughput | high | very high (sequential I/O, batching) |
| Consumer state | broker tracks each message | consumer group tracks one offset per partition |
| Typical use | task queues, RPC-style work, per-message routing | event streams, CDC, activity logs, stream processing |

### Guarantees and non-guarantees

A queue broker guarantees that an acked message is not redelivered and that an unacked message is redelivered after a consumer failure. A log broker guarantees ordered, durable storage per partition for the retention period, and lets consumers resume from their committed offsets. Neither guarantees exactly-once processing, ordering across partitions or across competing consumers, or that messages survive a broker failure unless replication and acknowledgement settings are chosen for it.

### Failure and diagnosis

- **Messages lost on broker restart.** Non-persistent messages or non-durable queues (RabbitMQ), or producer acknowledgements that do not wait for replication (Kafka `acks=1` and a leader failure, Section 25.4). Evidence: gaps in sequence numbers produced versus consumed. Fix: durable queues, persistent messages, `acks=all` with a minimum in-sync replica count.
- **Unacked messages piling up.** A consumer holds its prefetch and stalls. Evidence: the broker's unacked count for a consumer stays at its prefetch limit. Fix: timeouts in handlers, smaller prefetch, more consumers.
- **Replay impossible.** A team needs to reprocess last week's events, but the queue deleted them on ack. Fix: a log for event history, or archiving messages to storage.

### Backend connection

Use a queue broker (or a database queue) for work that one worker should do, with per-message retries and dead-lettering; use a log for events that several consumers need, that must stay in order per entity, or that you may need to replay. Many systems run both. Configure durability explicitly in either: persistent messages and replicated storage, and producers that wait for the broker's confirmation before treating a message as sent.

---

## 25.4 Kafka-Style Log Mechanics

### Definitions

- A **topic** is a named stream of messages (records), for example `orders`. Each **record** has an optional **key**, a **value** (the payload), headers and a timestamp.
- A **partition** is one ordered, append-only log within a topic; a topic has a fixed number of partitions spread across brokers. Each partition has a **leader** replica that handles writes and reads, and **follower** replicas that copy it; the followers that are fully caught up form the **in-sync replica set** (ISR).
- An **offset** is a record's position in its partition: 0, 1, 2, … Offsets are per partition, not per topic.
- A **consumer group** is a set of consumers that share the work of reading a topic: each partition is assigned to exactly one consumer in the group at a time. Each group stores a **committed offset** per partition, the position from which it will resume.
- A **rebalance** is the reassignment of partitions among a group's consumers when one joins, leaves or is considered dead.
- **Retention** is how long the log keeps records (by time or size). **Log compaction** is an alternative retention policy that keeps at least the latest record for each key and discards older records with the same key.

### Why it matters

These five ideas determine every property of a log-based system that a backend engineer must reason about: how messages are ordered, how much parallelism consumers can have, what happens when a consumer crashes or is slow, how long a consumer can be down without losing data, and how a new service can bootstrap its state.

### Mechanism: producing, partitioning and ordering

```text
 producer.send(topic "orders", key "customer-81", value {…})
   partition = hash(key) mod number_of_partitions     (keyless records are spread over partitions)
   → all records with key "customer-81" go to the SAME partition, in the order sent

 partition 4 (leader on broker 2, followers on brokers 1 and 3)
   offset: 120        121        122        123
           c81:placed c17:placed c81:paid   c81:shipped
   ORDER IS GUARANTEED WITHIN A PARTITION: every consumer of partition 4 sees 120, 121, 122, 123
   no order between partitions: c81's records in partition 4 and c44's in partition 7 interleave
   arbitrarily in what any consumer observes
```

The rule to remember: **ordering is per partition, hence per key**. If events about one order (or one customer) must be processed in order, they must share a key. Two consequences follow. Changing the number of partitions changes `hash(key) mod N` for most keys, so records for one key can be in two partitions around the change, and ordering for that key is broken across the boundary; partition counts are chosen generously up front for this reason (the same mod-N problem as Chapter 23, Section 23.3). And a hot key overloads one partition and one consumer (Chapter 23, Section 23.4).

**Producer acknowledgements** decide durability (the same commit-acknowledgement question as Chapter 22, Section 22.3):

| Producer `acks` | The broker acknowledges when | Survives |
|---|---|---|
| `0` | never; the producer does not wait | nothing guaranteed |
| `1` | the leader has appended the record | not a leader failure before followers copy it |
| `all` (with `min.insync.replicas = 2`) | all in-sync replicas have it, and at least 2 replicas are in sync | the loss of any one broker |

**Retries and ordering.** A producer that retries a failed send can reorder or duplicate records: batch 1 fails, batch 2 succeeds, batch 1 is retried and lands after batch 2; or batch 1 actually succeeded but its acknowledgement was lost, and the retry writes it twice. Kafka's **idempotent producer** fixes both within a partition: the broker assigns each producer an ID, each batch carries a sequence number, and the broker rejects duplicates and out-of-order sequences. It is enabled by default in recent Kafka client versions, at the time of writing; check your client. It does not deduplicate across producer restarts with a new ID, or across application-level retries that call `send` again with a new record.

### Concrete: consumer groups, offsets and a rebalance

```text
 topic "orders", 6 partitions; consumer group "email-sender"

 BEFORE: 2 consumers
   consumer A: p0 p1 p2         committed offsets: p0=500 p1=322 p2=870
   consumer B: p3 p4 p5                            p3=411 p4=123 p5=699

 consumer C starts and joins the group  →  REBALANCE
   1. the group coordinator (a broker) notices the join
   2. partitions are revoked from members (all of them in the classic "eager" protocol;
      only the ones that move in the newer "cooperative" protocol) and members commit their offsets
   3. a new assignment is computed and distributed
 AFTER: 3 consumers
   consumer A: p0 p1            C starts p2 at its COMMITTED offset 870, and p5 at 699
   consumer B: p3 p4
   consumer C: p2 p5

 if A had processed p2 up to 880 but committed only 870 before giving it up,
 C re-processes 870–879: DUPLICATES (Section 25.5)
 a 7th consumer in a 6-partition group would get NO partitions: parallelism ≤ partition count
```

A consumer is considered dead, triggering a rebalance, when it stops sending heartbeats or when it does not call `poll` again within a maximum interval (`max.poll.interval.ms`). The second trigger catches a common mistake: a consumer that processes a large batch too slowly is ejected, its partitions move, it finishes its batch and fails to commit, and the new owner reprocesses the batch, possibly also too slowly: a **rebalance storm**.

Offsets are committed to an internal compacted topic. A consumer controls **when** it commits: automatically on a timer (simple, and the source of both duplicates and losses, Section 25.5), or manually after processing.

### Mechanism: retention and compaction

```text
 TIME/SIZE RETENTION (e.g. 7 days): segments older than 7 days are deleted whole
   a consumer group offline for 8 days finds its committed offset gone: it must start
   from the earliest available offset (data lost to it) or the latest (data skipped)

 LOG COMPACTION (key → latest value), e.g. topic "customer-addresses"
   before:  [c1:A] [c2:X] [c1:B] [c3:P] [c2:Y] [c1:C]
   after:   [c3:P] [c2:Y] [c1:C]           (older values per key removed; order of survivors kept)
   a record with key c2 and value NULL (a TOMBSTONE) deletes c2 after a grace period
   use: a new service reads the compacted topic from the start and gets the latest state of every
        key: the topic acts as a replicated table (CDC topics, configuration, state stores)
```

### Guarantees and non-guarantees

A Kafka-style log guarantees order within each partition, durability according to the producer's acknowledgement setting and replication, that each partition is consumed by one member of a group at a time, and that consumers resume from committed offsets. It does not guarantee order across partitions, exactly-once processing by itself (Section 25.5), that a committed offset matches what was actually processed, that data outlives retention, or stable ordering for a key when the partition count changes.

### Failure and diagnosis

- **Out-of-order processing for one entity.** Events for one order processed out of sequence. Evidence: the events were produced with different keys (or no key), or partitions were added. Fix: key by the entity whose order matters; avoid changing partition counts on ordered topics.
- **One consumer overloaded, others idle.** Evidence: lag concentrated on one partition. Cause: a hot key (Chapter 23, Section 23.4). Fix: a better key, or splitting the hot entity's work.
- **Rebalance storms.** Evidence: frequent rebalances in consumer logs; throughput collapsing; duplicates. Causes: processing time per poll exceeding the maximum poll interval; consumers restarting in a loop; deploys restarting all consumers at once. Fix: smaller batches, faster or asynchronous processing, cooperative rebalancing, static group membership for planned restarts.
- **Data loss after a broker failure.** Evidence: offsets acknowledged to producers missing after a leader change. Cause: `acks=1`, or unclean leader election allowing an out-of-sync replica to lead. Fix: `acks=all`, `min.insync.replicas`, unclean election disabled.

> **Framework lens:** In Spring for Apache Kafka, `@KafkaListener` methods run in listener containers that manage polling, commits (by default after the records returned by a poll are processed, in the container's default acknowledgement mode at the time of writing) and error handlers with retry and dead-letter publishing. The Java client's `KafkaConsumer` is not thread-safe: one consumer per thread, with parallelism coming from partitions.

### Backend connection

Choose keys by the entity whose events must stay ordered, choose partition counts for future parallelism and never change them casually on ordered topics, produce with `acks=all` and the idempotent producer, and commit offsets only after processing. Keep per-poll processing well within the poll interval, and size retention for the longest consumer outage you must survive. Use compacted topics when consumers need the latest state per key rather than the full history.

---

## 25.5 Delivery Semantics: At-Most-Once, At-Least-Once, and "Exactly-Once"

### Definitions

- **Delivery semantics** describe how many times a message may be delivered to, and processed by, a consumer when failures happen.
- **At-most-once**: each message is processed zero or one times. Failures can lose messages, never duplicate them.
- **At-least-once**: each message is processed one or more times. Failures can duplicate messages, never lose them (while retries and retention last).
- **Exactly-once**: each message's **effect** happens exactly one time. As this section shows, no broker or network can guarantee this on its own; it is a property of the whole system, achieved by combining at-least-once delivery with idempotent or transactional processing.

### Why it matters

Chapter 6 (Section 6.9) showed that a TCP acknowledgement means "the bytes reached the other kernel", not "the application processed them". The same gap exists between a broker and a consumer: the broker cannot know whether a consumer that crashed had finished, half-finished or not started the work. Every messaging system resolves that uncertainty one way or the other, and the choice decides whether your system can lose emails or send them twice.

### Concrete: acknowledge before or after processing, with crashes

The only difference between the two basic semantics is the order of two steps: processing the message, and telling the broker you are done with it (an ack, or an offset commit).

```text
 ACK BEFORE PROCESSING (at-most-once)
 consumer                         broker
 receive m42 ◄──────────────────
 ack m42 ───────────────────────► m42 marked done (deleted / offset committed past it)
 process m42 (send the email)
   crash here ✕ → the email is NEVER sent; the broker will not redeliver m42      → LOST

 ACK AFTER PROCESSING (at-least-once)
 receive m42 ◄──────────────────
 process m42 (send the email)
   crash here ✕ → m42 is still unacked → redelivered → processed again
                  (if the email had already gone out: a DUPLICATE email)           → DUPLICATED
 ack m42 ───────────────────────► done
   crash before the ack arrives ✕ → same: redelivered, processed twice           → DUPLICATED
```

There is no third ordering that avoids both: the crash can always happen between the side effect and the acknowledgement. Almost every production system chooses **at-least-once**, because a lost message is usually invisible and unrecoverable, while a duplicate can be detected and made harmless (Section 25.6). At-most-once is acceptable only for data whose loss does not matter: metrics samples, best-effort notifications.

The same reasoning applies on the producer side. A producer that sends a message and does not receive the broker's confirmation does not know whether the broker stored it (Chapter 7, Section 7.15's unknown outcome). If it retries, it may publish a duplicate; if it does not, it may lose the message. Producers that retry give at-least-once publishing.

### Mechanism: what "exactly-once" can and cannot mean

```text
 "EXACTLY-ONCE" CLAIM                       WHAT IT ACTUALLY GUARANTEES
 ─────────────────────────────────────────────────────────────────────────────────────────────
 idempotent producer (Kafka)                no duplicate records in a partition from producer
                                            retries within one producer session
 transactions within the broker             a consumer's offsets and the records it produced to
 (Kafka read-process-write)                 other topics are committed atomically; consumers
                                            reading with read_committed see either all or none
 deduplication window (SQS FIFO)            the broker drops a message with the same
                                            deduplication ID within a time window (5 minutes)
 NONE of them covers                        side effects outside the broker: the email sent,
                                            the card charged, the row written to your database,
                                            the HTTP call to a partner
```

Kafka's transactional processing is genuinely useful for pipelines whose input and output are both Kafka topics (stream processing): reading a record, producing a derived record and committing the input offset happen atomically. But the moment a consumer charges a card or sends an email, the effect is outside that transaction, and a crash after the effect and before the commit causes a redelivery and a second effect.

So, stated precisely: **brokers provide at-most-once or at-least-once delivery, plus features that help; exactly-once is an end-to-end property of the application**, obtained by one of two methods:

1. **Idempotent processing**: at-least-once delivery, with a consumer that recognises a repeat and does not apply its effect twice (Section 25.6).
2. **Transactional processing**: the consumer stores its effect and its progress (the offset or the processed message ID) in the **same** transaction, in the same database, so that either both happen or neither does. On restart, it resumes from the stored progress.

```sql
-- transactional processing: effect and progress commit together
BEGIN;
UPDATE accounts SET balance = balance + 500 WHERE id = 81;           -- the effect
UPDATE consumer_offsets SET next_offset = 1043                         -- the progress
WHERE consumer = 'ledger' AND topic = 'payments' AND part = 4;
COMMIT;
-- on restart: read next_offset from the database and seek the consumer to it,
-- ignoring the broker's committed offset
```

### Guarantees and non-guarantees

Acknowledging after processing guarantees that no message is lost while the broker retains it and redelivers it. It guarantees nothing about duplicates. Broker features (idempotent producers, transactions, deduplication windows) guarantee no duplicates within their scope, which never includes your external side effects. No configuration of a broker alone delivers exactly-once business effects.

### Failure and diagnosis

- **Lost messages with auto-commit.** The consumer's offsets are committed on a timer while records are still being processed. If it crashes after an automatic commit but before finishing those records, they are skipped on restart: committed, but never processed. Evidence: gaps between produced and processed message IDs after a crash. Fix: commit after processing.
- **Duplicate side effects after restarts and rebalances.** Evidence: two emails, two charges, two rows with the same message ID, around consumer restarts. Fix: idempotent consumers (Section 25.6).
- **Belief that the broker is "exactly-once".** Evidence: a design document that relies on a broker setting to prevent double charges. Fix: idempotency at the effect.

### Backend connection

Assume at-least-once delivery everywhere: from producers that retry, brokers that redeliver, and consumers that restart. Acknowledge or commit only after processing, and make every consumer's effect idempotent or transactional with its progress. Use broker transactions for broker-to-broker pipelines, and never as a substitute for idempotency at external side effects.

---

## 25.6 Idempotent Consumers and Deduplication

### Definitions

- An **idempotent consumer** is a consumer for which processing the same message twice has the same effect as processing it once.
- A **message ID** is a unique identifier assigned by the producer to each message (for example a UUID), carried with the message and preserved across retries and redeliveries.
- The **inbox pattern** (processed-messages table) records the IDs of messages a consumer has processed, in the same database transaction as the message's effect, so that a redelivered message is recognised and skipped.

### Why it exists

At-least-once delivery (Section 25.5) makes duplicates normal: every consumer restart, rebalance, lease expiry and producer retry can produce one. The consumer is the only place where duplicates can be neutralised, because only the consumer knows what its effect is and where it is stored. Idempotency is to messaging what idempotency keys are to HTTP APIs (Chapter 13, Section 13.5).

### Mechanism: three ways to make an effect idempotent

1. **Naturally idempotent effects.** Some operations are idempotent by their nature: "set the order status to SHIPPED", "upsert the customer's address", "delete the session". Processing them twice leaves the same state. Prefer designing messages so that their effects are of this kind: absolute values ("balance is now 120", "status is SHIPPED") rather than relative changes ("add 20", "advance status").
2. **A business unique constraint.** The effect itself has a natural unique key: one invoice per order, one shipment per order line, one ledger entry per payment ID. A unique index (Chapter 14, Section 14.3) makes the second insert fail, and the consumer treats that failure as "already done".
3. **The inbox (processed-messages) table**, when neither of the above applies.

### Concrete: an idempotent consumer with an inbox table

```sql
CREATE TABLE processed_messages (
  consumer     text        NOT NULL,          -- which consumer (different consumers process the
  message_id   uuid        NOT NULL,          --   same message independently)
  processed_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer, message_id)
);
```

```pseudo
function handle(message):                       // message.id set by the producer
    db.transaction:
        inserted = db.execute(
            "INSERT INTO processed_messages (consumer, message_id)
             VALUES ('loyalty-points', $1)
             ON CONFLICT DO NOTHING", message.id)        // affected rows: 1 = new, 0 = duplicate
        if inserted == 0:
            return                                       // already processed: skip the effect
        db.execute("UPDATE loyalty SET points = points + $1 WHERE customer_id = $2",
                   message.points, message.customer_id) // the effect, in the SAME transaction
    broker.ack(message)                                  // after commit
```

The timeline with a crash shows why the effect and the record must share one transaction:

```text
 delivery 1:  BEGIN; insert m42 → 1 row; points += 50; COMMIT;   crash before ack ✕
 delivery 2:  BEGIN; insert m42 → 0 rows (already there) → skip; COMMIT; ack ✔
 result: points added once

 WRONG (two transactions): insert m42 COMMIT; crash ✕; points never added; redelivery sees m42
 → skips → the effect is LOST. Or: add points COMMIT; crash ✕ before recording m42 → redelivery
 adds the points AGAIN.
```

When the two concurrent deliveries arrive at the same time (possible during a rebalance), the primary key makes one of the inserts wait for the other's transaction and then report a conflict (Chapter 17, Section 17.11): only one applies the effect.

### Mechanism: when the effect is outside your database

An email, an SMS, a card charge or a partner API call cannot share a transaction with the inbox table. Options, from best to weakest:

- **Pass an idempotency key downstream.** Use the message ID (or a key derived from the business operation) as the idempotency key on the call to the payment provider or email service, if it supports one. The downstream deduplicates.
- **Record intent, then act, then record completion**, and on redelivery check the state: `PENDING` (unknown: query the downstream if it offers a status lookup, otherwise decide whether a possible duplicate or a possible omission is worse), `SENT` (skip).
- **Accept rare duplicates** where they are harmless and cheap (a duplicate push notification), and deduplicate on the receiving side where possible (email clients collapse identical messages only sometimes; do not rely on it).

### Mechanism: deduplication windows and their limits

An inbox table grows with every message. It can be pruned after a period longer than any plausible redelivery: the broker's retention, the longest consumer outage, the producer's retry window. Some brokers offer deduplication themselves (SQS FIFO queues drop a message whose deduplication ID was seen in the last five minutes; Kafka's idempotent producer drops retried batches); these windows cover broker-level retries only, not a producer that restarts and republishes an hour later, or an operator who replays a day of messages (Section 25.7). A replay is a deliberate mass duplicate: consumers that are idempotent by business keys survive it; consumers that rely on a short window do not.

### Guarantees and non-guarantees

An inbox table updated in the same transaction as the effect guarantees that each message ID's effect is applied at most once, and, combined with at-least-once delivery, exactly once, for effects inside that database. Business unique constraints guarantee it for effects with natural keys. Nothing guarantees exactly-once external side effects unless the external system honours an idempotency key. Deduplication is only as good as the message ID: a producer that generates a new ID on each retry defeats it.

### Failure and diagnosis

- **Duplicate emails.** Evidence: the email provider's log shows two sends with the same order and template seconds or minutes apart; the consumer's logs show the same message ID processed twice around a rebalance or restart. Diagnosis Lab 2. Fix: idempotency key to the provider, or a sent-state record checked before sending.
- **Duplicates despite an inbox.** Evidence: the inbox insert and the effect are in separate transactions, or the producer assigns a new ID per retry. Fix: one transaction; IDs assigned once when the business event happens (for example in the outbox row, Section 25.8).
- **Lost effects despite an inbox.** The ID is recorded before the effect in a separate commit. Fix: same transaction.
- **Inbox table growth.** Evidence: a table of billions of rows. Fix: prune by age beyond the redelivery horizon; partition by time.

### Backend connection

Give every message a stable ID at the moment the business event happens. Prefer naturally idempotent messages (absolute values) and business unique constraints; otherwise record processed IDs in an inbox table in the same transaction as the effect. Carry the ID as an idempotency key to every external system that accepts one. Then replays, rebalances and retries become safe by construction.

---

## 25.7 Ordering, Poison Messages, Retries, Dead-Letter Queues and Replay

### Definitions

- A **poison message** is a message that fails every time it is processed (malformed data, a bug triggered by its content, a reference to an entity that does not exist), so retrying it can never succeed.
- A **dead-letter queue** (DLQ, or dead-letter topic) is where messages are moved after repeated failure, so that they stop blocking or consuming the main flow and can be inspected, fixed and **redriven** (sent back for processing).
- A **retry topic** (or delay queue) holds failed messages until a later time, implementing backoff for messages.
- **Replay** is reprocessing messages from the past: rewinding a consumer group's offsets in a log, or redriving a DLQ.

### Why it matters

At-least-once delivery means a failed message is redelivered. If the failure is transient, that is exactly right. If it is permanent, a naive consumer retries forever: in a queue, the poison message cycles and consumes capacity; in a log, it **blocks its partition**, because the consumer cannot commit past it, and every message behind it waits. A single bad record can stop a partition for days if nobody notices. Retries can also break the ordering that Section 25.4 worked so hard to provide.

### Mechanism: ordering, precisely

| Setup | Ordering guaranteed |
|---|---|
| Log, one partition, one consumer per partition, records keyed by entity | per key, in processing order, if the consumer processes sequentially |
| Log consumer processing a partition's records **in parallel** (a thread pool per partition) | none, unless work is grouped by key inside the consumer |
| Queue with one consumer and prefetch 1 | queue order |
| Queue with **competing consumers** | none: m1 to A, m2 to B; B can finish first |
| Any setup with **retries that move a message aside** (retry topic, requeue to the back) | broken for that key: later messages overtake the retried one |
| Producer retries without an idempotent producer | possibly reordered within a partition (Section 25.4) |

The retry case is the subtle one. If `order-81: PAID` fails and goes to a retry topic, `order-81: SHIPPED` behind it is processed first, and the consumer's view of order 81 goes `PLACED → SHIPPED → PAID`. Three ways to cope:

- **Block the key, not the partition**: when a message for key K fails, park it **and all later messages for K** until it succeeds, while other keys proceed. More complex, but preserves per-key order.
- **Make the consumer order-tolerant**: carry a version or sequence number per entity in each message, and ignore (or buffer) messages older than the state already applied: `UPDATE orders SET status = $s, version = $v WHERE id = $id AND version < $v`. Out-of-order arrivals then cannot move state backwards.
- **Accept unordered processing** where messages are independent (each email is separate).

### Mechanism: a retry pipeline with a dead-letter queue

```text
 main topic "orders" ──► consumer
                           ├ success ────────────────────────────────► commit offset, next
                           ├ transient failure (timeout, 503):
                           │    retry in place a few times with short backoff (ms to seconds),
                           │    then publish to "orders.retry.1m" and commit
                           ├ permanent failure (validation error, unknown entity, bug):
                           │    publish straight to "orders.dlq" with the error and attempt
                           │    count in headers, commit
 "orders.retry.1m" ──► retry consumer: waits until (timestamp + 1 min), processes;
                         on failure → "orders.retry.10m" → … → "orders.dlq"
 "orders.dlq" ──► nobody processes automatically; ALERT; humans inspect, fix the bug or data,
                  then REDRIVE: republish to "orders" (consumers must be idempotent)
```

Two classifications drive the pipeline: **transient versus permanent** (retrying a validation error is pointless; giving up on a timeout after one attempt is premature), and **how long to keep trying** (the total retry horizon should cover the outages you expect from the dependency, for example a partner API's typical maintenance window). Queue brokers provide much of this directly: RabbitMQ dead-letter exchanges with per-message TTL for delays, SQS redrive policies that move a message to a DLQ after a maximum receive count.

### Concrete: a message stuck in a retry loop

```text
 consumer log, partition 3 (no DLQ, infinite retry on any exception)
 10:00:01 offset 88231 attempt 1   error: customer c_5521 not found
 10:00:02 offset 88231 attempt 2   error: customer c_5521 not found
 10:00:04 offset 88231 attempt 3   …
 …
 14:12:40 offset 88231 attempt 2,973
 lag on partition 3: 0 at 10:00, 1.9 million at 14:12; other partitions: lag ~0
```

One message referencing a customer that was deleted (or not yet replicated to this service) blocked a partition for four hours. Diagnosis Lab 3 follows it.

### Mechanism: replay

A log's retention makes replay possible: reset a consumer group's offsets to a timestamp (Kafka provides tooling to reset a group's offsets "to datetime") and the group reprocesses everything since. Uses: rebuilding a corrupted read model, backfilling a new consumer, reprocessing after fixing a bug. Requirements, all of which must be true **before** the replay:

- consumers are idempotent by business keys, not by a short deduplication window;
- side effects that must not repeat (emails, charges) are suppressed or deduplicated during the replay, often with a "replay mode" flag;
- the downstream can absorb the volume (a replay is a planned overload, Chapter 24);
- the messages are still within retention.

### Guarantees and non-guarantees

A DLQ guarantees that a poison message stops blocking its partition or consuming retries, and is kept for inspection. Retry topics guarantee delayed retries without blocking. Neither guarantees ordering for the affected key, that anyone looks at the DLQ, or that redriving is safe; replay guarantees reprocessing only within retention and only safely with idempotent consumers.

### Failure and diagnosis

- **A partition blocked by a poison message.** Evidence: lag growing on one partition only; the consumer logging the same offset repeatedly. Fix: classify failures, DLQ after N attempts.
- **Messages silently accumulating in a DLQ.** Evidence: DLQ depth graph rising for weeks. Fix: alert on DLQ depth and age; an owner for every DLQ.
- **State moving backwards.** An entity's status reverts after retries. Evidence: event versions applied out of order. Fix: version checks in the consumer, or per-key blocking.
- **A replay that sends 40,000 emails.** Evidence: a spike in sends after an offset reset. Fix: replay mode, business-key idempotency.

### Backend connection

Decide for each consumer whether order matters per key; if it does, key by that entity, process sequentially per key, and make updates version-checked so that late arrivals cannot move state backwards. Classify failures as transient or permanent, retry transient ones with backoff (in place briefly, then via delayed retry), send permanent ones to a DLQ with the error attached, alert on every DLQ, and make redrive and replay routine, safe operations.

---

## 25.8 The Dual-Write Problem and the Transactional Outbox

### Definitions

- The **dual-write problem** (Chapter 22, Section 22.10) is the impossibility of atomically updating two separate systems (here, a database and a message broker) from application code: one write can succeed while the other fails, and concurrent requests can apply them in different orders.
- The **transactional outbox** pattern writes the message to an **outbox table** in the **same database transaction** as the business change; a separate **relay** process reads the outbox and publishes its rows to the broker.
- A **polling relay** (polling publisher) queries the outbox periodically. A **log-tailing relay** reads committed outbox inserts from the database's replication log with change data capture (Chapter 22, Section 22.10).

### Why it exists

A service that changes its database and publishes an event about the change must not let the two disagree: an order that exists with no "order placed" event means no confirmation email, no warehouse shipment and no invoice; an event about an order that was rolled back means a shipment for an order that does not exist. Two-phase commit across the database and the broker would solve it at the costs that Chapter 23 (Section 23.9) explained, and most brokers do not support it anyway. The outbox makes the database the only system written by the request, so its own transaction is enough.

### Concrete: the dual-write failure timeline

```text
 ORDER: DATABASE FIRST, THEN BROKER
 app                              database                    broker
 BEGIN; INSERT order 9001 ──────► 
 COMMIT ─────────────────────────► committed ✔
 publish "order.placed 9001" ───────────────────────────────► ✕ broker unreachable / app crashes /
                                                                 deploy kills the process here
 result: order 9001 exists; no event, ever. No email, no shipment, no invoice.  → LOST EVENT

 ORDER: BROKER FIRST, THEN DATABASE
 publish "order.placed 9001" ───────────────────────────────► stored ✔ (consumers start work)
 BEGIN; INSERT order 9001 ──────► constraint violation / deadlock / crash → ROLLBACK
 result: the warehouse ships an order that does not exist.                       → PHANTOM EVENT

 PUBLISH INSIDE THE TRANSACTION (before COMMIT)
 BEGIN; INSERT; publish ✔; COMMIT fails ✕ → phantom event again
 (and the transaction holds locks while waiting on the broker: Chapter 18, Section 18.12)
```

Every ordering fails somewhere, because the two writes have no shared commit point.

### Mechanism: the transactional outbox

```text
 request handler                              relay (separate process)              broker
 BEGIN;
   INSERT INTO orders … (9001)
   INSERT INTO outbox (id, aggregate, key,
          type, payload, created_at)
     VALUES (uuid, 'order', '9001',
          'order.placed', {…}, now())
 COMMIT;   ← ONE atomic commit: both rows or neither
                                               reads unpublished outbox rows
                                               publishes each, keyed by the
                                               aggregate ID ──────────────────────► "orders"
                                               marks published / deletes rows
```

```sql
CREATE TABLE outbox (
  id           uuid        PRIMARY KEY,         -- also the MESSAGE ID consumers deduplicate on
  aggregate    text        NOT NULL,            -- 'order'
  aggregate_id text        NOT NULL,            -- '9001' → the message key (ordering per order)
  type         text        NOT NULL,            -- 'order.placed'
  payload      jsonb       NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz                      -- NULL until the relay publishes it
);
CREATE INDEX outbox_unpublished ON outbox (created_at) WHERE published_at IS NULL;

-- polling relay, one batch (in a loop, every ~100 ms–1 s)
BEGIN;
SELECT id, aggregate_id, type, payload FROM outbox
WHERE published_at IS NULL
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;           -- several relay instances can share the work
-- publish each row to the broker; wait for the broker's acknowledgement (acks=all)
UPDATE outbox SET published_at = now() WHERE id = ANY($1);
COMMIT;
```

The relay publishes **at least once**: if it crashes after publishing and before marking rows, it publishes them again. That is why the outbox row's `id` is the message ID, and why consumers are idempotent (Section 25.6). The outbox guarantees that every committed business change produces its event and that rolled-back changes produce none; it moves the remaining problem (duplicates) to where Section 25.6 already solved it.

### Mechanism: polling relay versus CDC

| | Polling relay | CDC relay (log tailing, e.g. Debezium with an outbox router) |
|---|---|---|
| How | queries the outbox table periodically | reads committed inserts to the outbox from the WAL or binlog (Chapter 22, Section 22.10) |
| Latency | the polling interval (100 ms to seconds) | near real time |
| Load on the database | queries plus updates to mark rows | reading the log (a replication slot); no queries on the table |
| Ordering | by `created_at` or ID, which can differ from **commit** order: a transaction that started earlier can commit later, so a relay that polls "rows after the last ID I saw" can skip it | commit order, exactly as the log records it |
| Operations | simple; only the database and the relay | a CDC platform (connectors, often a Kafka Connect cluster), slot monitoring (Chapter 15, Section 15.10) |
| Cleanup | delete or archive published rows | the outbox rows can be deleted right after insert (the log already holds them) |

The ordering row deserves care. Sequence and identity values are assigned when rows are **inserted**, not when transactions **commit**, so a relay that reads "IDs greater than the last one published" can permanently miss a row from a slow transaction that committed after a higher ID was already published. Polling by "`published_at IS NULL`" (as above) never misses rows, at the cost of ordering only approximately by creation time; per-key ordering within one aggregate is preserved when each aggregate's events are created in separate, sequential transactions. CDC avoids the question by following commit order.

CDC can also be used **without** an outbox, publishing changes to business tables directly. That couples consumers to the internal table schema (a column rename becomes a breaking change to every consumer) and exposes low-level row changes rather than meaningful business events. The outbox lets a service publish deliberate, versioned events (Section 25.10) while still getting CDC's delivery.

### Guarantees and non-guarantees

The transactional outbox guarantees that an event is published for every committed change and for no rolled-back change, at least once, eventually. It does not guarantee exactly-once publication (consumers deduplicate), immediate publication (relay latency), global ordering across aggregates, or that the relay is running: a stopped relay delays every event silently unless its lag is monitored.

### Failure and diagnosis

- **A lost event.** An order exists, but no event was ever consumed. Evidence: no outbox row for it (the code path published directly to the broker, bypassing the outbox), or a polling relay that skipped it by ID. Diagnosis Lab 4. Fix: all publishing through the outbox; poll by unpublished state, or use CDC.
- **Events delayed by minutes.** Evidence: outbox rows with old `created_at` and null `published_at`. Cause: relay down or slow, or the broker unreachable. Fix: alert on the age of the oldest unpublished row.
- **Outbox table growing forever.** Evidence: millions of published rows. Fix: delete published rows in batches, or partition by day and drop old partitions.
- **WAL accumulation with CDC.** A stopped connector's slot retains WAL (Chapter 15, Section 15.10). Fix: slot lag alerts.

### Backend connection

Never publish to a broker from the request path as a second write. Insert outbox rows in the business transaction, give each the message ID consumers will deduplicate on and the aggregate ID as the key, relay them with a polling publisher or CDC, and monitor the age of the oldest unpublished event. For jobs within one service, a database-backed job queue enqueued in the same transaction is the same pattern (Section 25.2).

---

## 25.9 Sagas: Coordinating Multi-Service Business Processes

### Definitions

- A **saga** is a sequence of local transactions, each in one service's own database, that together implement a business process spanning several services. Each step publishes an event or message that triggers the next; if a step fails, **compensating transactions** undo the effects of the steps already completed.
- A **compensating transaction** (compensation) semantically reverses an earlier step: refund a payment, release a reservation, cancel an order. It is a new forward action, not a rollback: the original step happened and was visible.
- **Choreography**: each service reacts to events from others and publishes its own; there is no central coordinator.
- **Orchestration**: a coordinator (the **orchestrator**) tells each service what to do, waits for the reply, and decides the next step or the compensations, keeping the saga's state durably.

### Why it exists

An order that must be recorded, paid, reserved in inventory and confirmed touches four services with four databases. A single ACID transaction across them would need two-phase commit, which blocks, couples and is unsupported by most of the participants (Chapter 23, Section 23.9). A saga gives up atomic isolation and replaces it with **eventual** completion or compensation: either all steps eventually succeed, or the completed ones are undone.

### Concrete: an order saga, orchestrated

```text
 ORDER SAGA ORCHESTRATOR (state stored in its own table, one row per saga)
 step                       command to                 on success          on failure
 ─────────────────────────────────────────────────────────────────────────────────────────────
 1 create order (PENDING)   orders (local)             → step 2            end: rejected
 2 authorise payment        payments: Authorize        → step 3            compensate 1: cancel order
 3 reserve inventory        inventory: Reserve         → step 4            compensate 2: void payment,
                                                                            then 1
 4 capture payment          payments: Capture          → step 5            retry (see failure matrix)
 5 confirm order            orders: Confirm (local)    → step 6            retry
 6 notify customer          notifications: Send        → DONE              retry; never compensates

 saga state row: {saga_id, order_id: 9001, step: 3, status: RUNNING,
                  payment_auth_id: "au_77", attempts: 1, updated_at}
```

```text
 timeline: inventory fails
 orchestrator ──Authorize(9001, 49.98)──► payments    ◄── authorised au_77
 orchestrator ──Reserve(9001, lines)────► inventory   ◄── FAILED: out of stock
 orchestrator: start compensation, newest step first
 orchestrator ──Void(au_77)─────────────► payments    ◄── voided
 orchestrator: cancel order 9001 (local), status CANCELLED, reason OUT_OF_STOCK
 orchestrator ──Send(order cancelled email)──► notifications
```

### Concrete: the same saga, choreographed

```text
 orders:        INSERT order PENDING + outbox "OrderCreated"
 payments:      on OrderCreated → authorise → outbox "PaymentAuthorized" (or "PaymentFailed")
 inventory:     on PaymentAuthorized → reserve → "InventoryReserved" (or "InventoryFailed")
 payments:      on InventoryReserved → capture → "PaymentCaptured"
                on InventoryFailed → void the authorisation → "PaymentVoided"
 orders:        on PaymentCaptured → CONFIRMED + "OrderConfirmed"
                on PaymentFailed or PaymentVoided → CANCELLED + "OrderCancelled"
 notifications: on OrderConfirmed / OrderCancelled → email
```

| | Choreography | Orchestration |
|---|---|---|
| Where the process is defined | spread across services' event handlers | in one place (the orchestrator's state machine) |
| Coupling | services know each other's events | services know only their own commands; the orchestrator knows all |
| Visibility | "where is order 9001?" needs correlating events from several services | the saga row says which step it is on |
| Adding a step | change several services; risk of cyclic event chains | change the orchestrator |
| Failure handling | each service must react to failure events correctly | centralised compensation logic |
| Fits | short sagas with few participants | longer processes, many steps, timeouts, human steps |

Workflow engines (Temporal, Camunda, AWS Step Functions and others) implement durable orchestration: the process is written as code or a state machine, and the engine persists each step's state, retries, timers and compensations.

### Mechanism: the failure matrix

The heart of saga design is deciding, for every step, what happens if it fails, and what happens if its **compensation** fails:

| Step fails | Effect so far | Response | If the compensation fails |
|---|---|---|---|
| 1 create order | nothing | reject the request | n/a |
| 2 authorise payment (declined) | order PENDING | cancel order (compensate 1) | retry the local cancel until it succeeds |
| 2 authorise payment (timeout: unknown) | maybe an authorisation exists | query payment status by idempotency key; then continue or compensate | void by idempotency key; retry |
| 3 reserve inventory | order PENDING, payment authorised | void authorisation, cancel order | retry void with backoff; authorisations also expire on their own; alert if still failing |
| 4 capture payment | stock reserved | **retry** (capture of a valid authorisation should succeed; this is the **pivot**: after it, the saga goes forward only) | n/a: steps after the pivot are not compensated, they are retried |
| 5 confirm order | money captured | retry until success | n/a |
| 6 notify customer | everything done | retry; eventually DLQ and alert; never refund because an email failed | n/a |

Three rules come out of the matrix. **Compensations must be idempotent and retryable** until they succeed, because they are triggered by messages that can repeat and by failures that can recur. **Order the steps so that the hardest-to-undo step comes late**: the **pivot transaction** (here, capturing money) is the point of no return; steps before it are compensatable, steps after it must be designed to succeed eventually with retries. **Every request to another service carries an idempotency key** derived from the saga ID and step, so retries and timeouts (unknown outcomes) never double-charge or double-reserve.

### Mechanism: the missing isolation

Local transactions commit as the saga proceeds, so other requests can see intermediate states: an order that is PENDING, stock that is reserved for an order that will be cancelled, money authorised and then voided. This is the "I" missing from a saga's ACID. Countermeasures:

- **Semantic locks**: states such as `PENDING` mark data as part of an unfinished saga; other operations respect them (for example, a pending order cannot be modified, and its reserved stock is not shown as available).
- **Commutative updates**: design steps whose order does not matter (increments and decrements of available stock rather than "set stock to N").
- **Rereading values** before acting, or version checks (Section 25.7), so a step does not act on data that changed since the saga started.

### Guarantees and non-guarantees

A saga with durable state, idempotent steps and retried compensations guarantees that the process eventually reaches a consistent outcome: all steps done, or the completed compensatable steps undone. It does not guarantee isolation (intermediate states are visible), atomicity in time (the business is inconsistent for a while), that compensations restore the exact original state (a refund is not an un-charge; an email cannot be unsent), or progress if a compensation can fail permanently: those cases need human handling.

### Failure and diagnosis

- **Sagas stuck in intermediate states.** Evidence: saga rows (or orders) in `PENDING` far longer than normal. Cause: a lost message, a consumer down, a step waiting forever. Fix: timeouts per step that trigger retry or compensation; a sweeper over stale sagas; alerts.
- **Double charges or double refunds.** Evidence: two provider transactions for one saga step. Fix: idempotency keys per saga step.
- **Compensation that cannot succeed.** A refund rejected because the card was closed. Evidence: compensations in DLQ. Fix: a manual resolution queue with an owner; business processes for the exceptional case.
- **Cyclic or tangled choreography.** Evidence: nobody can say which service reacts to which event; events triggering each other in loops. Fix: move the process to an orchestrator; document the event flow.

### Backend connection

Use sagas for business processes that span services, and keep each step a local transaction with its outbox. Write the failure matrix before the code: for each step, its compensation, whether it is before or after the pivot, its idempotency key and its timeout. Prefer orchestration for anything with more than three or four steps, so that the process has one owner and one place to answer "where is this order?". Make intermediate states explicit and visible to users ("payment processing"), because a saga is consistent only eventually.

---

## 25.10 Event-Driven Architecture, Event Sourcing and CQRS

### Definitions

- A **command** is a message that asks one specific recipient to do something: `ReserveInventory(order 9001)`. It may be rejected, and its sender expects an outcome.
- An **event** is a message that announces that something has already happened: `OrderPlaced(order 9001)`. It is a fact about the past, named in the past tense, published without knowing who will react, and it cannot be rejected, only ignored.
- **Event-driven architecture** is a style in which services communicate mainly by publishing and reacting to events, rather than calling each other.
- An **event notification** carries only identifiers ("order 9001 was placed"); consumers call back for details. **Event-carried state transfer** carries the data consumers need ("order 9001 was placed: customer, lines, total"). These are the thin and fat events of Chapter 13 (Section 13.10).
- **Event sourcing** stores an entity's state as the ordered sequence of events that changed it, rather than as a current row; the current state is derived by replaying the events.
- **CQRS** (command query responsibility segregation) separates the model that handles writes (commands) from one or more models optimised for reads (queries), with the read models updated from the write side, usually by events.

### Why it exists

Commands keep the caller in charge: the order service tells inventory what to do and needs to know inventory exists. Events invert that dependency: the order service states what happened, and inventory, email, analytics and next year's fraud service each decide for themselves whether to react. New consumers can be added without touching the producer. The cost is that the overall behaviour of the system is no longer visible in any one place, and that each consumer's view of the world is eventually consistent with the producer's.

### Mechanism: events versus commands in practice

```text
 COMMAND  "ReserveInventory {order: 9001, lines: […]}"   → to: inventory (one recipient)
          sender waits for (or later receives) "Reserved" or "Rejected"; the sender owns the process

 EVENT    "OrderPlaced {order: 9001, customer: 81, lines: […], total: 49.98, at: …}"
          → to: topic "orders" (anyone); publisher does not know or care who consumes
          consumers: inventory reserves; email sends; analytics counts; fraud scores
```

A useful test: if the sender would be wrong to proceed without knowing the outcome, it is a command (or a synchronous call). If the sender is done once the fact is recorded, it is an event. Disguised commands ("`OrderPlacedPleaseReserveStock`") are a sign that the process belongs in an orchestrator (Section 25.9).

### Mechanism: event schemas and their evolution

Chapter 13 (Section 13.11) explained backward, forward and full compatibility, Protocol Buffers and Avro, and schema registries. Events add two pressures that APIs do not have:

- **Consumers upgrade independently and lag.** A producer that publishes a new event version reaches consumers that may be days behind in deploys. So event changes must be **backward compatible** for consumers (new optional fields with defaults; never remove or retype a field consumers use; never reuse a Protobuf field number), and breaking changes go to a new event type or topic (`OrderPlaced.v2`) published alongside the old one until every consumer has moved (expand and contract, Chapter 13, Section 13.9).
- **Old events live on.** A log with long retention, a compacted topic, or an event store keeps events written years ago. Any consumer that replays must still read every old version. Event-sourced systems commonly **upcast** old events to the current shape as they are read.

An event's envelope usually carries the fields every consumer and every tool needs, whatever the payload:

```json
{ "id": "9b1c…",                 "type": "order.placed",   "schema_version": 3,
  "source": "orders-service",    "occurred_at": "2026-10-24T10:02:11.482Z",
  "subject": "order/9001",       "correlation_id": "req-7f3a…",
  "data": { "order_id": "9001", "customer_id": "81", "total_cents": 4998, "lines": [ … ] } }
```

The `id` is the message ID for deduplication (Section 25.6), the `subject` is the partition key, and the `correlation_id` links the event to the request that caused it (Chapter 10, Section 10.11).

### Mechanism: event sourcing

```text
 STATE-ORIENTED (usual)              EVENT-SOURCED
 accounts: id=81, balance=120        events for account 81 (append-only, ordered by version)
                                       v1 AccountOpened      {owner: "Ana"}
                                       v2 MoneyDeposited     {amount: 200}
                                       v3 MoneyWithdrawn     {amount: 50}
                                       v4 MoneyWithdrawn     {amount: 30}
                                     current state = fold(events) → balance 120
                                     a command "withdraw 150" loads v1–v4, checks 120 ≥ 150 → reject;
                                     a valid command appends v5 with "expected version = 4"
                                     (optimistic concurrency, Chapter 17, Section 17.9)
```

What it gives: a complete audit history by construction; the ability to rebuild state, or build new read models, by replaying; and temporal queries ("what was the balance on 1 March?"). What it costs: every read of current state needs a fold (mitigated with **snapshots** every N events); events become a permanent public schema that must be versioned and upcast forever; correcting a mistake means appending a compensating event, never editing history; deleting personal data (for privacy law) from an immutable log needs special techniques (for example, encrypting each person's data with a per-person key and deleting the key); and most developers and tools expect current-state tables. It suits domains where history is the business (ledgers, audit-heavy workflows), and is costly overhead elsewhere.

### Mechanism: CQRS

```text
 commands ──► WRITE MODEL (normalised, enforces invariants, e.g. PostgreSQL orders tables)
                 │ events via outbox / CDC
                 ▼
            projectors (consumers) ──► READ MODEL A: "orders by customer" (denormalised table)
                                   ──► READ MODEL B: search index (Elasticsearch)
                                   ──► READ MODEL C: dashboard aggregates (column store)
 queries ──► read models (fast, shaped for each screen; EVENTUALLY consistent with the write model)
```

CQRS lets each read path have a store and a shape built for it, and lets reads and writes scale separately. The cost is the lag between a write and its appearance in read models, which users notice right after their own writes (Section 25.1), plus the code and operations of every projector. It is a pattern for specific read-heavy or query-diverse parts of a system, not a default architecture; the polyglot-persistence rules of Chapter 23 (Section 23.11) apply: one source of truth, derived stores rebuildable from it.

### Guarantees and non-guarantees

Event-driven communication guarantees that producers are decoupled from the set of consumers and that consumers can be added without changing producers. It does not guarantee that the overall process completes (no one owns it), that consumers agree at any instant, or that schema changes are safe without compatibility discipline. Event sourcing guarantees a complete, replayable history; it does not make current-state queries cheap or history editable. CQRS guarantees read models shaped for their queries; it does not make them current.

### Failure and diagnosis

- **A breaking event change.** Consumers fail to deserialise after a producer deploy. Evidence: deserialisation errors and DLQ growth starting at the deploy. Fix: compatibility checks in the schema registry or in CI; new types for breaking changes.
- **Read model drift.** A projection disagrees with the write model. Evidence: a reconciliation query comparing counts or checksums. Cause: a projector bug, a skipped event, non-idempotent projection. Fix: rebuild the projection by replay; idempotent, version-checked projectors.
- **Nobody can explain the flow.** Evidence: incidents where teams cannot say which services react to an event. Fix: an event catalogue (each event's owner, schema and consumers), tracing with correlation IDs, orchestration for processes that need an owner.

### Backend connection

Use commands when the sender owns the outcome and events when it only records a fact. Version every event schema, keep changes backward compatible for consumers, and put an ID, type, version, subject and correlation ID in every envelope. Adopt event sourcing only where history is the product, and CQRS only where one model cannot serve both writes and reads; in both cases, design for replay and for the lag users will see.

---

## 25.11 Backpressure and Consumer Lag

### Definitions

- **Consumer lag** is how far a consumer is behind its input. In a log, per partition: **lag = log-end offset − committed offset**, in messages. **Time lag** is how old the oldest unprocessed message is, which is what users feel. In a queue, the equivalent is **queue depth** and the age of the oldest message.
- **Consumer throughput** is messages processed per second; **consumer capacity** is the maximum sustainable throughput of all consumers together.

### Why it matters

A broker is a buffer, and buffers hide overload. When producers publish faster than consumers process, nothing fails: lag simply grows. The symptoms appear elsewhere and later: emails arriving hours late, a read model showing yesterday's data, and finally data loss when lag exceeds retention and unprocessed messages are deleted. Lag is the one metric that tells a messaging system's health.

### Mechanism: the capacity arithmetic

Little's Law (Chapter 11, Section 11.3) sizes consumers like any other server:

```text
 incoming: 2,000 messages/s at peak; processing time per message: 25 ms (mostly a DB write)
 concurrency needed = 2,000 × 0.025 = 50 messages in progress at once
 with one message at a time per partition-consumer, that needs ≥ 50 partitions (and 50 consumer
 threads); with 12 partitions, each consumer must process concurrently within its partition
 (grouping by key to keep per-key order, Section 25.7), or processing must get faster (batching
 50 rows per DB write cuts per-message time)

 lag growth when capacity is short: incoming 2,000/s, capacity 1,600/s → lag grows 400/s
 = 1.44 million messages per hour; time to drain after the peak (incoming back to 800/s):
 1.44 M / (1,600 − 800) = 1,800 s = 30 min
```

A pull-based consumer (Kafka-style) applies backpressure naturally: it fetches only what it can handle, and lag accumulates in the broker, which is designed to hold it. A push-based broker (RabbitMQ-style) needs a **prefetch** limit, or it floods consumers' memory (Chapter 11, Section 11.8's unbounded buffer). Producers are usually not slowed at all, which is the point of the buffer, so the protection that matters is: lag stays well within retention, and consumers protect **their** downstream (a database, a partner API) with their own concurrency limits and rate limits (Chapter 24, Sections 24.5–24.6), even if that means lag grows during a spike.

### Concrete: lag graphs and what they mean

```text
 lag (messages)
  ▲
  │ A: steady climb                    B: sawtooth                  C: one partition
  │        ╱                           ╱╲  ╱╲  ╱╲  ╱╲                 p3  ╱──────────
  │      ╱                            ╱  ╲╱  ╲╱  ╲╱  ╲               ╱
  │    ╱                                                         ───╱  others ~0 ─────
  │  ╱
  └────────────────► time
  A: capacity < input rate      B: batch producer or periodic       C: hot key, slow message,
     (consumer too slow, too       consumer pauses (GC, commits,       or a POISON message
     few partitions/consumers,     rebalances); fine if the troughs    blocking one partition
     a slow downstream)            return to ~0

  D: sudden step up then recovery: a rebalance or deploy paused consumption (normal if brief;
     repeated steps = rebalance storm, Section 25.4)
```

```bash
# per-partition lag for a consumer group (Kafka's command-line tool)
kafka-consumer-groups.sh --bootstrap-server broker:9092 --describe --group email-sender
# GROUP         TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG      CONSUMER-ID ...
# email-sender  orders  0          1203391         1203402         11       c-1 ...
# email-sender  orders  3          880412          2781553         1901141  c-2 ...   ← one partition
```

Alert on time lag (or on message lag converted to time at the current rate) against what the business can tolerate ("confirmation emails within 5 minutes"), and separately on lag approaching retention.

### Mechanism: what to do about lag

| Cause | Evidence | Fix |
|---|---|---|
| consumers too slow for the input | graph A; all partitions lag; consumer CPU or downstream saturated | faster processing (batching, fewer round trips), more consumers up to the partition count, more partitions (carefully for ordered topics) |
| a slow downstream | consumer threads waiting on the database or an API | fix or scale the downstream; bulkhead and limit (the consumer should not overload it); accept lag |
| a poison message | graph C; same offset in logs repeatedly | DLQ after N attempts (Section 25.7) |
| a hot key | graph C without errors; one partition's input rate high | better keys; split the entity's work |
| rebalance storms | graph D repeated; rebalance logs | processing within poll interval; cooperative rebalancing; static membership |

Autoscaling consumers by lag is common (Chapter 26), with one limit: in a partitioned log, consumers beyond the partition count sit idle.

### Guarantees and non-guarantees

A broker with sufficient retention guarantees that messages wait while consumers are slow. Lag metrics guarantee visibility of how far behind consumers are. Nothing guarantees that lag is bounded: if input exceeds capacity for long enough, lag reaches retention and messages are lost to that consumer.

### Failure and diagnosis

- **Growing lag.** Diagnosis Lab 1 walks through it.
- **Lag that looks fine while messages are stuck.** A consumer commits offsets but parks failures in an in-memory retry buffer that is lost on restart, or commits before processing. Evidence: produced and processed counts diverge while lag stays low. Fix: commit after processing; durable retry topics.
- **Lag beyond retention.** Evidence: a consumer resetting to earliest or latest offset with a warning about an out-of-range offset. Result: data skipped. Fix: alerts on lag versus retention; longer retention for critical topics.

### Backend connection

Size consumers with Little's Law, partitions for the parallelism you will need, and retention for the longest outage you must survive. Monitor time lag per consumer group and per partition with alerts tied to business expectations, protect consumers' downstreams with limits even at the cost of lag, and read lag graph shapes as diagnoses: a climb is capacity, a single partition is a key or a poison message, repeated steps are rebalances.

---

## 25.12 Worked Workflow: Order, Payment, Inventory, Notification

### The design

This section assembles the chapter into one workflow and then breaks it deliberately, step by step. The design:

```text
 client ── POST /orders (Idempotency-Key: k-55) ──► ORDERS SERVICE
   one transaction: INSERT order 9001 (PENDING) + INSERT saga row + INSERT outbox "OrderPlaced"
   ◄── 202 Accepted {order 9001, status PENDING, status URL}   (Chapter 13, Section 13.10)

 ORDER SAGA ORCHESTRATOR (inside the orders service; state in the saga table)
   reads OrderPlaced → sends commands via outbox → waits for replies (events) → next step
   step 1 Authorize payment   → PAYMENTS (idempotency key: saga-9001-authorize)
   step 2 Reserve inventory   → INVENTORY (key: saga-9001-reserve)
   step 3 Capture payment     → PAYMENTS (key: saga-9001-capture)            ← pivot
   step 4 Confirm order       (local)
   step 5 Notify customer     → NOTIFICATIONS (key: saga-9001-notify)

 every service: commands consumed at least once, inbox table for deduplication, effects and
 replies committed together with outbox rows; topics keyed by order ID (per-order ordering)
```

### Failures injected at each step

| # | Failure injected | What happens | Mechanism that handles it |
|---|---|---|---|
| 1 | The client's connection drops after it sends `POST /orders`; the client retries | the retry carries the same idempotency key; the orders service returns the stored `202` for order 9001 | API idempotency key (Chapter 13, Section 13.5) |
| 2 | The orders service crashes right after committing order 9001 | the outbox row was committed with the order; the relay publishes `OrderPlaced` when it next runs | transactional outbox (Section 25.8) |
| 3 | The relay publishes `OrderPlaced`, crashes before marking the row, and publishes it again | the orchestrator's inbox sees the same message ID twice and skips the second | idempotent consumer (Section 25.6) |
| 4 | The payment provider times out on `Authorize` | outcome unknown; the payments service queries the provider by idempotency key `saga-9001-authorize`; finds an authorisation, replies `PaymentAuthorized` | idempotency keys downstream; unknown outcome (Section 24.1) |
| 5 | Inventory is out of stock | inventory replies `ReservationFailed`; the orchestrator voids the authorisation (key `saga-9001-void`), cancels the order, and sends a cancellation email | compensation (Section 25.9) |
| 6 | The inventory service is down for 20 minutes | `ReserveInventory` waits in its topic; the saga's step timeout (30 minutes) has not expired; when inventory returns, it processes the backlog | durable messaging; step timeouts |
| 7 | Inventory is down for 2 hours; the payment authorisation would expire | the step timeout fires; the orchestrator compensates (void, cancel) and notifies the customer; late `Reserved` replies after cancellation trigger a release compensation | saga timeouts; late-reply handling |
| 8 | `Capture` fails with a transient error | after the pivot, the saga retries capture with backoff (never compensates); after the retry horizon, it alerts a human | pivot rule; DLQ and alert (Section 25.7) |
| 9 | The orchestrator crashes between steps 2 and 3 | on restart, it loads saga 9001 at step 2 complete and sends `Capture` (via outbox, same key) | durable saga state |
| 10 | `PaymentCaptured` and `OrderConfirmed` events are delivered to the notification consumer out of order after a retry | the notification consumer acts only on `OrderConfirmed` and checks the order version; a stale event is ignored | version checks (Section 25.7) |
| 11 | The email provider rejects a malformed address | permanent failure → DLQ with the reason; the order stays confirmed; support is alerted | failure classification; never compensate a notification |
| 12 | A consumer rebalance makes the notification consumer re-process `OrderConfirmed` | the email request carries idempotency key `saga-9001-notify`; the provider (or the service's sent-log check) suppresses the duplicate | idempotency at the external effect |
| 13 | The broker loses its partition leader during step 2 | producers with `acks=all` retry against the new leader; no acknowledged message is lost; duplicates from retries are removed by the idempotent producer or inboxes | replication and acknowledgements (Section 25.4) |

### Concrete: failure 7 traced in detail

```text
 10:00  OrderPlaced 9001 → orchestrator: step 1 Authorize → PaymentAuthorized (au_77, valid 7 days)
 10:00  step 2: outbox → "ReserveInventory 9001" (deadline 10:30)
 10:00–12:00  inventory service down; the command waits in topic "inventory.commands", partition 5
 10:30  orchestrator's timer: step 2 deadline passed, no reply
        → compensation: outbox "VoidAuthorization au_77 (key saga-9001-void)"; order → CANCELLED
          (reason: INVENTORY_TIMEOUT); outbox "SendEmail order-cancelled (key saga-9001-notify-cancel)"
 10:31  payments: voided au_77 → "AuthorizationVoided"; notifications: cancellation email sent
 12:05  inventory recovers, processes the backlog: reserves stock for 9001 → "Reserved 9001"
 12:05  orchestrator: saga 9001 is CANCELLED; a late success for a cancelled saga
        → compensation for the late step: outbox "ReleaseReservation 9001 (key saga-9001-release)"
 12:06  inventory releases the stock. Final state: order cancelled, no charge, no reserved stock,
        customer informed. Consistent, eventually, without any distributed transaction.
```

The design questions that this trace forces are the real lesson: what the step timeout should be (shorter than the payment authorisation's validity, longer than normal inventory outages), what happens to late replies, and what the customer is told at each moment.

### Guarantees and non-guarantees

This design guarantees that every accepted order ends in a consistent final state (confirmed and charged, or cancelled with no charge and no reserved stock), that no step's effect is applied twice, and that no event is lost between a database commit and the broker. It does not guarantee how long that takes, that the customer never sees an intermediate state, or that steps after the pivot succeed without human help in extreme cases.

### Backend connection

Build multi-service workflows from four pieces: outbox for every state change, idempotent consumers with inbox tables, a durable saga with a failure matrix and step timeouts, and idempotency keys on every external effect. Then inject each failure in this section's table in a test environment (Chapter 24, Section 24.12) before trusting the workflow with real orders.

---

## 25.13 Diagnosis Toolkit: Seeing Messages Move

### Definition

The **messaging diagnosis toolkit** is the set of broker metrics, consumer metrics, logs and identifiers that show where a message is, how long it has waited, how many times it was attempted, and whether its effect happened.

### Why it matters

Asynchronous failures are separated from their causes in time and place. A complaint ("I never got my confirmation email") arrives hours after the event, in a different service from the one that failed. Without the right identifiers and metrics, finding one message's path through a producer, a relay, a broker, a consumer and an external provider is guesswork.

### Concrete: the signals

| Question | Signal |
|---|---|
| Are consumers keeping up? | time lag and message lag per group and partition; queue depth and oldest message age |
| Are producers publishing? | outbox: count and age of unpublished rows; broker: incoming messages per topic |
| Is anything stuck? | DLQ depth and age; the same offset or message ID repeated in consumer logs; sagas in non-final states beyond their normal duration |
| Are there duplicates? | inbox conflicts per consumer (a rising rate is normal at rebalances, suspicious otherwise); external effect logs grouped by idempotency key |
| Is the group stable? | rebalance count and duration; consumer restarts |
| Is the broker healthy? | under-replicated partitions, offline partitions, ISR shrinks, disk usage versus retention |
| Where is message X? | message ID and correlation ID in every log line from request to outbox to consumer to provider |

Three practices make the table possible: put the **message ID** and **correlation ID** in every log line that touches a message; record **attempt counts** in headers; and keep a **message journal** for critical flows (a table or log of "order 9001: OrderPlaced published 10:00:01, consumed by email 10:00:03, email accepted by provider 10:00:04").

### Mechanism: symptom to first check

| Symptom | First check | Typical cause |
|---|---|---|
| work delayed for everyone | lag graph shape; consumer and downstream saturation | consumer capacity, slow downstream |
| work delayed for some entities | per-partition lag; repeated offsets | poison message, hot key |
| duplicate effects | the effect's log grouped by message ID; rebalances at those times | non-idempotent consumer, idempotency key missing downstream |
| an effect never happened | outbox row exists? published? consumed? DLQ? | bypassed outbox, skipped by relay, DLQ, lag beyond retention |
| sagas stuck | saga rows by step and age; the step's consumer | lost reply, missing timeout, consumer down |

### Backend connection

Instrument messaging end to end: outbox age, lag per partition in time, DLQ depth with owners, rebalance counts, inbox conflict rates, and message and correlation IDs in every log line. Those signals turn "the email never arrived" from a day of searching into a five-minute query.

---

## 25.14 Placement-Round Topics: JMS and Messaging Protocols

> **Placement-round focus:** This section is revision material for Java/J2EE and middleware questions in campus placement interviews and university courses (enterprise Java, distributed systems, IoT). It is not part of the main flow. What carries into backend work: the queue/topic distinction is the point-to-point/pub-sub distinction of Section 25.3, and every protocol's "exactly once" is a per-hop guarantee, never an end-to-end one (Section 25.5).

### JMS (Java Message Service)

**JMS**, now called **Jakarta Messaging**, is a Java API standard for **message-oriented middleware** (software that lets applications communicate by exchanging messages through a broker). It defines interfaces, not a wire protocol; brokers such as ActiveMQ implement it.

- **Two messaging domains**: **point-to-point**, through a **Queue** (each message consumed by one receiver), and **publish/subscribe**, through a **Topic** (each message delivered to every subscriber).
- **Core objects**: `ConnectionFactory` → `Connection` → `Session` → `MessageProducer` / `MessageConsumer`, sending to and receiving from a `Destination` (a `Queue` or a `Topic`).
- **Message types**: `TextMessage`, `BytesMessage`, `MapMessage`, `StreamMessage`, `ObjectMessage` (serialised Java objects, with the deserialisation risks of Chapter 12, Section 12.7).
- **Acknowledgement modes**: `AUTO_ACKNOWLEDGE` (the session acknowledges automatically after delivery), `CLIENT_ACKNOWLEDGE` (the application acknowledges explicitly), `DUPS_OK_ACKNOWLEDGE` (lazy acknowledgement that may cause duplicates), and transacted sessions (acknowledge on commit).
- **Delivery modes**: **persistent** (stored by the broker, survives restarts) and **non-persistent** (may be lost).
- A **durable subscription** keeps messages for a topic subscriber while it is disconnected; a non-durable subscriber misses messages published while it is offline.
- A **message-driven bean** (MDB) is a Jakarta EE component invoked by the container for each message, the classic way to consume JMS messages in application servers.

### AMQP, MQTT and STOMP

| Protocol | What it is | Typical use | Delivery levels |
|---|---|---|---|
| **AMQP** | a binary wire protocol for messaging; version 0-9-1 (RabbitMQ's native model of exchanges, queues and bindings) and version 1.0 (a different, more general standard) | enterprise messaging, task queues | acknowledgements and publisher confirms (at-least-once with persistence) |
| **MQTT** | a lightweight binary publish/subscribe protocol over TCP, with small headers, designed for unreliable networks and small devices | IoT sensors, mobile push | QoS 0 (at most once), QoS 1 (at least once), QoS 2 (exactly once **between one client and the broker**, via a four-step handshake) |
| **STOMP** | a simple text-based protocol (frames like HTTP's) | browser and scripting clients, often over WebSocket (Chapter 10, Section 10.9) | acknowledgements, depending on the broker |

MQTT's QoS 2 is a common exam question and a common misunderstanding: it prevents duplicates on one hop (client to broker, or broker to client) through the PUBLISH, PUBREC, PUBREL, PUBCOMP exchange, but a message that crosses two hops, or triggers an effect in an application, still needs the end-to-end reasoning of Section 25.5.

### Drill for this section

**C1. Explain the JMS point-to-point and publish/subscribe domains, durable subscriptions and acknowledgement modes.**
A strong answer must mention:
- Queue: one consumer per message; Topic: every subscriber gets each message.
- Durable subscriptions keep messages for offline subscribers; non-durable ones miss them.
- AUTO, CLIENT and DUPS_OK acknowledgement and transacted sessions; persistent versus non-persistent delivery.
- JMS is an API, not a protocol.
*Common weak answer:* "A topic is a queue with many consumers." Competing consumers on a queue share messages; topic subscribers each receive all of them.

**C2. What do MQTT's QoS levels guarantee, and why is QoS 2 not end-to-end exactly-once?**
A strong answer must mention:
- QoS 0 at most once, QoS 1 at least once (duplicates possible), QoS 2 exactly once per hop via a four-step handshake.
- The guarantee covers one client–broker hop; multi-hop paths and application effects need idempotency.
- Higher QoS costs more round trips and state.
*Common weak answer:* "QoS 2 means the message is processed exactly once by the application." It only prevents duplicate delivery on one hop.

---

## Chapter Summary

Asynchronous work moves everything the user does not need to see immediately off the request path: latency falls to the steps before the hand-off, failures are isolated behind a durable buffer, and the price is **eventual consistency**, delayed errors and harder debugging. **Background jobs** run on workers from a task queue with leases, retries with backoff and jitter, delays, scheduling and dead states; a **database-backed queue** with `SKIP LOCKED` can be enqueued in the business transaction itself. **Point-to-point queues** distribute work, **publish/subscribe** distributes information, and **partitioned logs** keep an ordered, replayable history; RabbitMQ-style brokers track and delete each message, Kafka-style brokers append to partitions and let each **consumer group** track one **offset** per partition. In a log, **ordering is per partition, hence per key**; consumer groups split partitions among members, **rebalances** move them (and cause reprocessing), **retention** deletes old segments and **compaction** keeps the latest value per key.

**Delivery semantics** come down to when a consumer acknowledges: before processing loses messages on a crash (**at-most-once**), after processing duplicates them (**at-least-once**). **Exactly-once** is an end-to-end application property, obtained by at-least-once delivery plus **idempotent** or **transactional** processing; broker features (idempotent producers, transactions, deduplication windows) never cover external side effects. **Idempotent consumers** use naturally idempotent effects, business unique constraints, or an **inbox table** written in the same transaction as the effect, and pass message IDs as idempotency keys to external systems. **Poison messages** go to **dead-letter queues** after classified, bounded retries; retries and parallel processing break per-key **ordering** unless keys are blocked or updates are version-checked; **replay** is safe only with business-key idempotency.

The **dual-write problem** has no ordering of "write the database, publish the event" that survives a crash; the **transactional outbox** commits the event with the business change and a **polling relay** or **CDC** publishes it at least once. **Sagas** coordinate multi-service processes with local transactions and **compensations**, by **choreography** or **orchestration**, designed from a **failure matrix** with a **pivot** step, idempotent and retried compensations, step timeouts and explicit intermediate states. **Events** state facts and decouple producers from consumers; **commands** ask for outcomes; event schemas evolve compatibly because consumers lag and old events are replayed. **Event sourcing** stores history as the source of truth and **CQRS** builds read models from it, each with real costs. **Consumer lag**, sized and diagnosed with Little's Law and lag graph shapes, is the health metric of every asynchronous system.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Asynchronous hand-off | Producer unaffected by consumer speed or failure; work waits while consumers are down | When work happens; consistency seen by the user meanwhile |
| Task queue with leases | Each job runs at least once; crashed workers' jobs are retried | Exactly-once execution; exact timing; ordering |
| Queue broker (ack-based) | Acked messages not redelivered; unacked redelivered | Ordering with competing consumers; replay after ack |
| Partitioned log | Order per partition; durable retention; resume from committed offsets | Order across partitions; that committed offsets match processing |
| Producer `acks=all` + idempotent producer | No loss of acknowledged records on one broker failure; no duplicates from producer retries | Deduplication across producer restarts or application resends |
| Ack after processing | No message loss within retention | No duplicates |
| Broker transactions | Atomic consume-transform-produce within the broker | Exactly-once external side effects |
| Inbox table (same transaction as effect) | Each message ID's effect applied once (in that database) | External effects; deduplication if IDs change on retry |
| DLQ with bounded retries | Poison messages stop blocking and are kept | That anyone handles them; ordering for the key |
| Transactional outbox | An event for every committed change and none for rolled-back ones, at least once | Exactly-once publication; immediate publication |
| Saga with compensations | Eventual consistent outcome or compensation | Isolation; restoring exact prior state; progress if compensation fails permanently |
| Event-driven communication | Producers decoupled from the set of consumers | Process completion; consistent views across consumers |
| Event sourcing | Complete, replayable history | Cheap current-state reads; editable history |
| CQRS read models | Reads shaped for their queries | Freshness |
| Consumer lag monitoring | Visibility of how far behind consumers are | That lag stays within retention |

---

## Interview Drill

**Q1. When should work be asynchronous? What does it cost?**
A strong answer must mention:
- Async for steps the user need not see in the response and that may fail without failing the action (emails, notifications, exports, image processing).
- Benefits: lower latency, failure isolation, load levelling, decoupling.
- Costs: eventual consistency in the UI, errors handled after the user has left, duplicates and ordering to manage, harder debugging.
- Required steps stay synchronous or become explicit pending states.
*Common weak answer:* "Make everything async to scale." Steps whose outcome the user needs cannot simply be deferred.

**Q2. How does a background job queue work? A database-backed queue versus a broker?**
A strong answer must mention:
- Enqueue, claim with a lease, process, mark done; expired leases retried; backoff via run_at; dead state after max attempts.
- Jobs can run more than once: handlers must be idempotent.
- DB queue (`SKIP LOCKED`): enqueue in the business transaction, simple, limited throughput, table bloat.
- Broker: higher throughput, built-in delivery features, but a separate system and the dual-write problem.
*Common weak answer:* "Use a cron job." Cron schedules; it does not provide durable, retried, distributed execution.

**Q3. Queue versus pub/sub versus log: RabbitMQ versus Kafka?**
A strong answer must mention:
- Queue: one consumer per message, deleted on ack; pub/sub: every subscriber gets a copy; log: ordered, retained, replayable, read by offset.
- RabbitMQ: exchanges, bindings, per-message acks, routing, DLX, priorities; poor replay.
- Kafka: partitions, consumer groups with offsets, retention and compaction, ordering per partition, high throughput; per-message retry built by the application.
- Choose by need: work distribution versus event streams and replay.
*Common weak answer:* "Kafka is a faster RabbitMQ." They implement different models.

**Q4. How does Kafka store and deliver messages? Partitions, offsets, consumer groups?**
A strong answer must mention:
- Topics split into partitions, each an append-only replicated log; records addressed by offset.
- Keys hashed to partitions; producers acknowledge with acks 0, 1 or all.
- A consumer group assigns each partition to one member; one committed offset per partition; parallelism ≤ partitions.
- Rebalances on join, leave or missed polls; reprocessing from the committed offset; retention and compaction.
*Common weak answer:* "Kafka deletes messages after they are consumed." Retention, not consumption, deletes them.

**Q5. How does Kafka guarantee ordering, and how can retries break it?**
A strong answer must mention:
- Order only within a partition; same key → same partition → ordered per key.
- Changing the partition count remaps keys and breaks ordering at the boundary.
- Producer retries can reorder or duplicate without the idempotent producer.
- Consumer-side retries via retry topics, or parallel processing within a partition, let later messages overtake; fixes: per-key blocking, version checks, sequential per-key processing.
*Common weak answer:* "Kafka guarantees global order." Only per partition.

**Q6. What do at-most-once, at-least-once and exactly-once delivery mean?**
A strong answer must mention:
- Ack before processing → at-most-once (loss on crash); ack after → at-least-once (duplicates on crash).
- A crash between the effect and the ack is always possible; no ordering avoids both.
- Exactly-once is an end-to-end application property: at-least-once plus idempotent or transactional processing.
- Broker features (idempotent producer, transactions, dedup windows) do not cover external side effects.
*Common weak answer:* "Kafka has exactly-once, so duplicates cannot happen." Not for effects outside Kafka.

**Q7. How do you make a consumer idempotent?**
A strong answer must mention:
- Stable message IDs assigned when the business event happens.
- Naturally idempotent effects (absolute values, upserts) or business unique constraints.
- Inbox table: insert the message ID and apply the effect in the same transaction; skip on conflict; ack after commit.
- External effects: pass the ID as an idempotency key; record intent and completion.
*Common weak answer:* "Check whether the message was processed, then process it." A check and an effect in separate steps race and can lose or duplicate.

**Q8. What is a poison message? How do retries and dead-letter queues work?**
A strong answer must mention:
- A message that always fails; in a log it blocks its partition, in a queue it cycles.
- Classify failures: transient (retry with backoff, in place then via delayed retry topics) versus permanent (straight to DLQ).
- DLQ with error details, alerts and an owner; redrive after fixing; consumers idempotent for redrive.
- Retries affect ordering for the key.
*Common weak answer:* "Retry until it works." A poison message never works and stalls everything behind it.

**Q9. What is the dual-write problem? What is the transactional outbox pattern?**
A strong answer must mention:
- Writing to a database and a broker from application code cannot be atomic: lost events (DB then broker) or phantom events (broker then DB).
- Outbox: write the event to an outbox table in the same transaction as the change; a relay publishes it.
- At-least-once publication; the outbox row ID is the message ID for consumer deduplication.
- Monitoring the oldest unpublished row; cleanup of published rows.
*Common weak answer:* "Publish after commit and retry on failure." A crash between commit and publish loses the event.

**Q10. Outbox polling relay versus CDC: which and why?**
A strong answer must mention:
- Polling: simple, interval latency, queries and updates on the table, ordering by creation not commit (IDs can be skipped if polled by "greater than last ID").
- CDC: reads the WAL/binlog, near real time, commit order, no table polling, but connector infrastructure and slot monitoring.
- CDC directly on business tables couples consumers to internal schemas; the outbox publishes deliberate events.
*Common weak answer:* "CDC replaces the outbox." CDC is a way to relay the outbox; without it, consumers depend on internal tables.

**Q11. What is a saga? Choreography versus orchestration? What is a compensating action?**
A strong answer must mention:
- A sequence of local transactions across services, with compensations undoing completed steps on failure; replaces 2PC.
- Choreography: services react to each other's events; orchestration: a coordinator with durable state issues commands.
- Compensation: a semantic reversal (refund, release), idempotent and retried; the pivot step after which the saga only goes forward.
- Missing isolation: semantic locks, commutative updates, version checks; idempotency keys per step; timeouts.
*Common weak answer:* "A saga is a distributed transaction that rolls back." It never rolls back; it compensates visible effects.

**Q12. What is the difference between an event and a command? How do you evolve event schemas?**
A strong answer must mention:
- Command: a request to one recipient that may be rejected; event: a fact about the past, published to anyone.
- Thin (notification) versus fat (state-carrying) events.
- Backward-compatible changes for lagging consumers; new types or topics for breaking changes, expand and contract; schema registry checks.
- Old events are replayed: readers handle all versions; upcasting.
*Common weak answer:* "They are both messages." The difference is ownership of the outcome.

**Q13. What are event sourcing and CQRS, and what do they cost?**
A strong answer must mention:
- Event sourcing: state as an append-only sequence of events, folded into current state; snapshots; optimistic concurrency on expected version.
- Benefits: audit, rebuilds, temporal queries; costs: permanent event schemas, upcasting, no editing, privacy deletion, unfamiliarity.
- CQRS: separate write and read models, read models updated by events; eventual consistency; projector code and operations.
- Use selectively, not as defaults.
*Common weak answer:* "Event sourcing means using Kafka." It is a storage model for an entity's state, independent of any broker.

**Q14. What is consumer lag, and how do you diagnose and fix it?**
A strong answer must mention:
- Lag = log-end offset − committed offset per partition; time lag is what users feel.
- Graph shapes: steady climb (capacity), sawtooth (batches, pauses), one partition (hot key or poison message), repeated steps (rebalances).
- Little's Law sizing; parallelism limited by partitions; protect the downstream.
- Alerts on time lag versus business expectations and versus retention.
*Common weak answer:* "Add more consumers." Beyond the partition count they sit idle, and a poison message is not fixed by more consumers.

---

## Scenario Exercises

1. A signup endpoint takes 2.4 seconds because it creates the account, sends a welcome email, creates a CRM contact, and generates a default avatar. Redesign it, stating which steps stay synchronous, how the others are triggered reliably, and what the user sees.
2. A team uses a single Kafka topic with 6 partitions for all customer events, keyed by event type ("profile_updated", "address_changed", …). Consumers report out-of-order updates for individual customers, and one consumer is far busier than the others. Explain both problems and redesign the key.
3. A consumer of `payments` updates a ledger in PostgreSQL and then commits its Kafka offset. After a deploy, auditors find a handful of payments recorded twice. Explain how, and give two designs that prevent it.
4. An e-commerce service saves an order and then publishes `OrderPlaced` to RabbitMQ. During a broker maintenance window, 300 orders were saved but never shipped. Explain, and design a fix that would also survive the relay itself crashing.
5. Design the saga for a travel booking (flight, hotel, car) in which the car rental partner has no cancellation API and charges a fee for no-shows. Where is the pivot? What is the failure matrix?
6. A notification consumer's lag grows every weekday between 09:00 and 11:00 and recovers by 14:00. Customers complain that password-reset emails arrive up to 40 minutes late. Propose a diagnosis and a design change that does not simply add more of the same consumers.
7. A team wants to replay three months of `orders` events to build a new analytics read model. List what must be true before they press the button.

---

## Diagnosis Lab

**Lab 1: Growing consumer lag**
- *Symptom:* The `invoice-generator` consumer group's lag grows from near zero to 4 million messages over a morning; invoices arrive hours late. Producers report normal rates.
- *Hypotheses:* (a) consumers slower than the input; (b) a poison message; (c) rebalance storms.
- *Evidence:* Lag rises on all 12 partitions at the same slope (graph A), ruling out (b), which would affect one partition. Rebalance logs show none during the morning, ruling out (c). Consumer throughput is flat at 900 messages per second while the input is 1,400. Per-message processing time rose from 8 ms to 13 ms after a deploy the previous evening; traces show a new synchronous call to a tax-rate service for every invoice, at 5 ms. With 12 partitions processed one message at a time: capacity ≈ 12 / 0.013 ≈ 920 messages per second, below the 1,400 input.
- *Root cause:* (a): a per-message remote call cut capacity below the input rate; parallelism was capped at 12 by the partition count.
- *Fix (now):* Cache tax rates in process with a short TTL (they change rarely), restoring 8 ms per message: capacity ≈ 12 / 0.008 = 1,500 per second. That is barely above the input, so the 4 million backlog would drain at about 100 per second net, taking 11 hours. So also raise concurrency within each partition: process messages for different keys in parallel (for example 8 workers per partition, each owning a subset of keys, keeping per-key order). Capacity rises to roughly 12,000 per second, limited now by the database, and the backlog drains within minutes. *Fix (lasting):* batch database writes, review the partition count against future input, and add a capacity test that compares consumer throughput with peak input before deploys.
- *Prevention:* A time-lag alert at 5 minutes; per-message latency tracked per consumer.

**Lab 2: Duplicate emails sent**
- *Symptom:* Some customers receive two or three identical order-confirmation emails, minutes apart, mostly on days with deploys.
- *Hypotheses:* (a) the producer publishes duplicates; (b) the consumer reprocesses messages after rebalances or restarts; (c) the email provider retries on its side.
- *Evidence:* The broker shows one `OrderConfirmed` record per order (same offset consumed twice), ruling out (a). Consumer logs show message `9b1c…` processed by consumer instance A at 14:02:10 and again by instance B at 14:02:40; between them, a rebalance triggered by the deploy. The consumer sends the email and then relies on auto-commit every 5 seconds; A sent the email, was stopped by the deploy before the next commit, and B resumed from the old committed offset. The provider's log shows two separate API calls with different request IDs, ruling out (c).
- *Root cause:* (b): at-least-once delivery with a non-idempotent external effect, made frequent by auto-commit and rebalances during deploys.
- *Fix:* Send the email with the provider's idempotency key set to the message ID (or record "confirmation sent for order 9001" in a table with a unique key, checked and inserted in the same transaction before the send, with status reconciliation for the unknown case); commit offsets after processing; graceful shutdown that finishes the current batch and commits before leaving the group (Chapter 24, Section 24.10); cooperative rebalancing.
- *Prevention:* A test that kills the consumer between send and commit and asserts one email.

**Lab 3: A message stuck in a retry loop**
- *Symptom:* Shipping labels for about one in twelve orders stopped being created at 10:00; by 14:00, those orders are four hours late. All other orders are fine.
- *Hypotheses:* (a) a hot key; (b) a poison message blocking one partition; (c) one slow consumer instance.
- *Evidence:* Lag is near zero on 11 of 12 partitions and 1.9 million on partition 3 (graph C). The consumer log for partition 3 repeats offset 88231 with "customer c_5521 not found", attempt 2,973. The customer was deleted by a GDPR erasure job at 09:59 while an order for them was in flight. The consumer retries every exception forever, in place, and never commits past the failing offset.
- *Root cause:* (b): a permanent failure treated as transient, with no retry limit or DLQ, blocking every later message on that partition.
- *Fix (now):* Move the offending message to a DLQ manually (or deploy a fix that classifies "entity not found" as permanent), commit past it, and let the partition drain; handle the erased customer's order according to the privacy process. *Fix (lasting):* failure classification, bounded in-place retries, delayed retry topics for transient errors, DLQ for permanent ones with alerts; ordering-aware handling for the affected order key.
- *Prevention:* Alert on per-partition lag skew and on repeated offsets.

**Lab 4: A lost event from a dual write**
- *Symptom:* A customer paid for order 7731, which shows as paid in the orders database, but it was never shipped and no confirmation email was sent. The warehouse has no record of it.
- *Hypotheses:* (a) the warehouse consumer failed and dead-lettered the event; (b) the event was never published; (c) the event was published and lost by the broker.
- *Evidence:* The DLQs contain nothing for 7731, and the warehouse consumer's inbox has no record, so (a) is ruled out. The broker's topic has no record with key 7731 in the relevant time range (searching by timestamp), and producer metrics show no failures. The outbox table has **no row** for order 7731, although every other paid order has one. The code path for payments confirmed by a webhook from the payment provider (a path added three weeks earlier) updates the order to `PAID`, commits, and then publishes `OrderPaid` directly to the broker; the application logs show a broker connection timeout at that moment, logged as a warning and ignored.
- *Root cause:* (b): a dual write on a new code path that bypassed the outbox; the broker was briefly unreachable after the commit.
- *Fix (now):* Find all paid orders without a published `OrderPaid` (a reconciliation query between `orders` and the outbox or a consumer's records) and publish them; ship order 7731. *Fix (lasting):* the webhook path writes its event to the outbox in the same transaction as the status change; a code-level rule (and a test) that services publish only through the outbox; a daily reconciliation job comparing business state with downstream records.
- *Prevention:* An architecture test that fails the build if code calls the broker client outside the relay.

---

## Solution Sketches

1. Synchronous: validate and create the account (and return a session). Asynchronous, reliably: in the same transaction, insert outbox rows (`UserRegistered`) or database jobs for the welcome email, CRM contact and avatar; workers process them idempotently (keyed by user ID) with retries and DLQs. The user sees the account immediately; the avatar shows a placeholder until generated; the email arrives shortly. Latency falls to the account creation alone.
2. Keying by event type sends all events of one type to one partition, so one customer's events are spread across partitions (no per-customer order) and the most common event type overloads one partition. Key by customer ID: all of a customer's events land in one partition, in order; load spreads across customers. If some event types need separate consumers, use separate topics, each keyed by customer.
3. The consumer wrote the ledger row, then crashed (or was stopped by the deploy, or its partition was revoked) before committing the offset; the new owner reprocessed the message. Designs: (a) transactional processing: store the consumed offset (or message ID) in PostgreSQL in the same transaction as the ledger row and seek to it on startup; (b) an inbox table or a unique constraint on the payment ID in the ledger, in the same transaction, skipping duplicates.
4. A dual write: the orders were committed, publishing failed during maintenance, and nothing retried. Fix: a transactional outbox (order and outbox row in one transaction) with a relay that publishes at least once and marks rows only after the broker confirms; if the relay crashes, unpublished rows remain and are published on restart; consumers deduplicate by the outbox row ID; alert on outbox age. Reconcile the 300 orders by republishing their events.
5. Order the steps from most to least compensatable: hold the hotel (cancellable), book the flight (refundable within 24 hours, or with a fee), and book the car last, because it cannot be cancelled: the car booking is the pivot, after which the saga only retries forward. Failure matrix: hotel fails → nothing to undo; flight fails → cancel hotel hold; car fails → cancel flight (refund or fee) and hotel; after the car is booked, remaining steps (confirmation, itinerary email) retry until success. Idempotency keys per step, step timeouts shorter than the hotel hold, and a human queue for compensations that fail.
6. A weekday-morning climb that recovers by afternoon suggests a capacity shortfall at peak (graph A pattern daily), and password resets share a queue with bulk notifications, so they wait behind marketing or digest emails sent in the morning. Diagnosis: lag per message type, input rates by type over the morning, consumer throughput. Change: separate topics or queues (and consumer groups) by priority, with password resets in a small, dedicated, always-fast lane; rate-limit bulk sends so they cannot fill the shared provider quota; then size each lane with Little's Law.
7. The events must still be within retention for the whole three months (or available from an archive); the new consumer must be idempotent by business keys and version-check updates; it must be a separate consumer group (so existing consumers are not reset) with side effects disabled (no emails or calls); every historical schema version must be readable (upcasting); the read model's database must handle the write rate of the replay (throttle it); and the team needs a way to verify the result (counts and checksums against the source of truth).

---

**END OF CHAPTER 25**
