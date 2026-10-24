# Chapter 14: Relational Data Modelling and SQL

## Where We Are

Chapter 1 named the **system of record**, the store whose data is the truth for the business, and every chapter since has leaned on it. Chapter 11 used an atomic conditional `UPDATE` and an `INSERT ... ON CONFLICT` to coordinate instances. Chapter 12 said that invariants which must survive concurrency belong in **database constraints**, and Chapter 13 guarded idempotency keys and payments with unique constraints and paginated with keyset queries. All of that assumed you can design tables and write SQL. This chapter teaches both.

It starts with the idea underneath every relational database (relations, keys and constraints), moves to designing a schema (entity–relationship modelling and normalisation), and then to the language: how a `SELECT` is evaluated, joins, aggregation, subqueries and common table expressions, window functions, the three-valued logic of `NULL`, set operations, upserts and views, and the data types that cause real bugs. It ends with the classic interview queries, solved and explained.

This chapter is about **what** a query means. How the database executes it (indexes, query plans, join algorithms) is Chapter 16; what happens when two transactions run at once is Chapter 17. Here, assume one user and a correct result.

**After this chapter you can:**
- explain what a database and a DBMS are, what a DBMS provides that plain files do not, and the three-schema architecture;
- explain the relational model and why declarative queries and data independence made it the default;
- define and choose keys, and use constraints to make invalid data impossible;
- turn requirements into an entity–relationship model and map it to tables, including many-to-many relationships;
- detect update, insert and delete anomalies, identify functional dependencies, normalise a table step by step to BCNF, and decide when to denormalise;
- trace the logical evaluation order of a `SELECT`, and use it to explain `WHERE` versus `HAVING` and alias errors;
- write and draw every kind of join, including semi- and anti-joins;
- use aggregation, correlated and uncorrelated subqueries, CTEs and recursive CTEs;
- use window functions for ranking, comparisons with neighbouring rows, running totals and top-N per group;
- reason about `NULL` with three-valued logic, and avoid the `NOT IN` trap;
- use set operations, upserts, views and materialised views;
- choose data types for money, time, text and JSON without the classic bugs;
- solve the classic SQL interview problems and explain why each solution works.

**Interview questions this chapter prepares you for:**
1. What is a database? What is a DBMS, and why use one instead of files? What is the relational model, and why did it win?
2. What are super, candidate, primary, alternate and foreign keys? Surrogate or natural keys?
3. What constraints does SQL offer, and when are they checked?
4. How do you model a many-to-many relationship?
5. What is normalisation? Explain 1NF, 2NF, 3NF and BCNF with an example. When would you denormalise?
6. In what order is a `SELECT` evaluated? Why can't `WHERE` use a column alias?
7. What types of joins are there? What is the difference between a left join and an anti-join?
8. What is the difference between `WHERE` and `HAVING`?
9. Correlated versus uncorrelated subqueries; what is a CTE, and what is a recursive CTE for?
10. What are window functions? `ROW_NUMBER` versus `RANK` versus `DENSE_RANK`?
11. How does `NULL` behave in comparisons, aggregates and `NOT IN`?
12. `UNION` versus `UNION ALL`; `DELETE` versus `TRUNCATE` versus `DROP`; what is an upsert?
13. Why must money not be stored as a float? How should timestamps be stored?
14. Find the nth highest salary, duplicates, gaps, the top N per group and a running total.

**Scope.** Here: the relational model, keys, constraints, ER modelling, normalisation and denormalisation, SQL's sublanguages and logical evaluation order, joins, aggregation, subqueries, CTEs, window functions, `NULL`, set operations, upserts, views, data types, classic queries, and a placement-round section on extended ER, dependency theory, relational algebra and calculus, integrity rules, triggers and stored procedures (Section 14.14). Elsewhere: how the database stores data and survives crashes is Chapter 15; indexes, query plans and join algorithms are Chapter 16; transactions, isolation and locking are Chapter 17; drivers, connection pools, ORMs, SQL injection and migrations are Chapter 18. Examples use standard SQL, with PostgreSQL and MySQL differences noted where they matter.

---

## 14.1 Databases, the DBMS, and the Relational Model

### Definitions

**Persistence** means keeping data beyond the life of the process that created it, so that it survives restarts, crashes and deployments. A variable in a program's memory disappears when the process exits (Chapter 3); a row written to a database and committed is still there tomorrow. Part IV of this book, **data persistence**, is about how backends store data durably and retrieve it correctly.

A **database** is an organised, persistent collection of related data, structured so that many programs and users can store, find and change it reliably. The orders, customers and products of a shop, kept so that the website, the warehouse system and the finance reports all work from the same facts, form a database.

A **database management system** (DBMS) is the software that creates and manages databases and mediates every access to them. A DBMS provides:

- a **data model** and a **schema**: a declared description of what the data looks like;
- a **query language** for asking questions and making changes (SQL, for relational systems);
- **integrity**: rules (constraints) that the data must always satisfy, enforced on every change;
- **transactions**: groups of changes that happen completely or not at all, even when many users work at once;
- **durability and recovery**: committed changes survive crashes (Chapter 15);
- **concurrency control**: many users at the same time without corrupting each other's work (Chapter 17);
- **security**: who may read or change which data;
- **efficient access**: indexes and a query optimiser, so that questions stay fast as data grows (Chapter 16).

A **query** is a request to a DBMS to retrieve or change data, written in its query language. In SQL, `SELECT` statements read data, and `INSERT`, `UPDATE` and `DELETE` statements change it (Section 14.6).

A **database system** is a database together with its DBMS. In everyday speech "the database" means any of the three. "We use PostgreSQL as our database" names the DBMS, while "the database is 2 TB" means the data. PostgreSQL, MySQL, Oracle, SQL Server and SQLite are DBMSs. Most run as a separate **client–server** process that applications connect to over a network (Chapter 15, Section 15.1). **Embedded** ones, such as SQLite, run as a library inside the application process.

A **data model** is the set of concepts used to describe data: its structure, the relationships between pieces of it, and its constraints. The **relational** model (tables of rows) is the subject of this chapter. Others include the **document** model (nested JSON-like documents), **key–value** stores, **wide-column** stores and **graph** databases, collectively called NoSQL and compared in Chapter 23. The older **hierarchical** and **network** models appear below as history.

People interact with a DBMS in different roles. A **database administrator** (DBA) designs and operates the database: schemas, access rights, backups, performance. **Application programmers** write the code that queries and updates it, and **end users** use it through those applications. In many backend teams, application engineers take on much of the DBA's role, which is one reason this book goes into storage and execution in Chapters 15 and 16.

The **relational model**, proposed by Edgar F. Codd in 1970, represents all data as **relations** and manipulates them with a small set of operations whose results are again relations.

- A **relation** is a set of tuples that all have the same attributes. In SQL it is a **table**.
- A **tuple** is one element of a relation: one value for each attribute. In SQL it is a **row**.
- An **attribute** is a named property with a domain. In SQL it is a **column**.
- A **domain** is the set of values an attribute may take: integers, dates, strings of at most 200 characters, the values `'PLACED'`, `'PAID'` and `'SHIPPED'`. In SQL it is a **data type** plus any constraints on it.
- The **schema** of a relation is its name and attributes (`customers(id, name, city)`); the **instance** is the set of rows it holds at a moment.

A **relational database management system** (RDBMS) stores relations and evaluates queries written in **SQL** (Structured Query Language), the standard language for relational databases.

### Why a DBMS exists: the trouble with plain files

An application could keep its data in ordinary files (Chapter 4, Section 4.2), and early business systems did exactly that. Imagine the shop doing it: the web application appends orders to `orders.csv`, a warehouse script updates `stock.csv`, and the finance team's program reads both and writes `invoices.csv`, which copies each customer's name and address.

```text
 web app ──► orders.csv     (order id, customer name, address, product, quantity)
 warehouse ─► stock.csv      (product, quantity on hand)
 finance ──► invoices.csv   (invoice id, customer name, address, amount)  ← addresses copied again
```

Every classic weakness of the **file-based approach** appears within weeks:

1. **Redundancy and inconsistency.** A customer's address is stored in `orders.csv` and again in `invoices.csv`. When the customer moves and only one file is updated, the two disagree, and nothing detects it.
2. **Difficult access.** "Which customers in Berlin ordered a keyboard last month?" needs a new program that parses two files and matches names by hand. Every new question means new code.
3. **Isolated, scattered formats.** Each program invents its own file layout, delimiters and encoding. Combining them is a project in itself.
4. **Integrity rules buried in code.** "Stock never goes below zero" lives in the warehouse script. A second script that forgets the rule breaks it, and the file accepts any value.
5. **Atomicity problems.** A program moves stock between warehouses by subtracting from one line and adding to another. If it crashes between the two writes, stock vanishes. A file has no notion of "both or neither".
6. **Concurrent-access anomalies.** Two copies of the web application each read "stock = 1", each subtract one, and each write "stock = 0". Two orders were placed for one item. This is the lost update of Chapter 5 (Section 5.2), now between processes on different machines.
7. **Security.** File permissions are all or nothing for each file (Chapter 4, Section 4.10). Finance can read the whole of `orders.csv` or none of it, and cannot be limited to totals.
8. **Durability.** A crash or power loss in the middle of rewriting `stock.csv` can leave a truncated or half-written file, unless every program carefully implements the write-temp-`fsync`-rename pattern of Chapter 4 (Section 4.4).

A DBMS exists to solve these problems **once**, in one place, for every program that uses the data:

| File-based problem | What the DBMS provides | Where this book teaches it |
|---|---|---|
| Redundancy and inconsistency | normalised schemas; each fact stored once | Section 14.5 |
| A new program per question | a declarative query language | Sections 14.6–14.13 |
| Scattered formats | one schema and data model shared by all programs | Sections 14.1–14.4 |
| Integrity in code | declared constraints, checked on every change | Section 14.3 |
| Atomicity | transactions | Chapter 17 (Chapter 15 for how) |
| Concurrent anomalies | concurrency control: locks, MVCC, isolation levels | Chapter 17 |
| Security | users, roles and privileges per table, column and row (`GRANT`, Section 14.6) | Chapter 20 |
| Durability | write-ahead logging and crash recovery | Chapter 15 |
| Slow access as data grows | indexes and a query optimiser | Chapter 16 |

The price is a separate system to run, learn and tune, and some overhead on every access. For data that several programs share, that must survive crashes, or that many users change at once, the price is always worth paying. Small, single-program, read-mostly data, such as configuration or a local cache, can still live in files.

### Why the relational model won

Before the relational model, DBMSs were **navigational**: in the hierarchical and network models of the 1960s, records pointed to other records, and a program found data by following those pointers in a particular order ("start at this customer, follow the chain to its orders, follow each order to its lines"). Two problems followed:

1. **Programs depended on the storage structure.** If the database administrator reorganised the pointers or added an access path, programs broke. Every question needed a program written for that structure.
2. **New questions needed new code.** "Which customers in Berlin bought a keyboard last month?" meant writing a new navigation routine, by a programmer, against the current structure.

Codd's idea separated **what** data means from **how** it is stored. Relations have no pointers and no order; relationships are expressed by **values** (an order row contains its customer's ID). Queries are **declarative**: they describe the result wanted, and the database decides how to get it.

### Mechanism: the two independences and declarative queries

```text
 application                                    SQL: what I want
   SELECT c.name FROM customers c                (no mention of files, indexes, pointers, order)
   JOIN orders o ON o.customer_id = c.id
   WHERE o.created_at >= '2026-10-01'
        │
        ▼
 ┌──────────────────────── RDBMS ──────────────────────────┐
 │ parser → planner/optimiser: chooses HOW (Chapter 16)    │
 │   use an index on orders(created_at)? scan customers?   │
 │   which join algorithm? in which order?                 │
 │ executor → storage engine: pages, files (Chapter 15)    │
 └─────────────────────────────────────────────────────────┘
```

This separation is usually described with the **three-schema architecture** (also called the ANSI/SPARC architecture), which describes one database at three levels:

```text
 EXTERNAL level      view for the website   view for finance      view for the warehouse
 (what each user       (orders, products)    (invoices, totals)    (stock, shipments)
  or program sees)            │                     │                     │
                              └─────────────────────┼─────────────────────┘
                                                    │  logical data independence
 CONCEPTUAL (logical) level        tables, columns, types, keys, constraints, relationships
 (what the data is)                                 │
                                                    │  physical data independence
 INTERNAL (physical) level          pages, heap files, indexes, compression, which disk
 (how it is stored)                 (Chapter 15)
```

- **Physical data independence**: the internal level can change (add an index, move to another disk, change the file layout) without changing the conceptual schema or any query. The optimiser simply picks a different plan.
- **Logical data independence** (only partly achieved in practice): the conceptual schema can change (new tables, new columns, a table split in two) without breaking the external views, and so without breaking the programs that use them. **Views** (Section 14.11) are the tool that presents an old shape over a new one.
- **Declarative queries**: the same SQL can be executed many ways, and the DBMS chooses the cheapest one it can find. When data grows from a thousand rows to a hundred million, the query text stays the same and the plan changes.

A relation is a **set**, which has consequences that SQL only partly follows:

| Relational model | SQL in practice |
|---|---|
| No duplicate tuples | Tables can contain duplicate rows unless a key forbids them; `SELECT` returns duplicates unless `DISTINCT` is used |
| No order among tuples | Rows come back in **no guaranteed order** unless `ORDER BY` is given |
| No order among attributes | Columns have a defined order (`SELECT *` returns them in it) |
| Every value is known | SQL has `NULL` for missing or unknown values, with its own logic (Section 14.10) |

The second row causes real bugs: a query without `ORDER BY` often returns rows in insertion order on a small table, so tests pass, and then returns them in a different order once the table grows or the plan changes (Chapter 16).

### Concrete: one relation, described precisely

```sql
CREATE TABLE customers (
    id         bigint       PRIMARY KEY,
    email      varchar(254) NOT NULL UNIQUE,
    name       varchar(200) NOT NULL,
    city       varchar(100),
    created_at timestamptz  NOT NULL DEFAULT now()
);
```

```text
 relation customers                 attributes (columns) and domains (types + constraints)
 ┌────┬───────────────────┬──────┬────────┬────────────────────────┐
 │ id │ email             │ name │ city   │ created_at             │
 ├────┼───────────────────┼──────┼────────┼────────────────────────┤
 │ 1  │ ana@example.com   │ Ana  │ Berlin │ 2026-09-01 10:00:00+00 │  ← a tuple (row)
 │ 2  │ ben@example.com   │ Ben  │ Munich │ 2026-09-03 08:30:00+00 │
 │ 3  │ chen@example.com  │ Chen │ NULL   │ 2026-09-07 17:45:00+00 │  ← city unknown
 └────┴───────────────────┴──────┴────────┴────────────────────────┘
 degree (number of attributes) = 5; cardinality (number of tuples) = 3
```

### Guarantees and non-guarantees

A DBMS guarantees that every program sees the same data through the same schema, with its declared constraints, transactions, durability and access rules applied to every change, whichever program makes it. It does not guarantee a good schema (that is design, Sections 14.4–14.5), correctness of rules you never declared, or protection against a committed mistake (Chapter 15, Section 15.9). The relational model guarantees that queries are expressed against the logical schema only, so they keep working when storage changes, and that every query's result is itself a table that other queries can use. It does not guarantee performance (a declarative query can still be slow, and Chapter 16 is about why), row order without `ORDER BY`, or the absence of duplicates without a key.

### Failure and diagnosis

- **Files or spreadsheets as a system of record.** Data shared between programs lives in files (a CSV export edited by hand, a JSON file on a shared disk). Symptoms are the file-based problems above: copies that disagree, lost updates when two jobs run at once, truncated files after crashes. Fix: move the data into a DBMS and make files exports of it, not the source.
- **Order assumed, never requested.** A list or export changes order after a data or version change. Evidence: the query has no `ORDER BY`, or orders only by a non-unique column. Fix: always order explicitly, with a unique tie-breaker (Chapter 13, Section 13.6).
- **Navigational thinking in SQL.** Application code loops over rows and runs one query per row (the N+1 pattern, Chapter 13, Section 13.12; Chapter 18). Evidence: hundreds of similar queries per request. Fix: one set-based query with a join.

### Backend connection

Write queries that say what you want and let the database decide how. Never depend on an order you did not request, and treat the database as a set-processing engine: one query over many rows beats many queries over one row each.

---

## 14.2 Keys

### Definitions

A **superkey** is a set of columns whose values are unique for every row: no two rows can agree on all of them. The set of all columns is always a superkey of a table without duplicate rows.

A **candidate key** is a **minimal** superkey: removing any column from it would make it no longer unique. A table can have several candidate keys.

The **primary key** is the candidate key chosen as the main identifier of each row. It must be unique and not null, and other tables refer to rows through it. The other candidate keys are **alternate keys**, usually enforced with `UNIQUE` constraints.

A **composite key** consists of more than one column.

A **foreign key** is a column (or set of columns) in one table whose values must match a candidate key of another (or the same) table. It expresses a relationship and enforces **referential integrity**: no reference to a row that does not exist.

A **natural key** is made of data that has meaning in the real world (an email address, a country code, an ISBN). A **surrogate key** is an identifier with no business meaning, generated by the system (a sequence number or a UUID).

### Why it exists

Without keys, the database cannot tell two rows apart. Duplicates creep in, an update meant for one customer changes two, and other tables have nothing reliable to refer to. Keys turn "these rows describe the same thing" from a guess into a rule the database enforces.

### Mechanism: finding the keys of a table

Consider the `customers` table above, where each customer has a unique email:

```text
 columns: {id, email, name, city, created_at}

 superkeys (some):      {id}, {email}, {id, email}, {id, name}, {email, city}, {id, email, name, city, created_at}
 candidate keys:        {id}, {email}            ← minimal: neither can lose a column and stay unique
 not a key:             {name}                   ← two customers can share a name
 primary key (chosen):  {id}
 alternate key:         {email}                  ← enforced with UNIQUE
```

`{id, name}` is a superkey (it contains `id`, so it is unique) but not a candidate key, because `name` can be removed and the remainder `{id}` is still unique.

A composite candidate key appears naturally in tables that connect things. In an `order_lines` table where an order lists each product at most once, `{order_id, product_id}` is a candidate key: neither column alone is unique (an order has many lines; a product appears in many orders), but the pair is.

### Mechanism: foreign keys

```sql
CREATE TABLE orders (
    id          bigint      PRIMARY KEY,
    customer_id bigint      NOT NULL REFERENCES customers (id),   -- foreign key
    status      text        NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

```text
 customers                 orders
 ┌────┬──────┐             ┌─────┬─────────────┬────────┐
 │ id │ name │             │ id  │ customer_id │ status │
 ├────┼──────┤      ┌──────┼─────┼─────────────┼────────┤
 │ 1  │ Ana  │◄─────┤      │ 101 │ 1           │ PAID   │
 │ 2  │ Ben  │◄─────┼──────│ 102 │ 1           │ PLACED │
 └────┴──────┘      └──────│ 103 │ 2           │ PAID   │
                           └─────┴─────────────┴────────┘
 INSERT INTO orders VALUES (104, 9, 'PLACED', now());
 ERROR:  insert or update on table "orders" violates foreign key constraint "orders_customer_id_fkey"
 DETAIL:  Key (customer_id)=(9) is not present in table "customers".          (PostgreSQL)
```

The foreign key points from the "many" side (orders) to the "one" side (customers). A foreign key can also reference its own table: `employees.manager_id REFERENCES employees(id)` builds a hierarchy (used in Section 14.8). What happens to orders when a customer is deleted is decided by the foreign key's **action** (Section 14.3).

### Mechanism: surrogate versus natural keys

| | Natural key (`email`, `isbn`, `country_code`) | Surrogate key (`id bigint`, `id uuid`) |
|---|---|---|
| Meaning | yes; readable | none |
| Stability | can change (people change emails; companies change tax IDs) | never changes |
| Uniqueness in reality | sometimes assumed wrongly (two people share a phone; ISBNs get reused) | guaranteed by the generator |
| Size in foreign keys | often long strings, copied into every referencing row | 8 bytes (`bigint`) or 16 bytes (`uuid`) |
| Exposure | reveals data in URLs and logs | reveals only an ID (but sequential IDs reveal volume, Chapter 13, Section 13.3) |

The common practice combines both: a surrogate primary key that other tables reference, **plus** a `UNIQUE` constraint on the natural key, so that the real-world uniqueness rule is still enforced. When a customer changes email, one row changes; every order still points to the same `id`.

Surrogate keys come from two families. **Sequences** (identity columns: `id bigint GENERATED ALWAYS AS IDENTITY` in standard SQL and PostgreSQL, `AUTO_INCREMENT` in MySQL) give compact, increasing integers. **UUIDs** are 128-bit identifiers that any process can generate without asking the database, which suits distributed systems and lets clients choose IDs (useful for idempotent `PUT`, Chapter 13, Section 13.5). Fully random UUIDs (version 4) are inserted at random places in an index, which hurts insert performance on large tables; time-ordered UUIDs (version 7) avoid that. Chapter 16 explains why the position of inserts in an index matters.

### Guarantees and non-guarantees

A primary key guarantees that each row is uniquely identified and that the identifier is never null. A `UNIQUE` constraint guarantees no duplicates of a candidate key, even under concurrent inserts from many instances (Chapter 11, Section 11.7; Chapter 17 explains how). A foreign key guarantees that every reference points to an existing row. None of them guarantees that the key means what the business thinks: a "unique" email that is case-sensitive allows `Ana@example.com` and `ana@example.com` as two customers (Section 14.12 covers collations).

### Failure and diagnosis

- **Duplicates of a real-world entity.** Two customer rows for one person. Evidence: `GROUP BY lower(email) HAVING count(*) > 1` (Section 14.13). Cause: no `UNIQUE` on the natural key, or a case-sensitive one. Fix: clean up, then add the constraint (on a normalised form of the value).
- **Natural key changes.** A product code is re-issued, and references must be updated in twenty tables. Fix: surrogate keys for references, natural keys as alternate keys.
- **Orphans.** Orders referencing customers that no longer exist. Evidence: an anti-join finds them (Section 14.7). Cause: no foreign key. Fix: clean up, add the foreign key.

### Backend connection

Give every table a primary key, usually a surrogate one; protect every real-world uniqueness rule with a `UNIQUE` constraint; and express every relationship with a foreign key. Keys are the cheapest correctness guarantee you can buy, and the database enforces them across every instance of every service.

---

## 14.3 Constraints

### Definitions

A **constraint** is a rule declared in the schema that the database enforces on every change: a change that would violate it is rejected with an error, and the statement has no effect.

| Constraint | Rule |
|---|---|
| `NOT NULL` | the column must have a value |
| `UNIQUE` | no two rows have the same values in these columns (rows with `NULL` in them are not considered equal, Section 14.10) |
| `PRIMARY KEY` | `UNIQUE` and `NOT NULL`; at most one per table |
| `FOREIGN KEY` | values must exist in the referenced key, or be `NULL` |
| `CHECK` | a boolean condition on the row must not be false |
| `DEFAULT` | not a rule, but the value used when an insert omits the column |

### Why it exists

Every rule that is enforced only in application code is enforced only in that code path. A second service, a batch job, a migration script, a support engineer with a SQL console, or two concurrent requests that each pass a check-then-act test (Chapter 5, Section 5.2) can all break it. A constraint is enforced in one place that every writer must pass through, and it is checked atomically with the change. That is why Chapter 12 (Section 12.3) placed invariants that must survive concurrency in the database.

### Mechanism 1: the constraints on a real table

```sql
CREATE TABLE products (
    id           bigint        GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku          varchar(32)   NOT NULL UNIQUE,
    name         varchar(200)  NOT NULL,
    price_cents  integer       NOT NULL CHECK (price_cents >= 0),
    stock        integer       NOT NULL DEFAULT 0 CHECK (stock >= 0),
    discontinued boolean       NOT NULL DEFAULT false
);

CREATE TABLE order_lines (
    order_id    bigint  NOT NULL REFERENCES orders (id) ON DELETE CASCADE,
    product_id  bigint  NOT NULL REFERENCES products (id) ON DELETE RESTRICT,
    quantity    integer NOT NULL CHECK (quantity BETWEEN 1 AND 100),
    unit_cents  integer NOT NULL CHECK (unit_cents >= 0),
    PRIMARY KEY (order_id, product_id)
);
```

```text
 UPDATE products SET stock = stock - 5 WHERE id = 17;          -- stock was 3
 ERROR:  new row for relation "products" violates check constraint "products_stock_check"
 DETAIL:  Failing row contains (17, KB-42, Keyboard, 1999, -2, f).                   (PostgreSQL)
```

The `CHECK (stock >= 0)` constraint makes overselling impossible however many instances try to decrement at once: the decrement that would go below zero fails. It complements the conditional update of Chapter 11 (Section 11.7) rather than replacing it.

One detail of `CHECK` follows from `NULL` logic (Section 14.10): a check constraint rejects a row only if the condition is **false**. If the condition is **unknown** because a column is `NULL`, the row is accepted. `CHECK (discount_pct <= 50)` accepts a `NULL` discount; add `NOT NULL` if the value is required.

> **MySQL note:** MySQL parsed `CHECK` constraints but silently ignored them before version 8.0.16. Schemas migrated from older versions may contain checks that were never enforced, and data that violates them.

### Mechanism 2: foreign-key actions

When a referenced row is deleted, or its key is updated, the foreign key's **action** decides what happens to the referencing rows:

| Action | On `DELETE FROM customers WHERE id = 1` | Typical use |
|---|---|---|
| `NO ACTION` (default) | error if orders still reference customer 1 (checked at the end of the statement, or at commit if deferred) | most relationships |
| `RESTRICT` | error, checked immediately, never deferrable | the same, when deferral must be impossible |
| `CASCADE` | customer 1's orders are deleted too | parts that cannot exist alone (order lines with their order) |
| `SET NULL` | the orders' `customer_id` becomes `NULL` | optional relationships (an order kept after a customer is erased, if allowed) |
| `SET DEFAULT` | `customer_id` becomes its default value | rare |

`CASCADE` deserves care: a cascade can chain through several levels, and one `DELETE` of a customer can silently remove orders, lines, payments and shipments. Use it only for true parts of a whole (lines of an order), and prefer `RESTRICT` or `NO ACTION` where deletion should be a deliberate, multi-step business operation. In MySQL, `RESTRICT` and `NO ACTION` behave the same.

> **Performance note:** PostgreSQL does not create an index on the **referencing** column of a foreign key automatically (MySQL's InnoDB does). Without one, deleting a customer makes the database scan the whole `orders` table to check for references. Chapter 16 covers this.

### Mechanism 3: when constraints are checked

By default, constraints are checked **immediately**: as part of each statement, so a statement that violates one fails and changes nothing. Only constraints declared **`DEFERRABLE INITIALLY DEFERRED`** (or switched with `SET CONSTRAINTS ... DEFERRED` inside a transaction) are checked at **commit**. A **transaction** groups several statements so that they take effect together or not at all (Chapter 12, Section 12.2 gave the working definition; Chapter 17 is its home).

Deferral solves the chicken-and-egg problem of mutually referencing rows. Suppose every department must have a manager, and every employee must belong to a department:

```sql
CREATE TABLE departments (
    id         bigint PRIMARY KEY,
    manager_id bigint NOT NULL
);
CREATE TABLE employees (
    id            bigint PRIMARY KEY,
    department_id bigint NOT NULL REFERENCES departments (id) DEFERRABLE INITIALLY DEFERRED
);
ALTER TABLE departments ADD FOREIGN KEY (manager_id) REFERENCES employees (id)
    DEFERRABLE INITIALLY DEFERRED;

BEGIN;
INSERT INTO departments VALUES (10, 500);   -- employee 500 does not exist yet: allowed, check deferred
INSERT INTO employees   VALUES (500, 10);   -- now it does
COMMIT;                                      -- both foreign keys checked here: both satisfied
```

With immediate checking, the first `INSERT` would fail, because employee 500 does not exist, and the second could not run first either.

```text
 timeline of constraint checks
 statement 1 ─► [check immediate constraints] ─► statement 2 ─► [check immediate] ─► COMMIT ─► [check deferred]
```

Two engine differences matter:

- **PostgreSQL** checks non-deferrable `UNIQUE` and `PRIMARY KEY` constraints **row by row**, not at the end of the statement as the SQL standard says. So `UPDATE items SET position = position + 1` can fail with a duplicate-key error partway through, when row 1 moves to position 2 before row 2 has moved to 3. Declaring the constraint `DEFERRABLE` (even `INITIALLY IMMEDIATE`) gives the standard's end-of-statement behaviour.
- **MySQL** does not support deferred constraints at all, and InnoDB checks uniqueness and foreign keys row by row; the same renumbering `UPDATE` needs an `ORDER BY position DESC` to process rows in a safe order.

### Guarantees and non-guarantees

A declared constraint guarantees that no committed data violates it, whatever code wrote the data and however many writers ran at once. It does not guarantee rules that span rows in ways the constraint types cannot express ("each customer has at most three open orders"), which need transactions with suitable isolation or locking (Chapter 17); it does not guarantee meaningful values within the type (a `CHECK` you did not write); and in MySQL before 8.0.16 it does not guarantee `CHECK` at all.

### Failure and diagnosis

- **Constraint added to existing data fails.** `ALTER TABLE ... ADD CONSTRAINT` is rejected because old rows violate it. Evidence: the error names a failing row. Fix: find violations with a query, repair them, then add the constraint (PostgreSQL can add a `CHECK` or foreign-key constraint as `NOT VALID`, which enforces it for new rows at once, and then `VALIDATE CONSTRAINT` it later, after old rows are repaired).
- **Unexpected mass deletion.** A `DELETE` of one parent removed thousands of rows. Evidence: the schema shows `ON DELETE CASCADE` chains. Fix: `RESTRICT` where deletion should not cascade.
- **Application receives constraint errors.** These are expected signals, not bugs: translate them into domain errors at the repository (unique violation is SQLSTATE `23505` in PostgreSQL, error `1062` in MySQL; Chapter 12, Section 12.8).

### Backend connection

Declare every rule the schema can express: `NOT NULL` by default, `UNIQUE` for real-world uniqueness, foreign keys for every relationship, `CHECK` for ranges and simple invariants. Choose foreign-key actions deliberately, defer only when mutual references require it, and translate constraint violations into clear API errors rather than `500`s.

---

## 14.4 Entity–Relationship Modelling

### Definitions

**Entity–relationship (ER) modelling**, introduced by Peter Chen in 1976, describes a domain before any tables are designed.

- An **entity** is a thing the business keeps data about, with its own identity: a customer, an order, a product. An **entity type** is the category; an entity is one instance.
- An **attribute** is a property of an entity: a customer's name. Attributes can be **simple** (a name), **composite** (an address made of street, city and postcode), **multivalued** (several phone numbers) or **derived** (age, computed from date of birth).
- A **relationship** connects entities: a customer *places* an order; an order *contains* products.
- **Cardinality** states how many entities on each side can take part: **one-to-one (1:1)**, **one-to-many (1:N)** or **many-to-many (M:N)**.
- **Participation** states whether taking part is **mandatory** (total: every order must have a customer) or **optional** (partial: a customer may have no orders).
- A **weak entity** has no identity of its own and is identified through its owner: an order line exists only within its order.

### Why it exists

Tables are an implementation detail. Starting with tables invites decisions driven by the first screen or the first query. An ER model forces the questions that decide correctness: what things exist, which can exist alone, how many of one can relate to how many of the other, and which relationships carry data of their own. Getting cardinality wrong is expensive: modelling "an order has one shipment" as a column on `orders` breaks the day an order ships in two parcels.

### Concrete: the shop as an ER diagram

Requirements: customers place orders; an order contains several products, each with a quantity and the price charged; a product belongs to one category; a product can have many tags and a tag many products; each customer has at most one loyalty account; an order can ship in several shipments.

```text
 crow's-foot notation:  ──||  exactly one     ──o|  zero or one     ──|<  one or many     ──o<  zero or many

 ┌──────────┐ 1       places        0..N ┌──────────┐ 1     contains     1..N ┌─────────────┐ N   for   1 ┌──────────┐
 │ CUSTOMER │──||──────────────────────o<│  ORDER   │──||──────────────────|<│ ORDER_LINE  │>|─────────||│ PRODUCT  │
 │ id       │                            │ id       │                        │ (weak)      │             │ id       │
 │ email    │                            │ status   │                        │ quantity    │             │ sku      │
 │ name     │                            │ placed_at│                        │ unit_cents  │             │ name     │
 └────┬─────┘                            └────┬─────┘                        └─────────────┘             └──┬────┬──┘
      │ 1                                     │ 1                                                     N │    │ M
      ||                                      ||                                               belongs to│    │ tagged
      │ has                                   │ ships in                                               ||    o<
      o|                                      o<                                                      1 │    │ N
 ┌────┴──────────┐                      ┌─────┴─────┐                                         ┌──────┴──┐ ┌┴──────┐
 │ LOYALTY_ACCT  │ 0..1                 │ SHIPMENT  │ 0..N                                    │CATEGORY │ │  TAG  │
 │ points        │                      │ tracking  │                                         └─────────┘ └───────┘
 └───────────────┘                      └───────────┘
```

Reading it: a customer places zero or more orders, and each order is placed by exactly one customer (1:N, mandatory on the order side). An order contains one or more lines; each line belongs to one order and refers to one product. Products and orders are therefore related **many-to-many through order lines**, and that relationship carries data (quantity, price), which is why it is an entity of its own. Products and tags are many-to-many with no data on the relationship. Customers and loyalty accounts are 1:1, optional on the account side.

### Mechanism: mapping the model to tables

| Model element | Table design | Example |
|---|---|---|
| Entity | a table; a key; one column per simple attribute | `customers(id, email, name)` |
| Composite attribute | several columns (or a separate table if shared) | `address_line1, address_city, address_postcode` |
| Multivalued attribute | a separate table with a foreign key | `customer_phones(customer_id, phone)` |
| Derived attribute | usually not stored; computed in queries or views | age from `date_of_birth` |
| 1:N relationship | a foreign key on the **N** side | `orders.customer_id → customers.id` |
| 1:1 relationship | a foreign key on one side with a `UNIQUE` constraint (or a shared primary key) | `loyalty_accounts.customer_id UNIQUE REFERENCES customers(id)` |
| M:N relationship | a **junction table** (also called an associative or link table) with two foreign keys and a composite primary key | `product_tags(product_id, tag_id)` |
| M:N with attributes | a junction table with extra columns | `order_lines(order_id, product_id, quantity, unit_cents)` |
| Weak entity | primary key includes the owner's key; `ON DELETE CASCADE` | `order_lines` primary key `(order_id, product_id)` |
| Mandatory participation | `NOT NULL` on the foreign key | `orders.customer_id NOT NULL` |

The junction table is the one piece that cannot be avoided. A many-to-many relationship cannot be stored as a column on either side without breaking 1NF (Section 14.5), as a "tags" column holding `'gift,sale'` would.

```sql
CREATE TABLE tags (
    id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name varchar(50) NOT NULL UNIQUE
);
CREATE TABLE product_tags (
    product_id bigint NOT NULL REFERENCES products (id) ON DELETE CASCADE,
    tag_id     bigint NOT NULL REFERENCES tags (id)     ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)        -- a product cannot have the same tag twice
);
CREATE TABLE loyalty_accounts (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id bigint NOT NULL UNIQUE REFERENCES customers (id),   -- UNIQUE makes it 1:1
    points      integer NOT NULL DEFAULT 0 CHECK (points >= 0)
);
```

```text
 products                 product_tags              tags
 ┌────┬──────────┐        ┌────────────┬────────┐   ┌────┬─────────┐
 │ id │ name     │        │ product_id │ tag_id │   │ id │ name    │
 ├────┼──────────┤        ├────────────┼────────┤   ├────┼─────────┤
 │ 17 │ Keyboard │◄───────│ 17         │ 1      │──►│ 1  │ gift    │
 │ 18 │ Mouse    │◄─┐     │ 17         │ 2      │──►│ 2  │ sale    │
 └────┴──────────┘  └─────│ 18         │ 2      │   └────┴─────────┘
                          └────────────┴────────┘
 Keyboard has tags {gift, sale}; the tag "sale" has products {Keyboard, Mouse}.
```

### Guarantees and non-guarantees

An ER model mapped with these rules guarantees that each fact has a place and that cardinalities are enforced by keys: a `UNIQUE` foreign key cannot hold two loyalty accounts for one customer; a composite primary key cannot tag a product twice. It does not guarantee **minimum** cardinalities beyond "at least zero": "every order has at least one line" cannot be expressed by a foreign key, and must be enforced in the application's transaction or with deferred checks or triggers (Section 14.14).

### Failure and diagnosis

- **Comma-separated lists in a column.** `tags = 'gift,sale'`. Symptoms: queries with `LIKE '%sale%'` that also match "wholesale"; no way to rename a tag in one place; no foreign key. Fix: a junction table.
- **Wrong cardinality.** A `shipment_id` column on `orders` when orders can ship in parts. Evidence: business requests that the schema cannot represent, or duplicated orders created to hold a second shipment. Fix: a separate table with a foreign key to orders.
- **Missing relationship data.** Quantity stored on the product, or the order's price read from the current product price. Evidence: old orders' totals change when prices change. Fix: the relationship (order line) holds its own data, including the price charged.

### Backend connection

Sketch the ER model before writing `CREATE TABLE`: name the entities, draw each relationship with both cardinalities, and ask of every relationship whether it carries data. Map 1:N to foreign keys on the many side, 1:1 to unique foreign keys, and M:N to junction tables, and let `NOT NULL` express mandatory participation.

---

## 14.5 Normalisation and Denormalisation

### Definitions

**Normalisation** is the process of organising a database's tables so that each fact is stored once, by splitting tables whose columns depend on different things. It removes **redundancy**, and with it three kinds of **anomaly**:

- An **update anomaly**: one fact is stored in several rows, so changing it requires changing all of them, and missing one leaves the data contradicting itself.
- An **insert anomaly**: a fact cannot be recorded without also recording an unrelated fact.
- A **delete anomaly**: deleting one fact destroys another fact that happened to share its row.

A **functional dependency** (FD), written `X → Y`, holds when any two rows that agree on the columns `X` must agree on the columns `Y`: `X` **determines** `Y`. `customer_id → customer_city` means a customer has one city. FDs come from the business rules, not from the data that happens to be in the table today.

Some variations:

- A dependency on a whole composite key is **full**; a dependency on only part of a candidate key is **partial**.
- A **transitive dependency** is one that goes through a non-key column: `order_id → customer_id` and `customer_id → customer_city`, so `order_id → customer_city` transitively.
- A **prime attribute** is a column that belongs to some candidate key; the others are **non-prime**.

The **normal forms** are increasingly strict conditions on a table:

| Normal form | Condition (each includes the previous) | In plain words |
|---|---|---|
| **1NF** | every column holds one atomic value; no repeating groups; rows are identifiable by a key | one value per cell, one fact per row |
| **2NF** | no non-prime column depends on only **part** of a candidate key | every non-key column needs the whole key |
| **3NF** | for every non-trivial FD `X → A`, either `X` is a superkey or `A` is prime | non-key columns depend on nothing but the key |
| **BCNF** (Boyce–Codd) | for every non-trivial FD `X → Y`, `X` is a superkey | every determinant is a key |

The classic summary of 2NF and 3NF is that every non-key column must depend on "the key, the whole key, and nothing but the key".

**Denormalisation** is deliberately storing redundant data, usually to make reads cheaper, and accepting the work of keeping the copies consistent.

### Why it exists

Redundancy is not just wasted space. Every copy of a fact is a place where the fact can go wrong. A customer's city stored on each of their 40 orders will, sooner or later, be updated on 39 of them. Normalisation makes inconsistency structurally impossible by giving each fact exactly one home, so that the database's own keys keep everything else consistent.

### Concrete: the walkthrough, starting from one messy table

A shop starts with a single spreadsheet-like table. Each order also records the **account manager** responsible for that customer in that product category. The business rules are: each customer has one manager per category, and each manager works in exactly one category.

**Unnormalised (UNF).**

```text
 order_sheet
 order_id │ order_date │ cust │ cust_name │ cust_city │ items
 ─────────┼────────────┼──────┼───────────┼───────────┼─────────────────────────────────────────────────────────────
 1001     │ 2026-10-01 │ C1   │ Ana       │ Berlin    │ KB-42 Keyboard [Peripherals] x2 @19.99 (mgr Ravi);
          │            │      │           │           │ MS-7 Mouse [Peripherals] x1 @9.99 (mgr Ravi)
 1002     │ 2026-10-02 │ C2   │ Ben       │ Munich    │ LT-1 Laptop [Computers] x1 @899.00 (mgr Tom)
 1003     │ 2026-10-05 │ C1   │ Ana       │ Berlin    │ LT-1 Laptop [Computers] x1 @879.00 (mgr Sofia)
 1004     │ 2026-10-06 │ C3   │ Chen      │ Hamburg   │ MS-7 Mouse [Peripherals] x3 @9.99 (mgr Ravi)
```

The `items` column holds a **repeating group**: a list of several facts packed into one cell. The database cannot answer "how many keyboards were sold?" without parsing strings, cannot enforce that a quantity is a positive number, and cannot reference a product. This violates 1NF.

**Step 1 → 1NF: one atomic value per cell, one order line per row.**

```text
 order_lines_1nf            key: {order_id, product_id}
 order_id│product_id│order_date│cust│cust_name│cust_city│product_name│category   │qty│unit_price│manager
 ────────┼──────────┼──────────┼────┼─────────┼─────────┼────────────┼───────────┼───┼──────────┼───────
 1001    │ KB-42    │2026-10-01│ C1 │ Ana     │ Berlin  │ Keyboard   │Peripherals│ 2 │  19.99   │ Ravi
 1001    │ MS-7     │2026-10-01│ C1 │ Ana     │ Berlin  │ Mouse      │Peripherals│ 1 │   9.99   │ Ravi
 1002    │ LT-1     │2026-10-02│ C2 │ Ben     │ Munich  │ Laptop     │Computers  │ 1 │ 899.00   │ Tom
 1003    │ LT-1     │2026-10-05│ C1 │ Ana     │ Berlin  │ Laptop     │Computers  │ 1 │ 879.00   │ Sofia
 1004    │ MS-7     │2026-10-06│ C3 │ Chen    │ Hamburg │ Mouse      │Peripherals│ 3 │   9.99   │ Ravi
```

Every value is now queryable and typeable. But the table has these functional dependencies:

```text
 F1  order_id                 → order_date, cust
 F2  cust                     → cust_name, cust_city
 F3  product_id               → product_name, category
 F4  {order_id, product_id}   → qty, unit_price          (the price charged on this order)
 F5  {cust, category}         → manager
 F6  manager                  → category
```

and it suffers all three anomalies:

- **Update:** Ana moves to Hamburg. Her city appears in three rows; an update that misses one leaves her in two cities.
- **Insert:** a new product `HS-3 Headset` cannot be recorded until someone orders it, because `order_id` is part of the primary key and cannot be null.
- **Delete:** deleting order 1002, Ben's only order, also deletes the facts that Ben lives in Munich and that Tom manages Ben's computers.

**Step 2 → 2NF: remove partial dependencies.** F1 depends on `order_id` alone and F3 on `product_id` alone: each is a part of the key `{order_id, product_id}`. Move each to a table keyed by that part:

```text
 orders_2nf (order_id PK)                         products (product_id PK)
 order_id│order_date│cust│cust_name│cust_city      product_id│product_name│category
 1001    │2026-10-01│ C1 │ Ana     │ Berlin        KB-42     │ Keyboard   │Peripherals
 1002    │2026-10-02│ C2 │ Ben     │ Munich        MS-7      │ Mouse      │Peripherals
 1003    │2026-10-05│ C1 │ Ana     │ Berlin        LT-1      │ Laptop     │Computers
 1004    │2026-10-06│ C3 │ Chen    │ Hamburg

 order_lines_2nf ({order_id, product_id} PK)
 order_id│product_id│qty│unit_price│manager
 1001    │ KB-42    │ 2 │  19.99   │ Ravi
 1001    │ MS-7     │ 1 │   9.99   │ Ravi
 1002    │ LT-1     │ 1 │ 899.00   │ Tom
 1003    │ LT-1     │ 1 │ 879.00   │ Sofia
 1004    │ MS-7     │ 3 │   9.99   │ Ravi
```

Removed: the product name and category are stored once per product, so renaming "Mouse" touches one row, and the headset can be added to `products` before anyone orders it. The order date is stored once per order.

Note that `unit_price` stayed on the order line. It looks like a duplicate of a product price, but it is a different fact: the price **charged on this order** (Ana paid 879.00 for a laptop that Ben bought at 899.00). The product's current list price, if needed, belongs in `products`. Recognising which "duplicates" are separate facts is half of normalisation.

What remains: Ana's name and city still repeat on orders 1001 and 1003, and the manager is repeated on every line of order 1001. Both are **transitive** dependencies, through columns that are not keys.

**Step 3 → 3NF: remove transitive dependencies.** In `orders_2nf`, `order_id → cust → cust_name, cust_city`: the customer's details depend on the key only through `cust`. In `order_lines_2nf`, `manager` depends on the key only through `{cust, category}`, which are not columns of that table at all (they are reached through `order_id` and `product_id`). Move each to a table keyed by its real determinant:

```text
 customers (cust PK)          orders (order_id PK, cust FK)        order_lines ({order_id, product_id} PK)
 cust│name│city               order_id│order_date│cust              order_id│product_id│qty│unit_price
 C1  │Ana │Berlin             1001    │2026-10-01│ C1               1001    │ KB-42    │ 2 │  19.99
 C2  │Ben │Munich             1002    │2026-10-02│ C2               1001    │ MS-7     │ 1 │   9.99
 C3  │Chen│Hamburg            1003    │2026-10-05│ C1               1002    │ LT-1     │ 1 │ 899.00
                              1004    │2026-10-06│ C3               1003    │ LT-1     │ 1 │ 879.00
                                                                    1004    │ MS-7     │ 3 │   9.99
 products (unchanged)         account_managers
                              cust│category   │manager
                              C1  │Peripherals│ Ravi
                              C1  │Computers  │ Sofia
                              C2  │Computers  │ Tom
                              C3  │Peripherals│ Ravi
```

Removed: Ana's move to Hamburg is one update. Deleting order 1002 no longer deletes Ben or his manager. Every non-key column now depends on the key, the whole key and nothing but the key.

**Step 4 → BCNF.** Look closely at `account_managers`. Its candidate keys are `{cust, category}` (by F5) and `{cust, manager}` (since `manager → category`, knowing the customer and the manager determines the row). So every column is prime, and the table **is in 3NF**: F6 `manager → category` has a non-superkey on the left, but its right side, `category`, is prime, which 3NF allows.

It is **not** in BCNF, because `manager` determines `category` without being a superkey. The redundancy is visible: "Ravi works in Peripherals" is stored twice (for C1 and C3), and with 500 customers it would be stored 500 times.

- **Update:** Ravi moves to Computers; every one of his rows must change.
- **Insert:** a new manager, Lina, who handles Computers, cannot be recorded until she has a customer.
- **Delete:** if Tom's only customer leaves, the fact that Tom handles Computers disappears.

Decompose on the violating dependency, `manager → category`:

```text
 managers (manager PK)      customer_managers ({cust, manager} PK)
 manager│category           cust│manager
 Ravi   │Peripherals        C1  │ Ravi
 Sofia  │Computers          C1  │ Sofia
 Tom    │Computers          C2  │ Tom
 Lina   │Computers          C3  │ Ravi
```

Now each manager's category is stored once, and Lina can exist before her first customer. Joining the two tables gives back exactly `account_managers`: the decomposition is **lossless** (Section 14.14 explains the test).

But something was lost: the rule F5, "one manager per customer per category". In `customer_managers`, nothing stops a row `(C1, Tom)`, which would give Ana **two** Computers managers (Sofia and Tom). The dependency `{cust, category} → manager` now spans two tables, so no key on either table can enforce it. This is the classic cost of BCNF: a decomposition into BCNF is always possible without losing information, but **not always without losing dependencies**, while 3NF can always preserve them.

A practical schema keeps both properties by storing the category in the link table as well and letting constraints keep the copy honest:

```sql
CREATE TABLE managers (
    manager  text PRIMARY KEY,
    category text NOT NULL,
    UNIQUE (manager, category)                       -- a target for the composite foreign key below
);
CREATE TABLE customer_managers (
    cust     text NOT NULL REFERENCES customers (cust),
    category text NOT NULL,
    manager  text NOT NULL,
    PRIMARY KEY (cust, category),                    -- F5: one manager per customer per category
    FOREIGN KEY (manager, category) REFERENCES managers (manager, category)   -- F6 kept consistent
);
```

The category is still stored redundantly, so this table is not in BCNF, but the composite foreign key makes it impossible for the copy to disagree with `managers`. Normal forms remove redundancy; constraints make the redundancy you keep safe.

**The final schema** (3NF, with the managers fragment handled as above):

```sql
CREATE TABLE customers   (cust text PRIMARY KEY, name text NOT NULL, city text);
CREATE TABLE products    (product_id text PRIMARY KEY, product_name text NOT NULL, category text NOT NULL);
CREATE TABLE orders      (order_id bigint PRIMARY KEY, order_date date NOT NULL,
                          cust text NOT NULL REFERENCES customers (cust));
CREATE TABLE order_lines (order_id bigint REFERENCES orders (order_id),
                          product_id text REFERENCES products (product_id),
                          qty integer NOT NULL CHECK (qty > 0),
                          unit_price numeric(10,2) NOT NULL,
                          PRIMARY KEY (order_id, product_id));
```

### Mechanism: how to normalise any table

```text
 1. List the columns and the business rules; write every functional dependency.
 2. Find the candidate keys (Section 14.14 gives a method based on attribute closure).
 3. 1NF: split repeating groups and lists into rows (or a child table).
 4. 2NF: for each FD whose left side is a PART of a candidate key → move its right side to a table keyed by that part.
 5. 3NF: for each FD whose left side is not a superkey and whose right side is non-prime → move it out.
 6. BCNF: for each FD whose left side is not a superkey (even if the right side is prime) → decompose,
          and check which dependencies the decomposition can no longer enforce.
 7. Check every decomposition is lossless: joining the pieces must give back exactly the original rows.
```

In practice, most tables that come out of a careful ER model (Section 14.4) are already in 3NF or BCNF, because each entity's attributes depend on its key. Normalisation is most useful as a diagnostic: when a table shows an anomaly, find the dependency causing it.

### Mechanism: denormalisation, deliberately

Normalised schemas sometimes make important reads expensive: a dashboard that shows each order's total sums its lines every time; a product list that shows each product's review count counts reviews every time. **Denormalisation** stores the derived or copied value so that reads become cheap.

| Technique | Example | Kept consistent by |
|---|---|---|
| Stored derived value | `orders.total_cents` = sum of its lines | updating it in the same transaction as the lines |
| Counter | `products.review_count` | an atomic `UPDATE … SET review_count = review_count + 1` in the same transaction, or a trigger (Section 14.14) |
| Copied column for filtering or sorting | `order_lines.order_date` copied from `orders`, to filter lines by date without a join | write path discipline; a check query |
| Precomputed aggregate table | `daily_sales(day, category, revenue)` | a scheduled job, or a materialised view (Section 14.11) |
| Read model in another store | a search index with product, category and tags merged | asynchronous updates from change events (Chapter 25) |

Not every copy is denormalisation. The price on an order line, the shipping address on an order, and the customer's name on an issued invoice are **historical facts**: they must not change when the product's price, the customer's current address or name changes later. Storing them is correct normalised design.

The cost of denormalisation is that the database no longer guarantees the copies agree, so you must:

1. update every copy in the **same transaction** as its source wherever possible;
2. decide what happens when they diverge (which one is the truth);
3. run a **reconciliation query** that finds divergence:

```sql
-- orders whose stored total disagrees with the sum of their lines
SELECT o.id, o.total_cents, l.line_total
FROM orders o
JOIN (SELECT order_id, SUM(quantity * unit_cents) AS line_total
      FROM order_lines GROUP BY order_id) l ON l.order_id = o.id
WHERE o.total_cents <> l.line_total;
```

Denormalise when a measured read path is too slow, the read is far more frequent than the write, and you have a mechanism that keeps the copy consistent. Do it last, and do it one field at a time.

### Guarantees and non-guarantees

A schema in 3NF or BCNF guarantees that each fact is stored once, so update, insert and delete anomalies of the kinds above cannot occur. It does not guarantee that the FDs you wrote were the real business rules (a wrong FD produces a wrong design), that every dependency is enforceable after a BCNF decomposition, or good read performance (more joins). Denormalised data is guaranteed consistent only to the extent your write path or reconciliation makes it so.

### Failure and diagnosis

- **Contradictory copies.** Two rows disagree about the same fact (a customer in two cities; an order total that does not match its lines). Evidence: `GROUP BY` the determinant `HAVING COUNT(DISTINCT dependent) > 1`, or the reconciliation query above. Cause: an unnormalised table or an unmaintained denormalisation.
- **Lost facts after deletes.** A customer's details vanish when their last order is deleted. Cause: customer data stored only on orders.
- **Over-normalisation.** Every read joins eight tables for data that is always read together and never changes independently. Evidence: slow, complex queries for simple screens. Fix: reconsider whether the split reflects a real dependency; consider a view or a measured denormalisation.

### Backend connection

Normalise to 3NF (or BCNF where dependencies allow) by default, because the database then keeps your data consistent for free. Store historical facts deliberately where values must not change later. Denormalise only for a measured read problem, update copies in the same transaction, and keep a reconciliation query running to catch drift.

---

## 14.6 SQL Fundamentals and the Logical Evaluation Order

### Definitions

SQL statements fall into groups by purpose:

| Group | Purpose | Statements |
|---|---|---|
| **DDL** (data definition language) | define and change the schema | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME` |
| **DML** (data manipulation language) | read and change rows | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE` (some texts put `SELECT` alone in a **DQL**, data query language) |
| **DCL** (data control language) | grant and remove privileges | `GRANT`, `REVOKE` |
| **TCL** (transaction control language) | group statements into transactions | `BEGIN`/`START TRANSACTION`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` (Chapter 17) |

The **logical evaluation order** of a `SELECT` is the order in which its clauses are defined to take effect. It differs from the order in which they are written:

```text
 written order:    SELECT … FROM … WHERE … GROUP BY … HAVING … ORDER BY … LIMIT
 logical order:    1 FROM (and JOINs)  2 WHERE  3 GROUP BY  4 HAVING  5 SELECT (expressions, aliases,
                   window functions)  6 DISTINCT  7 ORDER BY  8 LIMIT / OFFSET
```

### Why it matters

Most confusing SQL errors and wrong results come from writing a clause as though it ran in written order: using a `SELECT` alias in `WHERE`, filtering an aggregate in `WHERE`, expecting `LIMIT` to apply before `ORDER BY`. The logical order explains each of them. It is the order in which the **result is defined**; the database may physically execute things differently (pushing filters down, using an index to avoid sorting) as long as the result is the same (Chapter 16).

### Concrete: the logical order on one query

```text
 orders
 id  │ customer_id │ status    │ total_cents
 101 │ 1           │ PAID      │ 3000
 102 │ 1           │ PAID      │ 2500
 103 │ 2           │ PAID      │ 9900
 104 │ 2           │ CANCELLED │ 4000
 105 │ 3           │ PAID      │ 1200
 106 │ 3           │ PAID      │ 1500
 107 │ 4           │ PAID      │ 7000
```

```sql
SELECT customer_id, COUNT(*) AS paid_orders, SUM(total_cents) AS spent
FROM orders
WHERE status = 'PAID'
GROUP BY customer_id
HAVING SUM(total_cents) > 5000
ORDER BY spent DESC
LIMIT 2;
```

```text
 1 FROM orders                 7 rows
 2 WHERE status = 'PAID'       6 rows (104 removed)
 3 GROUP BY customer_id        4 groups:  1 → {3000, 2500}   2 → {9900}   3 → {1200, 1500}   4 → {7000}
 4 HAVING SUM(...) > 5000      group 1: 5500 ✔   group 2: 9900 ✔   group 3: 2700 ✘   group 4: 7000 ✔
 5 SELECT                      (1, 2, 5500)   (2, 1, 9900)   (4, 1, 7000)      ← aliases paid_orders, spent exist from here
 6 DISTINCT                    (none)
 7 ORDER BY spent DESC         (2, 1, 9900)   (4, 1, 7000)   (1, 2, 5500)
 8 LIMIT 2                     (2, 1, 9900)   (4, 1, 7000)
```

Result:

```text
 customer_id │ paid_orders │ spent
 2           │ 1           │ 9900
 4           │ 1           │ 7000
```

### Mechanism: what the order explains

**`WHERE` versus `HAVING`.** `WHERE` filters **rows** before grouping; `HAVING` filters **groups** after aggregation. An aggregate does not exist yet at step 2, so it cannot appear in `WHERE`:

```text
 SELECT customer_id FROM orders WHERE SUM(total_cents) > 5000 GROUP BY customer_id;
 ERROR:  aggregate functions are not allowed in WHERE                                  (PostgreSQL)
```

A condition on individual rows belongs in `WHERE` even when the query groups: `WHERE status = 'PAID'` throws rows away before the work of grouping them. Putting it in `HAVING` would be an error here (`status` is not grouped) and, where allowed, wasteful.

**Aliases.** A `SELECT` alias is created at step 5, so `WHERE` (step 2) cannot see it, while `ORDER BY` (step 7) can:

```text
 SELECT total_cents / 100.0 AS total_eur FROM orders WHERE total_eur > 50;
 ERROR:  column "total_eur" does not exist                                              (PostgreSQL)
 → repeat the expression, or wrap the query in a subquery or CTE (Section 14.8) and filter outside it
```

(MySQL relaxes this in a few places as an extension, for example allowing aliases in `HAVING`; portable SQL does not rely on it.)

**Grouping rules.** After step 3, each group becomes one output row, so every `SELECT` expression must have a single value per group: either a grouped column or an aggregate. `SELECT customer_id, status, COUNT(*) … GROUP BY customer_id` is rejected in PostgreSQL ("column "orders.status" must appear in the GROUP BY clause or be used in an aggregate function") and in MySQL with its default `ONLY_FULL_GROUP_BY` mode, because a customer's group may contain several statuses.

**`LIMIT` after `ORDER BY`.** "The top 2" is meaningful only after sorting; without `ORDER BY`, `LIMIT 2` returns any 2 rows.

**Window functions run at step 5**, after `WHERE`, `GROUP BY` and `HAVING`. That is why a window function cannot be used in `WHERE` (Section 14.9 shows the subquery pattern for "filter on a row number").

### Concrete: `DELETE` versus `TRUNCATE` versus `DROP`

A classic interview question compares three ways of removing data:

| | `DELETE FROM t WHERE …` | `TRUNCATE t` | `DROP TABLE t` |
|---|---|---|---|
| Group | DML | DDL | DDL |
| Removes | the rows matching `WHERE` (or all) | all rows | the table itself: structure, data, indexes, constraints |
| How | row by row, logged per row | deallocates the table's storage | removes the table from the catalogue |
| Speed on a large table | slow | fast | fast |
| Fires row-level `DELETE` triggers | yes | no | no |
| Rolls back | yes, inside a transaction | PostgreSQL: yes (transactional); MySQL: no, it commits implicitly | PostgreSQL: yes; MySQL: implicit commit |
| Identity/auto-increment counter | unchanged | MySQL resets it; PostgreSQL only with `RESTART IDENTITY` | gone |
| Blocked by foreign keys | only by rows actually referenced | refused if another table references it (PostgreSQL needs `CASCADE`) | refused if referenced, without `CASCADE` |

### Guarantees and non-guarantees

The logical evaluation order guarantees what a query's result is: which rows are filtered, grouped and returned, in what order. It does not describe the physical execution (the database may use an index to return rows already sorted, or apply `LIMIT` early when it can prove the result is the same), and it does not guarantee any order unless `ORDER BY` is present.

### Failure and diagnosis

- **"Column does not exist" for an alias.** An alias used in `WHERE` or `GROUP BY` (portable SQL). Fix: repeat the expression or filter in an outer query.
- **Wrong totals from filtering in the wrong place.** A filter on rows written in `HAVING` (or a filter on groups attempted in `WHERE` with a subquery that changes meaning). Evidence: compare the result with the step-by-step evaluation on a small sample.
- **Non-deterministic results.** A `LIMIT` without `ORDER BY`, or an `ORDER BY` on a non-unique column with `LIMIT`. Evidence: the same query returns different rows on repeated runs or after a plan change. Fix: order by a unique column combination.

### Backend connection

Read every query in logical order: from where the rows come, which rows survive, how they are grouped, which groups survive, what is computed, in what order, and how many are returned. It turns SQL errors into predictable consequences and makes reviews of complex queries tractable.

---

## 14.7 Joins

### Definitions

A **join** combines rows from two tables into result rows, according to a **join condition**. Conceptually, a join forms every pairing of a row from the left table with a row from the right table (the **Cartesian product**), and keeps the pairings for which the condition is true; outer joins then add back rows that found no partner.

| Join | Keeps |
|---|---|
| **Inner join** | pairs that satisfy the condition |
| **Left outer join** | every inner pair, plus each left row with no partner, padded with `NULL`s on the right |
| **Right outer join** | every inner pair, plus each right row with no partner, padded with `NULL`s on the left |
| **Full outer join** | every inner pair, plus unmatched rows from both sides |
| **Cross join** | every pairing (the Cartesian product); no condition |
| **Self join** | a table joined with itself, under two aliases |
| **Semi-join** | each left row that has **at least one** partner, **once**, with no columns from the right (`EXISTS`) |
| **Anti-join** | each left row that has **no** partner (`NOT EXISTS`) |

### Why it exists

Normalisation (Section 14.5) puts each fact in its own table. Joins put the facts back together for a question: an order with its customer's name, a customer with their orders, products never ordered. Choosing the wrong join type is one of the most common sources of wrong results: rows silently dropped, rows silently duplicated.

### Concrete: two small tables

```text
 customers              orders
 id │ name              id  │ customer_id │ total
 1  │ Ana               101 │ 1           │ 30.00
 2  │ Ben               102 │ 1           │ 12.50
 3  │ Chen              103 │ 2           │ 99.00
                        104 │ NULL        │  5.00     ← a guest checkout: no customer
```

Ana has two orders, Ben one, Chen none, and order 104 has no customer.

**Inner join**

```sql
SELECT c.name, o.id AS order_id, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

```text
 customers ●───● orders          name │ order_id │ total
   ┌─────┬─────┬─────┐           Ana  │ 101      │ 30.00
   │Chen │ ▓▓▓ │ 104 │           Ana  │ 102      │ 12.50
   └─────┴─────┴─────┘           Ben  │ 103      │ 99.00
     only the overlap (▓)         3 rows: Chen (no orders) and 104 (no customer) are dropped
```

**Left outer join**

```sql
SELECT c.name, o.id AS order_id, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```

```text
   ┌─────┬─────┬─────┐           name │ order_id │ total
   │▓Chen│ ▓▓▓ │ 104 │           Ana  │ 101      │ 30.00
   └─────┴─────┴─────┘           Ana  │ 102      │ 12.50
     all customers + overlap      Ben  │ 103      │ 99.00
                                  Chen │ NULL     │ NULL      ← kept, padded with NULLs
```

**Right outer join** (the mirror image; rarely written, because it can always be rewritten as a left join with the tables swapped)

```sql
SELECT c.name, o.id AS order_id, o.total
FROM customers c
RIGHT JOIN orders o ON o.customer_id = c.id;
```

```text
   ┌─────┬─────┬─────┐           name │ order_id │ total
   │Chen │ ▓▓▓ │▓104▓│           Ana  │ 101      │ 30.00
   └─────┴─────┴─────┘           Ana  │ 102      │ 12.50
     overlap + all orders         Ben  │ 103      │ 99.00
                                  NULL │ 104      │  5.00     ← the guest order, no customer
```

**Full outer join**

```sql
SELECT c.name, o.id AS order_id, o.total
FROM customers c
FULL OUTER JOIN orders o ON o.customer_id = c.id;
```

```text
   ┌─────┬─────┬─────┐           name │ order_id │ total
   │▓Chen│ ▓▓▓ │▓104▓│           Ana  │ 101      │ 30.00
   └─────┴─────┴─────┘           Ana  │ 102      │ 12.50
     everything from both         Ben  │ 103      │ 99.00
                                  Chen │ NULL     │ NULL
                                  NULL │ 104      │  5.00
```

MySQL does not support `FULL OUTER JOIN`; emulate it with a left join `UNION` a right join filtered to the right side's unmatched rows.

**Cross join**

```sql
SELECT c.name, o.id FROM customers c CROSS JOIN orders o;
```

```text
 3 customers × 4 orders = 12 rows: (Ana,101) (Ana,102) (Ana,103) (Ana,104) (Ben,101) … (Chen,104)
```

A cross join is useful deliberately (every product with every warehouse, to fill a stock grid) and disastrous accidentally: a forgotten join condition between two tables of 100,000 rows each produces 10 billion rows.

**Self join**

```text
 employees
 id │ name │ manager_id
 1  │ Maya │ NULL
 2  │ Omar │ 1
 3  │ Lea  │ 1
 4  │ Kai  │ 2
```

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
```

```text
 employee │ manager
 Maya     │ NULL          ← kept by the LEFT join; an inner join would drop the top of the hierarchy
 Omar     │ Maya
 Lea      │ Maya
 Kai      │ Omar
```

The two aliases, `e` and `m`, let the same table play two roles. Walking the whole hierarchy to any depth needs a recursive CTE (Section 14.8).

**Semi-join: customers who have ordered**

```sql
SELECT c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

```text
 name
 Ana        ← once, although she has two orders
 Ben
```

An inner join would return Ana twice, once per order. `EXISTS` asks only "is there at least one?", so each customer appears at most once and the database can stop looking after the first match. Using `JOIN … SELECT DISTINCT` for the same question works but does more work and hides the intent.

**Anti-join: customers who have never ordered**

```sql
SELECT c.name
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

```text
 name
 Chen
```

An equivalent form uses a left join and keeps the rows that found no partner:

```sql
SELECT c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;                -- test a column that cannot be NULL in a real match: the key
```

The tempting third form is **wrong** on this data:

```sql
SELECT c.name FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);
```

```text
 name
 (0 rows)       ← Chen is missing
```

Order 104's `customer_id` is `NULL`, and `3 NOT IN (1, 1, 2, NULL)` is not true but **unknown**, so no row qualifies. Section 14.10 explains why; the rule is to write anti-joins with `NOT EXISTS`.

### Mechanism: two traps in outer joins

**A condition on the right table in `WHERE` turns a left join into an inner join.** "List every customer with their orders over 20.00, including customers who have none":

```sql
-- Wrong: the filter runs after the join and removes the NULL-padded rows
SELECT c.name, o.id FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.total > 20;
```

```text
 Ana  │ 101
 Ben  │ 103            Chen is gone: for her row o.total is NULL, and NULL > 20 is unknown
```

```sql
-- Right: the filter is part of the join condition, so it only decides which orders match
SELECT c.name, o.id FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.total > 20;
```

```text
 Ana  │ 101
 Ben  │ 103
 Chen │ NULL           Chen is kept: no order of hers matched, so she is padded
```

**Joining two independent one-to-many relationships multiplies rows (fan-out).** Ana has 2 orders and, in a `tickets` table, 3 support tickets. Joining customers to both:

```sql
SELECT c.name, SUM(o.total) AS spent, COUNT(t.id) AS tickets
FROM customers c
JOIN orders o  ON o.customer_id = c.id
JOIN tickets t ON t.customer_id = c.id
GROUP BY c.name;
```

```text
 before grouping, Ana has 2 × 3 = 6 rows: each order paired with each ticket
 spent = (30.00 + 12.50) × 3 = 127.50     ← three times her real 42.50
 tickets = 6                               ← twice her real 3
```

Fix: aggregate each child table separately, then join the aggregates:

```sql
SELECT c.name, o.spent, t.tickets
FROM customers c
LEFT JOIN (SELECT customer_id, SUM(total) AS spent FROM orders  GROUP BY customer_id) o ON o.customer_id = c.id
LEFT JOIN (SELECT customer_id, COUNT(*) AS tickets FROM tickets GROUP BY customer_id) t ON t.customer_id = c.id;
```

```text
 Ana │ 42.50 │ 3
```

### Guarantees and non-guarantees

Each join type guarantees exactly the rows defined above: inner joins only matches, outer joins every row of the preserved side at least once, semi-joins each qualifying left row once, anti-joins only rows with no partner. Joins do not guarantee that the result has one row per left row (one-to-many joins multiply), that the condition you wrote is the relationship you meant, or any order without `ORDER BY`. How the database executes a join (nested loops, hashing, sorting) is Chapter 16's subject.

### Failure and diagnosis

- **Rows silently missing.** An inner join where a left join was needed, or a right-side filter in `WHERE`. Evidence: count rows of the driving table before and after the join.
- **Totals too large.** Fan-out from joining two one-to-many tables, or joining a non-unique column. Evidence: `COUNT(*)` per group larger than expected; check the join keys' uniqueness. Fix: aggregate before joining.
- **Anti-join returns nothing.** `NOT IN` against a column that contains `NULL`. Fix: `NOT EXISTS`.
- **Enormous results or runaway queries.** A missing or wrong join condition producing a cross join. Evidence: result row counts equal to the product of table sizes; the query plan shows a nested loop without a condition (Chapter 16).

### Backend connection

Choose the join from the question: "with their partners" is inner, "all of these, with partners if any" is left, "those that have any" is `EXISTS`, "those that have none" is `NOT EXISTS`. Put conditions on the optional side in `ON`, aggregate before joining several one-to-many children, and sanity-check row counts on every new query.

---

## 14.8 Aggregation, Subqueries and Common Table Expressions

### Definitions

An **aggregate function** computes one value from a set of rows: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. With `GROUP BY`, it computes one value per group; without it, one value for the whole input.

A **subquery** is a `SELECT` nested inside another statement. A **scalar subquery** returns one value; a **table subquery** returns rows (used with `IN`, `EXISTS`, or in `FROM` as a **derived table**). A subquery is **uncorrelated** if it can be evaluated on its own, and **correlated** if it refers to columns of the outer query, so that its meaning changes for each outer row.

A **common table expression** (CTE) is a named subquery defined with `WITH` before the main query. A **recursive CTE** refers to itself, which lets SQL walk hierarchies and generate sequences.

### Why it exists

Business questions are rarely about single rows: revenue per day, orders per status, customers above their own average, everyone who reports to a given manager at any depth. Aggregation summarises; subqueries let one question use the answer to another; CTEs make multi-step queries readable; recursion handles data whose depth is not known in advance.

### Mechanism 1: aggregation and conditional aggregation

Using the `orders` table from Section 14.6 (customers 1 to 4, statuses `PAID` and `CANCELLED`):

```sql
SELECT customer_id,
       COUNT(*)                                          AS orders,
       COUNT(*) FILTER (WHERE status = 'PAID')           AS paid,
       SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled,
       MAX(total_cents)                                  AS largest
FROM orders
GROUP BY customer_id
ORDER BY customer_id;
```

```text
 customer_id │ orders │ paid │ cancelled │ largest
 1           │ 2      │ 2    │ 0         │ 3000
 2           │ 2      │ 1    │ 1         │ 9900
 3           │ 2      │ 2    │ 0         │ 1500
 4           │ 1      │ 1    │ 0         │ 7000
```

**Conditional aggregation** (an aggregate over only some rows of each group) turns rows into columns in one pass. The `FILTER (WHERE …)` clause is standard SQL and supported by PostgreSQL; the `SUM(CASE …)` form works everywhere, including MySQL.

`GROUP BY` can also produce subtotals: `GROUP BY ROLLUP (region, category)` (PostgreSQL; `GROUP BY region, category WITH ROLLUP` in MySQL) adds a row per region with all categories combined, and a grand-total row, marking the rolled-up columns with `NULL`.

### Mechanism 2: uncorrelated and correlated subqueries

**Uncorrelated:** orders larger than the overall average.

```sql
SELECT id, total_cents
FROM orders
WHERE total_cents > (SELECT AVG(total_cents) FROM orders);
```

```text
 inner query, evaluated once: (3000 + 2500 + 9900 + 4000 + 1200 + 1500 + 7000) / 7 = 29100 / 7 ≈ 4157.14

 id  │ total_cents
 103 │ 9900
 107 │ 7000
```

**Correlated:** orders larger than **that customer's** average.

```sql
SELECT o.id, o.customer_id, o.total_cents
FROM orders o
WHERE o.total_cents > (SELECT AVG(i.total_cents)
                       FROM orders i
                       WHERE i.customer_id = o.customer_id);      -- refers to the outer row
```

```text
 the inner query is defined per outer row:
   customer 1: avg(3000, 2500) = 2750   → 101 (3000) qualifies, 102 does not
   customer 2: avg(9900, 4000) = 6950   → 103 (9900) qualifies
   customer 3: avg(1200, 1500) = 1350   → 106 (1500) qualifies
   customer 4: avg(7000)       = 7000   → 107 is not greater than its own average

 id  │ customer_id │ total_cents
 101 │ 1           │ 3000
 103 │ 2           │ 9900
 106 │ 3           │ 1500
```

Conceptually, a correlated subquery runs once per outer row. Optimisers often rewrite it into a join with a pre-aggregated table, so it is not necessarily slow (Chapter 16), but on large tables it is worth checking. The same result can be written with a window function, which makes one pass (Section 14.9):

```sql
SELECT id, customer_id, total_cents
FROM (SELECT o.*, AVG(total_cents) OVER (PARTITION BY customer_id) AS cust_avg FROM orders o) t
WHERE total_cents > cust_avg;
```

A scalar subquery must return at most one row. If it returns two, the statement fails at run time ("more than one row returned by a subquery used as an expression" in PostgreSQL), which is a common production surprise when data that "always" had one match gets a second.

### Mechanism 3: CTEs

A CTE names an intermediate result, so a query reads as a sequence of steps:

```sql
WITH paid_totals AS (
    SELECT customer_id, SUM(total_cents) AS spent
    FROM orders
    WHERE status = 'PAID'
    GROUP BY customer_id
),
big_spenders AS (
    SELECT customer_id, spent FROM paid_totals WHERE spent > 5000
)
SELECT c.name, b.spent
FROM big_spenders b
JOIN customers c ON c.id = b.customer_id
ORDER BY b.spent DESC;
```

A CTE is the readable alternative to nesting subqueries three levels deep, and the way to filter on an alias or a window function (Sections 14.6 and 14.9). On performance: PostgreSQL before version 12 always computed a CTE separately (an "optimisation fence"); from version 12 it inlines a non-recursive CTE that is used once, unless written `WITH … AS MATERIALIZED`. MySQL supports CTEs from version 8.0.

### Mechanism 4: recursive CTEs

A recursive CTE has an **anchor** query (the starting rows) and a **recursive** query that refers to the CTE itself, joined with `UNION ALL`. The database runs the anchor, then repeatedly runs the recursive part on the rows produced by the previous iteration, until an iteration produces no rows.

Everyone in Maya's reporting chain, using the `employees` table of Section 14.7:

```sql
WITH RECURSIVE chain AS (
    SELECT id, name, 1 AS depth, CAST(name AS varchar(500)) AS path    -- anchor
    FROM employees
    WHERE id = 1
  UNION ALL
    SELECT e.id, e.name, c.depth + 1, CAST(c.path || ' > ' || e.name AS varchar(500))   -- recursive step
    FROM employees e
    JOIN chain c ON e.manager_id = c.id
)
SELECT name, depth, path FROM chain ORDER BY path;
```

```text
 iteration   rows produced (working set)
 anchor      Maya (depth 1, "Maya")
 1           employees whose manager is in {Maya}:       Omar (2, "Maya > Omar"), Lea (2, "Maya > Lea")
 2           employees whose manager is in {Omar, Lea}:  Kai (3, "Maya > Omar > Kai")
 3           employees whose manager is in {Kai}:        none → stop

 name │ depth │ path
 Maya │ 1     │ Maya
 Lea  │ 2     │ Maya > Lea
 Omar │ 2     │ Maya > Omar
 Kai  │ 3     │ Maya > Omar > Kai
```

(MySQL uses `CONCAT(c.path, ' > ', e.name)` instead of `||`, which in MySQL means logical OR by default.)

Recursive CTEs also generate sequences, such as every day in a range, which Section 14.13 uses to find gaps:

```sql
WITH RECURSIVE days AS (
    SELECT DATE '2026-10-01' AS day
  UNION ALL
    SELECT day + 1 FROM days WHERE day < DATE '2026-10-05'     -- PostgreSQL date + integer = next day
)
SELECT day FROM days;            -- 2026-10-01, 10-02, 10-03, 10-04, 10-05
```

### Guarantees and non-guarantees

Aggregates guarantee one value per group computed over exactly the rows that survived `WHERE`. A recursive CTE guarantees termination only if the data and the recursive step eventually stop producing rows: a cycle in the data (an employee who is their own indirect manager) makes it run until a depth limit or resource limit stops it. Correlated subqueries guarantee per-row meaning, not per-row execution; the plan may differ.

### Failure and diagnosis

- **Runaway recursion.** A recursive CTE runs until it errors or fills memory. Cause: a cycle in the data. Guard: a depth condition (`WHERE c.depth < 50`), or a cycle check (PostgreSQL 14 added a `CYCLE` clause; otherwise track visited IDs in an array or path). Prevent: constraints or checks that stop cycles being written.
- **Scalar subquery returned two rows.** A production error after a data change. Fix: decide which row is meant (`ORDER BY … LIMIT 1`, or an aggregate), or add the `UNIQUE` constraint the query assumed.
- **Counts inflated by joins inside aggregates.** See fan-out in Section 14.7.

### Backend connection

Aggregate in the database rather than in application loops: it moves one result over the network instead of every row. Use CTEs to make multi-step queries reviewable, recursive CTEs for hierarchies of unknown depth (with a cycle guard), and remember that a subquery's meaning is per row while its cost depends on the plan.

---

## 14.9 Window Functions

### Definition

A **window function** computes a value for each row from a set of related rows, called its **window**, **without collapsing** those rows into one. It is written `function(...) OVER (PARTITION BY … ORDER BY … frame)`:

- **`PARTITION BY`** divides rows into independent groups (like `GROUP BY`, but every row stays).
- **`ORDER BY`** orders rows within each partition, which ranking and offset functions need.
- The **frame** (`ROWS BETWEEN …` or `RANGE BETWEEN …`) chooses which rows of the partition, relative to the current row, an aggregate such as `SUM` covers.

### Why it exists

Many questions need a row **and** information about its neighbours or its group: each sale with its rank in the region, each day's revenue with yesterday's, a running balance, each order's share of the customer's total. `GROUP BY` loses the individual rows; self-joins and correlated subqueries can express these questions but are verbose and often slow. Window functions answer them in one pass over the data.

### Concrete: ranking, with ties

```text
 sales
 rep │ region │ amount
 Ana │ North  │ 500
 Ben │ North  │ 300
 Cy  │ North  │ 300
 Dee │ North  │ 200
 Eve │ South  │ 700
 Fay │ South  │ 400
 Gus │ South  │ 400
```

```sql
SELECT rep, region, amount,
       ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC, rep) AS row_num,
       RANK()       OVER (PARTITION BY region ORDER BY amount DESC)      AS rnk,
       DENSE_RANK() OVER (PARTITION BY region ORDER BY amount DESC)      AS dense_rnk,
       SUM(amount)  OVER (PARTITION BY region)                           AS region_total
FROM sales
ORDER BY region, amount DESC, rep;
```

```text
 rep │ region │ amount │ row_num │ rnk │ dense_rnk │ region_total
 Ana │ North  │ 500    │ 1       │ 1   │ 1         │ 1300
 Ben │ North  │ 300    │ 2       │ 2   │ 2         │ 1300
 Cy  │ North  │ 300    │ 3       │ 2   │ 2         │ 1300
 Dee │ North  │ 200    │ 4       │ 4   │ 3         │ 1300
 Eve │ South  │ 700    │ 1       │ 1   │ 1         │ 1500
 Fay │ South  │ 400    │ 2       │ 2   │ 2         │ 1500
 Gus │ South  │ 400    │ 3       │ 2   │ 2         │ 1500
```

The three ranking functions differ only on ties:

- **`ROW_NUMBER`** numbers rows 1, 2, 3, 4 with no ties. Between equal values the order is arbitrary unless the `ORDER BY` includes a tie-breaker (here `rep`), so always add one.
- **`RANK`** gives equal values the same rank and then **skips**: 1, 2, 2, 4.
- **`DENSE_RANK`** gives equal values the same rank **without gaps**: 1, 2, 2, 3.

`SUM(amount) OVER (PARTITION BY region)` attaches the region's total to every row: a `GROUP BY` result without losing the rows, from which `amount * 100.0 / region_total` gives each rep's share.

### Concrete: comparing with neighbouring rows

```text
 daily_revenue
 day        │ revenue
 2026-10-01 │ 100
 2026-10-02 │ 150
 2026-10-03 │ 120
 2026-10-04 │ 180
```

```sql
SELECT day, revenue,
       LAG(revenue)  OVER (ORDER BY day)                                  AS prev_day,
       revenue - LAG(revenue) OVER (ORDER BY day)                         AS change,
       LEAD(revenue) OVER (ORDER BY day)                                  AS next_day,
       SUM(revenue)  OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
       ROUND(AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3
FROM daily_revenue
ORDER BY day;
```

```text
 day        │ revenue │ prev_day │ change │ next_day │ running_total │ moving_avg_3
 2026-10-01 │ 100     │ NULL     │ NULL   │ 150      │ 100           │ 100.00
 2026-10-02 │ 150     │ 100      │ 50     │ 120      │ 250           │ 125.00
 2026-10-03 │ 120     │ 150      │ -30    │ 180      │ 370           │ 123.33
 2026-10-04 │ 180     │ 120      │ 60     │ NULL     │ 550           │ 150.00
```

`LAG(x)` reads `x` from the previous row of the window's order, `LEAD(x)` from the next; both return `NULL` at the edges unless given a default (`LAG(revenue, 1, 0)`). The moving average covers at most three rows: the current one and the two before it, so the first two rows average over fewer.

### Mechanism: frames, and the default that surprises

When a window has an `ORDER BY` but no explicit frame, the standard default frame is **`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`**. `RANGE` treats rows with equal `ORDER BY` values (**peers**) as one position, so all peers are included together. With ties, a "running total" is then not what most people expect:

```sql
SELECT rep, amount,
       SUM(amount) OVER (ORDER BY amount)                                              AS default_frame,
       SUM(amount) OVER (ORDER BY amount, rep ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS rows_frame
FROM sales WHERE region = 'North'
ORDER BY amount, rep;
```

```text
 rep │ amount │ default_frame │ rows_frame
 Dee │ 200    │ 200           │ 200
 Ben │ 300    │ 800           │ 500        ← RANGE includes Cy (a peer, also 300) already
 Cy  │ 300    │ 800           │ 800
 Ana │ 500    │ 1300          │ 1300
```

For running totals, state the frame explicitly with `ROWS` and order by a unique combination. The same default makes `LAST_VALUE(x) OVER (ORDER BY …)` return the current row's value (the frame ends at the current row), not the partition's last value; it needs `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

### Concrete: top N per group

"The top two reps in each region" needs a ranking per partition and a filter on it. A window function is computed at step 5 of the logical order (Section 14.6), after `WHERE`, so it cannot be filtered in the same query's `WHERE`; wrap it:

```sql
SELECT rep, region, amount
FROM (
    SELECT rep, region, amount,
           ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC, rep) AS rn
    FROM sales
) ranked
WHERE rn <= 2
ORDER BY region, rn;
```

```text
 rep │ region │ amount
 Ana │ North  │ 500
 Ben │ North  │ 300        ← Cy (also 300) is excluded by the tie-breaker on rep
 Eve │ South  │ 700
 Fay │ South  │ 400        ← Gus (also 400) excluded likewise
```

Replacing `ROW_NUMBER` with `RANK` and `rn <= 2` with `rnk <= 2` includes ties: North returns Ana, Ben and Cy; South returns Eve, Fay and Gus. Which is right is a business decision ("exactly two winners" or "everyone tied for second place"), and the choice of function encodes it.

MySQL supports window functions from version 8.0; on 5.7 these queries need self-joins or variables.

### Guarantees and non-guarantees

A window function guarantees a value for every input row computed over that row's partition and frame, without changing the number of rows. It does not guarantee a deterministic `ROW_NUMBER` between ties without a tie-breaker, or the frame you assumed when you leave it to the default. Windows are computed after `WHERE` and `GROUP BY`, so they see only the surviving rows.

### Failure and diagnosis

- **Running totals jump at ties.** The default `RANGE` frame. Evidence: equal values share a cumulative value. Fix: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` with a unique ordering.
- **"Window functions are not allowed in WHERE".** A filter on a window result in the same query. Fix: a subquery or CTE.
- **Different "top N" results between runs.** `ROW_NUMBER` without a tie-breaker. Fix: add a unique column to the window's `ORDER BY`.

### Backend connection

Use window functions for rankings, period-over-period comparisons, running balances and per-group top-N in a single query, instead of self-joins, correlated subqueries or post-processing in application code. Always give ranking windows a tie-breaker and running aggregates an explicit `ROWS` frame.

---

## 14.10 NULL and Three-Valued Logic

### Definitions

**`NULL`** is SQL's marker for a missing value: unknown (a customer's city not yet given), not applicable (a cancellation reason on an order that was never cancelled), or absent (the right side of an unmatched outer join row). `NULL` is not a value: it is not zero, not an empty string, and not equal to anything, including another `NULL`.

**Three-valued logic** is the logic SQL uses for conditions: a condition is **TRUE**, **FALSE** or **UNKNOWN**. Any comparison with `NULL` (`=`, `<>`, `<`, `>`, `IN`) yields UNKNOWN.

### Why it exists

Real data has gaps, and a database needs some way to record "we do not know". `NULL` provides one marker for all of them, at the cost of a logic in which "not equal to X" and "not in this list" stop behaving like their everyday meanings. Most `NULL` bugs are not about `NULL` itself but about forgetting that a condition can be UNKNOWN, and that `WHERE` keeps only TRUE.

### Mechanism: the truth tables

```text
 AND     │ TRUE     FALSE    UNKNOWN          OR      │ TRUE   FALSE    UNKNOWN        NOT
 ────────┼─────────────────────────────         ────────┼──────────────────────────      ─────────────────
 TRUE    │ TRUE     FALSE    UNKNOWN          TRUE    │ TRUE   TRUE     TRUE           TRUE    → FALSE
 FALSE   │ FALSE    FALSE    FALSE            FALSE   │ TRUE   FALSE    UNKNOWN        FALSE   → TRUE
 UNKNOWN │ UNKNOWN  FALSE    UNKNOWN          UNKNOWN │ TRUE   UNKNOWN  UNKNOWN        UNKNOWN → UNKNOWN
```

A way to remember them: treat UNKNOWN as "could be either". `FALSE AND anything` is FALSE whatever the unknown turns out to be; `TRUE OR anything` is TRUE; everything else involving UNKNOWN stays UNKNOWN.

How each clause treats UNKNOWN:

| Clause | Keeps / accepts when the condition is |
|---|---|
| `WHERE`, `HAVING`, `ON` | TRUE only (UNKNOWN is treated like FALSE) |
| `CHECK` constraint | not FALSE (UNKNOWN is accepted, Section 14.3) |
| `CASE WHEN` | TRUE only; UNKNOWN falls through to the next branch |

### Concrete: the `NOT IN` pitfall

```text
 x NOT IN (1, 2, NULL)    means    x <> 1  AND  x <> 2  AND  x <> NULL

 x = 3:   TRUE  AND TRUE  AND UNKNOWN  =  UNKNOWN   → row NOT kept
 x = 1:   FALSE AND TRUE  AND UNKNOWN  =  FALSE     → row not kept
 → with a NULL in the list, NOT IN is never TRUE, so it returns no rows at all
```

```text
 x IN (1, 2, NULL)        means    x = 1  OR  x = 2  OR  x = NULL

 x = 1:   TRUE  OR …                 =  TRUE      → kept
 x = 3:   FALSE OR FALSE OR UNKNOWN  =  UNKNOWN   → not kept (correctly: 3 is not in the list)
 → IN behaves as expected; only its negation breaks
```

This is why the anti-join with `NOT IN` in Section 14.7 returned no rows: the subquery's results included order 104's `NULL` customer. `NOT EXISTS` does not have the problem, because it asks whether a matching row **exists**, and a row whose `customer_id` is `NULL` never matches (`NULL = 3` is UNKNOWN, so it is not a match), which is the intended meaning.

```sql
-- Wrong whenever orders.customer_id can be NULL
SELECT name FROM customers WHERE id NOT IN (SELECT customer_id FROM orders);
-- Right
SELECT name FROM customers c WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
-- Also right, if you insist on NOT IN
SELECT name FROM customers WHERE id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);
```

### Mechanism: testing for NULL, and comparing safely

- `col = NULL` is always UNKNOWN; use **`col IS NULL`** and **`col IS NOT NULL`**.
- **`a IS DISTINCT FROM b`** (standard SQL, supported by PostgreSQL) compares treating two `NULL`s as equal and `NULL` versus a value as different: it is always TRUE or FALSE. MySQL's equivalent is the null-safe equality operator `a <=> b` (and `NOT (a <=> b)`).
- **`COALESCE(a, b, …)`** returns the first non-`NULL` argument: `COALESCE(city, 'unknown')`.
- **`NULLIF(a, b)`** returns `NULL` if `a = b`, otherwise `a`; the classic use is avoiding division by zero: `revenue / NULLIF(orders, 0)`.
- Arithmetic and most functions with a `NULL` argument return `NULL`: `price * NULL` is `NULL`; in PostgreSQL `'a' || NULL` is `NULL`, and MySQL's `CONCAT('a', NULL)` is `NULL` too (`CONCAT_WS` skips `NULL`s).

### Concrete: NULL in aggregates

```text
 reviews
 id │ rating
 1  │ 5
 2  │ NULL      ← review text without a star rating
 3  │ 3
 4  │ 4
```

```sql
SELECT COUNT(*)       AS all_rows,
       COUNT(rating)  AS rated,
       SUM(rating)    AS total,
       AVG(rating)    AS avg_rating,
       SUM(rating) * 1.0 / COUNT(*) AS avg_over_all_rows
FROM reviews;
```

```text
 all_rows │ rated │ total │ avg_rating │ avg_over_all_rows
 4        │ 3     │ 12    │ 4.0        │ 3.0
```

Aggregates other than `COUNT(*)` **ignore** `NULL`s. That is usually what you want (the unrated review should not count as zero stars), but it must be a conscious choice: `AVG(rating)` is 4.0, not 3.0. Two more rules: `SUM` over a set with no non-`NULL` values returns `NULL`, not 0 (use `COALESCE(SUM(x), 0)` for totals), and `COUNT(*)` of an empty set is 0 while `MAX` of an empty set is `NULL`.

### Mechanism: NULL in grouping, uniqueness and sorting

| Operation | How `NULL`s are treated |
|---|---|
| `GROUP BY`, `DISTINCT`, `UNION`, `INTERSECT`, `EXCEPT` | all `NULL`s form **one** group or are one distinct value (unlike `=`) |
| `UNIQUE` constraint | `NULL`s are **not** equal, so many rows may have `NULL` in a unique column (PostgreSQL 15 adds `UNIQUE NULLS NOT DISTINCT` to allow only one) |
| Equality join `ON a.x = b.x` | rows with `NULL` keys never match |
| `ORDER BY … ASC` | PostgreSQL puts `NULL`s **last** (it treats `NULL` as larger than any value) and supports `NULLS FIRST/LAST`; MySQL puts them **first** and has no `NULLS FIRST/LAST` (use `ORDER BY col IS NULL, col`) |

### Guarantees and non-guarantees

Three-valued logic guarantees consistent, defined results for every expression involving `NULL`. It does not guarantee intuitive ones: `WHERE status <> 'CANCELLED'` silently excludes rows whose status is `NULL`; `NOT IN` with a `NULL` returns nothing; a `UNIQUE` column can hold many `NULL`s; `NULL` sorts at different ends in different databases.

### Failure and diagnosis

- **Rows missing from a "not equal" filter.** Evidence: `SELECT COUNT(*) WHERE col IS NULL` shows the missing rows. Fix: `WHERE col IS DISTINCT FROM 'X'`, or `col <> 'X' OR col IS NULL`, or make the column `NOT NULL`.
- **Empty anti-join.** `NOT IN` with a nullable subquery column. Fix: `NOT EXISTS`.
- **Averages or totals that look wrong.** `AVG` ignoring `NULL`s, or `SUM` returning `NULL`. Evidence: compare `COUNT(*)` with `COUNT(col)`.
- **Duplicate "unique" rows.** Several rows with `NULL` in a column covered by a unique constraint. Fix: `NOT NULL`, or `NULLS NOT DISTINCT` where available.

### Backend connection

Make columns `NOT NULL` unless "missing" is a real state the business needs, so that most queries never meet three-valued logic. Where `NULL` is allowed, test with `IS NULL`, compare with `IS DISTINCT FROM`, write anti-joins with `NOT EXISTS`, wrap totals in `COALESCE`, and decide deliberately whether averages include missing values. Map `NULL` carefully to JSON too: `null` and an absent field are different things in an API (Chapter 12, Section 12.7).

---

## 14.11 Set Operations, Upserts, Views and Materialised Views

### Definitions

**Set operations** combine the results of two queries with the same number of columns and compatible types: **`UNION`** (rows in either, duplicates removed), **`UNION ALL`** (rows in either, duplicates kept), **`INTERSECT`** (rows in both) and **`EXCEPT`** (rows in the first but not the second; `MINUS` in some databases).

An **upsert** inserts a row, or updates the existing row if one with the same key already exists, as a single atomic statement.

A **view** is a named, stored query that can be used like a table. It stores no data; each use runs the query. A **materialised view** stores the query's result as data, which must be **refreshed** to reflect later changes.

### Why they exist

Set operations combine results that come from different tables or conditions (all contact emails from customers and newsletter subscribers). Upserts solve a concurrency problem: "insert if absent, else update" written as a `SELECT` followed by an `INSERT` or `UPDATE` is the check-then-act race of Chapter 5 (Section 5.2), and two concurrent requests can both decide to insert. Views give a stable, simpler or restricted interface over tables (logical data independence, Section 14.1). Materialised views make expensive queries cheap to read, at the price of staleness.

### Concrete: set operations

```text
 subscribers(email)        customers(email)
 ana@example.com           ana@example.com
 chen@example.com          ben@example.com
 dan@example.com           chen@example.com
```

```sql
SELECT email FROM subscribers UNION     SELECT email FROM customers;   -- ana, ben, chen, dan        (4 rows)
SELECT email FROM subscribers UNION ALL SELECT email FROM customers;   -- ana, chen, dan, ana, ben, chen (6 rows)
SELECT email FROM subscribers INTERSECT SELECT email FROM customers;   -- ana, chen                  (2 rows)
SELECT email FROM subscribers EXCEPT    SELECT email FROM customers;   -- dan                        (1 row)
```

Rules: column names come from the first query; `ORDER BY` applies to the combined result and goes at the end; set operations compare `NULL`s as equal (Section 14.10). `UNION` must sort or hash the combined rows to remove duplicates, so prefer **`UNION ALL`** whenever duplicates are impossible or wanted: it is cheaper and states the intent. MySQL supports `INTERSECT` and `EXCEPT` from version 8.0.31.

### Mechanism: upserts

Adding a product to a cart: if the cart already contains it, increase the quantity; otherwise insert it. The cart table has the key `(cart_id, product_id)`.

```sql
-- PostgreSQL (and SQLite): ON CONFLICT names the unique constraint that decides "already exists"
INSERT INTO cart_items (cart_id, product_id, quantity)
VALUES (7, 17, 2)
ON CONFLICT (cart_id, product_id)
DO UPDATE SET quantity = cart_items.quantity + EXCLUDED.quantity;   -- EXCLUDED = the row we tried to insert
```

```text
 before                          after
 (no row for cart 7, product 17) (7, 17, 2)       ← inserted
 (7, 17, 1)                      (7, 17, 3)       ← conflict on the key → updated: 1 + 2
```

```sql
-- MySQL: fires on a duplicate in ANY unique index of the table
INSERT INTO cart_items (cart_id, product_id, quantity)
VALUES (7, 17, 2) AS new
ON DUPLICATE KEY UPDATE quantity = cart_items.quantity + new.quantity;   -- row alias: MySQL 8.0.19+
```

Variants and cautions:

- **`ON CONFLICT DO NOTHING`** inserts only if absent: the "claim" pattern of Chapter 11 (Section 11.7) and the idempotency-key insert of Chapter 13 (Section 13.5).
- **Idempotency.** The upsert above **adds** quantity, so repeating it adds again: a retried request changes the cart twice. `DO UPDATE SET quantity = EXCLUDED.quantity` (set, not add) is idempotent. The choice is the same as in Chapter 13's idempotent design.
- **MySQL's trigger is any unique key.** With several unique indexes, a conflict on an unexpected one updates a row you did not intend.
- **`MERGE`** is the SQL standard's general statement for "match rows from a source against a target and insert, update or delete". PostgreSQL supports it from version 15; MySQL does not. For a single-row upsert under concurrency, PostgreSQL's documentation recommends `INSERT … ON CONFLICT`, because `MERGE` can still fail with a unique violation when concurrent transactions insert the same key. How the database makes `ON CONFLICT` atomic is Chapter 17's subject.

### Mechanism: views

```sql
CREATE VIEW active_products AS
SELECT id, sku, name, price_cents
FROM products
WHERE NOT discontinued;

SELECT name FROM active_products WHERE price_cents < 2000;
-- the database substitutes the view's definition, roughly:
-- SELECT name FROM (SELECT id, sku, name, price_cents FROM products WHERE NOT discontinued) v
-- WHERE price_cents < 2000;
```

Views are used to:

- **simplify**: name a complex join once (`order_summaries` joining orders, customers and line totals);
- **restrict**: grant a reporting role access to a view that exposes only some columns and rows, not to the underlying table;
- **preserve compatibility**: after a table is split or renamed, a view with the old name and shape keeps old queries working during a migration (Chapter 18).

A view over a single table, without aggregation, `DISTINCT`, `GROUP BY`, set operations or window functions, is **automatically updatable** in PostgreSQL and MySQL: `UPDATE active_products SET price_cents = 1899 WHERE id = 17` changes the underlying row. **`WITH CHECK OPTION`** makes the database reject inserts and updates through the view that would produce rows the view cannot see (for example, inserting a discontinued product through `active_products`). Views with joins or aggregates are generally not updatable (Section 14.14 has the details).

### Mechanism: materialised views

```sql
-- PostgreSQL
CREATE MATERIALIZED VIEW daily_sales AS
SELECT o.created_at::date AS day, p.category, SUM(l.quantity * l.unit_cents) AS revenue_cents
FROM orders o
JOIN order_lines l ON l.order_id = o.id
JOIN products p    ON p.id = l.product_id
WHERE o.status = 'PAID'
GROUP BY 1, 2;

CREATE UNIQUE INDEX ON daily_sales (day, category);   -- required for CONCURRENTLY

REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;   -- e.g. every 10 minutes from a scheduler
```

```text
 time  ──────────────────────────────────────────────────────────────────►
       refresh          orders keep arriving             refresh
         │  daily_sales shows the state as of the last refresh │
         ▼  (reads are fast; data up to 10 minutes stale)      ▼
```

A plain `REFRESH MATERIALIZED VIEW` recomputes the view and blocks reads of it while doing so; `CONCURRENTLY` lets readers continue with the old contents, at the cost of a slower refresh and the unique index. MySQL has no materialised views; the same effect is built with a summary table and a scheduled job (or maintained incrementally by the write path, Section 14.5). A materialised view is a form of denormalisation that the database recomputes for you.

### Guarantees and non-guarantees

`UNION` and `INTERSECT` guarantee duplicate-free results; `UNION ALL` guarantees nothing is lost. An upsert guarantees that "insert or update" is decided atomically against the unique constraint it names (in PostgreSQL), so two concurrent upserts cannot both insert. A view guarantees always-current results with the view's restrictions; a materialised view guarantees fast reads of data **as of the last refresh** only. None of them guarantees performance: a view hides a query, it does not make it cheaper.

### Failure and diagnosis

- **Duplicate rows from a race.** "Insert if absent" written as `SELECT` then `INSERT`. Evidence: duplicate key errors, or duplicates if no constraint exists. Fix: a unique constraint plus `ON CONFLICT`.
- **A retried upsert doubles a value.** An additive `DO UPDATE`. Fix: set, not add, or idempotency keys.
- **Slow queries hidden behind views.** Simple-looking `SELECT * FROM order_summaries WHERE …` is slow. Evidence: the query plan expands the view's joins (Chapter 16). Fix: query what is needed; index the underlying tables.
- **Stale dashboard numbers.** A materialised view not refreshed, or refresh failing. Evidence: compare its newest row with the source; check the scheduler's logs.

### Backend connection

Use `UNION ALL` by default, `ON CONFLICT` (or its MySQL equivalent) for every insert-or-update, with set semantics where retries are possible, views for stable or restricted interfaces and migrations, and materialised views or summary tables for expensive reads that can tolerate a known staleness.

---

## 14.12 Data Types That Matter

### Definition

A column's **data type** decides which values it can hold, how they are compared and sorted, how arithmetic on them behaves, and how much space they take. Four families cause most real bugs: numbers used for money, timestamps and time zones, text and its collation, and JSON.

### Why it matters

Type mistakes do not fail loudly. A float column accepts money and loses a cent once in a while; a timestamp without a time zone accepts local times and makes a daily report wrong twice a year; a collation silently treats two emails as equal or as different. The bugs surface months later as reconciliation differences and duplicate accounts, long after the schema was written.

### Mechanism 1: money is exact, floats are not

**Floating-point types** (`real`, `double precision`, MySQL's `FLOAT` and `DOUBLE`) store numbers in binary, as IEEE 754 values. Most decimal fractions, including 0.1, have no exact binary representation, so they are stored as the nearest binary value and errors accumulate. **Exact numeric types** (`numeric(p, s)`, `decimal(p, s)`) store decimal digits exactly, with `p` total digits and `s` after the decimal point.

```sql
-- PostgreSQL
SELECT 0.1::float8 + 0.2::float8              AS float_sum,      -- 0.30000000000000004
       0.1::numeric + 0.2::numeric            AS numeric_sum,    -- 0.3
       0.1::float8 + 0.2::float8 = 0.3::float8 AS float_equal;   -- false
```

```text
 a table with ten payments of 0.10 each
 column type          SUM(amount)
 double precision     0.9999999999999999
 numeric(10,2)        1.00
```

An accounting report that compares a sum of floats with a bank statement will disagree by fractions of a cent, and comparisons such as `WHERE amount = 0.3` will miss rows. For money:

- store it as `numeric(12, 2)` (or wider), or as an integer number of **minor units** (`bigint` cents), as Chapter 12 (Section 12.3) did in code;
- store the **currency** beside it, because minor units differ (a yen has none, a Kuwaiti dinar has three fils digits);
- round at defined points with a defined rule (tax per line or per invoice; half-up or half-even), because exact types still round when you divide or multiply by rates;
- avoid PostgreSQL's `money` type, whose output and input depend on a locale setting.

Integers have their own limit: `integer` holds up to about 2.1 billion. A counter or an identity column that passes that fails with an "out of range" error; use `bigint` for identifiers and running totals.

### Mechanism 2: timestamps and time zones

An **instant** is a point on the global timeline ("2026-10-21 08:00 UTC"). A **local date-time** is a wall-clock reading without a zone ("2026-10-21 10:00"), which names different instants in Berlin and in Chennai. A **time zone** (such as `Europe/Berlin`) is a set of rules mapping instants to local times, including daylight-saving changes; an **offset** (`+02:00`) is the difference from UTC at one moment.

| Type | Stores | Use for |
|---|---|---|
| PostgreSQL `timestamp with time zone` (`timestamptz`) | an instant, normalised to UTC; shown in the session's `TimeZone` | when something happened: created, paid, shipped |
| PostgreSQL `timestamp` (without time zone) | a local date-time, no zone | rarely; local schedules together with a separate zone column |
| MySQL `TIMESTAMP` | an instant (converted from and to the session `time_zone` via UTC); range 1970 to 2038-01-19 03:14:07 UTC | instants, if the 2038 limit is acceptable |
| MySQL `DATETIME` | a local date-time, no zone | instants **in UTC by convention** (the application always writes UTC), or local schedules |
| `date` | a calendar date | birthdays, business days |

```sql
-- PostgreSQL: one stored instant, shown in two session time zones
SET TIME ZONE 'UTC';            SELECT timestamptz '2026-10-21 10:00:00+02';   -- 2026-10-21 08:00:00+00
SET TIME ZONE 'Europe/Berlin';  SELECT timestamptz '2026-10-21 10:00:00+02';   -- 2026-10-21 10:00:00+02
```

The classic pitfalls, using Berlin's 2026 daylight-saving changes (clocks go forward at 02:00 on Sunday 29 March and back at 03:00 on Sunday 25 October):

```text
 1. Local times that do not exist.   2026-03-29 02:30 Europe/Berlin never happens: the clock jumps 02:00 → 03:00.
                                     Databases and libraries resolve it by a rule (often shifting it by the gap)
                                     rather than rejecting it; check what yours does.
 2. Local times that happen twice.   2026-10-25 02:30 Europe/Berlin occurs at 02:30+02 and again at 02:30+01.
                                     Stored without an offset, the two are indistinguishable.
 3. "One day" is not 24 hours.       SET TIME ZONE 'Europe/Berlin';
                                     SELECT timestamptz '2026-03-28 12:00' + interval '1 day';    -- 2026-03-29 12:00+02
                                     SELECT timestamptz '2026-03-28 12:00' + interval '24 hours'; -- 2026-03-29 13:00+02
                                     (PostgreSQL adds calendar days in the session's zone; that day has 23 hours)
 4. "Today" depends on the zone.     WHERE created_at::date = current_date   uses the SESSION time zone: the same
                                     query gives a different "day" on a server set to UTC and one set to Berlin.
```

The fourth pitfall produces reports that disagree: finance's "orders on 21 October" (Berlin days) and the dashboard's (UTC days) differ by everything ordered between 00:00 and 02:00 Berlin time. State the boundaries explicitly:

```sql
SELECT COUNT(*) FROM orders
WHERE created_at >= timestamptz '2026-10-21 00:00 Europe/Berlin'
  AND created_at <  timestamptz '2026-10-22 00:00 Europe/Berlin';
```

(The range form also lets the database use an index on `created_at`, which casting the column to a date can prevent; Chapter 16.)

The rules that follow: store instants as `timestamptz` (or as UTC by convention); convert to local time only for display and for business-day boundaries, with an explicit zone; and store **future local events** (a delivery slot "09:00 Berlin on 15 June next year") as a local date-time plus a zone name, not as an instant, because a government can change the zone's rules before then.

### Mechanism 3: text, encodings and collations

- **Types.** `varchar(n)` limits length in characters in PostgreSQL and MySQL; `text` has no declared limit; `char(n)` pads with spaces and is rarely what you want. Writing a longer value fails in PostgreSQL (`value too long for type character varying(200)`) and in MySQL's default strict mode; in non-strict MySQL it is silently truncated with a warning.
- **Encoding.** Use UTF-8 throughout (Chapter 7, Section 7.7). In MySQL, the character set called `utf8` is a three-byte subset (`utf8mb3`) that cannot store characters outside the Basic Multilingual Plane, such as emoji; use **`utf8mb4`**.
- **Collation** is the set of rules for comparing and sorting strings: whether case matters, whether accents matter, and the order of letters. It affects `=`, `ORDER BY`, `GROUP BY`, `DISTINCT` and **unique constraints**.

```text
 comparison                     PostgreSQL default (deterministic)   MySQL 8 default (utf8mb4_0900_ai_ci)
 'Ana' = 'ana'                  false                                true   (ci = case-insensitive)
 'resume' = 'résumé'            false                                true   (ai = accent-insensitive)
 UNIQUE(email) with             two rows allowed                     second insert rejected as a duplicate
   'Ana@x.com' and 'ana@x.com'
```

Neither default is wrong, but each surprises someone. In PostgreSQL, case-insensitive uniqueness for emails needs an expression index (`CREATE UNIQUE INDEX ON users (lower(email))`), the `citext` extension, or a non-deterministic ICU collation; queries must then compare `lower(email) = lower($1)`. In MySQL, a column that must distinguish case or accents (passwords are never compared in SQL, but codes and tokens are) needs a binary or case-sensitive collation.

### Mechanism 4: JSON columns

PostgreSQL offers `json` (stores the text as given) and `jsonb` (stores a parsed binary form: duplicate keys removed, key order not kept, faster to query, indexable with GIN indexes, Chapter 16). MySQL has a `JSON` type stored in a binary form.

```sql
-- PostgreSQL
CREATE TABLE product_attributes (product_id bigint PRIMARY KEY REFERENCES products (id), attrs jsonb NOT NULL);
INSERT INTO product_attributes VALUES (17, '{"color": "black", "layout": "ISO-DE", "switches": "brown"}');

SELECT product_id FROM product_attributes WHERE attrs ->> 'color' = 'black';      -- ->> returns text
SELECT product_id FROM product_attributes WHERE attrs @> '{"layout": "ISO-DE"}';  -- containment
-- MySQL: WHERE attrs->>'$.color' = 'black'
```

JSON columns suit data whose shape really varies (attributes that differ by product category), data received from elsewhere and stored as received (webhook payloads, Chapter 13, Section 13.10), and optional settings. They are a poor home for core relational data: values inside JSON cannot have foreign keys, are weakly typed, need `CHECK` constraints or application validation for every rule, and are harder to index and aggregate. A rule of thumb: if you filter, join or aggregate on a field routinely, it wants to be a column.

### Guarantees and non-guarantees

`numeric` guarantees exact decimal arithmetic up to its precision, but still rounds when results need more digits than the scale allows. `timestamptz` guarantees an unambiguous instant; it does not remember the zone the value was entered in (store that separately if it matters). Collations guarantee consistent comparison rules within the database, not that they match your application's comparisons. JSON columns guarantee valid JSON, nothing about its contents.

### Failure and diagnosis

- **Totals off by cents.** Evidence: `float`/`double` columns or JSON numbers parsed as floats somewhere in the path. Fix: exact types end to end.
- **Reports differ by a few hours' worth of data.** Evidence: date boundaries computed in different session time zones; columns of type `timestamp` without zone written by servers in different zones. Fix: `timestamptz`, explicit zone boundaries.
- **Duplicate accounts differing only in case**, or legitimate distinct values rejected as duplicates. Evidence: compare `lower(email)` groups; check the column's collation. Fix: the case rule the business wants, encoded in the index or collation.
- **"Incorrect string value" on insert in MySQL.** A four-byte character into a `utf8mb3` column. Fix: `utf8mb4`.

### Backend connection

Pick types for meaning: exact decimals or integer minor units with a currency for money, `timestamptz` (or UTC by convention) for instants with zones applied only at the edges, UTF-8 text with a deliberately chosen collation, and JSON only for genuinely variable data. These decisions are cheap at design time and expensive to migrate later (Chapter 18).

---

## 14.13 Classic Interview Queries

### Definition

These are the SQL problems most often set in interviews and online assessments. Each combines the tools of this chapter, and each has a naive solution that fails on ties, `NULL`s or empty results. Every example below shows the data and the result.

```text
 staff
 id │ name  │ dept  │ salary │ manager_id
 1  │ Asha  │ Eng   │ 9000   │ NULL
 2  │ Bilal │ Eng   │ 7000   │ 1
 3  │ Chloe │ Eng   │ 9000   │ 2
 4  │ Dev   │ Sales │ 6000   │ 5
 5  │ Ema   │ Sales │ 6500   │ NULL
 6  │ Finn  │ Sales │ 5000   │ 5
 7  │ Gita  │ Ops   │ 4000   │ 1
```

### Problem 1: the nth highest salary

Distinct salaries in descending order are 9000, 7000, 6500, 6000, 5000, 4000, so the second highest is **7000**. Three solutions:

```sql
-- (a) DENSE_RANK: handles ties, gives every employee at that salary
SELECT name, salary
FROM (SELECT name, salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk FROM staff) t
WHERE rnk = 2;                                                   -- Bilal │ 7000

-- (b) DISTINCT + OFFSET, wrapped so that "no such salary" returns NULL instead of no rows
SELECT (SELECT DISTINCT salary FROM staff ORDER BY salary DESC LIMIT 1 OFFSET 1) AS second_highest;  -- 7000

-- (c) correlated count: the salary with exactly n-1 distinct salaries above it
SELECT DISTINCT salary FROM staff s1
WHERE 1 = (SELECT COUNT(DISTINCT s2.salary) FROM staff s2 WHERE s2.salary > s1.salary);           -- 7000
```

The common wrong answer uses `ROW_NUMBER() … = 2` or `ORDER BY salary DESC LIMIT 1 OFFSET 1` without `DISTINCT`: both return **9000**, because Asha and Chloe tie for first. Per department, add `PARTITION BY dept` to (a): Eng → Bilal 7000; Sales → Dev 6000; Ops has no second salary and returns nothing.

### Problem 2: employees who earn more than their manager

```sql
SELECT e.name, e.salary, m.name AS manager, m.salary AS manager_salary
FROM staff e
JOIN staff m ON m.id = e.manager_id
WHERE e.salary > m.salary;
```

```text
 name  │ salary │ manager │ manager_salary
 Chloe │ 9000   │ Bilal   │ 7000
```

A self join (Section 14.7). The inner join correctly ignores Asha and Ema, who have no manager.

### Problem 3: count per group, including zero

"Number of orders per customer, including customers with none", using the tables of Section 14.7:

```sql
SELECT c.name, COUNT(o.id) AS orders          -- COUNT(o.id), not COUNT(*)
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.name
ORDER BY c.name;
```

```text
 name │ orders
 Ana  │ 2
 Ben  │ 1
 Chen │ 0          ← COUNT(*) would say 1: it counts Chen's NULL-padded row
```

### Problem 4: find and remove duplicates

```text
 users
 id │ email
 1  │ ana@x.com
 2  │ Ben@x.com
 3  │ ANA@x.com
 4  │ chen@x.com
 5  │ ana@x.com
```

```sql
-- find them
SELECT lower(email) AS email, COUNT(*) AS copies, MIN(id) AS keep_id
FROM users
GROUP BY lower(email)
HAVING COUNT(*) > 1;                                     -- ana@x.com │ 3 │ 1

-- delete all but the lowest id in each group (portable form with a window function)
DELETE FROM users
WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY lower(email) ORDER BY id) AS rn FROM users
    ) d
    WHERE rn > 1
);                                                       -- deletes ids 3 and 5

-- then make it impossible to recur
CREATE UNIQUE INDEX users_email_lower_key ON users (lower(email));   -- PostgreSQL expression index
```

In practice, before deleting, decide what to do with the duplicates' related rows (orders pointing to user 3) and re-point them to the kept row in the same transaction. (MySQL does not allow a `DELETE` to select from the same table in a subquery this way; wrap it in a derived table, or use a multi-table `DELETE … JOIN`.)

### Problem 5: gaps and islands

**Gaps**: invoice numbers 1, 2, 3, 5, 6, 9 should be consecutive. Which are missing?

```sql
SELECT num + 1 AS gap_start, next_num - 1 AS gap_end
FROM (SELECT num, LEAD(num) OVER (ORDER BY num) AS next_num FROM invoices) t
WHERE next_num - num > 1;
```

```text
 num │ next_num          gap_start │ gap_end
 1   │ 2                 4         │ 4
 2   │ 3                 7         │ 8
 3   │ 5   → gap
 5   │ 6
 6   │ 9   → gap
 9   │ NULL
```

For gaps in time (days with no orders), generate the full range and anti-join (PostgreSQL's `generate_series`, or the recursive CTE of Section 14.8):

```sql
SELECT d::date AS day
FROM generate_series(date '2026-10-01', date '2026-10-07', interval '1 day') AS d
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.created_at::date = d::date);
```

**Islands**: a user logged in on 1, 2, 3, 5, 6 and 9 October. What are the consecutive streaks? The trick: subtract a row number from each date; dates in the same streak give the same result.

```sql
SELECT MIN(day) AS streak_start, MAX(day) AS streak_end, COUNT(*) AS days
FROM (SELECT day, day - CAST(ROW_NUMBER() OVER (ORDER BY day) AS integer) AS grp FROM logins) t
GROUP BY grp
ORDER BY streak_start;
```

```text
 day     │ row_number │ day − row_number         streak_start │ streak_end │ days
 Oct 1   │ 1          │ Sep 30                   2026-10-01   │ 2026-10-03 │ 3
 Oct 2   │ 2          │ Sep 30                   2026-10-05   │ 2026-10-06 │ 2
 Oct 3   │ 3          │ Sep 30                   2026-10-09   │ 2026-10-09 │ 1
 Oct 5   │ 4          │ Oct 1
 Oct 6   │ 5          │ Oct 1
 Oct 9   │ 6          │ Oct 3
```

The longest streak is `MAX(days)` over these groups: 3.

### Problem 6: top N per group

The two highest salaries in each department, with ties included:

```sql
SELECT dept, name, salary
FROM (SELECT dept, name, salary,
             DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS rnk
      FROM staff) t
WHERE rnk <= 2
ORDER BY dept, salary DESC, name;
```

```text
 dept  │ name  │ salary
 Eng   │ Asha  │ 9000
 Eng   │ Chloe │ 9000
 Eng   │ Bilal │ 7000
 Ops   │ Gita  │ 4000
 Sales │ Ema   │ 6500
 Sales │ Dev   │ 6000
```

`DENSE_RANK ≤ 2` means "the top two salary levels" (three people in Eng). `ROW_NUMBER ≤ 2` would mean "exactly two people", choosing between Asha and Chloe by the tie-breaker. Section 14.9 discusses the choice.

### Problem 7: running totals and balances

```text
 transactions
 id │ account │ amount
 1  │ A       │ 100
 2  │ A       │ -30
 3  │ A       │ 50
 4  │ A       │ -20
```

```sql
SELECT id, account, amount,
       SUM(amount) OVER (PARTITION BY account ORDER BY id
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS balance
FROM transactions
ORDER BY account, id;
```

```text
 id │ account │ amount │ balance
 1  │ A       │ 100    │ 100
 2  │ A       │ -30    │ 70
 3  │ A       │ 50     │ 120
 4  │ A       │ -20    │ 100
```

### Problem 8: the department with the highest average salary

```sql
SELECT dept, ROUND(AVG(salary), 2) AS avg_salary
FROM staff
GROUP BY dept
ORDER BY avg_salary DESC
LIMIT 1;                                         -- Eng │ 8333.33
```

(Eng 25,000 / 3 = 8333.33; Sales 17,500 / 3 = 5833.33; Ops 4000.) If two departments could tie for highest, `RANK() OVER (ORDER BY AVG(salary) DESC) = 1` in a subquery returns all of them.

### Guarantees and non-guarantees

These solutions are correct on ties, `NULL`s and empty groups as shown. They do not guarantee performance on large tables; Chapter 16 explains which indexes make each one fast (for example, an index on `(dept, salary DESC)` for top-N per department).

### Failure and diagnosis

The wrong answers share causes covered in this chapter: ties handled with `ROW_NUMBER` or `LIMIT` without `DISTINCT`; `COUNT(*)` counting padded rows; `NOT IN` with `NULL`s; default window frames; and "no result" returned as no rows when the question expects `NULL`. Test every query on a sample that contains ties, `NULL`s and an empty group.

### Backend connection

Interview problems are compressed versions of real reports: leaderboards (top N per group), finance (running balances, gaps in invoice numbers), data quality (duplicates), engagement (streaks). Solving them in one set-based query, rather than in application loops, is both the interview answer and the production answer.

---

## 14.14 Placement-Round Topics: Extended ER, Dependency Theory, Decomposition, Relational Algebra and Calculus, Integrity Rules, Triggers and Stored Procedures

> **Placement-round topics.** This section is for campus placement rounds, university DBMS exams and written tests, where these topics are asked often. Product-company backend interviews rarely go this deep into theory. **Give your main focus to Sections 14.1–14.13**; read this section before placement rounds and come back to it for revision. Where a topic connects to real backend work, its "Backend connection" says so.

### Extended ER: specialisation, generalisation and aggregation

The **extended ER** (EER) model adds three features to Section 14.4's ER model.

- **Specialisation** divides an entity type into **subclasses** that have extra attributes or relationships: `PAYMENT` specialises into `CARD_PAYMENT` (card network, last four digits) and `BANK_TRANSFER` (IBAN). Each subclass entity **is a** superclass entity and inherits its attributes (the "ISA" relationship, drawn as a triangle).
- **Generalisation** is the same structure reached bottom-up: noticing that `CARD_PAYMENT` and `BANK_TRANSFER` share an amount, a currency and a status, and factoring them into `PAYMENT`.
- Two constraints describe a specialisation. **Disjoint** (one subclass at most) versus **overlapping** (an entity can be in several, like a person who is both `EMPLOYEE` and `CUSTOMER`); **total** (every superclass entity belongs to some subclass) versus **partial**.
- **Aggregation** treats a relationship as an entity so that it can take part in another relationship: the relationship "employee *works on* project" becomes a unit that a `MANAGER` *monitors*.

Mapping a specialisation to tables has three standard options, the same choices ORMs offer for inheritance (Chapter 18, Section 18.8):

| Option | Tables | Trade-off |
|---|---|---|
| One table for the hierarchy | `payments(id, amount, kind, card_last4, iban)`, with nullable subclass columns and a type column | No joins; many `NULL`s; subclass `NOT NULL` rules need `CHECK` constraints per kind |
| Table per subclass plus superclass | `payments(id, amount, …)`, `card_payments(payment_id PK FK, card_last4)`, `bank_transfers(payment_id PK FK, iban)` | Normalised and constraint-friendly; a join per read |
| Table per concrete subclass | `card_payments(id, amount, …, card_last4)`, `bank_transfers(id, amount, …, iban)` | No joins for one kind; "all payments" needs a `UNION`, and a unique id across tables needs a shared sequence |

### Armstrong's axioms and attribute closure

**Armstrong's axioms** are the inference rules for functional dependencies. From a set of FDs `F`, they derive every FD that `F` implies (the **closure** `F+`):

- **Reflexivity:** if `Y ⊆ X`, then `X → Y` (trivial dependencies).
- **Augmentation:** if `X → Y`, then `XZ → YZ`.
- **Transitivity:** if `X → Y` and `Y → Z`, then `X → Z`.

Derived rules: **union** (`X → Y` and `X → Z` give `X → YZ`), **decomposition** (`X → YZ` gives `X → Y` and `X → Z`) and **pseudotransitivity** (`X → Y` and `WY → Z` give `WX → Z`).

The **attribute closure** `X+` is the set of all attributes that `X` determines. The algorithm:

```text
 result = X
 repeat
     for each FD (A → B) in F:
         if A ⊆ result: result = result ∪ B
 until result does not change
```

`X` is a superkey exactly when `X+` contains all the attributes.

**Worked example.** `R(A, B, C, D, E)` with `F = { A → B, B → C, CD → E, E → A }`.

```text
 {A}+  = A → B → C                         = {A, B, C}              not a key
 {D}+  = {D}                                                         not a key
 {AD}+ = A, D → B → C → (CD → E) → (E → A) = {A, B, C, D, E}        a key
```

### Finding all candidate keys

Two observations shrink the search:

- An attribute that appears on **no right-hand side** must be in every key (nothing else can determine it). Here, that is `D`.
- An attribute that appears **only** on right-hand sides is in no key. Here, there is none.

So every key contains `D`. `D` alone is not a key. Try `D` with each other attribute:

```text
 {AD}+ = all → key       {BD}+ = B → C, CD → E, E → A          = all → key
 {CD}+ = CD → E, E → A, A → B = all → key       {DE}+ = E → A, A → B, B → C = all → key
```

Each pair is minimal (`D` alone is not a key, and no pair's other attribute is a key alone, since D must appear), so the candidate keys are **AD, BD, CD and DE**. Every attribute is prime, so `R` is in 3NF; it is **not** in BCNF, because `A → B` has a left side `A` that is not a superkey.

### Canonical (minimal) cover

A **canonical cover** `Fc` is a simplified set of FDs equivalent to `F`: every right side is a single attribute (or merged per left side), no left side has an **extraneous** attribute, and no FD is redundant. It is the input to 3NF synthesis.

**Worked example.** `F = { A → BC, B → C, A → B, AB → C }`.

```text
 1. Split right sides:              A → B, A → C, B → C, A → B, AB → C      (drop the duplicate A → B)
 2. Remove extraneous LHS attributes: in AB → C, is B extraneous? Compute {A}+ under F = {A, B, C} ∋ C,
                                    so A → C already holds; AB → C reduces to A → C (a duplicate)
 3. Remove redundant FDs:           A → C follows from A → B and B → C, so drop it
 Fc = { A → B, B → C }
```

### Lossless-join and dependency-preserving decomposition

A decomposition of `R` into `R1` and `R2` is **lossless** if joining `R1` and `R2` always gives back exactly `R`: no rows lost and, more importantly, no **spurious** rows added. The test for a binary decomposition: it is lossless if and only if the common attributes `R1 ∩ R2` form a superkey of `R1` or of `R2`.

Using `account_managers(cust, category, manager)` from Section 14.5 (with `manager → category`):

```text
 lossless:  R1 = (manager, category), R2 = (cust, manager)     R1 ∩ R2 = {manager}, a key of R1 ✔

 lossy:     R1 = (cust, category),    R2 = (category, manager) R1 ∩ R2 = {category}, a key of neither ✘
   R1: (C1, Peripherals) (C1, Computers) (C2, Computers) (C3, Peripherals)
   R2: (Peripherals, Ravi) (Computers, Sofia) (Computers, Tom)
   R1 ⋈ R2:  C1 Peripherals Ravi   C1 Computers Sofia   C1 Computers Tom   ← spurious
             C2 Computers Sofia ← spurious   C2 Computers Tom   C3 Peripherals Ravi
   6 rows instead of the original 4: the join invents facts that were never true
```

A decomposition is **dependency-preserving** if every FD of `F` can be checked within a single piece (formally, the union of the FDs that hold within each piece implies all of `F`). Section 14.5's BCNF decomposition was lossless but lost `{cust, category} → manager`. The standard results:

| Target | Lossless? | Dependency-preserving? |
|---|---|---|
| 3NF (by synthesis from a canonical cover) | always achievable | always achievable |
| BCNF | always achievable | not always |

### Multivalued dependencies and 4NF

A **multivalued dependency** (MVD) `X ↠ Y` holds when, for each value of `X`, the set of `Y` values is independent of the remaining attributes. It appears when one table stores two independent many-valued facts about the same thing.

```text
 course_staff(course, teacher, textbook)        course ↠ teacher,  course ↠ textbook
 DBMS │ Rao  │ Korth
 DBMS │ Rao  │ Navathe
 DBMS │ Iyer │ Korth
 DBMS │ Iyer │ Navathe        every teacher must be paired with every textbook;
                              adding a third textbook means adding one row per teacher
```

The table is in BCNF (its only key is all three columns, and there are no non-trivial FDs), but it is redundant. **4NF** requires that for every non-trivial MVD `X ↠ Y`, `X` is a superkey. Decompose into `course_teachers(course, teacher)` and `course_textbooks(course, textbook)`. (**5NF**, or project-join normal form, deals with redundancy that only a decomposition into three or more tables removes; it is rarely asked beyond its definition.)

### Relational algebra

**Relational algebra** is the formal query language of the relational model: each operator takes relations and returns a relation, so operators compose.

| Operator | Symbol | Meaning | SQL equivalent |
|---|---|---|---|
| Selection | σ<sub>condition</sub>(R) | rows satisfying the condition | `WHERE` |
| Projection | π<sub>columns</sub>(R) | only some columns (duplicates removed, as a set) | `SELECT DISTINCT cols` |
| Union | R ∪ S | rows in either | `UNION` |
| Difference | R − S | rows in R not in S | `EXCEPT` |
| Intersection | R ∩ S | rows in both | `INTERSECT` |
| Cartesian product | R × S | every pairing | `CROSS JOIN` |
| Rename | ρ<sub>name</sub>(R) | rename a relation or columns | aliases |
| Join | R ⋈<sub>condition</sub> S | σ<sub>condition</sub>(R × S); natural join matches equal-named columns | `JOIN … ON` |
| Division | R ÷ S | values of R's other columns paired with **every** row of S | double `NOT EXISTS` |

"Names of customers who ordered product 17": π<sub>name</sub>(customers ⋈<sub>customers.id = orders.customer_id</sub> σ<sub>product_id = 17</sub>(order_lines ⋈ orders)).

**Division** answers "for all" questions, and is the operator exams test most:

```text
 enrolled(student, course)        required(course)
 S1 C1   S1 C2                    C1
 S2 C1                            C2
 S3 C1   S3 C2   S3 C3
 enrolled ÷ required = { S1, S3 }     students enrolled in EVERY required course (S2 lacks C2)
```

```sql
-- "students for whom there is no required course they are not enrolled in"
SELECT DISTINCT e.student FROM enrolled e
WHERE NOT EXISTS (
    SELECT 1 FROM required r
    WHERE NOT EXISTS (SELECT 1 FROM enrolled e2 WHERE e2.student = e.student AND e2.course = r.course)
);
-- or by counting
SELECT student FROM enrolled WHERE course IN (SELECT course FROM required)
GROUP BY student HAVING COUNT(DISTINCT course) = (SELECT COUNT(*) FROM required);
```

Exams also use extended operators: the **outer joins** (⟕ left, ⟖ right, ⟗ full), and **grouping/aggregation**, written <sub>dept</sub>𝒢<sub>AVG(salary)</sub>(staff).

### Relational calculus

**Relational calculus** is the other formal query language of the relational model. Where relational algebra says **how** to build the result (a sequence of operators), calculus says only **what** the result must satisfy, as a formula of logic. It is the theoretical root of SQL's declarative style.

- **Tuple relational calculus** (TRC) has variables that range over tuples: `{ t.name | t ∈ customers ∧ t.city = 'Berlin' }` means "the names of customer tuples whose city is Berlin".
- **Domain relational calculus** (DRC) has variables that range over attribute values: `{ ⟨n⟩ | ∃ i, c ( ⟨i, n, c⟩ ∈ customers ∧ c = 'Berlin' ) }`.
- Both use the quantifiers ∃ ("there exists") and ∀ ("for all"). Division becomes direct: students `s` such that ∀ required course `r`, ∃ an enrolment of `s` in `r`.
- A **safe** expression is one whose result is finite and drawn from values in the database; `{ t | ¬(t ∈ customers) }` ("everything that is not a customer") is unsafe.

**Codd's theorem** states that relational algebra and safe relational calculus have the same expressive power. A query language that can express every query of relational algebra is called **relationally complete**; SQL is, and adds more (aggregation, ordering, recursion).

### Integrity constraint categories and Codd's rules

Textbooks group the constraints of Sections 14.2–14.3 into named categories:

| Category | Rule | SQL mechanism |
|---|---|---|
| **Domain constraint** | every value comes from the attribute's domain | data types, `CHECK`, `NOT NULL` |
| **Key constraint** | no two tuples share a candidate key value | `PRIMARY KEY`, `UNIQUE` |
| **Entity integrity** | no primary key attribute is `NULL` | `PRIMARY KEY` implies `NOT NULL` |
| **Referential integrity** | every foreign key value matches an existing referenced key, or is `NULL` | `FOREIGN KEY` with actions |
| General (semantic) constraints, **assertions** | any condition over the database | `CHECK` for single rows; the standard's `CREATE ASSERTION` for multi-table conditions is not implemented by PostgreSQL or MySQL; use triggers or transactions |

**Codd's rules** are thirteen criteria (numbered 0 to 12) that Codd published in 1985 to say what a fully relational DBMS must do. No commercial product satisfies all of them; exams ask for the list:

```text
  0  Foundation: the system manages data entirely through its relational capabilities
  1  Information: all data is represented as values in tables
  2  Guaranteed access: every value is reachable by table name + primary key + column name
  3  Systematic treatment of NULL: a missing-value marker, distinct from zero or empty string
  4  Active online catalogue: the schema itself is stored in tables queryable like data
  5  Comprehensive data sublanguage: one language for definition, manipulation, integrity,
     authorisation and transactions (SQL)
  6  View updating: every theoretically updatable view is updatable by the system
  7  High-level insert, update and delete: set-at-a-time, not only row-at-a-time
  8  Physical data independence
  9  Logical data independence
 10  Integrity independence: integrity rules are defined in the catalogue, not in programs
 11  Distribution independence: programs work unchanged if data is distributed
 12  Non-subversion: a low-level interface cannot bypass the integrity rules
```

### Views: which are updatable

A view can be updated through (its `INSERT`, `UPDATE` and `DELETE` translated to the base table) only when each view row corresponds to exactly one base row. In PostgreSQL and MySQL that means, roughly: one base table in `FROM`; no aggregates, `GROUP BY`, `HAVING`, `DISTINCT`, set operations, window functions or `LIMIT`; and only plain columns for the columns being written. (MySQL can also update some join views, one base table per statement.) Other views can be made writable with `INSTEAD OF` triggers (below), and `WITH CHECK OPTION` prevents writes that would fall outside the view (Section 14.11).

### Triggers

A **trigger** is a function the database runs automatically when a specified event happens on a table: `BEFORE` or `AFTER` an `INSERT`, `UPDATE` or `DELETE`, once per affected row (**row-level**) or once per statement (**statement-level**). An `INSTEAD OF` trigger on a view replaces the operation.

```sql
-- PostgreSQL: keep products.review_count in step with reviews (a maintained denormalisation, Section 14.5)
CREATE FUNCTION bump_review_count() RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE products SET review_count = review_count + 1 WHERE id = NEW.product_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE products SET review_count = review_count - 1 WHERE id = OLD.product_id;
    END IF;
    RETURN NULL;                                  -- the return value is ignored for AFTER triggers
END $$ LANGUAGE plpgsql;

CREATE TRIGGER reviews_count AFTER INSERT OR DELETE ON reviews
FOR EACH ROW EXECUTE FUNCTION bump_review_count();
```

`NEW` and `OLD` are the row after and before the change. Typical uses: audit tables recording every change, maintaining counters or `updated_at` columns, and rules that constraints cannot express.

### Stored procedures and functions

A **stored procedure** (or stored function) is code written in a procedural language inside the database (PL/pgSQL in PostgreSQL, SQL/PSM in MySQL) and invoked by name (`CALL transfer(1, 2, 500)` or `SELECT f(...)`). Functions return values and can be used in queries; procedures (in PostgreSQL from version 11) can also control transactions.

| Advantages | Disadvantages |
|---|---|
| fewer network round trips for multi-step logic | logic split between application and database, harder to find and review |
| runs close to the data | versioning, testing and deployment need separate discipline (migrations, Chapter 18) |
| can act as a permission boundary (grant `EXECUTE`, not table access) | database CPU is the hardest resource to scale (Chapter 23) |
| | ties the system to one database's language |

### Backend connection

Dependency theory is how you justify a schema in a review; relational algebra is what the optimiser actually manipulates (Chapter 16 shows plans as trees of these operators). Triggers and stored procedures are real tools with a real cost: logic that runs invisibly on every write surprises developers, slows bulk operations and is easy to forget during migrations. Use triggers for audit trails and small integrity rules, keep business workflows in the application, and document every trigger next to its table.

### Drill for this section

**C1. Given R(A, B, C, D, E) and F = {A → B, B → C, CD → E, E → A}, find all candidate keys. Is R in 3NF? In BCNF?**
A strong answer must mention:
- D appears on no right side, so it is in every key; D alone is not a key.
- Closures: {AD}+, {BD}+, {CD}+, {DE}+ each contain all attributes; candidate keys AD, BD, CD, DE.
- Every attribute is prime, so R is in 3NF.
- A → B violates BCNF because A is not a superkey.
*Common weak answer:* stopping at the first key found (AD). Questions about "all candidate keys" expect a systematic search.

**C2. When is a decomposition lossless? Show a lossy one.**
A strong answer must mention:
- Lossless if the join of the pieces equals the original; binary test: the common attributes are a superkey of one piece.
- A lossy decomposition creates spurious tuples on join (the account-managers example joined on category).
- Lossless matters more than dependency preservation: losing it corrupts data; losing dependency preservation only makes a rule harder to enforce.
*Common weak answer:* "lossless means no rows are lost." The danger is extra, invented rows.

**C3. Why might a designer stop at 3NF instead of BCNF?**
A strong answer must mention:
- 3NF can always be reached losslessly while preserving dependencies; BCNF sometimes cannot preserve them.
- The lost dependency then cannot be enforced by a key on a single table (the customer/category/manager example).
- The remaining 3NF redundancy can be made safe with constraints (a composite foreign key).
*Common weak answer:* "BCNF is too strict." Says nothing about dependency preservation, which is the actual reason.

**C4. Express "students enrolled in all required courses" in relational algebra and SQL.**
A strong answer must mention:
- Division: enrolled ÷ required.
- SQL with double `NOT EXISTS` (no required course that the student lacks), or `GROUP BY … HAVING COUNT(DISTINCT course) =` the number of required courses.
- The counting version must restrict to required courses, or extra courses inflate the count.
*Common weak answer:* `WHERE course IN (required courses)`, which returns students enrolled in any of them, not all.

**C5. What are specialisation, generalisation and aggregation in the EER model, and how do you map a specialisation to tables?**
A strong answer must mention:
- Specialisation splits an entity type into subclasses (ISA, inheritance of attributes); generalisation factors shared attributes into a superclass; aggregation treats a relationship as an entity.
- Disjoint versus overlapping, total versus partial.
- Three mappings: one table with a type column, a table per subclass joined to a superclass table, or a table per concrete subclass, with their trade-offs (nulls and checks, joins, unions).
*Common weak answer:* describing ISA without any mapping to tables, which is what the question is testing.

**C6. How does relational calculus differ from relational algebra? Name the integrity constraint categories.**
A strong answer must mention:
- Algebra is procedural (a sequence of operators); calculus is declarative (a logical formula the result satisfies), in tuple or domain form.
- Codd's theorem: algebra and safe calculus are equally expressive; relational completeness.
- Domain, key, entity integrity (no null primary key), referential integrity, and general constraints or assertions.
*Common weak answer:* "calculus is the mathematical version of SQL." It misses procedural versus declarative and the equivalence result.

---

## Chapter Summary

The relational model stores everything as **relations** (tables) of **tuples** (rows) over **attributes** (columns) with **domains** (types), and queries them **declaratively**: you say what you want, and the database chooses how. That separation, physical and logical **data independence**, is why relational databases replaced navigational ones. Results are sets with **no order** unless you ask for one.

**Keys** identify rows: superkeys are unique, candidate keys are minimal superkeys, one is chosen as the **primary key**, the rest are alternate keys enforced with `UNIQUE`, and **foreign keys** express relationships and forbid dangling references. Surrogate primary keys plus unique natural keys are the usual design. **Constraints** (`NOT NULL`, `UNIQUE`, `CHECK`, foreign keys with deliberate `ON DELETE` actions) are checked **immediately** by default, per statement (PostgreSQL and MySQL check uniqueness row by row), and only `DEFERRABLE INITIALLY DEFERRED` constraints at commit; they are the only rules enforced for every writer, under any concurrency.

**ER modelling** turns requirements into entities, attributes and relationships with cardinalities; 1:N becomes a foreign key on the many side, 1:1 a unique foreign key, and M:N a **junction table**. **Normalisation** removes redundancy and its **update, insert and delete anomalies** by following **functional dependencies**: 1NF (atomic values), 2NF (no partial dependencies), 3NF (no transitive dependencies), BCNF (every determinant is a key), with BCNF sometimes costing **dependency preservation**. **Denormalisation** trades that safety for read speed and needs a write path and a reconciliation query to stay correct; historical facts (the price charged) are not denormalisation.

A `SELECT` is defined in **logical order**: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`, which explains why `WHERE` filters rows and `HAVING` groups, and why aliases and window functions are unavailable in `WHERE`. **Joins** reassemble normalised facts: inner (matches), left/right/full outer (plus unmatched rows padded with `NULL`), cross, self, **semi** (`EXISTS`) and **anti** (`NOT EXISTS`); conditions on the optional side belong in `ON`, and joining two one-to-many children multiplies rows. **Aggregation**, **subqueries** (correlated or not), **CTEs** and **recursive CTEs** answer summary, comparative and hierarchical questions. **Window functions** compute per-row values over partitions and frames: `ROW_NUMBER`, `RANK` and `DENSE_RANK` differ only on ties, `LAG`/`LEAD` read neighbours, and running totals need an explicit `ROWS` frame.

**`NULL`** brings **three-valued logic**: comparisons with `NULL` are UNKNOWN, `WHERE` keeps only TRUE, `NOT IN` against a list containing `NULL` returns nothing, aggregates ignore `NULL`s, and `UNIQUE` allows many of them. **Set operations**, atomic **upserts** (`ON CONFLICT`, `ON DUPLICATE KEY UPDATE`, `MERGE`), **views** and **materialised views** round out the language. **Types** carry meaning: exact decimals or integer minor units for money, `timestamptz` for instants with zones applied explicitly at the edges, UTF-8 text with a chosen **collation**, and JSON only for genuinely variable data. The classic interview queries combine all of these, and each has a naive version that breaks on ties, `NULL`s or empty groups.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Declarative queries | The result is defined independently of storage; plans adapt as data grows | Performance; row order without `ORDER BY` |
| Primary key | Unique, non-null identity per row | That the identifier means anything to the business |
| `UNIQUE` constraint | No duplicates of the key, under any concurrency | Uniqueness of values differing only by case (depends on collation); only one `NULL` |
| Foreign key | No reference to a missing row | That deletes cascade only where intended; an index on the referencing column (PostgreSQL) |
| `CHECK` constraint | Rows where the condition is false are rejected | Rejection when the condition is unknown (`NULL`); enforcement in MySQL before 8.0.16 |
| Deferred constraint | Checked at commit, allowing temporary violations inside a transaction | Anything in MySQL (unsupported) |
| 3NF / BCNF schema | Each fact stored once; no update, insert or delete anomalies | That the FDs were right; dependency preservation (BCNF); read performance |
| Denormalised copy | Faster reads | Consistency, unless the write path or reconciliation keeps it |
| Logical evaluation order | What the result is | How it is executed |
| Outer join | Every preserved-side row appears at least once | One row per preserved row (one-to-many multiplies) |
| `EXISTS` / `NOT EXISTS` | Each qualifying row once; correct with `NULL`s | Speed without suitable indexes |
| Window function | A value per row without collapsing rows | Deterministic ties without a tie-breaker; the frame you assumed if left default |
| Three-valued logic | Defined results with `NULL` | Intuitive results (`NOT IN`, `<>`, `AVG`) |
| `ON CONFLICT` upsert | Atomic insert-or-update against the named unique constraint | Idempotency if the update adds rather than sets |
| View | Always-current results with restrictions | Performance; updatability beyond simple views |
| Materialised view | Fast reads | Freshness beyond the last refresh |
| `numeric` / `timestamptz` | Exact decimals; unambiguous instants | No rounding on division; the original zone of an entered value |

---

## Interview Drill

**Q1. What is persistence? What is a database? What is a DBMS, and why use one instead of files? What is the relational model, and why did it win?**
A strong answer must mention:
- Persistence: data outlives the process that wrote it. A database is an organised, persistent collection of related shared data; a DBMS is the software that manages it (schema, query language, constraints, transactions, durability, concurrency control, security, indexes).
- The file-based approach suffers redundancy and inconsistency, a program per question, integrity rules buried in code, no atomicity, lost updates under concurrent access, coarse security and fragile durability; a DBMS solves each once, for every program.
- Data as relations (tables of rows over typed columns), relationships expressed by values (keys), not pointers.
- Declarative queries: the database chooses the execution plan; the same query works as data and storage change.
- The three-schema architecture (external, conceptual, internal) gives physical and (partly) logical data independence, unlike navigational databases, where programs followed pointers and broke when storage changed.
- Results are sets: no order without `ORDER BY`, duplicates only if allowed.
*Common weak answer:* "A database is where data is stored" and "because SQL is easy." A file also stores data; what makes a DBMS is the guarantees it enforces for every program, and the relational model's decisive advantages were declarativeness and data independence.

**Q2. What are super, candidate, primary, alternate and foreign keys? Surrogate or natural?**
A strong answer must mention:
- Superkey: unique set of columns; candidate key: minimal superkey; primary key: the chosen candidate key (unique, not null); alternate keys: the other candidates, enforced with `UNIQUE`.
- Foreign key: references a candidate key of another (or the same) table; enforces referential integrity.
- Surrogate keys are stable and compact; natural keys can change and carry meaning.
- Common practice: surrogate primary key plus `UNIQUE` on the natural key.
*Common weak answer:* "A primary key is unique and a foreign key links tables." Misses minimality and the difference between candidate and primary keys.

**Q3. What constraints exist, and when are they checked?**
A strong answer must mention:
- `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY` (with `CASCADE`, `SET NULL`, `RESTRICT`, `NO ACTION`), `CHECK`.
- Checked immediately, per statement, by default; only `DEFERRABLE INITIALLY DEFERRED` constraints at commit (for mutually referencing rows).
- PostgreSQL checks non-deferrable uniqueness row by row; MySQL has no deferred constraints; MySQL ignored `CHECK` before 8.0.16.
- `CHECK` accepts `NULL` (unknown is not false).
*Common weak answer:* "Constraints are checked when you commit." Only deferred constraints are.

**Q4. How do you model a many-to-many relationship?**
A strong answer must mention:
- A junction table with foreign keys to both sides and a composite primary key (or unique pair).
- Attributes of the relationship (quantity, price charged, role) live in the junction table.
- Never a comma-separated list column (breaks 1NF, no foreign keys, no indexing).
- 1:N is a foreign key on the many side; 1:1 a unique foreign key.
*Common weak answer:* "Store an array of IDs in a column." No referential integrity, no simple joins, update anomalies.

**Q5. What is normalisation? Explain 1NF, 2NF, 3NF and BCNF with an example.**
A strong answer must mention:
- Goal: each fact stored once, removing update, insert and delete anomalies; driven by functional dependencies.
- 1NF atomic values and no repeating groups; 2NF no partial dependency on part of a composite key; 3NF no transitive dependency of non-key columns; BCNF every determinant is a superkey.
- A concrete walk-through: an order sheet split into customers, orders, order lines, products; the manager/category case that is 3NF but not BCNF.
- BCNF can lose dependency preservation; 3NF can always keep it.
*Common weak answer:* reciting the definitions without an anomaly or an example. The point of each step is the anomaly it removes.

**Q6. When would you denormalise, and what does it cost?**
A strong answer must mention:
- When a measured read path is too slow and reads vastly outnumber writes (stored totals, counters, summary tables, read models).
- Cost: copies can diverge; every write path must maintain them, ideally in the same transaction or via triggers or refreshes.
- A reconciliation query to detect drift; a decision about which copy is the truth.
- Historical facts (price charged, shipping address on an order) are correct design, not denormalisation.
*Common weak answer:* "Denormalise for performance." Without the consistency cost and maintenance mechanism, that is just a future bug.

**Q7. In what order is a `SELECT` evaluated? Why can't `WHERE` use a column alias?**
A strong answer must mention:
- `FROM`/joins → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`.
- Aliases are created in `SELECT`, after `WHERE`, so `WHERE` cannot see them; `ORDER BY` can.
- Window functions are computed in `SELECT`, so filtering on them needs a subquery or CTE.
- This is the logical order; the physical plan may differ (Chapter 16).
*Common weak answer:* "Top to bottom as written." That cannot explain alias errors or aggregate placement.

**Q8. What is the difference between `WHERE` and `HAVING`?**
A strong answer must mention:
- `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation.
- Aggregates are not allowed in `WHERE`, because they do not exist yet at that step.
- Row conditions belong in `WHERE` even in grouped queries (less work, correct semantics).
- Example: `WHERE status = 'PAID' … GROUP BY customer_id HAVING SUM(total) > 5000`.
*Common weak answer:* "`HAVING` is `WHERE` for `GROUP BY`." Close, but misses why aggregates fail in `WHERE`.

**Q9. What types of joins are there? Left join versus anti-join?**
A strong answer must mention:
- Inner, left, right, full outer, cross, self, semi (`EXISTS`), anti (`NOT EXISTS`), with what each keeps.
- A left join keeps all left rows, padded with `NULL`s; an anti-join keeps only left rows with no match (left join plus `WHERE right.key IS NULL`, or `NOT EXISTS`).
- A semi-join returns each left row once, unlike an inner join, which duplicates per match.
- Pitfalls: right-side filters in `WHERE` turn a left join into an inner join; `NOT IN` with `NULL`s; fan-out from two one-to-many joins.
*Common weak answer:* drawing Venn diagrams without mentioning `NULL` padding, duplicate multiplication or the `NOT IN` trap.

**Q10. Correlated versus uncorrelated subqueries? What are CTEs and recursive CTEs?**
A strong answer must mention:
- Uncorrelated: independent of the outer query, conceptually evaluated once; correlated: references the outer row, conceptually per row (optimisers may rewrite it).
- A CTE (`WITH`) names an intermediate result for readability and reuse; PostgreSQL 12+ inlines single-use CTEs.
- A recursive CTE has an anchor and a recursive part joined by `UNION ALL`, iterating until no new rows: hierarchies, series.
- Guard recursion against cycles.
*Common weak answer:* "Correlated subqueries are always slow." Meaning is per row; execution depends on the plan.

**Q11. What are window functions? `ROW_NUMBER` versus `RANK` versus `DENSE_RANK`?**
A strong answer must mention:
- A per-row value computed over a window (`PARTITION BY`, `ORDER BY`, frame) without collapsing rows, unlike `GROUP BY`.
- On ties: `ROW_NUMBER` 1,2,3,4 (arbitrary between equals without a tie-breaker); `RANK` 1,2,2,4; `DENSE_RANK` 1,2,2,3.
- `LAG`/`LEAD`, running totals with `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, the default `RANGE` frame trap with ties.
- Top-N per group with a ranking in a subquery and a filter outside.
*Common weak answer:* "They're like `GROUP BY`." The defining property is that rows are not collapsed.

**Q12. How does `NULL` behave?**
A strong answer must mention:
- Three-valued logic: comparisons with `NULL` are UNKNOWN; `WHERE` keeps only TRUE; `NULL = NULL` is not TRUE; use `IS NULL`, `IS DISTINCT FROM`.
- Truth tables: `FALSE AND UNKNOWN = FALSE`, `TRUE OR UNKNOWN = TRUE`, otherwise UNKNOWN.
- `x NOT IN (…, NULL)` is never TRUE; use `NOT EXISTS`.
- Aggregates ignore `NULL`s (`COUNT(*)` vs `COUNT(col)`, `AVG`), `SUM` of nothing is `NULL`; `GROUP BY` groups `NULL`s together; `UNIQUE` allows many; sort position differs between PostgreSQL and MySQL.
*Common weak answer:* "`NULL` means zero or empty." It means unknown, with its own logic.

**Q13. `UNION` versus `UNION ALL`; `DELETE` versus `TRUNCATE` versus `DROP`; what is an upsert?**
A strong answer must mention:
- `UNION` removes duplicates (extra sort or hash); `UNION ALL` keeps them and is cheaper.
- `DELETE` removes rows by condition, fires triggers, is transactional; `TRUNCATE` removes all rows fast (DDL; transactional in PostgreSQL, implicit commit in MySQL; resets auto-increment in MySQL); `DROP` removes the table.
- Upsert: atomic insert-or-update against a unique constraint (`ON CONFLICT`, `ON DUPLICATE KEY UPDATE`, `MERGE`), avoiding the check-then-act race.
- An additive upsert is not idempotent; a setting one is.
*Common weak answer:* "`TRUNCATE` is a fast `DELETE`." Misses triggers, transactional differences and identity reset.

**Q14. Why must money not be stored as a float? How should timestamps be stored?**
A strong answer must mention:
- Binary floating point cannot represent most decimal fractions: `0.1 + 0.2 = 0.30000000000000004`; sums drift; equality fails. Use `numeric`/`decimal` or integer minor units, plus a currency.
- Store instants as `timestamptz` (PostgreSQL) or UTC by convention (MySQL `DATETIME`; `TIMESTAMP` ends in 2038).
- DST: non-existent and repeated local times; "1 day" is not always 24 hours; "today" depends on the session zone.
- Future local events: local time plus zone name.
*Common weak answer:* "Use `double` with two decimal places." Rounding for display does not fix accumulated binary error.

**Q15. Find the second highest salary. What goes wrong with the obvious answer?**
A strong answer must mention:
- `DENSE_RANK() … = 2`, or `DISTINCT … ORDER BY DESC LIMIT 1 OFFSET 1`, or the correlated count of distinct higher salaries.
- Ties: `ROW_NUMBER` or `OFFSET` without `DISTINCT` returns the top salary again when two people share it.
- Returning `NULL` when there is no second salary (wrap in a scalar subquery).
- Per department with `PARTITION BY`.
*Common weak answer:* `ORDER BY salary DESC LIMIT 1 OFFSET 1`, which fails on ties.

**Q16. How do you find duplicates and keep only one?**
A strong answer must mention:
- `GROUP BY` the identifying expression (often normalised, `lower(email)`) `HAVING COUNT(*) > 1`.
- Delete with `ROW_NUMBER() OVER (PARTITION BY … ORDER BY id)` and remove `rn > 1` (or a self-join on the key with `id >` the kept one).
- Re-point related rows first, in one transaction.
- Add the unique constraint (or expression index) so it cannot recur.
*Common weak answer:* `SELECT DISTINCT`, which hides duplicates in a result without fixing the data.

**Q17. How do you find gaps in a sequence, and streaks of consecutive days?**
A strong answer must mention:
- Gaps: `LEAD(num) OVER (ORDER BY num)` and keep rows where the difference exceeds 1; for dates, generate the full range and anti-join.
- Islands: subtract `ROW_NUMBER()` from each value; consecutive values share the difference; group by it.
- The longest streak is the maximum group size.
*Common weak answer:* a self-join comparing every row with every other, which is quadratic and hard to read.

**Q18. Top N per group and running totals: how?**
A strong answer must mention:
- Rank within partitions (`ROW_NUMBER` for exactly N, `RANK`/`DENSE_RANK` to include ties) in a subquery, filter outside.
- Running total: `SUM(x) OVER (PARTITION BY account ORDER BY id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`.
- Tie-breakers for determinism; explicit `ROWS` frames.
*Common weak answer:* `GROUP BY dept ORDER BY salary DESC LIMIT 2`, which limits the whole result, not each group.

---

## Scenario Exercises

1. A support tool shows that a customer has three different cities, depending on which order you open. The schema has a `city` column on `orders`. Explain the anomaly, the functional dependency behind it, and the migration to a normalised design that keeps the shipping address of past orders correct.
2. A report of "customers who have never placed an order" returned zero rows the day after a guest-checkout feature launched. The query uses `NOT IN`. Explain exactly why, with the truth values involved, and give two correct rewrites.
3. A finance analyst says the dashboard's revenue for Sunday 29 March 2026 is higher than the accounting system's by the revenue of one hour, while every other day in March matches. Both read the same `orders` table, and the business runs on Berlin time. Give two hypotheses involving time and how you would confirm each.
4. A marketing query reports each customer's total spend and number of support tickets in one query. Totals for some customers are exactly two or three times their real spend. Explain the cause and rewrite the query.
5. You inherit a MySQL schema where `users.email` has a unique index. A user named "René" cannot sign up because "rene@x.com" already exists, belonging to someone else. Explain why, and what you would change.
6. A product team wants to store all product attributes, including price, stock and category, in a single JSON column "for flexibility". Argue which fields should stay columns and why, referring to constraints, joins and indexing.
7. An `UPDATE positions SET pos = pos + 1 WHERE list_id = 7` fails with a duplicate-key error in PostgreSQL, although after the update every position would again be unique. Explain why and give two fixes.

---

## Diagnosis Lab

**Lab 1: Order totals that do not add up**
- *Symptom:* A reconciliation job flags 214 orders whose `orders.total_cents` differs from the sum of their lines. Customers were charged the stored total.
- *Hypotheses:* (a) line prices changed after the order; (b) the stored total is not updated on some write path; (c) rounding differences.
- *Evidence:* The reconciliation query (Section 14.5) shows all 214 orders had a line added or removed by the support tool after creation. The API updates `orders.total_cents` in the same transaction as line changes; the support tool writes `order_lines` directly with SQL and never touches the total. Line prices on these orders are unchanged.
- *Root cause:* (b). A denormalised value maintained only on one write path.
- *Fix:* Route the support tool through the same service method, or maintain the total with a trigger on `order_lines`; recompute the 214 totals and handle customer refunds or charges.
- *Prevention:* Run the reconciliation query daily with an alert; restrict direct table writes to the service's database role (Chapter 20).

**Lab 2: An anti-join that went silent**
- *Symptom:* The nightly "inactive customers" email, sent to customers who never ordered, went to nobody for a week.
- *Hypotheses:* (a) every customer has ordered; (b) the query's logic broke; (c) the email job failed.
- *Evidence:* The job ran and logged "0 recipients". The query is `WHERE id NOT IN (SELECT customer_id FROM orders)`. A migration a week ago made `orders.customer_id` nullable for guest checkout, and guest orders now have `NULL` there. `SELECT COUNT(*) FROM orders WHERE customer_id IS NULL` returns 1,904.
- *Root cause:* (b). With a `NULL` in the subquery, `NOT IN` is never true (Section 14.10).
- *Fix:* Rewrite with `NOT EXISTS`.
- *Prevention:* A lint rule against `NOT IN (subquery)`; tests with `NULL`s in sample data; an alert when a recipient count drops to zero.

**Lab 3: Daily revenue disagrees between two systems**
- *Symptom:* All summer, the dashboard's daily revenue matched finance's to the cent. From 25 October 2026 onwards, every day differs by roughly one hour's worth of orders.
- *Hypotheses:* (a) a time-zone mismatch in day boundaries; (b) the daylight-saving change on that date; (c) late-arriving data.
- *Evidence:* Finance groups orders by Berlin business day using the zone name `Europe/Berlin`. The dashboard's query, written in July, computes `(created_at AT TIME ZONE '+02:00')::date`: a **fixed offset**, which was Berlin's offset in summer. On 25 October Berlin's clocks went back and its offset became +01:00, so from then on the dashboard's "day" starts at 23:00 Berlin time, one hour early. Late data (c) would not produce a sudden permanent shift on a DST date.
- *Root cause:* (a), exposed by (b): an offset was used where a time zone was meant. An offset is one moment's difference from UTC; a zone is the rules that change it.
- *Fix:* Use the zone name (`AT TIME ZONE 'Europe/Berlin'`), or explicit boundaries such as `created_at >= timestamptz '2026-10-25 00:00 Europe/Berlin' AND created_at < timestamptz '2026-10-26 00:00 Europe/Berlin'`; recompute the affected days.
- *Prevention:* One shared definition of a business day; tests that run reports across both DST transitions; a lint rule against hard-coded offsets.

**Lab 4: Duplicate accounts despite a unique constraint**
- *Symptom:* Customer support finds users with two accounts, `Ana@example.com` and `ana@example.com`, in a PostgreSQL database with `UNIQUE (email)`.
- *Hypotheses:* (a) the constraint is missing; (b) the comparison is case-sensitive; (c) whitespace differences.
- *Evidence:* The constraint exists. `SELECT lower(trim(email)), COUNT(*) FROM users GROUP BY 1 HAVING COUNT(*) > 1` returns 3,120 groups. PostgreSQL's default collation compares case-sensitively, so the two values are distinct. The sign-up form preserved the case the user typed.
- *Root cause:* (b). Uniqueness enforced on the raw value, but the business rule is "one account per email, ignoring case".
- *Fix:* Merge the duplicate accounts (re-pointing their orders), normalise emails on write (trim, lowercase the domain or the whole address as the business decides), and add `CREATE UNIQUE INDEX ON users (lower(email))`; compare with `lower(email) = lower($1)` at log-in.
- *Prevention:* Encode the real uniqueness rule in the index from the start; test sign-up with case variants.

---

## Solution Sketches

1. The dependency is `customer_id → city` (a customer's current city), but `city` is stored on every order, so it can disagree between orders: an update anomaly from a transitive dependency (`order_id → customer_id → city`). But a shipping address on a past order is a historical fact that must not change when the customer moves. Migration: create `customers.city` as the current city (backfilled from the most recent order); keep, on `orders`, a deliberately named `shipping_address` snapshot copied at order time, and stop reading `orders.city` as "the customer's city".
2. The guest-checkout orders have `customer_id = NULL`. For a customer with ID 3 who never ordered, `3 NOT IN (1, 1, 2, NULL)` expands to `3 <> 1 AND 3 <> 1 AND 3 <> 2 AND 3 <> NULL` = TRUE AND TRUE AND TRUE AND UNKNOWN = UNKNOWN, so `WHERE` discards every row. Rewrites: `WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)`; or a left join with `WHERE o.id IS NULL`; or `NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL)`.
3. (a) 29 March 2026 is the day Berlin's clocks go forward, so the local day has only 23 hours. A system that computes each day as "local midnight plus 24 hours" (`start + interval '24 hours'` instead of the next local midnight) ends this day at 01:00 on 30 March and includes one extra hour, while every 24-hour day comes out right; confirm by recomputing the day with explicit zone boundaries (`timestamptz '2026-03-29 00:00 Europe/Berlin'` to `'2026-03-30 00:00 Europe/Berlin'`) and counting the orders in the extra hour. (b) The systems use different zones for the boundaries (UTC days versus Berlin days), which would normally show a difference on every day; it would explain only this day if the dashboard switched from UTC to local boundaries then, so check the dashboard query's history and recompute a normal day both ways.
4. Fan-out: joining customers to orders (N rows) and to tickets (M rows) produces N × M rows per customer, so `SUM(total)` counts each order M times and `COUNT(ticket)` counts each ticket N times; customers with 2 or 3 tickets show 2× or 3× spend. Rewrite: aggregate each child table in its own subquery or CTE grouped by customer, then left-join the aggregates to customers.
5. MySQL 8's default collation `utf8mb4_0900_ai_ci` is accent- and case-insensitive, so `rene@x.com` and `rené@x.com` compare as equal and the unique index rejects the second. Decide the real rule for emails (typically: exact characters matter, case of the domain does not); give the column a collation that respects accents (for example a binary or accent-sensitive collation) and normalise case explicitly in the application or with a generated column.
6. Keep as columns everything that has constraints or is filtered, joined, sorted or aggregated routinely: price (`numeric`, `CHECK >= 0`), stock (`CHECK >= 0`, updated atomically), category (a foreign key to categories), name and SKU (`NOT NULL`, `UNIQUE`). Inside JSON these lose types, constraints and foreign keys, and need special indexes to filter. Put only genuinely variable, category-specific attributes (switch type, colour, screen size) in a `jsonb` column.
7. PostgreSQL checks non-deferrable unique constraints row by row, so while the `UPDATE` processes rows in some order, a row moved from 1 to 2 can collide with the not-yet-moved row at 2. Fixes: declare the constraint `DEFERRABLE INITIALLY IMMEDIATE`, which gives the standard end-of-statement check; or perform the shift in two steps (move every position to a temporary range, such as negated values, then to the final values); in MySQL, `UPDATE … ORDER BY pos DESC` processes rows in a safe order.

---

**END OF CHAPTER 14**
