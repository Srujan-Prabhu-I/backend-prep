# Chapter 16: Indexing and Query Execution

## Where We Are

Chapter 14 defined what a query **means**: the rows it returns, regardless of how they are found. Chapter 15 showed where those rows physically live: in 8 KB or 16 KB pages, cached in a buffer pool, read from devices whose latency is measured in microseconds or milliseconds. This chapter connects the two. Between the SQL you write and the pages on disk sits the **query engine**. It decides, for every query, which pages to read and in what order, and that decision is the difference between a query that takes 0.4 milliseconds and one that takes 40 seconds.

The chapter starts with **indexes**, the structures that let a database find rows without reading every page: the B+tree in depth, clustered and secondary indexes, composite, covering, partial and expression indexes, the other index types, and what indexes cost. Then it follows a query through the engine: parsing, planning, the **optimiser** and its statistics, scan types, join algorithms, and finally how to read `EXPLAIN` output, the single most useful diagnostic skill for a backend engineer working with a relational database. It ends with the query patterns that defeat indexes, table partitioning, and four diagnosis labs.

**After this chapter you can:**
- explain why a full scan is linear and an index lookup logarithmic, with numbers;
- draw a B+tree, trace a search, a range scan, an insert with a split and a delete, and compute its height from its fan-out;
- explain clustered versus secondary indexes in InnoDB and PostgreSQL, and when an index-only scan really avoids the table;
- design composite indexes with the leftmost-prefix and equality-then-range rules;
- choose covering, partial, expression, unique and specialised indexes (hash, GIN, GiST, BRIN, bitmap);
- quantify the write and storage cost of indexes, and find unused or bloated ones;
- trace a query through parse, rewrite, plan and execute, and distinguish logical from physical plans;
- explain how the optimiser uses statistics, selectivity and a cost model, and why stale statistics produce bad plans;
- choose between sequential, index, index-only and bitmap scans, and between nested-loop, hash and merge joins;
- read `EXPLAIN ANALYZE` output and find the node that is slow and why;
- fix the common query anti-patterns, and partition large tables with pruning.

**Interview questions this chapter prepares you for:**
1. How does a B+tree index work? Why a B+tree and not a binary tree or a hash table?
2. How many levels does an index on a billion rows have?
3. Why not index every column?
4. Clustered versus non-clustered (secondary) indexes: what is the difference? What happens on a secondary-index lookup in InnoDB and in PostgreSQL?
5. How should you order the columns of a composite index?
6. What is a covering index? What is an index-only scan?
7. What are partial and expression indexes for?
8. When would you use a GIN, GiST, BRIN or hash index?
9. How does the query planner choose a plan? What are selectivity and cardinality?
10. Why did a query suddenly become slow without any code change?
11. Nested-loop versus hash versus merge join: when does each win?
12. How do you read `EXPLAIN` and `EXPLAIN ANALYZE`?
13. Why does `WHERE lower(email) = …` or `LIKE '%term'` not use an index?
14. What is table partitioning, and what is partition pruning?

**Scope.** Here: index structures and design, the query pipeline, the optimiser, scan and join algorithms, `EXPLAIN`, anti-patterns, partitioning within one database, and a placement-round section on textbook indexing and hashing (Section 16.15). Recalled, not re-taught: pages, the buffer pool and storage engines (Chapter 15), SQL semantics and logical evaluation order (Chapter 14), relational algebra (Chapter 14, Section 14.14). Elsewhere: locks taken by queries and index-related locking are Chapter 17; prepared statements, ORMs and the N+1 problem in application code are Chapter 18; caching query results is Chapter 21; splitting data across machines (sharding) is Chapter 23. Examples use PostgreSQL's `EXPLAIN` format, with MySQL equivalents where they differ.

---

## 16.1 Why Indexes Exist

### Definition

An **index** is a separate data structure, maintained by the database alongside a table, that maps values of one or more columns (the **index key**) to the locations of the rows that contain them, so that rows can be found without reading the whole table.

### Why it exists

Without an index, the only way to find the orders of customer 81 is a **full table scan** (sequential scan): read every page of the table and test every row. The cost grows linearly with the table.

Take an `orders` table with 10 million rows of about 64 bytes, which at about 120 rows per 8 KB page (Chapter 15, Section 15.2) occupies about 83,000 pages, 650 MB:

```text
 SELECT * FROM orders WHERE customer_id = 81;          -- 104 matching rows

 full scan:     read 83,334 pages, test 10,000,000 rows to return 104
                at about 1 GB/s from an SSD or memory: on the order of a second, plus CPU for every row
 with an index: read 3 or 4 index pages to find the 104 entries, then the pages holding those rows
                on the order of a millisecond or less
```

The full scan reads the same 650 MB whether the table has 104 matching rows or none, and doubles when the table doubles. An index lookup costs about one more page read each time the table grows a few hundred times, because a B+tree's height grows logarithmically (Section 16.2). That difference, linear versus logarithmic, is why indexes exist.

### Mechanism: the idea, before the structure

An index is the same idea as the index at the back of a book: entries sorted by key, each pointing to where the full information lives. Sorting is what makes it fast. In a sorted structure you can find any key by repeatedly halving the search space (binary search), and all entries in a range are next to each other.

```text
 table (heap, in insertion order)            index on customer_id (sorted)
 page 0:  (101, cust 17) (102, cust 81) …     customer_id │ row location
 page 1:  (221, cust 42) (222, cust 9)  …     9           │ (1, 2)
 …                                            17          │ (0, 1)
 page 83,333: …                               42          │ (1, 1)
                                              81          │ (0, 2), (412, 7), (90,331, 3) …
                                              …
```

A sorted array would support fast lookups, but inserting into the middle of a sorted array of 10 million entries means shifting millions of them. A balanced search tree supports fast lookups **and** fast inserts. On disk, the tree must also minimise the number of pages read, which is why databases use a tree with very wide nodes, the B+tree.

### Guarantees and non-guarantees

An index guarantees that rows can be found by its key in time logarithmic in the table size, and that it always reflects the table's committed contents (the database updates it in the same transaction as the table). It does not guarantee that the database will use it (the optimiser decides, Section 16.9), that it helps queries on other columns or on expressions of its column (Section 16.13), or that it is free: every index slows every write (Section 16.7).

### Failure and diagnosis

- **A query that was fast becomes slow as the table grows.** Evidence: its time grows in proportion to table size; `EXPLAIN` shows a sequential scan. Cause: no suitable index; the scan was cheap while the table was small.
- **A full scan where an index exists.** Evidence: `EXPLAIN` shows `Seq Scan` despite an index on the column. Causes: the query cannot use the index (Section 16.13), or the optimiser estimates that the scan is cheaper because many rows match (Section 16.9), which may be correct.

### Backend connection

Every query in a request path that filters, joins or sorts on a large table needs an index that matches it, or its latency will grow with the data. Design indexes from the queries your application actually runs, not from the columns your tables happen to have.

---

## 16.2 The B+tree

### Definitions

A **B+tree** is a balanced search tree designed for storage in pages:

- Each **node** is one page. A node holds many keys, sorted.
- **Internal nodes** hold keys and pointers to child nodes. The keys separate the children's ranges.
- **Leaf nodes** hold the actual index entries: each key with the location of its row (or, in a clustered index, the row itself, Section 16.3).
- **Leaves are linked** to their neighbours, so that a range can be read by walking along the leaves.
- The tree is **balanced**: every leaf is at the same depth.

The **fan-out** is the number of children an internal node can have, set by how many keys fit in a page. The **height** is the number of levels from the root to the leaves.

### Why it exists

A binary search tree has two children per node, so a tree over 10 million keys is about 24 levels deep. If each level were a page read from storage, a lookup would need 24 random reads. A B+tree puts hundreds of keys in each node, so each page read narrows the search by a factor of hundreds rather than two, and the same 10 million keys need only 3 levels. A hash table finds a single key in one step but cannot answer ranges, prefixes or sorted order; the B+tree does all of these, which is why it is the default index type in almost every relational database.

### Mechanism: the shape, a search and a range scan

A small B+tree, with at most four keys per node to keep the picture readable (real nodes hold hundreds):

```text
                                     ┌───────────┐
 root (level 1)                      │  30 │  60 │
                                     └──┬───┬───┬┘
                       keys < 30 ┌──────┘   │   └──────┐ keys ≥ 60
                                 ▼          ▼          ▼
 internal (level 2)      ┌────────────┐ ┌─────────┐ ┌──────────────┐
                         │ 10 │ 20    │ │ 40 │ 50 │ │ 70 │ 80 │ 90 │
                         └┬───┬────┬──┘ └┬───┬───┬┘ └┬────┬────┬───┬┘
                          ▼   ▼    ▼     ▼   ▼   ▼   ▼    ▼    ▼   ▼
 leaves (level 3)      [1 5][10 15][20 25][30 35][40 45][50 55][60 65][70 75][80 85][90 95]
                         ──►    ──►    ──►    ──►    ──►    ──►    ──►    ──►    ──►      (sibling links)
                       each leaf entry: key → row location, e.g. 45 → (page 512, slot 7)
```

**Search for key 45:**

```text
 1. root:      45 ≥ 30 and < 60 → follow the middle pointer
 2. internal:  45 ≥ 40 and < 50 → follow the pointer between 40 and 50
 3. leaf:      [40 45] → found 45 → row location (page 512, slot 7)
 3 page reads, whatever the size of the key range below
```

**Range scan for 35 ≤ key ≤ 65:**

```text
 1. search for 35 as above → leaf [30 35]; start at 35
 2. read along the leaf links:  35 → [40 45] → [50 55] → [60 65] → stop at 65
 descent once, then sequential steps along the leaves, in key order
```

The leaf links are what make the B+tree good at ranges, `ORDER BY` and prefix searches (`LIKE 'abc%'`): after one descent, matching entries come out already sorted.

### Mechanism: insert, and a node split

Inserting a key means searching for the leaf where it belongs and adding it there in order. If the leaf is full, it **splits**:

```text
 insert 52 into leaf [50 55] (room for 4): → [50 52 55]                     no split needed

 now insert 51, then 53 (the leaf can hold only 4 keys):
   [50 51 52 55] → inserting 53 overflows: [50 51 52 53 55]

 SPLIT:
   1. allocate a new leaf page
   2. move the upper half to it:        left [50 51 52]   right [53 55]
   3. fix the sibling links:            … [40 45] ⇄ [50 51 52] ⇄ [53 55] ⇄ [60 65] …
   4. insert a separator key for the new page into the parent:
        parent [40 50] → [40 50 53]      (keys ≥ 53 now go to the new leaf)
   5. if the parent overflows, split it too and push a separator into ITS parent;
      if the root splits, create a new root above it: the tree grows one level taller, at the top
```

Because the tree only ever grows at the root, every leaf stays at the same depth: the tree is always balanced, without any rebalancing rotations.

### Mechanism: delete

Deleting a key removes its entry from its leaf. Textbook B+trees then **merge** or **redistribute** underfull nodes to keep every node at least half full (Section 16.15). Real engines are lazier. PostgreSQL removes index entries during VACUUM and deletes a leaf page only when it becomes completely empty; it does not merge half-empty pages. InnoDB merges pages when they fall below a fill threshold. In both, a table after mass deletions can carry indexes that are larger than their live contents require (index bloat, Section 16.7).

### Mechanism: why the order of inserted keys matters

Where new keys land decides how often pages split and how full they stay:

```text
 increasing keys (identity column, timestamp, time-ordered UUIDv7):
   every insert goes to the RIGHTMOST leaf
   splits leave the left page full and start a fresh right page
   → leaves nearly 100% full (subject to fillfactor), one hot page in cache, few page writes

 random keys (UUIDv4):
   each insert lands in a random leaf anywhere in the tree
   splits happen everywhere and leave both halves about half full; pages average roughly two-thirds full
   → a larger index, every leaf page a candidate for the next insert, so the whole index must be
     cached or inserts cause random reads; many more dirty pages to write (Chapter 15, Section 15.3)
```

This is the reason, promised in Chapter 14 (Section 14.2), that random UUIDs hurt insert performance on large tables, and time-ordered identifiers do not. For a table of a few thousand rows it does not matter; for a billion rows whose index does not fit in memory it can decide whether inserts keep up.

### Concrete: fan-out and height arithmetic

In PostgreSQL, an index entry on a `bigint` column takes about 20 bytes (an 8-byte key, a small tuple header containing the 6-byte row location, and a 4-byte slot). An 8 KB page therefore holds about **400** entries.

```text
 height 1: the root alone                       400 entries
 height 2: root → 400 leaves                    400 × 400        = 160,000 entries
 height 3: root → 400 → 160,000 leaves          400³             = 64,000,000 entries
 height 4:                                      400⁴             = 25,600,000,000 entries
```

So an index on 10 million or 60 million rows has three levels, and an index on 25 billion rows has four. A lookup reads one page per level. The root and the second level together are only 401 pages, about 3 MB, so they are almost always in the buffer pool; a lookup in a 64-million-row index typically costs **one** read from storage, for the leaf, plus the read of the row itself.

InnoDB's 16 KB pages hold more entries (several hundred to about a thousand for a `bigint` key, depending on overheads), so its trees are even flatter. Wider keys reduce fan-out: a 100-byte text key fits about 70 entries per 8 KB page, and the same 64 million entries need five levels instead of three (70⁴ ≈ 24 million is too few; 70⁵ ≈ 1.7 billion is enough). Narrow keys keep indexes shallow.

### Guarantees and non-guarantees

A B+tree guarantees a lookup in a number of page reads equal to its height (logarithmic in the number of entries), range scans in key order after one descent, and balanced depth under any insert order. It does not guarantee compact storage under random inserts or after deletes, cheap inserts when the tree does not fit in memory, or usefulness for queries that do not constrain its leading key (Section 16.4).

### Failure and diagnosis

- **Insert throughput falls as a table grows.** Evidence: rising read I/O during inserts; the index larger than memory; a random-key primary key or index (UUIDv4). Fix: time-ordered keys, or fewer random-key indexes.
- **An index much larger than expected.** Evidence: index size versus `rows × entry size`; heavy deletes or random inserts. Fix: rebuild (Section 16.7).

### Backend connection

The B+tree is why indexed lookups stay fast as data grows by orders of magnitude: three or four levels reach billions of rows. It rewards narrow keys and keys that arrive in increasing order, and punishes wide random keys on large tables. Choose primary keys with both in mind.

---

## 16.3 Clustered and Secondary Indexes; Index-Only Scans

### Definitions

A **clustered index** (index-organised table) stores the table's rows themselves in the leaves of a B+tree, ordered by the index key. A table can have at most one, because the rows can be stored in only one order. InnoDB always stores tables this way, clustered on the primary key.

A **secondary index** (non-clustered index) is any other index: its leaves hold the key and a **reference** to the row. In PostgreSQL every index is secondary, and the reference is the row's TID `(page, slot)` in the heap (Chapter 15, Section 15.2). In InnoDB, a secondary index's reference is the row's **primary-key value**.

A **covering index** for a query contains every column the query needs, so the query can be answered from the index alone. Reading only the index is an **index-only scan**.

### Why it matters

The choice between clustered and heap storage changes the cost of every lookup path. In InnoDB, a primary-key lookup is one tree descent that ends at the row itself, while a secondary-index lookup is **two** descents. In PostgreSQL every index lookup is a descent plus a heap page read, and an index-only scan works only when PostgreSQL can be sure the rows are visible without looking at the heap. These differences explain why the same index design performs differently on the two engines.

### Mechanism: the lookup paths

```text
 InnoDB: SELECT * FROM orders WHERE customer_id = 81           (secondary index on customer_id)

  secondary index (customer_id)            clustered index (PRIMARY KEY id) = the table
  ┌──────────┐                             ┌──────────┐
  │ root     │                             │ root     │
  └────┬─────┘                             └────┬─────┘
       ▼                                        ▼
  ┌──────────┐                             ┌──────────┐
  │ internal │                             │ internal │
  └────┬─────┘                             └────┬─────┘
       ▼                                        ▼
  leaf: (81 → id 102) (81 → id 4410) …  ──► leaf: [id 102 | customer 81 | PAID | 2500 | …]   ← the row
  1st descent gives the PRIMARY KEY         2nd descent per matching row ("double lookup")

 PostgreSQL: same query, index on customer_id

  index (customer_id)                         heap (table pages, unordered)
  root → internal → leaf: (81 → TID (0,2)) (81 → TID (412,7)) …
                                    │                │
                                    ▼                ▼
                               heap page 0      heap page 412       one heap page read per row
                               slot 2           slot 7              (more if the rows are scattered)
```

Consequences:

- **InnoDB primary-key lookups are cheapest**: the row is in the leaf. Range scans by primary key read rows in key order from adjacent pages.
- **InnoDB secondary indexes contain the primary key**, so a wide primary key (a long string, a composite key) makes **every** secondary index larger. A narrow primary key matters more in InnoDB than in PostgreSQL.
- **InnoDB secondary-index lookups repeat a tree descent per row.** For thousands of matching rows that is thousands of descents into the clustered index.
- **PostgreSQL heap reads depend on correlation.** If rows with nearby key values sit on nearby pages (the table was loaded in key order), matching rows share pages and few heap pages are read. If they are scattered, every matching row may cost a different page. The optimiser estimates this with a statistic called **correlation** (Section 16.9).
- **PostgreSQL row updates may touch every index.** A new tuple version at a new TID needs new entries in every index, unless the update is HOT (Chapter 15, Section 15.2). In InnoDB, an update that does not change indexed columns or the primary key changes only the clustered row.

### Mechanism: index-only scans, and why visibility matters

If every column a query needs is in the index, the database could skip the table entirely:

```sql
CREATE INDEX orders_customer_total ON orders (customer_id, total_cents);
SELECT customer_id, SUM(total_cents) FROM orders WHERE customer_id = 81 GROUP BY customer_id;
-- needs only customer_id and total_cents: both are in the index
```

In PostgreSQL there is a catch. Whether a row version is visible to a transaction is recorded in the heap tuple, not in the index (Chapter 15, Section 15.2; the rules are Chapter 17's subject). An index entry may point to a dead or not-yet-committed version. PostgreSQL resolves this with the **visibility map**, a small structure with a bit per heap page meaning "every tuple on this page is visible to all transactions". During an index-only scan, for each entry PostgreSQL checks the bit for the entry's heap page: if set, it trusts the index; if not, it must fetch the heap page to check visibility after all. VACUUM sets the bits; any change to a page clears its bit.

```text
 Index Only Scan using orders_customer_total on orders  (actual rows=104 loops=1)
   Index Cond: (customer_id = 81)
   Heap Fetches: 0          ← every page all-visible: the heap was never read
 … on a table with many recent updates and lagging autovacuum:
   Heap Fetches: 97         ← most rows needed a heap visit: barely better than an index scan
```

InnoDB has an analogous issue for secondary indexes, which store a page-level marker of the newest transaction to change the page. When that marker shows a recent change, InnoDB checks visibility in the clustered index. In both engines, index-only scans are most effective on data that is not being modified heavily.

PostgreSQL (11+) can add non-key columns to an index with `INCLUDE`, so that an index can cover a query without those columns affecting the index's order or uniqueness:

```sql
CREATE UNIQUE INDEX orders_number ON orders (order_number) INCLUDE (status, total_cents);
-- unique on order_number only; status and total_cents ride along in the leaves for index-only scans
```

In InnoDB, any secondary index already "includes" the primary key, and other columns can only be added as additional key columns.

### Guarantees and non-guarantees

A clustered index guarantees that rows are stored in key order and found by key in one descent. A secondary index guarantees a path to every row with a given key. An index-only scan guarantees correct results; it does not guarantee that the table is never read (that depends on the visibility map or page markers). Neither kind guarantees good performance when a query matches a large fraction of the table (Section 16.10).

### Failure and diagnosis

- **An index-only scan that is slow.** Evidence: `Heap Fetches` close to the number of rows. Cause: autovacuum not keeping the visibility map current on a frequently updated table. Fix: vacuum more often for that table (per-table autovacuum settings).
- **Large secondary indexes in InnoDB.** Evidence: secondary indexes far larger than their key columns suggest. Cause: a wide primary key copied into every one. Fix: a narrow surrogate primary key (Chapter 14, Section 14.2).
- **Many random heap reads for a range.** Evidence: `EXPLAIN (ANALYZE, BUFFERS)` showing many shared reads for an index scan returning a few thousand rows; low correlation. Fix: a covering index, or clustering the table physically (PostgreSQL's one-off `CLUSTER` command, which is not maintained afterwards).

### Backend connection

In InnoDB, choose a narrow, increasing primary key, because it is the table's physical order and part of every secondary index. In PostgreSQL, expect every index lookup to read heap pages unless the index covers the query and the table is well vacuumed. For hot read paths, a covering index that turns many scattered reads into one index range scan is often the single biggest speed-up available.

---

## 16.4 Composite Indexes

### Definitions

A **composite** (multi-column) index has a key made of several columns, such as `(customer_id, status, created_at)`. Its entries are sorted by the first column, then by the second within equal first values, and so on, like a phone book sorted by surname, then first name.

The **leftmost-prefix rule**: a composite index can be used to **seek** (descend directly to a starting point) only on a prefix of its columns, starting from the first.

The **equality-then-range rule**: in a composite index, columns compared with equality should come before a column compared with a range, because after the first range column, later columns can no longer narrow the seek.

### Why it exists

Real queries filter on several columns at once (`WHERE customer_id = ? AND status = ?`) and sort by another (`ORDER BY created_at DESC`). Separate single-column indexes can be combined only at extra cost (bitmap combination, Section 16.10), and cannot provide sorted output for the combination. One composite index that matches the query can find exactly the matching rows, in the needed order, in one range scan.

### Mechanism: the sorted order is everything

An index on `(a, b, c)`, shown as its sorted entries:

```text
 a │ b │ c
 1 │ 1 │ 5
 1 │ 1 │ 9
 1 │ 2 │ 3
 1 │ 2 │ 7
 2 │ 1 │ 4
 2 │ 3 │ 1
 2 │ 3 │ 8
 3 │ 1 │ 2
```

- Entries with `a = 1` are contiguous; within them, entries with `b = 2` are contiguous; within those, `c` is sorted.
- Entries with `b = 1` are **not** contiguous: they appear under `a = 1`, `a = 2` and `a = 3`, scattered through the index. Without a condition on `a`, the database cannot seek to them.
- After a range on `a` (`a > 1`), the `b` values are no longer sorted across the matching entries (1, 3, 3, 1), so a condition on `b` can only filter, not seek.

### Concrete: which clauses can use an index on (a, b, c)

| Query clause | Uses the index? | How |
|---|---|---|
| `WHERE a = 1` | yes | seek to `a = 1`, scan the contiguous block |
| `WHERE a = 1 AND b = 2` | yes | seek to `(1, 2)` |
| `WHERE a = 1 AND b = 2 AND c = 7` | yes | seek to the exact entry |
| `WHERE a = 1 AND b > 1` | yes | seek to `(1, >1)`, scan while `a = 1` |
| `WHERE a = 1 AND b = 2 AND c > 5` | yes | seek on all three |
| `WHERE a = 1 AND c = 7` | partly | seek on `a = 1`; check `c = 7` on each entry in that block (a filter, not a seek) |
| `WHERE a > 1 AND b = 3` | partly | seek on `a > 1`; `b = 3` only filters (b is not sorted within the range) |
| `WHERE b = 2` | not for seeking | no condition on `a`: a full index scan at best (see "skip scan" below) |
| `WHERE b = 2 AND c = 7` | not for seeking | same |
| `WHERE a = 1 ORDER BY b` | yes, and no sort needed | within `a = 1`, entries are already ordered by `b` |
| `WHERE a = 1 ORDER BY b DESC, c DESC` | yes | scan the block backwards |
| `WHERE a = 1 ORDER BY b, c DESC` | for the filter; a sort is still needed | mixed directions do not match the index order (unless the index is declared `(a, b, c DESC)`) |
| `ORDER BY a, b LIMIT 10` | yes | read the first 10 entries in order |
| `WHERE a IN (1, 2) AND b = 1` | yes | one seek per value of `a` |

**Skip scan.** Some recent versions (MySQL from 8.0.13, PostgreSQL from version 18) can use an index on `(a, b)` for `WHERE b = 2` by seeking separately for each distinct value of `a`. This helps only when `a` has few distinct values; it is not a substitute for an index that leads with `b`.

### Mechanism: choosing the column order

For a query `WHERE customer_id = ? AND status = ? AND created_at >= ? ORDER BY created_at DESC LIMIT 20`:

```text
 index (customer_id, status, created_at)
   equality columns first (customer_id, status), then the range/sort column (created_at)
   → seek to (81, 'PAID', now), read 20 entries backwards: exactly the result, already sorted, then stop

 index (customer_id, created_at, status)
   → seek to customer 81; walk created_at in order, FILTERING on status: reads many non-PAID entries

 index (created_at, customer_id, status)
   → a range on the FIRST column: seek to the date, then filter every order of every customer since then
```

The rules of thumb that follow:

1. **Equality columns first**, then **one** range or sort column. Only the first range column can be used for seeking; later columns only filter.
2. Among equality columns, order rarely matters for that query, so choose an order that lets the index also serve **other** queries (put the column that appears alone in other queries first).
3. Put the sort column right after the equality prefix, in the sort's direction, so the index supplies the order and `LIMIT` can stop early (this is how keyset pagination stays fast, Chapter 13, Section 13.6).
4. Do not lead with a low-selectivity column for its own sake. The old advice "most selective column first" matters less than the equality-then-range rule and the set of queries the index must serve.
5. One composite index `(a, b)` usually makes a separate index on `(a)` redundant: `(a, b)` serves every query `(a)` serves, at a slightly larger size.

### Guarantees and non-guarantees

A composite index guarantees seeks on any leftmost prefix of its columns, filtering on later columns inside the scanned range, and ordered output matching its column order and directions. It does not guarantee seeks on non-leading columns (except limited skip scans), seeks past the first range column, or help for sort orders that do not match it.

### Failure and diagnosis

- **An index exists but the plan filters many rows.** Evidence: `EXPLAIN ANALYZE` shows an index scan with a large `Rows Removed by Filter` or a non-indexed condition; the index's column order puts a range before an equality column. Fix: reorder to equality-then-range.
- **A sort on top of an index scan.** Evidence: a `Sort` node above the index scan for a query with `ORDER BY … LIMIT`. Cause: the sort column is not right after the equality prefix, or directions differ. Fix: an index matching the order.
- **Redundant indexes.** `(a)` and `(a, b)` both exist. Evidence: compare index definitions; `(a)` rarely chosen. Fix: drop the redundant one after checking usage.

### Backend connection

Design composite indexes from your most important queries: list each query's equality filters, its range filter and its sort, and build the index in that order. One well-ordered composite index often serves a filter, a sort and a `LIMIT` in a single range scan, which is what keeps list endpoints fast at any table size.

---

## 16.5 Covering, Partial, Expression and Unique Indexes

### Definitions

- A **covering index** contains every column a query reads, so the query can run as an index-only scan (Section 16.3).
- A **partial index** indexes only the rows that satisfy a condition given when the index is created (`CREATE INDEX … WHERE status = 'PENDING'`). PostgreSQL supports them; MySQL does not.
- An **expression index** (functional index) indexes the result of an expression over columns, such as `lower(email)` or `(created_at::date)`, instead of a column itself. PostgreSQL supports them; MySQL supports them from 8.0.13 as functional key parts.
- A **unique index** rejects two entries with the same key. It is how `UNIQUE` and `PRIMARY KEY` constraints are enforced (Chapter 14, Section 14.3).

### Why they exist

A plain index on a column is often the wrong shape for the query: it includes millions of rows the query never wants, indexes the raw value when queries search a transformed one, or forces a visit to the table for columns it lacks. These variants reshape the index to the query, often making it both faster and much smaller.

### Mechanism and concrete: covering indexes

A list endpoint shows a customer's recent orders with their status and total:

```sql
SELECT id, status, total_cents, created_at
FROM orders
WHERE customer_id = 81
ORDER BY created_at DESC
LIMIT 20;
```

```text
 index (customer_id, created_at)                          → seek, 20 entries in order, then 20 heap visits
 index (customer_id, created_at) INCLUDE (status, total_cents)   (PostgreSQL)
 index (customer_id, created_at, status, total_cents)            (MySQL; InnoDB adds id automatically)
                                                          → seek, 20 entries in order, no table access
```

The covering version answers the query from about one leaf page. Covering indexes cost space, and every included column makes updates of that column touch the index too, so they are for hot read paths, not for every query.

### Mechanism and concrete: partial indexes

A job table holds 50 million completed jobs and a few thousand pending ones, and workers repeatedly ask for pending jobs:

```sql
CREATE INDEX jobs_pending ON jobs (run_at) WHERE status = 'PENDING';

SELECT id FROM jobs WHERE status = 'PENDING' AND run_at <= now() ORDER BY run_at LIMIT 100;
```

```text
 full index on (status, run_at): 50 million entries, about 1.5 GB, updated for every job
 partial index:                  a few thousand entries, a few hundred KB; completed jobs are not in it
```

The planner uses a partial index only when it can prove that the query's `WHERE` clause implies the index's condition. Here the query says `status = 'PENDING'`, which matches exactly. A query with `status = $1`, where the value is a parameter, may not qualify when the plan is prepared for unknown values (prepared statements are Chapter 18's subject).

Partial **unique** indexes express rules that a plain constraint cannot:

```sql
-- one active account per email; soft-deleted rows may repeat the email
CREATE UNIQUE INDEX users_email_active ON users (lower(email)) WHERE deleted_at IS NULL;

-- at most one successful payment per order (Chapter 13, Section 13.5)
CREATE UNIQUE INDEX one_success_per_order ON payments (order_id) WHERE status = 'SUCCEEDED';
```

### Mechanism and concrete: expression indexes

An index on `email` stores the values as written. A query for `lower(email) = 'ana@x.com'` asks about a **different** value, computed per row, so the index on `email` cannot answer it:

```sql
CREATE INDEX users_email_lower ON users (lower(email));
SELECT id FROM users WHERE lower(email) = lower('Ana@X.com');   -- uses the expression index
```

The query's expression must match the index's expression. In PostgreSQL, the function must be **immutable** (always the same result for the same input), which is why an expression on `timestamptz` that depends on the session's time zone cannot be indexed directly (Chapter 14, Section 14.12). Expression indexes also gather their own statistics, which improves estimates for that expression (Section 16.9).

### Mechanism: unique indexes

Inserting into a unique index means searching for the key first; if an entry exists for a visible or in-progress row, the insert fails (or, for an in-progress row, waits for the other transaction to finish, the behaviour Chapter 13's idempotency keys rely on; Chapter 17 explains it). Two consequences:

- A unique index doubles as a fast lookup index; there is no need for a second, non-unique index on the same columns.
- `NULL`s are not equal to each other, so a unique index allows many `NULL`s unless declared otherwise (Chapter 14, Section 14.10).

### Guarantees and non-guarantees

Covering indexes guarantee that the needed data is in the index, not that the table is never touched (visibility, Section 16.3). Partial indexes guarantee a small index for the subset; they are used only for queries the planner can prove fall inside it. Expression indexes are used only for the matching expression. Unique indexes guarantee uniqueness of their key among the rows they cover, under any concurrency.

### Failure and diagnosis

- **A partial index is ignored.** Evidence: `EXPLAIN` shows another plan; the query's condition does not textually imply the index predicate (for example `status IN ('PENDING','RETRY')` against an index on `status = 'PENDING'`). Fix: align the predicate, or widen the index's condition.
- **An expression index is ignored.** The query writes a different expression (`LOWER(TRIM(email))` versus `lower(email)`) or compares a different type. Fix: use exactly the indexed expression.
- **Duplicates slip past "unique".** The unique index covers a different expression from the business rule (case, whitespace, soft deletes). Fix: index the normalised expression with the right condition.

### Backend connection

Shape indexes to queries: cover the hottest list and lookup queries, index only the rows a queue or workflow actually polls, index the normalised form that the application compares, and use partial unique indexes to encode rules like "one active" or "one successful". These variants routinely make indexes both faster and a fraction of the size.

---

## 16.6 Other Index Types

### Definitions

B+trees handle equality, ranges and sorting on ordinary scalar values. Other data and other questions need other structures:

- A **hash index** maps the hash of the key to entries: equality only.
- An **inverted index** maps each element contained in a value (a word in a document, a key in a JSON object, an element of an array) to the list of rows that contain it. PostgreSQL's implementation is **GIN** (Generalised Inverted Index).
- **GiST** (Generalised Search Tree) is PostgreSQL's framework for balanced trees over data that can be bounded and overlapped rather than sorted: geometric shapes, ranges, nearest-neighbour searches. **SP-GiST** handles space-partitioned structures such as quadtrees.
- A **BRIN** (Block Range Index) stores, for each range of consecutive table pages, a summary such as the minimum and maximum value of a column.
- A **bitmap index** (in Oracle and some analytical databases) stores, for each distinct value of a column, a bitmap with one bit per row. PostgreSQL has no persistent bitmap indexes; it builds bitmaps temporarily during **bitmap scans** (Section 16.10).

### Why they exist

A B+tree cannot answer "which documents contain the word *keyboard*", "which JSON objects contain `{"color": "black"}`", "which bookings overlap this time range", or "which stores are nearest this point", because none of those questions is about the sorted order of a single scalar value. And for a 2 TB append-only table of events, a B+tree on the timestamp may itself take hundreds of gigabytes, when a summary per block range would take a few megabytes.

### Mechanism: the inverted index (GIN)

```text
 documents (rows)                                   inverted index: term → posting list (row TIDs)
 row (0,1): "mechanical keyboard, black"            black      → (0,1) (0,3)
 row (0,2): "wireless mouse"                        keyboard   → (0,1) (0,3)
 row (0,3): "black keyboard with numpad"            mechanical → (0,1)
                                                    mouse      → (0,2)
                                                    numpad     → (0,3)
                                                    wireless   → (0,2)

 query: documents containing "black" AND "keyboard"
   → intersect posting lists: {(0,1),(0,3)} ∩ {(0,1),(0,3)} = {(0,1),(0,3)}
```

```sql
-- full-text search
CREATE INDEX products_fts ON products USING gin (to_tsvector('english', name || ' ' || description));
SELECT id FROM products WHERE to_tsvector('english', name || ' ' || description) @@ to_tsquery('black & keyboard');

-- JSON containment (Chapter 14, Section 14.12)
CREATE INDEX product_attrs_gin ON product_attributes USING gin (attrs);
SELECT product_id FROM product_attributes WHERE attrs @> '{"color": "black"}';

-- substring search with trigrams (the pg_trgm extension): makes LIKE '%board%' indexable
CREATE INDEX products_name_trgm ON products USING gin (name gin_trgm_ops);
```

A row with many elements (a long document) adds many index entries, so inverted indexes are expensive to update. PostgreSQL's GIN reduces that cost by collecting new entries in a pending list and merging them in bulk (the `fastupdate` option), at the price of slower searches until the merge.

### Mechanism: GiST, ranges and exclusion constraints

GiST indexes keep, at each internal node, a **bounding** description of everything below it (for shapes, a bounding box; for ranges, the covering range). A search descends only into children whose bounds can satisfy the query, such as "overlaps this range" or "is within 5 km of this point".

Combined with an **exclusion constraint**, GiST enforces a rule that no `UNIQUE` index can express: no two rows may overlap.

```sql
-- PostgreSQL; the btree_gist extension lets "=" on a plain column take part in a GiST index
CREATE TABLE room_bookings (
    room_id bigint    NOT NULL,
    during  tstzrange NOT NULL,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)   -- same room AND overlapping time: rejected
);
INSERT INTO room_bookings VALUES (7, '[2026-10-21 10:00, 2026-10-21 11:00)');
INSERT INTO room_bookings VALUES (7, '[2026-10-21 10:30, 2026-10-21 11:30)');
-- ERROR:  conflicting key value violates exclusion constraint "room_bookings_room_id_during_excl"
```

This is the database-level guard for the classic double-booking race: two concurrent requests that each "check availability" and then insert (Chapter 5, Section 5.2) cannot both succeed.

### Mechanism: BRIN

```text
 events table, 2 TB, rows appended in time order; BRIN with one summary per 128 pages

 block range       │ min(created_at)    │ max(created_at)
 pages 0–127       │ 2025-01-01 00:00   │ 2025-01-01 00:41
 pages 128–255     │ 2025-01-01 00:41   │ 2025-01-01 01:19
 …                 │ …                  │ …

 WHERE created_at BETWEEN '2026-10-21 10:00' AND '2026-10-21 11:00'
   → only the few block ranges whose [min, max] overlaps the hour are read
```

A BRIN index is the row-store version of the zone maps in Chapter 15 (Section 15.7). It is tiny (a few entries per thousand pages) and cheap to maintain, but only useful when the column's values follow the physical order of the table. On a column whose values are scattered (a customer ID), every block range covers nearly every value and the index skips nothing.

### Mechanism: hash indexes and bitmap indexes

A **hash index** answers only `=`, cannot provide order or ranges, and cannot enforce uniqueness in PostgreSQL. It can be smaller than a B+tree for long keys (it stores hash codes, not the keys). In practice a B+tree is usually just as fast for equality and far more versatile, so hash indexes are rarely the right choice. (InnoDB builds an internal **adaptive hash index** automatically over hot B+tree pages; you do not create it.)

A **bitmap index** on a low-cardinality column, such as `status` with five values, stores five bitmaps over all rows. `status = 'PAID' AND region = 'EU'` becomes a bitwise AND of two bitmaps, which is very fast for analytics. But changing one row's status means updating two bitmaps that cover millions of rows, and those bitmap segments are shared between concurrent writers, so bitmap indexes suit read-mostly warehouses, not OLTP tables.

### Concrete: choosing an index type

| Question | Index | Example |
|---|---|---|
| Equality, ranges, sorting, prefixes on scalar values | B+tree | `WHERE customer_id = ? ORDER BY created_at` |
| Equality only, very long keys | hash (rarely better than B+tree) | `WHERE long_token = ?` |
| Contains a word, a key, an array element | GIN | full-text search, `jsonb @>`, `tags @> ARRAY['sale']` |
| Substring or fuzzy text match | GIN or GiST with trigrams | `name ILIKE '%board%'` |
| Overlap, containment, nearest neighbour on ranges and geometry | GiST (SP-GiST) | bookings, maps, `ORDER BY location <-> point` |
| Huge append-only table filtered by a naturally ordered column | BRIN | event logs by time |
| Low-cardinality filters in a read-mostly warehouse | bitmap (where offered) | `status`, `region` in analytics |
| Similarity search over embeddings | vector indexes (for example HNSW, through PostgreSQL's pgvector extension) | semantic search |

### Guarantees and non-guarantees

Each index type guarantees fast answers to its own family of operators. None of them answers questions outside that family: a GIN index does not help a range query, a BRIN index does not help a scattered column, a hash index does not help `ORDER BY`. Specialised indexes generally cost more per write than B+trees (GIN especially) and their effectiveness depends on data distribution (BRIN especially).

### Failure and diagnosis

- **Full-text or JSON queries scanning the table.** Evidence: `Seq Scan` with a `@@` or `@>` filter. Cause: no GIN index, or the query's expression differs from the indexed one (a different text-search configuration, a different expression). Fix: index exactly the queried expression.
- **Slow writes on a table with a GIN index.** Evidence: insert latency spikes when the pending list is merged; GIN index size large. Fix: tune `fastupdate`, batch writes, or move search to a dedicated search engine.
- **A BRIN index that does not help.** Evidence: `EXPLAIN ANALYZE` shows a bitmap heap scan reading most pages ("Rows Removed by Index Recheck" huge). Cause: the column is not physically ordered. Fix: a B+tree, or load data in order.

### Backend connection

Before building search, overlap checking or geospatial logic in application code, check whether an index type already does it in the database: GIN for search and JSON, GiST with exclusion constraints for "no overlaps", BRIN for enormous time-ordered tables. Use them deliberately, because each is fast for its own questions and expensive or useless for others.

---

## 16.7 The Cost of Indexes

### Definition

The **cost of an index** is the extra work and space it adds to the database: work on every insert, on every update that changes an indexed column (in PostgreSQL, on every non-HOT update), and on every delete's cleanup; storage on disk and in the buffer pool; WAL volume; and maintenance such as vacuuming and rebuilding. **Index bloat** is space in an index occupied by entries for dead rows or by half-empty pages, beyond what its live entries need.

### Why it matters

"Why not index every column?" has a quantitative answer. Every index makes reads that use it faster and makes every write slower, larger and more I/O-intensive. On a write-heavy table, unused indexes are pure cost. On a table whose indexes no longer fit in memory, they also make reads slower, by competing for the buffer pool.

### Mechanism: what one insert does

```text
 INSERT INTO orders (…) VALUES (…);            orders has a primary key and 5 secondary indexes

 PostgreSQL                                    InnoDB
 1 heap tuple written                          1 row inserted into the clustered PK B+tree
 6 index entries inserted (pk + 5)             5 secondary index entries (each holds the PK)
   each: descend the tree, modify a leaf,        each: descend, modify a leaf, possibly split
   possibly split
 WAL records for all 7 page changes            redo for all 6 page changes
 up to 7 dirty pages, each written later       up to 6 dirty pages, each written later
 (plus full-page images after checkpoints)     (InnoDB may buffer secondary-index changes for
                                                pages not in memory: its change buffer)
```

### Concrete: the numbers

Assume a table of 100 million rows, and the costs per row:

```text
 index                                          entry size (approx.)    index size at ~90% fill
 primary key (id bigint)                        ~22 bytes               ~2.4 GB
 (customer_id bigint)                           ~22 bytes               ~2.4 GB
 (customer_id, status text, created_at)         ~40 bytes               ~4.4 GB
 (lower(email)) on a 30-byte email              ~45 bytes               ~5 GB
 (status) alone                                 ~20 bytes               ~2.2 GB  (rarely useful: 5 distinct values;
                                                                         PostgreSQL 13+ deduplicates repeated keys,
                                                                         which shrinks such an index considerably)
 total indexes                                                          ~16.4 GB
 the table itself (≈ 64-byte rows)                                      ~6.5 GB
```

The indexes are more than twice the size of the table. All of them compete with the table for the buffer pool; if the working set no longer fits in memory, every index whose leaves are touched by inserts brings random reads with it.

For writes, assume, as an illustration, that each index insertion costs roughly as much as the row insertion itself (a descent plus a leaf modification, plus its share of WAL and page writes):

| Indexes on the table | Structures modified per insert | Relative insert cost (illustrative) | WAL per insert (illustrative) |
|---|---|---|---|
| 1 (primary key) | 2 | 1× | ~200 bytes |
| 3 | 4 | ~2× | ~350 bytes |
| 6 | 7 | ~3.5× | ~600 bytes |
| 10 | 11 | ~5.5× | ~900 bytes |

At 5,000 inserts per second with 6 indexes, that is about 35,000 page modifications per second and on the order of 3 MB of WAL per second (more right after checkpoints), all of which must also be replicated and archived (Chapter 15, Section 15.9). The numbers depend heavily on key widths, caching and engine, but the shape is general: write cost grows roughly linearly with the number of indexes.

Updates follow the same logic. In PostgreSQL a non-HOT update creates a new tuple version and new entries in **every** index, even those on unchanged columns; adding an index on a frequently updated column turns HOT updates into non-HOT ones for that column. In InnoDB an update touches only the indexes whose columns change (and the clustered row).

### Mechanism: bloat and maintenance

- **Dead entries.** In PostgreSQL, index entries for dead tuples stay until VACUUM removes them (Chapter 15, Section 15.2). Under heavy update or delete rates, indexes grow beyond their live size.
- **Half-empty pages.** Random inserts leave pages partly full (Section 16.2), and deletes leave gaps that are not reclaimed until pages empty completely.
- **Rebuilding.** A rebuilt index is compact. PostgreSQL's `REINDEX INDEX CONCURRENTLY` (version 12+) rebuilds without blocking writes; plain `REINDEX` blocks writes to the table. MySQL's `OPTIMIZE TABLE` rebuilds an InnoDB table and its indexes, mostly online.
- **Creating indexes on live tables.** A plain `CREATE INDEX` in PostgreSQL blocks writes for its duration; `CREATE INDEX CONCURRENTLY` does not, takes longer, and can leave an invalid index behind if it fails, which must be dropped. MySQL's InnoDB builds most indexes online. Migrations must use the non-blocking forms (Chapter 18).

### Concrete: finding indexes that do not pay for themselves

```sql
-- PostgreSQL: indexes never used since statistics were last reset, largest first
SELECT s.relname AS table_name, s.indexrelname AS index_name, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS index_size
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0 AND NOT i.indisunique              -- unique indexes enforce rules even if never scanned
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

```text
 table_name │ index_name              │ idx_scan │ index_size
 orders     │ orders_status_idx       │ 0        │ 2215 MB
 orders     │ orders_created_at_idx   │ 0        │ 2203 MB
```

Before dropping, check that statistics cover a representative period (including month-end reports), that the index is not used on replicas (each server keeps its own usage statistics), and that it is not needed for a rare but critical query. MySQL's `sys.schema_unused_indexes` view gives a similar list.

### Guarantees and non-guarantees

Every index is kept consistent with its table in the same transaction, which is exactly why it costs on every write. Nothing guarantees that an index is used, that it stays compact, or that it fits in memory.

### Failure and diagnosis

- **Write throughput falls after "adding an index to fix a slow query".** Evidence: insert latency and WAL rate up after the migration. Fix: check that the new index replaced, rather than added to, a less useful one; consider a partial or covering variant.
- **Indexes larger than the table, buffer pool misses up.** Evidence: index sizes; hit ratio falling. Fix: remove unused and redundant indexes; rebuild bloated ones.
- **Writes blocked during a migration.** Evidence: lock waits on the table while `CREATE INDEX` runs. Fix: `CONCURRENTLY` (PostgreSQL) or online DDL.

### Backend connection

Treat each index as a cost you pay on every write to buy speed for specific reads. Add indexes for queries that need them, measure their use, drop the ones that do not earn their keep, and create them without blocking writes. "Index every column" makes writes several times slower and the cache smaller, for queries that may never run.

---

## 16.8 Query Processing: From SQL to Execution

### Definitions

**Query processing** is the pipeline that turns a SQL string into results:

1. **Parsing** checks the syntax and builds a **parse tree**.
2. **Analysis** (binding) resolves names against the catalogue (which table is `orders`, what type is `total_cents`) and checks types and permissions.
3. **Rewriting** applies rules to the tree: expanding views into their definitions (Chapter 14, Section 14.11), and, in PostgreSQL, adding row-level security conditions.
4. **Planning** (optimisation) chooses how to execute the query, producing a **physical plan**.
5. **Execution** runs the plan and produces rows.

A **logical plan** describes **what** to compute, as a tree of relational-algebra operators (selection, projection, join, aggregation; Chapter 14, Section 14.14). A **physical plan** describes **how**: which access method reads each table (sequential scan, index scan) and which algorithm implements each operator (hash join, merge join, sort). One logical plan has many possible physical plans.

### Why it matters

Declarative SQL (Chapter 14, Section 14.1) leaves the "how" to the database, so the planner's choice decides performance. Seeing the pipeline explains where things go wrong: syntax errors appear at parse time, unknown columns at analysis, a view's hidden joins after rewriting, and slow queries almost always at planning, where the database guessed wrong about how many rows something would return.

### Mechanism: the pipeline

```text
 "SELECT c.name, SUM(o.total_cents) FROM customers c JOIN orders o ON o.customer_id = c.id
  WHERE o.created_at >= '2026-10-01' GROUP BY c.name"
        │
 PARSE  ▼  parse tree: SELECT node, FROM with a JOIN, WHERE expression, GROUP BY
        │
 ANALYSE▼  resolve: customers → table OID 16402, o.total_cents → integer, permissions OK
        │
 REWRITE▼  (no views here; a view would be replaced by its definition)
        │
 PLAN   ▼  logical:   γ name, SUM(total) ( customers ⋈ σ created_at ≥ … (orders) )
           physical:  HashAggregate
                        └─ Hash Join (o.customer_id = c.id)
                             ├─ Index Scan on orders using orders_created_at_idx (created_at ≥ …)
                             └─ Hash
                                  └─ Seq Scan on customers
        │
 EXECUTE▼  rows
```

### Mechanism: the iterator model of execution

Most row stores execute a physical plan as a tree of **iterators** (the "Volcano" model). Each node offers one operation, `next()`, which returns its next row. A node calls `next()` on its children as needed:

```pseudo
class SeqScan:   next(): read rows page by page; return the next row that passes the filter
class IndexScan: next(): step along the index leaves; fetch the row; return it
class HashJoin:
    open():  build ← for each row from the build child: insert into a hash table on the join key
    next():  loop: probeRow ← probe child.next(); for each match in build[probeRow.key]: return combined row
class Sort:      open(): read ALL rows from the child, sort them; next(): return them in order
class Limit:     next(): if returned < n: return child.next() else: stop
```

Rows flow up the tree one at a time. Most operators are **pipelined**: they produce output as their input arrives, so a `LIMIT 20` above a pipelined plan stops the whole tree after 20 rows. Some are **blocking**: a sort, the build side of a hash join, and hash aggregation must consume their entire input before producing anything. That is why `ORDER BY created_at LIMIT 20` is instant with an index that provides the order (pipelined) and slow without one (a blocking sort of every matching row first). Column stores use the vectorised variant of this model (Chapter 15, Section 15.7).

### Mechanism: planning once, executing many times

Planning a complex query can take milliseconds, a real cost for a query that runs in a fraction of a millisecond thousands of times per second. Databases can reuse plans for **prepared statements**, which separate the SQL text from its parameter values (Chapter 18 covers prepared statements in drivers). PostgreSQL plans the first several executions of a prepared statement with the actual parameter values (**custom plans**) and may then switch to a **generic plan** that ignores the specific values, if its estimated cost is similar. A generic plan is faster to reuse but can be wrong for skewed values: a plan good for a customer with 10 orders may be terrible for a customer with 10 million. Section 16.9 explains why the values matter.

### Guarantees and non-guarantees

The pipeline guarantees that every physical plan it chooses returns the same rows as the logical query (the result of Chapter 14's semantics), and that permissions and views are applied before planning. It does not guarantee that the chosen physical plan is the fastest one: the planner chooses by **estimated** cost, and estimates can be wrong.

### Failure and diagnosis

- **Planning time dominates.** Evidence: `EXPLAIN ANALYZE` shows "Planning Time" larger than "Execution Time" for short queries; many joined tables or partitions. Fix: prepared statements, fewer partitions touched (Section 16.14), simpler queries.
- **A prepared statement is slow only for some values.** Evidence: the same statement fast in a console (custom plan) and slow from the application (generic plan); skewed data. Fix: force custom plans for that statement (PostgreSQL's `plan_cache_mode`), or split the query by case.
- **A view hides expensive work.** Evidence: the plan for a simple query over a view contains joins the query did not mention. Fix: query the base tables for hot paths, or simplify the view.

### Backend connection

Every slow query is a planning decision you can inspect. Knowing the pipeline tells you where to look: the plan's shape for where time goes, the estimates for why the planner chose it, and the statement's preparation for whether the plan you see in a console is the plan the application runs.

---

## 16.9 The Optimiser: Statistics, Selectivity, Cardinality and Cost

### Definitions

The **optimiser** (query planner) chooses a physical plan for a query by enumerating candidate plans and picking the one with the lowest **estimated cost**.

**Statistics** are summaries of each table's data, gathered by sampling: number of rows and pages, fraction of `NULL`s, number of distinct values, most common values and their frequencies, a histogram of value distribution, and the correlation between value order and physical row order.

**Selectivity** is the fraction of rows that satisfy a condition: 1.0 for a condition every row meets, 0.00001 for a rare value. **Cardinality** (in a plan, "rows") is the number of rows an operator produces; the estimated cardinality is selectivity × input rows. **Cardinality estimation** is predicting these numbers before running the query.

The **cost model** turns estimated rows and pages into a single number per plan, in arbitrary units, by weighting operations: reading a page sequentially, reading one at random, processing a row, evaluating an operator.

### Why it exists

The same query can run as a sequential scan or through any of several indexes, with joins in many orders and with several algorithms each. Their running times differ by factors of thousands, and which is fastest depends on the data: how many rows match. The optimiser cannot run every plan to find out, so it estimates. **Almost every bad plan is a bad cardinality estimate**: the cost model is reasonable, but it was fed the wrong number of rows. That is why plans change when data changes, and why statistics matter.

### Mechanism 1: what the statistics look like

```sql
-- PostgreSQL: statistics for two columns of orders (10 million rows)
SELECT attname, null_frac, n_distinct, most_common_vals, most_common_freqs, correlation
FROM pg_stats WHERE tablename = 'orders' AND attname IN ('status', 'customer_id');
```

```text
 attname     │ null_frac │ n_distinct │ most_common_vals                           │ most_common_freqs              │ correlation
 status      │ 0         │ 5          │ {PAID,SHIPPED,DELIVERED,PLACED,CANCELLED}  │ {0.62,0.18,0.12,0.05,0.03}     │ 0.19
 customer_id │ 0         │ 452318     │ {…a few heavy customers…}                  │ {…}                            │ 0.01
```

`ANALYZE` (run automatically by autovacuum, or manually) builds these from a random **sample**: in PostgreSQL, 300 × the statistics target rows, so 30,000 rows with the default target of 100 at the time of writing. A histogram divides the sampled values of a column into buckets holding **equal numbers of rows** (equi-depth), so dense value ranges get narrow buckets. **Correlation** near 1 or −1 means rows are stored roughly in that column's order (as `created_at` is in an append-only table); near 0 means scattered.

InnoDB keeps persistent statistics computed from a sample of index pages, plus optional histograms (MySQL 8.0, `ANALYZE TABLE … UPDATE HISTOGRAM ON …`); it can also "dive" into an index at planning time to count entries in a small range.

### Mechanism 2: estimating selectivity

| Condition | Estimate (PostgreSQL's approach, simplified) | Example on `orders` (10 M rows) |
|---|---|---|
| `col = v`, `v` a most common value | its frequency | `status = 'CANCELLED'`: 0.03 → **300,000** rows |
| `col = v`, `v` not a most common value | (1 − sum of MCV frequencies − null_frac) / (n_distinct − number of MCVs) | `customer_id = 81`: ≈ 1 / 452,318 → **≈ 22** rows |
| `col > v` or `BETWEEN` | the fraction of the histogram covered, interpolating inside the bucket | `total_cents > 50000`: covers 4 of 100 buckets → 0.04 → 400,000 rows |
| `cond1 AND cond2` | product: s1 × s2 (assumes independence) | |
| `cond1 OR cond2` | s1 + s2 − s1 × s2 | |
| equi-join `R.a = S.b` | rows(R) × rows(S) / max(distinct(R.a), distinct(S.b)) | orders ⋈ customers: 10 M × 452 k / 452 k = 10 M |

The **independence assumption** is the weakest link. Customers in Berlin are all in Germany, but the planner assumes city and country are unrelated:

```text
 customers: 1,000,000 rows; city = 'Berlin' selectivity 0.05; country = 'DE' selectivity 0.10
 WHERE city = 'Berlin' AND country = 'DE'
   estimate: 0.05 × 0.10 × 1,000,000 =  5,000 rows
   actual:                              50,000 rows   (every Berlin customer is in DE)
   → a 10× underestimate, which can flip the plan below it to a nested loop (Section 16.11)
```

PostgreSQL (10+) can learn such relationships with **extended statistics**: `CREATE STATISTICS customers_city_country (dependencies) ON city, country FROM customers;` followed by `ANALYZE`.

Errors also **compound**. A 10× error on one table, joined with a 10× error on another, can make a join's estimate off by 100× or more, and each operator above it inherits the error.

### Mechanism 3: the cost model, and why the plan flips

PostgreSQL's cost constants (defaults at the time of writing; each can be configured): reading a page sequentially costs **1.0** (`seq_page_cost`); reading one at random **4.0** (`random_page_cost`); processing a row **0.01**; processing an index entry **0.005**; evaluating an operator **0.0025**.

**Sequential scan** of the 10-million-row `orders` table (83,334 pages) with one filter:

```text
 83,334 pages × 1.0  +  10,000,000 rows × 0.01  +  10,000,000 × 0.0025 (the filter)  =  208,334
```

**Index scan** returning N rows scattered across the table: roughly a few index pages plus up to one random heap page per row, so about **4 × N** plus small CPU terms.

```text
 N = 22      (customer_id = 81)        index ≈ 100          seq = 208,334   → index scan, by far
 N = 300,000 (status = 'CANCELLED')    index ≈ 1,200,000    seq = 208,334   → sequential scan
 crossover:  4 × N ≈ 208,334  →  N ≈ 52,000 rows ≈ 0.5% of the table
```

So with scattered rows, an index stops paying when a query matches more than roughly half a percent of the table: random page reads become more expensive than reading everything in order. Two things move the crossover. **Correlation**: if matching rows are physically clustered (correlation near 1), the index scan reads few, nearly sequential pages, and stays cheaper for much larger fractions. **`random_page_cost`**: its default of 4 reflects HDD-era assumptions; on SSDs, where random reads are much closer to sequential ones (Chapter 2, Section 2.4), it is commonly lowered towards about 1.1, which moves the crossover to roughly 190,000 rows here.

This is the whole mechanism of a **plan flip**. A query whose estimated row count crosses the crossover (because the data grew, the statistics changed, or a different parameter value was used) gets a different plan. If the estimate was right, the new plan is better. If the estimate was wrong, the new plan can be catastrophically worse.

### Mechanism 4: searching the plan space

For a query joining several tables, the optimiser must choose an order of joins, an access path for each table and an algorithm for each join. The number of join orders grows factorially: 4 tables have 24 orders of a simple left-deep chain, 10 tables over 3.6 million. Optimisers use **dynamic programming**: find the cheapest plan for each pair of tables, then for each set of three built from the best pairs, and so on (the approach of IBM's System R optimiser, described by Selinger and colleagues in 1979). For many tables, they switch to heuristics: PostgreSQL to a genetic search above a threshold (12 tables by default at the time of writing), MySQL to a greedy search limited in depth.

### Mechanism 5: why statistics go stale

Statistics are refreshed only when `ANALYZE` runs. PostgreSQL's autovacuum analyses a table after a fraction of its rows have changed (by default about 10% plus a small constant, at the time of writing). For a 100-million-row table that means 10 million changes between refreshes, which can take days. Meanwhile:

- A new value appears that the statistics have never seen (a new status `'REVIEW'`), and the estimate falls to the minimum, often 1 row.
- Queries for **recent** data ask about values above the histogram's highest bucket (`created_at > now() - interval '1 hour'` on data inserted since the last `ANALYZE`). PostgreSQL mitigates this for indexed columns by checking the index's actual extreme values, but estimates for such ranges can still be far off, and other engines and unindexed columns are not protected.
- A bulk load or deletion changes the distribution completely, and the next queries run with the old picture.

### Guarantees and non-guarantees

The optimiser guarantees a correct plan and, under its estimates and cost model, a cheap one. It does not guarantee accurate estimates (sampling, independence assumptions and staleness all introduce errors), stable plans over time, or the same plan for different parameter values.

### Failure and diagnosis

- **Estimated rows far from actual rows.** The defining symptom. Evidence: `EXPLAIN ANALYZE` with `rows=1` estimated where thousands are produced (Section 16.12). Fixes: `ANALYZE` the table; more frequent analysis for large or fast-changing tables; a higher statistics target on skewed columns; extended statistics for correlated columns.
- **A plan flip after a data change.** Evidence: the same query fast yesterday and slow today, with a different plan (`auto_explain` or stored plans show both). Diagnosis: compare estimates in both plans; find the node whose estimate crossed a crossover.
- **Skewed parameter values.** A plan good for typical customers terrible for a huge one. Evidence: slowness only for specific values. Fix: custom plans, query variants, or partial indexes for the heavy case.

### Backend connection

When a query is slow, the first question is not "which index is missing?" but "how many rows did the planner think each step would produce, and how many did it produce?". Keep statistics fresh for large and fast-changing tables, teach the planner about correlated columns, and remember that a plan is a prediction that depends on data you will keep changing.

---

## 16.10 Scan Types

### Definitions

A **scan** is the leaf operation of a plan: reading rows from one table.

- A **sequential scan** reads every page of the table in physical order.
- An **index scan** walks an index range and fetches each matching row from the table as it goes, in index order.
- An **index-only scan** answers from the index alone, visiting the table only when visibility cannot be confirmed (Section 16.3).
- A **bitmap scan** (PostgreSQL) first scans one or more indexes to build a **bitmap** of matching row locations, sorts them by page, then reads each needed table page once, in physical order.

### Why there are several

Each wins in a different range of selectivity and correlation. Reading everything sequentially is the cheapest way to get a large fraction of a table; following an index row by row is cheapest for a few rows; between them, collecting all matches first and reading their pages in order avoids reading the same page twice and turns random reads into nearly sequential ones.

### Mechanism: where each wins

```text
 fraction of the table matching  →   0.001%      0.01%        0.1%         1%          10%          100%
 index-only scan (if covering, vacuumed)   ████████████████████████████████████████████
 index scan                            ██████████████████████
 bitmap index + bitmap heap scan                  ██████████████████████████████
 sequential scan                                                       ███████████████████████████████
 (boundaries move with correlation, caching and random_page_cost; Section 16.9)
```

**Bitmap scan mechanism:**

```text
 Bitmap Index Scan on orders_status_idx (status = 'REVIEW')
   → for each matching index entry, set the bit for its (page, slot) in an in-memory bitmap
 [optionally BitmapAnd / BitmapOr with bitmaps from other indexes:
   (status = 'REVIEW') AND (region = 'EU') → AND the two bitmaps: uses two single-column indexes together]
 Bitmap Heap Scan on orders
   → visit the marked pages in increasing page order, each at most once; return matching rows
   if the bitmap grows beyond memory (work_mem) it becomes "lossy": it remembers only pages, not slots,
   and every row on those pages is rechecked against the condition ("Recheck Cond" in the plan)
```

The bitmap scan returns rows in **physical** order, not index order, so it cannot satisfy an `ORDER BY` by itself.

MySQL's `EXPLAIN` names its access methods in the `type` column: `const` and `eq_ref` (one row by a unique key), `ref` (index lookup by non-unique key), `range` (index range), `index` (full index scan) and `ALL` (full table scan). The `Extra` column shows `Using index` for an index-only (covering) access, `Using where` for a filter, `Using filesort` for a sort and `Using temporary` for a temporary table.

### Guarantees and non-guarantees

All scan types return exactly the rows matching the condition. Only index scans and index-only scans return them in index order; sequential and bitmap scans do not guarantee any order. Parallel scans (several workers each reading part of the table, combined by a `Gather` node in PostgreSQL) speed up large scans at the cost of CPU, and also return rows in no particular order.

### Failure and diagnosis

- **A sequential scan for a selective query.** Usually a missing or unusable index (Section 16.13) or a large estimate error (Section 16.9).
- **An index scan for an unselective query.** An underestimate made the index look cheap; the query reads many random pages. Evidence: high `Buffers: shared read` counts and an actual row count far above the estimate.
- **A lossy bitmap.** Evidence: `Heap Blocks: lossy=…` and many rows removed by index recheck. Fix: more `work_mem` for the query, or a more selective index.

### Backend connection

There is no single "fast" scan. For each important query, know how many rows it matches as a fraction of the table: that number, more than any rule of thumb, decides whether an index can help, and which kind of scan the planner should choose.

---

## 16.11 Join Algorithms and Join Ordering

### Definitions

A **join algorithm** is the physical method used to compute a join (Chapter 14, Section 14.7):

- **Nested-loop join**: for each row of the outer input, find the matching rows of the inner input. With an index on the inner join key, this is an **index nested-loop join**.
- **Hash join**: build a hash table on the join key from one input (the **build** side, usually the smaller), then scan the other (the **probe** side) and look each row up.
- **Sort-merge join** (merge join): sort both inputs on the join key (or read them already sorted from indexes), then walk both in step, matching equal keys.

**Join ordering** is the choice of which tables to join first, which decides the sizes of intermediate results.

### Why there are several

The best way to combine two inputs depends on their sizes, on indexes, on memory and on whether the output must be sorted. Joining 20 orders to their customers and joining 10 million orders to 500,000 customers are different problems, and the optimiser picks the algorithm per join from its cardinality estimates.

### Mechanism: the three algorithms

```pseudo
// Nested loop: N = outer rows, M = inner rows
for o in outer:                                   // N iterations
    for i in inner:                               // M iterations each → N × M comparisons
        if o.key == i.key: emit(o, i)

// Index nested loop: an index on inner.key
for o in outer:                                   // N iterations
    for i in innerIndex.lookup(o.key):            // a B+tree descent: ~log M, often 1–3 cached pages
        emit(o, i)

// Hash join
table = HashTable()
for b in build:  table.insert(b.key, b)            // one pass over the smaller input (needs memory)
for p in probe:                                    // one pass over the larger input
    for b in table.lookup(p.key): emit(b, p)       // O(1) per lookup on average

// Sort-merge join
L = sortByKey(left); R = sortByKey(right)          // skipped if inputs already arrive sorted
i = 0; j = 0
while i < len(L) and j < len(R):
    if   L[i].key < R[j].key: i++
    elif L[i].key > R[j].key: j++
    else: emit every pair of L rows and R rows sharing this key; advance past them
```

| Algorithm | Cost (roughly) | Needs | Wins when | Notes |
|---|---|---|---|---|
| Nested loop (no index) | N × M | nothing | one side is tiny (a few rows) | the only option for arbitrary join conditions (`<`, `LIKE`) without other help |
| Index nested loop | N × (index lookup) | an index on the inner key | the outer side is small (up to thousands of rows) | pipelined; starts returning rows immediately; good with `LIMIT` |
| Hash join | N + M (plus building the table) | memory for the build side; an equality condition | both sides large, no useful order | blocking on the build side; spills to disk in batches if the table exceeds memory |
| Sort-merge join | N log N + M log M, or N + M if already sorted | sortable keys | both sides large and already sorted (from indexes), or the output must be sorted | equality (and some range) conditions |

MySQL implements nested-loop joins (with block buffering) and, since 8.0.18, hash joins; it has no merge join. PostgreSQL has all three.

### Concrete: the same join, two sizes

```text
 orders: 10,000,000 rows (index on customer_id); customers: 500,000 rows (primary key id)

 A) the 20 most recent orders with their customers' names
    index nested loop: 20 × (one primary-key lookup ≈ 3 cached pages + 1 leaf read) → ~80 page accesses
    hash join: read and hash all 500,000 customers (~4,000 pages) to join 20 rows → far more work
    → nested loop wins

 B) total spending of every customer over all orders
    index nested loop: 10,000,000 primary-key lookups → ~30–40 million page accesses
    hash join: hash 500,000 customers (~4,000 pages, a few tens of MB of memory), scan 83,334 order pages once
    → hash join wins by orders of magnitude
```

The crossover between them depends on the outer row count, which is again an **estimate**. A nested loop chosen because the planner expected 1 outer row, when there are 48,000, is the most common shape of a disastrous plan (Section 16.12, Plan 2).

### Mechanism: join ordering

```sql
-- revenue from keyboards
SELECT SUM(l.quantity * l.unit_cents)
FROM products p
JOIN order_lines l ON l.product_id = p.id
JOIN orders o      ON o.id = l.order_id
WHERE p.category = 'Keyboards' AND o.status = 'PAID';
```

```text
 sizes: products 10,000 (100 are keyboards); order_lines 40,000,000; orders 10,000,000 (62% PAID)

 order 1: products(filtered: 100) ⋈ order_lines (index on product_id: ~400,000 lines) ⋈ orders (400,000 lookups)
          intermediate results: 100 → 400,000 → ~248,000
 order 2: orders(filtered: 6,200,000) ⋈ order_lines (~24,800,000) ⋈ products (filter at the end)
          intermediate results: 6,200,000 → 24,800,000 → ~248,000
```

Both orders return the same result. The first carries at most 400,000 rows through the plan; the second carries 24.8 million. The general principle is to **apply the most selective filters first** and join outward from them, keeping intermediate results small. The optimiser finds this from its estimates; if it believed `category = 'Keyboards'` matched half the products, it would have no reason to start there.

### Guarantees and non-guarantees

Every join algorithm returns the same rows. Hash joins and nested loops do not guarantee output order; merge joins produce output in key order. No algorithm guarantees speed when its input estimates are wrong. Outer joins restrict which orders are legal (the preserved side must be kept), so not every reordering is available.

### Failure and diagnosis

- **A nested loop with thousands of loops.** Evidence: an inner node with `loops=48213` in `EXPLAIN ANALYZE`, under an outer node estimated at `rows=1`. Cause: an underestimate. Fix: statistics (Section 16.9).
- **A hash join spilling to disk.** Evidence: `Batches: 8` (more than 1) on the `Hash` node; temporary-file writes. Fix: more memory for that query (`SET LOCAL work_mem`), fewer or narrower columns in the build side, or a better plan.
- **A huge intermediate result.** Evidence: a join node producing millions of rows that a later filter removes. Fix: make the selective filter usable early (an index), or rewrite the query so the filter is applied before the join.

### Backend connection

Know which join your important queries use and why: index nested loops for small, `LIMIT`-ed, request-path joins (which need indexes on join keys, including foreign keys), hash joins for reports over large sets (which need memory). Most "the database suddenly got slow" incidents on join queries are a nested loop chosen for an input that is not small.

---

## 16.12 Reading EXPLAIN and EXPLAIN ANALYZE

### Definition

`EXPLAIN` shows the physical plan the optimiser chose, with its **estimates**: cost and rows for every node. `EXPLAIN ANALYZE` also **runs** the query and adds the **actual** time, rows and loops for every node. In PostgreSQL, `EXPLAIN (ANALYZE, BUFFERS)` adds the pages each node found in the buffer pool (`hit`) and read from the OS or disk (`read`).

### Why it matters

`EXPLAIN ANALYZE` is the only direct evidence of where a query spends its time and why the planner chose its plan. Reading it turns "the query is slow" into "the planner expected 1 row from this index scan, got 48,213, and therefore ran the inner index scan 48,213 times".

`EXPLAIN ANALYZE` executes the statement. For `INSERT`, `UPDATE` or `DELETE`, run it inside a transaction and roll back:

```sql
BEGIN;
EXPLAIN ANALYZE DELETE FROM sessions WHERE expires_at < now() - interval '30 days';
ROLLBACK;
```

### Mechanism: how to read a plan

```text
 Node Type on relation  (cost=STARTUP..TOTAL rows=EST_ROWS width=BYTES) (actual time=STARTUP..TOTAL rows=ROWS loops=N)
   Condition lines (Index Cond, Filter, Hash Cond, Join Filter …)
   Detail lines (Rows Removed by Filter, Buffers, Sort Method, Batches …)
   ->  child node …
```

1. **It is a tree; read it from the innermost, most indented nodes outward.** Children produce rows for their parent.
2. **`cost=a..b`** is estimated, in the cost units of Section 16.9: `a` until the first row, `b` for all rows. Compare costs only between plans for the same query.
3. **`rows`** in the first parentheses is the estimate; in the second, the actual number **per loop**.
4. **`actual time=a..b`** is in milliseconds, **per loop**. A node's total time is roughly `b × loops`.
5. **Look for the biggest mismatch between estimated and actual rows.** Usually the slow part of the plan sits directly above it.
6. **Look for where the time goes**: the node whose total time (× loops) accounts for most of the execution time, after subtracting its children's times.
7. **`Rows Removed by Filter`** large means the access method read many rows that a filter then discarded: a missing or mis-ordered index.

### Plan 1: a sequential scan, then an index scan

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 81;
```

```text
 Seq Scan on orders  (cost=0.00..208334.00 rows=22 width=64) (actual time=0.031..1123.442 rows=104 loops=1)
   Filter: (customer_id = 81)
   Rows Removed by Filter: 9999896
 Planning Time: 0.112 ms
 Execution Time: 1123.501 ms
```

Reading it: one node, a sequential scan. Its estimated cost, 208,334, is exactly the arithmetic of Section 16.9 (83,334 pages + 10 million rows × 0.0125). It examined 10 million rows to keep 104 (`Rows Removed by Filter: 9,999,896`): 99.999% wasted work. The estimate (22) is off by about 5×, but either number is tiny compared with the table. There is no index on `customer_id`.

```sql
CREATE INDEX CONCURRENTLY orders_customer_id_idx ON orders (customer_id);
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 81;
```

```text
 Index Scan using orders_customer_id_idx on orders  (cost=0.43..92.81 rows=22 width=64) (actual time=0.025..0.412 rows=104 loops=1)
   Index Cond: (customer_id = 81)
 Planning Time: 0.140 ms
 Execution Time: 0.451 ms
```

Now the condition is an `Index Cond` (used to seek), not a `Filter` (applied after reading), and no rows are removed. Execution time falls from 1,123 ms to 0.45 ms, about 2,500 times faster, and it will stay near 0.45 ms as the table grows.

### Plan 2: a bad estimate that chooses the wrong join

A new order status, `'REVIEW'`, was introduced last week. Since then 48,213 orders have entered it, but `orders` (10 million rows) has not been analysed because 0.5% of its rows changing is far below the auto-analyse threshold.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, e.payload
FROM orders o JOIN order_events e ON e.order_id = o.id
WHERE o.status = 'REVIEW';
```

```text
 Nested Loop  (cost=0.87..16.92 rows=1 width=72) (actual time=0.041..2841.507 rows=192852 loops=1)
   Buffers: shared hit=402118 read=61273
   ->  Index Scan using orders_status_idx on orders o  (cost=0.43..8.45 rows=1 width=8) (actual time=0.019..412.330 rows=48213 loops=1)
         Index Cond: (status = 'REVIEW'::text)
   ->  Index Scan using order_events_order_id_idx on order_events e  (cost=0.44..8.46 rows=1 width=72) (actual time=0.009..0.048 rows=4 loops=48213)
         Index Cond: (order_id = o.id)
 Planning Time: 0.201 ms
 Execution Time: 2860.114 ms
```

Reading it, inside out:

1. The outer index scan on `orders` was **estimated at 1 row** and produced **48,213**. `'REVIEW'` does not exist in the statistics, so the planner assumed the minimum. This is the mismatch.
2. Trusting "1 row", a nested loop looked perfect: one inner lookup. The inner index scan therefore ran **48,213 times** (`loops=48213`), each about 0.048 ms: 48,213 × 0.048 ≈ 2,300 ms of the total.
3. The outer scan itself took 412 ms: 48,213 index entries fetched in index order from scattered heap pages (`read=61273` pages from outside the buffer pool across the plan).

Fix the statistics, and the plan changes:

```sql
ANALYZE orders;
```

```text
 Hash Join  (cost=60512.13..141820.77 rows=191004 width=72) (actual time=389.207..702.954 rows=192852 loops=1)
   Hash Cond: (e.order_id = o.id)
   ->  Seq Scan on order_events e  (cost=0.00..38534.00 rows=2000000 width=72) (actual time=0.011..142.880 rows=2000000 loops=1)
   ->  Hash  (cost=59909.47..59909.47 rows=48213 width=8) (actual time=388.902..388.903 rows=48213 loops=1)
         Buckets: 65536  Batches: 1  Memory Usage: 2398kB
         ->  Bitmap Heap Scan on orders o  (cost=903.11..59909.47 rows=48213 width=8) (actual time=12.611..371.044 rows=48213 loops=1)
               Recheck Cond: (status = 'REVIEW'::text)
               Heap Blocks: exact=41877
               ->  Bitmap Index Scan on orders_status_idx  (cost=0.00..891.06 rows=48213 width=0) (actual time=7.951..7.951 rows=48213 loops=1)
                     Index Cond: (status = 'REVIEW'::text)
 Planning Time: 0.233 ms
 Execution Time: 712.408 ms
```

With correct estimates: a bitmap scan collects the 48,213 orders and reads their 41,877 pages in physical order; their IDs go into an in-memory hash table (2.4 MB, one batch, no spill); the 2 million events are scanned once and probed against it. 2,860 ms becomes 712 ms, and the plan now scales with the data instead of with the error. The lasting fix is to keep statistics fresh for this table (a lower per-table auto-analyse threshold), not to run `ANALYZE` by hand after each incident.

### Plan 3: a report with a join, aggregation, sort and spill

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.id, c.name, SUM(o.total_cents) AS spent
FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.created_at >= now() - interval '30 days'
GROUP BY c.id, c.name
ORDER BY spent DESC
LIMIT 10;
```

```text
 Limit  (actual time=1290.112..1290.118 rows=10 loops=1)
   ->  Sort  (actual time=1290.110..1290.114 rows=10 loops=1)
         Sort Key: (sum(o.total_cents)) DESC
         Sort Method: top-N heapsort  Memory: 26kB
         ->  HashAggregate  (actual time=1179.402..1251.877 rows=287441 loops=1)
               Group Key: c.id
               Batches: 5  Memory Usage: 8241kB  Disk Usage: 38960kB
               ->  Hash Join  (actual time=161.774..903.215 rows=1230088 loops=1)
                     Hash Cond: (o.customer_id = c.id)
                     ->  Index Scan using orders_created_at_idx on orders o  (actual time=0.022..421.906 rows=1230088 loops=1)
                           Index Cond: (created_at >= (now() - '30 days'::interval))
                           Buffers: shared hit=18211 read=9917
                     ->  Hash  (actual time=160.908..160.909 rows=500000 loops=1)
                           Buckets: 262144  Batches: 4  Memory Usage: 8193kB
                           ->  Seq Scan on customers c  (actual time=0.010..61.540 rows=500000 loops=1)
 Planning Time: 0.402 ms
 Execution Time: 1296.310 ms
```

(Estimate columns omitted for width.) Reading it:

1. **Bottom right:** all 500,000 customers are scanned and hashed. `Batches: 4` means the hash table did not fit in the memory allowed (`work_mem`, Chapter 15, Section 15.1), so it was split and partly written to temporary files.
2. **Bottom left:** an index scan on `created_at` finds 1.23 million recent orders; `read=9917` pages came from outside the buffer pool, so part of the time is I/O.
3. **Hash join:** about 900 ms to produce 1.23 million joined rows.
4. **HashAggregate:** groups them into 287,441 customers. Its hash table needed far more than the memory allowed, so it also spilled (`Batches: 5`, about 39 MB written to temporary files). The hash join's table and the aggregate's table are each limited separately, to `work_mem` times `hash_mem_multiplier` (4 MB × 2 = 8 MB with the defaults at the time of writing), which is why both stop at about 8 MB in memory. `GROUP BY c.id` alone would suffice, since `c.name` depends on the primary key (PostgreSQL accepts this).
5. **Sort with top-N heapsort:** because of `LIMIT 10`, the sort keeps only the best 10 in a 26 KB heap instead of sorting all 287,441 rows.

Options, in order of impact: maintain a daily per-customer summary table or materialised view for this dashboard (Chapter 14, Section 14.11) so the query reads thousands of rows instead of millions; or give this reporting query more memory (`SET LOCAL work_mem = '64MB'` inside its transaction) to remove both spills; and run it on a replica (Chapter 15, Section 15.7; Chapter 22).

### The MySQL equivalents

MySQL's traditional `EXPLAIN` prints one row per table access:

```text
 +----+-------------+--------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
 | id | select_type | table  | partitions | type | possible_keys   | key             | key_len | ref   | rows | filtered | Extra |
 +----+-------------+--------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
 |  1 | SIMPLE      | orders | NULL       | ref  | idx_customer_id | idx_customer_id | 8       | const |  104 |   100.00 | NULL  |
 +----+-------------+--------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
```

`type = ref` with `key = idx_customer_id` is an index lookup; `type = ALL` would be a full scan; `rows` is the estimate. Since MySQL 8.0.18, `EXPLAIN ANALYZE` prints a tree with estimated and actual rows and times per node, read the same way as PostgreSQL's.

### Guarantees and non-guarantees

`EXPLAIN` shows the plan the optimiser would choose now, for these literal values. `EXPLAIN ANALYZE` shows what that plan actually did, this time, with this cache state. Neither guarantees that the application's execution uses the same plan (prepared statements may use a generic plan, Section 16.8; parameters differ) or the same timings (cold versus warm cache; concurrency). Timing every row also adds overhead to the measured query.

### Failure and diagnosis

- **"It's fast when I run it."** The console run used a warm cache, literal values or a custom plan; the application uses a cold path, other values or a generic plan. Evidence: capture the application's actual plans with PostgreSQL's `auto_explain` module, which logs the plans of statements slower than a threshold.
- **Misreading loops.** A node showing `actual time=0.048` that runs 48,213 times is not cheap. Always multiply by `loops`.

### Backend connection

Make `EXPLAIN (ANALYZE, BUFFERS)` the first step of every slow-query investigation, and read it in the same order each time: estimated versus actual rows, time per node times loops, rows removed by filters, buffers read. The answer is almost always visible in one or two lines.

---

## 16.13 Query Anti-Patterns

### Definition

A **query anti-pattern** is a way of writing a query that prevents the database from using an index it could otherwise use, or makes it do far more work than the result requires. Many come down to one idea: a condition is **sargable** (from "search argument able") when it compares an indexed column, as it is stored, against a value the database can compute once. Wrapping the column in a function, converting its type or searching from the middle of its value makes the condition non-sargable, and the index cannot seek on it.

### Why it matters

These patterns look harmless and pass every test on small data. On a table of millions of rows each one turns an index seek into a full scan, and they are among the most common causes of slow queries in production code, often introduced by ORM-generated SQL or a convenient function call.

### Concrete: before and after

**1. A function or arithmetic on the indexed column.**

```sql
-- before: the index on created_at holds timestamps, not dates; every row's date must be computed
SELECT id FROM orders WHERE created_at::date = '2026-10-21';             -- PostgreSQL
SELECT id FROM orders WHERE DATE(created_at) = '2026-10-21';             -- MySQL
-- after: a range on the raw column (sargable), with explicit time-zone boundaries (Chapter 14, Section 14.12)
SELECT id FROM orders
WHERE created_at >= timestamptz '2026-10-21 00:00 Europe/Berlin'
  AND created_at <  timestamptz '2026-10-22 00:00 Europe/Berlin';
```

```text
 before:  Seq Scan on orders  (actual time=0.03..1301.7 rows=13702 loops=1)
            Filter: ((created_at)::date = '2026-10-21'::date)   Rows Removed by Filter: 9986298
 after:   Index Scan using orders_created_at_idx on orders  (actual time=0.02..9.8 rows=13702 loops=1)
            Index Cond: ((created_at >= …) AND (created_at < …))
```

The same applies to arithmetic: `WHERE total_cents / 100 > 50` cannot use an index on `total_cents`; `WHERE total_cents > 5000` can. Where the transformation is inherent to the query (`lower(email)`), index the expression instead (Section 16.5).

**2. Implicit type conversion.**

```sql
-- MySQL: phone is VARCHAR; comparing it with a NUMBER converts every row's phone to a number
SELECT id FROM customers WHERE phone = 4915112345678;        -- before: full scan (and odd matches)
SELECT id FROM customers WHERE phone = '4915112345678';      -- after: index lookup
```

In PostgreSQL the classic case is a driver binding a parameter with a different type from the column, for example a `numeric` value against a `bigint` column: the comparison is then done in `numeric`, the column side is converted, and an index on the `bigint` column cannot be used for it. In MySQL, joining columns with different character sets or collations (a `utf8mb3` column to a `utf8mb4` column) forces a conversion on one side and can disable the index for the join. Evidence in both: the plan shows a cast around the column (`(customer_id)::numeric`, or `convert(…)`). Fix: make types match exactly, in the schema and in parameters.

**3. A leading wildcard.**

```sql
-- before: no B+tree can seek to "anything ending in board"
SELECT id FROM products WHERE name LIKE '%board%';
-- after (PostgreSQL): a trigram GIN index makes substring search indexable (Section 16.6)
CREATE INDEX products_name_trgm ON products USING gin (name gin_trgm_ops);
-- or: full-text search for words; or an expression index on reverse(name) for pure suffix searches
```

A **prefix** search, `LIKE 'board%'`, can use a B+tree, because matching values are contiguous in sorted order. In PostgreSQL with a non-C collation, the ordinary B+tree on a text column cannot be used for `LIKE` prefixes; an index declared with `text_pattern_ops` (or the C collation) can.

**4. `OR` across different columns.**

```sql
-- before: one index cannot serve both branches; the planner may choose a full scan
SELECT id FROM orders WHERE customer_id = 81 OR guest_email = 'ana@x.com';
```

PostgreSQL can combine two indexes with a `BitmapOr` (Section 16.10), and MySQL with an index merge, but only when each branch has an index and the estimates favour it. When they do not, rewrite as a union of two indexable queries:

```sql
SELECT id FROM orders WHERE customer_id = 81
UNION                                             -- UNION, not UNION ALL: an order matching both appears once
SELECT id FROM orders WHERE guest_email = 'ana@x.com';
```

**5. `SELECT *`.**

```sql
-- before: needs every column → no index-only scan; reads large TOASTed columns; more network and memory
SELECT * FROM orders WHERE customer_id = 81 ORDER BY created_at DESC LIMIT 20;
-- after: only what the screen shows, which a covering index can serve (Section 16.5)
SELECT id, status, total_cents, created_at FROM orders WHERE customer_id = 81 ORDER BY created_at DESC LIMIT 20;
```

```text
 before:  Index Scan using orders_customer_created on orders (… rows=20) + 20 heap fetches, wide rows
 after:   Index Only Scan using orders_customer_created_cover on orders (… rows=20)  Heap Fetches: 0
```

`SELECT *` also couples code to the table's column list: adding a 2 MB JSON column to the table silently makes every such query heavier.

**6. Offset pagination.** Chapter 13 (Section 13.6) explained the semantics; the plan shows the cost:

```text
 OFFSET 100000 LIMIT 20:
   Limit  (actual rows=20 loops=1)
     ->  Index Scan using orders_customer_created on orders  (actual rows=100020 loops=1)   ← read 100,020 to return 20
 keyset (WHERE (created_at, id) < ($1, $2) … LIMIT 20):
   Limit  (actual rows=20 loops=1)
     ->  Index Scan using orders_customer_created on orders  (actual rows=20 loops=1)       ← read 20
```

### Guarantees and non-guarantees

A sargable condition makes an index usable; it does not oblige the planner to use it (Section 16.9). Rewrites must preserve meaning: the date-range rewrite must use the same time zone the original cast used, and an `OR` rewritten as `UNION ALL` instead of `UNION` would duplicate rows that match both branches.

### Failure and diagnosis

Every one of these shows the same evidence: `EXPLAIN ANALYZE` with a `Filter` (not an `Index Cond`) on the column you expected to be indexed, and a large `Rows Removed by Filter`, or a cast or function wrapped around the column in the plan. Search the codebase for the patterns: functions applied to columns in `WHERE`, `LIKE '%…`, `OFFSET` on large tables, `SELECT *` in hot paths.

### Backend connection

Write predicates against columns as they are stored, with parameters of the same type; move transformations to the value side or into an expression index; select only needed columns; paginate by key. Add a review checklist for these six patterns, because ORMs and helper functions produce them by default.

---

## 16.14 Table Partitioning and Partition Pruning

### Definitions

**Table partitioning** splits one logical table into several physical tables, the **partitions**, by the value of a **partition key**, inside a single database server. Queries and DML address the parent table; the database routes rows to partitions.

- **Range partitioning**: each partition holds a range of key values (one partition per month of `created_at`).
- **List partitioning**: each partition holds listed values (one per `region`).
- **Hash partitioning**: a hash of the key decides the partition (spreading rows evenly over N partitions).

**Partition pruning** is the planner or executor skipping partitions that cannot contain rows matching the query. Partitioning within one server is not **sharding**, which spreads data across servers (Chapter 23).

### Why it exists

Some tables grow without bound: events, logs, payments history. At hundreds of millions or billions of rows, three operations become painful: deleting old data (a `DELETE` of 100 million rows writes 100 million tuple deletions, WAL for each, and leaves bloat for VACUUM), maintaining the table (vacuuming or rebuilding indexes on one enormous table), and querying recent data through indexes that cover all history. Partitioning by the column that queries and retention both use solves all three.

### Mechanism: a range-partitioned table

```sql
-- PostgreSQL declarative partitioning
CREATE TABLE events (
    id         bigint      NOT NULL,
    created_at timestamptz NOT NULL,
    kind       text        NOT NULL,
    payload    jsonb,
    PRIMARY KEY (id, created_at)            -- must include the partition key
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_09 PARTITION OF events FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
CREATE TABLE events_2026_10 PARTITION OF events FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
CREATE INDEX ON events (kind, created_at);  -- created on every partition, each index local to it
```

```text
 events (parent: no rows of its own)
 ├── events_2026_08   [Aug 1, Sep 1)   140 M rows   its own heap and indexes
 ├── events_2026_09   [Sep 1, Oct 1)   150 M rows
 └── events_2026_10   [Oct 1, Nov 1)    95 M rows (current, receives all inserts)
```

**Pruning** in a plan:

```sql
EXPLAIN SELECT count(*) FROM events WHERE created_at >= '2026-10-20' AND kind = 'checkout';
```

```text
 Aggregate
   ->  Index Only Scan using events_2026_10_kind_created_at_idx on events_2026_10 events
         Index Cond: ((kind = 'checkout'::text) AND (created_at >= '2026-10-20 …'))
 (only one partition appears: the other partitions were pruned at planning time)
```

When the boundary value is not known until execution (a parameter, or `now()`), PostgreSQL (11+) prunes at **run time**, and `EXPLAIN ANALYZE` reports how many partitions were skipped (`Subplans Removed: 23`).

**Retention** becomes a metadata operation:

```sql
-- instead of DELETE FROM events WHERE created_at < '2026-09-01' (140 M row deletions, WAL, bloat):
ALTER TABLE events DETACH PARTITION events_2026_08;   -- PostgreSQL 14+ can do this CONCURRENTLY
DROP TABLE events_2026_08;                            -- frees the files at once; little WAL
```

### Mechanism: what partitioning costs

| Effect | Consequence |
|---|---|
| Queries without the partition key in `WHERE` | probe every partition (one index lookup per partition): slower than one large index for selective lookups |
| Unique constraints and primary keys | must include the partition key (PostgreSQL and MySQL), because uniqueness is checked per partition: `id` alone cannot be guaranteed unique across partitions |
| Many partitions | planning time and memory grow with the number of partitions a query might touch; hundreds are typical, many thousands problematic |
| Updating the partition key | moves the row to another partition (a delete plus an insert) |
| Partition maintenance | someone must create future partitions before data arrives (or a default partition catches strays), usually a scheduled job |
| Hash partitioning | spreads writes evenly, but prunes only for equality on the key, and cannot drop "old" data by partition |

### Guarantees and non-guarantees

Partitioning guarantees that each row lives in exactly one partition, chosen by its key, and that queries whose conditions constrain the key can skip the rest. It does not make queries that ignore the key faster (usually slower), does not spread load across machines (that is sharding), and does not enforce uniqueness on columns that exclude the key.

### Failure and diagnosis

- **Inserts fail at the start of a month.** `no partition of relation "events" found for row`. Cause: the next partition was never created. Fix: create partitions ahead of time automatically; add a default partition with an alert.
- **Partitioning made lookups slower.** Evidence: plans showing an `Append` over every partition for queries by `id`. Cause: the main lookup key is not the partition key. Fix: choose the partition key from the dominant query and retention pattern, or keep a non-partitioned lookup table.
- **Planning time grew.** Evidence: planning time comparable to execution time for simple queries on a table with thousands of partitions. Fix: fewer, larger partitions.

### Backend connection

Partition tables that grow forever and are queried and expired by time: events, logs, audit trails, metrics, old orders. Choose the partition key from both the query pattern and the retention policy, automate partition creation, and expire data by dropping partitions instead of deleting rows. Do not partition a table merely because it is large if its queries do not filter on a natural partition key.

---

## 16.15 Placement-Round Topics: Textbook Indexing, B-Tree Exercises, Hashing and Query-Cost Formulas

> **Placement-round topics.** This section is for campus placement rounds, university DBMS exams and written tests, where these calculations are asked often. Product-company backend interviews rarely ask them. **Give your main focus to Sections 16.1–16.14**; read this section before placement rounds and come back to it for revision.

### Primary, clustering and secondary indexes; dense and sparse

Exams use a classification from file-organisation textbooks:

- A **primary index** is built on the ordering **key** field of a file that is physically sorted by that key. It can be **sparse**: one entry per **block** (the first key in each block), because the file's order locates the rest.
- A **clustering index** is built on an ordering field that is **not** unique: one entry per distinct value, pointing to the first block holding it.
- A **secondary index** is built on a field by which the file is **not** ordered. It must be **dense**: one entry per **record** (or per distinct value, with a list of pointers).
- A **dense** index has an entry for every search-key value; a **sparse** index has entries for only some (one per block).
- A **multilevel index** indexes the index itself: the first level is a sorted file of index entries, the second level a sparse index on it, and so on until one block remains. A B+tree is a dynamic multilevel index.

### Worked example: block accesses

A file of r = 30,000 records, record size R = 100 bytes, block size B = 1,024 bytes, sorted by a key of V = 9 bytes; a block pointer is P = 6 bytes.

```text
 blocking factor  bfr = ⌊1024 / 100⌋ = 10 records per block → b = 30,000 / 10 = 3,000 blocks
 binary search on the data file:          ⌈log₂ 3,000⌉ = 12 block accesses

 primary (sparse) index: entry = V + P = 15 bytes → bfrᵢ = ⌊1024 / 15⌋ = 68 entries per block
   index entries = one per data block = 3,000 → index blocks = ⌈3,000 / 68⌉ = 45
   binary search on the index: ⌈log₂ 45⌉ = 6, plus 1 access to the data block = 7 block accesses

 multilevel index (fan-out fo = 68):
   level 1: 45 blocks;  level 2: ⌈45 / 68⌉ = 1 block (top)
   accesses: one per level + 1 data block = 2 + 1 = 3
```

### B-tree versus B+tree

| | B-tree | B+tree |
|---|---|---|
| Where data pointers live | in every node, internal and leaf | only in leaves |
| Internal nodes | keys with data pointers and child pointers | separator keys and child pointers only |
| Fan-out | lower (internal nodes carry data pointers) | higher → shorter tree |
| Leaves linked | no | yes → efficient range scans |
| A key appears | once | possibly twice (as a separator and in a leaf) |
| Search can end | at an internal node | always at a leaf |

In a B+tree of **order p** (textbook convention; definitions vary between books, so state yours), an internal node has at most p children and at least ⌈p/2⌉ (the root at least 2), and every leaf is at the same level and at least half full.

### Worked example: building a B+tree

Leaves hold at most 3 keys; internal nodes hold at most 3 keys. Insert 10, 20, 30, 40, 50, 60, 70, 80, 90, 100. When a leaf splits, the first key of the new right leaf is **copied** up; when an internal node splits, its middle key is **moved** up.

```text
 insert 10, 20, 30            [10 20 30]
 insert 40 → leaf overflows   [10 20 30 40] → split → [10 20] [30 40], copy 30 up → root [30]
 insert 50                    [10 20] [30 40 50]
 insert 60 → overflow         [30 40 50 60] → [30 40] [50 60], copy 50 up → root [30 50]
 insert 70                    … [50 60 70]
 insert 80 → overflow         [50 60 70 80] → [50 60] [70 80], copy 70 up → root [30 50 70]
 insert 90                    … [70 80 90]
 insert 100 → leaf overflow   [70 80 90 100] → [70 80] [90 100], copy 90 up → root [30 50 70 90]: overflows
            → internal split: move the middle key 70 up; left [30 50], right [90]

 final tree (height 3):
                    [70]
              ┌──────┴──────┐
           [30 50]         [90]
          ┌───┼───┐       ┌─┴──┐
     [10 20][30 40][50 60][70 80][90 100]      (leaves linked left to right)
```

Note how increasing keys left every leaf holding only 2 of its 3 keys, with the left halves of earlier splits never filled again: textbook B+trees split evenly, while real engines optimise rightmost splits for this pattern and leave the left page full (Section 16.2).

### Hashing: static, extendible and linear

- **Static hashing**: M buckets; bucket = h(key) mod M. When buckets fill, **overflow chains** grow and lookups slow down; changing M means rehashing everything.
- **Extendible hashing**: a **directory** of 2^d pointers, where d is the **global depth**, indexed by the last (or first) d bits of the hash. Each bucket has a **local depth** ≤ d. When a bucket overflows: if its local depth < d, split it and redistribute its keys by one more bit, updating the directory pointers; if its local depth = d, first **double the directory** (d + 1), then split. Only the overflowing bucket is rehashed.
- **Linear hashing**: no directory. Buckets are split one at a time in a fixed round-robin order, tracked by a **split pointer**, whenever a load threshold is exceeded, regardless of which bucket overflowed. Keys are addressed with h mod M or h mod 2M depending on whether their bucket has been split in the current round.

```text
 extendible hashing, global depth 2, bucket capacity 2 (directory indexed by the last 2 bits)
 directory 00 → A [4, 8]      01 → B [5]      10 → C [6]      11 → B   (B has local depth 1: shared by 01 and 11)
 insert 12 (…1100, last bits 00) → A overflows; A's local depth 2 = global depth 2
   → double the directory to depth 3 (000…111); split A by the 3rd bit:
     000 → A [8]  (…000)      100 → A' [4, 12]  (…100)      other entries point as before
```

### Heuristic optimisation and equivalence rules

Relational algebra expressions (Chapter 14, Section 14.14) can be rewritten without changing their result. Key **equivalence rules**: a conjunction of selections can be split into a cascade (σ<sub>a ∧ b</sub>(R) = σ<sub>a</sub>(σ<sub>b</sub>(R))); selections commute; a selection over a join can be pushed to the side whose attributes it uses (σ<sub>c</sub>(R ⋈ S) = σ<sub>c</sub>(R) ⋈ S when c refers only to R); joins are commutative and associative; projections can be pushed down, keeping the attributes later operators need.

**Heuristic optimisation** applies them in a standard order: perform selections as early as possible, then projections, combine a cross product followed by a selection into a join, and perform the most restrictive joins first. Section 16.11's keyboard example (filter products first) is this heuristic; cost-based optimisers (Section 16.9) choose among the alternatives by estimated cost instead of fixed rules.

### Join cost formulas

Textbooks count **block transfers**. Let relation r have n<sub>r</sub> tuples in b<sub>r</sub> blocks and s have n<sub>s</sub> tuples in b<sub>s</sub> blocks, with r the outer relation:

| Algorithm | Block transfers (worst case, little memory) |
|---|---|
| Tuple nested loop | n<sub>r</sub> × b<sub>s</sub> + b<sub>r</sub> |
| Block nested loop | b<sub>r</sub> × b<sub>s</sub> + b<sub>r</sub> |
| Indexed nested loop | b<sub>r</sub> + n<sub>r</sub> × c (c = cost of one index lookup and fetch) |
| Merge join (inputs sorted) | b<sub>r</sub> + b<sub>s</sub> (plus sorting costs if not sorted) |
| Hash join (with partitioning) | 3 × (b<sub>r</sub> + b<sub>s</sub>) |

Example: depositor (5,000 tuples, 100 blocks) as the outer relation and customer (10,000 tuples, 400 blocks):

```text
 tuple nested loop:  5,000 × 400 + 100   = 2,000,100 block transfers
 block nested loop:  100 × 400 + 100     =    40,100
 hash join:          3 × (100 + 400)     =     1,500
```

### External sorting

When a relation is larger than the memory available for sorting, databases use **external merge sort** (the `Sort Method: external merge` of a spilled sort in PostgreSQL). With M buffer blocks and a relation of b<sub>r</sub> blocks:

1. **Run creation:** read M blocks at a time, sort them in memory, write each as a sorted **run**: ⌈b<sub>r</sub> / M⌉ runs.
2. **Merging:** merge M − 1 runs at a time (one buffer for output) into longer runs; repeat until one run remains. Each merge pass reads and writes every block.

```text
 passes   = ⌈log_(M−1) (b_r / M)⌉
 transfers ≈ b_r × (2 × passes + 1)      (2 per pass, plus the reads of run creation; the final
                                          write is often not counted, since the output goes to the next operator)

 example: b_r = 10,000 blocks, M = 101
   runs:   ⌈10,000 / 101⌉ = 100 runs of 101 blocks
   passes: ⌈log_100 (100)⌉ = 1   (a 100-way merge finishes in one pass)
   cost:   10,000 × (2 × 1 + 1) = 30,000 block transfers
 with M = 11: 910 runs, ⌈log_10 (910)⌉ = 3 passes → 10,000 × 7 = 70,000 transfers
```

More memory means fewer runs and fewer passes, which is why raising `work_mem` for one report removes a spilled sort (Section 16.12).

### Drill for this section

**C1. Compute the block accesses to find a record by key with no index, with a sparse primary index, and with a multilevel index, for r = 30,000, R = 100 bytes, B = 1,024 bytes, V = 9, P = 6.**
A strong answer must mention:
- bfr = 10, b = 3,000; binary search ⌈log₂ 3,000⌉ = 12.
- Index entry 15 bytes, bfrᵢ = 68, 45 index blocks; ⌈log₂ 45⌉ + 1 = 7.
- Multilevel with fan-out 68: 2 levels + 1 = 3.
*Common weak answer:* forgetting the final access to the data block, or using a dense index count for a primary index.

**C2. Differences between a B-tree and a B+tree? Why do databases use B+trees?**
A strong answer must mention:
- Data pointers only in leaves; internal nodes are pure separators → higher fan-out, shorter trees.
- Linked leaves → efficient range scans and ordered output.
- Every search ends at a leaf, giving uniform cost.
*Common weak answer:* "B+tree is a B-tree with more keys." The structural differences are the point.

**C3. Insert a given sequence into a B+tree of given order and draw the result.**
A strong answer must mention:
- Leaf splits copy the first key of the new leaf up; internal splits move the middle key up.
- A root split increases the height.
- State the order convention used.
*Common weak answer:* moving (not copying) keys up on leaf splits, which loses keys from the leaf level.

**C4. Explain extendible hashing with an example.**
A strong answer must mention:
- Directory of 2^d pointers, global depth d, local depth per bucket.
- On overflow: split the bucket if local < global; otherwise double the directory first.
- Only the overflowing bucket is rehashed; no long overflow chains.
*Common weak answer:* describing static hashing with overflow chains.

**C5. How does external merge sort work, and what does it cost for b_r = 10,000 blocks and M = 101 buffers?**
A strong answer must mention:
- Run creation: sort M blocks at a time into ⌈b_r / M⌉ sorted runs; then (M − 1)-way merge passes until one run remains.
- Passes = ⌈log_(M−1)(b_r / M)⌉; each pass reads and writes every block.
- Here: 100 runs, one merge pass, about 10,000 × 3 = 30,000 block transfers.
- More memory, fewer passes: the reason a spilled sort speeds up with more `work_mem`.
*Common weak answer:* treating the sort as one in-memory pass, or forgetting that each merge pass both reads and writes the data.

---

## Chapter Summary

An **index** turns a linear scan into a logarithmic lookup. The **B+tree** does it with wide nodes (hundreds of keys per page), so three or four levels reach billions of rows and the upper levels stay cached; searches descend once, ranges walk linked leaves in order, inserts split full nodes and grow the tree at the root. Increasing keys append at the right edge; random keys (UUIDv4) scatter inserts, split pages everywhere and grow the index. In **InnoDB**, the table is a clustered B+tree on the primary key, and secondary indexes store the primary key, so a secondary lookup is two descents. In **PostgreSQL**, every index points to heap TIDs, and **index-only scans** skip the heap only where the visibility map says pages are all-visible.

**Composite indexes** seek on leftmost prefixes; put **equality columns first, then one range or sort column**, so a single range scan filters, orders and stops at `LIMIT`. **Covering**, **partial**, **expression** and **unique** indexes shape indexes to queries and rules; **GIN**, **GiST**, **BRIN**, hash and bitmap indexes answer questions B+trees cannot. Every index costs on **every write** (more entries, more WAL, more dirty pages) and in **space** that competes for the buffer pool; indexes often total more than the table, and unused ones are pure cost.

A query is **parsed**, **analysed**, **rewritten**, **planned** and **executed**, the executor pulling rows through a tree of iterators in which sorts and hash builds block and most else pipelines. The **optimiser** picks the physical plan with the lowest estimated cost, using **statistics** (most common values, histograms, distinct counts, correlation) to estimate **selectivity** and **cardinality**. Independence assumptions, sampling and **stale statistics** produce wrong estimates, and wrong estimates cross cost **crossovers** (an index stops paying around half a percent of scattered rows under default costs), producing **plan flips**. **Scans** range from index-only through index and bitmap to sequential; **joins** are nested loops (small outer inputs, with indexes), hash joins (large inputs, with memory) and merge joins (sorted inputs); **join order** keeps intermediate results small by applying selective filters first.

`EXPLAIN ANALYZE` shows the plan with estimated and actual rows, times and loops: read it inside out, find the biggest estimate error and the node where time goes, multiply by loops, and check rows removed by filters and buffers read. **Anti-patterns** (functions or casts on indexed columns, leading wildcards, cross-column `OR`, `SELECT *`, offset pagination) make conditions non-sargable or work excessive. **Partitioning** splits huge tables by key inside one server, so queries prune partitions and retention drops partitions instead of deleting rows, at the cost of key-less lookups and uniqueness rules.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| B+tree index | Lookups in height-many page reads; ordered range scans; balance under any insert order | Compactness under random inserts or deletes; use by the planner; help for non-leading columns |
| Clustered index (InnoDB) | Rows stored in key order; one descent by primary key | Cheap secondary lookups (two descents); narrow secondary indexes with a wide primary key |
| Index-only scan | Correct results from the index | No heap/table visits (depends on visibility) |
| Composite index | Seeks on leftmost prefixes; order by its columns | Seeks after the first range column or on non-leading columns |
| Partial / expression index | Small index for a subset / an index on a computed value | Use by queries that do not provably match the predicate / expression |
| Unique index | No duplicate keys under concurrency | Uniqueness of a different normalisation than indexed; a single `NULL` |
| GIN / GiST / BRIN | Fast answers for their operator families | Help outside them; cheap writes (GIN); usefulness on unordered data (BRIN) |
| Optimiser | A correct plan, cheapest by its estimates | Accurate estimates; stable plans; the same plan for all parameters |
| Statistics | A sampled picture as of the last `ANALYZE` | Freshness; awareness of correlated columns without extended statistics |
| Hash join | Linear-time equi-join of large inputs | Fitting in memory; output order |
| Nested-loop join | Fast for small outer inputs with an inner index | Speed when the outer input is large |
| `EXPLAIN ANALYZE` | What this plan did this time | The application's plan and cache state |
| Partitioning | Pruning when the key is constrained; cheap retention by dropping partitions | Faster key-less lookups; cross-partition uniqueness without the key; distribution across servers |

---

## Interview Drill

**Q1. How does a B+tree index work? Why not a binary tree or a hash table?**
A strong answer must mention:
- Nodes are pages with hundreds of sorted keys; internal nodes route, leaves hold entries; leaves are linked.
- Search descends one node per level; range scans walk the leaves in order; inserts split nodes and grow the tree at the root, keeping it balanced.
- A binary tree would be ~log₂ N levels (24 for 10 million), each a page read; a B+tree is 3–4.
- A hash table cannot serve ranges, prefixes or ordering.
*Common weak answer:* "It's a balanced tree that makes lookups O(log n)." True of binary trees too; the point is the fan-out and leaf links that suit pages.

**Q2. How many levels does an index on a billion rows have?**
A strong answer must mention:
- Fan-out from page size and entry size: ~400 for a `bigint` in an 8 KB page.
- 400³ = 64 million, 400⁴ = 25.6 billion: four levels for a billion.
- Upper levels (~3 MB) are cached: typically one or two device reads per lookup.
- Wider keys mean lower fan-out and more levels.
*Common weak answer:* "About 30, since log₂ of a billion is 30." That is a binary tree.

**Q3. Why not index every column?**
A strong answer must mention:
- Each index adds work to every insert, every delete's cleanup and many updates (all indexes on non-HOT updates in PostgreSQL), plus WAL and dirty pages.
- Space: indexes often exceed the table and compete for the buffer pool.
- Single-column indexes on low-selectivity columns are rarely used; composite indexes matched to queries are what help.
- Find and drop unused indexes from usage statistics.
*Common weak answer:* "It uses disk space." Space is the smaller cost; write amplification and cache pressure are the larger.

**Q4. Clustered versus non-clustered (secondary) indexes? What happens on a secondary-index lookup in InnoDB and in PostgreSQL?**
A strong answer must mention:
- Clustered: rows stored in the index leaves in key order; at most one per table; InnoDB clusters on the primary key.
- InnoDB secondary leaves hold the primary key → a second descent into the clustered index per row; wide primary keys inflate all secondary indexes.
- PostgreSQL: all indexes secondary, pointing to heap TIDs → a heap page read per row (fewer if correlated).
- Covering indexes and index-only scans avoid the second step.
*Common weak answer:* "Clustered means the index is stored with the table." Misses the double lookup and the PostgreSQL heap model.

**Q5. How should you order the columns of a composite index?**
A strong answer must mention:
- Entries are sorted by the first column, then the second; seeks need a leftmost prefix.
- Equality columns first, then a single range or sort column; columns after the first range only filter.
- Put the `ORDER BY` column right after the equality prefix to avoid a sort and stop at `LIMIT`.
- Choose the order to also serve other queries; `(a, b)` makes `(a)` redundant.
*Common weak answer:* "Most selective column first." Often wrong; the equality-then-range rule matters more.

**Q6. What is a covering index? What is an index-only scan?**
A strong answer must mention:
- An index containing every column the query reads, so the table need not be read.
- PostgreSQL `INCLUDE` adds non-key columns; InnoDB secondary indexes already include the primary key.
- PostgreSQL still visits the heap for pages not marked all-visible (`Heap Fetches`); depends on VACUUM.
- Trade-off: bigger index, more update work.
*Common weak answer:* "An index on all columns of the table." It is per query, and wider indexes have costs.

**Q7. What are partial and expression indexes for?**
A strong answer must mention:
- Partial: index only rows matching a predicate (pending jobs, active users); tiny and fast; used only when the query implies the predicate.
- Partial unique indexes encode rules (one active account per email; one successful payment per order).
- Expression: index a computed value (`lower(email)`); queries must use the same expression; functions must be immutable in PostgreSQL.
*Common weak answer:* "To save space." That is a side effect; the main value is matching queries and rules.

**Q8. When would you use GIN, GiST, BRIN or hash indexes?**
A strong answer must mention:
- GIN: inverted index for containment: full text, `jsonb @>`, arrays, trigram substring search; costly writes.
- GiST: overlap, containment, nearest neighbour on ranges and geometry; exclusion constraints against overlapping bookings.
- BRIN: tiny min/max per block range for huge, physically ordered tables (append-only by time).
- Hash: equality only; rarely better than a B+tree.
*Common weak answer:* "GIN is faster than B-tree." Each serves different operators.

**Q9. How does the query planner choose a plan? What are selectivity and cardinality?**
A strong answer must mention:
- Enumerates access paths, join orders and algorithms; picks the lowest estimated cost (dynamic programming, heuristics for many tables).
- Statistics (MCVs, histograms, n_distinct, correlation) → selectivity → estimated cardinality per operator.
- Cost model weights sequential and random page reads and CPU per row; example crossover ~0.5% scattered rows for index vs sequential scan under default costs.
- Independence assumption and compounding errors; extended statistics.
*Common weak answer:* "It uses an index if there is one." The decision is cost-based and depends on estimated row counts.

**Q10. Why did a query suddenly become slow without any code change?**
A strong answer must mention:
- A plan flip: data growth or a new value moved an estimate across a cost crossover.
- Stale statistics (large tables analysed rarely; new values; recent ranges beyond the histogram).
- Parameter-sensitive or generic plans for prepared statements.
- Evidence: estimated vs actual rows in `EXPLAIN ANALYZE`; `auto_explain`; fix with `ANALYZE`, analysis settings, extended statistics.
*Common weak answer:* "The database is overloaded." Possible, but a plan change is the first thing to check.

**Q11. Nested-loop versus hash versus merge join: when does each win?**
A strong answer must mention:
- Nested loop (with an inner index): small outer input; pipelined; good with `LIMIT`; disastrous when the outer input is underestimated.
- Hash join: large inputs, equality condition; linear; needs memory (spills in batches).
- Merge join: inputs already sorted (indexes) or sorted output needed.
- MySQL: nested loop and (8.0.18+) hash join, no merge join.
*Common weak answer:* "Hash join is always fastest." Not for a 20-row outer input with an index.

**Q12. How do you read `EXPLAIN` and `EXPLAIN ANALYZE`?**
A strong answer must mention:
- A tree read inside out; cost and rows are estimates; actual time and rows are per loop.
- Find the largest estimated-versus-actual row mismatch; multiply time by loops.
- `Index Cond` versus `Filter`, `Rows Removed by Filter`, `Batches`, `Buffers` hit/read.
- `EXPLAIN ANALYZE` runs the statement (roll back writes); console plans may differ from the application's.
*Common weak answer:* "Look at the total cost." Costs are estimates; actual rows and loops reveal the problem.

**Q13. Why does `WHERE lower(email) = …` or `LIKE '%term'` not use an index?**
A strong answer must mention:
- The index holds the raw values in order; a function of the column or a leading wildcard is not a range in that order (non-sargable).
- Fixes: an expression index, a range rewrite for dates, trigram GIN or full-text indexes for substrings.
- Type mismatches (implicit casts) have the same effect.
*Common weak answer:* "Indexes don't support functions." Expression indexes do; the point is matching what is indexed.

**Q14. What is table partitioning, and what is partition pruning?**
A strong answer must mention:
- Splitting a table by range, list or hash of a key into physical partitions within one server.
- Pruning skips partitions the conditions exclude, at plan time or run time.
- Benefits: cheap retention by dropping partitions, smaller indexes, per-partition maintenance.
- Costs: key-less lookups probe every partition; unique keys must include the partition key; planning overhead; not sharding.
*Common weak answer:* "Partitioning spreads data across servers." That is sharding (Chapter 23).

**Q15. Why do random UUID primary keys hurt insert performance on large tables?**
A strong answer must mention:
- Random keys land on random leaves: splits everywhere, pages around two-thirds full, a larger index.
- Every leaf becomes a candidate for the next insert: the whole index must be cached or inserts cause random reads.
- In InnoDB, the primary key is the table order, so the table itself suffers; secondary indexes carry the 16-byte key.
- Time-ordered keys (identity, UUIDv7) append at the right edge.
*Common weak answer:* "UUIDs are bigger than integers." Size matters less than insert locality.

**Q16. A query filters 30% of a large table. Why might the planner ignore your index, and is it right?**
A strong answer must mention:
- With scattered rows, each index match costs a random page read; reading 30% that way costs more than a sequential scan.
- The crossover depends on correlation and on `random_page_cost` (lower on SSDs).
- It is usually right; a covering index or a bitmap scan may change the answer.
*Common weak answer:* "The planner is broken; force the index." Forcing often makes it slower.

---

## Scenario Exercises

1. An endpoint `GET /customers/{id}/orders?status=PAID&sort=-created_at&limit=20` takes 800 ms on a 50-million-row `orders` table that has single-column indexes on `customer_id`, `status` and `created_at`. Design the index that makes it fast, say which existing indexes it makes redundant, and show the plan you expect.
2. After a weekend data migration added 30 million rows, a query that took 20 ms on Friday takes 40 seconds on Monday. No code changed. List the hypotheses, the evidence that separates them, and the immediate and lasting fixes.
3. A product team asks for "search anywhere in product names and descriptions" over 5 million products, currently implemented with `ILIKE '%term%'`. Compare trigram indexes, full-text search and a separate search engine, with their write costs.
4. A table has 14 indexes; write latency has doubled over six months, and the buffer pool hit ratio is falling. Describe how you would decide which indexes to drop, and what you would check before dropping each.
5. Your MySQL `orders` table uses a `CHAR(36)` UUIDv4 primary key and has six secondary indexes. Inserts slow down as the table passes 300 million rows. Explain why, with the clustered-index mechanism, and propose a migration path.
6. A billing system keeps 4 years of invoice line items (3 billion rows), deletes data older than 4 years each night with a `DELETE`, and suffers bloat and long vacuums. Design a partitioning scheme, and explain its impact on the primary key and on lookups by invoice ID.
7. A sales report joins `order_lines` to `products` with `WHERE p.brand = 'Acme' AND p.product_line = 'Acme Pro'` on a 200,000-product catalogue. `brand = 'Acme'` matches 1% of products and `product_line = 'Acme Pro'` 0.5%. `EXPLAIN ANALYZE` shows the products scan estimated at 10 rows, actual 1,000, followed by a nested loop whose inner index scan on `order_lines` runs 1,000 times and dominates the run time. Explain where the estimate of 10 comes from, why it is wrong, and two fixes, one for the estimate and one that makes the plan less sensitive to it.

---

## Diagnosis Lab

**Lab 1: The most expensive query is not the slowest one (pg_stat_statements)**
- *Symptom:* Database CPU is at 85% during peak, with no single query looking slow; the slowest queries in the logs run a few times an hour.
- *Hypotheses:* (a) a few slow reports; (b) many cheap queries; (c) background maintenance.
- *Evidence:*
  ```sql
  SELECT calls, round(total_exec_time) AS total_ms, round(mean_exec_time::numeric, 2) AS mean_ms, rows, left(query, 60) AS query
  FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 3;
  ```
  ```text
   calls    │ total_ms  │ mean_ms │ rows      │ query
   21480117 │ 39822004  │ 1.85    │ 21480117  │ SELECT id, status FROM carts WHERE session_token = $1
   412      │ 2381090   │ 5779.34 │ 412000    │ SELECT c.region, sum(...) FROM orders o JOIN ...
   9381552  │ 1690400   │ 0.18    │ 9381552   │ SELECT * FROM products WHERE id = $1
  ```
  The cart lookup runs 21 million times at 1.85 ms each: 39,800 seconds of execution, 17 times more than the slow report. `EXPLAIN ANALYZE` for it shows a sequential scan of a 400,000-row `carts` table: `session_token` has no index.
- *Root cause:* (b): a cheap-looking query multiplied by a huge call count.
- *Fix:* `CREATE UNIQUE INDEX CONCURRENTLY carts_session_token ON carts (session_token);` the mean time falls to about 0.05 ms and CPU to 30%.
- *Prevention:* Review `pg_stat_statements` ordered by **total** time (not mean) regularly; alert on new entries in the top ten.

**Lab 2: The slow-query log points at a non-sargable predicate (MySQL)**
- *Symptom:* The order-history page is slow for every user.
- *Evidence:* The slow-query log (statements over `long_query_time`) shows:
  ```text
  # Query_time: 3.512188  Lock_time: 0.000102  Rows_sent: 20  Rows_examined: 4012345
  SELECT id, total_cents FROM orders WHERE customer_id = 81 AND DATE(created_at) >= '2026-01-01' ORDER BY created_at DESC LIMIT 20;
  ```
  Twenty rows sent, four million examined. Customer 81 is a large business account with about 4 million orders. `EXPLAIN` shows `type: ALL`, with the index on `(customer_id, created_at)` listed in `possible_keys` but not chosen: because `DATE(created_at)` wraps the column, only the `customer_id` prefix of the index is usable, and for 4 million matching rows the optimiser judged a full scan cheaper. Nor could the index supply the `ORDER BY created_at DESC LIMIT 20` order while the date condition had to be checked by a function on every row.
- *Root cause:* a function on an indexed column made the range condition unusable, so the index could neither narrow the scan nor deliver the first 20 rows in order.
- *Fix:* `WHERE customer_id = 81 AND created_at >= '2026-01-01' ORDER BY created_at DESC LIMIT 20` → `type: range` on the composite index; `Rows_examined: 20`.
- *Prevention:* Aggregate the slow log with a digest tool (such as pt-query-digest) weekly; a lint rule for functions on columns in `WHERE`.

**Lab 3: A plan that flipped**
- *Symptom:* Every Monday morning after the weekend import job, the "orders awaiting review" dashboard query times out; by Monday afternoon it is fast again.
- *Hypotheses:* (a) load from the import; (b) stale statistics after the import; (c) locks.
- *Evidence:* `auto_explain` captured both plans. Monday morning: `Nested Loop` with the outer `Index Scan … rows=12 (actual rows=61344)`, inner loops 61,344. Monday afternoon: `Hash Join`, estimates within 10%. `pg_stat_user_tables.last_autoanalyze` for `orders` is Monday 11:40: autovacuum analysed the table only after enough changes accumulated, hours after the import.
- *Root cause:* (b): the import added tens of thousands of orders in the reviewed status; until `ANALYZE` ran, the planner believed there were 12.
- *Fix:* Run `ANALYZE orders` as the import job's last step; lower the table's auto-analyse scale factor so that analysis follows large changes sooner.
- *Prevention:* Any bulk load ends with `ANALYZE`; dashboards on estimate errors via `auto_explain` samples.

**Lab 4: A missing index found from evidence (an unindexed foreign key)**
- *Symptom:* Deleting a customer (a GDPR erasure request) takes 9 seconds and blocks other writes to `orders` while it runs.
- *Hypotheses:* (a) cascades deleting many rows; (b) lock contention; (c) a full scan per delete.
- *Evidence:*
  ```text
  BEGIN; EXPLAIN ANALYZE DELETE FROM customers WHERE id = 31337; ROLLBACK;
   Delete on customers  (actual time=0.052..0.053 rows=0 loops=1)
     ->  Index Scan using customers_pkey on customers  (actual time=0.031..0.033 rows=1 loops=1)
   Trigger for constraint orders_customer_id_fkey: time=8984.120 calls=1
   Execution Time: 8984.301 ms
  ```
  The delete itself takes 0.05 ms; the foreign-key check on `orders` takes 8,984 ms. `pg_stat_user_tables` shows `orders.seq_scan` increasing by one for each customer deletion, with 10 million `seq_tup_read` each time. `orders.customer_id` has a foreign key but no index (PostgreSQL does not create one automatically; Chapter 14, Section 14.3).
- *Root cause:* (c): each delete checks for referencing orders with a sequential scan of `orders`.
- *Fix:* `CREATE INDEX CONCURRENTLY orders_customer_id_idx ON orders (customer_id);` the constraint check becomes an index lookup; the delete takes milliseconds.
- *Prevention:* A schema check in CI that every foreign-key column has an index whose leading column is that foreign key.

---

## Solution Sketches

1. `CREATE INDEX CONCURRENTLY orders_cust_status_created ON orders (customer_id, status, created_at DESC);` (equality, equality, then the sort column). Expected plan: `Limit → Index Scan using orders_cust_status_created (Index Cond: customer_id = $1 AND status = 'PAID')` returning 20 rows in order with no sort; adding `INCLUDE (total_cents, …)` for the listed columns makes it an index-only scan. It makes the single-column `customer_id` index redundant (the new index serves all its queries); the `status` index is likely useless alone (low selectivity); keep `created_at` only if other queries need it.
2. Hypotheses: a plan flip from stale statistics (estimates now far off); a parameter-sensitive or generic plan; resource pressure (cache cold, I/O saturated by vacuum after the migration); lock waits. Evidence: `EXPLAIN (ANALYZE, BUFFERS)` versus Friday's plan (`auto_explain`), estimated versus actual rows, `last_analyze`, wait events, I/O metrics. Immediate: `ANALYZE` the migrated tables; if a generic plan is at fault, force custom plans for that statement. Lasting: `ANALYZE` as part of every migration and bulk load; per-table analyse thresholds for large tables; extended statistics if correlation is involved.
3. Trigram GIN (`pg_trgm`) supports `ILIKE '%term%'` directly with typo-tolerant similarity, but indexes many trigrams per row (large index, costly writes) and does not rank by relevance. Full-text search (`tsvector` with a GIN index) finds words with stemming and ranking, is cheaper than trigrams for long text, but does not match arbitrary substrings. A separate search engine adds relevance tuning, facets and scale, at the cost of another system, an indexing pipeline (Chapter 25) and eventual consistency between the database and search results. For 5 million products, start with full-text search in the database, add trigrams for names if substring matching is required, and move to a search engine when relevance and features demand it.
4. Collect usage over a full business cycle (`pg_stat_user_indexes.idx_scan`, or `sys.schema_unused_indexes`) on the primary **and** replicas; list redundant indexes whose columns are a leading prefix of another; measure sizes. Before dropping each: confirm it does not enforce uniqueness or back a constraint; search the code and reports for queries whose plans use it (rare month-end jobs); check replicas' usage. Drop candidates one at a time (or make them invisible first where supported, as MySQL's invisible indexes allow), watching latency of affected queries.
5. InnoDB stores rows in a clustered B+tree on the primary key. Random UUIDv4 values make every insert land on a random leaf page of a 300-million-row tree that no longer fits in the buffer pool, so inserts read pages from disk and split pages everywhere; the 36-byte primary key is also copied into each of the six secondary indexes. Migration: add a `bigint` identity (or UUIDv7 stored in `BINARY(16)`) as the new primary key, keep the old UUID as a `UNIQUE` alternate key for external references, rebuild the table online (in batches or with an online schema-change tool), and switch application lookups to the new key over time.
6. Range-partition by invoice date, one partition per month (48 partitions plus future ones created in advance). Retention: detach and drop the oldest partition each month instead of deleting rows: no bloat, little WAL. The primary key must include the partition key: `(invoice_id, invoice_date)`. Lookups by invoice ID alone probe every partition, so callers should pass the date (often encoded in the invoice number or known from the parent invoice), or keep a small non-partitioned lookup table from invoice ID to date.
7. The planner multiplies the two selectivities as if the columns were independent: 0.01 × 0.005 × 200,000 = 10 rows. But every "Acme Pro" product is an Acme product, so the `brand` condition removes nothing once the `product_line` condition holds: the true selectivity is 0.005, or 1,000 rows, a 100× underestimate. Ten outer rows make a nested loop with an inner index scan look ideal; with 1,000 outer rows, a hash join over the matching `order_lines` would be cheaper. Fix the estimate: `CREATE STATISTICS products_brand_line (dependencies) ON brand, product_line FROM products; ANALYZE products;` (or drop the redundant `brand` condition, since the product line implies it). Reduce sensitivity: an index on `order_lines (product_id)` that covers the report's columns makes each inner lookup cheap, so even a wrong nested loop stays tolerable.

---

**END OF CHAPTER 16**
