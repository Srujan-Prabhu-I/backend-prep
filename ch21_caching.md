# Chapter 21: Caching

## Where We Are

Caches have appeared throughout this book, each time as part of something else. Chapter 2 showed CPU caches exploiting locality (Section 2.3); Chapter 3 taught page replacement, LRU, Clock and Belady's anomaly (Section 3.16); Chapter 4 the operating system's page cache (Section 4.3); Chapter 7 the HTTP cache with `Cache-Control`, `ETag` and `Vary` (Section 7.12); Chapter 10 the CDN at the edge (Section 10.1); Chapter 11 the rule that a cache key must include everything the value depends on (Section 11.1); Chapter 15 the database's buffer pool (Section 15.3) and Redis's persistence options (Section 15.8). Chapter 19 put sessions in Redis.

This chapter is the home of **application caching**: the caches a backend engineer designs, sizes and operates on purpose. It explains why caching works and how to compute what it buys, the layers where a response can be cached on its way to the user, the patterns for reading and writing through a cache, how eviction is implemented, and above all how caches go wrong. Most caching incidents are not "the cache was slow". They are stale data that a customer saw, a popular key expiring and a thousand requests hitting the database at once, a cache cluster restart that turned into a database outage. The chapter ends with the internals of Redis, the uses of a cache beyond reads, and sizing a multi-level cache.

The rule that runs through the whole chapter: **a cache is never the source of truth.** It is a copy that may be missing, stale or wrong, and every design must stay correct when it is.

**After this chapter you can:**
- explain why caching works (locality and skew) and compute effective latency and origin load from a hit ratio;
- place caches on a request path (browser, CDN, reverse proxy, in-process, distributed, database) and say what each is good at;
- choose among cache-aside, read-through, write-through, write-behind and refresh-ahead, and draw each as a sequence;
- implement LRU with a hash map and a doubly linked list in O(1), and compare LRU, LFU, FIFO, TTL and Redis's approximations;
- choose an invalidation strategy, explain the delete-versus-update race and its fixes, and state what a cache does not guarantee;
- prevent stampedes, penetration, avalanches and hot-key overload;
- explain Redis's single-threaded execution, data structures, persistence, replication and eviction policies;
- use a cache for rate-limit counters, sessions, idempotency results and locks, knowing each one's caveats;
- design and size a multi-level cache;
- diagnose low hit rates, stale data, hot-key latency and a cache outage that takes down the database.

**Interview questions this chapter prepares you for:**
1. What is a cache, and why does caching work? How do you compute the benefit of a cache?
2. Where can you cache on a request path, and what does each layer cost and offer?
3. Cache-aside versus read-through versus write-through versus write-behind: how do they differ?
4. How would you implement an LRU cache with O(1) operations?
5. LRU versus LFU: when is each better? How does Redis approximate LRU?
6. How do you invalidate a cache? TTLs, explicit deletes or versioned keys?
7. How do you keep a cache consistent with the database? What race makes it stale?
8. What is a cache stampede (thundering herd), and how do you prevent it?
9. What are cache penetration and cache avalanche, and how do you prevent them?
10. What do you do about a hot key?
11. Why is Redis single-threaded, and why is it still fast?
12. What happens to the database when the cache goes down?
13. Can you use Redis for rate limiting, sessions and locks? What are the caveats?
14. How do you size a cache?

**Scope.** Here: the caching cost model, cache layers, read and write patterns, eviction algorithms and their implementation, invalidation and consistency, failure modes, Redis and Memcached internals at concept level, non-read uses, multi-level caches and sizing, cache diagnosis, and a placement-round section on hardware cache organisation (Section 21.11). Recalled, not re-taught: CPU caches and locality (Chapter 2, Section 2.3), page replacement algorithms (Chapter 3, Section 3.16), HTTP caching headers (Chapter 7, Section 7.12), CDNs (Chapter 10, Section 10.1), cache keys (Chapter 11, Section 11.1), the buffer pool (Chapter 15, Section 15.3), bloom filters (Chapter 15, Section 15.6), Redis persistence (Chapter 15, Section 15.8), sessions (Chapter 19, Sections 19.3–19.4), idempotency keys (Chapter 13, Section 13.5). Elsewhere: replication and its lag are Chapter 22; consistent hashing, sharding and distributed locks with fencing are Chapter 23; rate-limiting algorithms are Chapter 24; change data capture for invalidation is Chapter 25.

---

## 21.1 What a Cache Is, and Why Caching Works

### Definitions

A **cache** is a store that keeps **copies** of data whose authoritative version lives elsewhere (the **origin** or **source of truth**: a database, a service, a file system), placed so that reading the copy is faster or cheaper than reading the original. **Caching** is the practice of serving requests from such copies.

- A **hit** is a request answered from the cache; a **miss** is one that must go to the origin. The **hit ratio** (hit rate) is hits ÷ (hits + misses); the **miss ratio** is 1 − hit ratio.
- **Staleness** is the difference between a cached copy and the current origin value; a copy is **stale** once the origin has changed and the copy has not.
- A **time to live** (TTL) is how long a cached entry may be served before it expires.
- The **working set** is the set of items accessed within a period of interest (Chapter 2, Section 2.2).
- **Skew** is unevenness in how often items are accessed: a few items take most of the requests.

### Why it exists

Some reads are expensive: a database query that joins five tables, a call to another service across a region, a rendered page, a computation over millions of rows. Many requests ask for the same few results again and again: the home page, the product that is on sale, the current user's profile. If each request recomputes the answer, the expensive resource (database CPU, a slow dependency, money for a paid API) caps the whole system's throughput. Serving repeated reads from a fast copy removes most of that load and most of that latency, at the price of staleness and of another component to run.

### Mechanism: why caching works

Caching works because access patterns are not uniform:

- **Temporal locality**: an item used now is likely to be used again soon (the same user reloads their cart; a trending article is read by thousands within an hour).
- **Spatial locality**: items near one used now are likely to be used soon (Chapter 2, Section 2.3); in application caches this appears as related data fetched together (a product and its price).
- **Skew**: popularity often follows a heavy-tailed distribution, commonly approximated by **Zipf's law**, in which the item of rank *k* is requested in proportion to 1/*k*. Under such a distribution a small cache holds a large share of the requests.

```text
 1,000,000 products, requests Zipf-distributed (exponent 1)        share of requests for the top N items
 top 100 items        (0.01% of the catalogue)                    ≈ 36%
 top 10,000 items     (1%)                                         ≈ 68%
 top 100,000 items    (10%)                                        ≈ 84%
 (share ≈ H(N) / H(1,000,000), where H(n) ≈ ln n + 0.577 is the harmonic number)
```

Caching 1% of the catalogue serves about two-thirds of the requests. Without skew (every item equally popular) the same cache would serve only 1%. Measuring how skewed your traffic is comes before deciding to cache.

### Concrete: the cost model

**Effective latency** with hit ratio *h*, cache latency *t_c* and origin latency *t_o* (a miss pays the cache check and then the origin):

```text
 effective latency = h × t_c + (1 − h) × (t_c + t_o) = t_c + (1 − h) × t_o

 t_c = 0.5 ms (Redis in the same zone), t_o = 20 ms (a database query)
 h = 0.50  →  0.5 + 0.50 × 20 = 10.5 ms
 h = 0.90  →  0.5 + 0.10 × 20 =  2.5 ms
 h = 0.99  →  0.5 + 0.01 × 20 =  0.7 ms
```

Going from 90% to 99% cuts latency by more than three times, because what matters is the **miss** ratio: 10% versus 1%.

**Origin load** falls with the miss ratio too:

```text
 10,000 reads/s arriving
 h = 0.90 → 1,000 reads/s reach the database
 h = 0.99 →   100 reads/s reach the database
```

Read the second line the other way round: a database sized for 100 reads per second **depends** on the cache keeping its hit ratio at 99%. If the hit ratio drops to 90% (a deploy that changes cache keys, a cold restart), database load rises tenfold. This dependency is the source of the worst caching incidents (Section 21.6, Diagnosis Lab 4).

**Staleness** is the third term of the cost model. With a TTL of *T* and no other invalidation, a reader can see a value up to *T* old. Every cached item needs an answer to: how stale may this be, for whom, and what happens if it is staler than that?

### Guarantees and non-guarantees

A cache guarantees nothing about its contents except what its own rules say: an entry is served at most until its TTL (if it has one), and a miss falls through to the origin. It does not guarantee that an entry is present (it may be evicted at any time), that it is current (unless invalidation is perfect, which it rarely is), or that it survives a restart. The origin remains the only source of truth.

### Failure and diagnosis

- **Caching that does not help.** A low hit ratio because access is not skewed, keys are too specific (a timestamp or request ID in the key), or TTLs are shorter than the time between reuses. Evidence: hit ratio metrics; key cardinality. Diagnosis Lab 1.
- **Caching that hides a capacity problem.** The origin can no longer survive without the cache. Evidence: origin load at a low hit ratio exceeds origin capacity. Fix: capacity planning for the cache-cold case (Section 21.6).

### Backend connection

Before adding a cache, measure: how often is each item read per change, how skewed is the access, what is the origin's cost per miss, and how stale may the answer be? Then compute effective latency and origin load at realistic hit ratios, including the cold case, and remember that adding a cache creates a new failure mode as well as removing load.

---

## 21.2 Cache Layers on a Request Path

### Definitions

A **cache layer** is a cache at one position between the user and the source of truth. Layers differ in **scope** (who shares the cached entries), **control** (who decides what is cached and when it is removed) and **cost** (latency saved and freshness lost).

- **Private** caches serve one user (the browser cache); **shared** caches serve many users (a CDN, a reverse proxy, Redis).
- An **in-process cache** (local cache) lives inside each application instance's memory. A **distributed cache** is a separate service shared by all instances (Redis, Memcached).

### Why it matters

Each layer removes a different cost. A CDN removes the network distance to the origin and absorbs traffic peaks; an in-process cache removes even the network hop to Redis; the database's own buffer pool removes disk reads. They also fail differently, and stacking them multiplies staleness: a value can be stale in Redis, then served stale again by a CDN for its own TTL.

### Concrete: all the layers on one request path

```text
 GET /products/17  (a product page for an anonymous user)

  ┌──────────────┐ 1. BROWSER cache (private)           Cache-Control: max-age=60 → no request at all
  │   browser    │    0 ms on a hit                     controlled by HTTP headers (Chapter 7, 7.12)
  └──────┬───────┘
         ▼
  ┌──────────────┐ 2. CDN edge (shared, near the user)   hit: ~10–30 ms instead of ~150 ms across
  │  CDN / PoP   │    s-maxage, cache key, purge API      continents (Chapter 10, Section 10.1)
  └──────┬───────┘
         ▼
  ┌──────────────┐ 3. REVERSE PROXY cache (shared,       microcaching (1–5 s) absorbs bursts
  │ reverse proxy│    at the origin's edge)              before they reach the application
  └──────┬───────┘
         ▼
  ┌──────────────┐ 4. IN-PROCESS cache (per instance)    ~100 ns–µs; no network; N copies, each
  │ app instance │    a map in the heap                   possibly different; lost on restart
  └──────┬───────┘
         ▼
  ┌──────────────┐ 5. DISTRIBUTED cache (shared)          ~0.2–1 ms within a zone; one copy for all
  │    Redis     │    product:17 → serialised object       instances; explicit invalidation
  └──────┬───────┘
         ▼
  ┌──────────────┐ 6. DATABASE buffer pool (inside the    a cached page turns a ~0.1 ms SSD read into
  │   database   │    database, Chapter 15, Section 15.3)  ~µs; automatic; not controlled by the app
  └──────────────┘
```

| Layer | Best for | Shared by | Invalidation control | Main risk |
|---|---|---|---|---|
| Browser | static assets, per-user pages for short times | one user | weak: only via TTL and validation (`ETag`, Chapter 7, Section 7.12); you cannot reach into a browser to delete | users seeing old versions; personalised data cached in shared machines' browsers |
| CDN | static assets, public pages and API responses, media | all users of a region | purge APIs (seconds to propagate), TTLs, versioned URLs | caching personalised or authenticated responses for everyone (Chapter 19, Section 19.2) |
| Reverse proxy | bursts of identical requests; microcaching | all users of the origin | TTL, purge, cache keys | same as CDN, at a smaller scale |
| In-process | very hot, small, rarely changing data (configuration, feature flags, exchange rates) | one instance | per instance: hard to invalidate everywhere at once | inconsistency between instances; memory and GC pressure (Chapter 2, Section 2.8) |
| Distributed (Redis, Memcached) | query results, computed objects, sessions, counters | all instances | explicit deletes, TTLs, versioned keys | the network hop; a cluster-wide outage; stale data after missed invalidations |
| Database buffer pool | everything the database reads | all queries | automatic | none for correctness; capacity when the working set exceeds memory |

The **first** layer that can answer a request saves the most: a browser hit costs nothing, a CDN hit saves the origin entirely. But layers closer to the user are also the hardest to invalidate, so they are best for content that is immutable (versioned asset URLs such as `app.3f9a1c.js` with a long `max-age` and `immutable`) or that tolerates staleness.

### Guarantees and non-guarantees

Each layer guarantees only its own rules (TTL, purge, eviction). Together they guarantee nothing about freshness: the staleness a user sees can be the sum of several layers' TTLs. No layer except the origin guarantees that data exists or is correct.

### Failure and diagnosis

- **Stale content after a fix is deployed.** The origin and Redis are correct, but users see old data. Evidence: response headers show `Age` and a CDN hit (`X-Cache: HIT`, or the standard `Cache-Status` header, RFC 9211); the browser shows "from disk cache". Fix: purge the CDN, use versioned URLs for assets, and keep `max-age` short for content that changes.
- **Personal data served to the wrong user.** A shared cache stored a response for an authenticated user. Evidence: the CDN logs a hit for a personalised path; responses without `Cache-Control: private`. Fix: `private, no-store` on authenticated responses; cache keys that include identity only when deliberately designed.

### Backend connection

Push immutable and public content as close to the user as possible, with versioned names, and keep mutable or personalised data in layers you can invalidate (Redis, in-process with short TTLs). For each layer, write down its key, its TTL and how to purge it: during an incident you will need all three.

---

## 21.3 Caching Patterns: How Reads and Writes Flow

### Definitions

A **caching pattern** defines who reads and writes the cache and the origin, and in what order:

- **Cache-aside** (lazy loading): the application checks the cache; on a miss it reads the origin and **writes the result into the cache** itself. On a write, it updates the origin and **invalidates** (deletes) the cached entry.
- **Read-through**: the application reads only from the cache; on a miss, **the cache** loads the value from the origin through a loader it has been configured with.
- **Write-through**: the application writes to the cache, and the cache **synchronously** writes to the origin before acknowledging; the cache always holds the latest written value.
- **Write-behind** (write-back): the application writes to the cache, which acknowledges at once and writes to the origin **later**, often in batches.
- **Write-around**: writes go directly to the origin and bypass the cache; the cache is filled only by reads.
- **Refresh-ahead**: the cache reloads popular entries **before** they expire, so that readers rarely see a miss.

### Why it exists

Every pattern answers the same two questions differently: who is responsible for filling the cache on a miss, and what happens to the cache when data changes. The answers decide latency on misses, the risk of stale reads, the risk of losing writes, and how much logic lives in the application versus in a caching library or product.

### Concrete: the sequences

**Cache-aside**, the most common pattern for application caches:

```text
 READ                                                    WRITE
 app ── GET product:17 ──► cache                         app ── UPDATE products … WHERE id = 17 ──► DB
 app ◄── nil (miss) ───────                              app ◄── ok (committed) ─────────────────────
 app ── SELECT … WHERE id = 17 ──► DB                    app ── DEL product:17 ──► cache
 app ◄── row ─────────────────────                       (the next read misses and reloads the new value)
 app ── SET product:17 <row> EX 300 ──► cache
 app returns the row
```

**Read-through**:

```text
 app ── get(product:17) ──► cache ── miss → loader(17) ── SELECT … ──► DB
                            cache ◄── row ───────────────────────────
                            cache stores it (TTL)
 app ◄── row ──────────────
```

**Write-through**:

```text
 app ── put(product:17, v) ──► cache ── UPDATE … ──► DB
                               cache ◄── ok ─────────
                               cache stores v
 app ◄── ok ───────────────────  (acknowledged only after BOTH the DB and the cache have v)
```

**Write-behind**:

```text
 app ── put(counter:17, v) ──► cache stores v, queues the change
 app ◄── ok ──────────────────  (acknowledged immediately: the DB does NOT have v yet)
            … 1 s later, batched …
                               cache ── UPDATE … (many changes in one batch) ──► DB
```

**Refresh-ahead**: for an entry with a 300-second TTL that is read often, a background task reloads it when, say, 80% of the TTL has passed, so it is replaced before it expires.

### Mechanism: comparing them

| Pattern | Miss latency | Freshness after a write | Write latency | Risk of lost writes | Where the logic lives |
|---|---|---|---|---|---|
| Cache-aside | origin + cache write | stale until the delete (race in Section 21.5) | origin only | none (origin first) | application |
| Read-through | origin (in the cache's loader) | as cache-aside, depending on how writes invalidate | n/a | none | cache library or product |
| Write-through | low after the first write | cache and origin updated together (but see failures below) | origin + cache | none if the origin write is part of the acknowledgement | cache library or product |
| Write-behind | low | the cache is ahead of the origin | **lowest** | **yes**: queued writes are lost if the cache fails before flushing | cache library or product |
| Write-around | origin on the first read after a write | the cache may still hold the old value until TTL or invalidation | origin only | none | application |
| Refresh-ahead | rarely a miss for hot keys | bounded by the refresh interval | n/a | none | cache or background job |

Choices that follow:

- **Cache-aside** is the default for application caches over a database: simple, the cache can fail without losing data, and only what is read gets cached.
- **Read-through** is cache-aside with the loading logic moved into a library (an in-process cache with a loader function, Section 21.9), which also makes request coalescing easier (Section 21.6).
- **Write-through** keeps hot data fresh in the cache, but writes data into the cache that may never be read, and its "both updated" promise holds only if the two writes cannot partially fail (they can: the DB commits and the cache write times out).
- **Write-behind** is for data where losing the last second of writes is acceptable (view counters, analytics, "last seen" times), or for systems where the cache is itself durable. It is dangerous for anything financial, and it makes the cache a **temporary source of truth**, contradicting this chapter's rule.

> **Framework lens:** Spring's cache abstraction implements cache-aside declaratively: `@Cacheable("products")` on a method checks the cache and calls the method on a miss, `@CacheEvict` deletes on writes, `@CachePut` writes through. Like `@Transactional`, these are proxies, so self-invocation bypasses them (Chapter 12, Section 12.5), and the order of the cache and transaction proxies decides whether an eviction happens before or after the commit (Section 21.5). Caffeine is a common in-process read-through cache in Java; `functools.lru_cache` is a simple in-process one in Python.

### Guarantees and non-guarantees

Cache-aside and read-through guarantee that a miss eventually reads the origin, and that cache failures never lose data. Write-through guarantees the cache reflects writes made through it, not writes made elsewhere (a batch job writing the database directly). Write-behind guarantees low write latency and nothing about durability until the flush. No pattern, by itself, guarantees that a reader never sees a stale value (Section 21.5).

### Failure and diagnosis

- **Stale values after writes made outside the application.** A migration or an admin tool updates the database directly; the cache-aside entries are never invalidated. Evidence: values in Redis that differ from the database for keys whose rows changed recently. Fix: TTLs as a backstop; invalidation from database change events (Chapter 25); route writes through one service.
- **Lost writes with write-behind.** Evidence: after a cache node restart, counters or values lower than before. Fix: use it only for loss-tolerant data, or persist the queue.
- **Cache filled with unread data.** Write-through on rarely read entities. Evidence: low hit ratio despite high memory use. Fix: cache-aside or write-around for those types.

### Backend connection

Default to cache-aside (or read-through in an in-process cache) with delete-on-write and a TTL. Use write-through only where hot data must be fresh in the cache and the double write is acceptable, and write-behind only for data you can afford to lose. Whatever the pattern, make sure every writer of the source of truth also invalidates, or accept the TTL as your staleness bound.

---

## 21.4 Eviction

### Definitions

**Eviction** is removing entries from a full cache to make room for new ones. An **eviction policy** decides which entry goes. **Expiry** is different: an entry whose TTL has passed is no longer valid, whether or not the cache is full.

- **LRU** (least recently used) evicts the entry whose last access is oldest.
- **LFU** (least frequently used) evicts the entry accessed least often, usually with **aging** so that formerly popular entries do not stay forever.
- **FIFO** (first in, first out) evicts the oldest-inserted entry, regardless of use.
- **Random** eviction picks a victim at random.
- **TTL-based** eviction (for example Redis's `volatile-ttl`) evicts the entries closest to expiry.
- **Admission** is deciding whether a new entry should enter the cache at all, which some modern caches do before evicting anything.

Chapter 3 (Section 3.16) taught these policies for memory pages, with Belady's optimal policy and the Clock approximation; Chapter 15 (Section 15.3) showed a database's buffer-pool variants. This section is about implementing and choosing them for application caches.

### Why it exists

A cache has a fixed amount of memory, and the set of items that might be cached is usually much larger. The eviction policy decides the hit ratio for a given memory size: a policy that keeps the items that will be reused soonest gets the most hits. No policy can know the future, so each one bets on a different signal from the past: recency (LRU), frequency (LFU), or nothing at all (FIFO, random).

### Mechanism: LRU in O(1), with a hash map and a doubly linked list

The classic interview implementation needs two operations in constant time: find an entry by key, and move an entry to the "most recently used" position (or remove the least recently used one). A hash map gives the first; a doubly linked list ordered by recency gives the second, because unlinking a node whose neighbours you know takes constant time.

```text
 map:   key → node
 list:  HEAD ⇄ [k3] ⇄ [k1] ⇄ [k7] ⇄ [k2] ⇄ TAIL
              most recent               least recent (the next victim)
```

```pseudo
class Node: key, value, prev, next

class LRUCache:
    capacity, map = HashMap()
    head = Node(sentinel), tail = Node(sentinel)       // sentinels avoid null checks at the ends
    head.next = tail; tail.prev = head

    function unlink(n):        n.prev.next = n.next; n.next.prev = n.prev            // O(1)
    function pushFront(n):     n.next = head.next; n.prev = head
                               head.next.prev = n; head.next = n                      // O(1)

    function get(key):
        n = map.get(key)                                 // O(1) average
        if n is null: return MISS
        unlink(n); pushFront(n)                          // now the most recent
        return n.value

    function put(key, value):
        n = map.get(key)
        if n is not null:
            n.value = value; unlink(n); pushFront(n); return
        if map.size() == capacity:
            victim = tail.prev                           // least recently used
            unlink(victim); map.remove(victim.key)
        n = Node(key, value); pushFront(n); map.put(key, n)
```

```text
 capacity 3: put(A) put(B) put(C)     list: C B A
 get(A)                               list: A C B      (A moved to the front)
 put(D)  → full: evict tail = B        list: D A C
 get(B)  → MISS
```

Both operations are O(1) on average (hash map lookups) and O(1) list surgery. In a concurrent cache, every `get` modifies the list, so a single lock around it becomes a bottleneck at high read rates; production caches avoid this with striped locks, sampled or buffered recency updates, or approximations such as Clock (Chapter 3, Section 3.16).

### Mechanism: LFU, and the weaknesses of each policy

A simple LFU keeps a counter per entry and evicts the smallest. An O(1) version keeps a map from frequency to a list of entries with that frequency, plus the current minimum frequency, so that an access moves an entry from list *f* to list *f + 1*.

| Policy | Bets on | Good for | Fails when |
|---|---|---|---|
| LRU | recently used items will be used again | most application workloads; bursts of interest | **a scan**: one pass over many cold items (a nightly export reading every product) pushes all hot items out (scan pollution) |
| LFU | frequently used items will stay popular | stable popularity (a catalogue's best sellers) | popularity **changes**: yesterday's top item keeps a high count and stays; new items are evicted before they build a count; needs aging |
| FIFO | nothing | simplicity, when entries are of similar value | hot items are evicted on schedule, even while in use |
| Random | nothing | very large caches where precise ordering costs too much; surprisingly close to LRU on many workloads | small caches with strong locality |
| TTL-ordered | entries about to expire are least valuable | caches where TTL reflects value | TTLs unrelated to access patterns |

Modern in-process caches combine ideas. Caffeine (Java) uses **W-TinyLFU**: a small LRU "window" admits new entries, and a compact frequency sketch decides whether a new entry deserves to replace an existing one in the main area, which resists scans and adapts to changing popularity. The general lesson is that admission (should this enter at all?) matters as much as eviction.

### Mechanism: how Redis evicts, approximately

Keeping an exact LRU list for millions of keys costs memory and CPU on every access. Redis approximates instead:

- Each key stores a small access-time stamp (or, under LFU policies, an 8-bit **logarithmic counter** that increments probabilistically, so a few bits can represent large counts, and decays over time).
- When memory exceeds `maxmemory`, Redis **samples** a few keys (`maxmemory-samples`, 5 by default at the time of writing), keeps the best candidates in a small pool across rounds, and evicts the one with the oldest access time (or lowest counter). More samples bring it closer to true LRU at more CPU cost.
- Expired keys are removed **lazily** (when accessed) and by a periodic **active** process that samples keys with TTLs, so expired keys can occupy memory for a while after their TTL.

Which keys are candidates depends on the **eviction policy** (Section 21.7): all keys, or only keys with a TTL.

### Guarantees and non-guarantees

An eviction policy guarantees that the cache stays within its memory limit (given a policy other than "no eviction") and that it evicts according to its rule (exactly, or approximately for Redis). It does not guarantee that a particular entry stays cached, that the hit ratio is good for your workload, or that expired entries disappear at the instant of expiry.

### Failure and diagnosis

- **Hit ratio collapses during a batch job.** Evidence: hit ratio drops while an export or report runs; evictions spike. Cause: scan pollution under LRU. Fix: bypass the cache for scans; a scan-resistant policy; a separate cache for batch work.
- **Memory full of expired keys** (Redis). Evidence: memory high while many keys are past their TTL; `expired_keys` low relative to keys with TTLs. Fix: usually self-correcting; check that the active expiry is not starved by a blocked server (Section 21.7).
- **Lock contention in an in-process LRU.** Evidence: profiling shows threads blocked on the cache's lock (Chapter 5, Section 5.14). Fix: a concurrent cache library.

### Backend connection

Use a mature cache library rather than a hand-written LRU in production (the hand-written one is for interviews). Pick the policy from the access pattern: LRU-like for bursts, frequency-aware for stable popularity, and scan resistance wherever batch jobs share the cache. Watch evictions as a metric: steady evictions mean the cache is too small for its working set.

---

## 21.5 Invalidation and Consistency

### Definitions

- **Invalidation** is removing or replacing a cached entry because its origin changed.
- **TTL-based expiry** relies on time alone: entries expire after a fixed period, and staleness is bounded by the TTL.
- **Explicit invalidation** deletes (or overwrites) the entry when the origin changes, from the code path that changes it or from a stream of change events.
- **Versioned keys** put a version in the key (`product:17:v42`); a change increments the version stored with the origin, so readers build a new key and old entries are simply never read again.
- **Cache consistency** is the degree to which readers of the cache see what readers of the origin would see. Caches provide at best **eventual** consistency with their origin: after writes stop, the cache eventually agrees.

### Why it exists

"There are only two hard things in computer science: cache invalidation and naming things," as Phil Karlton is often quoted. Invalidation is hard because the cache and the database are **two separate systems** with no shared transaction: a write commits in the database at one instant and reaches the cache at another, and concurrent readers can repopulate the cache in between. Every strategy is a trade-off between staleness, complexity and load on the origin.

### Mechanism: three strategies

| Strategy | Staleness | Complexity | Load on origin | Fits |
|---|---|---|---|---|
| TTL only | up to the TTL, always | lowest | a reload per key per TTL | data that tolerates bounded staleness (product listings, exchange rates for display) |
| Explicit delete on write (+ TTL backstop) | small, except in races (below) | every writer must invalidate every affected key | reloads only after changes | most application data |
| Versioned keys | none for readers that read the current version first | an extra read of the version (or the version travels with a parent object) | old versions linger until evicted | aggregates whose version is already known (an `updated_at` or version column), CDN assets |

Updating the cache on write ("set the new value") instead of deleting it looks more efficient but is more fragile: two concurrent writers can set their values in the opposite order to their database commits, leaving the older value cached indefinitely. Deleting is idempotent and order-insensitive, which is why cache-aside deletes.

### Concrete: the delete-versus-read race

Even delete-on-write can leave a stale value, because a **reader** can repopulate the cache with data it read **before** the write:

```text
 product 17: price 1999 in the DB; not in the cache (just expired)

 reader R (cache-aside)                          writer W
 GET product:17 → miss
 SELECT price … → 1999   (old value read)
        … R is paused (GC, slow network) …
                                                 UPDATE products SET price = 2499 … COMMIT
                                                 DEL product:17          (nothing to delete)
 SET product:17 1999 EX 300   ← writes the OLD value
 result: the cache holds 1999 for up to 5 minutes, although the DB says 2499,
         and no further write will come to fix it
```

The window is small (the reader must read before the commit and write after the delete), but at high traffic it happens.

### Mechanism: the fixes

1. **Delete after commit, never before.** Deleting before the database commit invites a reader to reload the old value in between. When the cache delete is in a transaction proxy or annotation, check the order (Section 21.3, Framework lens).
2. **Short TTLs as a backstop.** Every entry expires eventually, so a race leaves bounded staleness.
3. **Delayed double delete.** Delete after commit, then delete again after a short delay (longer than a typical read-then-set), catching most racing readers. It narrows the window; it does not close it.
4. **Conditional set with versions.** Store a version with the value, and let the reader's `SET` succeed only if the cached version is not newer (a compare-and-set, implemented in Redis with a small Lua script or a transaction), or use versioned keys, so that an old read cannot overwrite a newer value.
5. **Leases.** The cache gives a missing reader a lease token; a delete invalidates outstanding leases, so the racing reader's `SET` with an old lease is rejected. Facebook's memcache paper (2013) describes this design, which also limits stampedes (Section 21.6).
6. **Invalidation from the database's change stream.** Delete keys when the database's log shows a committed change (change data capture, Chapter 25), so that every writer, including scripts and batch jobs, triggers invalidation, in commit order.

### Mechanism: what a cache does not give you

```text
 BEGIN;
 UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- in the database transaction
 DEL account:1                                               -- NOT part of the transaction
 ROLLBACK;                                                    -- the DB undoes the update; the cache delete stays
                                                              (harmless here: a delete only causes a miss)
 … but the opposite order with SET would leave a value the DB never committed
```

- **No atomicity with the database.** A database transaction cannot include a cache write; either can succeed while the other fails.
- **No isolation.** Readers of the cache can see values from any moment, including values that never coexisted (read skew across cached objects, Chapter 17, Section 17.3).
- **No durability.** An entry can disappear at any time.
- **No ordering across keys.** Invalidations of two keys can arrive in any order.

Consequently: **never make a decision that must be correct from cached data.** Read stock, balances and permissions for a write from the database, inside the transaction (Chapter 17); use the cache to display them. And never write a value into the cache that the database has not committed.

### Concrete: read-your-writes with a cache

A user edits their display name and is redirected to their profile page, served from cache-aside. With delete after commit, their next read misses and reloads the new name. With a CDN or browser cache in front, or replica reads behind (Chapter 18, Section 18.16), they may still see the old name. Personal views right after a write should bypass shared caches or use versioned keys that change with the write.

### Guarantees and non-guarantees

TTLs guarantee bounded staleness, if nothing extends them. Explicit invalidation guarantees prompt removal on the paths that perform it. Versioned keys guarantee that readers of the current version never see an old one. None of them guarantees atomicity with the database, freshness across layers (CDN, browser), or correctness of decisions made from cached data.

### Failure and diagnosis

- **A stale value that persists.** Evidence: the cached value differs from the database, the key's TTL is long or absent, the last write to the row was a few minutes ago, and logs show a cache `SET` shortly after the write's delete. Cause: the race above. Diagnosis Lab 2. Fix: TTL backstop, delayed double delete or versioned sets.
- **Invalidation storms.** One write invalidates thousands of keys (every page that lists the product), and the reloads overload the origin. Fix: coarser keys, versioned namespaces (`catalogue:v91:*`) that change with one write, background recomputation.
- **Missed invalidations.** A new endpoint writes the table but does not delete keys. Evidence: stale data only after that endpoint is used. Fix: invalidation in one place (the repository) or from change events.

### Backend connection

Delete, do not update; delete after commit; keep a TTL on everything as the backstop; and make the cache's staleness explicit per data type ("product prices: up to 60 seconds; account balances: never cached for decisions"). For data many writers change, move invalidation out of application code and into the database's change stream (Chapter 25).

---

## 21.6 Failure Modes: Stampede, Penetration, Avalanche, Hot Keys, Cold Caches

### Definitions

- A **cache stampede** (dog-piling, or a **thundering herd** at the cache, introduced in Chapter 10, Section 10.1) is many concurrent requests missing on the same key at once, typically when a popular entry expires, and all of them recomputing it against the origin.
- **Cache penetration** is requests for keys that do not exist in the origin (and therefore never get cached), so every one reaches the origin.
- A **cache avalanche** is many keys expiring at about the same time, or a whole cache layer becoming unavailable, so that load shifts to the origin at once.
- A **hot key** is a single key that receives a large share of traffic, more than one cache node or one shard can serve.
- A **cold cache** is an empty or nearly empty cache, after a restart, a deploy that changes key formats, or a failover.
- **Request coalescing** (single-flight) is letting one request compute a missing value while concurrent requests for the same key wait for that result.

### Why it matters

A cache that protects a database also creates a dependency (Section 21.1): the database is sized for the hit ratio, not for the full traffic. Each failure mode is a way for the hit ratio to drop suddenly, either for one key (stampede, hot key) or for everything (avalanche, cold cache), and the database receives load it was never sized for.

### Concrete: a stampede, then the fix

The home page's "top products" entry takes 800 ms to compute and is read 2,000 times per second. Its TTL expires at t = 0.

```text
 WITHOUT protection
 t = 0 ms       entry expires
 t = 0–800 ms   every request misses: 2,000 req/s × 0.8 s ≈ 1,600 requests each run the 800 ms query
                the database, sized for a few of these at a time, saturates; each query now takes
                5 s instead of 0.8 s, so even more requests miss while waiting (the herd grows)
 t = 5 s        the first results arrive and are SET 1,600 times; meanwhile timeouts and errors

 WITH request coalescing + lock
 t = 0 ms       request #1 misses, acquires lock:top-products (SET lock:top-products <token> NX PX 5000)
 t = 0–800 ms   requests #2…#1,600 miss, see the lock, and either wait briefly and re-read the cache,
                or are served the previous (stale) value kept with a longer "grace" lifetime
 t = 800 ms     #1 SETs the new value, releases the lock; one query reached the database
```

### Mechanism: stampede defences

1. **Request coalescing in process.** In one instance, the first caller for a key starts the load and later callers for the same key wait on its future (Chapter 5, Section 5.12). Many in-process caches do this automatically for read-through loads. With N instances, at most N origin loads per key instead of thousands.
2. **A distributed lock per key.** Before recomputing, a request takes a short lock in the cache (`SET key NX PX`), and others wait or serve stale data. The lock needs an expiry, in case its holder dies, and a token, so that only the holder deletes it (locking caveats: Section 21.8 and Chapter 23).
3. **Serve stale while revalidating.** Store a soft expiry inside the value and a longer hard TTL. After the soft expiry, one request refreshes in the background while others keep receiving the slightly stale value. HTTP has the same idea as `stale-while-revalidate` (Chapter 7, Section 7.12).
4. **Probabilistic early refresh.** Each reader, as an entry approaches expiry, refreshes it early with a probability that rises as expiry nears (and with how long recomputation takes). One request almost always refreshes before expiry, so the entry never actually expires under load. The "XFetch" algorithm (Vattani and colleagues, 2015) formalises this.
5. **Refresh-ahead** for a known set of hot keys (Section 21.3).

### Mechanism: penetration

```text
 GET /products/999999999  (does not exist), 5,000 times per second, by a bot or a bug
 cache: miss → DB: no row → nothing to cache → next request: miss → DB …
 every request reaches the database
```

Defences:

- **Negative caching**: cache the absence (`product:999999999 → NOT_FOUND`) with a short TTL, so repeated misses are answered by the cache. Keep the TTL short, because the item may be created later.
- **A bloom filter of existing keys** (Chapter 15, Section 15.6) in front of the cache: "definitely not present" answers are rejected without touching the database. It must be updated when items are created, and it gives occasional false positives (which fall through to the normal path).
- **Input validation and rate limits**: reject impossible IDs early (wrong format, out of range), and rate-limit clients that generate many misses (Chapter 24).

### Mechanism: avalanche and cold caches

If a batch job loads 1 million keys with the same TTL of one hour, they all expire within the same second an hour later:

```text
 TTL = 3600 s exactly                      TTL = 3600 s + random(0, 600) s
 expiries                                  expiries
  ▲                                         ▲
  │            █                            │       ▁▂▃▃▃▃▃▃▃▃▃▂▁
  │            █                            │
  └────────────┴──── time                   └────────────────────── time
        all at 01:00:00                         spread over ten minutes
```

**TTL jitter** (a random extra few percent to tens of percent) spreads expiries. The other kind of avalanche is losing the cache itself (a cluster restart, a network partition, a deploy that changes the key format so that every lookup misses). Defences:

- **Capacity for the cold case**, or a deliberate degraded mode: know how much origin load a 0% hit ratio produces, and either size for it or have a plan.
- **Warm-up** before taking traffic: preload the hot keys, or shift traffic gradually (Chapter 10, Section 10.3, slow start).
- **Key format changes** rolled out with both versions readable, or with a gradual shift.
- **Load shedding and timeouts** at the origin (Chapter 24) so that the database protects itself instead of collapsing.
- **Replicated, highly available cache clusters**, so that one node's loss does not empty the cache.

### Mechanism: hot keys

A single key such as `config:flags` or the product in a flash sale can receive 100,000 reads per second. In a sharded cache, all of them go to one node (Chapter 23), whose CPU or network saturates, while other nodes idle. Defences:

- **An in-process cache in front** (L1, Section 21.9) with a short TTL: each instance reads the key from Redis at most once per second, not once per request.
- **Key replication**: store copies under several keys (`flags#0` … `flags#7`, which land on different shards) and read a random one; writes update all copies.
- **Read replicas** of the cache node, for read-heavy hot keys.
- **Splitting large values** (a hot key that is also big, such as a 2 MB JSON document, also saturates network bandwidth).

### Guarantees and non-guarantees

Coalescing and locking guarantee at most one (or one per instance) recomputation per key at a time; serve-stale guarantees continuous availability of a value, at the price of staleness. Negative caching and bloom filters guarantee that repeated lookups of absent keys do not all reach the origin. Jitter guarantees spread-out expiries. None of them guarantees protection against the loss of the whole cache: only origin capacity, load shedding and gradual warm-up do.

### Failure and diagnosis

- **Periodic latency spikes on one endpoint.** Evidence: spikes every TTL period; database slow-query logs show many identical queries at the same second. Cause: stampede on a hot key. Fix: coalescing, serve-stale, early refresh.
- **Database load from misses on non-existent IDs.** Evidence: many queries returning zero rows for IDs that never existed; cache miss rate high on a narrow set of patterns. Fix: negative caching, validation, rate limits.
- **Database overload after a cache restart or deploy.** Evidence: hit ratio drops to near zero, database CPU and connections saturate, pools exhaust (Chapter 18, Section 18.3). Diagnosis Lab 4.
- **One cache node at 100% CPU.** Evidence: per-node CPU and network, the node's busiest keys. Diagnosis Lab 3.

### Backend connection

Assume the cache will be cold at the worst moment and design for it: TTL jitter on every bulk-loaded key, coalescing for expensive recomputations, serve-stale for hot entries, negative caching with validation for lookups by ID, an L1 cache for hot keys, and an origin that sheds load instead of collapsing when the hit ratio falls.

---

## 21.7 Inside a Distributed Cache: Redis and Memcached

### Definitions

**Redis** is an in-memory data-structure server: clients send commands over TCP that operate on values of several types stored under keys. **Memcached** is a simpler in-memory key–value cache that stores opaque byte strings. Both are commonly used as distributed caches. (Valkey, an open-source fork of Redis, behaves the same way for the purposes of this chapter.)

- An **eviction policy** in Redis (`maxmemory-policy`) decides which keys may be evicted when memory reaches `maxmemory`, and how they are chosen (Section 21.4).
- **Replication** in Redis copies a primary's data to replicas asynchronously. **Redis Cluster** splits the key space into 16,384 **hash slots** spread over several primaries.

### Why it matters

A distributed cache is usually the busiest server in a backend: tens of thousands of commands per second, each expected to take well under a millisecond. Knowing how it executes commands explains both why it is fast and how one careless command can stall every client, and knowing its persistence, replication and eviction settings decides whether it can be trusted with sessions, counters or locks (Section 21.8).

### Mechanism: single-threaded command execution

Redis executes commands on **one thread**, in an event loop (Chapter 5, Section 5.13) driven by I/O multiplexing (`epoll`, Chapter 4, Section 4.5):

```text
 clients ──► sockets ──► event loop (ONE thread executes commands)
                           ├─ read a command from client A → execute → queue reply
                           ├─ read a command from client B → execute → queue reply
                           └─ timers: active expiry, background task checks
                         (Redis 6+ can use extra I/O threads for reading and writing sockets;
                          command execution itself stays on one thread)
```

Why one thread is fast: all data is in memory, most commands are O(1) or O(log N) and take microseconds, there are no locks to take between commands, and the cost is dominated by network round trips, which the event loop overlaps across thousands of connections. Why it is dangerous: **while one command runs, nothing else runs**. A command that takes 2 seconds blocks every client for 2 seconds.

| Command | Complexity | Risk |
|---|---|---|
| `GET`, `SET`, `INCR`, `HGET`, `SADD`, `EXPIRE` | O(1) | none |
| `ZADD`, `ZRANGE` with small ranges | O(log N) (+ range size) | none |
| `KEYS pattern` | O(N) over **all** keys | blocks the server on large databases: use `SCAN` (incremental cursor) instead |
| `SMEMBERS`, `HGETALL`, `LRANGE 0 -1` on huge collections | O(N) | blocks; returns huge replies; use incremental `SSCAN`, `HSCAN` |
| `DEL` of a key holding millions of elements | O(N) | blocks; use `UNLINK`, which frees memory in the background |
| `FLUSHALL` | O(N) | blocks unless the asynchronous form is used |
| Lua scripts (`EVAL`) | as written | run atomically, so a long script blocks everyone |

The same property makes every single command **atomic**: no other command interleaves with it. That is why `INCR` is a safe counter and `SET key value NX` is a safe "claim", without locks (Section 21.8). A sequence of commands is not atomic unless grouped with `MULTI`/`EXEC` (queued and executed together) or written as a Lua script.

### Mechanism: data structures

| Type | What it is | Typical use |
|---|---|---|
| String | bytes (up to 512 MB), with integer operations | cached objects (serialised JSON), counters, flags |
| Hash | a map of fields to values under one key | an object's fields (`HSET user:81 name Ana`), sessions with field-level updates (Chapter 19, Section 19.3) |
| List | a linked sequence, push and pop at both ends | simple queues, recent-items lists |
| Set | unordered unique members | tags, "users online", deduplication |
| Sorted set | members ordered by a score (implemented with a skip list plus a hash map) | leaderboards, time-ordered indexes, sliding-window rate limits |
| Stream | an append-only log of entries with IDs and consumer groups | lightweight event streams (brokers in Chapter 25) |
| HyperLogLog | a probabilistic distinct counter in about 12 KB | unique visitors, with about 1% error |
| Bitmap, geospatial | bit operations on strings; sorted sets keyed by location | feature flags per user ID, "stores near me" |

Small collections are stored in compact encodings (contiguous memory instead of hash tables or skip lists) and converted automatically when they grow, which is why many small hashes use less memory than many separate string keys.

### Mechanism: persistence, replication and eviction policies

**Persistence** was covered in Chapter 15 (Section 15.8): RDB snapshots by `fork` and copy-on-write, AOF logging with `appendfsync always`, `everysec` or `no`. For a pure cache, persistence is often off (the origin can rebuild everything); for sessions, rate limits or idempotency results, losing the data on restart has consequences, so choose deliberately.

**Replication** is asynchronous: the primary acknowledges a write before replicas receive it, so a failover can lose recent writes, and a replica read can return an older value (Chapter 22). Automatic failover is provided by Redis Sentinel or by Redis Cluster.

**Eviction policies** (`maxmemory-policy`):

| Policy | Evicts | Use for |
|---|---|---|
| `noeviction` (the default at the time of writing) | nothing: writes fail with an out-of-memory error when full | data that must not disappear silently (sessions, queues), with memory alerts |
| `allkeys-lru` / `allkeys-lfu` | any key, approximately least recently / least frequently used | a pure cache |
| `volatile-lru` / `volatile-lfu` | only keys **with a TTL** | mixed instances (but if no key has a TTL, nothing can be evicted) |
| `allkeys-random` / `volatile-random` | random keys | rarely |
| `volatile-ttl` | keys with the nearest expiry | when TTL reflects value |

The practical rule: do not mix data that may be evicted (cache entries) and data that must not be (sessions, locks, rate-limit state) on one instance under an `allkeys-*` policy. Run separate instances, or databases with different policies.

**Sharding**: Redis Cluster maps each key to one of 16,384 hash slots (CRC16 of the key, modulo 16,384) and assigns slots to primaries; clients learn the map and send each command to the right node. Commands over several keys work only if all keys are in one slot, which **hash tags** force (`{user:81}:cart` and `{user:81}:profile` hash only the part in braces). Consistent hashing and resharding are Chapter 23's subject.

### Mechanism: Memcached, for contrast

Memcached is multithreaded, stores only strings (values up to 1 MB by default at the time of writing), has no persistence and no replication, and is sharded by the **clients** (usually with consistent hashing, Chapter 23). It allocates memory in **slabs**: fixed-size chunk classes, each with its own LRU, which avoids fragmentation but can strand memory in a slab class that the workload no longer uses. It suits a pure, large, multithreaded cache of serialised objects; Redis suits cases that need data structures, atomic operations, persistence or replication.

### Guarantees and non-guarantees

Redis guarantees that each command (and each `MULTI`/`EXEC` block or Lua script) executes atomically, and, with persistence enabled, durability to the configured interval. It does not guarantee that acknowledged writes survive a failover (replication is asynchronous), that a slow command will not block every client, or that keys survive memory pressure under an eviction policy.

### Failure and diagnosis

- **Latency spikes for all clients at once.** Evidence: `SLOWLOG GET` shows a `KEYS *`, a large `HGETALL`, a big `DEL` or a long Lua script at that moment; latency monitoring reports the event. Fix: `SCAN`, `UNLINK`, smaller collections.
- **Spikes during persistence.** Evidence: the duration of the latest `fork` in `INFO`; spikes at snapshot times (Chapter 15, Section 15.8).
- **Write errors "OOM command not allowed when used memory > 'maxmemory'".** Evidence: `noeviction` and memory at the limit. Fix: more memory, TTLs on cache keys, or a policy that may evict, on the right instance.
- **CROSSSLOT errors in a cluster.** A multi-key command over keys in different slots. Fix: hash tags, or separate commands.

### Backend connection

Treat Redis as a single-threaded server whose latency is shared by every client: forbid O(N) commands over large keys in code review, keep values small, and watch the slow log. Choose persistence, replication and the eviction policy per instance from what the data means: evictable cache, or state that must not silently vanish.

---

## 21.8 Caching Beyond Reads

### Definitions

A distributed cache is often used to hold **shared, short-lived state** that is not a copy of anything: counters, session records, idempotency results, locks. In these uses the cache is, for the lifetime of the data, its **only** home, so the rule of Section 21.1 must be reconsidered for each use: what happens when this data disappears or is wrong?

### Why it matters

These uses exploit exactly what Redis is good at (atomic single commands, TTLs, sub-millisecond latency), and they are everywhere: rate limits on login (Chapter 20, Section 20.2), sessions (Chapter 19), idempotency results (Chapter 13, Section 13.5), "only one instance runs this job" (Chapter 11, Section 11.7). Each has a failure mode that differs from a read cache, because losing the data is no longer just a miss.

### Mechanism: four uses and their caveats

**Rate-limit counters.** A fixed-window limit of 100 requests per minute per client:

```text
 key = "rl:client42:" + floor(now / 60)
 n = INCR key                         ← atomic: concurrent requests never lose increments
 if n == 1: EXPIRE key 60             ← (or SET key 0 EX 60 NX first; or one Lua script for both)
 if n > 100: reject with 429 (Chapter 13, Section 13.13)
```

Caveat: if the cache is lost, limits reset (usually acceptable); if it is unreachable, decide whether to fail open (allow) or closed (reject). Better algorithms (sliding windows, token buckets) are Chapter 24's subject.

**Sessions.** Chapter 19 (Sections 19.3–19.4) stored sessions as keys with TTLs. Caveat: eviction or a restart logs users out, so use `noeviction` (or a separate instance), persistence, and replication.

**Idempotency results.** Store the response of a completed request under its idempotency key, so retries get the same answer:

```text
 SET idem:acct7:K91 "IN_PROGRESS" NX EX 86400   → OK: we own it; nil: another request has it
 … process …
 SET idem:acct7:K91 "<status + response body>" EX 86400
```

Caveat: if the key is evicted or lost and the client retries, the operation runs **twice**. Chapter 13 (Section 13.5) therefore used a database unique constraint as the guard for operations with real effects (payments); a cache can hold the stored response for speed, but should not be the only defence against duplicates.

**Locks.** `SET lock:report-2026-10 <random-token> NX PX 30000` takes a lock that expires after 30 seconds; the holder releases it only if the value is still its token (a check-and-delete in one Lua script), so it never releases someone else's lock. Caveats, all developed in Chapter 23:

- the lock expires while the holder is still working (a GC pause, a slow call), and a second holder starts: two holders at once;
- with asynchronous replication, a failover can lose a freshly taken lock;
- therefore a cache lock is fine for **efficiency** (avoid doing the same work twice, as in stampede protection, Section 21.6) but not for **correctness** (never two writers), which needs fencing tokens checked by the resource itself, or the database's own locks (Chapter 17, Section 17.8).

### Guarantees and non-guarantees

Single Redis commands guarantee atomic counters and claims on one node. They do not guarantee that the state survives eviction, restart or failover, that a lock is held exclusively for as long as the holder thinks, or that an idempotency key prevents a duplicate once it has been lost.

### Failure and diagnosis

- **Duplicate operations despite idempotency keys.** Evidence: keys missing from the cache after a restart or under memory pressure (`evicted_keys` rising). Fix: a database constraint as the real guard.
- **Two job instances running together.** Evidence: overlapping runs in logs, both "holding" the lock. Cause: lock expiry during a long pause. Fix: fencing (Chapter 23) or a database lock.
- **Rate limits not enforced after a failover.** Evidence: counters reset. Usually acceptable; document it.

### Backend connection

Use the cache for shared short-lived state when losing it is acceptable or recoverable, and say so explicitly for each use. Put a durable guard (a database constraint or lock) behind anything whose loss would cause a duplicate payment, a double booking or a broken invariant.

---

## 21.9 Multi-Level Caching and Memory Sizing

### Definitions

A **multi-level cache** places a small, fast cache (**L1**, usually in-process) in front of a larger, shared one (**L2**, usually Redis), in front of the origin. **Cache sizing** is choosing memory per level so that the hit ratio and cost are acceptable, from the working set and the access distribution.

### Why it exists

An in-process cache answers in nanoseconds and survives a Redis outage, but each instance holds its own copy (N copies of every entry, N chances to be stale). Redis gives one shared copy, but every hit costs a network round trip and every hot key concentrates on one node. Combining them puts the hottest keys in L1 (protecting Redis from hot keys, Section 21.6) and the long tail in L2.

### Mechanism: the read path and the invalidation problem

```text
 request → L1 (in-process, ~1 µs) ── hit → done
             │ miss
             ▼
           L2 (Redis, ~0.5 ms)     ── hit → put in L1 (short TTL) → done
             │ miss
             ▼
           origin (DB, ~20 ms)     → put in L2 (TTL) and L1 (short TTL) → done

 write → origin, then DELETE from L2, then tell every instance to drop the key from L1:
           publish "invalidate product:17" on a pub/sub channel; each instance subscribes
           and evicts locally (messages can be missed, so L1 TTLs stay SHORT, seconds)
```

Effective latency with two levels, extending Section 21.1 (h₁ = 0.80 in L1 at 0.001 ms, h₂ = 0.90 of the remaining in L2 at 0.5 ms, origin 20 ms):

```text
 t = h₁ × 0.001 + (1 − h₁) × [0.5 + (1 − h₂) × 20]
   = 0.8 × 0.001 + 0.2 × [0.5 + 0.1 × 20]
   = 0.0008 + 0.2 × 2.5  ≈  0.50 ms        (versus 2.5 ms with L2 alone at 90%)
 origin load: (1 − 0.8) × (1 − 0.9) = 2% of reads
```

### Concrete: sizing a cache

Assumptions: 2 million products; values of about 2 KB serialised; Zipf-like popularity; target hit ratio at L2 of 95%.

```text
 1. working set: which fraction of keys yields 95% of hits? (measure from logs; under the
    Zipf model of Section 21.1, the top 50% of items gives ≈ 95% of requests here)
    → about 1,000,000 keys
 2. memory per entry in Redis: value 2 KB + key and per-key overhead (~50–100 bytes)
    + allocator fragmentation (often 10–30%)  → ≈ 2.4 KB
 3. memory: 1,000,000 × 2.4 KB ≈ 2.4 GB for the data; plus replication buffers, a fork's
    copy-on-write headroom if persistence is on (Chapter 15, Section 15.8): plan ~4–5 GB
 4. L1 per instance: top 1,000 keys × 2 KB ≈ 2 MB of heap (plus object overhead in the language)
 5. verify: the hit ratio curve. Plot hit ratio against cache size from a trace (or from
    the cache's own statistics at several sizes); beyond the knee, extra memory buys little.
```

The hit-ratio curve has diminishing returns: under skewed access, doubling memory from the knee might raise the hit ratio from 95% to 97%. Whether that is worth it depends on the miss cost, and the right comparison is "memory cost versus origin load and latency saved", computed with the formulas of Section 21.1.

### Guarantees and non-guarantees

L1 guarantees the lowest latency for hot keys and continued service of those keys through an L2 outage. It does not guarantee consistency between instances (each L1 may hold a different version until its TTL or an invalidation message), and pub/sub invalidation does not guarantee delivery. Sizing calculations guarantee nothing until verified against real traffic.

### Failure and diagnosis

- **Instances disagree.** Two page loads show different values. Evidence: L1 entries with long TTLs; missed invalidation messages during a reconnect. Fix: short L1 TTLs (seconds), versioned keys.
- **Memory pressure in the application.** Evidence: GC pauses (Chapter 2, Section 2.8) and heap growth after adding an L1 without a size bound. Fix: a bounded cache with size-based eviction.
- **Evictions at L2 with spare memory.** Evidence: `evicted_keys` rising while `used_memory` is below what you expected. Cause: fragmentation or a lower `maxmemory` than planned. Fix: check `INFO memory` (fragmentation ratio), set `maxmemory` deliberately.

### Backend connection

Use an L1 with short TTLs for the hottest, smallest, least frequently changing keys, an L2 for the long tail, and size each from measured working sets and hit-ratio curves. Write down the staleness each level adds, because a value can be stale in L1 for seconds and in L2 for minutes, and the user sees the sum.

---

## 21.10 Diagnosis Toolkit: Reading a Cache

### Definition

The **cache diagnosis toolkit** is the set of metrics, commands and headers that show how a cache is behaving: hit and miss rates, evictions and expirations, memory, slow commands, per-key hotness and size, and, for HTTP caches, the cache status of each response.

### Why it matters

A cache problem usually appears as something else: database load, latency spikes, wrong data on a page. The cache's own counters distinguish the causes quickly: a falling hit ratio, rising evictions, one slow command, one hot key.

### Concrete: the signals

```text
 redis-cli INFO stats      keyspace_hits, keyspace_misses        → hit ratio = hits / (hits + misses)
                           evicted_keys                           → memory too small, or wrong policy
                           expired_keys                           → TTL activity
 redis-cli INFO memory     used_memory, maxmemory, mem_fragmentation_ratio
 redis-cli INFO clients    connected_clients, blocked_clients
 redis-cli SLOWLOG GET 10  commands slower than the configured threshold, with arguments
 redis-cli --bigkeys       largest keys per type (sampled with SCAN)
 redis-cli --hotkeys       most accessed keys (requires an LFU eviction policy)
 redis-cli --latency       round-trip latency from this client, continuously
 MONITOR                   prints every command: useful for seconds, dangerous under load
```

For HTTP caches, the response tells you which layer answered: `Age` (seconds since the origin produced it, Chapter 7, Section 7.12), vendor headers such as `X-Cache: HIT`, and the standard `Cache-Status` header (RFC 9211), for example `Cache-Status: ExampleCDN; hit; ttl=42`.

| Symptom | First signal to check | Typical cause |
|---|---|---|
| database load rising, no traffic change | hit ratio falling | key format change, shorter TTLs, evictions, cold cache |
| hit ratio fine, but stale data | TTLs, invalidation logs, `SET` after `DEL` | the race of Section 21.5, missed invalidations |
| periodic latency spikes | slow log, expiry times of hot keys | O(N) commands, stampedes, persistence forks |
| one node saturated | per-node CPU and network, `--hotkeys` | a hot key |
| writes failing | memory at `maxmemory` with `noeviction` | missing TTLs, undersized instance |

### Backend connection

Put hit ratio, evictions, memory, command latency and per-node load on the cache's dashboard, and the cache status of HTTP responses in access logs. Most cache incidents are diagnosed from those five numbers and the slow log.

---

## 21.11 Placement-Round Topics: Hardware Cache Organisation

> **Placement-round focus:** This section is revision material for computer-organisation questions in campus placement rounds and written tests, which ask cache-mapping and access-time numericals. It complements Chapter 2 (Section 2.3) and is not part of the main flow. What carries into backend work: the same ideas (hit ratios, effective access time, write-through versus write-back) govern every cache in this chapter.

### Mapping: where a memory block may be placed

Memory is divided into **blocks** the size of a cache line; the cache into **lines**. A **mapping** decides which lines a block may occupy:

- **Direct-mapped**: each block has exactly one possible line (`line = block number mod number of lines`). Simple and fast, but two hot blocks that map to the same line evict each other (**conflict misses**).
- **Fully associative**: a block may go in any line; every lookup compares the tag with all lines. No conflict misses, but expensive hardware.
- **k-way set-associative**: lines are grouped into **sets** of k; a block maps to one set (`set = block number mod number of sets`) and may use any line in it. The common compromise (L1 caches are often 8-way).

### Address breakdown, worked

32-bit addresses, a 32 KB cache, 64-byte lines, 4-way set-associative:

```text
 lines = 32 KB / 64 B = 512        sets = 512 / 4 = 128
 offset bits = log₂ 64  = 6        (which byte within the line)
 index bits  = log₂ 128 = 7        (which set)
 tag bits    = 32 − 7 − 6 = 19     (which block, stored and compared)
 direct-mapped instead: 512 sets → index 9 bits, tag 17 bits
 fully associative:     1 set    → index 0 bits, tag 26 bits
```

### Average memory access time

```text
 AMAT = hit time + miss rate × miss penalty
 two levels: AMAT = t_L1 + m_L1 × (t_L2 + m_L2 × t_mem)

 t_L1 = 1 ns, m_L1 = 5%;  t_L2 = 4 ns, m_L2 = 20% (local);  t_mem = 80 ns
 AMAT = 1 + 0.05 × (4 + 0.20 × 80) = 1 + 0.05 × 20 = 2 ns
```

This is the formula of Section 21.1 with hardware numbers.

### Write policies and miss types

- **Write-through** writes to the cache and to memory on every store; **write-back** writes only to the cache, marks the line **dirty**, and writes it to memory when it is evicted (the same trade-off as Section 21.3's write-through and write-behind).
- **Write-allocate** loads the block into the cache on a write miss; **no-write-allocate** writes directly to memory. Write-back usually pairs with write-allocate, write-through with no-write-allocate.
- The **three Cs** classify misses: **compulsory** (first access to a block), **capacity** (the working set does not fit), **conflict** (blocks competing for the same set in a direct-mapped or set-associative cache). A fourth, **coherence** misses, comes from other cores invalidating lines (Chapter 2, Section 2.3).

### Drill for this section

**C1. A 64 KB, 2-way set-associative cache with 32-byte lines serves 32-bit addresses. How many bits are tag, index and offset?**
A strong answer must mention:
- Lines = 64 KB / 32 B = 2,048; sets = 2,048 / 2 = 1,024.
- Offset = 5 bits, index = 10 bits, tag = 32 − 15 = 17 bits.
- Direct-mapped would use 11 index bits; fully associative none.
*Common weak answer:* computing the index from the number of lines instead of the number of sets.

**C2. Compute the AMAT for L1 hit time 2 ns, L1 miss rate 10%, L2 hit time 10 ns, L2 local miss rate 25%, memory 100 ns.**
A strong answer must mention:
- AMAT = 2 + 0.10 × (10 + 0.25 × 100) = 2 + 0.10 × 35 = 5.5 ns.
- The L2 miss rate is local (of L1 misses), not global.
*Common weak answer:* adding all latencies, or multiplying the global and local rates incorrectly.

**C3. Compare direct-mapped, fully associative and set-associative caches, and name the three Cs.**
A strong answer must mention:
- One possible line, any line, or any line within one set; the hardware cost and conflict behaviour of each.
- Compulsory, capacity and conflict misses (plus coherence on multicore).
- Higher associativity reduces conflict misses at the cost of comparison hardware and hit time.
*Common weak answer:* "fully associative is always best", ignoring cost and hit time.

---

## Chapter Summary

A **cache** is a faster copy of data whose truth lives elsewhere, and it works because access is **skewed** and **local**: under Zipf-like popularity, caching 1% of items can serve two-thirds of requests. Its value is computed from the **miss ratio**: effective latency is the cache's latency plus the miss ratio times the origin's, and origin load is the miss ratio times traffic, which means the origin becomes **dependent** on the hit ratio it was sized for. Every cached value also carries **staleness**, which must be bounded and stated.

Caches sit at every **layer** of a request path: browser, CDN, reverse proxy, in-process, distributed (Redis, Memcached) and the database's buffer pool. Layers near the user save the most and are the hardest to invalidate, so they suit immutable, versioned or public content; layers near the origin suit mutable data. Reads and writes flow through **cache-aside** (the default: read, miss, load, set; write, commit, delete), **read-through**, **write-through**, **write-behind** (fast, but a temporary source of truth that can lose writes) and **refresh-ahead**.

**Eviction** keeps a cache within its memory: **LRU** (a hash map plus a doubly linked list gives O(1) operations) bets on recency and suffers from scans; **LFU** bets on frequency and needs aging; Redis approximates both by sampling. **Invalidation** uses TTLs, explicit deletes or versioned keys; deleting beats updating; deleting **after commit** and keeping a TTL backstop handles most cases; the **read-before-write race** can still cache an old value, which delayed double deletes, versioned conditional sets, leases or change-data-capture invalidation address. A cache offers no atomicity, isolation, durability or ordering with the database, so decisions that must be correct are never made from cached data.

Caches fail as **stampedes** on hot keys (fixed by coalescing, locks, serve-stale and early refresh), **penetration** by absent keys (negative caching, bloom filters, validation), **avalanches** of simultaneous expiries or a lost cache (TTL jitter, warm-up, origin capacity and load shedding), and **hot keys** (L1 caches, key replication). **Redis** executes commands on one thread, which makes each command atomic and makes slow commands block everyone; it offers rich data structures, optional persistence, asynchronous replication and explicit **eviction policies** that must match what each instance stores. Used **beyond reads** (rate limits, sessions, idempotency results, locks), a cache becomes the only home of short-lived state, so each use needs a stated answer to "what if this disappears?", and correctness-critical guards belong in the database. **Multi-level** caches put hot keys in a short-TTL L1 and the tail in L2, and are **sized** from measured working sets and hit-ratio curves.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Cache (any) | Faster reads on hits; misses fall through to the origin | Presence, freshness or survival of any entry; truth |
| TTL | Entries are not served beyond their TTL (by that layer) | Freshness within the TTL; removal at the exact instant |
| Cache-aside with delete after commit | Most reads after a write see the new value; cache failure loses no data | Freshness under the read-before-write race |
| Write-through | Cache reflects writes made through it | Writes made elsewhere; atomicity of the two writes |
| Write-behind | Lowest write latency | Durability of queued writes |
| LRU / LFU / Redis sampling | Memory bound; eviction by (approximate) recency or frequency | A good hit ratio for every workload; scan resistance (LRU) |
| Versioned keys | Readers of the current version never see an old one | Memory reclaimed from old versions until eviction |
| Request coalescing / lock | One recomputation per key (per instance) at a time | Protection when the whole cache is lost |
| Negative caching / bloom filter | Absent keys do not all reach the origin | That a newly created item is seen before the negative TTL ends (negative caching) |
| TTL jitter | Spread-out expiries | Protection from a cache-wide outage |
| Redis single command | Atomic execution | Low latency when another client runs a slow command |
| Redis replication | Copies on replicas | Acknowledged writes surviving a failover |
| `noeviction` | Keys are never silently evicted | That writes succeed when memory is full |
| Cache lock with expiry | Efficient deduplication of work in normal operation | Mutual exclusion under pauses or failovers |
| L1 + L2 | Lowest latency for hot keys; resilience of L1 to L2 outages | Consistency between instances |

---

## Interview Drill

**Q1. What is a cache, and why does caching work? How do you compute its benefit?**
A strong answer must mention:
- A copy of data whose source of truth is elsewhere, faster or cheaper to read.
- Temporal and spatial locality, and skew (Zipf-like popularity): a small cache serves a large share of requests.
- Effective latency = t_cache + miss ratio × t_origin; origin load = miss ratio × traffic.
- The origin becomes dependent on the hit ratio; staleness must be bounded.
*Common weak answer:* "A cache makes things faster." It misses why (skew) and what it costs (staleness, dependency).

**Q2. Where can you cache on a request path, and what does each layer cost?**
A strong answer must mention:
- Browser, CDN, reverse proxy, in-process, distributed cache, database buffer pool.
- Closer to the user saves more but is harder to invalidate (browser and CDN: TTL, purge, versioned URLs).
- In-process: fastest, per-instance copies, inconsistency; distributed: shared, a network hop, explicit invalidation.
- Never cache personalised responses in shared layers.
*Common weak answer:* naming Redis only, or listing layers without saying what each is for.

**Q3. Cache-aside versus read-through versus write-through versus write-behind?**
A strong answer must mention:
- Cache-aside: the application loads on a miss and deletes on write.
- Read-through: the cache loads on a miss through a configured loader.
- Write-through: writes go through the cache to the origin synchronously.
- Write-behind: the cache acknowledges and writes to the origin later, so writes can be lost.
- Trade-offs in miss latency, freshness, write latency and loss risk; cache-aside as the default.
*Common weak answer:* treating write-behind as a free performance win without the durability risk.

**Q4. How would you implement an LRU cache with O(1) operations?**
A strong answer must mention:
- A hash map from key to node, and a doubly linked list ordered by recency.
- `get`: look up, unlink, move to the front; `put`: update and move, or insert at the front and evict the tail when full.
- Sentinel head and tail nodes; O(1) average for both operations.
- Concurrency: a lock per operation is a bottleneck; real caches use approximations or striping.
*Common weak answer:* a list scanned to find the least recently used entry, which is O(n).

**Q5. LRU versus LFU? How does Redis approximate LRU?**
A strong answer must mention:
- LRU bets on recency; suffers scan pollution. LFU bets on frequency; needs aging to forget old popularity.
- Modern hybrids (W-TinyLFU) add admission control.
- Redis samples a few keys (`maxmemory-samples`) and evicts the oldest-accessed among a candidate pool; its LFU uses a logarithmic 8-bit counter with decay.
*Common weak answer:* "LFU is always better because it counts usage." Not when popularity changes.

**Q6. How do you invalidate a cache?**
A strong answer must mention:
- TTL only (bounded staleness), explicit delete on write (plus TTL backstop), versioned keys.
- Delete rather than update, because concurrent updates can be applied out of order.
- Delete after the database commit.
- Invalidation from change data capture covers writers outside the application.
*Common weak answer:* "Update the cache when you update the database." Ordering and partial failures make this unreliable.

**Q7. How do you keep a cache consistent with the database? What race makes it stale?**
A strong answer must mention:
- The race: a reader misses, reads the old value, the writer commits and deletes, the reader sets the old value.
- Fixes: TTL backstop, delayed double delete, versioned conditional sets, leases, CDC-driven invalidation.
- No transactional guarantees between cache and database: no atomicity, isolation, durability or ordering.
- Correctness-critical reads come from the database inside the transaction.
*Common weak answer:* "Use a transaction around both." A database transaction cannot include a cache write.

**Q8. What is a cache stampede, and how do you prevent it?**
A strong answer must mention:
- A popular key expires and many concurrent requests recompute it against the origin, which slows and lets even more requests miss.
- Request coalescing, a per-key lock (with expiry and token), serve-stale while one request refreshes, probabilistic early refresh, refresh-ahead.
- Numbers: requests per second × recomputation time ≈ concurrent recomputations without protection.
*Common weak answer:* "Increase the TTL." It delays the stampede, it does not prevent it.

**Q9. What are cache penetration and avalanche, and how do you prevent them?**
A strong answer must mention:
- Penetration: requests for absent keys always miss; fix with negative caching (short TTL), bloom filters, input validation, rate limits.
- Avalanche: many keys expiring at once or the cache lost entirely; fix with TTL jitter, warm-up, gradual traffic shifts, origin capacity and load shedding, replicated caches.
*Common weak answer:* confusing penetration with stampede; one is absent keys, the other a single hot key expiring.

**Q10. What do you do about a hot key?**
A strong answer must mention:
- One key's traffic concentrates on one cache node or shard.
- An in-process L1 with a short TTL; replicate the key under several names across shards; read replicas.
- Split large hot values; find hot keys with per-node metrics and hot-key tools.
*Common weak answer:* "Add more cache nodes." The hot key still maps to one node.

**Q11. Why is Redis single-threaded, and why is it still fast?**
A strong answer must mention:
- Commands execute on one thread in an event loop with I/O multiplexing; extra I/O threads handle sockets in newer versions.
- In-memory data and O(1)/O(log N) commands take microseconds; no locks between commands.
- Each command is atomic; a slow command (KEYS, large HGETALL, big DEL, long Lua) blocks every client.
- Use `SCAN`, `UNLINK`, small values; watch the slow log.
*Common weak answer:* "Because it is written in C." Language is not the reason; the execution model and in-memory data are.

**Q12. What happens to the database when the cache goes down?**
A strong answer must mention:
- Load rises by the inverse of the miss ratio (from 1% to 100% of reads: 100×).
- Connection pools exhaust, latency rises, the database may collapse (Chapter 18, Section 18.3).
- Mitigations: replicated caches, L1 caches, warm-up, gradual traffic, load shedding and timeouts at the origin, capacity for the cold case.
*Common weak answer:* "Requests just get slower." They often fail, because the origin was sized for the hit ratio.

**Q13. Can you use Redis for rate limiting, sessions and locks? What are the caveats?**
A strong answer must mention:
- Atomic `INCR` with `EXPIRE` for counters; keys with TTLs for sessions; `SET NX` with a token and expiry for locks.
- Data can be evicted, lost on restart or failover: choose `noeviction`, persistence and replication per use.
- Idempotency and locks for correctness need a durable guard (database constraint, fencing, Chapter 23).
*Common weak answer:* "Yes, Redis is atomic, so it is safe." Atomic per command, not durable or exclusive across failures.

**Q14. How do you size a cache?**
A strong answer must mention:
- Measure the working set and access distribution; find how many keys yield the target hit ratio.
- Memory per entry including key, overhead and fragmentation; persistence and replication headroom.
- The hit-ratio curve has diminishing returns; compare memory cost with origin load and latency saved.
- L1 sizes per instance, bounded.
*Common weak answer:* "Make it as big as the database." Skew means a fraction is enough; beyond the knee, memory buys little.

---

## Scenario Exercises

1. A product catalogue API has a 70% hit ratio in Redis, and the database is at 80% CPU. Engineers propose doubling Redis memory. What data would you collect first, and what else could raise the hit ratio?
2. After a marketing email, the product in the email is read 50,000 times per second; one Redis node is at 100% CPU and p99 latency is 400 ms, while the other nodes are idle. Explain and fix, now and permanently.
3. A support agent changes a customer's shipping address; the customer's next three page loads show the old address, then the new one, then the old one again. The service uses an in-process cache (TTL 60 s) in front of Redis (TTL 10 min). Explain each observation and redesign.
4. A team caches "available stock" per product with a 30-second TTL and decides whether to accept orders from the cached value. Explain what can go wrong and redesign.
5. A bot requests `/users/{random-id}` 3,000 times per second for IDs that do not exist; the database is struggling although the cache hit ratio for real users is 98%. Explain and propose three defences.
6. Your Redis instance stores both cached query results and user sessions, with `allkeys-lru`. During a traffic peak, users report being logged out. Explain and fix.
7. You must migrate the cache key format from `product:{id}` to `catalogue:v2:product:{id}` across 40 instances during business hours. Plan it so that the database is never hit by a cold cache.

---

## Diagnosis Lab

**Lab 1: A cache that does not cache (low hit rate)**
- *Symptom:* A new Redis cache in front of the search endpoint shows a 4% hit ratio; database load is unchanged.
- *Hypotheses:* (a) access is not skewed; (b) keys are too specific; (c) entries expire before reuse.
- *Evidence:* Sampling keys with `SCAN` shows keys like `search:shoes:page1:1761302215123`: the key includes the request timestamp (added for "uniqueness"). Key cardinality equals the request count. Without the timestamp, a sample of the logs shows that the top 1,000 queries make up 61% of searches.
- *Root cause:* (b): every request created a new key, so no entry was ever reused.
- *Fix:* Build the key from the normalised query only (lower-cased terms, sorted filters, page, locale: everything the result depends on, nothing else, Chapter 11, Section 11.1); hit ratio rises to about 60%.
- *Prevention:* Review key definitions; track key cardinality versus request rate.

**Lab 2: A price that will not update (stale data)**
- *Symptom:* After a price change, a product page shows the old price for exactly ten minutes in about one case in a few hundred; the database is correct immediately.
- *Hypotheses:* (a) a CDN caching the page; (b) a missed invalidation; (c) the read-before-write race.
- *Evidence:* The page is not cached by the CDN (`Cache-Status` shows a miss and no `Age`). Application logs for an affected product show: 10:00:01.120 read miss on `product:17` and a database read of the old price by request R; 10:00:01.180 the price update commits and deletes `product:17`; 10:00:01.240 request R sets `product:17` with the old price, TTL 600 s.
- *Root cause:* (c): a reader that loaded the old value before the commit stored it after the delete.
- *Fix:* Store a version (`updated_at`) with the value and set only if not older (a small Lua compare-and-set), plus a delayed second delete one second after each update; shorter TTL (60 s) as the backstop.
- *Prevention:* Invalidation from the database change stream for price changes (Chapter 25); a monitor comparing a sample of cached prices with the database.

**Lab 3: One node melting (hot key)**
- *Symptom:* p99 latency of every cached endpoint jumps from 3 ms to 400 ms during a flash sale; one of six Redis Cluster primaries is at 100% CPU and its network interface is saturated.
- *Hypotheses:* (a) a slow command; (b) a hot key; (c) the node's host is degraded.
- *Evidence:* `SLOWLOG` shows nothing unusual. `redis-cli --hotkeys` (the cluster uses an LFU policy) and the client's per-key metrics show `product:sale-item` at 85,000 reads per second; its value is 180 KB (the full product with reviews), so the node sends about 15 GB/s worth of replies it cannot sustain, and every other key on that node waits behind it.
- *Root cause:* (b), amplified by a large value.
- *Fix (now):* Enable an in-process L1 for that key with a 1-second TTL (reads to Redis fall to about one per instance per second). *Fix (lasting):* Split the value (reviews loaded separately), keep large hot values in L1, replicate hot keys across slots when needed, and alert on per-node CPU and network skew.
- *Prevention:* Load tests of sale scenarios with realistic skew.

**Lab 4: The cache outage that took down the database**
- *Symptom:* A Redis maintenance restart at 02:00 is followed by a full outage: database CPU at 100%, connection pools exhausted, API errors for 25 minutes, although Redis was back within 40 seconds.
- *Hypotheses:* (a) the restart corrupted data; (b) the cold cache sent full traffic to the database, which collapsed and could not refill the cache; (c) an unrelated database problem.
- *Evidence:* The hit ratio fell from 98% to 0% and stayed below 30% for 20 minutes. Database queries per second rose from 400 to 20,000 (the full read rate); each query's latency rose from 5 ms to seconds; application pools of 10 connections per instance exhausted (Chapter 18, Section 18.3); requests timed out **before** writing their results into the cache, so the cache refilled very slowly; retries from clients added load (Chapter 24).
- *Root cause:* (b): the database was sized for a 98% hit ratio; a cold cache needed 50 times its capacity, and the overloaded database could not produce results fast enough for the cache to recover (a metastable failure, Chapter 11, Section 11.8).
- *Fix (now):* Shed load at the edge (serve a reduced page, reject non-essential endpoints), let the cache warm with limited traffic, then restore. *Fix (lasting):* replicated Redis with failover so that maintenance does not empty the cache; warm-up from a list of hot keys before taking traffic; request coalescing so that each key is loaded once; database statement timeouts and load shedding; retry budgets in clients.
- *Prevention:* A game day that restarts the cache under realistic load; capacity planning that includes the cold-cache case.

---

## Solution Sketches

1. Collect: the hit-ratio curve (would more memory help, or are we past the knee?), evictions (if `evicted_keys` is low, memory is not the limit), key cardinality and TTLs (short TTLs or overly specific keys cause misses that memory cannot fix), the access distribution, and which queries dominate database CPU. Other levers: fix keys (normalise), lengthen TTLs where staleness allows, cache at a coarser granularity (a whole page fragment instead of many small objects), add an L1 for hot keys, use refresh-ahead for expensive popular entries, and fix the expensive queries themselves (Chapter 16).
2. A hot key on one node (Section 21.6): all reads for that product hash to one slot. Now: an in-process L1 with a short TTL for that key, or a CDN cache for the product page if public. Permanently: L1 for keys above a read-rate threshold, hot-key replication across slots, smaller values, per-node skew alerts, and load tests of campaigns.
3. First load: the instance's L1 still had the old address (60 s TTL) or L2 had not been invalidated yet. Second: another instance whose L1 had expired read the new value from Redis. Third: a third instance with an old L1 entry. Instances disagree because each L1 is independent. Redesign: delete from L2 after commit and broadcast an L1 invalidation; L1 TTL of a few seconds for user-specific data, or no L1 for it; for the user's own view right after a write, bypass caches (read-your-writes, Section 21.5).
4. Decisions made from cached data: two orders can both see "stock 1" in the cache and both be accepted, or orders can be rejected for stock that has been replenished. A cache gives no isolation with the database. Redesign: display the cached value, but decide inside the database transaction with a conditional update (`UPDATE … WHERE stock >= $1`, Chapter 17, Section 17.11); optionally keep a cached "probably available" flag to fail fast for products known to be sold out.
5. Cache penetration: absent IDs are never cached, so every request reaches the database. Defences: validate the ID format and reject impossible ones; negative caching of "not found" with a short TTL; a bloom filter of existing user IDs; rate-limit or block the client generating mostly misses (Chapter 24); and require authentication for the endpoint if it should not be public.
6. Under `allkeys-lru`, sessions are eviction candidates like cache entries; at peak, memory pressure evicts the least recently used keys, which include sessions of users who paused, logging them out. Fix: move sessions to a separate Redis instance with `noeviction`, persistence and replication (Chapter 19); keep `allkeys-lru` only for the pure cache; alert on memory and evictions.
7. Deploy code that reads the new key format and, on a miss, falls back to the old key (copying the value to the new key) while writing and invalidating both formats; roll it out gradually across the 40 instances; optionally pre-warm the new keys for the hot set from the old ones with a background job that throttles itself; once the new keys' hit ratio matches the old one, remove the fallback and let the old keys expire. Watch hit ratio and database load at each step, with a rollback plan.

---

**END OF CHAPTER 21**
