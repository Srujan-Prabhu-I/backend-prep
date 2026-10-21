# Chapter 13: Designing Web APIs

## Where We Are

Chapter 7 taught HTTP itself: message format, methods and their safety and idempotency, status codes, headers, caching and conditional requests. Chapter 10 followed a request into the server, and Chapter 12 organised the code behind the handler: layers, DTOs, the request data flow, serialisation and a central error strategy.

All of that is machinery. This chapter is about the **interface** that the machinery exposes to other programs: the API. An API is a promise to people you may never meet, written in code they will build on for years. Once clients depend on it, every detail becomes hard to change: the shape of a JSON field, the status code of an error, the order of a list, whether a retry is safe. Good API design is mostly about making promises you can keep, under retries, concurrency, evolution and attack.

The chapter moves from what a contract is, through REST and a full worked design of an orders-and-payments API, to the hard parts that every serious API must solve: safe retries with idempotency keys, pagination that survives large tables, concurrent edits, consistent errors, versioning, long-running work and webhooks, schema evolution in binary formats, the alternatives gRPC and GraphQL, the security surface, and the API's own outbound calls, including server-side request forgery.

**After this chapter you can:**
- explain what an API contract includes, and decide whether a change is compatible;
- explain REST's constraints, and design resources and representations for a real domain;
- choose methods and status codes for each operation, and design partial updates;
- design and implement idempotency keys, including storage, concurrency and expiry, and explain how clients should retry;
- design offset and keyset pagination, and explain why offset pagination breaks at scale;
- prevent lost updates through an API with ETags, `If-Match` and version fields;
- design a consistent error format with RFC 9457 problem details, validation errors and error codes;
- version and evolve an API, telling breaking from non-breaking changes;
- design long-running operations and reliable, signed webhooks;
- evolve JSON, Protocol Buffers and Avro schemas with forward and backward compatibility;
- compare REST, gRPC and GraphQL by mechanism, and fix the GraphQL N+1 problem;
- place authentication, authorisation, validation and rate limiting in an API;
- configure outbound HTTP clients safely, and defend a URL-fetching feature against SSRF;
- describe an API with OpenAPI and protect it with contract tests.

**Interview questions this chapter prepares you for:**
1. What is an API contract? What counts as a breaking change?
2. What makes an API RESTful? Is HATEOAS used in practice?
3. Design the API for orders and payments.
4. `PUT` versus `PATCH`: how do you design partial updates? Which status codes do you return?
5. How do idempotency keys work? What happens if two retries arrive at the same time?
6. How do you design pagination? Why is offset pagination slow, and what is keyset pagination?
7. Two users edit the same resource at once. How does the API prevent lost updates?
8. How do you handle errors consistently across an API?
9. How do you version an API? How do you deprecate a version?
10. How do you expose an operation that takes ten minutes? How do you make webhooks reliable?
11. How do Protocol Buffers and Avro handle schema evolution? What are backward and forward compatibility?
12. REST versus gRPC versus GraphQL: how do they differ, and when would you choose each?
13. What is the N+1 problem in GraphQL, and how is it solved?
14. Where do authentication, authorisation and rate limiting belong in an API?
15. What is SSRF, and how do you defend a feature that fetches user-supplied URLs?

**Scope.** Here: API contracts, REST, resource modelling, method and status usage, idempotency keys, pagination, API-level concurrency control, error design, versioning, long-running operations, webhooks, schema evolution, gRPC, GraphQL, the API security surface, outbound HTTP clients, SSRF, API documentation, and a short placement-round section on SOAP and RPC styles (Section 13.16). Recalled, not re-taught: method semantics, status codes, conditional requests and the basic problem-details format (Chapter 7, Sections 7.4, 7.5 and 7.13); HMAC and webhook verification (Chapter 8, Section 8.5); CORS (Chapter 9, Section 9.7); timeouts and deadlines across hops (Chapter 10, Section 10.10); pools and Little's Law (Chapter 11); DTOs, mass assignment, serialisation and the error taxonomy (Chapter 12). Elsewhere: SQL and constraints are Chapter 14; indexes are Chapter 16; transactions and isolation are Chapter 17; authentication and authorisation in depth are Chapter 20; caching beyond HTTP is Chapter 21; retry, backoff and rate-limiting algorithms are Chapter 24; message brokers, the outbox and sagas are Chapter 25.

---

## 13.1 API Contracts and Compatibility

### Definitions

An **API** (application programming interface) is the set of operations a program offers to other programs, together with the rules for calling them. A **web API** offers them over HTTP (Chapter 1, Section 1.4 gave the working definition).

An API's **consumers** (or clients) are the programs that call it: your own web and mobile apps, other teams' services, partners' systems, scripts written by customers.

An **API contract** is everything a consumer can rely on: the endpoints, methods, request and response formats, field meanings, status codes, error formats, headers, ordering, limits, and the behaviour behind them (what is idempotent, what is eventually consistent, how long operations take).

A change is **backward compatible** (for an API) if every consumer that worked before the change still works after it, without changes of its own. A change that can break an existing consumer is a **breaking change**.

### Why it exists

A function inside one program can be renamed in minutes: the compiler finds every caller, and they are all deployed together. An API's callers live in other codebases, owned by other people, released on other schedules, and some (a mobile app installed two years ago) can never be updated. The API's owner cannot see them, cannot recompile them, and often does not know they exist. The contract is the only coordination mechanism between the two sides. If it is vague, consumers guess, and their guesses become constraints you did not know you had.

### Mechanism: what consumers actually depend on

The written contract (documentation, a schema) is only part of what consumers rely on. They also depend on everything they can observe:

```text
 written contract                        observable behaviour consumers also depend on
 ─────────────────                       ───────────────────────────────────────────────
 GET /v1/orders/{id} → 200 Order         the order of fields in the JSON (a hand-written parser)
 Order.status: "PLACED" | "PAID" | ...   that status values are UPPERCASE
 404 if the order does not exist         that the 404 body has a "message" field
 list is sorted by created_at desc       that ties are broken by id (never promised)
 p99 latency target: 300 ms              that it is usually 40 ms (a client timeout set to 100 ms)
 IDs are strings                         that IDs are always 12 digits (a database column sized for it)
```

This is summarised as **Hyrum's Law**: with enough consumers, every observable behaviour of an API will be depended on by somebody, whatever the contract says. The practical consequences are to make the contract explicit (documented formats, documented ordering, documented limits), to avoid exposing accidental behaviour (opaque IDs rather than sequential numbers, documented "do not rely on field order"), and to treat any change in observable behaviour as potentially breaking.

### Mechanism: Postel's law and its limits

The **robustness principle**, or **Postel's law** (from early internet protocol specifications), says: "be conservative in what you send, be liberal in what you accept." For APIs it gives two useful rules:

- **Send** exactly what the contract says, in its documented format, with no extra quirks.
- **Accept** inputs that differ in harmless ways. In particular, a consumer reading responses should ignore fields it does not know, so that the provider can add fields without breaking it (the **tolerant reader** of Chapter 12, Section 12.7).

Applied without limits, the second half causes harm. If a server accepts malformed input "to be helpful", clients start sending malformed input, and the server can never become strict again: the lenient behaviour has become part of the contract. Two parsers that are lenient in different ways disagree about what a message means, which is the root of request smuggling (Chapter 7, Section 7.3). The IETF's RFC 9413 ("Maintaining Robust Protocols") argues this case: tolerance hides bugs and entrenches them. A useful modern reading:

| Situation | Be strict or tolerant? | Why |
|---|---|---|
| Your API receiving requests | strict: reject malformed or unknown input with a clear `400` | errors are visible early; leniency would become contract |
| Your client reading another service's responses | tolerant of **additions** (unknown fields, new enum values) | lets the provider evolve |
| Anyone parsing security-relevant data (framing, signatures, tokens) | strict, always | ambiguity is an attack surface |

### Concrete: a compatibility check on one change

The orders API returns:

```json
{ "id": "ord_7Kq2", "status": "PAID", "total": { "amount": "39.98", "currency": "EUR" } }
```

Proposed change: add `"status": "PARTIALLY_REFUNDED"` as a new possible value. Is it compatible? Ask what each kind of consumer does when it sees a value it does not know:

- A consumer that displays the status as text: fine.
- A consumer with `switch (status)` and a default branch: fine, if the default is sensible.
- A consumer that maps the string to an enumeration and **fails on unknown values**: breaks on the first partially refunded order.
- A consumer that treats "not `PAID`" as "unpaid" and re-sends a payment request: a real-money bug.

The change is additive and looks compatible, but it is only compatible if the contract said from the start that new status values may appear and that consumers must handle unknown values. That sentence, written on day one, is what makes the change safe on day 400. Section 13.9 turns this into a table.

### Guarantees and non-guarantees

A written contract guarantees what the provider has promised, and therefore what it must not change without a new version. It does not guarantee that consumers depend only on what was promised (Hyrum's Law), that consumers read it, or that the implementation matches it (contract tests check that, Section 13.15).

### Failure and diagnosis

- **"We didn't change the API, but clients broke."** Something observable changed: field order, timing, an error message, default sorting, a new enum value. Evidence: diff real responses before and after the release (recorded traffic or contract tests), not the documentation.
- **Unremovable leniency.** The server accepts a malformed form of input for years, and tightening it breaks a large client. Evidence: log how often each lenient path is used before tightening it; announce and measure before changing.

### Backend connection

Write the contract down before writing code, including the parts that are easy to forget: ordering, limits, which fields may be added, which enumerations may grow, which operations are idempotent. Be strict about what you accept and precise about what you send; then evolve by adding, never by changing what was already observable.

---

## 13.2 REST: Constraints, Resources, Representations, HATEOAS

### Definitions

**REST** (Representational State Transfer) is an architectural style for distributed hypermedia systems, defined by Roy Fielding in his 2000 doctoral dissertation as a set of **constraints**. An API is called **RESTful** when it follows those constraints. In everyday use, "REST API" usually means something looser: an HTTP API organised around resources, using HTTP methods and status codes for their standard meanings, with JSON representations.

A **resource** is anything important enough to be named and addressed: an order, a customer, the collection of a customer's orders, a payment, even a process such as an export job. Each resource is identified by a URI (Chapter 7, Section 7.2).

A **representation** is a document that describes a resource's current or intended state, in a particular format: the JSON of order 42 is one representation of it; a PDF invoice could be another (Chapter 7, Section 7.7). Clients never touch resources directly; they exchange representations of them.

**HATEOAS** (Hypermedia As The Engine Of Application State) is the REST constraint that responses contain **links** to the actions and related resources available next, so that a client navigates the API by following links rather than by constructing URLs from documentation.

### Why it exists

Fielding derived REST from the design of the web itself, which had to scale to millions of independent servers and clients evolving separately. Each constraint buys a property: statelessness lets any server instance handle any request, a uniform interface lets intermediaries (caches, proxies) work without understanding the application, and hypermedia lets servers change URLs without breaking clients. Without some shared style, every API invents its own conventions for naming operations, reporting errors and signalling retry safety, and every client must learn each one from scratch.

### Mechanism: the six constraints

| Constraint | Meaning | What it buys | What it looks like in an HTTP API |
|---|---|---|---|
| **Client–server** | user interface and data storage are separated | each side evolves independently | front end and API are separate programs |
| **Stateless** | each request contains everything needed to understand it; the server keeps no per-client **session state** between requests | any instance can serve any request (no stickiness, Chapter 10, Section 10.3); simple retries and scaling | credentials and context in every request (a token header); resource state lives in the database, not in server memory per client |
| **Cacheable** | responses say whether and how long they may be cached | fewer requests, less latency | `Cache-Control`, `ETag` (Chapter 7, Section 7.12) |
| **Uniform interface** | one generic interface for all resources | intermediaries and generic clients understand every request | URIs identify resources; standard methods; self-descriptive messages (media types, status codes); hypermedia links |
| **Layered system** | a client cannot tell whether it talks to the origin or an intermediary | CDNs, gateways, load balancers can be inserted (Chapter 10) | the whole Chapter 10 path |
| **Code on demand** (optional) | servers may send code for clients to run | extensible clients | JavaScript sent to browsers |

"Stateless" is the most misunderstood. It does **not** mean the server has no state: the orders in the database are **resource state**, and they are the whole point. It means the server does not remember **where a particular client is in a conversation** between requests. "The user is on step 3 of the checkout wizard, stored in this instance's memory" violates it; "the checkout resource `/checkouts/c_19` has `step: 3`, stored in the database" does not, because any instance can read it.

### Mechanism: the Richardson Maturity Model

A widely used interview yardstick, proposed by Leonard Richardson, grades HTTP APIs by how much of REST they use:

```text
 Level 0  one endpoint, one method, operations named in the body   POST /api  {"action":"getOrder","id":42}
          (HTTP used as a tunnel, as in SOAP or JSON-RPC)
 Level 1  resources: many URIs, but still one method               POST /orders/42  {"action":"get"}
 Level 2  resources + HTTP methods and status codes                GET /orders/42 → 200; POST /orders → 201
 Level 3  level 2 + hypermedia links in representations            {"id":42, "_links": {"cancel": {"href": ...}}}
```

Most well-designed public APIs sit at level 2. Fielding has said that an API without hypermedia (below level 3) is not REST in his sense. In interviews, the useful answer is: describe the constraints, say which ones your design follows, and be honest that most "REST APIs" are level 2.

### Mechanism: HATEOAS, and how it is really used

A level-3 representation of an order might look like this (using a link convention similar to HAL, a common JSON hypermedia format):

```json
{
  "id": "ord_7Kq2",
  "status": "PLACED",
  "total": { "amount": "39.98", "currency": "EUR" },
  "_links": {
    "self":    { "href": "/v1/orders/ord_7Kq2" },
    "payment": { "href": "/v1/orders/ord_7Kq2/payments", "method": "POST" },
    "cancel":  { "href": "/v1/orders/ord_7Kq2/cancellation", "method": "POST" },
    "items":   { "href": "/v1/orders/ord_7Kq2/items" }
  }
}
```

The client shows a "Cancel" button only if a `cancel` link is present. When the order ships, the server stops including the link, and the client's behaviour follows the server's state machine without hard-coding it.

In practice, full HATEOAS is rare in JSON APIs. Most clients are written against documentation by people who construct URLs from templates, and generic hypermedia clients never became common. What has survived is valuable and widely used:

- **The `Location` header** on `201 Created` and `202 Accepted`, telling the client the URL of the new resource or status resource.
- **Pagination links**: `next` and `prev` URLs in a response or in a `Link` header (`Link: </v1/orders?cursor=eyJ...>; rel="next"`, a format standardised in RFC 8288). Clients follow them instead of computing page URLs (Section 13.6).
- **Action availability**: flags or links that say what is currently allowed (`"can_cancel": true`), which keeps business rules on the server.

### Guarantees and non-guarantees

Following REST's constraints guarantees properties at the architecture level: statelessness lets you add instances freely; cacheability lets intermediaries serve responses; a uniform interface lets generic tools (proxies, caches, client libraries, monitoring) understand requests. REST does not guarantee a good design: it says nothing about how to name resources, paginate, version, report errors or handle retries of `POST`. Those are the rest of this chapter.

### Failure and diagnosis

- **RPC disguised as REST.** Endpoints such as `POST /getOrder` or `GET /deleteOrder?id=42`. Symptoms: caches and crawlers misbehave (a state-changing `GET`, Chapter 7, Section 7.4), retry logic cannot tell what is safe, and every endpoint needs its own documentation of side effects.
- **Hidden server-side conversation state.** A multi-step flow stored in one instance's memory. Symptom: the flow breaks when the load balancer sends step 2 to a different instance, or after a deployment. Evidence: failures correlate with instance changes; sticky sessions "fix" it. Fix: make the conversation a resource in shared storage.

### Backend connection

Use REST's constraints as a checklist of properties you want: stateless requests so that any instance can serve them, cacheable reads, standard methods and status codes so that the whole HTTP ecosystem helps you, and links where they remove client-side guesswork (`Location`, pagination). Do not chase level 3 for its own sake, and do not call an RPC API RESTful.

---

## 13.3 Resource Modelling: An Orders and Payments API End to End

### Definition

**Resource modelling** is deciding which resources an API exposes, how they are identified and related, what their representations contain, and which operations each supports. It turns a domain model (Chapter 12, Section 12.3) into a public interface, which is not the same thing: the API exposes what consumers need, in a shape that can stay stable while the internal model changes.

### Why it matters

A resource model is the hardest part of an API to change later. URLs get bookmarked and hard-coded, identifiers get stored in other systems, and the boundaries between resources decide which operations can be atomic, which can be cached, and which require several round trips. A model copied from database tables exposes internal structure (and changes whenever the schema does); a model built from HTTP verbs on everything ends up with actions that do not fit.

### Concrete: the requirements

A shop needs an API for its web app, its mobile app and a partner marketplace:

1. A customer creates an order from a list of products and quantities, with a shipping address.
2. The customer pays for the order, through an external payment provider. Payment can succeed, fail, or need extra customer action (a bank's 3-D Secure check).
3. The customer can cancel an order before it ships.
4. Staff can refund all or part of a payment.
5. The warehouse marks orders as shipped, with a tracking number.
6. Everyone can list orders (customers their own, staff all), newest first, filtered by status.

### Mechanism: modelling step by step

**Step 1: find the nouns, not the verbs.** The candidate resources are the things that have identity and state: **orders**, the **items** of an order, **payments**, **refunds**, **shipments**, **customers**, **products**. "Cancel", "pay", "ship" and "refund" are verbs; each will become either a state change of a resource or the creation of a new resource.

**Step 2: decide identity and hierarchy.** Give every top-level resource a stable, opaque identifier. Nest a resource under another only if it cannot exist without it and is always accessed through it.

```text
 /v1/orders                         collection of orders
 /v1/orders/{orderId}               one order
 /v1/orders/{orderId}/items         its line items (exist only within an order)
 /v1/payments                       payments: top level, because providers, webhooks and refunds
 /v1/payments/{paymentId}             refer to them directly, and an order can have several attempts
 /v1/payments/{paymentId}/refunds   refunds of one payment
 /v1/orders/{orderId}/shipments     shipments of one order
```

Keep nesting to one level. `/v1/customers/81/orders/ord_7Kq2/items/3/discounts` forces clients to know the whole chain to reach a discount, and breaks when an item moves between orders. The identifiers are opaque strings with a type prefix (`ord_7Kq2`, `pay_9Xz1`). Sequential integers would reveal business volume to competitors, invite enumeration of other customers' resources (an authorisation problem if checks are weak, Section 13.13), and exceed JavaScript's safe integer range if they grow past 2^53 (Chapter 7, Section 7.7).

**Step 3: map operations onto methods.** Create, read, replace and delete fit methods directly (Chapter 7, Section 7.4). Actions that do not fit CRUD have three common designs:

| Action | Option A: create a resource that represents the action | Option B: state change with `PATCH` | Option C: custom action endpoint |
|---|---|---|---|
| cancel an order | `POST /v1/orders/{id}/cancellation` | `PATCH /v1/orders/{id}` `{"status":"CANCELLED"}` | `POST /v1/orders/{id}:cancel` |
| pay an order | `POST /v1/payments` `{"order_id": ...}` | (does not fit: a payment has its own lifecycle) | `POST /v1/orders/{id}:pay` |

Option A is usually best when the action has its own data or lifecycle (a payment has an amount, a provider reference, a status; a refund has an amount and a reason). Option B is tempting for cancellation but makes the client set an internal field directly, so the server must reject every other status transition a client attempts, and the request cannot carry action-specific data (a reason, a refund choice) cleanly. Option C, colon-suffixed custom methods, is the convention of Google's API design guide; it is honest that the operation is a command, and is fine when used sparingly. This design uses A for payments, refunds and shipments, and A for cancellation (a cancellation records who, when and why).

**Step 4: design representations.** Representations are response DTOs (Chapter 12, Section 12.3): what consumers need, in stable formats.

```json
{
  "id": "ord_7Kq2",
  "status": "PLACED",
  "customer_id": "cus_81",
  "items": [
    { "product_id": "prd_17", "name": "Mechanical keyboard", "quantity": 2,
      "unit_price": { "amount": "19.99", "currency": "EUR" } }
  ],
  "total": { "amount": "39.98", "currency": "EUR" },
  "shipping_address": { "line1": "Hauptstr. 5", "city": "Berlin", "postal_code": "10115", "country": "DE" },
  "created_at": "2026-10-21T10:15:03Z",
  "updated_at": "2026-10-21T10:15:03Z",
  "version": 1
}
```

The choices visible here: money as a decimal **string** plus currency (no binary floating point, no precision loss in JavaScript); timestamps in ISO 8601 UTC; one naming convention (`snake_case` here) everywhere; enumerations in one case; a `version` for concurrency control (Section 13.7); item names copied into the order (the order records what was bought at that time, even if the product is renamed later).

**Step 5: write the endpoint table.**

| Method and path | Purpose | Success | Idempotent? | Typical errors |
|---|---|---|---|---|
| `POST /v1/orders` | create an order | `201` + `Location` | with `Idempotency-Key` (Section 13.5) | `400`, `422` (validation, unknown product), `409` (out of stock) |
| `GET /v1/orders?status=PAID&limit=20&cursor=...` | list orders, newest first | `200` | yes (safe) | `400` (bad filter or cursor) |
| `GET /v1/orders/{id}` | read one order | `200` + `ETag` | yes (safe) | `404` (also for orders the caller may not see) |
| `PATCH /v1/orders/{id}` | change the shipping address before shipment | `200` | merge patch: in practice | `409` (already shipped), `412` (stale `If-Match`), `422` |
| `POST /v1/orders/{id}/cancellation` | cancel | `201` (or `200` if already cancelled) | naturally: a second cancel finds it cancelled | `409` (already shipped) |
| `POST /v1/payments` | pay an order | `201` (succeeded or failed), `202` (pending customer action) | with `Idempotency-Key`: **mandatory** | `409` (order not payable), `422`, `502`/`503` (provider) |
| `GET /v1/payments/{id}` | payment status | `200` | yes | `404` |
| `POST /v1/payments/{id}/refunds` | refund part or all | `201` | with `Idempotency-Key` | `422` (amount exceeds refundable) |
| `POST /v1/orders/{id}/shipments` | warehouse records shipment | `201` | with `Idempotency-Key` | `409` (cancelled or not paid) |

### Concrete: one purchase, end to end

```text
 client                                   orders API                         payment provider
   │ POST /v1/orders                           │
   │ Idempotency-Key: 5f0c…                    │
   │ {"items":[…],"shipping_address":{…}}     │
   │──────────────────────────────────────────►│ validate, reserve stock, save (Chapter 12, §12.2)
   │◄── 201 Location: /v1/orders/ord_7Kq2 ─────│ status PLACED, version 1
   │                                            │
   │ POST /v1/payments                          │
   │ Idempotency-Key: 81aa…                     │
   │ {"order_id":"ord_7Kq2","method_token":"tok_…"}
   │──────────────────────────────────────────►│ save payment (status PROCESSING)
   │                                            │── charge (its own idempotency key) ─►│
   │                                            │◄─ "requires customer action" ────────│
   │◄── 202 {"id":"pay_9Xz1","status":"REQUIRES_ACTION","next_action":{"redirect_url":…}}
   │  (customer completes the bank check in a browser)
   │                                            │◄── webhook: payment.succeeded ──────│ (signed, Section 13.10)
   │                                            │ verify; payment SUCCEEDED; order PAID (version 2)
   │ GET /v1/orders/ord_7Kq2                    │
   │──────────────────────────────────────────►│
   │◄── 200 {"status":"PAID","version":2,…} ───│
```

The design copes with each awkward fact of payments. The payment is a resource with its own status, so "pending customer action" is representable. Every unsafe `POST` carries an idempotency key, so a lost response can be retried (Section 13.5). The provider's final answer arrives asynchronously by webhook, and the order's state changes only after the signature is verified (Chapter 8, Section 8.5).

### Guarantees and non-guarantees

A resource model guarantees a stable, uniform way to name and act on things. It does not guarantee atomicity across resources: creating an order and paying for it are two requests, and the API must define what happens between them (an order that stays `PLACED` and unpaid, and expires after some time). It also does not decide authorisation: `GET /v1/orders/{id}` must still check that the caller may see that order (Section 13.13).

### Failure and diagnosis

- **Table-shaped APIs.** Endpoints mirror tables (`/order_lines`, `/order_status_history`), and clients make many calls and join data themselves. Evidence: clients' request logs show bursts of calls per screen. Fix: model what consumers need; include or expand related data deliberately (`?expand=items`).
- **Verbs in URLs everywhere.** `/createOrder`, `/updateOrderStatus`. Evidence: the endpoint list reads like a function list; method semantics do not match behaviour.
- **Leaked internal identifiers.** Sequential IDs let anyone count orders or iterate through them. Evidence: IDs are integers increasing by one.

### Backend connection

Model from the consumer's view and the domain's lifecycles, not from tables. Make things with their own lifecycle into resources (payments, refunds, shipments), give them opaque IDs, keep nesting shallow, and write the endpoint table with idempotency and error columns before writing code.

---

## 13.4 Methods, Status Codes and Partial Updates in Practice

### Definition

This section turns Chapter 7's definitions of methods and status codes (Sections 7.4 and 7.5) into conventions for an API: which method and status each kind of operation uses, and how a client changes part of a resource. A **partial update** changes some fields of a resource and leaves the rest unchanged.

### Why it matters

Consumers and intermediaries make automatic decisions from methods and status codes: whether to retry, whether to cache, whether to show an error, whether to log in again. An API that returns `200` with an error body, or `500` for a validation failure, breaks those decisions. Partial updates are where data loss hides: a client that does not know about a newly added field can erase it.

### Mechanism: the conventions for each operation

| Operation | Method | Success status | Response body | Notes |
|---|---|---|---|---|
| Create, server picks the ID | `POST` on the collection | `201 Created` + `Location` | the new resource | idempotency key for safe retries |
| Create or replace, client picks the ID | `PUT /things/{id}` | `201` (created) or `200`/`204` (replaced) | the resource, or none | naturally idempotent |
| Read one | `GET` | `200` + `ETag` | the resource | `304` for conditional reads (Chapter 7, Section 7.13) |
| Read a collection | `GET` with query parameters | `200` (even if empty) | a page of items + pagination info | an empty list is not a `404` |
| Partial update | `PATCH` | `200` with the updated resource, or `204` | updated resource saves a `GET` | `If-Match` for concurrency (Section 13.7) |
| Delete | `DELETE` | `204` (or `200` with a body) | none | deleting an already deleted resource: `204` or `404`, but documented |
| Start long work | `POST` | `202 Accepted` + `Location` of a status resource | the status resource | Section 13.10 |
| Action creating a record | `POST` on a sub-collection | `201` | the record | cancellation, refund |

Two recurring questions have conventional answers:

- **What does a second `DELETE` return?** The state is the same either way (the resource is gone), so `DELETE` is idempotent whatever is returned. Returning `204` both times is friendliest to retries; returning `404` the second time is also correct, as long as clients know a retry may see it. Document which.
- **`409` or `422` for a business rule?** Use `409 Conflict` when the request conflicts with the **current state** of the resource and could succeed in a different state (cancel a shipped order, pay an already paid order). Use `422 Unprocessable Content` when the request itself is well-formed but semantically invalid regardless of state (a refund larger than the payment, a quantity of zero). Out-of-stock sits between the two, and APIs differ; Chapter 7's example used `422`, Chapter 12's error map used `409`. Either is defensible: pick one, document it, and keep it consistent.

### Mechanism: three ways to update part of a resource

Chapter 7 (Section 7.4) introduced the two standard patch formats. Here is how they behave on a real update, applied to this order:

```json
{ "id": "ord_7Kq2", "status": "PLACED", "note": "Ring twice",
  "shipping_address": { "line1": "Hauptstr. 5", "city": "Berlin", "postal_code": "10115", "country": "DE" },
  "tags": ["gift", "priority"] }
```

**1. `PUT` with the full representation.** The client sends everything, and the server replaces everything. Any field the client omits is reset. An old mobile app that does not know the `note` field sends a body without it, and the customer's note disappears. `PUT` is only safe for partial changes if every client always sends back the complete representation it read, including fields it does not understand.

**2. JSON Merge Patch** (`Content-Type: application/merge-patch+json`, RFC 7396). The body looks like the resource, containing only what changes. Rules: a member present with a value replaces that value; a member with `null` **removes** the field; objects merge recursively; **arrays are replaced whole**.

```http
PATCH /v1/orders/ord_7Kq2 HTTP/1.1
Content-Type: application/merge-patch+json
If-Match: "v3"

{ "shipping_address": { "line1": "Hauptstr. 7" }, "note": null }
```

Result: `line1` changes, the other address fields stay (objects merge), and `note` is removed. Merge Patch is simple and covers most updates, with two limits: it cannot set a field to the value `null` (null means "remove"), and it cannot change one element of an array (to remove "priority" from `tags`, the client sends the whole new array `["gift"]`, which can overwrite a concurrent change to the same array without `If-Match`).

**3. JSON Patch** (`Content-Type: application/json-patch+json`, RFC 6902). The body is a list of operations, each with a **JSON Pointer** path (RFC 6901: `/tags/1` means the element at index 1 of `tags`). Operations are `add`, `remove`, `replace`, `move`, `copy` and `test`. The whole list is applied **atomically**: if any operation fails, none is applied.

```http
PATCH /v1/orders/ord_7Kq2 HTTP/1.1
Content-Type: application/json-patch+json

[
  { "op": "test",    "path": "/status",            "value": "PLACED" },
  { "op": "remove",  "path": "/tags/1" },
  { "op": "replace", "path": "/shipping_address/line1", "value": "Hauptstr. 7" }
]
```

The `test` operation is a built-in precondition: the patch applies only if the order is still `PLACED`. JSON Patch can express array edits and explicit nulls, but index-based paths are fragile under concurrency (index 1 may be a different tag by the time the patch arrives), so array edits still need `If-Match` or `test`. Operations such as `add` to the end of an array are not idempotent (Chapter 7, Section 7.4).

| | `PUT` | JSON Merge Patch | JSON Patch |
|---|---|---|---|
| Body | full representation | partial document | list of operations |
| Omitted field | reset or removed | unchanged | unchanged |
| Set a field to null | yes | no (null removes) | yes |
| Edit one array element | no | no (replace whole array) | yes (by index) |
| Atomic preconditions | via `If-Match` | via `If-Match` | `test` operations and `If-Match` |
| Idempotent | yes | in practice yes | depends on operations |
| Typical use | small resources, client owns all fields | most partial updates | complex edits, arrays |

### Mechanism: validating a partial update

A patch is applied to the **current** resource, and the **result** is validated as a whole, not just the patch body. The server's steps:

```pseudo
function patchOrder(orderId, patchBody, contentType, ifMatch, caller):
    order = orders.findById(orderId)                     // 404 if missing or not visible to caller
    authorise(caller, "update", order)                   // Section 13.13
    if ifMatch is present and ifMatch != etagOf(order): return 412     // Section 13.7
    dto = toUpdatableView(order)                         // ONLY client-updatable fields (mass assignment, Ch 12)
    patched = applyPatch(dto, patchBody, contentType)    // merge patch or JSON patch; 400 if malformed;
                                                         // 409 if a JSON Patch "test" fails
    errors = validate(patched)                           // whole-result validation → 422
    if errors: return problem(422, errors)
    order.applyChanges(patched)                          // domain rules: e.g. address frozen after shipping → 409
    orders.save(order)                                   // with version check (Section 13.7)
    return 200, toResponse(order), header("ETag", etagOf(order))
```

Applying the patch to an "updatable view" rather than to the full entity is what stops a patch such as `{"status": "SHIPPED", "total": {"amount": "0.01"}}` from editing fields the client must not control (Chapter 12, Section 12.3).

### Guarantees and non-guarantees

Consistent method and status conventions guarantee that generic clients, caches and retry logic behave correctly, and that consumers can handle whole classes of responses with one code path. They do not guarantee idempotency of `POST` or `PATCH` (that takes keys or careful design, Section 13.5), and patch formats do not prevent lost updates on their own (that takes `If-Match`, Section 13.7).

### Failure and diagnosis

- **Fields reset after edits.** `PUT` used for partial updates by clients that do not know newer fields. Evidence: audit history shows a field set to null by a request from an older client version. Fix: `PATCH` with Merge Patch, or require `If-Match` on `PUT` so the client must have read the latest version first.
- **Inconsistent statuses.** The same business failure returns `400` from one endpoint and `409` from another. Evidence: an error-catalogue review across endpoints (Section 13.8).
- **Patch bypasses validation.** A patch produces an object that would have been rejected at creation (a negative quantity). Cause: only the patch body was validated. Fix: validate the patched result.
- **`200` with an error inside.** Monitoring and caches see success. Evidence: responses with an `error` field and status `200`.

### Backend connection

Publish one convention table for the API and follow it everywhere: `201` with `Location` for creations, `202` for accepted long work, `204` for empty successes, `409` for state conflicts, `422` for semantic validation. Use JSON Merge Patch for ordinary partial updates and JSON Patch where arrays or explicit nulls matter, apply patches to an updatable view, and validate the result as a whole.

---

## 13.5 Idempotency and Idempotency Keys

### Definitions

Recall from Chapter 7 (Section 7.4) that an operation is **idempotent** if performing it several times leaves the same server state as performing it once.

**Idempotent design** means shaping operations so that they are idempotent by nature: `PUT` to a client-chosen ID, "set the status to cancelled" rather than "toggle the status", conditional updates that do nothing when already applied.

An **idempotency key** is a unique value that a client generates for one logical operation and sends with every attempt of it. The server records the key with the operation's outcome, so that a repeated attempt with the same key returns the recorded outcome instead of performing the operation again. It is usually sent in an `Idempotency-Key` request header. Payment APIs popularised the header; at the time of writing, the IETF's HTTP API working group is standardising it as an Internet-Draft.

### Why it exists

Chapter 7 (Section 7.15) and Chapter 10 (Section 10.10) established the uncomfortable fact behind this section: **a timeout does not prove the operation failed; the outcome is unknown**. The request may never have arrived, may have been processed with the response lost, or may still be running.

```text
 client                         API                               database / provider
   │ POST /v1/payments  ─────────►│ charge card ───────────────────────►│ charged €39.98 ✔
   │                              │ commit                               │
   │       ✕ ◄────────────────────│ 201 Created   (response lost: network reset, proxy timeout,
   │                              │                client killed by the OS, phone in a tunnel)
   │ timeout. Did it work?
   │ (a) do not retry  → if it failed, the customer never pays
   │ (b) retry blindly → if it succeeded, the customer pays twice
```

Neither choice is acceptable for money. `POST` is not idempotent, so the client cannot simply retry (Chapter 7, Section 7.4). The idempotency key makes a retry **recognisable**: the server can tell "a new payment" from "another attempt at the payment I already made".

### Mechanism 1: idempotent design first

Before reaching for keys, make operations naturally idempotent where the domain allows it:

| Instead of | Prefer | Why it is idempotent |
|---|---|---|
| `POST /orders/{id}/toggle-gift-wrap` | `PATCH /orders/{id}` `{"gift_wrap": true}` | setting a value twice gives the same value |
| `POST /accounts/{id}/add-credit {"amount": 10}` | `PUT /accounts/{id}/credits/{creditId}` with a client-chosen ID | the second `PUT` to the same ID replaces, not adds |
| cancelling fails if already cancelled | a second cancellation returns the existing cancellation | the state machine absorbs the repeat |
| `UPDATE stock SET qty = qty - 2` on every attempt | record the reservation under a unique reservation ID, decrement once | the unique ID makes the repeat a no-op |

Where the operation inherently creates something new each time (a payment, an order, a message), keys are needed.

### Mechanism 2: the client's side

```text
 user clicks "Pay"
   key = new random UUID          generated ONCE per logical operation, before the first attempt;
   persist key with the intent    stored locally (app storage, a database row) so that a crash or
                                  app restart still retries with the SAME key
 attempt 1  ──► timeout / connection reset / 502 / 503 / 504   → outcome unknown or not done: retry
 wait (exponential backoff with jitter, Chapter 24)
 attempt 2  ──► same method, same URL, same body, same Idempotency-Key
 attempt n  ──► 201 (new or replayed result) → done; forget the key later
             ──► 409 "still processing"      → wait and retry the same key
             ──► 422 "key reused with a different request" → a client bug: stop
             ──► 400 / 401 / 403 / 404 / other 422 → do not retry: the request itself is wrong
```

Two client rules matter more than any server detail. **The key belongs to the operation, not the attempt**: a client that generates a fresh key per attempt has keys and no idempotency. **The body must not change between attempts**: a retry is the same request again, not an amended one. If the user edits the cart, that is a new operation with a new key.

### Mechanism 3: the server's storage

The server keeps one row per (client, key):

```sql
CREATE TABLE idempotency_keys (
    account_id      bigint       NOT NULL,      -- keys are scoped per client account
    idem_key        text         NOT NULL,
    request_hash    bytea        NOT NULL,      -- SHA-256 of method + path + canonical body
    status          text         NOT NULL,      -- 'in_progress' | 'completed'
    locked_until    timestamptz,                -- lease: the current attempt's claim expires by itself at this time,
                                                --   so a crashed attempt cannot hold the key forever
    response_status integer,
    response_body   jsonb,
    resource_id     text,                       -- e.g. the payment created
    created_at      timestamptz  NOT NULL DEFAULT now(),
    PRIMARY KEY (account_id, idem_key)          -- the guard: one row per key, enforced by the database
);
CREATE INDEX ON idempotency_keys (created_at);  -- for expiry
```

Three design points are embedded in this schema:

- **Scope.** The primary key includes the account, so two customers who happen to choose the same key value do not see each other's results. A key from customer A must never return customer B's payment.
- **Fingerprint.** `request_hash` records what the key was first used for. The same key with a different request is a client bug and must not silently return the old result for a different payment.
- **The unique constraint is the guard.** Checking "does this key exist?" with a `SELECT` and then inserting is the check-then-act race of Chapter 5 (Section 5.2): two concurrent attempts both see "no" and both execute. The primary key makes the database decide, atomically, which attempt owns the key, exactly as in Chapter 11 (Section 11.7). Chapter 17 explains how the database enforces uniqueness under concurrency.

### Mechanism 4: the four cases

```pseudo
function handleWithIdempotency(account, key, request, operation):
    if key is missing: return problem(400, "idempotency-key-required")      // for endpoints that require it
    hash = sha256(request.method + request.path + canonicalJson(request.body))   // canonical: fixed key order and spacing, so equal bodies hash equally

    // Try to claim the key. Exactly one concurrent attempt can succeed.
    claimed = db.execute("""
        INSERT INTO idempotency_keys (account_id, idem_key, request_hash, status, locked_until)
        VALUES ($1, $2, $3, 'in_progress', now() + interval '60 seconds')
        ON CONFLICT (account_id, idem_key) DO NOTHING
        RETURNING idem_key""", account.id, key, hash)              // committed immediately

    if not claimed:
        row = db.queryOne("SELECT * FROM idempotency_keys WHERE account_id = $1 AND idem_key = $2",
                          account.id, key)
        if row.request_hash != hash:   return problem(422, "idempotency-key-reused")        // case 4
        if row.status == 'completed':  return replay(row.response_status, row.response_body) // case 3
        if row.locked_until > now():   return problem(409, "request-in-progress",
                                                      retryAfter = 2)                       // case 2
        // the previous attempt's lease expired (it crashed): take over the key
        taken = db.execute("UPDATE idempotency_keys SET locked_until = now() + interval '60 seconds'
                            WHERE account_id = $1 AND idem_key = $2 AND locked_until <= now()",
                           account.id, key).rowCount
        if taken == 0: return problem(409, "request-in-progress", retryAfter = 2)  // another retry won

    // case 1 (or a takeover): do the work. The operation must itself be safe to resume (Mechanism 5).
    result = operation(request, key)
    db.execute("""UPDATE idempotency_keys
                  SET status = 'completed', response_status = $3, response_body = $4,
                      resource_id = $5, locked_until = NULL
                  WHERE account_id = $1 AND idem_key = $2""",
               account.id, key, result.status, result.body, result.resourceId)
    return result
```

The four cases as timelines:

```text
 CASE 1  first request
   A1: INSERT key (in_progress) ✔  → charge → save payment → UPDATE key (completed, 201, body) → 201

 CASE 2  retry while the first attempt is still running
   A1: INSERT key ✔ ─── charging (provider takes 4 s) ──────────────────► completed → 201 (lost)
   A2:       INSERT key ✕ conflict → row: in_progress, lease valid → 409 Retry-After: 2
   A3:                                         (2 s later) INSERT ✕ → completed → replay 201

 CASE 3  retry after completion
   A1: … completed → 201 (response lost)
   A2: INSERT ✕ conflict → row: completed, hash matches → 201 with the SAME body (same payment id)
       nothing is charged again

 CASE 4  same key, different payload
   A1: key K for {"order_id":"ord_7Kq2","amount":"39.98"} → completed
   A2: key K for {"order_id":"ord_9Bc3","amount":"12.00"} → hash differs → 422 idempotency-key-reused
       (returning A1's result would tell the client that ord_9Bc3 was paid: a silent, serious bug)
```

The status codes for cases 2 and 4 (`409` and `422`) and for a missing key (`400`) follow the IETF draft's recommendations. Some APIs mark a replayed response with a header so that clients and logs can tell a replay from a first execution (Stripe, for example, sends `Idempotent-Replayed: true`).

### Mechanism 5: making the work itself safe to resume

The server-side record handles retries of the **HTTP request**. Two harder questions remain: what if the server crashes in the middle of the work, and what if the work includes a call to another system?

**When all the work is in the same database**, put the business writes and the key's completion in **one transaction**. Then the outcome is all or nothing: either the payment row and the completed key both exist, or neither does, and a retry after a crash simply starts again.

```sql
BEGIN;
  INSERT INTO idempotency_keys (account_id, idem_key, request_hash, status)
  VALUES (81, '5f0c…', '\x9a…', 'in_progress');   -- a concurrent duplicate INSERT now waits for us
  INSERT INTO orders (...) VALUES (...);
  UPDATE idempotency_keys SET status = 'completed', response_status = 201, response_body = '{…}'
  WHERE account_id = 81 AND idem_key = '5f0c…';
COMMIT;
```

In PostgreSQL, a concurrent transaction inserting the same primary key **waits** for this one to finish. If this one commits, the other sees a conflict and replays the stored response; if it rolls back, the other proceeds as the first attempt. This is the simplest correct design, at the cost of the duplicate holding a connection while it waits, which is acceptable only when the work is short.

**When the work calls another system** (a payment provider), the call cannot be inside the database transaction (Chapter 11, Section 11.5; Chapter 12, Section 12.10), and a crash can happen after the charge but before the result is saved. Two techniques make this safe:

1. **Pass idempotency downstream.** Send the provider a key derived from yours (for example `"pay:" + idem_key`). If your server crashes and a retry takes over the key, it calls the provider again with the **same** downstream key, and the provider returns the original charge instead of charging twice. Idempotency is only as strong as the weakest hop.
2. **Record progress in steps.** Save the payment as `PROCESSING` before calling the provider; after the call, save the provider's reference and result. A resumed attempt reads where the previous one stopped and continues from there. Operations that cannot finish synchronously are completed later by webhooks or by a reconciliation job that asks the provider for the outcome (Chapter 25 covers multi-step workflows).

A final layer of defence is a **business-level unique constraint**, independent of keys. Even if a client sends two different keys for the same order (a bug, or two browser tabs), the database can refuse a second successful payment:

```sql
-- PostgreSQL partial unique index: at most one succeeded payment per order
CREATE UNIQUE INDEX one_success_per_order ON payments (order_id) WHERE status = 'SUCCEEDED';
```

### Mechanism 6: expiry

Keys cannot be kept forever, and they need not be: a key only has to outlive the client's retry window. Keep keys for longer than the longest time a client might keep retrying (24 hours is a common choice; Stripe documents that keys may be removed after they are at least 24 hours old), and delete old rows in batches:

```sql
DELETE FROM idempotency_keys
WHERE (account_id, idem_key) IN (
    SELECT account_id, idem_key FROM idempotency_keys
    WHERE created_at < now() - interval '24 hours'
    LIMIT 5000);                       -- repeat until no rows are deleted; small batches avoid long locks
```

After expiry, a request with the old key is treated as new. The contract must say so, so that clients do not retry for days with the same key. Storing keys in an in-memory cache with a short time-to-live instead of a durable store weakens the guarantee: a cache eviction or restart turns a retry into a duplicate.

### Guarantees and non-guarantees

With a durable, uniquely constrained key store, a fingerprint check and downstream keys, idempotency keys guarantee that retries of one logical operation, with the same key and body, within the retention window, are applied at most once and return the same result. They do not guarantee **exactly-once** behaviour in general (Chapter 25): two different keys for the same intent create two operations; a key that expired is forgotten; and a downstream system without idempotency can still be called twice. They do not make clients retry correctly: a client that changes the key or the body defeats them.

### Failure and diagnosis

- **Duplicate charges despite keys.** Evidence: two payments for one order with **different** keys created seconds apart: the client generated a key per attempt, or two tabs submitted. Fix: key per operation, persisted; business-level unique constraint.
- **Duplicates with the same key.** Evidence: two rows of work for one key. Cause: check-then-insert without a unique constraint, or keys kept in an evicted cache. Fix: the primary key as the guard; durable storage.
- **Keys stuck "in progress".** Clients get `409` forever after a server crash. Cause: no lease expiry. Evidence: rows with `in_progress` and old `created_at`. Fix: leases and takeover, plus resumable work.
- **Wrong results replayed.** A client receives a response for a different request. Cause: no fingerprint check, or keys not scoped per account. Evidence: `request_hash` mismatches that were not rejected.

### Backend connection

Every endpoint that creates something or moves money should accept an idempotency key, and payment-like endpoints should require it. Make the key's uniqueness a database constraint, fingerprint the request, return `409` while in progress and the original response afterwards, pass keys to downstream systems, and keep a business-level unique constraint as the last guard. Write the client half too: one key per operation, persisted, reused on every retry.

---

## 13.6 Pagination, Filtering and Sorting

### Definitions

**Pagination** splits a large collection into pages that a client fetches one at a time. **Offset pagination** identifies a page by how many items to skip (`?offset=40&limit=20`, or `?page=3&size=20`). **Cursor pagination** identifies a page by an opaque **cursor**, a token from the previous response that marks where the next page starts. **Keyset pagination** (also called **seek pagination**) is the usual implementation of cursors: the next page is "items whose sort key comes after the last item I received", which the database finds with an index.

### Why it exists

Returning a whole collection in one response fails as data grows: a customer with 30,000 orders gets a 40 MB response, the server builds it in memory, and the database scans everything. Every collection endpoint needs a limit. The design question is how the client asks for the next part, and the obvious answer, offset, has two problems that only appear at scale.

### Mechanism 1: offset pagination and its two failures

```sql
-- page 3 of a customer's orders, 20 per page, newest first
SELECT id, created_at, status, total_cents
FROM orders
WHERE customer_id = 81
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 40;
```

**Failure 1: cost grows with depth.** `OFFSET 40` does not jump to row 41. The database must produce the first 60 rows in order and throw away 40. With an index on `(customer_id, created_at DESC, id DESC)` (Chapter 16 explains indexes), it walks 60 index entries. At page 5,001, `OFFSET 100000`, it walks 100,020 entries to return 20. The cost of a page is roughly proportional to its offset:

```text
 page    offset     rows produced and discarded     relative cost (assumption: cost ∝ rows walked)
 1       0          20                              1×
 50      980        1,000                           50×
 5,001   100,000    100,020                         ~5,000×
```

Deep pages are rare from humans but common from scripts that export everything page by page: the export gets slower with each page, and the total work is quadratic in the number of pages. Exports and crawlers that go deep can saturate a database with requests that each look harmless.

**Failure 2: rows shift between requests.** Offsets count positions, and positions change when rows are inserted or deleted:

```text
 t1  client fetches page 1 (offset 0):  orders 100, 99, 98 … 81
 t2  a new order 101 is created         now 101 is position 1, and 81 moves to position 21
 t3  client fetches page 2 (offset 20): starts at position 21 → order 81 AGAIN (duplicate)

 if instead order 95 is deleted at t2:  order 80 moves to position 20
 t3  page 2 (offset 20) starts at position 21 → order 79; order 80 is never seen (skipped)
```

For a user scrolling a feed, a duplicate is cosmetic. For a synchronisation job copying orders to an accounting system, a skipped row is a missing invoice.

### Mechanism 2: keyset pagination

Keyset pagination remembers the **sort key of the last row** returned and asks for rows after it:

```sql
-- first page
SELECT id, created_at, status, total_cents
FROM orders
WHERE customer_id = 81
ORDER BY created_at DESC, id DESC
LIMIT 21;                                         -- one extra row tells us whether there is a next page

-- next page: the last row of the previous page had created_at = '2026-10-19 08:02:11+00', id = 'ord_5Lm0'
SELECT id, created_at, status, total_cents
FROM orders
WHERE customer_id = 81
  AND (created_at, id) < ('2026-10-19 08:02:11+00', 'ord_5Lm0')   -- row-value comparison
ORDER BY created_at DESC, id DESC
LIMIT 21;
```

With the index on `(customer_id, created_at DESC, id DESC)`, the database **seeks** directly to the position after the last row and reads 21 entries, whatever the depth: page 5,001 costs the same as page 2. Inserted or deleted rows do not shift anything, because the next page is defined by a value, not a position.

Three details make it correct:

1. **The sort must be a total order.** `created_at` alone is not unique; two orders created in the same millisecond would be split unpredictably across pages, and one could be skipped. Adding the unique `id` as a **tie-breaker** makes every position unique.
2. **The comparison must match the sort.** For `DESC, DESC`, "after" means `<` on the pair. PostgreSQL evaluates the row-value comparison `(a, b) < (x, y)` and uses the index for it. Where a database does not, write it out: `created_at < $1 OR (created_at = $1 AND id < $2)`.
3. **The cursor is opaque.** The API does not expose the SQL. It encodes the last row's sort values into a token: for example Base64url of `{"c":"2026-10-19T08:02:11Z","i":"ord_5Lm0","s":"created_desc"}` (the last row's creation time and ID, plus the sort the cursor belongs to). Base64 is an encoding, not protection (Chapter 8, Section 8.2): clients can decode and forge it, so the server validates every field of a cursor it receives (or signs it with an HMAC, Chapter 8, Section 8.5, to reject tampering), and treats it as valid only with the same sort and filters.

```http
GET /v1/orders?status=PAID&limit=20 HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json
Link: </v1/orders?status=PAID&limit=20&cursor=eyJjIjoiMjAyNi0xMC0xOVQwODowMjoxMVoiLCJpIjoib3JkXzVMbTAiLCJzIjoiY3JlYXRlZF9kZXNjIn0>; rel="next"

{
  "data": [ { "id": "ord_7Kq2", ... }, ... 20 items ... ],
  "next_cursor": "eyJjIjoiMjAyNi0xMC0xOVQwODowMjoxMVoiLCJpIjoib3JkXzVMbTAiLCJzIjoiY3JlYXRlZF9kZXNjIn0",
  "has_more": true
}
```

### Mechanism 3: what keyset pagination costs

| | Offset | Keyset (cursor) |
|---|---|---|
| Cost per page | grows with depth | constant (with a matching index) |
| Inserts and deletes during paging | duplicates and skips | none for rows whose sort key does not change |
| Jump to page 37 | yes | no: only next (and previous, by reversing the comparison) |
| "Page 3 of 812" | needs a count | needs a count, or none |
| Sorting | any column | only sorts with a usable index and a tie-breaker |
| Implementation | trivial | more care; cursors must be validated |

Total counts deserve a warning: `SELECT count(*)` over a large filtered set can cost as much as reading it. Many APIs return `has_more` instead of a total, or an approximate count.

### Mechanism 4: filtering and sorting

Filters and sort orders are part of the contract, and each one is a query the database must be able to run efficiently:

- **Allowlist** filterable fields and operators (`status=PAID`, `created_after=2026-10-01T00:00:00Z`). Never pass client field names into SQL; map each allowed name to a column (SQL injection, Chapter 18).
- **Allowlist sort keys** (`sort=created_at`, `sort=-total`), each backed by an index that starts with the filter columns and ends with the tie-breaker. A sort the database cannot serve from an index means sorting the whole filtered set on every page.
- **Validate types and ranges** (`limit` between 1 and 100; dates parseable) and reject the rest with `400`.
- **Bind the cursor to the query.** A cursor created for `status=PAID&sort=-created_at` is meaningless for `sort=total`; reject the mismatch with `400` rather than returning wrong pages.

### Guarantees and non-guarantees

Keyset pagination guarantees that, for rows that exist throughout the traversal and whose sort keys do not change, every row is returned exactly once, at a cost per page independent of depth. It does not guarantee a consistent snapshot: rows inserted after the client passed their position are not seen, and a row whose sort key changes during the traversal (an order whose `updated_at` is the sort key and is updated) can be skipped or seen twice. A true point-in-time export needs a stable sort key (an immutable creation time or ID) or an export job that reads a database snapshot (Chapter 17). Offset pagination guarantees only simplicity.

### Failure and diagnosis

- **Deep pages slow; database CPU high from "simple" list queries.** Evidence: the slow-query log shows the same query with growing `OFFSET`; access logs show a client walking pages sequentially. Fix: keyset pagination; a dedicated export endpoint or job.
- **Missing or duplicated records in a sync.** Evidence: a consumer's copy differs from the source; the gaps correspond to rows inserted or deleted during a sync run. Fix: keyset over an immutable key; an `updated_since` cursor for incremental sync with a tie-breaker.
- **Unindexed sorts.** Some sort options are fast, others time out. Evidence: query plans showing a full sort (Chapter 16). Fix: remove the sort option or add a matching index.
- **Unbounded pages.** A client sends `limit=100000`. Fix: a documented maximum, enforced.

### Backend connection

Every collection endpoint needs a limit from day one, because adding one later is a breaking change. Use cursor (keyset) pagination by default, with a unique tie-breaker, an index that matches the filter and sort, and an opaque, validated cursor. Allowlist filters and sorts, cap the page size, and offer a separate export mechanism for "everything".

---

## 13.7 Concurrency Control in APIs

### Definition

**API concurrency control** prevents one client's update from silently overwriting another's, when both read the same version of a resource and then write changes. The standard HTTP mechanism is **optimistic concurrency control** with conditional requests (Chapter 7, Section 7.13): every representation carries a version, and a write succeeds only if the resource is still at the version the client read.

### Why it exists

Two support agents open the same order:

```text
 t1  Agent A: GET /v1/orders/ord_7Kq2   → address "Hauptstr. 5", note "Ring twice"   (version 3)
 t2  Agent B: GET /v1/orders/ord_7Kq2   → same                                         (version 3)
 t3  Agent A: PUT … address "Hauptstr. 7", note "Ring twice"       → saved (version 4)
 t4  Agent B: PUT … address "Hauptstr. 5", note "Leave at door"    → saved (version 5)
     Agent A's address change is gone. Nobody was told.
```

This is the **lost update** of Chapter 5 (Section 5.2), with HTTP requests seconds or minutes apart instead of threads nanoseconds apart. Database locks cannot help across that gap: no request holds a transaction open while a human edits a form (Chapter 11, Section 11.5 explained why holding resources across slow steps is fatal).

### Mechanism: ETags, If-Match, and the version check in the database

```text
 Agent A                               API                                  database
  │ GET /v1/orders/ord_7Kq2 ──────────►│ SELECT … version = 3                 │
  │◄── 200  ETag: "v3"  {…}            │                                      │
  │                                    │                                      │
  │ PATCH … If-Match: "v3"  ──────────►│ UPDATE orders SET …, version = 4    │
  │                                    │   WHERE id = 'ord_7Kq2' AND version = 3   → 1 row updated
  │◄── 200  ETag: "v4"                 │                                      │
 Agent B
  │ PATCH … If-Match: "v3"  ──────────►│ UPDATE … WHERE id = … AND version = 3     → 0 rows updated
  │◄── 412 Precondition Failed         │   (the version is now 4)
  │     (problem+json: "modified by someone else; re-read and retry")
```

The essential detail is **where the comparison happens**. Reading the current version, comparing it in application code, and then updating is check-then-act: another request can update between the check and the write. The comparison must be part of the `UPDATE` itself, with the number of affected rows deciding the outcome (the same pattern as Chapter 11, Section 11.7):

```pseudo
function updateOrder(id, changes, expectedVersion):
    rows = db.execute("UPDATE orders SET address = $1, note = $2, version = version + 1, updated_at = now()
                       WHERE id = $3 AND version = $4",
                      changes.address, changes.note, id, expectedVersion)
    if rows == 0:
        if not db.exists("SELECT 1 FROM orders WHERE id = $1", id): return 404
        return 412                                   // it exists, so the version did not match
    return 200 with ETag = '"v' + (expectedVersion + 1) + '"'
```

Details that make the scheme work:

- **The ETag must change on every change.** Derive it from a version column incremented on every write, not from `updated_at` with one-second resolution (two updates in one second share a timestamp, Chapter 7, Section 7.12), and not from a hash of a representation that includes volatile fields (a "last viewed" time makes the ETag change constantly).
- **`If-Match` requires a strong ETag.** Weak ETags (`W/"…"`) mean "semantically equivalent" and must not be used for write preconditions (Chapter 7, Section 7.13).
- **Make the precondition mandatory** on endpoints where lost updates matter. A request without `If-Match` gets **`428 Precondition Required`**, so a careless client cannot skip the check.
- **A version in the body** is an alternative for clients that cannot set headers or for batch operations: `{"version": 3, "address": {...}}`. A mismatch then conventionally returns `409 Conflict`, since no HTTP precondition header was involved. Pick one style per API.

What the client does with `412` is a product decision: re-read and show the user what changed, merge automatically when the changed fields do not overlap, or retry automatically for fields where last-writer-wins is acceptable.

**Pessimistic** alternatives (an explicit "lock" or "checkout" resource that other editors must respect, with an expiry) exist for long editing sessions where conflicts are expensive, such as document editors. They need lock expiry, lock stealing and UI for "locked by Agent A", and they fail when a client vanishes holding the lock. For most APIs, optimistic control is simpler and better.

### Guarantees and non-guarantees

Conditional writes with a version checked inside the database statement guarantee that no write is applied on top of a version the writer did not see: lost updates become visible `412` (or `409`) responses. They do not merge changes, do not guarantee progress under heavy contention (two clients can keep invalidating each other), and do not protect invariants spanning several resources (that needs transactions, Chapter 17). Clients that omit the precondition are unprotected unless it is required (`428`).

### Failure and diagnosis

- **Silent overwrites.** Users report that their changes "disappeared". Evidence: audit history shows two writes based on the same prior version seconds apart. Fix: ETags, `If-Match`, version checks; `428` when missing.
- **Every write fails with `412`.** The ETag changes when nothing meaningful changed (volatile fields in the hash), or the ETag sent with responses differs from the one checked (computed from different representations). Evidence: two consecutive `GET`s return different ETags with no write between them.
- **Rare lost updates despite ETags.** The version is compared in application code before an unconditional `UPDATE`. Evidence: concurrent tests show two successful writes from the same version. Fix: put the version in the `WHERE` clause.

### Backend connection

Give every mutable resource a version column, expose it as a strong ETag, require `If-Match` on updates where lost updates matter, and enforce the comparison inside the `UPDATE` statement. Decide with product owners what clients do on `412`.

---

## 13.8 Errors: Problem Details, Validation Errors, Error Codes

### Definition

An API's **error contract** is the format, status codes, identifiers and semantics of every error response it can return. Chapter 7 (Section 7.5) introduced **RFC 9457 problem details**, the standard JSON error body (`application/problem+json`) with the members `type`, `title`, `status`, `detail` and `instance`. Chapter 12 (Section 12.8) built the server-side machinery: an error taxonomy, translation at boundaries, and one central handler. This section designs the error contract that consumers see.

### Why it exists

Consumers write code against errors: show a message next to the right form field, retry later, ask the user to log in again, refresh and retry after a conflict, contact support with a reference. They can only do this if errors are consistent across endpoints and machine-readable. An API in which one endpoint returns `{"error": "bad"}`, another `{"message": "...", "code": 17}` and the gateway an HTML page forces every consumer to special-case each one.

### Mechanism 1: the error identifier

The `type` member is the problem's stable, machine-readable identifier: a URI, which may point to documentation. Consumers branch on `type` (and `status`), never on the human-readable `title` or `detail`, which may be reworded or translated. When a status code alone says everything, `type` may be omitted or set to `about:blank`, which means "no more than the status code".

Build an **error catalogue**: a documented list of problem types, each with its status, meaning and what the client should do.

| `type` (suffix of `https://api.shop.example/problems/…`) | Status | Meaning | Client action |
|---|---|---|---|
| `validation-failed` | `422` | one or more fields are invalid; see `errors` | fix the listed fields |
| `malformed-request` | `400` | body is not valid JSON or has wrong types | fix the client |
| `unauthenticated` | `401` | missing or invalid credentials | obtain new credentials (Chapter 20) |
| `forbidden` | `403` | authenticated, not allowed | do not retry |
| `not-found` | `404` | resource does not exist or is not visible | do not retry |
| `order-not-cancellable` | `409` | the order's state forbids it | refresh and show the state |
| `version-mismatch` | `412` | `If-Match` did not match | re-read, then retry or merge |
| `precondition-required` | `428` | `If-Match` missing | send it |
| `idempotency-key-reused` | `422` | key used with a different request | client bug: new key |
| `request-in-progress` | `409` | same key still being processed | retry after `Retry-After` |
| `rate-limited` | `429` | too many requests | wait for `Retry-After` |
| `dependency-unavailable` | `503` | a downstream system is failing | retry with backoff |
| `internal-error` | `500` | a bug; details are in the logs | retry may fail; report the `request_id` |

Adding a type is a compatible change if consumers were told to handle unknown types by falling back to the status code. Changing an existing type's meaning or status is a breaking change.

### Mechanism 2: validation errors

A form with three invalid fields should get all three errors in one response, each tied to its field. RFC 9457 allows **extension members**, and its own example uses an `errors` array with a JSON Pointer to each invalid member:

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.shop.example/problems/validation-failed",
  "title": "Your request is not valid.",
  "status": 422,
  "detail": "3 fields failed validation.",
  "instance": "/v1/orders",
  "request_id": "req_7f3a19",
  "errors": [
    { "pointer": "#/items/0/quantity",          "code": "must-be-positive",  "detail": "must be at least 1" },
    { "pointer": "#/items/1/product_id",        "code": "unknown-product",   "detail": "product prd_99 does not exist" },
    { "pointer": "#/shipping_address/postal_code", "code": "invalid-format", "detail": "must be 5 digits for DE" }
  ]
}
```

The `pointer` uses JSON Pointer syntax (RFC 6901) prefixed with `#`, so a client can attach each message to the right input even inside arrays. A short per-field `code` lets clients localise messages themselves.

### Mechanism 3: other examples

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{ "type": "https://api.shop.example/problems/order-not-cancellable",
  "title": "This order can no longer be cancelled.",
  "status": 409,
  "detail": "Order ord_7Kq2 shipped at 2026-10-21T14:02:00Z.",
  "instance": "/v1/orders/ord_7Kq2/cancellation",
  "order_status": "SHIPPED",
  "request_id": "req_91bc04" }
```

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 30

{ "type": "https://api.shop.example/problems/rate-limited",
  "title": "Too many requests.",
  "status": 429,
  "detail": "Limit of 100 requests per minute exceeded for this API key.",
  "request_id": "req_c3d2e1" }
```

```http
HTTP/1.1 500 Internal Server Error
Content-Type: application/problem+json

{ "type": "https://api.shop.example/problems/internal-error",
  "title": "An internal error occurred.",
  "status": 500,
  "request_id": "req_0a9f77" }
```

The `500` says nothing about the cause. The `request_id` lets support find the full stack trace in the logs (Chapter 12, Section 12.8; Chapter 10, Section 10.11).

### Mechanism 4: consistency across the whole path

Errors are produced by more components than the application: the gateway rejects unauthenticated requests, the load balancer returns `502` and `503`, the framework rejects malformed JSON before your code runs (Chapter 10, Section 10.8). Consumers see all of them as "the API". To keep the contract:

- configure the gateway and proxy to return problem-details bodies (or at least JSON) for the errors they generate, not default HTML pages;
- map the framework's own exceptions (binding, missing parameters, unsupported media type) in the central handler;
- add the request ID to every error, including those from the edge.

### Guarantees and non-guarantees

A documented catalogue of problem types guarantees consumers a stable identifier and status for each documented failure, and one parsing path for all of them. It does not guarantee that intermediaries follow it unless configured, that consumers handle new types gracefully unless told to, or that the `detail` text is stable (it is for humans).

### Failure and diagnosis

- **Clients parse `detail` strings.** A reworded message breaks a client. Evidence: client code matching message text. Fix: document that `type` and `code` are the stable identifiers.
- **HTML error pages from the edge.** Clients fail to parse a `502` or `413`. Evidence: `Content-Type: text/html` and a proxy `Server` header on error responses (Chapter 10, Section 10.8). Fix: custom error responses at the gateway.
- **Errors leak internals.** Evidence: stack traces, SQL or class names in `detail`. Fix: the generic `500` body; details only in logs.
- **One error at a time.** A form needs five round trips to fix five fields. Fix: collect all validation errors before responding.

### Backend connection

Adopt problem details for every error, publish the catalogue of types, return all validation errors at once with pointers, include a request ID everywhere, and make the edge speak the same format. Treat the catalogue as part of the API: new types are additions, changed types are breaking changes.

---

## 13.9 Versioning and Evolution

### Definitions

**API evolution** is changing an API while existing consumers keep working. **API versioning** is running more than one incompatible form of an API at the same time, so that consumers can move from the old one to the new one on their own schedule. **Deprecation** is announcing that a version or feature will be removed, and **sunset** is the date on which it stops working.

### Why it exists

Requirements change, and some changes cannot be made compatibly: a field's meaning changes, a resource is split, an error format is redesigned. Consumers cannot all update at once; mobile apps in particular stay installed for years. Without a strategy, the provider must either never change anything or break consumers.

### Mechanism 1: breaking versus non-breaking changes

The table assumes that consumers follow the tolerant-reader rules the contract asked for (ignore unknown fields, handle unknown enumeration values and error types).

| Change | Breaking? | Why |
|---|---|---|
| Add a new endpoint | no | nobody calls it yet |
| Add an optional request field (with a default that keeps old behaviour) | no | old clients do not send it and get the old behaviour |
| Add a response field | no, for tolerant readers | strict parsers that reject unknown fields break; the contract must forbid that |
| Add a value to an enumeration in a **response** | no, if the contract said enumerations may grow; otherwise yes | Section 13.1's `PARTIALLY_REFUNDED` |
| Add a new error type with a standard status | no, if clients fall back to the status | |
| Make a response faster, add a header | no | |
| Remove or rename a field | **yes** | clients reading it get nothing |
| Change a field's type (`"total": 39.98` → `"total": {"amount":"39.98",...}`) | **yes** | parsers fail or misread |
| Change a field's meaning or units (cents → euros) | **yes, and silently** | nothing fails; numbers become wrong |
| Make an optional request field required | **yes** | old requests are rejected |
| Add a new **required** request field | **yes** | old requests are rejected |
| Tighten validation (max length 100 → 50) | **yes** | previously valid requests fail |
| Change a status code (`200` → `201`, `404` → `410`) | **yes** | clients branch on it |
| Change default sorting, default page size, pagination style | **yes** | clients rely on order and page shape |
| Change authentication or required scopes | **yes** | |
| Remove an enumeration value clients send | **yes** | |
| Loosen validation | usually no | but beware: what was rejected may now be stored and returned to clients that cannot handle it |

Most evolution should be additive. A rename can be done without a new version by **expand and contract**: add the new field alongside the old (`shipping_address` and `delivery_address`), write both, document the old one as deprecated, measure until no client reads or writes it, and then remove it in the next major version.

### Mechanism 2: where the version goes

| Style | Example | Strengths | Weaknesses |
|---|---|---|---|
| URL path | `GET /v2/orders/ord_7Kq2` | visible in logs, links and browsers; easy routing at the gateway; easy caching | the same resource has several URLs; tempts teams to version everything at once |
| Custom header | `Api-Version: 2` | URLs stay stable | invisible in links and some logs; caches need `Vary: Api-Version`; easy to forget |
| Media type | `Accept: application/vnd.shop.order.v2+json` | versions each representation; uses content negotiation (Chapter 7, Section 7.8) | complex for clients and tools; caches need `Vary: Accept` |
| Query parameter | `GET /orders/ord_7Kq2?version=2` | simple | mixes versioning into the query; caching and logs as for URLs |
| Date-based, pinned per account | `Api-Version: 2026-10-01`, defaulting to the date the account was created | many small breaking changes, each opt-in; old clients never surprised | the provider must maintain transformations for every dated change |

URL-path versioning is the most common for public APIs because it is the most visible. The date-based model is used by some large API providers (Stripe is a well-known example): the server implements the newest behaviour once, and each older version is a small **transformation layer** that converts requests and responses between the client's pinned version and the current one:

```text
 client pinned to 2025-03-01
   request  ──► [ transform 2025-03-01 → 2025-09-15 ] ──► [ transform 2025-09-15 → current ] ──► handler (current)
   response ◄── [ transform back ]                     ◄── [ transform back ]                ◄──
```

This keeps one implementation of the business logic, rather than parallel copies of controllers and services per version, which is the usual and expensive alternative.

### Mechanism 3: deprecation and sunset

Removing a version is a project, not a commit:

```text
 1. announce      changelog, email to registered developers, documentation banner
 2. signal        on every response of the deprecated version:
                    Deprecation: @1792800000           (when it was deprecated, as a timestamp)
                    Sunset: Tue, 30 Jun 2027 00:00:00 GMT
                    Link: <https://api.shop.example/docs/migrate-v2>; rel="deprecation"
 3. measure       requests per version per API key / client application, every week
 4. contact       the remaining heavy users individually
 5. brownouts     planned, announced short periods in which the old version returns errors
                  (for example 10 minutes, then an hour), so that forgotten clients surface
                  while there is still time
 6. remove        the old version returns 410 Gone with a problem-details body pointing to the guide
```

The `Sunset` header is standardised in RFC 8594; a `Deprecation` header field was standardised more recently (RFC 9745). Both only inform; a client must be built to log or alert on them for them to be useful.

### Guarantees and non-guarantees

A versioning policy guarantees consumers that a given version's contract will not break while that version is supported. It does not guarantee that consumers migrate before the sunset, that "non-breaking" changes break nobody (Hyrum's Law, Section 13.1), or that running several versions is cheap: each supported version multiplies testing, documentation and security fixes.

### Failure and diagnosis

- **An additive change breaks a client.** A strict parser rejects a new field, or a new enum value triggers an error path. Evidence: a client error spike starting exactly at the release, with parsing errors in client logs. Prevention: the contract's tolerant-reader rule, consumer contract tests (Section 13.15), staged rollouts.
- **Silent semantic change.** A field's units or meaning changed without a version. Evidence: no errors, but downstream reports disagree after a date. This is the worst kind of break; treat any change of meaning as a new field.
- **Versions that never die.** v1 still receives traffic years after v3. Evidence: per-version request metrics. Fix: the deprecation process above, with brownouts.
- **Version sprawl inside the code.** Controllers and services copied per version, with fixes applied to some copies. Fix: one current implementation plus transformation layers or adapters at the edge (Chapter 12, Section 12.10).

### Backend connection

Design for evolution so that versions are rare: tolerant readers, additive changes, expand-and-contract for renames. When a breaking change is unavoidable, version visibly (usually in the path), keep one implementation of the business logic with transformations at the edge, measure who uses what, and retire versions with announced headers, brownouts and `410 Gone`.

---

## 13.10 Long-Running Operations and Webhooks

### Definitions

A **long-running operation** is work that takes longer than a client can reasonably wait on one HTTP request: generating a large export, importing a file, transcoding a video, a payment waiting for the customer's bank. The API accepts it with **`202 Accepted`** and returns the URL of a **status resource** (also called an operation resource) that the client can read to follow progress and find the result.

**Polling** is the client repeatedly reading the status resource until the work finishes.

A **webhook** is an HTTP request that a provider sends to a URL the consumer registered in advance, to notify it that an **event** happened ("payment succeeded", "export ready"). It reverses the direction of the API: the provider becomes the client.

### Why it exists

Every hop on the request path has a timeout (Chapter 10, Section 10.10): the client, the CDN, the load balancer, the proxy. Work that takes ten minutes cannot fit inside them, and a request held open for ten minutes holds a connection, often a worker, and is lost entirely if any hop restarts. Long work must be decoupled from the request that started it. Then the client needs a way to learn the outcome: by asking (polling) or by being told (webhooks, or the push channels of Chapter 10, Section 10.9).

### Mechanism 1: 202 and a status resource

```http
POST /v1/exports HTTP/1.1
Idempotency-Key: 3b7e…
Content-Type: application/json

{ "kind": "orders", "created_after": "2026-01-01T00:00:00Z", "format": "csv" }
```

```http
HTTP/1.1 202 Accepted
Location: /v1/exports/exp_4Tq8
Retry-After: 5
Content-Type: application/json

{ "id": "exp_4Tq8", "status": "QUEUED", "created_at": "2026-10-21T11:00:00Z" }
```

```text
 client                          API instance               job store (DB) + queue           worker
   │ POST /v1/exports ──────────►│ INSERT exports (QUEUED)  ──►│                                 │
   │                             │ enqueue job exp_4Tq8     ──►│ ───────────────────────────────►│ picks job
   │◄── 202 Location … ─────────│                             │◄── UPDATE status RUNNING, 10% ─│
   │ GET /v1/exports/exp_4Tq8 ──►│ SELECT …                ──►│                                 │
   │◄── 200 RUNNING, progress 10%, Retry-After: 10          │                                 │ … writes file
   │ … (polls with growing intervals)                       │◄── UPDATE SUCCEEDED, result_url│
   │ GET /v1/exports/exp_4Tq8 ──►│                             │                                 │
   │◄── 200 SUCCEEDED, "result": {"download_url": "…", "expires_at": "…"}
```

Design points:

- **The status lives in shared storage**, not in the memory of the instance that accepted the request. Any instance must be able to answer the `GET` (statelessness, Section 13.2), and the job must survive restarts. The queue and its delivery guarantees are Chapter 25's subject.
- **A clear state machine**: `QUEUED → RUNNING → SUCCEEDED | FAILED | CANCELLED`, with an error in problem-details form when `FAILED`, and optionally progress.
- **The creating `POST` is idempotent** with a key (Section 13.5), or a client that retries after a timeout starts two exports.
- **Cancellation**: `POST /v1/exports/{id}/cancellation` (or `DELETE` on the operation), honoured by the worker at safe points.
- **Polling guidance**: `Retry-After` tells clients when to ask again. Polling has a real cost: 10,000 clients polling every 2 seconds is 5,000 requests per second, almost all answering "not yet". Growing intervals, or a push mechanism, keep it bounded.

### Mechanism 2: webhooks, from the sender's side

Chapter 8 (Section 8.5) described how a **receiver** verifies a webhook: recompute an HMAC over the timestamp and raw body, compare in constant time, reject old timestamps, de-duplicate by event ID. This section builds the **sender**.

```text
 business transaction                    delivery system                         consumer endpoint
 ┌───────────────────────────┐
 │ UPDATE payments SET       │
 │   status = 'SUCCEEDED'    │
 │ INSERT INTO events        │  same transaction: the event exists if and only if the change does
 │   (evt_91, payment.       │  (the outbox pattern, Chapter 25)
 │    succeeded, data)       │
 └─────────────┬─────────────┘
               ▼
   for each subscribed endpoint: INSERT INTO deliveries (evt_91, endpoint 12, attempt 0, next_at = now())
               ▼
   delivery worker: SELECT deliveries WHERE next_at <= now() … (claim with a lease, Chapter 11, §11.7)
               │  body = canonical JSON of the event
               │  t    = now (Unix seconds)
               │  sig  = hex(HMAC-SHA256(endpoint.secret, t + "." + body))
               │  POST endpoint.url  (timeout 10 s, no redirects, SSRF checks: Section 13.14)
               │    Webhook-Id: evt_91
               │    Webhook-Signature: t=1792843200,v1=5257a869…
               ▼
   2xx within the timeout  → mark delivered
   anything else / timeout → attempt += 1; next_at = now() + backoff(attempt) with jitter;
                             after the last attempt → mark failed, notify the endpoint's owner,
                             and disable the endpoint after repeated failures
```

An example retry schedule (the numbers are a design choice, not a standard):

| Attempt | Delay after previous | Cumulative |
|---|---|---|
| 1 | immediate | 0 |
| 2 | 1 min | 1 min |
| 3 | 5 min | 6 min |
| 4 | 30 min | 36 min |
| 5 | 2 h | ~2.6 h |
| 6 | 6 h | ~8.6 h |
| 7 | 12 h | ~20.6 h |
| 8 | 24 h | ~44.6 h, then give up |

The properties that follow from this design must be written into the contract, because consumers have to cope with them:

- **At-least-once delivery.** A delivery whose `2xx` response was lost is retried, so the consumer receives the event twice. Consumers de-duplicate by event ID, ideally with a unique constraint on processed event IDs (the same guard as Section 13.5).
- **No ordering guarantee.** Retries and parallel workers mean `payment.refunded` can arrive before `payment.succeeded`. Events carry their creation time and the resource's version; consumers either ignore events older than the state they hold, or treat the event as a hint and **fetch the current state** from the API.
- **Thin versus fat events.** A **fat** event carries the resource's full data; a **thin** event carries only the type and ID (`{"type":"payment.succeeded","data":{"id":"pay_9Xz1"}}`), and the consumer fetches the resource. Thin events avoid stale or out-of-order data and leak less if a URL is misconfigured, at the cost of one more request.
- **Fast acknowledgement.** Consumers should verify the signature, store the event, return `2xx`, and process it afterwards. A consumer that does slow work before responding causes timeouts, which cause retries, which cause duplicates.
- **Secret rotation.** Each endpoint has its own secret. To rotate it without downtime, the sender signs with both the old and the new secret for a period (two `v1=` signatures in the header), and the receiver accepts either.
- **Webhook URLs are user-supplied URLs.** A customer can register `http://169.254.169.254/...` as their endpoint. The delivery system is a URL-fetching feature and needs every SSRF defence of Section 13.14.

### Guarantees and non-guarantees

A `202` plus a persistent status resource guarantees that the client can always find out what happened to the work, from any instance, after any restart. Webhooks with persistent deliveries and retries guarantee at-least-once delivery attempts within the retry window. Neither guarantees exactly-once processing, ordering, or that a consumer's endpoint is reachable; a consumer that is down longer than the retry window misses events unless it can list them through the API (offering `GET /v1/events?after=…` is the usual safety net).

### Failure and diagnosis

- **Duplicate side effects at consumers.** Evidence: the same event ID processed twice; deliveries retried after slow `2xx`. Fix: de-duplication by event ID; acknowledge fast.
- **Out-of-order effects.** A payment shown as succeeded after it was refunded. Evidence: event creation times versus processing order. Fix: version checks, or fetch current state.
- **Retry storms after a consumer outage.** When a large consumer comes back, every queued delivery fires at once and overwhelms it. Fix: jitter, per-endpoint concurrency limits.
- **Webhook queue backlog.** One dead endpoint with long timeouts occupies delivery workers. Evidence: delivery latency rising for all consumers; workers busy with timeouts to one host. Fix: per-endpoint isolation (separate queues or concurrency caps, a bulkhead, Chapter 24), short timeouts, automatic disabling.
- **Stuck operations.** Status resources that stay `RUNNING` forever because the worker died. Fix: heartbeats or leases on jobs, and a sweeper that fails or requeues expired ones (Chapter 25).

### Backend connection

Never hold an HTTP request open for long work: accept with `202`, persist a status resource, and let workers do the job. Offer webhooks for completion, built on an outbox, signed per endpoint, retried with backoff and jitter, documented as at-least-once and unordered, and protected against SSRF. Give consumers an event-listing endpoint so that a missed webhook is never a lost event.

---

## 13.11 Data Formats and Schema Evolution: JSON, Protocol Buffers, Avro

### Definitions

A **data format** (serialisation format) is the byte-level encoding of structured data (Chapter 12, Section 12.7). **Text formats** such as JSON are human-readable; **binary formats** such as Protocol Buffers and Avro are compact and fast to parse but need tools to read.

**Schema evolution** is changing a format's schema over time while data and programs written for older or newer schemas keep working. For data formats, compatibility has two directions:

- **Backward compatibility**: programs using the **new** schema can read data written with the **old** schema.
- **Forward compatibility**: programs using the **old** schema can read data written with the **new** schema.
- **Full compatibility** is both.

### Why it exists

In a distributed system, writers and readers are upgraded at different times. During a rolling deployment (Chapter 26), old and new versions of a service run side by side; messages written today sit in a queue or a log and are read next month (Chapter 25); a mobile app reads responses from a server three versions ahead. Every format change must therefore be readable in both directions for some time, or the system breaks partway through every deployment.

### Mechanism 1: JSON

JSON carries **field names** with every value, so it is self-describing: a reader can see what is there without a schema, ignore what it does not know, and notice what is missing. That makes additive evolution easy (Section 13.9) and costs space and parsing time: the names are repeated in every record. JSON has no built-in schema; JSON Schema (and OpenAPI, which uses it, Section 13.15) can describe and validate it. Its weak points for evolution are the ones from Chapter 7 (Section 7.7) and Chapter 12 (Section 12.7): numbers without a declared precision, no date type, and no distinction between absent and null unless the format defines one.

### Mechanism 2: Protocol Buffers on the wire

**Protocol Buffers** (protobuf) is a binary format in which every field is identified by a **field number** declared in a `.proto` schema. Names exist only in the schema and in generated code; the bytes carry numbers.

```proto
syntax = "proto3";
package shop.orders.v1;

message Order {
  int64  id     = 1;
  string status = 2;
}
```

Each field on the wire is a **tag** followed by a value. The tag combines the field number and a **wire type** that tells the reader how to find the value's end: `tag = (field_number << 3) | wire_type`, where wire type 0 is a **varint** (a variable-length integer: 7 bits per byte, the top bit set on every byte except the last) and wire type 2 is **length-delimited** (a varint length, then that many bytes).

```text
 Order { id: 150, status: "PAID" }

 08          tag: field 1, wire type 0 (varint)          (1 << 3) | 0 = 0x08
 96 01       150 as a varint: 0x96 = 1 0010110 → low 7 bits 0x16, "more follows"
                              0x01 = 0 0000001 → 1, last byte;  value = 0x16 + (1 << 7) = 150
 12          tag: field 2, wire type 2 (length-delimited) (2 << 3) | 2 = 0x12
 04          length 4
 50 41 49 44 "PAID"

 total: 9 bytes.   The same data as JSON, {"id":150,"status":"PAID"}, is 26 bytes.
```

The wire type is what makes evolution work. A reader that meets a tag for a field number it does not know can still compute where the field ends, skip it, and continue. Current protobuf implementations also keep such **unknown fields** and write them back out if the message is re-serialised, so a message passing through an old intermediate service does not lose new fields.

### Mechanism 3: protobuf field evolution, step by step

```proto
// v1
message Order {
  int64  id        = 1;
  string status    = 2;
  string gift_note = 3;
}

// v2: add a field, remove a field, rename a field
message Order {
  reserved 3;                                  // gift_note removed: its NUMBER can never be reused
  reserved "gift_note";                        // nor its name (protects the JSON mapping and code)
  int64  id            = 1;
  string order_status  = 2;                    // renamed: same number, so the binary wire is unchanged
  optional string coupon_code = 4;             // added: new number; "optional" tracks presence
  OrderStatus state    = 5;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;                // proto3 enums must start at 0: the default value
  ORDER_STATUS_PLACED      = 1;
  ORDER_STATUS_PAID        = 2;
}
```

What each reader sees:

```text
 v2 writer → v1 reader (forward compatibility)
   bytes: 08 96 01 | 12 04 "PAID" | 22 05 "AUT10" | 28 02
   v1 reader: field 1 ✔, field 2 ✔ (it calls it status),
              tag 0x22 = field 4, wire type 2 → unknown, skip 5 bytes,
              tag 0x28 = field 5, wire type 0 → unknown, skip the varint
   → works; the v1 reader simply does not see coupon_code or state

 v1 writer → v2 reader (backward compatibility)
   bytes: 08 96 01 | 12 04 "PAID" | 1a 0a "Ring twice"
   v2 reader: field 3 is reserved → treated as unknown and skipped;
              coupon_code absent → has_coupon_code() is false; state absent → UNSPECIFIED (0)
   → works
```

The rules that keep both directions working:

| Rule | Why |
|---|---|
| Add fields with **new** numbers | old readers skip them; new readers see defaults for old data |
| **Never reuse** a field number; mark removed ones `reserved` | an old writer still sending field 3 as a string would be read by a new reader as whatever field 3 now means: silent corruption |
| Renaming is safe for the binary format, **not** for protobuf's JSON format | the binary wire uses numbers; protobuf's JSON mapping uses names |
| Do not change a field's type | a few integer types share a wire type and parse, but values can be truncated or reinterpreted; treat it as a new field |
| Use `optional` (or wrapper types) where "not set" differs from zero | in proto3, an unset scalar reads as its default (`0`, `""`, `false`), so "discount 0" and "no discount" look the same |
| Start enums with an `…_UNSPECIFIED = 0` value | a missing or unknown enum value reads as 0, which then means "unknown", not a real state |
| Version the package (`shop.orders.v1` → `v2`) for breaking changes | a new package is a new contract, alongside the old |

### Mechanism 4: Avro and schema resolution

**Avro** is a binary format, common in data pipelines and with message brokers (Chapter 25), that takes a different approach: the bytes contain **neither names nor numbers**. Fields are written one after another in the order of the writer's schema. To decode, a reader must have the **writer's schema**, and Avro's **schema resolution** then maps the writer's fields onto the reader's schema **by name**.

```json
{ "type": "record", "name": "Order", "fields": [
    { "name": "id",          "type": "long" },
    { "name": "status",      "type": "string" },
    { "name": "coupon_code", "type": ["null", "string"], "default": null }
] }
```

- A field in the reader's schema but not in the writer's is filled from its **default**. So a new field must have a default for new readers to read old data (backward compatibility).
- A field in the writer's schema but not in the reader's is ignored. So removing a field is safe for new readers; for old readers still expecting it to read new data (forward compatibility), the removed field must have had a default in the old schema.
- Renames are done with **aliases** in the reader's schema.

Because each record needs its writer's schema, systems store schemas in a **schema registry** and prefix each message with a small schema identifier. The registry can also reject a new schema that breaks the configured compatibility mode before any producer uses it.

### Mechanism 5: compatibility modes and deployment order

| Mode | Guarantee | Typical safe changes | Who upgrades first |
|---|---|---|---|
| Backward | new readers read old data | add fields with defaults (Avro), add fields (protobuf), delete fields | **readers** (consumers) first: they can read both old and new data while producers still write old |
| Forward | old readers read new data | add fields old readers can ignore, delete fields that had defaults | **writers** (producers) first: old consumers can read what they now write |
| Full | both | add or delete fields with defaults | either order |

### Concrete: the three formats compared

| | JSON | Protocol Buffers | Avro |
|---|---|---|---|
| Encoding | text, field names in every record | binary, field numbers + wire types | binary, no names or numbers |
| Schema | optional (JSON Schema) | required at build time (`.proto`, generated code) | required at read time (writer's schema) |
| Size and speed | largest, slowest to parse | compact, fast | most compact for records |
| Unknown fields | readers can ignore | skipped by wire type, preserved | resolved away by schema |
| Field identity | name | number | name (with aliases) |
| Human-readable | yes | no (tools decode) | no |
| Typical use | public web APIs, browsers | gRPC, internal services, mobile | event streams, data pipelines, storage |

### Guarantees and non-guarantees

Following a format's evolution rules guarantees that old and new programs can read each other's data in the directions the compatibility mode promises. It does not guarantee semantic compatibility: a field whose **meaning** changes (prices now include tax) parses perfectly and is wrong. Binary formats also guarantee nothing to a reader without the schema (protobuf) or the writer's schema (Avro).

### Failure and diagnosis

- **Reused field number.** After a deployment, some messages show values in the wrong fields (a gift note appearing as a coupon code). Evidence: schema history shows a number reused; the bad messages come from old writers. Fix: `reserved`, and a linter or registry check that forbids reuse (tools such as `buf` can check protobuf changes for breaking ones).
- **Zero versus unset.** Orders with "discount 0" treated as "no discount set", or the reverse. Evidence: proto3 scalar fields without presence tracking. Fix: `optional` or wrapper types.
- **Missing defaults in Avro.** A schema change is rejected by the registry, or consumers fail to decode old records. Evidence: the error names the field without a default. Fix: add defaults; follow the deployment order for the compatibility mode.
- **Deserialisation failures partway through a rolling deploy.** Evidence: errors only while old and new versions coexist. Fix: make each change compatible in the needed direction, and deploy in the right order.

### Backend connection

For public HTTP APIs, JSON with a documented schema and tolerant readers is the norm. For internal high-volume calls and long-lived messages, binary formats with explicit schemas pay off, provided their evolution rules are enforced by tools rather than memory. Decide the compatibility mode per data stream, and let it decide whether producers or consumers deploy first.

---

## 13.12 gRPC and GraphQL

### Definitions

**gRPC** is a remote procedure call (RPC) framework. A **remote procedure call** makes a call to a function in another process look like a local function call: the caller invokes a generated **stub**, which serialises the arguments, sends them over the network, waits for the result and deserialises it. gRPC defines services and messages in Protocol Buffers and runs over HTTP/2.

**GraphQL** is a query language for APIs and a runtime that executes those queries. The server publishes a **schema** of types and fields; the client sends a query that names exactly the fields it wants, possibly across related objects; the server's **resolvers** (one function per field) fetch the data and the runtime assembles a response shaped like the query.

### Why they exist

REST over JSON is a good default for public, resource-oriented APIs, but it has costs in two situations. Between internal services making millions of calls, JSON's size and parsing cost, the lack of a compiled contract, and the absence of built-in deadlines and streaming add up; gRPC addresses those. For rich client screens that need data from many resources at once, REST forces either many round trips (**under-fetching**) or large responses with unused fields (**over-fetching**), and every new screen asks the backend for a new endpoint; GraphQL lets the client ask for exactly the graph of data it needs in one request.

### Mechanism 1: gRPC on the wire

```proto
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);                    // unary
  rpc ListOrders (ListOrdersRequest) returns (stream Order);         // server streaming
  rpc UploadItems (stream OrderItem) returns (UploadSummary);        // client streaming
  rpc TrackOrders (stream TrackRequest) returns (stream OrderEvent); // bidirectional streaming
}
```

A unary call `GetOrder` is one HTTP/2 stream (Chapter 7, Section 7.10):

```text
 HEADERS  (client → server)
   :method: POST
   :scheme: https
   :path: /shop.orders.v1.OrderService/GetOrder        ← /package.Service/Method
   :authority: orders.internal
   content-type: application/grpc
   te: trailers
   grpc-timeout: 250m                                   ← the deadline: 250 milliseconds (Chapter 10, §10.10)
   authorization: Bearer eyJ…                           ← "metadata" = ordinary headers
 DATA     00 | 00 00 00 05 | 08 96 01 …                ← length-prefixed message:
                                                          1 byte "compressed?" flag, 4-byte big-endian length,
                                                          then the protobuf bytes (Section 13.11)
          END_STREAM
 HEADERS  (server → client)   :status: 200   content-type: application/grpc
 DATA     00 | 00 00 00 09 | 08 96 01 12 04 50 41 49 44
 HEADERS  (trailers)          grpc-status: 0            ← the real outcome: 0 = OK
                              grpc-message: (empty)     END_STREAM
```

Note that the HTTP status is `200` even when the call fails: the outcome travels in the **trailers** as `grpc-status`, with gRPC's own codes:

| gRPC status | Code | Roughly the HTTP equivalent |
|---|---|---|
| `OK` | 0 | `200` |
| `INVALID_ARGUMENT` | 3 | `400` |
| `DEADLINE_EXCEEDED` | 4 | `504` (outcome unknown) |
| `NOT_FOUND` | 5 | `404` |
| `ALREADY_EXISTS` | 6 | `409` |
| `PERMISSION_DENIED` | 7 | `403` |
| `RESOURCE_EXHAUSTED` | 8 | `429` |
| `FAILED_PRECONDITION` | 9 | `409` / `412` / `422` |
| `UNAVAILABLE` | 14 | `503` (retryable) |
| `UNAUTHENTICATED` | 16 | `401` |
| `INTERNAL` | 13 | `500` |

What gRPC gives: compact binary messages; a compiled contract with generated clients in many languages; streaming in both directions on one connection; deadlines that propagate across services and cancel work when they expire; standard status codes. What it costs:

- **Browsers cannot call it directly**: browser JavaScript cannot read HTTP/2 trailers or control framing, so web clients need gRPC-Web and a translating proxy.
- **HTTP caching does not apply**: every call is a `POST` to a method path.
- **Load balancing needs care**: a client keeps one long-lived HTTP/2 connection and sends all calls over it, so a connection-level (L4) balancer pins all of a client's calls to one backend (Chapter 6, Section 6.13). gRPC needs request-level (L7) balancing or client-side balancing across backends.
- **Debugging needs tools**: the payload is binary; `curl` is not enough (tools such as `grpcurl` read the schema and decode it).

### Mechanism 2: GraphQL execution and resolvers

```graphql
type Query {
  orders(first: Int!, after: String): OrderConnection!
}
type Order {
  id: ID!
  status: String!
  customer: Customer!
  items(first: Int!): [OrderItem!]!
}
type Customer { id: ID!  name: String!  email: String! }
```

A client asks for one screen's data:

```graphql
query RecentOrders {
  orders(first: 50) {
    edges { node { id status customer { name } } }
  }
}
```

```http
POST /graphql HTTP/1.1
Content-Type: application/json

{ "query": "query RecentOrders { orders(first: 50) { edges { node { id status customer { name } } } } }" }
```

The runtime parses and validates the query against the schema, then walks it, calling a resolver for each field. Fields without a custom resolver just read the property of the parent object. The response mirrors the query:

```json
{ "data": { "orders": { "edges": [
    { "node": { "id": "ord_7Kq2", "status": "PAID", "customer": { "name": "Ana" } } },
    ...
] } } }
```

### Concrete: the N+1 problem in a resolver, and DataLoader

The obvious resolvers:

```pseudo
resolvers = {
  Query.orders:   (args)  -> db.query("SELECT * FROM orders ORDER BY created_at DESC LIMIT $1", args.first),
  Order.customer: (order) -> db.queryOne("SELECT * FROM customers WHERE id = $1", order.customer_id),
}
```

The runtime calls `Order.customer` once **per order**:

```text
 SELECT * FROM orders ORDER BY created_at DESC LIMIT 50          1 query
 SELECT * FROM customers WHERE id = 81                           ┐
 SELECT * FROM customers WHERE id = 17                           │ 50 queries, one per order
 SELECT * FROM customers WHERE id = 81      (again: same customer)│
 ...                                                              ┘
 total: 1 + N = 51 queries.  At 1 ms each that is ~51 ms of round trips for one screen,
 and a nested field (each order's items, each item's product) multiplies it again.
```

This is the **N+1 query problem**: one query for a list, then one per element. The standard fix is a **DataLoader**, a per-request batching and caching helper. A resolver asks the loader for a key instead of querying; the loader collects all keys requested during the current execution step, then issues **one** batched query:

```pseudo
customerLoader = DataLoader(batchFn = (ids) ->
    rows = db.query("SELECT * FROM customers WHERE id = ANY($1)", distinct(ids))
    return [rowsById[id] for id in ids])          // results in the same order as the keys

resolvers.Order.customer = (order, ctx) -> ctx.customerLoader.load(order.customer_id)   // returns a future
```

```text
 SELECT * FROM orders ORDER BY created_at DESC LIMIT 50
 SELECT * FROM customers WHERE id = ANY('{81,17,42,…}')        ← one query for all distinct customers
 total: 2 queries
```

The loader must be created **per request**: its cache holds data fetched under one caller's permissions, and sharing it across requests would leak one user's data to another (Chapter 11, Section 11.1). The same N+1 pattern appears in ORMs (Chapter 18) and REST response mapping (Chapter 12, Section 12.6); GraphQL makes it the default unless loaders are used.

### Mechanism 3: query cost, caching and errors in GraphQL

**Query cost.** Clients choose the shape of the work, so one request can ask for an enormous amount:

```graphql
{ orders(first: 50) { items(first: 20) { product { reviews(first: 10) { author { name } } } } } }
```

Up to 50 orders × 20 items × 10 reviews = 10,000 reviews (plus their authors) from a request a few hundred bytes long. Defences: require pagination arguments on every list; reject queries above a maximum **depth**; compute a **cost** before execution (for example, multiply list sizes: 50 + 50×20 + 50×20×10 = 11,050 objects) and reject or rate-limit by cost rather than by request count; set execution timeouts; and for first-party clients, use **persisted queries**: the client sends the ID of a query registered at build time, and the server refuses arbitrary query text.

**Caching.** All queries go to one URL, usually by `POST`, so HTTP caches and CDNs (Chapter 7, Section 7.12) see nothing cacheable. GraphQL clients keep their own normalised caches keyed by type and ID. Persisted queries sent as `GET /graphql?id=…&variables=…` restore some HTTP caching.

**Errors.** A GraphQL response can contain both `data` and an `errors` array: some fields resolved, others failed. Responses commonly use status `200` even with errors, so monitoring based on HTTP status sees no failures. Errors must be counted from the response body.

**Authorisation** must be enforced in resolvers (or a layer they call) for every field and object, because a single query can reach any part of the graph (Chapter 20).

### Concrete: REST, gRPC and GraphQL compared by mechanism

| | REST (HTTP + JSON) | gRPC | GraphQL |
|---|---|---|---|
| Transport | HTTP/1.1, 2 or 3; any method | HTTP/2 (`POST` to `/Service/Method`) | HTTP, usually `POST /graphql` |
| Contract | OpenAPI (optional) | `.proto`, mandatory, compiled | GraphQL schema in its schema definition language (SDL), mandatory |
| Payload | JSON text | protobuf binary | JSON text, shaped by the query |
| Who decides response shape | server, per endpoint | server, per method | client, per query |
| Typing and code generation | optional | built in | built in (schema introspection) |
| HTTP caching / CDN | yes, for `GET` | no | weak (persisted `GET` queries help) |
| Streaming | SSE, WebSockets, chunked (Chapter 10, Section 10.9) | native, both directions | subscriptions (usually over WebSockets) |
| Error model | status codes + problem details | `grpc-status` in trailers | `errors` array, often with `200` |
| Deadlines and cancellation | by convention (headers, timeouts) | built in (`grpc-timeout`) | by convention |
| Browser support | native | needs gRPC-Web proxy | native |
| Debugging | `curl`, browser | needs schema-aware tools | GraphQL tools; one endpoint obscures logs |
| Evolution | additive JSON, versions | field numbers, `reserved` | add fields; deprecate fields with `@deprecated`; no versions |
| Typical risk | over- and under-fetching; many endpoints | load-balancing pinning; browser gap | N+1, expensive queries, field-level authorisation |
| Typical fit | public APIs, resource CRUD, cacheable reads | internal service-to-service, streaming, polyglot microservices | aggregating many sources for varied client screens (often a backend-for-frontend) |

Many systems use all three: REST for the public API, gRPC between internal services, and GraphQL as an aggregation layer for their own front ends (a **backend-for-frontend**: a server-side layer built for one client application, which composes the calls that client needs).

### Guarantees and non-guarantees

gRPC guarantees a typed contract, efficient encoding, and deadline propagation when every service honours it. GraphQL guarantees that responses match the requested shape and the schema's types. Neither guarantees performance: gRPC can still be pinned to one backend, and GraphQL can still run 10,000 queries for one request. Neither replaces idempotency, pagination, authorisation or versioning discipline; they move those problems, they do not remove them.

### Failure and diagnosis

- **One gRPC backend overloaded, others idle.** Evidence: per-backend request rates; one long-lived HTTP/2 connection per client. Fix: L7 or client-side load balancing; periodic connection recycling.
- **GraphQL endpoint slow under modest traffic.** Evidence: database query counts per GraphQL request in the hundreds; traces showing many identical small queries. Fix: DataLoaders; query cost limits.
- **GraphQL errors invisible to monitoring.** Evidence: 100% `200` responses while users report failures. Fix: metrics from the `errors` array.
- **`DEADLINE_EXCEEDED` treated as failure.** The operation may have completed (the outcome is unknown, as for any timeout). Fix: idempotent methods or keys for retried calls.

### Backend connection

Choose by mechanism. Use REST where caching, broad client support and simplicity matter; gRPC where many internal calls need efficiency, typed contracts, streaming and deadlines; GraphQL where varied clients need to compose data from many sources. Each brings its own operational work: L7 load balancing for gRPC, loaders and cost limits for GraphQL.

---

## 13.13 The API Security Surface

### Definitions

An API's **attack surface** is every input an attacker can send and every behaviour they can trigger through it: endpoints, parameters, headers, bodies, files, and the URLs it fetches.

**Authentication** establishes who is calling; **authorisation** decides whether that caller may perform this operation on this object. Both are taught in depth in Chapter 20; this section decides **where** they sit in an API.

**Object-level authorisation** checks access to the specific object named in the request (may customer 81 read order `ord_9Bc3`?). Its failure, **Broken Object Level Authorization** (BOLA), is first in OWASP's API Security Top 10 (2023 edition); Chapter 20 treats it under its other common name, IDOR (insecure direct object reference).

**Rate limiting** caps how many requests (or how much work) a caller may consume in a period.

### Why it exists

An API exposes business operations directly to programs, including attackers' programs. There is no user interface to hide fields or disable buttons, and attackers can send any request in any order at any rate. The most common API breaches are not cryptographic: they are a missing ownership check on one endpoint, a field that should not have been writable, or an endpoint that returns far more data than any screen shows.

### Mechanism 1: where each check belongs

```text
 request ──► EDGE (CDN / WAF / load balancer, Chapter 10)
             TLS, size limits, coarse per-IP rate limits, known-bad patterns
          ──► GATEWAY (Chapter 10, Section 10.4)
             authenticate the token or key; coarse scopes ("orders:read"); per-key rate limits and quotas
          ──► SERVICE MIDDLEWARE
             re-validate identity (do not trust a header anyone inside the network could set);
             build the request context: caller, tenant, scopes (Chapter 11, Section 11.6)
          ──► CONTROLLER: shape validation, request DTO (only client-settable fields)
          ──► SERVICE / USE CASE: OBJECT-LEVEL and FUNCTION-LEVEL authorisation
             "may this caller cancel THIS order?" "may this caller call refund at all?"
          ──► REPOSITORY: tenant and owner scoping in the query itself
          ──► RESPONSE DTO: only fields this caller may see
```

Object-level checks cannot live at the gateway: the gateway knows the caller and the URL, but not who owns `ord_9Bc3`. They belong where the object is loaded, and the strongest form puts the ownership condition into the query, so that an object the caller may not see is simply not found:

```pseudo
// Vulnerable: loads any order by ID, checks nothing
function getOrder(caller, orderId):
    return orders.findById(orderId)

// Safer: the scope is part of the query; a stranger's order is indistinguishable from a missing one
function getOrder(caller, orderId):
    order = db.queryOne("SELECT … FROM orders WHERE id = $1 AND customer_id = $2", orderId, caller.customerId)
    if order is null: return 404                // not 403: do not confirm that the order exists
    return order
```

### Mechanism 2: input limits as security controls

Validation (Chapter 12, Sections 12.3 and 12.6) is also a defence against resource exhaustion, which OWASP lists as "unrestricted resource consumption":

| Limit | Example | Protects against |
|---|---|---|
| Body size | 1 MB for JSON endpoints; uploads via a separate path | memory exhaustion, slow parsing |
| Array lengths | at most 100 items per order | huge loops and queries |
| String lengths | names ≤ 200 characters | storage abuse, slow processing |
| Page size | `limit` ≤ 100 (Section 13.6) | expensive queries |
| Nesting depth and query cost | GraphQL depth and cost (Section 13.12); JSON depth (Chapter 12, Section 12.7) | parser and resolver blow-ups |
| File types and sizes | images only, ≤ 10 MB, checked by content | malicious files, storage cost |
| Expensive operations per caller | exports per hour, password-reset emails per hour | abuse of sensitive business flows |

### Mechanism 3: rate limiting from the client's point of view

Rate limits protect capacity, share it fairly between callers, cap costs, and slow brute-force attempts. The algorithms (token bucket, sliding windows) are Chapter 24's subject; the API contract is how limits are communicated:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1792843260
Content-Type: application/problem+json

{ "type": "https://api.shop.example/problems/rate-limited", "title": "Too many requests.", "status": 429 }
```

`429 Too Many Requests` (RFC 6585) and `Retry-After` (RFC 9110) are standard. The `X-RateLimit-*` headers are a widespread convention, not a standard, and their exact meanings differ between APIs (`Reset` as seconds remaining or as a Unix time). At the time of writing, the IETF is standardising `RateLimit` and `RateLimit-Policy` header fields to replace them. Whatever is used, document it, send limit information on successful responses too so clients can slow down before hitting the limit, and decide what the limit is keyed on: the API key, the user, the tenant, or the IP address (which punishes everyone behind one NAT, Chapter 6, Section 6.5).

### Mechanism 4: the OWASP API Security Top 10 as a checklist

| 2023 category | Where this book addresses it |
|---|---|
| API1 Broken Object Level Authorization | Mechanism 1 above; Chapter 20 |
| API2 Broken Authentication | Chapter 20 |
| API3 Broken Object Property Level Authorization (mass assignment, excessive data exposure) | Chapter 12, Section 12.3; Section 13.4 |
| API4 Unrestricted Resource Consumption | Mechanisms 2 and 3; Section 13.6; Chapter 24 |
| API5 Broken Function Level Authorization (calling admin functions as a user) | Mechanism 1; Chapter 20 |
| API6 Unrestricted Access to Sensitive Business Flows (bots buying all stock) | per-flow limits; Chapter 24 |
| API7 Server Side Request Forgery | Section 13.14 |
| API8 Security Misconfiguration (CORS, verbose errors, missing headers) | Chapter 9; Section 13.8 |
| API9 Improper Inventory Management (forgotten old versions and test endpoints) | Section 13.9; Section 13.15 |
| API10 Unsafe Consumption of APIs (trusting data from third-party APIs) | Section 13.14; Chapter 12, Section 12.7 |

### Guarantees and non-guarantees

Layered checks guarantee that each request is authenticated before reaching business code, that object access is checked where the object is known, and that resource use is bounded per caller. They do not guarantee that every endpoint has the object check (one forgotten endpoint is enough), that internal callers are trustworthy (Chapter 20 covers service identity), or that limits keyed on IP addresses identify attackers. CORS is not an access control mechanism (Chapter 9, Section 9.7): it only tells browsers which pages may read responses.

### Failure and diagnosis

- **BOLA.** A customer reads or changes another customer's order by changing an ID. Evidence: access logs showing one caller requesting many sequential or foreign IDs, with `200` responses. Prevention: scoping in queries, automated tests that call every endpoint with another tenant's IDs.
- **Function-level gaps.** An admin endpoint reachable with a customer token because the route was added without a role check. Evidence: an inventory of routes and their required roles, compared with the code. Fix: deny by default; checks declared per route and tested.
- **Resource exhaustion.** A single key consumes most of the capacity. Evidence: per-key request and cost metrics. Fix: limits per key and per expensive operation.
- **Forgotten endpoints.** An old `v0` or debug endpoint without new checks. Evidence: traffic to undocumented paths in access logs. Fix: an inventory generated from the gateway's configuration and the API specification (Section 13.15).

### Backend connection

Authenticate at the edge and again in the service, authorise every object where it is loaded (ideally inside the query), expose only fields the caller may set and see, bound every input, and publish rate limits with `429` and `Retry-After`. Review each new endpoint against the OWASP API list.

---

## 13.14 Outbound Calls: HTTP Clients, Pools, Timeouts, and SSRF

### Definitions

An **HTTP client** is the component a service uses to call other APIs. It manages a **connection pool** per destination (reusing kept-alive connections, Chapter 7, Section 7.9), timeouts, retries, TLS and redirects.

**Server-side request forgery (SSRF)** is a vulnerability in which an attacker makes a server send a request to a destination of the attacker's choosing. The request comes from the server's network position, with the server's access, so it can reach things the attacker cannot reach directly: internal services, administrative interfaces, databases' HTTP ports, and the cloud **instance metadata service** (Chapter 6, Section 6.4), which can hand out the server's cloud credentials.

### Why it exists

Every backend calls other systems: payment providers, other internal services, webhooks to customers, URLs that users supply (images to import, link previews, file imports). Each call is a dependency that can be slow, fail, or return an unknown outcome, and each user-supplied URL is an instruction from a stranger to the server's network stack. Chapter 11 (Lab 2) showed what a client without a timeout does to a whole service; SSRF shows what a client without destination checks does to the whole network.

### Mechanism 1: a well-configured client

```pseudo
paymentsClient = HttpClient(                               // ONE instance, shared (Chapter 12, Section 12.4)
    baseUrl              = config.payments.url,
    maxConnectionsPerHost = 50,                            // pool sized by Little's Law (Chapter 11, Section 11.3)
    poolAcquireTimeout   = 200ms,                          // fail fast when the pool is exhausted
    connectTimeout       = 500ms,                          // TCP + TLS establishment
    responseTimeout      = 2s,                             // first byte of the response
    totalTimeout         = min(3s, remainingDeadline()),   // never beyond the caller's deadline (Ch 10, §10.10)
    idleTimeout          = 50s,                            // shorter than the server's keep-alive timeout
    followRedirects      = false,
    maxResponseBytes     = 1 MB,
    tlsVerification      = ON                              // never disabled (Chapter 8, Section 8.12)
)

function charge(payment, idemKey):
    return withRetries(maxAttempts = 3, backoff = exponentialWithJitter(100ms),     // Chapter 24
                       retryOn = [connectFailed, 502, 503, 504, timeout],
                       onlyIf = idempotent or idemKey present) {
        paymentsClient.post("/charges", body = payment, headers = {"Idempotency-Key": idemKey})
    }
```

The common mistakes each have a recognisable signature:

| Mistake | What happens | Evidence |
|---|---|---|
| No timeouts | a slow dependency holds every worker forever (Chapter 11, Lab 2) | thread dumps all in socket reads |
| A new client per request | no connection reuse: a TCP and TLS handshake per call, thousands of sockets in `TIME_WAIT`, ephemeral port exhaustion (Chapter 6, Section 6.11) | `ss` shows huge `TIME_WAIT` counts; latency includes handshakes |
| Pool too small | requests wait for connections | pool-acquire waits in metrics and thread dumps |
| Idle timeout longer than the server's | requests sent on connections the server just closed (the keep-alive race, Chapter 7, Section 7.9) | sporadic `ECONNRESET` on the first request after idle periods |
| Retrying non-idempotent calls without keys | duplicates | duplicate records downstream |
| Retrying at every layer | multiplied load during incidents (Chapter 10, Section 10.4) | request rates to the failing service multiply |
| DNS cached forever | after a failover, the client keeps calling the old address | connection errors to an IP the name no longer resolves to; some runtimes cache DNS longer than the record's TTL, so check yours |

### Mechanism 2: why a user-supplied URL is dangerous

The import feature above calls `httpClient.get(url)` on whatever URL the user sends. The request then leaves from inside the server's network, with the server's reachability and credentials, not the attacker's. That is the whole problem: the network trusts the server in ways it does not trust the public internet.

Three categories of target are reachable from the server but not from an outsider:

- **Loopback and internal services.** `http://localhost:…` and private addresses (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, Chapter 6, Section 6.5) reach admin endpoints, internal search clusters, databases with HTTP interfaces, and other services that were never meant to face the internet.
- **The cloud instance metadata service** at the link-local address `169.254.169.254` (Chapter 6, Section 6.4). On a cloud virtual machine it can return the machine's temporary cloud credentials. An attacker who reaches it through a forged request obtains those credentials and can then act as the server from anywhere. A widely reported 2019 breach at Capital One involved a request forged through a misconfigured component to the metadata service, exposing credentials.
- **Internal-only state changes.** Even **blind SSRF**, where the attacker never sees the response, lets them map which internal ports are open (by how long a request takes) and trigger internal endpoints that change state.

A feature is vulnerable whenever untrusted input decides the destination of an outbound request: an obvious URL field, but also a webhook URL a customer registers (Section 13.10), a "fetch my profile picture from this link" field, an XML document with an external entity (Chapter 12, Section 12.7), or a redirect the server follows.

### Mechanism 3: why naive filtering fails

The instinct is to block the dangerous destinations by string matching ("reject any URL containing `169.254.169.254` or `localhost`"). This does not work, because the same destination has many spellings and because the name checked is not always the address connected to:

- **One address, many forms.** An IPv4 address can be written as a single decimal number, in hexadecimal, or as an IPv4-mapped IPv6 address, all resolving to the same host. A blocklist of literal strings misses every alternate form.
- **Names the attacker controls.** A hostname the attacker owns can resolve to an internal address, so the URL looks harmless but points inward.
- **Redirects.** An allowed public URL can respond with a `3xx` redirect to an internal address; a client that follows redirects then connects inward.
- **Time-of-check to time-of-use (DNS rebinding).** A name resolves to a safe public address when the server validates it, and to an internal address a moment later when the server actually connects. The check and the connection used different answers (the TOCTOU race of Chapter 5, Section 5.2).

The lesson is that filtering must act on the **IP address the client is really about to connect to**, after resolving names, and must be rechecked at connect time, not on the URL string.

### Mechanism 4: defences, in layers

1. **Avoid fetching arbitrary URLs.** The strongest defence is to not have the feature in this form: let users upload a file directly, or accept only URLs whose host is on an **allowlist** of known providers. An allowlist of permitted hosts is far more robust than a blocklist of forbidden ones.
2. **Restrict the scheme and port.** Permit only `https` (and `http` if unavoidable) and only standard web ports; reject `file:`, `gopher:`, `ftp:` and the rest, which exist only to reach non-web targets.
3. **Resolve the name yourself, and check the address.** Resolve the host to its IP addresses, and reject the request if any resolved address is loopback, private, link-local (including `169.254.169.254`), or otherwise internal. Then **connect to the address you checked**, not by passing the hostname to the client again, so that a second DNS lookup cannot return a different answer (this closes the rebinding race).
4. **Do not follow redirects**, or re-run every check on each redirect target. A redirect is a fresh destination chosen by the remote server.
5. **Do not echo the fetched content back** in responses or error messages. Returning the fetched body (or a detailed error containing it) turns blind SSRF into a window onto internal services. Validate that the content is what was expected (an image of an allowed type and size) and return only a generic result.
6. **Reduce what a forged request can reach.** Give the fetcher a network path with egress restricted to the destinations it legitimately needs (deny-by-default outbound rules, Chapter 12, Section 12.7). On clouds, require the hardened metadata service (for example AWS's IMDSv2, which requires a session token obtained with a `PUT` that simple SSRF cannot perform) so that a forged `GET` cannot read credentials. Run the fetch with the least privilege possible.

```pseudo
function safeFetch(rawUrl):
    url = parse(rawUrl)
    require url.scheme in {"https"}                         else reject("scheme")
    require url.port in {443, null}                          else reject("port")
    addresses = dnsResolve(url.host)                         // may return several
    for a in addresses:
        require not isPrivateOrLoopbackOrLinkLocal(a)        else reject("internal address")
    chosen = addresses[0]
    return httpClient.getByIp(chosen, hostHeader = url.host, // connect to the checked IP, not the name
                              followRedirects = false, maxBytes = 10 MB, timeout = 3s)
```

The webhook delivery system of Section 13.10 is exactly this kind of feature, because customers choose the destination URL, so it needs every one of these defences.

### Guarantees and non-guarantees

A correctly configured client guarantees bounded waiting and connection reuse for outbound calls; the SSRF defences guarantee that a fetch cannot reach internal addresses **if** the check is on the resolved address, rechecked at connect time, with redirects controlled. They do not guarantee safety if any layer is missed: following one redirect, resolving the name twice, or echoing the body each reopens the hole. Timeouts do not make a retried non-idempotent call safe (the outcome is still unknown).

### Failure and diagnosis

- **A slow dependency stalls the service.** Missing timeouts. Evidence: thread dumps in socket reads to one host (Chapter 11, Lab 2). Fix: connect, response and total timeouts on every client.
- **Connection and port exhaustion.** A new client per request, or no pooling. Evidence: many sockets in `TIME_WAIT` (`ss`, Chapter 6, Section 6.11); handshakes in every call's latency. Fix: one shared, pooled client.
- **SSRF indicators.** Outbound requests to `169.254.169.254`, loopback or private addresses; URL-input fields receiving internal addresses or odd encodings; the fetcher making unexpected connections. Evidence: egress logs, the destinations in fetch logs. Fix: the layered defences above; alert on any fetch aimed at an internal range.
- **Credential misuse after an SSRF.** Cloud credentials belonging to a server used from an unfamiliar location. Evidence: cloud audit logs showing the role used from outside. Response: rotate the credentials, require the hardened metadata service, restrict egress.

### Backend connection

Treat every outbound call as a dependency that needs timeouts, a shared pool, controlled retries and a deadline, and treat every user-influenced URL as an instruction to your network. Prefer uploads or allowlists; when you must fetch, validate the resolved address, connect to it directly, refuse redirects and internal ranges, never echo the body, and restrict the fetcher's egress and privileges.

---

## 13.15 Documentation and Contracts: OpenAPI and Contract Testing

### Definitions

An **API description** (or specification) is a machine-readable document that states an API's endpoints, parameters, request and response schemas, status codes and security. **OpenAPI** is the dominant such format for HTTP APIs (its 2.0 version was known as Swagger). The equivalent for gRPC is the `.proto` file, and for GraphQL the schema document.

A **contract test** checks that an implementation matches a contract. A **provider contract test** checks that the server conforms to its own specification. A **consumer-driven contract test** records what a particular consumer actually relies on and verifies the provider against that (Chapter 12, Section 12.12).

### Why it exists

A contract that lives only in prose drifts from the implementation: the documentation says one thing, the server does another, and consumers trust whichever bites them. A machine-readable description can generate documentation, client libraries, server stubs, mock servers, request validation and test cases from one source, and contract tests make "the documentation is accurate" a build-time check rather than a hope.

### Mechanism: OpenAPI, and where it plugs in

```yaml
openapi: 3.1.0
info: { title: Shop API, version: "2.3.0" }
paths:
  /v1/orders/{orderId}:
    get:
      operationId: getOrder
      parameters:
        - { name: orderId, in: path, required: true, schema: { type: string } }
      responses:
        "200":
          description: The order
          headers: { ETag: { schema: { type: string } } }
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Order" }
        "404":
          content:
            application/problem+json:
              schema: { $ref: "#/components/schemas/Problem" }
components:
  schemas:
    Order:
      type: object
      required: [id, status, total, version]
      properties:
        id:      { type: string, example: "ord_7Kq2" }
        status:  { type: string, enum: [PLACED, PAID, SHIPPED, DELIVERED, CANCELLED] }
        total:   { $ref: "#/components/schemas/Money" }
        version: { type: integer }
```

One OpenAPI document drives many tools:

```text
 OpenAPI (one source of truth)
   ├─► rendered reference documentation (browsable, with "try it" consoles)
   ├─► generated client libraries in several languages
   ├─► generated server stubs / request validation at the edge
   ├─► a mock server (consumers build against it before the real API exists)
   ├─► provider contract tests (every example and schema checked against the running API)
   └─► an inventory of endpoints (compared with the gateway's routes to find undocumented ones, API9)
```

Two ways of working exist: **design-first**, where the specification is written and reviewed before code (and the server is checked against it), and **code-first**, where the specification is generated from annotations in the code. Design-first keeps the contract a deliberate decision; code-first keeps it close to the code but risks documenting whatever the code happens to do, including mistakes.

### Mechanism: contract testing

A specification can be wrong, and an implementation can diverge from it. Two kinds of test guard the two sides:

- **Provider verification.** The build runs the real API against its own specification: every documented response schema, status and example is checked. This catches the day someone adds a field to the code but not the spec, or changes a status code.
- **Consumer-driven contracts.** Each consumer's test suite records the requests it makes and the parts of the responses it depends on, producing a contract. The provider's build replays those contracts against itself. If a provider change would break a real consumer, the provider's build fails before release, not the consumer's at runtime. Tools such as Pact implement this; Chapter 12 (Section 12.12) introduced the idea. This is what makes "non-breaking change" (Section 13.9) a checkable claim rather than a judgement.

### Guarantees and non-guarantees

A machine-readable specification with provider verification guarantees that the documented contract and the implementation agree on structure, status codes and schemas. Consumer-driven contracts guarantee that changes do not break the consumers whose expectations are recorded. Neither guarantees that undocumented behaviour is safe to change (Hyrum's Law again), that consumers without recorded contracts still work, or that the semantics behind the schema are correct (a correctly typed field with the wrong meaning passes every test).

### Failure and diagnosis

- **Documentation drift.** The spec and the API disagree. Evidence: provider verification failures, or consumers reporting responses that differ from the docs. Fix: generate one from the other, or verify one against the other in the build.
- **Undocumented or forgotten endpoints.** Routes exist that the spec does not mention (API9). Evidence: compare the gateway's route table with the specification. Fix: remove them or document and secure them.
- **A "safe" change breaks a consumer.** No consumer contract covered the behaviour it relied on. Fix: add that consumer's contract; broaden the recorded expectations.

### Backend connection

Keep one machine-readable description of the API and make it authoritative: generate documentation, clients and mocks from it, verify the running API against it in the build, and run consumer-driven contract tests so that compatibility is enforced automatically. A contract that is tested is a contract you can rely on when you evolve the API.

---

## 13.16 Placement-Round Topics: SOAP, WSDL and RPC Styles

> **Placement-round topics.** This section is for campus placement rounds, university exams and written tests, where "REST versus SOAP" and "what is a web service?" are asked often. Product-company backend interviews rarely ask them. **Give your main focus to Sections 13.1–13.15**; read this before placement rounds and come back to it for revision.

### Definitions

A **web service** is any software service offered to other programs over a network using web standards. REST APIs, SOAP services and gRPC services are all web services in this broad sense.

**SOAP** (originally "Simple Object Access Protocol") is an XML-based messaging protocol standardised by the W3C. Every message is an XML **envelope** containing an optional **header** (for metadata such as security tokens and transaction context) and a **body** (the operation and its data). Errors are returned as a **fault** element inside the body. SOAP is usually carried over HTTP `POST`, but is defined independently of the transport.

**WSDL** (Web Services Description Language) is an XML document that describes a SOAP service: its operations, the messages each takes and returns, the data types (in XML Schema), and the address and binding. Tools generate client stubs from it, much as gRPC tools do from a `.proto` file (Section 13.12).

**RPC styles over HTTP** send a procedure name and parameters rather than acting on resources. **XML-RPC** and its successor in spirit, **JSON-RPC** (version 2.0 is the common one), encode a call as a small document:

```json
{ "jsonrpc": "2.0", "method": "orders.cancel", "params": { "order_id": "ord_7Kq2" }, "id": 17 }
```

The response carries a `result` or an `error` object with the same `id`. A request without an `id` is a **notification** that expects no response, and several calls can be sent as one batch array.

### Concrete: one operation in SOAP and in REST

```xml
POST /OrderService HTTP/1.1
Content-Type: application/soap+xml; charset=utf-8

<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope" xmlns:o="urn:shop:orders">
  <soap:Header> ... security token ... </soap:Header>
  <soap:Body>
    <o:GetOrder><o:OrderId>ord_7Kq2</o:OrderId></o:GetOrder>
  </soap:Body>
</soap:Envelope>
```

```http
GET /v1/orders/ord_7Kq2 HTTP/1.1
Accept: application/json
```

The SOAP call is a `POST` to one endpoint with the operation named inside the XML: level 0 of the Richardson model (Section 13.2). HTTP sees every call as the same unsafe, uncacheable request. The REST call names the resource in the URL and uses `GET`, so caches, retries and monitoring understand it.

### REST versus SOAP

| | REST | SOAP |
|---|---|---|
| Nature | an architectural style (constraints) | a protocol with a formal specification |
| Message format | any media type; usually JSON | XML envelopes only |
| Contract | optional (OpenAPI, Section 13.15) | WSDL, typically mandatory |
| Use of HTTP | methods and status codes carry meaning | mostly `POST` to one endpoint; errors as faults |
| Caching | `GET` responses cacheable by intermediaries | not cacheable by HTTP caches |
| Standards for security and reliability | uses HTTP and TLS mechanisms (Chapters 8 and 20) | a family of WS-* standards (for example WS-Security for message-level signing and encryption) |
| Weight | light; easy to call from browsers and scripts | verbose; needs SOAP tooling |
| Where you still meet it | most modern public and internal APIs | banking, payments, telecoms, government and enterprise integrations built before REST became common |

### Guarantees and non-guarantees

SOAP with WSDL guarantees a formal, machine-checked contract and, with WS-Security, protection of the message itself even across intermediaries. It does not give the HTTP ecosystem's benefits (caching, method semantics, simple tooling), and it does not by itself make operations idempotent or retries safe. REST guarantees none of SOAP's formality unless you add a specification, and neither style removes the need for the idempotency, pagination and error-design work of this chapter.

### Backend connection

You will mostly build REST, gRPC or GraphQL. You may still have to call a SOAP service from a bank or a government system; treat it as an outbound integration behind an adapter (Chapter 12, Section 12.10), generate the client from its WSDL, and apply the same timeouts, idempotency and error translation as for any other dependency (Section 13.14).

### Drill for this section

**C1. REST versus SOAP: what is the difference, and when would you still use SOAP?**
A strong answer must mention:
- REST is an architectural style using HTTP's methods, status codes and caching on resources; SOAP is an XML messaging protocol with envelopes, headers, bodies and faults.
- SOAP services are described by WSDL, usually called with `POST` to one endpoint, and are not cacheable by HTTP caches.
- SOAP's WS-* standards offer message-level security and formal contracts.
- You use SOAP mainly when integrating with an existing system that offers only SOAP.
*Common weak answer:* "SOAP is old and REST is new." True as history, but it explains neither the mechanism nor when SOAP is still the right call.

**C2. What is WSDL, and what is its equivalent in modern APIs?**
A strong answer must mention:
- An XML description of a SOAP service's operations, messages, types and endpoint address.
- Used to generate client stubs and validate messages.
- Equivalents: OpenAPI for HTTP APIs, `.proto` files for gRPC, the GraphQL schema.
*Common weak answer:* "It is SOAP's documentation." It is a machine-readable contract that tools generate code from, not prose.

---

## Chapter Summary

An API is a contract with programs you cannot see or recompile, so the overriding goal is to make promises you can keep while the system is retried, edited concurrently, evolved and attacked. The **contract** is everything consumers can observe, not just what is documented (Hyrum's Law), so you make it explicit and stay strict about what you accept while tolerant of harmless additions in what you read.

**REST** is a set of constraints: stateless requests so any instance can serve them, cacheable reads, a uniform interface of resources, methods and status codes, and hypermedia where it removes guesswork. Most real APIs are "level 2": resources plus methods, with links such as `Location` and pagination cursors rather than full HATEOAS. **Resource modelling** turns a domain into stable, opaquely identified resources, makes things with their own lifecycle (payments, refunds, shipments) into resources, keeps nesting shallow, and writes the endpoint table, with idempotency and errors, before any code.

The hard parts each have a standard answer. Because a **timeout leaves the outcome unknown**, unsafe operations take an **idempotency key**: a client generates one per operation, reuses it on every retry, and the server guards it with a database unique constraint, fingerprints the request, returns `409` while in progress and the original response afterward, passes keys downstream, and keeps a business-level unique constraint as the last guard. **Pagination** uses keyset cursors rather than offsets, because offset cost grows with depth and offsets skip and duplicate rows as the data changes. **Concurrency control** uses ETags and `If-Match` with the version compared inside the `UPDATE`, turning lost updates into visible `412`s. **Errors** use RFC 9457 problem details with a documented catalogue of `type`s, all validation errors at once with JSON Pointers, a request ID everywhere, and no internals.

**Evolution** is additive wherever possible, with a breaking-versus-non-breaking table, expand-and-contract for renames, visible versioning (usually in the path) when a break is unavoidable, and deprecation with `Sunset`, brownouts and `410 Gone`. **Long-running work** returns `202` and a status resource in shared storage; **webhooks** are built on an outbox, signed per endpoint, retried with backoff and jitter, and documented as at-least-once and unordered, with an event-listing endpoint as a safety net. **Schema evolution** depends on the format: JSON carries names and tolerates additions; **Protocol Buffers** identify fields by number, so numbers are never reused and removed ones are reserved; **Avro** matches by name against the writer's schema, so new fields need defaults. Backward and forward compatibility decide whether consumers or producers deploy first.

**gRPC** (HTTP/2, protobuf, streaming, deadlines, generated clients) suits internal service-to-service calls but cannot be called directly from browsers, bypasses HTTP caching, and pins to one backend without L7 balancing. **GraphQL** lets clients fetch exactly the graph they need, at the cost of the **N+1** resolver problem (fixed by per-request DataLoaders), uncontrolled query cost (fixed by depth and cost limits and persisted queries), weak caching, and field-level authorisation. The **security surface** needs authentication at the edge and in the service, object-level authorisation where the object is loaded (ideally scoped in the query), request DTOs and response DTOs to block mass assignment and over-exposure, limits on every input, and published rate limits with `429` and `Retry-After`; the OWASP API Top 10 is the checklist. **Outbound calls** need shared pooled clients, timeouts, controlled retries and deadlines, and any feature that fetches a user-supplied URL needs SSRF defences: allowlists, scheme and address checks on the resolved IP, no redirects to internal ranges, no echoed bodies, and restricted egress. Finally, a machine-readable **OpenAPI** description, verified against the running API and backed by consumer-driven **contract tests**, makes the contract something you can evolve with confidence.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Written contract | What the provider promised, and must not change without a version | That consumers depend only on it (Hyrum's Law), or that the implementation matches it |
| REST constraints | Stateless scaling, cacheable reads, uniform tooling | A good design: naming, pagination, errors, versioning are extra |
| Idempotency key (durable, unique-constrained, fingerprinted) | Retries of one operation with the same key and body apply at most once and replay the result | Exactly-once across different keys; safety past expiry; downstream without its own idempotency |
| Keyset pagination | Each row returned once, at constant cost per page, for rows whose sort key is stable | A consistent snapshot; jumping to an arbitrary page |
| ETag + If-Match, version in the UPDATE | No write on top of an unseen version; lost updates become `412`/`409` | Merging; progress under contention; invariants across resources |
| Problem details + catalogue | A stable identifier, status and parse path per documented error | Stable `detail` text; graceful handling of new types unless clients were told |
| Additive evolution + versioning | A supported version's contract will not break | That "non-breaking" breaks nobody; cheap maintenance of many versions |
| 202 + status resource | The client can always find the outcome, from any instance | Completion by any deadline |
| Signed webhooks with retries | At-least-once delivery attempts within the window; authenticity | Exactly-once processing; ordering; reachability of the consumer |
| Protobuf/Avro evolution rules | Old and new programs read each other's data in the promised direction | Semantic compatibility; anything without the schema |
| gRPC | Typed contract, efficient encoding, propagated deadlines | Browser access; HTTP caching; even load without L7 balancing |
| GraphQL | Responses match the query and schema | Performance (N+1, query cost); field-level authorisation; HTTP caching |
| Layered authN/authZ + limits | Authenticated requests; object checks where the object is known; bounded resource use | That every endpoint has the check; trustworthy internal callers |
| SSRF defences (resolved-IP check, no redirects, allowlist) | A fetch cannot reach internal addresses, if every layer holds | Safety if one layer is skipped |
| OpenAPI + contract tests | Spec and implementation agree; recorded consumers keep working | Safety of undocumented behaviour; consumers without contracts; correct semantics |

---

## Interview Drill

**Q1. What is an API contract, and what counts as a breaking change?**
A strong answer must mention:
- Everything consumers can rely on: endpoints, formats, field meanings, status codes, errors, ordering, limits, idempotency, and observable behaviour (Hyrum's Law).
- Non-breaking: new endpoints, new optional request fields, new response fields and enum values **if** the contract told readers to tolerate them.
- Breaking: removing or renaming fields, changing a type or meaning or units, new required fields, tightened validation, changed status codes or default sorting.
- A changed meaning (cents to euros) is the worst kind: nothing errors, values are silently wrong.
*Common weak answer:* "A breaking change is anything that returns an error." Misses silent semantic breaks, which cause no errors.

**Q2. What makes an API RESTful? Is HATEOAS used in practice?**
A strong answer must mention:
- The constraints: client–server, stateless, cacheable, uniform interface, layered system, optional code on demand.
- Stateless means no per-client conversation state on the server between requests, not "no state"; resource state lives in the database.
- Richardson levels; most real APIs are level 2 (resources + methods + status codes).
- HATEOAS (level 3) is rare in full; what survives is `Location` headers, pagination links and action-availability flags.
*Common weak answer:* "It uses JSON over HTTP with GET and POST." That is level 1–2 at best and says nothing about the constraints or statelessness.

**Q3. Design the API for orders and payments.**
A strong answer must mention:
- Resources with opaque IDs: orders, order items, payments (top level), refunds, shipments; shallow nesting.
- Methods and statuses: `POST /orders` → `201` + `Location`; `POST /payments` with `202` for pending bank action; actions as created sub-resources (cancellation, refund).
- Idempotency keys on every creating `POST`; required on payments.
- Payment as a resource with its own state machine; the provider's final answer by signed webhook; the window where the order is unpaid.
- Pagination, filtering, ETags, problem-details errors.
*Common weak answer:* endpoints such as `/createOrder` and `/payOrder` with no idempotency, no payment lifecycle and no account of the unknown outcome.

**Q4. `PUT` versus `PATCH`: how do you design partial updates, and which status codes do you return?**
A strong answer must mention:
- `PUT` replaces the whole resource; omitted fields are reset, so older clients can erase new fields; it is idempotent.
- `PATCH` with JSON Merge Patch (partial document, null removes, arrays replaced) for ordinary updates; JSON Patch (operations, JSON Pointer, atomic, `test` preconditions) for array edits or explicit nulls.
- Apply the patch to an updatable view (not the entity) to block mass assignment, and validate the result as a whole.
- `200`/`204` on success, `409` for state conflicts, `422` for semantic validation, `412`/`428` for concurrency.
*Common weak answer:* "`PUT` updates, `PATCH` also updates." Misses the reset-on-omit data loss and the patch formats.

**Q5. How do idempotency keys work, including two retries at once?**
A strong answer must mention:
- The client generates one key per logical operation, persists it, and reuses it on every retry with the same body.
- The server stores one row per (account, key) with a unique constraint as the guard, fingerprints the request, and records the outcome.
- First request executes and records; retry after completion replays the stored response; same key with a different body returns `422`.
- Two concurrent retries: one wins the unique insert, the other sees in-progress and gets `409 Retry-After`; a lease handles a crashed holder; downstream keys and a business-level unique constraint back it up.
*Common weak answer:* "Store the key and check if it exists before processing." That check-then-act races; the unique constraint must be the guard.

**Q6. How do you design pagination? Why is offset slow?**
A strong answer must mention:
- Offset makes the database produce and discard all skipped rows, so cost grows with depth; and offsets skip or duplicate rows when the data changes under paging.
- Keyset (cursor) pagination asks for rows after the last row's sort key, seeking with an index at constant cost per page.
- A unique tie-breaker for a total order; a comparison matching the sort; an opaque, validated (or signed) cursor bound to the filters and sort.
- Offer `has_more` rather than expensive total counts; cap page size; a separate export path for everything.
*Common weak answer:* "Use `LIMIT` and `OFFSET` and add an index." The index does not stop offset from walking all skipped rows, nor the skip/duplicate problem.

**Q7. Two users edit the same resource at once. How does the API prevent lost updates?**
A strong answer must mention:
- Optimistic concurrency: a version (ETag) on every representation; the writer sends `If-Match`.
- The version is compared **inside** the `UPDATE` (`WHERE version = $expected`); zero rows affected means a `412` (or `409` for a body version).
- The ETag must change on every change (a version column, not a one-second timestamp); strong ETags only; `428` if the precondition is missing on protected endpoints.
- What the client does on `412` (re-read, merge, or retry) is a product decision.
*Common weak answer:* "Lock the row." No request holds a transaction across a human's editing time; that is optimistic concurrency's whole reason.

**Q8. How do you handle errors consistently across an API?**
A strong answer must mention:
- RFC 9457 problem details (`application/problem+json`) with `type` as the stable machine-readable identifier; clients branch on `type` and status, not on text.
- A documented catalogue of problem types with statuses and client actions.
- All validation errors at once, each with a JSON Pointer and a code; a request ID on every error; no internals in `5xx`.
- The edge (gateway, proxy) must speak the same format, not return HTML.
*Common weak answer:* "Return a JSON object with an error message." No stable identifier, no catalogue, no field-level errors; clients end up parsing strings.

**Q9. How do you version an API, and how do you deprecate a version?**
A strong answer must mention:
- Evolve additively so versions are rare; expand-and-contract for renames.
- When unavoidable, version visibly (usually in the path); or date-based per-account pinning with transformation layers over one implementation.
- Deprecation: announce, send `Deprecation`/`Sunset` headers, measure usage per client, contact heavy users, run brownouts, then `410 Gone`.
- Keep one implementation of the business logic, not controllers copied per version.
*Common weak answer:* "Put `/v2/` in the URL." That is where, not how: misses additive evolution, breaking-change rules and deprecation.

**Q10. How do you expose a ten-minute operation, and how do you make webhooks reliable?**
A strong answer must mention:
- `202 Accepted` + a status resource in shared storage; the client polls with `Retry-After`; a clear state machine; idempotent creation; cancellation.
- Webhooks built on an outbox (the event exists iff the change did), signed per endpoint with HMAC, retried with exponential backoff and jitter, auto-disabled after repeated failure.
- Documented as at-least-once and unordered, so consumers de-duplicate by event ID and check versions or fetch current state.
- An event-listing endpoint so a missed webhook is not a lost event; webhook URLs need SSRF defences.
*Common weak answer:* "Hold the request open and send the result when done." Breaks on every timeout and is lost on any restart.

**Q11. How do Protocol Buffers and Avro handle schema evolution?**
A strong answer must mention:
- Backward compatibility (new readers read old data) and forward compatibility (old readers read new data).
- Protobuf: fields by number; unknown fields skipped by wire type and preserved; never reuse a number, mark removed ones `reserved`; `optional`/wrapper for presence; enums start at `UNSPECIFIED = 0`.
- Avro: no names on the wire; the reader resolves the writer's schema by name; new fields need defaults; renames via aliases; a schema registry enforces compatibility.
- Compatibility mode decides whether producers or consumers deploy first.
*Common weak answer:* "They are backward compatible automatically." Only if the rules (reserved numbers, defaults) are followed.

**Q12. REST versus gRPC versus GraphQL: how do they differ, and when would you choose each?**
A strong answer must mention:
- Transport and payload: REST (HTTP, JSON), gRPC (HTTP/2, protobuf binary, streaming), GraphQL (one endpoint, client-shaped JSON).
- Who decides the response shape: server per endpoint, server per method, client per query.
- Caching: REST caches `GET`s; gRPC does not; GraphQL barely.
- Fit: REST for public cacheable resource APIs; gRPC for internal efficient typed streaming calls; GraphQL for composing many sources for varied clients.
- Each has a signature risk: over/under-fetching, L4 pinning, N+1 and query cost.
*Common weak answer:* "gRPC is faster, GraphQL is flexible, REST is simple." True slogans, no mechanism; misses caching, browser support, error models and the failure modes.

**Q13. What is the N+1 problem in GraphQL, and how is it solved?**
A strong answer must mention:
- A list resolver returns N items, then a field resolver runs once per item, making 1 + N queries, multiplied for nested fields.
- A DataLoader batches the keys requested in one execution step into a single query and caches within the request.
- The loader must be per request, because its cache holds data under one caller's permissions.
- The same pattern appears in ORMs and REST mapping; GraphQL makes it the default without loaders.
*Common weak answer:* "Add caching." A shared cache leaks data across users and does not batch the first requests.

**Q14. Where do authentication, authorisation and rate limiting belong in an API?**
A strong answer must mention:
- Authenticate at the edge/gateway and re-establish identity in the service (do not trust an internal header blindly).
- Object-level authorisation where the object is loaded, ideally scoped in the query, returning `404` not `403` to avoid confirming existence (BOLA/IDOR).
- Function-level checks per route, deny by default.
- Rate limits per key/user/tenant, with `429` and `Retry-After` and documented headers; input limits as resource-consumption defences.
*Common weak answer:* "The gateway checks the token." The gateway cannot do object-level authorisation; it does not know who owns the object.

**Q15. What is SSRF, and how do you defend a feature that fetches user-supplied URLs?**
A strong answer must mention:
- The server makes a request to an attacker-chosen destination, from inside the network, reaching internal services and the cloud metadata service (credentials).
- Why naive string blocklists fail: alternate IP encodings, attacker-controlled DNS names, redirects, DNS rebinding (check-then-connect race).
- Defences: avoid fetching (uploads or host allowlist); restrict scheme and port; resolve the name and check the IP, then connect to that IP; no redirects to internal ranges; do not echo the body; restrict egress and use a hardened metadata service.
- Webhook delivery is the same kind of feature.
*Common weak answer:* "Block localhost and 169.254.169.254." String blocklists miss encodings, DNS names, redirects and rebinding.

**Q16. Why can a timeout on a `POST` not simply be retried, and what makes it safe?**
A strong answer must mention:
- A timeout leaves the outcome unknown: the request may have succeeded with a lost response, failed, or still be running (Chapters 7 and 10).
- `POST` is not idempotent, so a blind retry can create a duplicate (second order, second charge).
- An idempotency key makes the retry recognisable and at-most-once; the key must reach downstream systems too.
- A business-level unique constraint is the final guard.
*Common weak answer:* "Just retry; the network probably dropped it." That double-charges when the first attempt actually succeeded.

---

## Scenario Exercises

1. A partner reports that your "non-breaking" release of last Tuesday broke their integration. You added one optional field to the order response and one new enum value for `status`. Nothing else changed. List the ways each of those could break a consumer, how you would confirm which one did, and what you would change in the contract and process to prevent it.
2. A mobile client double-charges customers about once in a thousand payments, always after poor connectivity. The server uses idempotency keys. Give the three most likely causes and the evidence that distinguishes them, then the fixes.
3. A "list transactions" endpoint using `?page=N&size=50` is fast for page 1 and times out past page 500; a nightly accounting sync that reads every page is also missing some transactions and importing others twice. Explain both problems and redesign the endpoint.
4. You must let customers register webhook endpoints and deliver events to them. Design the delivery system end to end: storage, signing, retries, ordering, duplicate handling, and the security checks on the URLs. State what you promise consumers and what you do not.
5. A team proposes moving the public API from REST to GraphQL "so clients fetch exactly what they need". The API is cached heavily at a CDN and serves mostly read traffic to third parties. Give an argued recommendation, including at least one mechanism that changes for the worse, and say where GraphQL would fit instead.
6. During a rolling deployment, a Kafka consumer starts failing to deserialise some messages; it works again once the deployment finishes. The messages use Protocol Buffers. Explain what probably changed in the schema and give the rule that would have prevented it, including which side should deploy first.
7. An internal review finds that `GET /v1/invoices/{id}` returns any invoice to any authenticated user if they guess the ID, and that IDs are sequential integers. Explain the vulnerability by name, why sequential IDs make it worse, the immediate fix, and two defences in depth.

---

## Diagnosis Lab

**Lab 1: Duplicate orders after a promotion**
- *Symptom:* During a flash sale, about 0.2% of customers have two identical orders created within a few seconds. The create endpoint accepts an `Idempotency-Key`.
- *Hypotheses:* (a) clients do not send the key; (b) clients send a different key per attempt; (c) the server does not enforce the key atomically.
- *Evidence:* The access log shows the duplicate pairs carry **different** `Idempotency-Key` values, generated milliseconds apart, from the same client version. The server's key table has a unique constraint and shows no conflicts around those times. In the client, the key is generated inside the retry function, so each attempt gets a new key; and a separate bug fires the submit handler twice on a double-tap.
- *Root cause:* (b). The key identifies the attempt, not the operation, so retries and double submits are not recognised as the same operation.
- *Fix:* Generate the key once when the user initiates checkout, persist it with the draft, and reuse it for every attempt; debounce the submit control. Add a business-level guard: a partial unique index on (customer, cart hash) over orders still in `PLACED` status, so a second identical open order is rejected by the database.
- *Prevention:* A client contract test that retries create and asserts one order; server-side alerting on duplicate orders per customer per minute.

**Lab 2: A list endpoint melts the database every night**
- *Symptom:* Each night at 02:00 the database CPU saturates for 40 minutes, and user-facing queries slow down. It coincides with a partner's export job.
- *Hypotheses:* (a) the export query is unindexed; (b) offset pagination over a deep list; (c) a missing limit.
- *Evidence:* The slow-query log shows thousands of executions of the same `SELECT … ORDER BY created_at DESC LIMIT 50 OFFSET N`, with `N` climbing from 0 to over 2,000,000. Each later execution reads and discards more rows; the query plan shows an index scan of growing length. The job walks pages sequentially to export everything.
- *Root cause:* (b). Offset pagination makes each page cost proportional to its offset; a deep sequential walk is quadratic in total.
- *Fix:* Add keyset pagination (cursor on `(created_at, id)` with the matching index) so each page costs the same; or give the partner a dedicated export that streams from a consistent snapshot (Chapter 17) or a nightly dump. Cap page size.
- *Prevention:* A performance test that walks to a deep page and checks constant per-page latency; an alert on list queries with large offsets.

**Lab 3: Payments stuck "processing" after a provider blip**
- *Symptom:* After the payment provider had a 90-second outage, a few hundred payments are stuck in `PROCESSING` in your database, though the provider's dashboard shows them as charged. Customers were told the payment failed and some paid again.
- *Hypotheses:* (a) the webhook for those payments was lost; (b) the synchronous charge call timed out and the result was never recorded; (c) a bug in the state machine.
- *Evidence:* For the stuck payments, the application log shows the charge call timing out at the 2 s client timeout during the outage; no `payment.succeeded` webhook was received (the provider's retries also failed during the outage and were not resumed); the payment row was saved as `PROCESSING` before the call and never updated. The provider shows the charges as successful, with the idempotency key your server sent.
- *Root cause:* The timeout left the outcome unknown (b), and there was no mechanism to resolve it afterward: the webhook was the only path to completion, and it was lost.
- *Fix (now):* A reconciliation job that lists payments in `PROCESSING` older than a few minutes and asks the provider for their status by the stored reference (or re-sends the charge with the same idempotency key, which the provider treats as a lookup), then settles each. Refund the genuine double payments.
- *Prevention:* Never rely on the webhook alone; run reconciliation continuously (Chapter 25). Use the idempotency key downstream so a retry during the outage would have been safe. Tell customers "payment is processing", not "failed", when the outcome is unknown.

**Lab 4: A link-preview feature reaches internal services**
- *Symptom:* Security monitoring alerts that application servers are making outbound requests to `169.254.169.254` and to several `10.x` addresses. The product has a feature that fetches a URL the user pastes and shows a preview.
- *Hypotheses:* (a) a compromised dependency; (b) SSRF through the preview feature; (c) a misconfigured internal client.
- *Evidence:* The fetch-service logs show user-supplied URLs including an alternate-encoding form of the metadata address and a public URL that redirected to an internal one. The fetcher follows redirects and returns the fetched body in the preview, and it resolves the hostname once for validation and again when connecting.
- *Root cause:* (b). SSRF: the feature fetched arbitrary URLs, a blocklist missed encoded and redirected forms, and a second DNS resolution allowed rebinding.
- *Fix (now):* Disable the feature; rotate any cloud credentials the servers could reach; require the hardened metadata service. *Fix (lasting):* allowlist hosts or switch to uploads; validate the resolved IP against internal ranges and connect to that IP; refuse redirects to internal targets; stop echoing the fetched body; restrict the fetcher's egress to deny-by-default.
- *Prevention:* A test suite of SSRF payloads (encoded addresses, redirects, rebinding) run against the fetcher; alerts on any outbound request to internal ranges from the fetch service.

---

## Solution Sketches

1. The new field breaks a consumer that uses a strict parser rejecting unknown fields, or whose code asserts the exact set of fields. The new enum value breaks a consumer that maps `status` to an enumeration and throws on unknown values, or that treats "not one of the known values" as an error state. Confirm by diffing recorded responses before and after the release and correlating the partner's error spike with the first appearance of each; ask whether their failures are parse errors (the field) or logic errors on `status` (the enum). Prevent by stating in the contract that responses may gain fields and that enumerations may gain values, so consumers must ignore unknown fields and handle unknown enum values; add consumer-driven contract tests and a staged rollout so the break shows up before full release.
2. (a) The client generates a key per attempt rather than per operation: the duplicates carry different keys; fix by generating once and persisting. (b) The key store is a cache with a short TTL or is not durable, so a retry after eviction is treated as new: duplicates carry the **same** key but the store has no record; fix with durable, uniquely constrained storage. (c) The downstream provider is called without passing the key, so the server's retry (after its own crash or a lost provider response) charges twice even though the server's key matched: the server shows one order but the provider shows two charges; fix by deriving a downstream key from the idempotency key. Evidence distinguishing them: compare the keys on the duplicate pairs and whether the duplication is at your server or at the provider.
3. Offset pagination: page 500 forces the database to read and discard 25,000 rows (and more as N grows), so deep pages time out; and because offsets count positions, rows inserted or deleted between the sync's page reads shift positions, so some rows are skipped and others seen twice. Redesign: keyset pagination with a cursor on `(created_at, id)` and a matching index, giving constant per-page cost and no skips or duplicates for stable rows; for the accounting sync, use an incremental cursor on an immutable key (creation time plus ID) or read from a consistent snapshot, and cap page size.
4. Storage: an `events` table written in the same transaction as the state change (outbox), and a `deliveries` table with one row per (event, endpoint) holding attempt count and next-attempt time. Signing: per-endpoint secret, HMAC-SHA256 over timestamp and raw body, sent in a signature header, with dual signing during secret rotation. Retries: a worker claims due deliveries with a lease, posts with a short timeout and no redirects, and on failure schedules the next attempt with exponential backoff and jitter, disabling the endpoint after repeated failures. Ordering: none promised; events carry creation time and resource version. Duplicates: at-least-once, so consumers de-duplicate by event ID. Security: the endpoint URL is user-supplied, so apply SSRF defences (resolved-IP checks, no internal ranges, no redirects). Promise consumers: authenticated, at-least-once, unordered delivery with retries, plus an event-listing endpoint to recover missed events; do not promise exactly-once, ordering, or immediate delivery.
5. Recommend keeping REST for this public, read-heavy, third-party API. The mechanism that changes for the worse is HTTP caching at the CDN: REST `GET`s are cached by URL at the edge, absorbing most of the read traffic (Chapter 7, Section 7.12), while GraphQL sends everything as `POST` to one endpoint, which the CDN cannot cache, moving all that load to the origin; field-level authorisation and query-cost control also become new burdens against untrusted third parties. GraphQL would fit better as an internal aggregation layer (a backend-for-frontend) for your own first-party apps, where you control the clients, can use persisted queries, and need to compose data from several services per screen.
6. A new version of the producer almost certainly **reused a field number** for a different field or changed a field's type, so the still-running old consumers (or new consumers reading old messages) interpret the bytes under the wrong field or wrong wire type. It works once the deployment finishes because then only one schema is in use. The rule that prevents it: never reuse a field number, mark removed ones `reserved`, and only add fields with new numbers. For a backward-compatible change (new readers read old data), deploy **consumers first**; for forward-compatible changes, producers first; enforce the rule with a schema check in CI (a breaking-change linter) and a registry.
7. The vulnerability is Broken Object Level Authorization (BOLA/IDOR): the endpoint returns an object without checking that the caller owns it. Sequential integer IDs make it worse because an attacker can simply count up through every invoice rather than having to guess. Immediate fix: scope the query by the caller (`WHERE id = $1 AND account_id = $2`) and return `404` when it does not match, so existence is not confirmed. Defences in depth: switch to opaque, non-sequential identifiers so IDs cannot be enumerated; add automated tests that call every object endpoint with another tenant's IDs and expect `404`; and log and alert on a caller requesting many foreign or sequential IDs.

---

**END OF CHAPTER 13**
