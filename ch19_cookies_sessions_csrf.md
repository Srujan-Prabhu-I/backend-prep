# Chapter 19: Web State Management: Cookies, Sessions, and CSRF

## Where We Are

Chapter 7 established that HTTP is **stateless**: each request carries everything the server needs to understand it, and the protocol itself remembers nothing between requests (Section 7.1). Chapter 7 also showed the wire syntax of the one standard mechanism that lets a server hand a browser a piece of state and get it back, the cookie (Section 7.14). Chapter 9 explained when a browser attaches a cookie to a request, what `SameSite` and `HttpOnly` do, and why cookies are **ambient credentials** that travel no matter which site caused the request (Section 9.5). Chapter 10 met sticky sessions at the load balancer and recommended against relying on them (Section 10.3), and Chapter 11 classified the kinds of state a server holds (Section 11.1).

This chapter assembles those pieces into the thing every logged-in web application runs on: the **session**. It answers three questions. Where should the state that ties requests together live: in the browser, in one server's memory, in a shared store, or in the database? How does a server create, find, expire and destroy a session securely, and how does that design change when there are twenty instances behind a load balancer? And what does the browser's habit of attaching cookies automatically let an attacker do, which is **cross-site request forgery** (CSRF), and how is it stopped?

**After this chapter you can:**
- explain why HTTP is stateless, what "state" a web application needs, and compare the places it can live;
- trace a cookie from `Set-Cookie` to its last request, and state what attack each cookie attribute defends against;
- design server-side sessions: identifier generation and entropy, storage, lookup, idle and absolute expiry, and invalidation;
- choose between sticky in-memory sessions, a shared session store and stateless signed or encrypted cookies, with their trade-offs;
- explain session hijacking and session fixation, and implement login, privilege changes and logout correctly;
- explain how CSRF works, why cookies enable it, and how `SameSite`, synchronizer tokens, double-submit cookies and `Origin` checks each block it, including when `SameSite` alone is not enough;
- explain why bearer tokens in an `Authorization` header are not exposed to CSRF, and which risk they take on instead;
- diagnose random logouts behind a load balancer, cookies missing on cross-site requests, and CSRF token mismatches.

**Interview questions this chapter prepares you for:**
1. Why is HTTP stateless, and how do sessions add state on top of it?
2. Where can session state live, and what are the trade-offs?
3. What does each cookie attribute (`Domain`, `Path`, `Expires`/`Max-Age`, `Secure`, `HttpOnly`, `SameSite`) protect against?
4. How does a server-side session work, from login to logout?
5. How should a session ID be generated, and why does its entropy matter?
6. What is the difference between an idle timeout and an absolute timeout?
7. Sticky sessions versus a shared session store versus signed cookies: which would you choose, and why?
8. What is session hijacking? What is session fixation, and how do you prevent it?
9. What does "logout" actually do, and why is it hard with stateless cookies?
10. What is CSRF, and why do cookies make it possible?
11. How do you prevent CSRF? When is `SameSite` enough, and when is it not?
12. Is an API that uses bearer tokens vulnerable to CSRF? What risk does it trade in?
13. Server-side sessions versus tokens: what is the difference?
14. How do the synchronizer token pattern and the double-submit cookie differ, and what is wrong with naive double-submit?
15. What is login CSRF, and how do you prevent it?

**Scope.** Here: where web state lives, cookies as a security mechanism end to end, server-side sessions, scaling sessions, session hijacking and fixation, logout, CSRF and its defences, the stateful-versus-stateless choice, session diagnosis, and a placement-round section on classic session-tracking techniques (Section 19.9). Recalled, not re-taught: statelessness of HTTP (Chapter 7, Section 7.1), cookie wire syntax (Chapter 7, Section 7.14), browser cookie-sending rules, `SameSite`, `HttpOnly`, cookie prefixes and token storage (Chapter 9, Section 9.5), the same-origin policy and CORS (Chapter 9, Sections 9.6–9.7), XSS (Chapter 9, Section 9.8), sticky sessions at the load balancer (Chapter 10, Section 10.3), HMAC and AEAD (Chapter 8, Sections 8.3 and 8.5), random number generators (Chapter 8, Section 8.2). Elsewhere: passwords, MFA, JWTs, OAuth 2.0 and OpenID Connect are Chapter 20; Redis as a data structure server and cache is Chapter 21; replication of a session store is Chapter 22; rate limiting of login attempts is Chapter 24.

---

## 19.1 State in a Stateless Protocol: Where State Can Live

### Definitions

**State**, in a web application, is information that the application must remember **between** requests in order to handle the next one correctly: who the user is, what is in their cart, which step of a checkout they reached, which language they chose.

A **session** is the series of interactions between one client and an application that the application treats as belonging together, usually from login to logout, together with the state attached to that series. The word is used for both the interaction and the data; **session state** means the data.

**Web state management** is the set of techniques for keeping session state across stateless HTTP requests: deciding where the state lives, how each request is linked to it, and how it is protected and eventually discarded.

Two kinds of state must be distinguished, as Chapter 13 did for REST (Section 13.2):

- **Resource state** is the application's real data: orders, accounts, saved addresses. It belongs in the system of record (Chapter 1, Section 1.6), the database.
- **Session (conversation) state** describes the current interaction: "this browser belongs to user 81, logged in at 10:02 with a second factor", "the checkout is at step 3". It is shorter-lived, often disposable, and read on almost every request.

**Authentication**, which Chapter 20 covers in full, is proving who a client is; a **credential** is whatever a client presents to prove it (a password, a token, a session cookie). A session cookie is a credential: whoever presents it is treated as that user.

### Why it exists

Recall from Chapter 7 (Section 7.1) why HTTP was designed stateless: any request can go to any server, intermediaries can cache and retry, and a server that crashes loses nothing the protocol depends on. Applications, however, are not stateless. A user who logs in expects the next page to know them; a shopping cart must survive a click. If nothing ties requests together, every request would have to carry the user's password, which is both slow (password checks are deliberately expensive, Chapter 8, Section 8.4) and dangerous (the password would be stored and sent everywhere). Sessions exist to check the user's credentials **once**, then let a short, revocable token stand in for them on later requests.

### Mechanism: the four places state can live

Every design puts session state somewhere, and every place has a different cost:

```text
 1. IN THE CLIENT                 2. IN ONE SERVER'S MEMORY        3. IN A SHARED STORE           4. IN THE DATABASE
 ┌─────────┐                      ┌─────────┐                      ┌─────────┐                    ┌─────────┐
 │ browser │ cookie holds the     │ browser │ cookie holds         │ browser │ cookie holds       │ browser │ cookie holds
 │  state  │ state itself         │   id    │ an id                │   id    │ an id              │   id    │ an id
 └────┬────┘ (signed/encrypted)   └────┬────┘                      └────┬────┘                    └────┬────┘
      │                                │                                │                              │
 ┌────▼────┐ any instance          ┌───▼─────┐ ONLY this           ┌────▼────┐ any instance       ┌────▼────┐ any instance
 │ app × N │ verifies it           │ app #3  │ instance knows      │ app × N │ looks it up        │ app × N │ looks it up
 └─────────┘                       │ {id→st} │ the id              └────┬────┘                    └────┬────┘
                                   └─────────┘                     ┌────▼────┐                    ┌────▼────┐
                                                                   │ Redis / │                    │ sessions│
                                                                   │ memcache│                    │  table  │
                                                                   └─────────┘                    └─────────┘
```

| Where | How a request finds its state | Strengths | Weaknesses |
|---|---|---|---|
| **Client** (cookie value, URL, hidden form field, browser storage) | it carries it | no server storage; any instance can serve the request | size limits (about 4 KB per cookie, sent on every request); must be signed to prevent tampering and encrypted to prevent reading; hard to revoke |
| **One server's memory** | the request must reach the same instance | fastest lookup (a hash map in the process) | lost when the instance restarts or dies; requires sticky routing; uneven load (Chapter 10, Section 10.3) |
| **Shared in-memory store** (Redis, Memcached) | any instance looks up the id in the store | instances stay stateless; sub-millisecond lookups; built-in expiry | another component that must be available; a network round trip per request |
| **Database** | any instance looks up the id in a table | durable; queryable ("list this user's sessions"); no new component | a database read (and often a write) per request; adds load to the system of record |

Requests are linked to state in the last three cases by a **session identifier**: a random, unguessable string sent with each request, almost always in a cookie (Section 19.2). The identifier is not the state; it is a key to it.

### Concrete: one shopping cart, four designs

A user adds a keyboard to their cart, then a mouse, across two requests that land on different instances behind a round-robin load balancer:

```text
 design                       request 1 (instance A)              request 2 (instance B)               result
 ───────────────────────────────────────────────────────────────────────────────────────────────────────────────
 in-memory, no stickiness     A stores cart {keyboard} in RAM     B has no cart for this id;            mouse only:
                                                                  creates {mouse}                       keyboard lost
 in-memory + sticky           LB pins user to A                   LB sends request 2 to A too           works, until A
                                                                                                        restarts
 shared store (Redis)         A writes sess:9f2… = {keyboard}     B reads sess:9f2…, writes             works; survives
                                                                  {keyboard, mouse}                     instance loss
 signed cookie                A returns Set-Cookie: cart=<signed  B verifies signature, appends,        works; cart
                              {keyboard}>                         returns new signed cookie             travels with
                                                                                                        every request
```

Real shops usually store the cart itself as resource state in the database (keyed by user or by an anonymous cart ID), because customers expect carts to survive days and devices. What remains in the session is the small, hot part: who is logged in, and how strongly.

### Guarantees and non-guarantees

Choosing a location guarantees only what that location guarantees: client-side state survives server restarts but can be read, replayed or tampered with unless protected cryptographically; in-memory state is fast but dies with its process; a shared store survives instance loss but not its own loss unless replicated (Chapter 22); the database is durable but slower. No location makes HTTP itself stateful, and no location protects a session identifier that has been stolen (Section 19.5).

### Failure and diagnosis

- **Users "randomly" lose their session or cart.** State kept in one instance's memory while requests are spread across instances, or lost when instances restart during deploys. Evidence: the session lookup misses on some instances but not others; failures correlate with deploy times or with load-balancer target changes. Diagnosis Lab 1 walks through it.
- **Tampered client state.** A cookie such as `role=user` or `price=1999` accepted without a signature. Evidence: values the server never issued appear in requests. Fix: never trust unsigned client state (Chapter 7, Section 7.14); keep authorisation-relevant facts on the server or in a signed value.
- **Database overload from sessions.** A sessions table updated on every request ("last seen at") becomes the hottest table in the database. Evidence: `pg_stat_statements` dominated by `UPDATE sessions SET last_seen …` (Chapter 18, Section 18.17). Fix: a store suited to the access pattern, or update at most every few minutes.

### Backend connection

Keep resource state in the database and keep session state small. Make application instances stateless (Chapter 11, Section 11.6) by putting session state either in a shared store or in a protected client-side value, so that any instance can serve any request, deployments do not log users out, and the load balancer can route freely.

---

## 19.2 Cookies End to End, as a Security Mechanism

### Definitions

A **session cookie**, in this chapter, means a cookie that carries a session credential: either a session identifier or a signed, self-contained session value. (Do not confuse it with the browser's term "session cookie" for a cookie without `Max-Age` or `Expires`, which disappears when the browser session ends, Chapter 7, Section 7.14. To avoid the clash, this chapter calls that kind a **non-persistent cookie**.)

The **cookie lifecycle** is the sequence a cookie goes through: the server **sets** it (`Set-Cookie`), the browser **stores** it in its cookie jar if its attributes are acceptable, **sends** it on every matching request (Chapter 9, Section 9.5), the server may **refresh** it (a new `Set-Cookie` with the same name, domain and path replaces it), and finally it is **deleted** by the server (`Max-Age=0`), by expiry, or by the user.

A **third-party cookie** is a cookie sent with a request to a site different from the one shown in the browser's address bar, for example by an embedded widget, image or iframe. A **partitioned cookie** is a third-party cookie that the browser stores separately for each top-level site that embeds the third party, so that it cannot be used to recognise the same user across different sites.

### Why it matters

Chapter 7 showed how cookies look on the wire and Chapter 9 how the browser decides to send them. For a backend engineer the decisive question is different: once a cookie **is** a credential, every attribute becomes a security control, and a wrong or missing attribute is a vulnerability with a name. This section maps each attribute to the attack it defends against, then follows a session cookie through its whole life.

### Concrete: what each attribute defends against

The session cookie used through this chapter:

```http
Set-Cookie: __Host-sid=Vq3k1wq8m1Y9Zt0cYd3s6rT2vKQ4bW8h; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=28800
```

| Attribute | What it does (Chapters 7 and 9) | Attack it defends against | What it does **not** defend against |
|---|---|---|---|
| `Secure` | sent only over HTTPS | **network sniffing**: a passive attacker on public Wi-Fi reading the cookie from a plaintext `http://` request (one plaintext image request is enough to leak it). Browsers also stop plaintext pages from **overwriting** a `Secure` cookie | theft at the endpoints (malware, XSS); requires HSTS (Chapter 8, Section 8.11) so the first request is not plaintext |
| `HttpOnly` | hidden from `document.cookie` | **theft by XSS**: an injected script copying the cookie to an attacker's server and replaying it from elsewhere (Chapter 9, Section 9.8) | XSS **using** the session from inside the page: the injected script can still send requests, and the browser attaches the cookie |
| `SameSite=Lax` / `Strict` | not sent on most cross-site requests | **CSRF** from other sites (Section 19.6); also leaks of the cookie into cross-site contexts (some timing and cross-site inference attacks) | attacks from the **same site** (a sibling subdomain), state-changing `GET`s under `Lax`, and older browsers (Section 19.6) |
| No `Domain` (host-only), or the `__Host-` prefix | sent only to the exact host that set it | **sibling-subdomain exposure and cookie tossing**: `blog.example.com` (perhaps a third-party hosted blog) receiving the session, or setting a `Domain=example.com` cookie with the same name that the application then reads instead of the real one | an attacker who controls the same host |
| `Path=/` | sent for the whole host | (not a security boundary: pages on any path of the same origin can read cookies of other paths through the DOM; Chapter 7, Section 7.14) | anything; use separate hosts, not paths, to isolate applications |
| `Max-Age` / `Expires` | browser forgets it after this time | **long-lived theft**: a stolen non-persistent or short-lived cookie stops working; a shared computer forgets the session | a stolen cookie used **before** expiry; the server must enforce its own expiry too (Section 19.3), because the client can keep sending an "expired" value |
| `__Host-` prefix | browser accepts it only with `Secure`, `Path=/` and no `Domain` | a weaker cookie of the same name being set by a subdomain or over HTTP | everything the individual attributes do not cover |
| (value design) an opaque random identifier, or a signed and encrypted value | attacker cannot forge or read it | **forgery** (`userId=1`) and **information disclosure** (a readable payload, Chapter 7, Section 7.14) | replay of the genuine value by whoever holds it |

The pattern to remember: `Secure` protects the cookie **in transit**, `HttpOnly` protects it **from page scripts**, `SameSite` controls **who can make the browser send it**, the host-only scope and `__Host-` prefix control **which servers receive and set it**, expiry limits **how long a stolen copy is useful**, and the value design decides **whether it can be forged**.

### Mechanism: cookie tossing, the attack that `__Host-` exists for

Cookies are scoped by domain, not by origin, and any subdomain can set a cookie for its parent domain. Suppose the application uses a cookie named `sid` and runs on `app.example.com`, while `promo.example.com` is a marketing site hosted by an agency, now compromised:

```text
 1. victim visits promo.example.com; its script runs:
      document.cookie = "sid=ATTACKER_SESSION_ID; Domain=example.com; Path=/app"
 2. victim visits app.example.com/app/settings; the browser sends BOTH cookies named sid:
      Cookie: sid=ATTACKER_SESSION_ID; sid=VICTIM_SESSION_ID
      (more specific Path first, so the attacker's cookie comes first)
 3. the server reads the first sid: the victim is now acting inside the ATTACKER's session.
    Anything the victim enters (a card number, an address) lands in the attacker's account,
    which the attacker can then read. (This is a form of session fixation, Section 19.5.)
```

With the session cookie named `__Host-sid`, step 1 fails: the browser refuses any `__Host-` cookie that carries a `Domain` attribute, and refuses one set from a different host. The same reasoning applies to the CSRF cookie of the double-submit pattern (Section 19.6).

### Mechanism: the lifecycle of a session cookie

```text
 login       Set-Cookie: __Host-sid=<new random id>; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=28800
             (browser validates the attributes and the prefix rules, stores it)
 requests    Cookie: __Host-sid=<id>            ← every matching request, ~40 bytes each time
 refresh     Set-Cookie: __Host-sid=<same id>; … Max-Age=28800   (sliding expiry in the browser, optional)
 privilege   Set-Cookie: __Host-sid=<NEW id>; …                 (rotation after login or elevation, Section 19.5)
 change
 logout      Set-Cookie: __Host-sid=; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=0
             (same name, no Domain, same Path: otherwise a different cookie is "deleted", Chapter 7, 7.14)
             + the server deletes the session record (Section 19.3)
```

Two backend details matter at every step:

- **Responses that set cookies must not be cached by shared caches.** A CDN or reverse proxy that caches a login response, `Set-Cookie` included, would hand the same session to every later visitor (Chapter 7, Section 7.12). Mark such responses `Cache-Control: private, no-store`, and configure caches never to store responses with `Set-Cookie`.
- **Cookies are re-sent with every request.** A session cookie of 40 bytes costs nothing; a signed cookie holding 3 KB of state adds 3 KB to every image and API request, and many large cookies together can push request headers past server limits, producing `431 Request Header Fields Too Large` or a proxy error (Chapter 7, Sections 7.5 and 7.6).

### Mechanism: third-party cookie restrictions, and why backends care

Browsers increasingly limit cookies in third-party contexts, mostly to stop cross-site tracking:

- Several browsers block third-party cookies by default, or store them **partitioned** by the embedding site.
- A server can opt a cookie into partitioned storage explicitly with the `Partitioned` attribute (the proposal known as CHIPS), which requires `Secure` and is usually combined with `SameSite=None`.
- Policies differ between browsers and have changed repeatedly; treat any design that needs an unpartitioned third-party cookie as fragile.

For a backend, this matters whenever the browser's address bar shows one site while your cookie belongs to another: a login widget embedded in a customer's page, a payment iframe, an API on a different registrable domain (`shop.com` calling `api.shop-services.net`), or single sign-on (one login shared by several applications; Chapter 20) flows that rely on a hidden iframe. The robust designs keep the credential **first-party**: serve the API under the same site as the page (`api.shop.com`), or use redirects and tokens instead of embedded cookies (Chapter 20). Diagnosis Lab 2 is this failure.

### Guarantees and non-guarantees

Correct attributes guarantee that a compliant browser keeps the session cookie off plaintext connections, out of page scripts, away from other hosts and from most cross-site requests, and forgets it at expiry. They do not guarantee anything about non-browser clients (a stolen cookie works perfectly in `curl`), about attackers who run code in your origin, about the same-site attacks of Section 19.6, or about server-side expiry, which the server must enforce itself.

### Failure and diagnosis

- **Cookie rejected or not set.** `SameSite=None` without `Secure`; `__Host-` with a `Domain`; `Secure` set on a plain-HTTP development server. Evidence: DevTools flags the `Set-Cookie` header with the reason (Chapter 9, Section 9.11). Fix: the attribute rules.
- **Session shared between users through a cache.** Two users see each other's accounts after a CDN change. Evidence: the CDN's logs show a cache hit for a response carrying `Set-Cookie`, or for a personalised page. Fix: `Cache-Control: private, no-store` on authenticated responses and a cache rule that bypasses requests with session cookies.
- **Logout does not log out.** The delete `Set-Cookie` uses a different `Path` or `Domain`, so the old cookie survives (Chapter 7, Section 7.14), and the server did not delete the session record. Fix: delete with identical scope, and always invalidate on the server (Section 19.5).
- **`431` errors or blank pages for some users.** Accumulated large cookies on the domain. Evidence: request header size in proxy logs. Fix: keep session cookies to an identifier; clean up obsolete cookies.

### Backend connection

Treat the `Set-Cookie` line of your login response as security configuration and review it like code: `__Host-` name, `Secure`, `HttpOnly`, explicit `SameSite`, a bounded `Max-Age`, an opaque value, and `Cache-Control: private, no-store` on the response. Keep the session credential first-party, so that third-party cookie restrictions never become an outage.

---

## 19.3 Server-Side Sessions

### Definitions

A **server-side session** keeps session state on the server side (in memory, a shared store or a database) under a **session identifier** (session ID), and gives the client only that identifier, usually in a cookie.

- A **session ID** is a random, unguessable token that names one session. It is a **bearer** credential: possession alone is enough to use the session.
- The **session store** is where the server keeps a record per session: a key derived from the ID, and the session data.
- **Session data** is what the record holds: typically the user ID, when and how the user authenticated (time, method, whether a second factor was used), the session's creation and last-activity times, and small per-session values such as a CSRF token (Section 19.6).
- **Session lookup** is finding the record for the ID presented on a request.
- An **idle timeout** (inactivity timeout) ends a session after a period with no requests. An **absolute timeout** ends it a fixed time after creation, however active it is. **Sliding expiry** is the implementation of idle timeouts: each request pushes the expiry forward.
- **Invalidation** is ending a session before its timeouts: at logout, on a password change, by an administrator, or after suspected compromise.

### Why it exists

A server-side session separates the **credential** the browser holds from the **state** the server trusts. The browser holds only a meaningless random string, so it cannot read or modify the state; the server can change or delete the state at any moment, so a session can be revoked instantly. That revocability, together with small cookies and no cryptographic key management on the client side, is why server-side sessions remain the default for browser-based applications.

### Concrete: login, Set-Cookie and a later request

```text
 browser                              app instance                                  session store (Redis)
 ───────────────────────────────────────────────────────────────────────────────────────────────────────────
 POST /login                          1. check credentials (password hash, Ch 20)
 email=ana@x.com&password=…  ───────► 2. id = 32 random bytes from a CSPRNG (Ch 8, 8.2)
                                         → base64url "Vq3k1wq8m1Y9Zt0cYd3s6rT2vKQ4bW8h…"
                                      3. record = {user_id: 81, auth_time: 10:02:11,
                                                   amr: [pwd, otp], created: 10:02:11,
                                                   last_seen: 10:02:11, csrf: "k29…"}
                                      4. SET sess:<sha256(id)> record EX 1800  ────────► stored with a TTL
                                      5. respond
            ◄─── 303 See Other, Location: /account
                 Set-Cookie: __Host-sid=<id>; Path=/; Secure; HttpOnly; SameSite=Lax
                 Cache-Control: no-store

 GET /account                         6. read cookie __Host-sid
 Cookie: __Host-sid=<id>   ─────────► 7. GET sess:<sha256(id)>                ────────► record (or nil)
                                      8. nil → 401 / redirect to login
                                         found → check absolute expiry (created + 8 h > now?)
                                      9. attach {user 81, strength: pwd+otp} to the request context
                                         (Chapter 11, Section 11.6)
                                     10. if last_seen older than 5 min: update it and the TTL
                                         (EXPIRE sess:<…> 1800)                ────────►
            ◄─── 200 OK, the account page for user 81
```

Steps 6 to 10 run on **every** request, in middleware before routing (Chapter 10, Section 10.6), so they must be fast: one store read per request, and a write only occasionally.

### Mechanism: generating the session ID

The ID's only protection is that nobody can guess it. Two properties are required:

1. **Generated by a cryptographically secure random number generator** (CSPRNG, Chapter 8, Section 8.2), such as the operating system's `/dev/urandom` or `getrandom`, or the language's secure random API. A general-purpose generator (a linear congruential generator, a Mersenne Twister, `Math.random()`) is predictable: after observing a few outputs an attacker can compute the next ones, and therefore the next user's session ID.
2. **Enough entropy.** **Entropy** here is the number of bits of unpredictability. OWASP's guidance sets a floor of 64 bits; common practice is 128 bits or more (16 to 32 random bytes), which costs nothing.

The arithmetic of guessing explains the numbers. Suppose an attacker can try 10,000 guesses per second against a site with 1,000,000 active sessions:

```text
 entropy     possible IDs     chance one guess hits any active session     expected time to first hit
 32 bits     4.3 × 10⁹        10⁶ / 4.3 × 10⁹ ≈ 2.3 × 10⁻⁴                  ~0.4 s            → broken
 64 bits     1.8 × 10¹⁹       ≈ 5.4 × 10⁻¹⁴                                ~59 years
 128 bits    3.4 × 10³⁸       ≈ 2.9 × 10⁻³³                                ~10²¹ years       → safe
```

What the ID must **not** be: a counter, a timestamp, a hash of the username and time, a user ID, or anything derived from data an attacker can know. It should also not encode anything: an opaque random string reveals nothing if logged or leaked.

### Mechanism: storing and finding sessions

- **Key the record by a hash of the ID**, not the ID itself (`sess:<SHA-256(id)>`). The request presents the raw ID, the server hashes it and looks up the hash. If the session store or a backup leaks, the attacker gets hashes, which cannot be presented as cookies. A fast hash is fine here: unlike passwords, the ID already has 128 bits of entropy, so there is nothing to brute-force (Chapter 8, Section 8.4).
- **Keep the record small.** It is read on every request. Store identifiers and authentication facts, not whole user profiles or permission lists; load those from their own source (with caching, Chapter 21) so that a role change takes effect without waiting for sessions to expire.
- **Index sessions by user** if you need "log out everywhere" or "list active devices": a set `user_sessions:81 → {hash1, hash2, …}` in the store, or a `user_id` column with an index in a sessions table.
- **Beware concurrent writes to one session.** Two requests from the same browser (two tabs, or parallel API calls) each load the whole session, modify different fields and write the whole record back: the last write wins and the other change is lost, the lost update of Chapter 17 (Section 17.3) in a key–value store. Write individual fields (a Redis hash and `HSET`), or keep session data immutable after login.

### Mechanism: expiry, idle versus absolute

```text
 idle timeout 30 min, absolute timeout 8 h, session created at 09:00

 09:00 ── 09:20 ── 09:45 ───────────────── 10:30 ✕         (idle: no request between 09:45 and 10:15
       request  request                    expired          → the session ended at 10:15)

 09:00 ── every few minutes, all day ─────────────────── 17:00 ✕   (absolute: active all day, but
                                                                       8 hours after creation → expired)
```

- The **idle timeout** limits the damage of an unattended, still-open browser and of a stolen ID that is no longer being used by the real user. Typical values range from minutes for banking to hours or days for low-risk consumer sites.
- The **absolute timeout** limits the lifetime of any one session ID, including a stolen one that the attacker keeps alive by using it. Without it, an attacker who steals an ID can keep it valid forever.
- **Enforce both on the server.** The browser's `Max-Age` is a convenience; a client can keep sending an old cookie. The store's TTL (time to live) implements the idle timeout; a `created` field checked on each request implements the absolute one.
- **"Remember me"** is a separate, longer-lived credential (a persistent, rotating token stored hashed in the database) that creates a fresh short session when the old one has expired, not a session whose timeout is set to a year.

A "remember me" token is built so that a database leak cannot be replayed and a stolen cookie is detected:

```text
 issue (at login, if the box is ticked)
   selector  = 12 random bytes  (public: used to find the row)
   validator = 32 random bytes  (secret: never stored in plain form)
   row       = {selector, SHA-256(validator), user_id: 81, expires: now + 30 days}
   Set-Cookie: __Host-remember=<selector>:<validator>; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=2592000

 use (a request arrives with no valid session but with the remember cookie)
   row = lookup(selector)                       → none or expired: ignore the cookie, delete it
   constant_time_equal(SHA-256(validator), row.hash)?
       yes → create a NEW short session (marked "not a fresh login": no sensitive actions
             without re-authentication); issue a NEW validator for the same selector (rotation)
       no  → the selector is right but the validator is old: someone used a copy of this cookie.
             Delete ALL of the user's remember tokens and sessions; notify the user
```

Splitting the token into a selector and a validator lets the server find the row with an ordinary index lookup while comparing the secret part in constant time. Rotating the validator on every use is what makes theft visible, the same idea as refresh-token reuse detection in Chapter 20 (Section 20.6). Password changes and "log out everywhere" delete these tokens too.

Sliding expiry has a cost: if every request rewrites the record to extend it, every read becomes a write. Extending only when the remaining lifetime falls below a threshold (or when `last_seen` is older than a few minutes) removes most of those writes.

### Mechanism: invalidation

| Event | What the server must do |
|---|---|
| Logout | delete the session record; expire the cookie |
| Login, privilege elevation (second factor, "sudo mode") | create a **new** session ID, move the data to it, delete the old one (Section 19.5) |
| Password change or reset | delete **all** sessions of the user except, optionally, the current one |
| Account disabled, suspected compromise | delete all sessions of the user |
| Role or permission change | nothing, if permissions are looked up per request; otherwise invalidate or refresh the affected sessions |

Because the state is on the server, every one of these takes effect on the user's **next request**. That immediacy is the main thing a stateless signed cookie gives up (Section 19.4).

> **Framework lens:** The Servlet API gives each request an `HttpSession` identified by the `JSESSIONID` cookie; Tomcat keeps sessions in memory by default and its default session timeout is 30 minutes at the time of writing. **Spring Session** replaces that storage with Redis or a JDBC table, so instances become stateless without changing application code. Spring Security changes the session ID on login by default (its session-fixation protection, Section 19.5). In Express, `express-session` with a store such as `connect-redis` plays the same role; Django's session framework stores sessions in the database by default, with cache-based and signed-cookie backends available.

### Guarantees and non-guarantees

A server-side session with a random 128-bit ID guarantees that the client cannot forge or read session state, that sessions can be revoked immediately, and that the ID itself reveals nothing. It does not guarantee anything once the ID is stolen (whoever holds it is the user until it is invalidated), does not survive the loss of the store unless the store is durable or replicated, and does not keep the session consistent under concurrent whole-record writes.

### Failure and diagnosis

- **Predictable IDs.** Sequential or time-based IDs, or a non-cryptographic generator. Evidence: collected IDs that share prefixes or increase; a code review of the generator. Fix: CSPRNG, 128 bits.
- **Sessions that never end.** Only an idle timeout, kept alive by polling (a dashboard that refreshes every minute keeps the session forever). Evidence: session records with `created` weeks ago. Fix: an absolute timeout.
- **Lost session changes.** A value written by one request disappears after another request. Evidence: concurrent requests from one session in the access log, whole-record writes in code. Fix: field-level writes.
- **Store saturation.** Every request writes the session ("touch"), and the store's write rate equals the site's request rate. Evidence: store metrics. Fix: extend at most every few minutes.

### Backend connection

Design the session record like a hot database row: tiny, read on every request, written rarely, keyed by a hash of a 128-bit random ID, with both an idle and an absolute expiry enforced on the server. Keep authorisation data out of it or make it cheap to refresh, and index sessions by user from the start, because "log out all devices" is always requested eventually.

---

## 19.4 Scaling Sessions: Sticky, Shared Store, or Stateless Cookie

### Definitions

With more than one application instance, every request must still find its session. Three designs are used:

- **Sticky sessions with in-memory storage**: each session lives in one instance's memory. The load balancer sends all of a client's requests to that instance (**session affinity**, Chapter 10, Section 10.3).
- **A shared session store**: sessions live in a store that every instance reaches over the network. This is usually an in-memory key–value store such as Redis or Memcached, sometimes a database table.
- **Stateless (client-side) sessions**: the session data itself travels in the cookie. It is **signed** so that the client cannot change it, and often **encrypted** so that the client cannot read it. No server keeps a record.

A **signed cookie** carries a value plus a message authentication code (an HMAC, Chapter 8, Section 8.5) computed with a key that only the servers know. An **encrypted cookie** uses authenticated encryption (AEAD, Chapter 8, Section 8.3), which gives confidentiality and integrity in one operation.

A fourth, older design copies every session to every instance (**session replication**, offered by some Java application servers). Its network traffic grows with the number of instances times the rate of session writes, so it is rarely chosen today.

### Why it matters

The session design decides what users experience during the most common production events. Those events are a deploy, an instance crash, a scale-out, a store outage, a stolen cookie, and a user who clicks "log out everywhere". Each design handles some of these well and others badly. A wrong choice shows up as users logged out after every deploy, as one Redis node taking the whole site down, or as a stolen cookie that cannot be revoked.

### Concrete: the three designs

```text
 A. STICKY + IN-MEMORY                 B. SHARED STORE                       C. SIGNED / ENCRYPTED COOKIE
 ┌────────┐                            ┌────────┐                            ┌────────┐
 │browser │ Cookie: sid=…; LB=app2     │browser │ Cookie: sid=…              │browser │ Cookie: s=<payload>.<mac>
 └───┬────┘                            └───┬────┘                            └───┬────┘
     ▼                                     ▼                                     ▼
 ┌────────┐ routes by affinity         ┌────────┐ any instance               ┌────────┐ any instance
 │   LB   │ cookie / IP hash           │   LB   │                            │   LB   │
 └┬──┬──┬─┘                            └┬──┬──┬─┘                            └┬──┬──┬─┘
  ▼  ▼  ▼                               ▼  ▼  ▼                               ▼  ▼  ▼
 app1 app2 app3                        app1 app2 app3                        app1 app2 app3
 {..} {sid→st} {..}                     │   │   │                            each verifies the MAC with
 state only on app2                     └───┼───┘                            the shared key, decrypts,
                                            ▼                                checks expiry: no lookup
                                       ┌──────────┐
                                       │  Redis   │ sess:<hash> → state, TTL
                                       │(+replica)│
                                       └──────────┘
```

| Concern | A. Sticky + in-memory | B. Shared store | C. Signed/encrypted cookie |
|---|---|---|---|
| Lookup cost per request | a hash-map read in the process | one network round trip (sub-millisecond within one zone) | a MAC check and decryption (microseconds of CPU) |
| Instance restart or crash | its users' sessions are lost | nothing lost | nothing lost |
| Deploys and scale-in | users on drained instances are logged out, or draining waits for sessions to expire | invisible to users | invisible to users |
| Load balance | uneven; new instances receive no existing users | even | even |
| Revocation (logout, "log out everywhere", compromise) | immediate, on that instance | immediate | **not possible** without extra server state, until the cookie expires |
| Size limit | none | none in practice | about 4 KB per cookie, re-sent on every request |
| New component to run | none | the store, which must be highly available | none, but **keys** to manage and rotate |
| Store outage | not applicable | every request that needs a session fails, or every user appears logged out | not applicable |
| Client can read the data | no | no | yes, if it is only signed and not encrypted |
| Replay of an old value | not applicable | not applicable | an old, unexpired cookie is accepted (for example one issued before a role was removed) |

The usual choice for a web application with more than one instance is **B**. Instances stay stateless, revocation is immediate, and a Redis lookup costs well under a millisecond. **C** suits low-risk data, or architectures with no shared store (some serverless deployments), as long as its revocation weakness is accepted or compensated. **A** survives mainly in older applications. Chapter 10 (Section 10.3) recommended stickiness only as a performance optimisation, never as the thing correctness depends on.

### Mechanism: how sticky routing works, and how it breaks

A load balancer makes a client "sticky" in one of two ways (Chapter 10, Section 10.3):

1. **An affinity cookie.** On the first response, the load balancer adds its own cookie naming the chosen instance (for example `LB=app2`, usually encrypted). Later requests carrying that cookie go to that instance.
2. **Hashing the client's IP address.** The same source IP always maps to the same instance.

Both keep working only while their assumptions hold. Affinity cookies fail when the cookie is blocked, stripped by a proxy, or scoped differently from the session cookie. IP hashing fails when a mobile client changes networks, when thousands of users share one NAT address (one instance gets all of them), and whenever the set of instances changes, because a change of N remaps most clients (Chapter 10, Section 10.3). And in every variant, a session in memory dies with its instance. Stickiness turns "this request must reach app2" into a correctness requirement that the load balancer was never designed to guarantee.

### Mechanism: the shared store in practice

Redis is the common choice. It holds data in memory, serves a lookup in tens to hundreds of microseconds plus the network round trip, and expires keys by itself:

```text
 HSET   sess:9c1f…e2  uid 81  auth 2026-10-24T10:02:11Z  amr pwd,otp  created 1761300131  csrf k29…
 EXPIRE sess:9c1f…e2  1800             ← idle timeout as a TTL
 HGETALL sess:9c1f…e2                  ← every request
 EXPIRE sess:9c1f…e2  1800             ← sliding expiry, at most every few minutes
 SADD   user_sessions:81 9c1f…e2       ← index for "log out everywhere"
 DEL    sess:9c1f…e2                   ← logout
```

The record is a Redis hash so that two requests can change different fields without overwriting each other (Section 19.3). Chapter 21 (Section 21.7) explains the data structures and why single commands are atomic.

Capacity arithmetic, with stated assumptions. At 5,000 requests per second, the store serves 5,000 reads per second plus a few hundred writes. One Redis instance handles that comfortably. For memory, assume 2 million active sessions at about 300 bytes each (key, fields and per-key overhead). That is about 600 MB.

Three operational decisions matter more than raw performance:

1. **Eviction policy.** Suppose the store is shared with a cache and configured to evict keys when memory is full (for example an LRU policy over all keys, Chapter 21, Section 21.4). Then sessions are evicted like cache entries, and users are logged out under memory pressure. Keep sessions on an instance with no eviction of session keys, and alert on memory use (Diagnosis Lab 4).
2. **Availability.** A single store node is a single point of failure for every logged-in request. Use a replicated setup with automatic failover (Chapter 22). Decide in advance what the application does when the store is unreachable. It must **fail closed**: treat the request as unauthenticated, or return `503`. It must never skip the check and continue.
3. **Durability.** Losing every session at once is not a data-loss incident, but it is an incident. Every user is logged out at the same moment, and the next minutes bring a surge of logins. Each login runs a deliberately slow password check (Chapter 20, Section 20.2), so the surge can saturate CPU on the login path. Enabling the store's persistence (Chapter 15, Section 15.8) and replication turns a store restart into a non-event.

Some applications add a tiny in-process cache in front of the store, holding each session for one to five seconds. It absorbs bursts of parallel requests from one page. The price is that a logout or revocation takes effect up to that many seconds late on other instances.

### Mechanism: stateless signed and encrypted cookies

A stateless session puts the session record into the cookie and protects it with a key:

```text
 payload = {"uid":81,"amr":["pwd","otp"],"iat":1761300131,"exp":1761328931,"ver":7}

 signed:     cookie = base64url(payload) + "." + base64url(HMAC-SHA256(K, base64url(payload)))
 encrypted:  cookie = kid + "." + base64url(nonce) + "." + base64url(AEAD_encrypt(K, nonce, payload,
                                                                    associated_data = cookie name))

 Set-Cookie: __Host-s=eyJ1aWQiOjgxLCJhbXIiOlsicHdkIiwib3RwIl0sImlhdCI6MTc2MTMwMDEzMX0.mT0qZ…;
             Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=28800
```

On every request, any instance runs the same steps:

```pseudo
function load_stateless_session(cookie):
    if cookie is missing: return ANONYMOUS
    (kid, body, tag) = split(cookie)
    key = keyring.get(kid)                       # the current key or a recent previous one
    if key is None: return ANONYMOUS             # unknown or retired key
    payload = verify_and_decode(key, body, tag)  # HMAC check (constant time) or AEAD decrypt
    if payload is INVALID: return ANONYMOUS      # tampered, truncated, or forged
    if now() > payload.exp: return ANONYMOUS     # server-side expiry: never trust Max-Age alone
    if payload.ver != user_epoch(payload.uid):   # optional revocation check (see below)
        return ANONYMOUS
    return payload
```

Three details in this design carry most of the security:

- **Expiry lives inside the payload.** The browser's `Max-Age` is advice to the browser. Anyone holding a copy of the cookie can keep sending it. Only an `exp` value that the server signed and checks makes the session end.
- **Keys need rotation.** The servers keep a **keyring**: they sign with the newest key and accept the newest few. The cookie names its key (`kid`). Retiring a key logs out every session signed with it, so rotation is gradual. A leaked key lets anyone forge a session for any user, so it is as valuable as every session at once. It belongs in a secret manager (Chapter 20, Section 20.11).
- **Associated data binds the value to its name.** With AEAD, passing the cookie name as associated data stops an attacker from moving a valid encrypted value from a low-value cookie into the session cookie.

### Mechanism: what stateless sessions give up, and the compensations

The design removes the lookup, and with it the place where a server could say "no". The consequence is easiest to see on a timeline:

```text
 10:00  Ana logs in; cookie C1 (exp 18:00) issued
 10:30  malware on Ana's laptop copies C1
 10:35  Ana notices, clicks "log out"           → her browser deletes C1
 10:36  attacker sends C1 from another machine  → signature valid, exp in the future → ACCEPTED
 ...    attacker keeps access until 18:00, and no server-side action can stop it
```

Every compensation brings back some server state:

| Compensation | How it works | State it reintroduces |
|---|---|---|
| Short lifetime plus re-issue | the cookie expires in minutes; each response re-issues a fresh one | none, but a stolen copy still works for the whole window |
| Per-user **session epoch** | the cookie carries `ver`; the server keeps one counter per user; "log out everywhere" increments it | one small, cacheable lookup per request, per user rather than per session |
| **Denylist** | revoked session identifiers are stored until their `exp` passes | a list that every instance must check |

With an epoch check, the timeline above ends at 10:35 if "log out" also increments Ana's counter. The lookup is cheaper than a session lookup, because the counter rarely changes and caches well, but it is a lookup. This is the core trade-off: **stateless sessions exchange revocation for not having a store**. Chapter 20 (Section 20.5) meets the same trade-off for JWTs.

> **Framework lens:** Rails stores sessions in an encrypted cookie by default, at the time of writing. Django offers a signed-cookie session backend (readable by the client, not encrypted) alongside its default database backend. In Node.js, `cookie-session` keeps the whole session in a signed cookie, while `express-session` keeps only an ID. All of them read their key from configuration. If the key is not set consistently, each instance or each restart uses a different key, and the failure section below follows.

### Guarantees and non-guarantees

A shared store guarantees that any instance can serve any request and that revocation is immediate, as long as the store is available. It does not guarantee survival of the store's own failure, or protection from its eviction policy. A signed cookie guarantees that the client cannot alter the session without the key. It does not guarantee secrecy (unless encrypted), revocation, freshness of the data inside it, or protection against replay of a stolen copy before `exp`. Sticky routing guarantees nothing about correctness: a restart, a network change or a change in the instance set breaks it.

### Failure and diagnosis

- **Everyone is logged out after every deploy.** Sessions live in instance memory, or each instance generates a random signing key at start-up because no key was configured. Evidence: "session not found" or "invalid signature" log lines spike at deploy times, and login traffic surges right after. Fix: a shared store, or a configured key on a keyring.
- **Random logouts that depend on which instance answers.** In-memory sessions without working stickiness, or instances configured with different signing keys. Evidence: the same cookie succeeds on some instances and fails on others; failing instances log a miss or a bad signature. Diagnosis Lab 1.
- **Logouts under memory pressure.** The store evicts session keys. Evidence: the store's evicted-keys counter rises together with "session not found" errors at peak traffic. Diagnosis Lab 4.
- **Site-wide authentication failure.** The store is down, and the application either fails all requests or, worse, treats a lookup error as "no session needed". Evidence: connection errors to the store in application logs. Fix: replication, failover, fail-closed handling, and an alert.
- **Everyone logged out after a key rotation.** The old key was removed instead of being kept for verification. Fix: add the new key, start signing with it, and remove the old key only after the longest session lifetime has passed.
- **`431` errors after moving to cookie sessions.** The session payload grew. Evidence: request header sizes in proxy logs. Fix: keep the cookie to identifiers, or move to a store.

### Backend connection

Make application instances stateless and put sessions in a replicated store with no eviction of session keys, persistence enabled, and fail-closed error handling. Choose stateless cookies only when you can accept that a stolen cookie works until it expires, or when you add an epoch check. Configure signing keys explicitly, keep them in a secret manager, and rotate them with overlap. Treat sticky routing as a cache-locality optimisation, never as the reason sessions work.

---

## 19.5 Session Security: Hijacking, Fixation, and Logout

### Definitions

- **Session hijacking** is an attack in which the attacker obtains a valid session credential belonging to another user, and uses it to act as that user. Hijacking by sniffing network traffic is also called **sidejacking**.
- **Session fixation** is an attack in which the attacker makes the victim's browser use a session ID that the attacker already knows. When the victim logs in, that session becomes authenticated, and the attacker can use it.
- **Session rotation** (session ID regeneration) means issuing a new session ID, moving the session data to it, and destroying the old ID.
- **Logout** is the user's request to end the session. Correct logout makes the credential useless on the server, removes it from the browser, and clears what the browser cached for the user.

### Why it exists

A session ID is a bearer credential (Section 19.3): whoever presents it is the user. Passwords, rate limits and second factors all protect the moment of login. After login, the session ID is the only thing standing between an attacker and the account. A stolen session cookie bypasses every factor (Chapter 20, Section 20.4). Session security is therefore about two things: keeping the ID secret for its whole life, and making sure the ID that becomes authenticated is one only the real user knows.

### Mechanism: how session IDs are stolen

| Channel | Example | Defence |
|---|---|---|
| Network sniffing | a café Wi-Fi user reads the cookie from one plaintext `http://` request | HTTPS everywhere, `Secure`, HSTS (Chapter 8, Section 8.11) |
| XSS exfiltration | an injected script sends `document.cookie` to the attacker | `HttpOnly`; fixing the XSS; CSP (Chapter 9, Section 9.8) |
| Malware on the device | an "infostealer" copies the browser's cookie database | device security; short absolute timeouts; re-authentication for sensitive actions; anomaly detection |
| Leakage through URLs | an ID in the URL (`;jsessionid=…`) ends up in history, shared links, server logs and `Referer` headers sent to other sites | never accept or emit session IDs in URLs |
| Leakage through logs | the application logs the `Cookie` header, and the logs are read by many people | redact credentials; log a short hash prefix instead |
| Prediction | IDs from a non-cryptographic generator | CSPRNG, 128 bits (Section 19.3) |
| Shared caches | a CDN caches a response that contains `Set-Cookie` | `Cache-Control: private, no-store` (Section 19.2) |

Each defence closes one channel. `HttpOnly` deserves the precise statement interviewers look for. It stops an injected script from **copying** the cookie out. It does not stop the script from **using** the session, because the script can send requests from inside the page and the browser attaches the cookie (Chapter 9, Section 9.8).

Of these channels, malware is the hardest for a server to prevent, because the attacker holds a perfect copy of the cookie. Browsers are experimenting with **device-bound session credentials**, which tie a session to a private key held in the device's secure hardware, so that a copied cookie stops working elsewhere. At the time of writing this is a proposal implemented in Chromium, not a universal feature.

### Mechanism: detecting and limiting hijacking

A server cannot tell a stolen copy from the original cookie by looking at it. It can only look at the **context** of each request. Early designs bound the session to the client's IP address and rejected any change. That breaks real users constantly: phones switch between Wi-Fi and mobile data, carriers share addresses through NAT, and IPv6 privacy addresses change. Binding to the `User-Agent` string is cheap but weak, since an attacker can copy it.

Current practice treats context as a **risk signal** rather than a hard rule:

```pseudo
on each authenticated request:
    s = session
    risk = 0
    if country(request.ip) != country(s.last_ip) within 10 minutes: risk += 2   # "impossible travel"
    if hash(request.user_agent) != s.ua_hash:                       risk += 2
    if network(request.ip) != network(s.last_ip):                   risk += 1   # different provider (ASN)
    if risk >= 3:
        require re-authentication (step-up, Chapter 20, Section 20.3) before sensitive actions
        record a security event; notify the user ("new sign-in from …")
    if risk >= 4 and action is sensitive:
        invalidate the session
```

Three further limits shrink what a stolen ID is worth. **Absolute timeouts** (Section 19.3) cap its lifetime. **Re-authentication** for sensitive actions (changing the email or password, adding a payee) means the ID alone cannot complete them. A **device list** ("active sessions", backed by the per-user index of Section 19.3) lets users see and end sessions they do not recognise.

### Mechanism: session fixation, step by step

Fixation does not steal an ID. It **plants** one. It works whenever two conditions hold: the attacker can put a chosen session ID into the victim's browser, and the server keeps the same ID when the user logs in.

```text
 SESSION FIXATION (vulnerable server: same ID before and after login)

 attacker                          victim's browser                        server
 ─────────────────────────────────────────────────────────────────────────────────────────────────────
 1. GET /  ─────────────────────────────────────────────────────────────► new anonymous session
    ◄── Set-Cookie: sid=FIX123                                             FIX123 → {anonymous}

 2. plants FIX123 in the victim's browser, for example by:
    • a link  https://shop.example/login;jsessionid=FIX123  (server accepts IDs from URLs)
    • cookie tossing from a compromised sibling subdomain (Section 19.2)
    • an XSS bug, or a plaintext HTTP page of the same site injecting a Set-Cookie
                                   Cookie: sid=FIX123

 3.                                POST /login (Ana's password)  ───────► credentials OK
                                   Cookie: sid=FIX123                       FIX123 → {user 81}   ✗ SAME ID
                                   ◄── 303 /account

 4. GET /account  ──────────────────────────────────────────────────────► FIX123 → user 81
    Cookie: sid=FIX123                                                     ◄── Ana's account page
    (the attacker never saw Ana's password or cookie: they chose the ID in advance)
```

```text
 THE FIX: rotate the ID at login

 3'.                               POST /login (Ana's password)  ───────► credentials OK
                                   Cookie: sid=FIX123                       delete FIX123
                                                                            create R7Q… → {user 81}
                                   ◄── Set-Cookie: sid=R7Q…; …

 4'. GET /account, Cookie: sid=FIX123  ─────────────────────────────────► FIX123 → nothing
                                                                           ◄── 401 / redirect to login
```

Rotation alone defeats the attack, because the ID the attacker knows no longer maps to anything. The other defences remove the ways of planting an ID:

- **Rotate on every change of privilege**: at login, after a second factor, on entering a "sudo mode", and when an administrator starts impersonating a user.
- **Accept only IDs the server issued.** Some session systems are **permissive**: if a client presents an unknown ID, they create a session under that ID. A **strict** system ignores unknown IDs and issues its own. (PHP, for example, has a `session.use_strict_mode` setting for this.)
- **Never accept session IDs from URLs or form fields.** Cookies only.
- **Use the `__Host-` prefix** so that a sibling subdomain cannot set the session cookie (Section 19.2).

The login handler that implements rotation:

```pseudo
function login(request, email, password):
    user = verify_credentials(email, password)            # Chapter 20, Section 20.2
    if user is None: return login_failed()
    old = request.session                                  # may be an anonymous session
    keep = old.pick("cart_id", "locale")                   # carry over harmless data only
    store.delete(hash(old.id))                             # the old ID is now worthless
    new_id = csprng_bytes(32)
    store.put(hash(new_id),
              keep + {user_id: user.id, auth_time: now(), amr: ["pwd"],
                      created: now(), csrf: csprng_bytes(32)},   # new CSRF token too (Section 19.6)
              ttl = IDLE_TIMEOUT)
    store.add("user_sessions:" + user.id, hash(new_id))
    response.set_cookie("__Host-sid", base64url(new_id),
                        path="/", secure=True, http_only=True, same_site="Lax")
    response.header("Cache-Control", "no-store")
    return redirect("/account", status=303)
```

### Mechanism: what logout must do

Logout looks trivial and is commonly incomplete. A complete logout performs four actions:

```text
 browser                                         server
 POST /logout                                    1. verify the CSRF token (Section 19.6)
 Cookie: __Host-sid=<id>  ─────────────────────► 2. store.delete(hash(id)); remove it from user_sessions:81
                                                 3. respond:
     ◄── 303 See Other, Location: /
         Set-Cookie: __Host-sid=; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=0
         Clear-Site-Data: "cookies", "storage", "cache"
         Cache-Control: no-store
```

1. **Invalidate on the server.** Deleting the record is what makes every copy of the ID useless, including a copy an attacker holds. Clearing only the cookie leaves a working credential behind.
2. **Expire the cookie** with the same name, `Path` and (absent) `Domain` as when it was set. Otherwise the browser treats the deletion as a different cookie (Chapter 7, Section 7.14).
3. **Clear the browser's copies of user data.** The `Clear-Site-Data` response header asks the browser to delete cookies, storage and cached responses for the origin. Authenticated pages should also carry `Cache-Control: no-store`, so that the back button on a shared computer does not show the previous user's account page from cache.
4. **Use `POST`, with CSRF protection.** A logout reachable by `GET` can be triggered by any `<img>` tag on another site. That is a nuisance by itself, and a step in some login-CSRF attacks (Section 19.6).

Two harder cases complete the picture:

- **Logout with stateless cookies.** Steps 2 and 3 work. Step 1 has nothing to delete, so a stolen copy stays valid until `exp`, unless the server keeps an epoch or denylist (Section 19.4).
- **Logout with single sign-on.** Ending the application's session does not end the identity provider's session. The next visit to the application may silently log the user back in. Whether logout should also end the provider's session, and other applications' sessions, is a design decision covered with SSO in Chapter 20 (Section 20.8).

"**Log out everywhere**" deletes every session in the user's index (`user_sessions:81`), or increments the user's epoch for stateless sessions. It should also run automatically after a password change or reset (Section 19.3).

### Guarantees and non-guarantees

Rotation at login guarantees that an ID known before login is useless after it. Server-side invalidation guarantees that a deleted session cannot be used again. Neither protects an ID stolen **after** login, which is what transport security, `HttpOnly`, timeouts and anomaly detection are for. Context checks give probabilistic signals, not proof: they produce false alarms for travelling users and miss attackers who run on the victim's own device. `Clear-Site-Data` is a request to a cooperating browser; it does nothing to copies already taken elsewhere.

### Failure and diagnosis

- **Same ID before and after login.** Evidence: in DevTools, the session cookie's value does not change across the login request; no `Set-Cookie` appears in the login response. Tool: a test that logs in and asserts that the ID changed. Fix: rotation in the login handler, or enable the framework's fixation protection.
- **Session IDs appearing in URLs.** Evidence: access logs or analytics show `jsessionid` or `sid=` in paths or query strings; `Referer` headers carry them to third parties. Fix: disable URL-based session tracking (Section 19.9).
- **"Logged out" users still active.** Evidence: requests succeeding with a session ID after the logout request for it appears in the log. Cause: logout cleared the cookie but not the record, or the sessions are stateless without an epoch. Fix: server-side invalidation.
- **Zombie sessions after logout.** An in-flight request from the same browser finishes after the logout and writes the whole session record back, re-creating it. Evidence: a session record with a `created` time before the logout and writes after it. Fix: update only existing keys (`EXPIRE`, which does nothing to a missing key, or Redis `SET … XX`), never an unconditional full write.
- **A user reports activity they did not perform.** Evidence: the session's requests come from two networks at once, with different `User-Agent` strings. Response: invalidate the user's sessions, force re-authentication, check for malware or XSS, and review how the ID could have leaked.

### Backend connection

Write the login, privilege-change and logout handlers once, centrally, and test them. Rotate the ID at login and at every elevation. Invalidate on the server at logout and on password change. Never accept IDs from anywhere but a `__Host-` cookie. Record enough context per session (creation time, last IP network, a user-agent hash) to show users their active sessions and to trigger step-up authentication on suspicious changes. Remember that a session cookie copied by malware is indistinguishable from the original: the defences that work are limits on what the session can do and how long it lives.

---

## 19.6 Cross-Site Request Forgery (CSRF)

### Definitions

**Cross-site request forgery** (CSRF, also called **session riding**) is an attack in which a page on another site causes the victim's browser to send a state-changing request to a target site where the victim is logged in. The browser attaches the victim's cookies automatically, so the target executes the request as the victim.

Some terms the section uses:

- An **ambient credential** is a credential the browser attaches to requests by itself, without the page's code choosing to send it (Chapter 9, Section 9.5). Cookies are the main one. HTTP Basic credentials that the browser remembers, client TLS certificates, and the browser's automatic Windows sign-in (Kerberos or NTLM) are others.
- A **state-changing request** is one that modifies something on the server: a transfer, a password change, a new email address, a deletion. Requests that only read data are not useful for CSRF, because the attacking page cannot read the response (the same-origin policy, Chapter 9, Section 9.6).
- A **CSRF token** (anti-CSRF token) is an unpredictable value that the server requires in every state-changing request, placed where the attacker's page cannot read or produce it.
- **Login CSRF** is the variant in which the attacker makes the victim's browser log in to the **attacker's** account.

CSRF is an instance of the **confused deputy** problem. The browser is a deputy that holds the user's authority (the cookie). The attacker's page tricks the deputy into using that authority for the attacker's purpose.

### Why it exists

Recall from Chapter 9 that the same-origin policy stops a page from **reading** another origin's responses, but not from **sending** requests to it (Section 9.6). A page can submit a form to any URL, load an image from any URL, and navigate anywhere. That openness is what lets the web link sites together. Combined with ambient cookies, it means that any site the user visits can make the user's browser send authenticated requests to your server. Your server sees a valid session cookie and has no other way to know that the user never intended the request.

CORS does not help. A cross-site form `POST` with an ordinary form encoding is a **simple request**: the browser sends it without any preflight, and the server executes it. CORS can only stop the attacking page from reading the response, which the attacker does not need (Chapter 9, Section 9.7).

### Concrete: the attack, step by step

Ana is logged in to `https://bank.example`. Its transfer form posts to `/transfer`. She then opens a link to `https://evil.example/prize`, which contains:

```html
<!-- on https://evil.example/prize -->
<h1>Loading your prize…</h1>
<form id="f" action="https://bank.example/transfer" method="POST">
  <input type="hidden" name="to"     value="ACCT-ATTACKER">
  <input type="hidden" name="amount" value="5000">
</form>
<script>document.getElementById("f").submit();</script>
```

The script submits the form as soon as the page loads. Ana's browser sends:

```http
POST /transfer HTTP/1.1
Host: bank.example
Origin: https://evil.example
Referer: https://evil.example/prize
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Content-Type: application/x-www-form-urlencoded
Cookie: __Host-sid=Vq3k1wq8m1Y9Zt0cYd3s6rT2vKQ4bW8h      ← attached ONLY if the cookie's SameSite allows it

to=ACCT-ATTACKER&amount=5000
```

If the session cookie is attached and the bank has no other check, the session middleware finds user 81, the handler moves 5,000 to the attacker, and the attack is complete. The attacker never saw the cookie. They did not need to.

Three preconditions made it possible, and every defence removes one of them:

1. **The action can be triggered by a request the attacker can cause** (a form post, a link, an image load).
2. **The request is authenticated only by an ambient credential** (the session cookie).
3. **Every parameter of the request is predictable** (`to` and `amount` are chosen by the attacker; nothing in the request is secret).

### Concrete: each defence blocking the same request

```text
 Defence                         What the server or browser sees            Outcome
 ────────────────────────────────────────────────────────────────────────────────────────────────────────
 SameSite=Lax (or Strict)        browser omits the cookie: the request is a  no session → 401 / login page
 on the session cookie           cross-site POST, not a top-level GET        (removes precondition 2)

 Synchronizer token              no "csrf" field in the body; the page on    403 "CSRF token missing"
                                 evil.example cannot read the bank's form    (removes precondition 3)
                                 to copy the token

 Double-submit cookie            cookie csrf=… may be sent, but the body     403 "CSRF token mismatch"
 (signed)                        has no matching value, and evil.example
                                 can neither read nor forge it               (removes precondition 3)

 Origin check                    Origin: https://evil.example is not in the  403 "origin not allowed"
                                 allowlist [https://bank.example]            (removes precondition 1 for
                                                                             requests from other origins)
 Fetch Metadata check            Sec-Fetch-Site: cross-site on a POST to a   403
                                 non-public endpoint
```

The following subsections give the mechanism of each, and its limits.

### Mechanism: SameSite, and exactly when it is enough

Recall the rule from Chapter 9 (Section 9.5). `Strict` cookies are never sent on cross-site requests. `Lax` cookies are sent cross-site only on **top-level navigations with a safe method**: the user follows a link and the address bar changes, with `GET`. `None` cookies are always sent (and must be `Secure`). The decision depends on the **site** (scheme plus registrable domain), not the origin.

`SameSite=Lax` blocks the attack above, because it is a cross-site `POST`. It is a strong, cheap baseline. It is **enough on its own** only when every one of these holds:

1. **No state change is reachable by `GET`.** `Lax` sends the cookie on a cross-site top-level `GET`. A link to `https://bank.example/transfer?to=ACCT-ATTACKER&amount=5000` is a CSRF that `Lax` allows. So is a `GET` that a framework treats as a `POST` through a method-override parameter (`?_method=POST`).
2. **No attacker is same-site.** Every host under your registrable domain is trusted. A compromised marketing subdomain, a user-content host such as `alice.pages.example.com`, or a forgotten subdomain pointing at a deleted cloud resource is **same-site**. Its requests carry `Lax` and even `Strict` cookies.
3. **The attribute is set explicitly.** Several major browsers apply `Lax` to cookies without a `SameSite` attribute, but not all do. Chromium also made a temporary exception for cookies without the attribute: for about two minutes after such a cookie is set, it is still sent on top-level cross-site `POST`s. Relying on defaults means relying on the oldest behaviour among your users' browsers.
4. **The cookie does not need `SameSite=None`.** Embedded widgets, payment iframes, and login flows in which an identity provider posts the result back to your site cross-site (the SAML POST binding, Chapter 20, Section 20.8) may need `None`. For such cookies, `SameSite` offers no protection at all.
5. **Your own pages cannot be turned into request senders.** If a page on your site reads a value from its URL and issues a request built from it (for example client-side code that calls `fetch(params.get("next"))` with credentials), an attacker who links to that page makes **your** page send the request. The request is same-site, so cookies flow. This is sometimes called **on-site request forgery** or a client-side CSRF.

When all five hold, `Lax` stops CSRF in current browsers. In practice, applications rarely know that all five hold forever: a new `GET` endpoint, a new subdomain or a new embed breaks the assumption without anyone noticing. Guidance such as OWASP's therefore treats `SameSite` as **defence in depth**: keep it, and add a token or an origin check for state-changing requests.

`Strict` closes condition 1 for cross-site links, but it has a usability cost. A user who follows a link from an email to your site arrives without cookies, so the first page appears logged out. A common pattern uses two cookies: a `Lax` cookie that lets the user see pages, and a `Strict` cookie that the server requires for state-changing requests.

### Mechanism: the synchronizer token pattern

The **synchronizer token pattern** stores a random token in the server-side session and requires every state-changing request to echo it.

```text
 1. session created (login)       session.csrf = base64url(csprng_bytes(32))         ← Section 19.5 rotates it
 2. server renders a form         <form method="POST" action="/transfer">
                                    <input type="hidden" name="csrf" value="k29fQ…">
                                    …
    or, for script-driven pages:  <meta name="csrf-token" content="k29fQ…">
                                  and the page's code sends it as a header:  X-CSRF-Token: k29fQ…
 3. every POST/PUT/PATCH/DELETE   middleware:
                                    sent = body.csrf or header["X-CSRF-Token"]
                                    if sent is missing or not constant_time_equal(sent, session.csrf):
                                        return 403
```

Why the attacker cannot supply it: the token appears only inside your pages. Reading them from `evil.example` is exactly what the same-origin policy forbids. The comparison uses a **constant-time** function (Chapter 8, Section 8.5) so that response timing reveals nothing about how many characters matched.

Design choices:

- **Per-session or per-request tokens.** A token per session is the common choice. A fresh token per request is slightly stronger but breaks the back button and pages open in several tabs, because older forms carry tokens that are no longer valid.
- **Safe methods never change state.** The middleware checks only unsafe methods. That is correct only if `GET`, `HEAD` and `OPTIONS` handlers really are safe (Chapter 7, Section 7.4). A `GET` that deletes something is unprotected by design.
- **Do not put the token in URLs.** URLs leak through logs and `Referer` headers, like session IDs (Section 19.5).
- **Mask the token in compressed pages.** A secret that appears in a compressed HTTPS response next to attacker-influenced text can be recovered byte by byte by measuring response sizes (the **BREACH** attack). Frameworks such as Django and Rails therefore send the token XOR-ed with a fresh random value in every response; the server un-masks it before comparing.

The pattern needs server-side session state, which a server-side session already has. A **stateless variant** computes the token instead of storing it: `token = HMAC(K, session_id ‖ timestamp)`, verified by recomputing it. This binds the token to the session without storing anything new.

### Mechanism: the double-submit cookie

The **double-submit cookie** pattern needs no server storage. The server sets a random value in a cookie. The page's code reads the cookie (so it must not be `HttpOnly`) and copies the value into a header or form field. The server checks that the two match.

```text
 response:  Set-Cookie: __Host-csrf=Rk2…; Path=/; Secure; SameSite=Lax         (readable by script)
 request:   Cookie: __Host-sid=…; __Host-csrf=Rk2…
            X-CSRF-Token: Rk2…                                                  (copied by the page)
 server:    header == cookie ?  → proceed  :  403
```

The attacker's page cannot read the cookie (it belongs to another site), and a cross-site form cannot set custom headers. But the **naive** version has a hole: it trusts that only your site can **write** the cookie. An attacker who can set cookies for your domain, through a compromised sibling subdomain (cookie tossing, Section 19.2) or a plaintext page of the same site, sets `csrf=X` and submits `X` in the form. The two match, and the check passes.

The fixes:

- **Sign the token and bind it to the session**: `token = HMAC(K, session_id ‖ random) ‖ random`. The server recomputes the HMAC with the current session ID. An attacker cannot create a valid token for the victim's session without the key. OWASP calls this the **signed double-submit cookie** and recommends it over the naive form.
- **Name the cookie with `__Host-`**, so that subdomains and plaintext pages cannot set it.

Use double-submit when the application has no server-side session storage, for example with the stateless cookies of Section 19.4.

### Mechanism: checking where the request came from

Browsers tell the server where a request came from, through headers that page scripts cannot set or change (forbidden request headers, Chapter 9, Section 9.4):

- **`Origin`** names the origin of the page that caused the request. Browsers send it on cross-origin requests, and on same-origin requests whose method is not `GET` or `HEAD`. In some privacy-sensitive cases (sandboxed frames, some redirects) its value is the literal string `null`.
- **`Referer`** holds the URL of the referring page, subject to its referrer policy (Chapter 9, Section 9.9). It is a fallback when `Origin` is absent.
- **`Sec-Fetch-Site`** (`same-origin`, `same-site`, `cross-site`, `none`) is one of the Fetch Metadata headers (Chapter 9, Section 9.4). Current versions of the major browsers send it; older browsers do not.

An origin check in middleware:

```pseudo
TRUSTED = {"https://bank.example", "https://www.bank.example"}     # from configuration

function check_origin(request):
    if request.method in {GET, HEAD, OPTIONS}: return ALLOW
    site = request.header("Sec-Fetch-Site")
    if site == "cross-site": return DENY                    # cheap first filter, when present
    origin = request.header("Origin")
    if origin is not None and origin != "null":
        return ALLOW if origin in TRUSTED else DENY
    referer = request.header("Referer")
    if referer is not None:
        return ALLOW if origin_of(referer) in TRUSTED else DENY
    return DENY                                              # no evidence at all: refuse
```

Three rules make it correct:

1. **Compare against configured origins, not the `Host` header.** Behind a reverse proxy, the application may see an internal host name. The public origin belongs in configuration (Chapter 10, Section 10.2).
2. **Exact match.** `https://bank.example.evil.com` starts with `https://bank.example`. Prefix or substring matching is a vulnerability.
3. **Decide what a missing header means.** Refusing requests with neither `Origin` nor `Referer` is the strict choice. It can break a small number of users behind privacy tools that strip `Referer`, which is why some applications log such requests before enforcing.

An origin check removes cross-origin attackers in one place, with no tokens in pages. It does **not** stop same-origin attacks (XSS), and treating `same-site` as trusted re-opens the sibling-subdomain hole of the `SameSite` discussion.

### Mechanism: custom headers and JSON APIs

A cross-site form can send only three content types (`application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`) and cannot add headers. A cross-origin `fetch` that adds a custom header, or that sends `Content-Type: application/json`, is not a simple request: the browser first sends a **preflight** `OPTIONS` request, and sends the real request only if your server's CORS response approves it (Chapter 9, Section 9.7).

So an API that **requires** a custom header (for example `X-CSRF-Token`, or even a constant `X-Requested-With`) on every state-changing request, and whose CORS policy does not approve other origins, cannot be reached by a cross-site form or a simple request. Two traps break this:

- **Lenient body parsing.** An attacker sends a JSON body with `Content-Type: text/plain`, which needs no preflight. If the server parses the body as JSON regardless of `Content-Type`, the request works. Reject state-changing requests whose `Content-Type` is not what the endpoint expects.
- **A permissive CORS policy.** A server that reflects any `Origin` in `Access-Control-Allow-Origin` together with `Access-Control-Allow-Credentials: true` lets every site make credentialed requests **and read the responses**. The attacker's page can then fetch your form, read the CSRF token, and submit it. This one misconfiguration defeats every CSRF defence in this section.

### Mechanism: login CSRF and logout CSRF

**Login CSRF** turns the attack around. The attacker's page submits **your** login form with the **attacker's** credentials. The victim's browser ends up logged in to the attacker's account without noticing. Whatever the victim does next is recorded in an account the attacker controls: their searches, an uploaded document, a card number saved "for next time". OAuth flows have the same problem at the callback, which is why Chapter 20 (Section 20.7) requires the `state` parameter.

Defences: protect the login form like any other state-changing form. A synchronizer token needs a session before login, so the server creates an anonymous **pre-session** that holds the token, and rotates it at login (Section 19.5). An origin check works without any pre-session.

**Logout CSRF** forces a user out, which is mostly a nuisance. It matters as the first step of a login-CSRF chain (log the victim out, then log them in as the attacker), which is why logout uses `POST` with a token (Section 19.5).

### Mechanism: why bearer tokens are not CSRF-prone, and what they trade

An API that authenticates with a **bearer token** in the `Authorization` header (`Authorization: Bearer eyJ…`, Chapter 7, Section 7.6) is not vulnerable to CSRF. The reason is mechanical. The browser never adds an `Authorization` header by itself. Your page's code adds it, using a token your code holds. The attacker's page has no access to that token (it lives in your origin's memory or storage, protected by the same-origin policy), and a cross-site form cannot add headers at all. Precondition 2 of the attack (an ambient credential) is gone.

The trade is in where the token lives. To add the header, your page's code must be able to **read** the token. That makes it readable by any script running in your origin, including an injected one:

```text
                          cookie session (HttpOnly)                  bearer token readable by script
 CSRF                     possible: browser attaches the cookie;     not possible: nothing is ambient
                          needs SameSite + token/origin checks
 XSS (script in page)     can USE the session while the page is      can USE the token, AND COPY it to the
                          open; cannot copy the cookie out           attacker, who uses it from anywhere,
                                                                     until it expires
```

So bearer tokens remove one attack class and enlarge the damage of another. XSS is the more common and more damaging bug in modern applications, so current guidance for browser applications prefers `HttpOnly` cookies plus CSRF defences, or a backend-for-frontend that keeps tokens off the browser entirely (Chapter 20, Section 20.6).

Token-based APIs **are** exposed to CSRF in these cases:

| Case | Why CSRF applies again |
|---|---|
| The token is stored in a cookie and the server reads it from the cookie ("JWT in a cookie") | it is an ambient credential again; the token format is irrelevant |
| The API accepts the token from a header **or** a cookie, as a fallback | the attacker uses the cookie path |
| HTTP Basic authentication in a browser | browsers remember Basic credentials and resend them automatically |
| Client TLS certificates or automatic Windows sign-in (Kerberos, NTLM) | the browser presents them by itself |
| Authentication by network position (an intranet allowlist, a service on `localhost` or a home router) | the victim's browser is inside the network, so its requests are trusted; browsers are adding restrictions on requests from public sites to private addresses, but support varies |
| A permissive CORS policy with credentials | any site can read your responses and forge whatever is needed |

### Mechanism: choosing the defences

| Application | Recommended combination |
|---|---|
| Server-rendered pages with server-side sessions | `SameSite=Lax` session cookie + synchronizer tokens in forms (the framework default) + origin check on unsafe methods |
| Single-page application on the same site as its API, with a cookie session (including a BFF) | `SameSite=Lax` or `Strict` + a token sent in a custom header (synchronizer via a page meta tag, or signed double-submit) + origin and Fetch Metadata checks + a strict CORS policy + `Content-Type` enforcement |
| Stateless signed-cookie sessions | `SameSite=Lax` + signed double-submit cookie + origin check |
| API used only with bearer tokens in the `Authorization` header, no cookies at all | no CSRF tokens needed; the effort goes into XSS prevention, token storage and short lifetimes (Chapter 20) |
| WebSocket endpoints authenticated by cookies | check `Origin` on the upgrade request (cross-site WebSocket hijacking, Chapter 10, Section 10.9) |

Every combination assumes there is no XSS. A script running in your origin can read any token and send any request, so no CSRF defence survives it (Chapter 9, Section 9.8).

> **Framework lens:** Spring Security enables CSRF protection by default for unsafe methods. It accepts the token as the `_csrf` form parameter or the `X-CSRF-TOKEN` header, and offers a cookie-based repository (`XSRF-TOKEN` cookie, `X-XSRF-TOKEN` header) for script-driven pages, the convention Angular's HTTP client follows. Django's `CsrfViewMiddleware` uses a `csrftoken` cookie, the `{% csrf_token %}` template tag and the `X-CSRFToken` header, checks the `Origin` header, and reads extra allowed origins from `CSRF_TRUSTED_ORIGINS`. Rails embeds an `authenticity_token` through `protect_from_forgery`. Teams that disable these protections "because we use JSON" should first check which of the cases above they fall into.

### Guarantees and non-guarantees

A correctly checked, unpredictable CSRF token guarantees that a state-changing request came from a page that could read your content: your own pages, or a script running in your origin. An origin check guarantees that the request was caused by a page from an allowed origin, in a browser that sends the headers. `SameSite=Lax` guarantees that a cross-site `POST`, subresource or `fetch` request does not carry the cookie. None of them protects state-changing `GET`s, same-origin attackers (XSS), or a CORS policy that lets other origins read responses. `SameSite` alone does not protect against same-site attackers or cookies set to `None`.

### Failure and diagnosis

- **Legitimate users get `403` on form submission.** Common causes: the form was loaded before login and submitted after it, so its token belongs to the rotated pre-session; the session expired while the form was open; a CDN cached a page with someone else's token; per-request tokens with several tabs open. Evidence: CSRF rejections logged by reason (missing, mismatch, no session), together with session age. Diagnosis Lab 3.
- **Script-driven requests all fail with `403`.** The page's code does not send the header, or reads the cookie from the wrong name. Evidence: the failing request in DevTools has no `X-CSRF-Token` header. Fix: configure the HTTP client to send it.
- **Origin check rejects everything behind a proxy.** The application compares `Origin` with its internal `Host`. Evidence: `Origin: https://shop.example` rejected, with the application seeing `Host: app-internal:8080`. Fix: configured trusted origins (Chapter 10, Section 10.2).
- **A CSRF vulnerability despite `SameSite=Lax`.** A state-changing `GET`, a same-site attacker, or a cookie with `None`. Evidence: a penetration test or a bug report with a link-based exploit. Fix: make the endpoint `POST`, and add tokens or origin checks.
- **CSRF protection silently missing.** It was disabled globally during development, or an endpoint was exempted. Evidence: an automated test that sends a cross-site `POST` without a token to every state-changing route and expects `403`.

### Backend connection

Treat every cookie-authenticated, state-changing endpoint as a CSRF target. Keep `GET` safe. Set `SameSite=Lax` explicitly, keep the framework's token protection on, add an origin check in middleware, and lock down CORS. Rotate the CSRF token with the session at login, and protect the login form with a pre-session token or an origin check. Use bearer tokens in headers when the client is not a browser, or when you have accepted the XSS trade-off deliberately, and remember that a token stored in a cookie is a cookie.

---

## 19.7 Stateful versus Stateless Authentication: A Preview

### Definitions

- **Stateful authentication** keeps the authoritative record of each login on the server. The client holds only a reference to it, such as the session ID of Section 19.3. Every request is checked by looking the reference up.
- **Stateless authentication** puts the authenticated facts into a **self-contained token**, signed by the issuer. A server checks a request by verifying the signature and the expiry with a key, with no lookup. The signed cookie of Section 19.4 is one; a JWT (Chapter 20, Section 20.5) is another.
- A **bearer token** is any credential that grants access to whoever presents it, with no further proof (Chapter 20, Section 20.5). Session IDs, signed cookies and most access tokens are bearer tokens.

### Why it matters

Discussions of "sessions versus tokens" often mix two independent decisions. Separating them prevents the most common design mistake of this chapter: adopting a token "to avoid CSRF" and then storing it in a cookie, which brings CSRF straight back.

### Mechanism: two independent axes

The first axis is **where the authoritative state lives**: on the server (stateful) or inside the credential (stateless). The second axis is **how the credential travels**: as a cookie the browser attaches by itself, or in a header that the client's code attaches.

| | Sent as a cookie (ambient) | Sent in the `Authorization` header (explicit) |
|---|---|---|
| **Stateful** (opaque ID, server lookup) | classic server-side session (Section 19.3) | opaque access token, common in mobile apps and with token introspection (Chapter 20, Section 20.5) |
| **Stateless** (signed, self-contained) | signed or encrypted cookie session (Section 19.4); "JWT in a cookie" | JWT access token in a header (Chapter 20, Section 20.5) |

Each axis decides a different set of properties:

- **The state axis decides revocation, freshness and scaling.** Stateful credentials can be revoked instantly and always reflect current data, at the cost of a lookup and a store to keep available. Stateless credentials need no store, and can be verified by any service holding the key, but they cannot be revoked before expiry without bringing state back (Section 19.4).
- **The transport axis decides browser attack exposure.** Cookies are ambient, so they need CSRF defences, but an `HttpOnly` cookie cannot be copied out by XSS. Header tokens are not ambient, so CSRF does not apply, but script must hold the token, so XSS can steal it (Section 19.6).

The same request in both transports:

```http
GET /api/orders HTTP/1.1                         GET /api/orders HTTP/1.1
Host: shop.example                               Host: api.shop.example
Cookie: __Host-sid=Vq3k1wq8m1Y9Zt0cYd3s6rT2vKQ4  Authorization: Bearer eyJhbGciOiJFUzI1NiIs…

browser added the cookie by itself               the page's (or app's) code added the header
→ CSRF defences needed; XSS cannot copy it       → no CSRF; XSS can copy it if script can read it
```

### Guarantees and non-guarantees

Neither axis makes a credential safe once stolen: every bearer credential works for whoever holds it. "Stateless" rarely means the system holds no state. Refresh tokens, denylists, session epochs and signing keys are all state; the difference is how much of it each request must consult.

### Failure and diagnosis

- **"We switched to JWTs to fix CSRF," with the JWT in a cookie.** The transport did not change, so the exposure did not change. Evidence: the token arrives in the `Cookie` header. Fix: CSRF defences as for any cookie, or move the token to a header with the XSS trade-off accepted.
- **"We switched to JWTs to scale," and logout no longer works.** The state axis changed, and revocation went with it. Evidence: requests succeed with a token after its user logged out. Fix: short lifetimes with refresh tokens, or a revocation check (Chapter 20, Section 20.6).

### Backend connection

Decide each axis separately. For a browser talking to its own backend, a stateful session in an `HttpOnly` cookie, with CSRF defences, is the simplest secure default. For mobile apps, third-party clients and service-to-service calls, which have no ambient credentials, header tokens fit naturally. Chapter 20 (Sections 20.5–20.6) covers tokens, JWTs, refresh tokens and the full trade-off between sessions and JWTs.

---

## 19.8 Diagnosing Session and CSRF Problems

### Definitions

**Session diagnosis** is the work of finding out, for a request that was treated as logged out or rejected, which link in the chain failed. The chain has five links: the server **set** the cookie, the browser **stored** it, the browser **sent** it, the server **found** the session, and the server **accepted** the request (CSRF and origin checks). A **miss reason** is a code that the server logs when the fourth link fails, naming why: no cookie, unknown ID, idle-expired, absolute-expired, bad signature, unknown key.

### Why it matters

Every session failure looks the same to the user: "it logged me out" or "the form doesn't work". The causes range from a browser's privacy policy to a Redis eviction setting, and each needs a different fix. Guessing wastes days. Walking the five links in order, with the right tool for each, usually finds the broken one in minutes.

### Mechanism: the five links and the tool for each

| Link | Question | Tool and evidence |
|---|---|---|
| 1. Set | Did the response carry `Set-Cookie` with the expected name and attributes? | DevTools Network panel: the response headers; `curl -si` |
| 2. Stored | Did the browser accept it? | DevTools Application panel: the cookie list; rejected `Set-Cookie` headers are flagged with a reason (Chapter 9, Section 9.11) |
| 3. Sent | Did the failing request carry it? | DevTools: the request's `Cookie` header and its Cookies tab, which lists cookies that were blocked for this request and why |
| 4. Found | Did the server find a valid session? | application logs with miss reasons and a short session fingerprint; the session store (`TTL`, `HGETALL`) |
| 5. Accepted | Did a CSRF or origin check reject it? | logs of CSRF rejections by reason; the `Origin`, `Referer` and `Sec-Fetch-Site` request headers |

Links 1 to 3 happen in the browser, so the browser's tools answer them. Links 4 and 5 happen on the server, so they need logs that the application must be built to write.

### Concrete: reproducing a session with `curl`

`curl` keeps cookies in a **cookie jar** file (`-c` writes it, `-b` reads it), which lets you replay a session exactly:

```bash
# log in and save the cookies
curl -s -o /dev/null -c jar.txt \
     -d 'email=ana@x.com&password=correct-horse' https://shop.example/login

# send the same session 20 times; print the status and which instance answered
for i in $(seq 1 20); do
  curl -s -o /dev/null -b jar.txt \
       -w '%{http_code} %header{x-served-by}\n' https://shop.example/account
done | sort | uniq -c
```

Here `X-Served-By` stands for a header your own application adds with its instance name; it is not a standard header, and the `%header{…}` output variable needs a recent `curl`. A result such as:

```text
      7 200 app-1
      6 401 app-2
      7 401 app-3
```

shows that the session exists only on `app-1`: in-memory sessions without working stickiness (Diagnosis Lab 1). A cookie jar is itself a credential file. Delete it after the test.

### Concrete: logs that make link 4 visible

The session middleware should log one line per miss, with a reason and a fingerprint. The fingerprint is the first few hex characters of the SHA-256 of the ID: enough to correlate requests, useless to an attacker. Never log the raw ID or the `Cookie` header (Section 19.5). An example format of your own design:

```text
ts=10:42:07.113 inst=app-2 evt=session_miss reason=unknown_id sid_fp=9c1fe2 path=/account ua_fp=55ab
ts=10:42:07.590 inst=app-1 evt=session_hit  sid_fp=9c1fe2 age_s=1840 idle_s=12
ts=10:51:30.004 inst=app-3 evt=csrf_reject  reason=mismatch sid_fp=77d0aa sess_age_s=3 origin=https://shop.example
```

Count these events by reason and by instance as metrics. A session-miss rate that is high on some instances and not others points at instance-local state or keys. A miss rate that rises with the store's memory points at eviction. A spike right after deploys points at in-memory sessions or unconfigured keys. A `csrf_reject` with a session age of a few seconds points at a token from before login rotation.

For a shared store, inspect the record directly, without scanning the whole keyspace (`KEYS` blocks Redis; use the fingerprint you already have, or `SCAN`, Chapter 21, Section 21.7):

```text
> TTL sess:9c1f…e2
(integer) 1734                ← seconds left; -2 means the key does not exist
> INFO stats                  ← look at evicted_keys: non-zero and rising means eviction is happening
> CONFIG GET maxmemory-policy ← "noeviction" is what a session store should report
```

### Mechanism: symptom to first check

| Symptom | First check | Likely causes |
|---|---|---|
| logged out at random, more often with more instances | miss reasons by instance; `curl` loop | in-memory sessions without stickiness; different keys per instance (Lab 1) |
| everyone logged out after each deploy | miss reasons at deploy time | in-memory sessions; key generated at start-up |
| logged out at peak traffic | store evictions and memory | eviction policy evicts sessions (Lab 4) |
| logged out after a fixed time despite activity | session `created` vs now | absolute timeout working as designed, or a cookie `Max-Age` shorter than the server's timeout |
| cookie set but never sent | DevTools blocked reason on the request | `SameSite` on a cross-site request; third-party cookie blocking; `Secure` on HTTP; wrong `Domain` or `Path` (Lab 2) |
| cookie expires immediately | the `Expires` date in the response vs the client's clock | server clock wrong, or a date format error; prefer `Max-Age`, which is relative |
| `403` on form submit | CSRF reason and session age | token from before login; cached page with a shared token; missing header (Lab 3) |
| `431` or proxy errors for some users | request header size | accumulated or oversized cookies |

### Guarantees and non-guarantees

This toolkit locates the failing link. It does not, by itself, explain browser policy changes you have not read about, and DevTools shows only what happens in the browser in front of you, which may differ from a user's browser, version or privacy settings. Ask affected users for their browser and version, and reproduce in that browser.

### Failure and diagnosis

The commonest diagnostic failure is having no miss reasons at all: the middleware treats every failure as "anonymous" and logs nothing, so a Redis eviction and a browser policy change look identical. The second is logging raw session IDs to diagnose the first, which turns the log system into a session-theft target. Build the reason codes and fingerprints in from the start.

### Backend connection

Make session failures observable: miss reasons, CSRF rejection reasons, per-instance counts, store evictions and logins per minute, each as a metric with an alert. Add an end-to-end test that logs in once and makes requests that are guaranteed to land on several instances. These cost an afternoon and turn the most common class of "it logged me out" tickets into a dashboard lookup.

---

## 19.9 Placement-Round Topics: Session-Tracking Techniques

> **Placement-round focus:** This section is revision material for campus placement interviews, university web-technology courses (servlets and JSP), and written tests. It is not part of the main flow. What carries into backend work: the techniques are the same four places state can live (Section 19.1), and their weaknesses are the attacks of Sections 19.5–19.6.

### The four classic techniques

Web-technology courses name four ways to "track a session", meaning to link the requests of one user:

| Technique | How it works | Advantages | Disadvantages |
|---|---|---|---|
| **Cookies** | the server sends `Set-Cookie`; the browser returns the value on each request | automatic; invisible in URLs; works on every page | the user can disable or delete cookies; about 4 KB per cookie; CSRF (Section 19.6) |
| **URL rewriting** | the session ID is appended to every link the server generates: `/cart;jsessionid=9C1F…` or `/cart?sid=9C1F…` | works with cookies disabled | every link must be rewritten; IDs leak through history, logs, bookmarks and `Referer`; enables session fixation (Section 19.5) |
| **Hidden form fields** | the server puts `<input type="hidden" name="sid" value="9C1F…">` in each form; the next submission carries it back | needs no cookies | works only for form submissions, not links; the value is visible in the page source |
| **Server-side session object** (`HttpSession` in Java) | an API that keeps per-user data on the server, identified by a cookie, or by URL rewriting if cookies are refused | data stays on the server; simple API | needs one of the other three to carry the ID; server memory or a store (Section 19.4) |

A fifth item sometimes listed is **HTTP authentication** (Basic), in which the browser resends credentials on every request; it identifies a user but is not a session (and is ambient, Section 19.6).

Modern practice keeps only cookies and server-side session objects. URL rewriting and hidden fields are taught because they show why cookies won, and because their leaks are the textbook causes of hijacking and fixation.

### Cookie facts that tests ask

- **Persistent versus non-persistent cookies**: a cookie with `Max-Age` or `Expires` is persistent and survives a browser restart; one without them is non-persistent and is discarded when the browser session ends (Section 19.2; browsers that restore sessions may keep it).
- **Limits**: RFC 6265 asks browsers to support at least 4,096 bytes per cookie, at least 50 cookies per domain, and at least 3,000 cookies in total. These are minimums for browsers, not a promise that larger cookies work.
- **Cookies versus sessions**: a cookie is client-side storage that the browser sends back; a session is server-side state that a cookie (usually) names. The common exam answer, "cookies are on the client, sessions are on the server", is correct as far as it goes; add that the session's identifier usually travels in a cookie, and that signed-cookie sessions (Section 19.4) blur the line.

> **Java lens:** The Servlet API exposes these techniques directly. `request.getSession()` returns the current `HttpSession`, creating one if none exists; `request.getSession(false)` returns `null` instead of creating. `session.setAttribute("cart", cart)` and `getAttribute("cart")` store and read values; `session.setMaxInactiveInterval(1800)` sets the idle timeout in **seconds**, while `<session-timeout>30</session-timeout>` in `web.xml` sets it in **minutes**; `session.invalidate()` ends the session. `request.changeSessionId()` (Servlet 3.1) rotates the ID, the fixation defence of Section 19.5. `response.encodeURL(url)` adds `;jsessionid=…` to a URL when the browser did not return the cookie (URL rewriting). Cookies are created with `new Cookie(name, value)`, configured with `setMaxAge`, `setHttpOnly` and `setSecure`, and sent with `response.addCookie(cookie)`. The container names its session cookie `JSESSIONID` by default.

### Drill for this section

**C1. Name the session-tracking techniques and compare them.**
A strong answer must mention:
- Cookies, URL rewriting, hidden form fields, and the server-side session object (`HttpSession`) built on one of them.
- URL rewriting leaks IDs through history, logs and `Referer`, and enables fixation; hidden fields work only for forms.
- Cookies are automatic but can be disabled and are ambient (CSRF).
- Modern applications use a cookie carrying an ID for a server-side session, or a signed cookie.
*Common weak answer:* "Sessions are stored in the browser." The session's data is on the server; only its identifier is in the browser.

**C2. What is the difference between `getSession()` and `getSession(false)`, and how do you end and rotate a session in a servlet?**
A strong answer must mention:
- `getSession()` creates a session if none exists; `getSession(false)` returns `null` instead, which is what a check like "is this user logged in?" needs.
- `invalidate()` ends the session; `changeSessionId()` rotates the ID at login.
- Idle timeout through `setMaxInactiveInterval` (seconds) or `web.xml` (minutes).
*Common weak answer:* calling `getSession()` in a filter that only checks login status: it creates an empty session for every anonymous request, filling memory and the store.

---

## Chapter Summary

HTTP is **stateless** by design, so any request can go to any server; applications are not. **Session state** (who is logged in, how strongly, what step they reached) is kept apart from **resource state** (orders, accounts), which belongs in the database. Session state can live in the **client** (a signed or encrypted cookie), in **one server's memory**, in a **shared store** such as Redis, or in the **database**; in the last three, a random **session identifier** in a cookie links each request to it.

Once a cookie carries a credential, every attribute is a security control. `Secure` keeps it off plaintext connections, `HttpOnly` keeps page scripts from copying it, `SameSite` decides whether other sites can make the browser send it, host-only scope and the `__Host-` prefix decide which servers can receive and overwrite it, expiry bounds the life of a stolen copy, and an opaque or signed value prevents forgery. Responses that set cookies must never be cached by shared caches.

A **server-side session** holds a small record under a key derived from a **128-bit ID from a CSPRNG**, looked up in middleware on every request, with an **idle timeout** (sliding, via the store's TTL) and an **absolute timeout** (a creation time checked on each request), both enforced on the server. Because the state is on the server, **invalidation** at logout, password change or compromise takes effect on the next request. To scale beyond one instance, sessions go into a **shared store** that is replicated, persistent, exempt from eviction, and handled fail-closed; **sticky sessions** with in-memory state break on restarts, deploys and network changes; **stateless signed or encrypted cookies** need no store but give up revocation unless a per-user epoch or denylist brings some state back, and they make the signing key as valuable as every session at once.

A session ID is a **bearer credential**. **Hijacking** steals it, by sniffing, XSS, malware, leaky URLs and logs, or prediction, and each channel has its own defence; context signals such as network and user-agent changes trigger step-up authentication rather than hard rejections. **Fixation** plants a known ID before login; **rotating the ID at login** and every privilege change defeats it, together with accepting only server-issued IDs from `__Host-` cookies. **Logout** must delete the server record, expire the cookie with identical scope, clear cached data, and arrive as a protected `POST`.

**CSRF** exploits ambient cookies: another site makes the victim's browser send a state-changing request that the server cannot distinguish from a real one. It needs an attacker-triggerable request, ambient authentication, and fully predictable parameters; the defences remove one each. `SameSite=Lax` blocks cross-site `POST`s but not state-changing `GET`s, same-site attackers, `None` cookies, or on-site request forgery, so it is defence in depth; **synchronizer tokens**, **signed double-submit cookies** and **Origin / Fetch Metadata checks** close the gaps, and a permissive credentialed CORS policy or any XSS defeats them all. **Bearer tokens in an `Authorization` header** are not CSRF-prone because nothing is ambient, but script must hold them, so XSS can steal them; a token stored in a cookie is just a cookie. The choice between **stateful and stateless** authentication (revocation, freshness, scaling) is independent of the choice between **cookie and header** transport (CSRF versus XSS exposure), and Chapter 20 develops the token side.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| HTTP statelessness | Any request can be served by any server; intermediaries can retry and cache | Any memory of the user between requests |
| `Secure` | The cookie is not sent over plaintext HTTP | Safety at the endpoints; protection of the first plaintext request without HSTS |
| `HttpOnly` | Page scripts cannot read the cookie | Protection from XSS **using** the session in the page |
| `SameSite=Lax` | No cookie on cross-site `POST`, subresource or `fetch` requests | Protection of state-changing `GET`s; protection from same-site attackers |
| Host-only / `__Host-` | Only this host receives and can set the cookie | Protection from an attacker on the same host |
| Cookie `Max-Age` | The browser forgets the cookie at that time | That a copied cookie stops working; server-side expiry |
| Server-side session (128-bit random ID) | Client cannot read or forge state; immediate revocation | Safety of a stolen ID; consistency under whole-record concurrent writes |
| Idle + absolute timeouts (server-enforced) | Bounded lifetime for unattended and stolen sessions | Protection within those windows |
| Shared session store | Any instance can serve any request; deploys do not log users out | Survival of the store's own failure or eviction without replication and `noeviction` |
| Sticky sessions | Repeated requests usually reach the same instance | Correctness across restarts, network changes and scaling |
| Signed cookie | The client cannot alter the data without the key | Secrecy, revocation, freshness, protection from replay before `exp` |
| Encrypted (AEAD) cookie | The client can neither read nor alter the data | Revocation; protection from replay before `exp` |
| ID rotation at login | An ID known before login is useless after it | Protection of an ID stolen after login |
| Server-side logout | Every copy of the ID stops working | Logout of the identity provider's SSO session |
| Synchronizer / signed double-submit token | Unsafe requests came from a page able to read your content | Protection against XSS; protection of unsafe `GET`s |
| Origin / Fetch Metadata check | Unsafe requests were caused by an allowed origin (in browsers that send the headers) | Protection from same-origin attackers; correctness if compared with an internal `Host` |
| Bearer token in `Authorization` header | No CSRF: the browser never attaches it automatically | Protection from theft by XSS when script can read it |

---

## Interview Drill

**Q1. Why is HTTP stateless, and how do sessions add state on top of it?**
A strong answer must mention:
- Each HTTP request is self-contained; the protocol keeps no memory between requests, so any server or intermediary can handle any request and crashes lose no protocol state.
- Applications need continuity (login, carts), so they check credentials once and issue a short, revocable credential (a session ID or signed cookie) that later requests present.
- The browser returns it automatically through the `Cookie` header; the server maps it to state in memory, a shared store, or the database, or verifies the signed state itself.
- The protocol stays stateless; the application layers state on top.
*Common weak answer:* "HTTP is stateless because it uses TCP." TCP connections are stateful; statelessness is a property of HTTP's request semantics, and requests on one connection are still independent.

**Q2. Where can session state live, and what are the trade-offs?**
A strong answer must mention:
- Client (signed/encrypted cookie): no store, any instance; size limits, sent on every request, hard to revoke.
- One server's memory: fastest; lost on restart, needs stickiness, uneven load.
- Shared store (Redis): stateless instances, fast, built-in TTL; another highly available component and a round trip.
- Database: durable and queryable; load on the system of record.
- Keep resource state (carts that must persist, orders) in the database and session state small.
*Common weak answer:* "In the server's memory, it's fastest." It ignores restarts, deploys and multiple instances.

**Q3. What does each cookie attribute protect against?**
A strong answer must mention:
- `Secure`: sniffing on plaintext connections (with HSTS for the first request).
- `HttpOnly`: copying the cookie with XSS, not using the session from within the page.
- `SameSite`: CSRF from other sites; `Lax` still allows top-level `GET` navigations; same-site attackers are not blocked.
- Host-only scope and `__Host-`: sibling subdomains receiving or tossing the cookie; `Path` is not a security boundary.
- `Max-Age`/`Expires`: the lifetime of a stolen or forgotten cookie in the browser; the server must enforce its own expiry.
*Common weak answer:* "`HttpOnly` prevents XSS." It prevents one consequence of XSS, cookie theft.

**Q4. How does a server-side session work, from login to logout?**
A strong answer must mention:
- Login verifies credentials, generates a new random ID, stores a record (user, auth time and methods, creation time, CSRF token) keyed by a hash of the ID with a TTL, and returns `Set-Cookie` with secure attributes and `Cache-Control: no-store`.
- Each request: middleware reads the cookie, looks up the record, checks absolute expiry, attaches the principal to the request context, and extends the idle TTL occasionally.
- Logout deletes the record (all copies die), expires the cookie with the same scope, and clears cached data.
- Password change or compromise deletes all the user's sessions through a per-user index.
*Common weak answer:* "The session is stored in the cookie." That describes stateless cookie sessions, a different design with different revocation properties.

**Q5. How should a session ID be generated, and why does its entropy matter?**
A strong answer must mention:
- From a CSPRNG (the operating system's generator), never a general-purpose PRNG, counter, timestamp or hash of user data.
- At least 64 bits of entropy per OWASP; 128 bits in practice.
- Guessing arithmetic: with a million active sessions and 10,000 guesses per second, 32 bits falls in under a second, 128 bits never.
- Opaque (encodes nothing), stored as a hash on the server so a store leak does not yield usable IDs.
*Common weak answer:* "Use a UUID." Some UUID versions are time-based or not generated from a CSPRNG; what matters is the source and the number of random bits.

**Q6. What is the difference between an idle timeout and an absolute timeout?**
A strong answer must mention:
- Idle: ends the session after a period with no requests; implemented as a sliding TTL.
- Absolute: ends it a fixed time after creation regardless of activity; implemented as a creation time checked per request.
- Idle limits unattended browsers and abandoned stolen IDs; absolute limits a stolen ID kept alive by use (and sessions kept alive by polling).
- Both enforced on the server; `Max-Age` is only advice to the browser; "remember me" is a separate credential.
*Common weak answer:* "They're the same, just set the cookie expiry." Cookie expiry is not enforcement, and one timeout cannot do both jobs.

**Q7. Sticky sessions versus a shared session store versus signed cookies: which would you choose, and why?**
A strong answer must mention:
- Sticky + in-memory: no store, but sessions die with instances, deploys log users out, load is uneven, and affinity breaks with network changes and scaling.
- Shared store: stateless instances, immediate revocation; the store must be replicated, persistent, exempt from eviction and handled fail-closed.
- Signed/encrypted cookies: no store, but no revocation before expiry without an epoch or denylist, size limits, key rotation.
- Default to a shared store for multi-instance web applications; stickiness only as an optimisation.
*Common weak answer:* "Sticky sessions, they're simpler." Simple until the first deploy or instance failure logs users out.

**Q8. What is session hijacking? What is session fixation, and how do you prevent it?**
A strong answer must mention:
- Hijacking: stealing a valid session ID (sniffing, XSS, malware, URL and log leaks, prediction) and using it; defences per channel (`Secure`/HSTS, `HttpOnly`/CSP, no IDs in URLs, redaction, CSPRNG), plus timeouts, step-up, anomaly detection.
- Fixation: the attacker plants a known ID in the victim's browser (URL, cookie tossing, XSS), the victim logs in, and the ID becomes authenticated.
- Fix: rotate the ID at login and every privilege change; accept only server-issued IDs; cookies only, with `__Host-`.
- The timeline: attacker gets ID → plants it → victim logs in with the same ID → attacker uses it.
*Common weak answer:* "Bind the session to the IP address." That breaks mobile and NAT users and does not stop an attacker on the same network or device.

**Q9. What does "logout" actually do, and why is it hard with stateless cookies?**
A strong answer must mention:
- Delete the server-side record so every copy dies; expire the cookie with identical name, path and domain; `Clear-Site-Data` and `no-store` for cached pages; `POST` with CSRF protection.
- With stateless cookies there is no record: the browser forgets its copy but a stolen copy stays valid until `exp`.
- Compensations: short lifetimes, a per-user epoch, a denylist, each reintroducing state.
- SSO: local logout does not end the identity provider's session (Chapter 20).
*Common weak answer:* "Delete the cookie." That only affects the browser that clicked logout.

**Q10. What is CSRF, and why do cookies make it possible?**
A strong answer must mention:
- Another site makes the victim's browser send a state-changing request to a site where the victim is logged in.
- Cookies are ambient: the browser attaches them whatever site caused the request; the same-origin policy blocks reading responses, not sending requests; simple form posts need no CORS preflight.
- The attacker cannot read the response, so CSRF targets actions, not data.
- Preconditions: triggerable request, ambient authentication, predictable parameters.
*Common weak answer:* "CSRF is when an attacker steals your cookie." The attacker never sees the cookie; the browser uses it on their behalf.

**Q11. How do you prevent CSRF? When is `SameSite` enough, and when is it not?**
A strong answer must mention:
- Defences: `SameSite=Lax`/`Strict`, synchronizer tokens, signed double-submit cookies, `Origin`/`Referer`/`Sec-Fetch-Site` checks, required custom headers with strict CORS.
- `Lax` is enough only if no state change is reachable by `GET`, no host on the registrable domain is untrusted, the attribute is explicit, the cookie never needs `None`, and no page of your own can be made to send requests.
- Otherwise add tokens or origin checks; `Strict` costs logged-out arrivals from links.
- XSS or a permissive credentialed CORS policy defeats all CSRF defences.
*Common weak answer:* "Use `SameSite` and you're done." It misses `GET` side effects and same-site attackers.

**Q12. Is an API that uses bearer tokens vulnerable to CSRF? What risk does it trade in?**
A strong answer must mention:
- Not if the token travels only in the `Authorization` header: the browser never adds it, and the attacker's page cannot read the token or add headers to a form post.
- The trade: script must hold the token, so XSS can copy it and use it anywhere until it expires, whereas an `HttpOnly` cookie can only be used while the page is open.
- Vulnerable again if the token is in a cookie, the API also accepts cookies, or authentication is Basic, client certificates, Windows sign-in or network position.
- Browser applications therefore often prefer `HttpOnly` cookies with CSRF defences, or a BFF.
*Common weak answer:* "JWTs prevent CSRF." The format is irrelevant; the transport decides.

**Q13. Server-side sessions versus tokens: what is the difference?**
A strong answer must mention:
- Two axes: where the state lives (server lookup versus self-contained signed token) and how it travels (cookie versus header).
- The state axis decides revocation, freshness and scaling; the transport axis decides CSRF versus XSS exposure.
- Sessions: immediate revocation, current data, a store to run. Self-contained tokens: verifiable by any service with the key, no revocation before expiry without state.
- Browsers talking to their own backend: sessions in `HttpOnly` cookies; mobile, third-party and service clients: header tokens (Chapter 20).
*Common weak answer:* "Tokens are stateless so they scale better." Refresh tokens and revocation reintroduce state, and a session lookup in Redis is rarely the bottleneck.

**Q14. Compare the synchronizer token pattern with the double-submit cookie. What is wrong with naive double-submit?**
A strong answer must mention:
- Synchronizer: a random token stored in the server-side session, embedded in pages, compared (constant time) on unsafe requests; the attacker cannot read your pages.
- Double-submit: the token is in a cookie and copied by the page into a header or field; the server compares the two, with no storage.
- Naive double-submit fails if an attacker can write cookies for your domain (sibling subdomain, plaintext page): they set both values.
- Fix: sign the token and bind it to the session (HMAC with the session ID), and use a `__Host-` cookie; mask tokens against BREACH.
*Common weak answer:* "Double-submit is just as secure and simpler." Only in its signed, session-bound form.

**Q15. What is login CSRF, and how do you prevent it?**
A strong answer must mention:
- The attacker's page submits your login form with the attacker's credentials; the victim is silently logged in to the attacker's account.
- The victim's subsequent activity (searches, saved cards, uploads) lands in an account the attacker can read.
- Defences: a CSRF token held in an anonymous pre-session and rotated at login, or an origin check on the login endpoint; `SameSite` does not help because the victim has no session cookie yet.
- The OAuth `state` parameter protects the callback against the same attack (Chapter 20, Section 20.7).
*Common weak answer:* "Login doesn't need CSRF protection because the user isn't logged in yet." That is exactly what makes the attack possible.

---

## Scenario Exercises

1. An application moves from one instance to three behind a round-robin load balancer. Support then reports that users are "kicked to the login page" a few times an hour, more often on phones. Give three hypotheses and the evidence that separates them.
2. A security review finds that an application keeps sessions in a signed, unencrypted cookie containing `{"uid":81,"role":"admin"}`, valid for 30 days. An administrator was dismissed yesterday and their account disabled. What is the exposure right now, what can you do today, and how should the design change?
3. Marketing wants to host customer-built landing pages at `<name>.pages.shop.example`. The application runs at `app.shop.example`, with a session cookie `sid` set with `Domain=shop.example` and `SameSite=Lax`. Which attacks does this open, and what should change before launch?
4. A team building a single-page application says, "our API only accepts JSON, so we disabled CSRF protection." The API authenticates with a session cookie. Under which conditions are they right, and which checks would you add so that they stay right?
5. On a shared library computer, a user logs out of a shop. The next user presses Back and sees the previous user's order history. Separately, a penetration tester shows that a session cookie copied before logout still works afterwards. Explain both findings and fix them.
6. During a planned failover of the Redis session store, the application logs connection errors for 30 seconds. Afterwards, an audit finds that several admin pages returned `200` to requests without valid sessions during that window. Explain what went wrong and how the session middleware should behave.

---

## Diagnosis Lab

**Lab 1: Users logged out randomly behind a load balancer**
- *Symptom:* After the application moved from one instance to an auto-scaled group of three to six, about 1 in 20 page loads shows the login page to a logged-in user. The rate rises when the group scales out.
- *Hypotheses:* (a) sessions in instance memory, with stickiness not working; (b) instances using different signing keys; (c) the session store evicting sessions.
- *Evidence:* The `curl` loop of Section 19.8 with one cookie jar returns `200` only from the instance that handled the login and `401` from the others. Miss logs on the other instances say `reason=unknown_id`, not `bad_signature`, which rules out (b); the store is not involved, which rules out (c). The load balancer has cookie-based stickiness enabled, but a CDN sits in front of it and forwards only an allowlist of cookies, which contains `__Host-sid` and not the load balancer's affinity cookie. With one instance, stickiness had never mattered.
- *Root cause:* (a): sessions live in each instance's memory (the framework's default), and stickiness never worked because the CDN stripped the affinity cookie. Scale-out made it worse by adding instances that knew no sessions.
- *Fix:* Move sessions to a replicated Redis store with `noeviction` (Section 19.4), making instances stateless. Leave stickiness off, or treat it as optional.
- *Prevention:* An end-to-end test that logs in and then makes requests reaching every instance; per-instance session-miss metrics with an alert.

**Lab 2: Cookie not sent on cross-site requests**
- *Symptom:* A shop's single-page application at `https://shop.example` calls its API at `https://api.shop-platform.net` with `credentials: "include"`. After a browser update, many users see "please log in" on every API call, although login succeeds. Users of one browser are all affected; users of another are not.
- *Hypotheses:* (a) the session cookie is not stored; (b) it is stored but not sent; (c) the API rejects the session.
- *Evidence:* In DevTools on an affected browser, the login response to `api.shop-platform.net` carries `Set-Cookie: sid=…; Secure; HttpOnly; SameSite=None`. The Application panel shows the cookie stored, partitioned or blocked depending on the browser. On the failing API call, the request's Cookies tab lists `sid` as blocked, with a reason pointing to third-party cookie restrictions. The page's site is `shop.example` and the API's site is `shop-platform.net`: different registrable domains, so every API call is a cross-site request and the API's cookie is a third-party cookie. The API's logs show `reason=no_cookie` for those users.
- *Root cause:* (b): the design depended on a third-party cookie, which the updated browser blocks by default. `SameSite=None` allowed it in principle; the browser's third-party cookie policy overrides that.
- *Fix:* Make the credential first-party: serve the API at `api.shop.example` (same site), or route API calls through the page's own origin (`https://shop.example/api/…` proxied to the platform, or a BFF, Chapter 20, Section 20.6). Then return the cookie to `SameSite=Lax` and add CSRF defences for the cookie-authenticated API (Section 19.6).
- *Prevention:* Treat any design that needs an unpartitioned third-party cookie as fragile (Section 19.2); test login flows in every major browser with default privacy settings.

**Lab 3: CSRF token mismatch**
- *Symptom:* After the CDN configuration was changed to cache "marketing pages", about 3% of newsletter sign-ups and some contact-form submissions fail with `403`. Logged-in users who submit the same forms from their account pages are unaffected.
- *Hypotheses:* (a) tokens from forms loaded before login, invalidated by rotation; (b) expired sessions; (c) a cache serving pages that contain someone else's token.
- *Evidence:* Rejection logs show `reason=mismatch` with session ages of minutes to hours, which argues against (a), where rejections cluster seconds after login. Logging a fingerprint of the **submitted** token shows that hundreds of rejected requests in one ten-minute window carry the **same** token, which belongs to one pre-session. The CDN logs show `HIT` for the marketing page paths, whose HTML includes the newsletter form with a hidden `csrf` field; the page's response had `Cache-Control: public, max-age=600`, overriding the framework's default.
- *Root cause:* (c): the CDN cached a page with an embedded per-session token for ten minutes, so every visitor in that window received the first visitor's token, which did not match their own session's token.
- *Fix:* Mark pages containing CSRF tokens `Cache-Control: private, no-store`, or keep marketing pages cacheable and load the form's token from an uncached endpoint by script. Configure the CDN to bypass requests that carry session cookies.
- *Prevention:* A test that fetches each cacheable page twice with different sessions and checks that no per-session value appears in both; CSRF rejection metrics by reason with an alert on spikes.

**Lab 4: Logouts at peak traffic (session eviction)**
- *Symptom:* During a sale, users report being logged out mid-checkout. The login rate triples, and the login path's CPU rises sharply. Outside peaks, nothing is wrong.
- *Hypotheses:* (a) the absolute timeout; (b) store failover; (c) the store evicting session keys under memory pressure.
- *Evidence:* Miss logs show `reason=unknown_id` for sessions only minutes old, which rules out (a). The store had no failover event. `INFO stats` shows `evicted_keys` rising fast during the peak; `CONFIG GET maxmemory-policy` returns `allkeys-lru`. The same Redis instance holds the product cache, which grew during the sale; sessions of users who paused at checkout were the least recently used keys.
- *Root cause:* (c): sessions shared an instance with a cache whose eviction policy treats every key as disposable. The login surge was a consequence, made expensive by the deliberately slow password check (Chapter 20, Section 20.2).
- *Fix:* Move sessions to a separate instance with `noeviction`, persistence and replication; keep `allkeys-lru` for the cache instance only (Chapter 21, Section 21.7).
- *Prevention:* Alerts on evictions and memory use for the session store; capacity planning from session count times record size (Section 19.4).

---

## Solution Sketches

1. (a) Sessions in instance memory without stickiness: check per-instance miss reasons (`unknown_id` only on instances that did not create the session) and the `curl` loop. (b) Stickiness by IP hash, broken when phones change networks, which fits "more often on phones": check whether misses coincide with a change of client IP for the same session fingerprint. (c) Instances with different signing keys (if sessions are cookie-based): misses logged as `bad_signature`. The lasting fix for all three is a shared store, or one configured keyring.
2. The signed cookie is readable (the role is visible) and unrevocable: the dismissed administrator's cookies remain valid for up to 30 days, and disabling the account does nothing unless the server checks the account on each request. Today: rotate the signing key (logging everyone out) if no per-user check exists, or add an immediate check of account status and role from the database per request. Redesign: server-side sessions in a shared store, or at least an epoch check with short-lived cookies; never put authorisation facts such as roles in a long-lived client-side value; load roles per request (Section 19.3).
3. Every customer page is same-site with the application, and that has three consequences. The `Domain=shop.example` session cookie is sent to every customer page's host, so any logging or code on that host sees it. A customer page that can run script can set cookies for `shop.example`: cookie tossing and session fixation (Sections 19.2 and 19.5). And its requests to the application are same-site, so `SameSite=Lax` gives no CSRF protection against it. Fix: host customer pages on a separate registrable domain (`shop-pages.example`); make the session cookie host-only with `__Host-`; add CSRF tokens and an origin check that allows only `https://app.shop.example`.
4. They are right only if every state-changing endpoint rejects any `Content-Type` other than `application/json` (so a cross-site form or `text/plain` request fails), the CORS policy never approves other origins for credentialed requests, and no state change is reachable by `GET`. Add: strict `Content-Type` enforcement, a required custom header carrying a token (signed double-submit or synchronizer), an origin and Fetch Metadata check, an explicit `SameSite=Lax`, and an automated test that sends cross-site form posts to every route.
5. The Back button showed a cached copy: authenticated pages lacked `Cache-Control: no-store`, so the browser kept them; add `no-store` on authenticated responses and `Clear-Site-Data: "cookies", "storage", "cache"` on logout. The copied cookie still works because logout cleared only the browser's cookie and did not delete the server-side session (or the sessions are stateless): delete the record at logout, and with stateless cookies add an epoch or denylist (Section 19.5).
6. The middleware failed open: a store error was treated as "no session information, continue", and some admin routes relied on the session lookup to deny access rather than requiring an authenticated principal. Correct behaviour: a lookup error means the request is unauthenticated (return `401`, or `503` for pages that cannot work without the session); authorisation denies by default when no principal exists (Chapter 20, Section 20.9). Make the store replicated with automatic failover, use client timeouts and retries with backoff during failover (Chapter 24), and test the store-outage path.

---

**END OF CHAPTER 19**
