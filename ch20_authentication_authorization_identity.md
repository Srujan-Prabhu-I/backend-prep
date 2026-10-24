# Chapter 20: Authentication, Authorization, and Identity

## Where We Are

Chapter 8 gave us the cryptographic tools: hashes, MACs, signatures, key exchange and certificates, and showed how TLS uses them to authenticate a **server** to a client (Sections 8.4–8.10). Chapter 9 showed what browsers can store and send (Section 9.5), Chapter 13 placed authentication and authorisation checks in an API and named broken object-level authorisation (Section 13.13), and Chapter 19 built sessions: a random identifier in a cookie that stands in for a user who has already proved who they are (Sections 19.1–19.4), protected against hijacking, fixation and cross-site request forgery (Sections 19.5–19.6).

This chapter is about the proving, and about what happens after it. How does a backend check that a user is who they claim to be, without storing anything that would let an attacker who steals the database do the same? How do second factors work? What is inside a token such as a JWT, and how is it validated? How does "Sign in with Google" work without the application ever seeing the user's Google password? And once a user is known, how does the backend decide what they may do, on every request, for every object? The chapter ends by stepping back: a synthesis of application security across the book, organised around the OWASP Top 10.

**After this chapter you can:**
- distinguish identity, authentication, authorisation and federation precisely;
- store passwords correctly with salts and slow key-derivation functions, and defend login against brute force, credential stuffing and account enumeration;
- design login, logout, password-reset and email-verification flows that do not leak or weaken accounts;
- explain how TOTP codes and WebAuthn passkeys work, and why passkeys resist phishing;
- compare opaque and self-contained tokens, decode and validate a JWT, and explain algorithm-confusion and revocation problems;
- design refresh-token rotation with reuse detection, and choose where a browser application keeps its tokens;
- trace the OAuth 2.0 authorization code flow with PKCE parameter by parameter, and the client credentials flow;
- explain what OpenID Connect adds to OAuth: ID tokens, discovery, JWKS and single sign-on;
- choose between ACLs, RBAC, ABAC and ReBAC, and prevent IDOR/BOLA by scoping queries;
- authenticate services to each other with API keys, mTLS, workload identity and token exchange, and protect credentials from SSRF;
- manage secrets and their rotation;
- threat-model a feature with STRIDE and map the OWASP Top 10 to the chapters of this book that defend against each risk;
- diagnose 401 versus 403 errors, clock-skew JWT failures, token-expiry storms and IDOR privilege escalation.

**Interview questions this chapter prepares you for:**
1. What is the difference between authentication and authorisation? What is identity federation?
2. How should passwords be stored? Why is SHA-256 wrong for passwords?
3. How do you defend a login endpoint against brute force and credential stuffing without locking out real users?
4. How should a password-reset flow work?
5. How does TOTP work? How do passkeys (WebAuthn) work, and why do they resist phishing?
6. What is a JWT? How is it validated, and what are its pitfalls?
7. Sessions versus JWTs: what are the trade-offs?
8. What are refresh tokens, and what is refresh-token rotation?
9. Explain the OAuth 2.0 authorization code flow with PKCE.
10. Why was the OAuth implicit flow abandoned?
11. What is the difference between OAuth 2.0 and OpenID Connect? Between an ID token and an access token?
12. RBAC versus ABAC versus ReBAC: when would you use each?
13. What is IDOR (BOLA), and how do you prevent it?
14. How do services authenticate to each other?
15. How should secrets be managed?
16. What is threat modelling? Walk through the OWASP Top 10.

**Scope.** Here: identity concepts, password storage and login defences, account flows, MFA, tokens and JWTs, refresh tokens, OAuth 2.0, OpenID Connect, SAML in brief, authorisation models and object-level authorisation, service identity, secrets management, a security synthesis, and a placement-round section on Kerberos, LDAP and classic access-control models (Section 20.13). Recalled, not re-taught: hashes, MACs, signatures and certificates (Chapter 8, Sections 8.4–8.8), mTLS and workload identity (Chapter 8, Section 8.10), cookie attributes and token storage in the browser (Chapter 9, Section 9.5), XSS (Chapter 9, Section 9.8), SSRF and the metadata service (Chapter 13, Section 13.14), sessions, session fixation, logout and CSRF (Chapter 19, Sections 19.1–19.7), configuration and secret delivery (Chapter 12, Section 12.9). Elsewhere: rate-limiting algorithms are Chapter 24; audit-log pipelines and security monitoring in production are Chapter 26.

---

## 20.1 Identity, Authentication, Authorisation

### Definitions

- An **identity** is the set of attributes by which a system knows an entity: an account ID, an email address, a name, roles, the organisation it belongs to. The entity may be a person, a service or a device.
- A **principal** (or **subject**) is the entity that is acting in a request, as the system represents it after authentication: "user 81", "the billing service".
- **Authentication** (often written **authN**) is verifying that a party really is the identity it claims to be, by checking something it presents. **Authorisation** (**authZ**) is deciding whether an authenticated principal is allowed to perform a specific action on a specific resource.
- An **authentication factor** is a kind of evidence: something you **know** (a password, a PIN), something you **have** (a phone, a security key, a private key), or something you **are** (a fingerprint, a face). **Multi-factor authentication** (MFA) requires evidence of two or more different kinds.
- An **identity provider** (IdP) is a system that authenticates users and vouches for their identity to other systems. **Identity federation** is the arrangement in which one system (a **relying party**) accepts authentication performed by another (the IdP) instead of doing it itself: "Sign in with Google", or a company's single sign-on (SSO) portal.
- **Accounting** (or **auditing**) is recording who did what, so that actions can be reviewed and attributed later. Together with authentication and authorisation it is sometimes called **AAA**.

### Why it matters

The two words are confused constantly, and the confusion produces real bugs. Authentication answers "who are you?" and happens once per session or per token; authorisation answers "may you do **this**?" and must happen on **every** request, for every object. A system that authenticates perfectly but checks authorisation only on the screen that lists a user's own orders, and not on the endpoint that fetches an order by ID, lets any logged-in user read any order: broken object-level authorisation, the risk that heads the OWASP API Security Top 10 (Section 20.9). The HTTP status codes encode the same distinction (Chapter 7, Section 7.5): **401 Unauthorized** means "not authenticated" (send credentials), **403 Forbidden** means "authenticated, but not allowed".

### Mechanism: the request pipeline

```text
 request ──► 1. AUTHENTICATE                   2. BUILD PRINCIPAL               3. AUTHORISE (per action, per object)
            credential present?                  user 81, tenant 7,               may user 81 cancel order 9001?
            session cookie / bearer token /      roles {customer},                - is 9001 in tenant 7?
            client certificate / API key         MFA at 10:02                     - does user 81 own it?
            valid? not expired? not revoked?                                       - is cancelling allowed in its state?
                 │ no                                                                   │ no
                 ▼                                                                      ▼
            401 Unauthorized                                                       403 Forbidden (or 404, to avoid
            + WWW-Authenticate                                                     revealing that 9001 exists)
                                                                      4. ACT, 5. AUDIT (who, what, when, outcome)
```

Authentication is usually done once, in middleware (Chapter 10, Section 10.6), and its result (the principal) is attached to the request context (Chapter 11, Section 11.6). Authorisation needs the resource, so most of it happens in the application or the query that loads the resource (Section 20.9). The principal must come **from the authenticated context, never from the request body** (Chapter 12, Section 12.6): a body field `"userId": 81` is a claim, not an identity.

### Concrete: the same request, three outcomes

```http
DELETE /v1/orders/9001 HTTP/1.1
Host: api.shop.example
Authorization: Bearer eyJhbGciOiJFUzI1NiIs…
```

```text
 token missing or expired                     → 401, WWW-Authenticate: Bearer error="invalid_token"
 token valid, user 81, order 9001 is user 44's → 404 (or 403); log "authz denied user=81 order=9001"
 token valid, user 81 owns order 9001          → 204; audit "user=81 cancelled order=9001"
```

### Guarantees and non-guarantees

Authentication guarantees, to the strength of its factors, that the principal is who it claims to be at the time of authentication. It does not guarantee that the same person is still at the keyboard, that a session or token has not been stolen since, or anything about what the principal may do. Authorisation guarantees that a specific action on a specific resource was allowed by policy; it guarantees nothing if it is not checked on every path that reaches the resource.

### Failure and diagnosis

- **Authorisation missing on one path.** A list endpoint filters by owner but the detail, update or export endpoint does not. Evidence: requests succeeding for objects belonging to other users or tenants in the audit log; a test that requests another user's object. Fix: Section 20.9.
- **Identity from the request body.** An endpoint trusts `userId` or `tenantId` from JSON or a query parameter. Evidence: code review; requests whose body user differs from the token's user. Fix: take identity only from the authenticated context.
- **401 and 403 confused.** Clients retry login on a 403, or show "forbidden" for an expired token. Diagnosis Lab 1.

### Backend connection

Authenticate once per request in one place, build a principal, and pass it down explicitly. Authorise every action against the specific resource, as close to the data as possible. Return 401 when credentials are missing or invalid and 403 (or 404) when they are valid but insufficient, and audit both outcomes.

---

## 20.2 Password Storage and Login Defences

### Definitions

- **Password storage** is how a system keeps what it needs to verify passwords later. Correctly done, it stores a **verifier** derived from the password, never the password itself, and never anything from which the password can be recovered cheaply.
- A **salt** is a random value, unique per password, stored next to the verifier and mixed into the computation, so that equal passwords produce different verifiers (Chapter 8, Section 8.2 introduced the term).
- A **key-derivation function** (KDF) for passwords is a deliberately **slow**, tunable function that turns a password and a salt into a verifier. **Memory-hard** KDFs also require a large amount of memory per computation. The recommended ones are **Argon2id**, **scrypt** and **bcrypt** (with PBKDF2 where compliance requires it).
- A **pepper** is a secret value, the same for all passwords and stored **outside** the database (in a secret manager or a hardware security module), mixed into the computation so that a stolen database alone is not enough to test guesses.
- **Offline attack**: the attacker has the stored verifiers (from a breach or a backup) and tests guesses on their own hardware, as fast as it allows. **Online attack**: the attacker tests guesses against the live login endpoint, limited by its defences.
- **Brute force** tries many passwords against one account. **Credential stuffing** tries username–password pairs leaked from **other** sites, exploiting password reuse. **Password spraying** tries a few very common passwords against many accounts, staying below per-account limits.
- **Account enumeration** is learning which accounts exist from differences in a system's responses (messages, status codes, timing).

### Why it exists

Databases leak: through SQL injection (Chapter 18, Section 18.7), stolen backups, over-privileged staff, misconfigured storage. When they do, every stored password is exposed to an offline attack, and because people reuse passwords, a leak from one site compromises accounts on others. Password storage is designed for the day the database is stolen: the goal is that, on that day, each guess costs the attacker so much that most passwords stay safe, and no two users with the same password are cracked together.

### Concrete: the evolution of password storage

| Stored | Example for password `autumn42` | Attack it defeats | Attack that still works |
|---|---|---|---|
| Plaintext | `autumn42` | none | reading the database |
| Unsalted fast hash (MD5, SHA-1, SHA-256) | `SHA256(autumn42)` | casual reading | **precomputed tables** (rainbow tables) and lookup sites map common hashes back instantly; equal passwords have equal hashes, so one crack cracks every user who chose it; GPUs compute billions of SHA-256 hashes per second |
| Salted fast hash | `SHA256(salt ‖ autumn42)`, salt stored | precomputed tables; identical-password grouping (each salt needs its own attack) | **fast guessing**: one GPU still tests billions of candidates per second per salt, so weak and medium passwords fall in seconds to hours |
| Salted slow KDF (Argon2id, scrypt, bcrypt) | `$argon2id$v=19$m=19456,t=2,p=1$<salt>$<hash>` | fast guessing: each guess costs tens of milliseconds and, for memory-hard KDFs, tens of megabytes, which GPUs and custom hardware cannot parallelise cheaply | guesses of very weak or breached passwords, still feasible but slow; online attacks |
| Slow KDF + pepper | as above, with a secret key mixed in (for example an HMAC of the password with the pepper before hashing) | offline attack with the database **alone** | attackers who also obtain the pepper |

Why SHA-256 is wrong for passwords (Chapter 8, Section 8.4): it is designed to be **fast**, which is a virtue for checking file integrity and a disaster for passwords, whose problem is not collision resistance but the small space of human choices. A password of eight lower-case letters and digits has about 36⁸ ≈ 2.8 × 10¹² possibilities. At roughly 10¹⁰ SHA-256 guesses per second on one modern GPU (an order of magnitude; the exact rate depends on hardware), the whole space is exhausted in about five minutes. With a KDF tuned to 50 ms per guess on the server, and the attacker achieving, say, 1,000 guesses per second per GPU against a memory-hard function, the same space takes about 90 years per salt.

### Mechanism: what a modern verifier looks like

Libraries store the algorithm, its parameters, the salt and the hash in one self-describing string, so that parameters can be raised later without breaking old records:

```text
 $argon2id$v=19$m=19456,t=2,p=1$c2FsdHNhbHRzYWx0c2FsdA$k7Pq…(32 bytes base64)
   │        │    │       │   │   │                       └── derived hash
   │        │    │       │   │   └── salt (16 random bytes, base64)
   │        │    │       │   └── parallelism (lanes)
   │        │    │       └── t: iterations (passes over memory)
   │        │    └── m: memory in KiB (19 MiB)
   │        └── version
   └── algorithm
```

```pseudo
function register(password):
    salt = csprng(16 bytes)
    record = argon2id(password, salt, m = 19 MiB, t = 2, p = 1)   // library returns the full string
    store(user, record)

function verify(password, record):
    params, salt, expected = parse(record)
    actual = argon2id(password, salt, params)
    if not constantTimeEquals(actual, expected): return false        // Chapter 8, Section 8.5
    if params weaker than current policy:                             // upgrade on successful login
        store(user, argon2id(password, newSalt(), currentParams))
    return true
```

Parameter guidance changes as hardware improves; OWASP's password-storage guidance at the time of writing suggests, as minimums, Argon2id with 19 MiB of memory, 2 iterations and parallelism 1; scrypt with N = 2¹⁷, r = 8, p = 1; bcrypt with a work factor of 10 or more; or PBKDF2-HMAC-SHA-256 with 600,000 iterations where FIPS compliance requires it. Tune so that one verification takes tens of milliseconds on your servers, and remember that bcrypt uses only the first 72 bytes of its input.

The cost also lands on you: at 50 ms of CPU per verification, one core verifies 20 logins per second. A login burst (a mobile app releasing, a credential-stuffing attack) can saturate login servers, so login endpoints need their own capacity planning and rate limits, and verification should not run on the same worker pool as the rest of the API (a bulkhead, Chapter 24).

### Mechanism: online attacks and their defences

| Attack | Pattern in the logs | Defence | Cost of the defence |
|---|---|---|---|
| Brute force on one account | many failures for one username | per-account throttling: increasing delays, temporary lock after N failures, CAPTCHA | **lockout as denial of service**: an attacker can lock out real users on purpose; prefer delays and step-up challenges over hard locks |
| Credential stuffing | many usernames, mostly one attempt each, from many IPs, with a high share of "unknown user" and a small share of successes | screen passwords against breach corpora at registration and login; MFA; bot detection; per-IP and per-network rate limits; notify users of new-device logins | friction for legitimate users; IP limits hurt users behind shared NAT (Chapter 6, Section 6.5) |
| Password spraying | a handful of common passwords across many accounts | ban common and breached passwords; MFA; global anomaly detection on failure rates | as above |
| Account enumeration | the attacker compares responses for existing and non-existing users | identical responses and similar timing for "wrong password" and "no such user" on login, registration and reset | slightly less helpful messages |

**Breached-password screening** checks whether a new or entered password appears in known leaks. A common privacy-preserving design (used by the Have I Been Pwned "range" API) sends only the first five hex characters of the password's SHA-1 hash and receives every leaked hash with that prefix, so the full hash never leaves the server; the server checks for a match locally. NIST's digital identity guidance (SP 800-63B) recommends this kind of screening, favours length over composition rules ("one upper-case letter, one symbol"), and advises against forced periodic password changes.

**Timing and enumeration.** If the server verifies the KDF only for existing users, "no such user" returns in 1 ms and "wrong password" in 50 ms, which reveals which emails have accounts. Run a KDF verification against a dummy record when the user does not exist, so both paths take the same time.

### Guarantees and non-guarantees

Salted, slow KDFs guarantee that a stolen database yields passwords only at a high cost per guess, per user, and never all users with the same password at once. They do not protect weak or breached passwords from determined offline attackers, do not stop online attacks (that is rate limiting, screening and MFA), and do not help if passwords are logged, sent to analytics, or captured by phishing.

### Failure and diagnosis

- **Fast hashes in a legacy schema.** Evidence: 64-hex-character values in the password column, no salt column. Fix: wrap the old hashes (store `argon2id(sha256(password))` for existing users immediately, so the database is protected now), then replace them with direct Argon2id hashes on each user's next login.
- **Passwords in logs.** Request logging captures login bodies, or exception messages include parameters. Evidence: grep logs for the password field name. Fix: redaction at the logging layer (Chapter 26); treat as a breach.
- **Login servers saturated.** CPU at 100% on login, requests queuing. Evidence: KDF time × login rate exceeds cores (for example 400 attempts per second × 50 ms = 20 cores). Causes: a credential-stuffing wave or an app release. Fix: rate limits before verification, autoscaling the login pool, bot detection.
- **Mass lockouts.** Evidence: support tickets, lock events across many accounts from a few sources. Cause: hard lockout abused by an attacker. Fix: progressive delays and challenges.

### Backend connection

Never write your own password hashing: use a maintained library's Argon2id (or bcrypt) with current parameters, upgrade parameters on login, and keep a pepper in a secret manager if you can. Treat the login endpoint as a separate, attack-facing service with its own rate limits, capacity and monitoring, and make its responses identical for every kind of failure.

---

## 20.3 Account Flows: Login, Logout, Password Reset, Email Verification

### Definitions

- An **account flow** is a multi-step interaction that creates, proves, recovers or ends access to an account: registration, email verification, login, logout, password reset and change, MFA enrolment and recovery.
- A **single-use token** (or one-time token) is a random value sent through a separate channel (an email or SMS link) that proves control of that channel; it is valid once and for a short time.
- **Re-authentication** (step-up) is asking an already logged-in user to prove themselves again before a sensitive action.

### Why it matters

The password check is the most protected door of an account; the flows around it are often the weakest. Password reset turns "control of an email inbox" into "control of the account", so its token is as powerful as the password. Registration and reset forms are the easiest places to enumerate accounts. Logout that does not invalidate the server session leaves a usable credential behind (Chapter 19, Section 19.5).

### Mechanism: login

```text
 1. POST /login over HTTPS (credentials only in the body, never in the URL: URLs end up in logs and history)
 2. rate-limit by account and by source (Section 20.2), before the expensive KDF
 3. look up the user; verify the password (dummy verification if the user does not exist)
 4. if MFA is enrolled: issue a short-lived "password verified, MFA pending" state, ask for the second factor
 5. on success: create a NEW session (new ID, against fixation: Chapter 19, Section 19.5), record auth
    time and methods; reset failure counters
 6. respond identically for "no such user" and "wrong password": "Incorrect email or password"
 7. audit: success or failure, user, source, user agent; notify the user of a login from a new device
```

### Mechanism: password reset

```text
 user                         application                                   email
 "forgot password" (email) ─► 1. respond the SAME way whether or not the account exists:
                                 "If an account exists, we have sent a link."
                              2. if it exists: token = 32 random bytes (CSPRNG)
                                 store SHA-256(token), user_id, expires_at = now + 30 min, used = false
                              3. send https://shop.example/reset?token=<token> ─────────────────────► inbox
 opens link ────────────────► 4. hash the token, look it up: exists, not expired, not used?
 new password ──────────────► 5. set the new password hash (Section 20.2), mark the token used,
                                 delete ALL the user's sessions and other reset tokens,
                                 notify the user by email that the password changed
```

Rules that follow from the mechanism:

- **Store only a hash of the token**, like a session ID (Chapter 19, Section 19.3): a database leak must not hand out working reset links.
- **Short expiry, single use, invalidated by any later reset or login.**
- **Build the link from configuration, never from the request's `Host` header.** If the application builds `https://<Host>/reset?token=…` from the incoming request, a forged `Host` header can make the email point to a host the attacker controls (host-header poisoning), and the victim's click delivers the token there.
- **Keep the reset page from leaking the token**: no third-party scripts or images on it, and `Referrer-Policy: no-referrer` (Chapter 9, Section 9.9), since the token is in the URL.
- **Security questions are not a recovery factor**: their answers are guessable or public.

### Mechanism: email verification, logout, and sensitive changes

- **Email verification** uses the same single-use token design, proving control of the address before it is trusted for recovery or notifications. Until it is verified, do not let an unverified address be used to reset a password or to link to an existing account with the same email (a common account-takeover route in federated logins, Section 20.8).
- **Logout** deletes the server-side session and expires the cookie (Chapter 19, Section 19.5); with tokens, it revokes the refresh token (Section 20.6). "Log out of all devices" deletes all of the user's sessions and refresh tokens.
- **Changing email, password or MFA settings** requires re-authentication, notifies the **old** email address, and invalidates other sessions.

### Guarantees and non-guarantees

Well-designed flows guarantee that each step proves control of a factor or channel before granting access, that tokens are single-use and short-lived, and that no response reveals whether an account exists. They do not make email itself secure: whoever controls the inbox controls the account, which is why high-value accounts need MFA that reset alone cannot bypass, and recovery procedures with human review.

### Failure and diagnosis

- **Reset links that work forever or twice.** Evidence: tokens without expiry or a used flag in the schema. Fix: the rules above.
- **Account existence leaked by the reset or registration page.** Evidence: different messages or status codes for known and unknown emails. Fix: identical responses; for registration, send an email ("you already have an account") instead of an on-screen error.
- **Old sessions alive after a password change.** Evidence: session records created before the change still in use. Fix: invalidate all sessions on password change.

### Backend connection

Implement account flows once, in one module, reviewed as security code, with every token random, hashed at rest, short-lived and single-use. Make every response the same for existing and non-existing accounts, every link built from configured URLs, and every sensitive change protected by re-authentication and an email to the old address.

---

## 20.4 Multi-Factor Authentication: TOTP and WebAuthn

### Definitions

- **HOTP** (HMAC-based one-time password, RFC 4226) derives a short numeric code from a shared secret and a counter. **TOTP** (time-based one-time password, RFC 6238) is HOTP with the counter replaced by the current time divided into steps, usually 30 seconds. Authenticator apps implement TOTP.
- **WebAuthn** (Web Authentication, a W3C standard) is a browser API for authenticating with **public-key cryptography**: an **authenticator** (a phone, a laptop's secure enclave, a hardware security key) holds a private key per site and proves possession by signing a challenge. A **passkey** is a WebAuthn credential that can replace the password entirely, often synchronised across a user's devices by their platform.
- **Phishing resistance** is the property that a credential cannot be relayed to the real site by a fake site that tricks the user.
- **Recovery codes** are single-use backup codes, issued at MFA enrolment, for when the second factor is lost.

### Why it exists

Passwords fail in predictable ways: reused, guessed, phished, leaked. A second factor means a stolen password alone is not enough. But second factors differ greatly in what they stop. SMS codes can be intercepted (SIM-swap fraud, where an attacker persuades a carrier to move the victim's number). TOTP codes stop credential stuffing but can be typed into a convincing fake page and relayed in real time. WebAuthn binds the credential to the site's domain, so a fake site cannot use it at all.

### Mechanism: how a TOTP code is computed

At enrolment the server generates a random secret (for example 20 bytes), stores it encrypted, and shows it to the user as a QR code that the authenticator app scans. Then both sides compute codes independently:

```text
 secret K (shared), time step X = 30 s, digits d = 6
 1. T = floor(unix_time / 30)                       e.g. 1,761,300,123 / 30 → T = 58,710,004
 2. H = HMAC-SHA1(K, T as an 8-byte big-endian integer)            → 20 bytes  (Chapter 8, Section 8.5)
 3. dynamic truncation: offset = (last byte of H) AND 0x0F        → a number 0..15
                        take 4 bytes H[offset .. offset+3], clear the top bit → a 31-bit integer
 4. code = that integer mod 10^6, padded to 6 digits             → "492039"
```

The server accepts the code for the current step and usually one step on either side, to tolerate clock drift and typing time; it must also remember the last accepted step for each user, so that the same code cannot be used twice. TOTP codes are only six digits, so the verification endpoint needs a strict attempt limit: one million possibilities is small.

### Mechanism: WebAuthn registration and authentication

```text
 REGISTRATION (once per device)
 browser / authenticator                                   server (relying party, rpId = shop.example)
 ◄── challenge (random), rpId, user id ──────────────────
 authenticator creates a NEW key pair for (rpId, user);
 private key never leaves the authenticator
 ── public key, credential id, attestation (optional) ──► store {credential id, public key, sign count} for the user

 AUTHENTICATION (each login)
 ◄── challenge c (random, single use), rpId ──────────────
 browser builds clientData = {type: "webauthn.get", challenge: c, origin: "https://shop.example"}
   (the ORIGIN is filled in by the browser, not by the page)
 authenticator checks rpId against the origin's domain, asks for user presence / verification
   (touch, fingerprint, PIN), signs authenticatorData ‖ SHA-256(clientData) with the private key
 ── credential id, authenticatorData, clientData, signature ──►
                                                           verify: signature with the stored public key;
                                                           challenge = c (fresh, unused); origin and rpId are ours;
                                                           user-presence flag set; sign count not decreasing
```

Why this resists phishing: on a fake site `shop-example.login.test`, the browser puts the fake **origin** in `clientData`, and the authenticator has no key for that `rpId` at all, so nothing usable can be produced for the real site. There is no code for the user to retype into the wrong page. And because the server stores only **public** keys, a database breach yields nothing an attacker can log in with (contrast with password verifiers and TOTP secrets).

### Concrete: comparing second factors

| Factor | Stops credential stuffing | Stops real-time phishing | Server stores | Typical weakness |
|---|---|---|---|---|
| SMS code | yes | no | phone number | SIM swap, interception, delivery failures |
| Email code or link | yes | no | email | as strong as the inbox |
| TOTP app | yes | no (codes can be relayed) | shared secret (must be encrypted at rest) | secret theft from the server; phishing |
| Push approval | yes | partly (number matching helps) | device registration | "MFA fatigue": users approving repeated prompts |
| WebAuthn / passkey | yes | **yes** (origin-bound) | public key only | account recovery when all devices are lost |

### Guarantees and non-guarantees

TOTP guarantees that the user had the shared secret within the current time window; it does not guarantee the code was entered on your real site. WebAuthn guarantees that a private key bound to your domain signed your fresh challenge; it does not protect a session after login (a stolen session cookie bypasses all factors, Chapter 19, Section 19.5), and it moves the hard problem to **recovery**: whatever recovery path you offer (codes, support, email) becomes the weakest factor.

### Failure and diagnosis

- **TOTP rejected for some users.** Evidence: their codes would match a step or two away; device clocks are wrong, or the server's clock is (check NTP). Fix: allow one step of drift; tell users to enable automatic time.
- **Code reuse.** The same code accepted twice within its window. Evidence: two successful verifications with the same step. Fix: store the last used step.
- **MFA bypass through another path.** The API accepts password-only logins for "legacy clients", or reset disables MFA. Evidence: logins without second-factor records in the audit log. Fix: every path to a session enforces the same factor policy.

### Backend connection

Offer passkeys where you can, TOTP as the widely supported fallback, and treat SMS as the weakest option. Record which factors each session used (`amr`, "authentication methods references", as OpenID Connect calls it, Section 20.8) so that sensitive actions can demand a stronger one, and design the recovery path as carefully as the factors themselves.

---

## 20.5 Tokens and JWTs

### Definitions

- A **token**, in authentication, is a string a client presents instead of re-proving its identity on each request. A **bearer token** is one that grants access to whoever presents it, with no further proof (RFC 6750 defines its use in the `Authorization: Bearer` header).
- An **opaque token** is a random identifier with no readable content; the server (or an authorisation server) looks it up to learn what it means, exactly like a session ID (Chapter 19, Section 19.3). A **self-contained token** carries its own claims, protected by a signature, so the receiver can validate it without a lookup.
- A **JSON Web Token** (JWT, RFC 7519) is the standard self-contained token format: a set of **claims** in JSON, signed (as a JWS, RFC 7515) or, less commonly, encrypted (as a JWE).
- A **claim** is one statement in the token: `sub` (subject, the principal), `iss` (issuer), `aud` (audience: who the token is for), `exp` (expiry), `iat` (issued at), `nbf` (not before), `jti` (token ID), plus application claims such as `scope` or `roles`.
- **Token introspection** (RFC 7662) is asking the issuer whether a token is active and what it means; it is how opaque tokens are validated by services that did not issue them.

### Why it exists

A server-side session needs a lookup in a store shared by every service that wants to know the user. In a system of many services, or across organisations, that lookup becomes a bottleneck and a coupling: every API must reach the session store of the login service. A self-contained token lets any service validate a request locally, with only the issuer's public key, at the cost of revocation (below). Opaque tokens keep revocation simple at the cost of the lookup. Neither is better in general; Section 20.6 compares them with sessions.

### Concrete: a decoded JWT

A JWT is three base64url-encoded parts separated by dots: `header.payload.signature`.

```text
 eyJhbGciOiJFUzI1NiIsImtpZCI6ImsyMDI2LTEwIiwidHlwIjoiSldUIn0
 .eyJpc3MiOiJodHRwczovL2lkLnNob3AuZXhhbXBsZSIsInN1YiI6InVzcl84MSIsImF1ZCI6Im9yZGVycy1hcGkiLC…
 .MEUCIQC3…(signature)

 header  {"alg":"ES256","kid":"k2026-10","typ":"JWT"}
 payload {"iss":"https://id.shop.example",
          "sub":"usr_81",
          "aud":"orders-api",
          "iat":1761300120, "nbf":1761300120, "exp":1761301020,      ← valid for 15 minutes
          "jti":"9b2f6c…",
          "scope":"orders:read orders:write",
          "tenant":"t_7"}
 signature ECDSA P-256 over  base64url(header) + "." + base64url(payload)
```

Anyone can decode the header and payload: they are encoded, not encrypted (Chapter 8, Section 8.2). Never put secrets or personal data in a JWT that the holder should not see.

### Mechanism: signing algorithms

| `alg` | Mechanism (Chapter 8) | Who can create tokens | Who can verify | Typical use |
|---|---|---|---|---|
| `HS256` | HMAC-SHA-256 with a shared secret | anyone holding the secret | anyone holding the **same** secret | one service that both issues and verifies |
| `RS256` | RSA signature (PKCS#1 v1.5) with SHA-256 | holder of the private key | anyone with the public key | issuers with many verifiers (OAuth/OIDC providers) |
| `ES256` | ECDSA P-256 with SHA-256 | holder of the private key | anyone with the public key | as RS256, with smaller keys and signatures |
| `EdDSA` | Ed25519 | holder of the private key | anyone with the public key | newer deployments, where supported |

With `HS256`, every verifier can also **mint** tokens, so sharing the secret with ten services means any one compromised service can impersonate anyone. Asymmetric algorithms separate issuing (one private key at the issuer) from verifying (public keys everywhere), which is why identity providers use them and publish their public keys as a **JWKS** (JSON Web Key Set, Section 20.8). The `kid` (key ID) header names which key signed the token, so issuers can rotate keys: publish the new key, start signing with it, keep the old one published until tokens signed with it have expired.

### Mechanism: the validation checklist

A service receiving `Authorization: Bearer <jwt>` must check, **in this order**, and reject on any failure:

```text
 1. format       three base64url parts; header and payload are valid JSON; size within limits
 2. algorithm    header.alg is in the server's ALLOWLIST for this issuer (e.g. exactly ES256);
                 the server decides the algorithm, never the token
 3. key          choose the key by (issuer, kid) from the server's trusted key set for that issuer;
                 never use a key supplied or referenced by an untrusted token header
 4. signature    verify over the exact header.payload bytes
 5. issuer       iss equals the expected issuer
 6. audience     aud contains THIS service's identifier (a token for "billing-api" must not work here)
 7. time         exp > now − leeway;  nbf ≤ now + leeway;  leeway small (say 30–60 s) for clock skew
 8. type/usage   the token is an access token (not an ID token or refresh token, Section 20.8)
 9. revocation   if required: jti or session not on a denylist; or introspect (opaque tokens)
10. authorise    scopes, roles, tenant: does this principal get to do THIS action? (Section 20.9)
```

In practice one uses a maintained JOSE/JWT library and configures steps 2–7 explicitly; the dangerous bugs come from libraries or code that let the **token** decide how it is checked.

### Mechanism: algorithm confusion, and why step 2 exists

Historically, some JWT libraries offered a single function such as `verify(token, key)` that read `alg` from the token's own header and used the supplied key with whatever algorithm the header named. Two classes of flaw followed:

- **`alg: none`.** The JWT specifications define an "unsecured" form with no signature. A verifier that honours `none` from the header accepts a token with any payload and an empty signature: authentication disappears.
- **Key confusion (RS256 versus HS256).** A service is configured with the issuer's RSA **public** key for verification. If its library also honours `HS256` from the header, it would compute an HMAC using the public key's bytes as the shared secret. The public key is, by definition, public, so whoever has it could produce a token that this verifier accepts. The confusion is between "the key used to check an asymmetric signature" and "a symmetric secret".

The fix for both is the same, and it is what the checklist encodes: the server pins the expected algorithm (or a small allowlist) **per issuer and per key**, and keys are typed (an RSA or EC public key can only be used with the matching asymmetric algorithm). The same principle covers other header fields that point to keys (`jku`, `x5u`, `jwk`): a verifier must not fetch or accept keys named by the token itself, only keys from its own trusted configuration. RFC 8725 (JSON Web Token Best Current Practices) collects these rules.

### Mechanism: expiry and the revocation problem

A signed JWT is valid until `exp`, wherever it is presented, because validation is local. If a user logs out, is disabled, or has a token stolen, the token keeps working until it expires. The options:

| Approach | How | Cost |
|---|---|---|
| **Short lifetime** | access tokens valid for minutes (5–15 is common), renewed with a refresh token (Section 20.6) | revocation delayed by up to the lifetime; more refresh traffic |
| **Denylist** | store revoked `jti` values (or "tokens of user 81 issued before T") until their expiry, check on each request | a shared lookup again, though a small one |
| **Introspection / opaque tokens** | ask the issuer on each request (with short caching) | a network call per request, as with sessions |
| **Sender-constrained tokens** | bind the token to a key the client holds (mTLS-bound tokens, RFC 8705; DPoP, RFC 9449), so a stolen token is useless without the key | client complexity |

The usual design combines short-lived access tokens with revocable refresh tokens: the window of an un-revocable token is minutes, and revocation of the refresh token ends the session at the next renewal.

### Guarantees and non-guarantees

A correctly validated JWT guarantees that the configured issuer signed these claims, for this audience, and that they are within their validity window. It does not guarantee that the claims are still true (roles change, users are disabled), that the token was not stolen, that its contents are confidential (they are readable), or anything if validation trusts the token's own header.

### Failure and diagnosis

- **Tokens accepted by the wrong service.** Evidence: a token issued for one API works on another. Cause: no audience check. Fix: step 6.
- **Valid tokens rejected around expiry or issuance.** Evidence: errors such as "token not yet valid" or "token expired" for freshly issued tokens, clustered on hosts whose clocks differ. Cause: clock skew between issuer and verifier. Diagnosis Lab 2.
- **Signature failures after a key rotation.** Evidence: `kid` not found in the cached key set. Cause: a verifier that cached the JWKS too long, or an issuer that removed the old key too early. Fix: refresh the key set when an unknown `kid` arrives (with a rate limit), and overlap old and new keys.
- **Oversized tokens.** Evidence: request headers growing past proxy limits (`431`, Chapter 7, Section 7.5) when roles or groups are embedded. Fix: keep claims small; look up permissions server-side.

### Backend connection

Validate every token with a library configured to pin issuer, audience, algorithm and keys, with a small clock leeway, and treat the payload as readable by anyone. Keep access tokens short-lived and put revocation where it is cheap: on refresh tokens or sessions. When many services validate tokens, asymmetric signatures and a published key set let them do it without sharing secrets.

---

## 20.6 Refresh Tokens, Token Storage, and Sessions versus JWTs

### Definitions

- An **access token** is a short-lived credential presented to APIs. A **refresh token** is a longer-lived credential presented **only** to the authorisation server, to obtain new access tokens without the user logging in again.
- **Refresh-token rotation** issues a new refresh token every time one is used, and invalidates the used one. **Reuse detection** treats the presentation of an already-used refresh token as evidence of theft and revokes the whole chain (the **token family**) derived from the original login.
- The **backend-for-frontend** (BFF) pattern keeps tokens on a server-side component that belongs to the web application, and gives the browser only a first-party session cookie.

### Why it exists

Short-lived access tokens limit the damage of a stolen token, but logging in every 15 minutes is unacceptable. A refresh token bridges the gap, but it is itself a long-lived bearer credential, and stealing it would give an attacker indefinite access. Rotation with reuse detection makes a stolen refresh token **detectable**: either the thief or the real client will soon present a token the other has already used.

### Concrete: rotation and reuse detection

```text
 client (legitimate)            attacker (stole RT1)                 authorisation server
 login ─────────────────────────────────────────────────────────────► family F created
 ◄── AT1 (15 min), RT1                                                RT1 active in F

 (15 min later) POST /token grant_type=refresh_token, RT1 ─────────► RT1 valid → mark RT1 USED
 ◄── AT2, RT2                                                         RT2 active in F

                                 POST /token, RT1 (stolen copy) ─────► RT1 already USED in family F
                                                                       → REUSE DETECTED
                                                                       → revoke the whole family F (RT2 too)
                                 ◄── 400 invalid_grant                 → alert / notify the user

 POST /token, RT2 ───────────────────────────────────────────────────► F revoked → 400 invalid_grant
 ◄── user must log in again (the only party able to do so is the real user)
```

If the attacker uses the stolen token **first**, the roles swap: the attacker gets AT2 and RT2, and the legitimate client's next refresh with RT1 triggers detection and revokes the attacker's tokens. Either way, the theft ends within one refresh interval. The server must store refresh tokens (hashed) with their family, status and expiry; that is server-side state, deliberately, because refresh tokens are where revocation lives. Two practical details: allow a short grace period for a legitimate client that retries a refresh after a network failure (it may present the same token twice within seconds), and give refresh tokens an absolute lifetime as well as an idle one, like sessions (Chapter 19, Section 19.3).

### Mechanism: where a browser application keeps tokens

Chapter 9 (Section 9.5) compared cookies, Web Storage and memory. In summary, for a single-page application:

| Storage | Readable by injected script (XSS) | Sent automatically (CSRF exposure, Chapter 19, Section 19.6) | Survives reload |
|---|---|---|---|
| `localStorage` | **yes**: an XSS can copy the token out and use it anywhere until it expires | no | yes |
| JavaScript memory | yes, while the page is open; cannot be persisted by the attacker after a reload | no | no (needs a refresh mechanism) |
| `HttpOnly` cookie | no | **yes**, mitigated by `SameSite` and CSRF defences | yes |
| BFF: tokens on the server, `HttpOnly` session cookie in the browser | no token in the browser at all | yes (a session cookie, with CSRF defences) | yes |

The pattern recommended in current guidance for browser applications is the **BFF**: the browser holds an `HttpOnly`, `SameSite`, `__Host-` session cookie for the application's own backend, which holds the access and refresh tokens and calls APIs on the user's behalf. It brings back a server-side session, which is the point: the browser is the place most exposed to XSS, so it should hold the least powerful credential.

### Mechanism: sessions versus JWTs, as a trade-off

| Concern | Server-side session (opaque ID in a cookie) | Self-contained JWT access token |
|---|---|---|
| Validation | a store lookup per request | local signature check; no shared store |
| Revocation | immediate: delete the record | not until `exp`, unless a denylist or introspection is added |
| Size on every request | ~40 bytes | hundreds of bytes to kilobytes |
| State on the server | yes (the session store) | none for access tokens; refresh tokens still need state |
| Works across services and organisations | only if they share the store | yes: anyone with the issuer's public key |
| Data freshness | always current (read per request) | claims frozen at issuance (a removed role persists until expiry) |
| Exposure in the browser | `HttpOnly` cookie: not readable by XSS; needs CSRF defences | if kept in script-readable storage: readable by XSS; not ambient, so no CSRF |
| Operational cost | a highly available store | key management and rotation, clock synchronisation |

Neither wins. For a browser application talking to its own backend, server-side sessions are simpler and safer. For APIs called by many clients and validated by many services, short-lived JWT access tokens with rotating refresh tokens fit better. Large systems commonly use both: a session between the browser and the BFF or gateway, and JWTs between the gateway and internal services (Chapter 10, Section 10.4).

### Guarantees and non-guarantees

Rotation with reuse detection guarantees that a stolen refresh token, once used by both parties, ends the session. It does not guarantee detection if the attacker uses it and the real client never refreshes again, and it does not shorten the life of already-issued access tokens. No storage location guarantees safety against XSS in the same origin; storage only limits what can be taken away.

### Failure and diagnosis

- **Users logged out unexpectedly after network errors.** Evidence: reuse detection triggered by the same client within seconds (a retried refresh request). Fix: a short reuse grace window, returning the same new pair.
- **Refresh storms.** All clients refresh at the same moment, for example after an access-token lifetime change or a mass revocation, overloading the authorisation server. Diagnosis Lab 3. Fix: refresh slightly before expiry with random jitter (Chapter 24).
- **Tokens in `localStorage` exfiltrated.** Evidence: API calls with a user's token from unfamiliar IPs and user agents shortly after an XSS report. Fix: revoke the user's refresh tokens; move to the BFF pattern; fix the XSS (Chapter 9, Section 9.8).

### Backend connection

Choose sessions or tokens by who validates the credential and how quickly it must be revocable, not by fashion. If you issue tokens, make access tokens short-lived and refresh tokens rotating, stored hashed with their families, and revocable. In browsers, prefer a BFF with an `HttpOnly` session cookie over tokens in script-readable storage.

---

## 20.7 OAuth 2.0

### Definitions

**OAuth 2.0** (RFC 6749) is a framework for **delegated authorisation**: it lets a user grant an application limited access to their resources on another service, without giving the application their password. It defines four **roles**:

- the **resource owner**: the user who owns the data;
- the **client**: the application that wants access (a web app, a mobile app, a backend job);
- the **authorisation server** (AS): the service that authenticates the user, asks for consent and issues tokens;
- the **resource server** (RS): the API that holds the data and accepts access tokens.

Further terms:

- A **grant type** is a method of obtaining a token: the **authorization code** grant (with PKCE), the **client credentials** grant, the **refresh token** grant, and others for devices.
- A **scope** is a named permission the client requests and the user approves, such as `photos.read` or `orders:write`.
- A **redirect URI** is the client's address to which the authorisation server sends the user back with the result; it must be registered in advance.
- A **confidential client** can keep a secret (a server-side application); a **public client** cannot (a single-page application or mobile app, whose code users can inspect).
- **PKCE** (Proof Key for Code Exchange, RFC 7636, pronounced "pixie") binds an authorisation code to the client instance that requested it, using a one-time secret.

### Why it exists

Before OAuth, an application that wanted to read a user's contacts from an email provider asked for the user's email **password**, stored it, and logged in as the user, with full access, forever. OAuth replaces that with a token that is scoped (only contacts, read-only), time-limited, revocable by the user, and issued without the client ever seeing the password. OAuth is about authorisation, not authentication: an access token says "the bearer may read these contacts", not "this is Ana"; OpenID Connect (Section 20.8) adds identity on top.

### Concrete: the authorization code flow with PKCE

```text
 user's browser           client (app)                 authorisation server (AS)          resource server (API)
 ──────────────────────────────────────────────────────────────────────────────────────────────────────────────
 clicks "Connect"  ─────► 1. code_verifier = 43–128 random characters (kept by the client)
                             code_challenge = BASE64URL(SHA-256(code_verifier))
                             state = random value, bound to the user's session (CSRF protection)
                          2. redirect the browser to the AS:
 ◄─── 302 Location: https://as.example/authorize?
          response_type=code
          &client_id=photo-app
          &redirect_uri=https://photo.app/callback
          &scope=photos.read
          &state=Xy7…
          &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
          &code_challenge_method=S256
 GET /authorize ────────────────────────────────────►  3. authenticate the user (password, MFA: Sections
                                                           20.2–20.4); show consent: "photo-app wants to read
                                                           your photos"; user approves
                                                        4. create a one-time authorization CODE, remember
                                                           (client_id, redirect_uri, scope, user, challenge),
                                                           short lifetime (e.g. 60 s)
 ◄─── 302 Location: https://photo.app/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=Xy7…
 GET /callback?code…&state… ─► 5. check state equals the one stored for this browser session
                              6. back-channel (server to server, or app to AS over TLS):
                                 POST https://as.example/token
                                   grant_type=authorization_code
                                   &code=SplxlOBeZQQYbYS6WxSbIA
                                   &redirect_uri=https://photo.app/callback
                                   &client_id=photo-app
                                   &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
                                   (+ client authentication if confidential)  ──►  7. check: code unused and
                                                                                       unexpired; client_id and
                                                                                       redirect_uri match;
                                                                                       SHA-256(code_verifier)
                                                                                       == stored challenge
                              ◄── {"access_token":"…","token_type":"Bearer","expires_in":900,
                                   "refresh_token":"…","scope":"photos.read"}   mark the code used
                              8. GET /v1/photos  Authorization: Bearer <access_token> ──────────────────────►
                                                                                       validate token (20.5),
                                                                                       check scope photos.read
                              ◄──────────────────────────────────────────────────────── 200 photos
```

Every parameter has a job:

| Parameter | Job |
|---|---|
| `response_type=code` | ask for a short-lived code, delivered through the browser, not a token |
| `client_id` | which registered client is asking |
| `redirect_uri` | where to return; must **exactly match** a registered value, or codes could be delivered to an attacker's address |
| `scope` | what access is requested; the AS may grant less |
| `state` | random per-request value the client checks on return: protects the callback against **login CSRF**, where an attacker makes the victim's browser complete a flow with the **attacker's** code (Chapter 19, Section 19.6) |
| `code_challenge`, `code_challenge_method=S256` | the hash of a secret only this client instance knows |
| `code` | one-time, short-lived, useless without the verifier |
| `code_verifier` | proves, at the token endpoint, that the party redeeming the code is the one that started the flow |
| `access_token`, `expires_in`, `refresh_token` | the result: a short-lived API credential and a longer-lived way to renew it |

Why PKCE matters: the code travels through the browser (the address bar, history, logs, and on mobile, the operating system's URL dispatch to apps). If another party obtains the code, it still cannot exchange it, because it does not have the `code_verifier`, and the hash cannot be reversed. For public clients that cannot keep a client secret, PKCE is what makes the code flow safe; current guidance (the OAuth 2.0 Security Best Current Practice, RFC 9700, and the OAuth 2.1 draft) requires PKCE for **all** clients.

### Mechanism: the client credentials flow

When there is no user, only a service acting on its own behalf (a nightly billing job calling the invoices API), the client authenticates directly:

```text
 billing-job ── POST https://as.example/token
                grant_type=client_credentials&scope=invoices:read
                (client authentication: client secret, or better, a private-key JWT or mTLS, Section 20.10) ──► AS
             ◄── {"access_token":"…","expires_in":600}
 billing-job ── GET /v1/invoices  Authorization: Bearer … ──► invoices API: validate, check scope
```

There is no refresh token: the client simply requests a new access token when the old one expires.

### Mechanism: why the implicit flow died

The **implicit flow** (`response_type=token`) was designed for browser applications at a time when they could not make cross-origin token requests: the AS returned the **access token directly in the redirect URL** fragment. That put a bearer token where URLs go: browser history, the `Referer` of later requests in some cases, logs, and any script on the page; it could not use refresh tokens safely, and there was no way to bind the token to the client that asked for it. Once CORS (Chapter 9, Section 9.7) let single-page applications call the token endpoint, the authorization code flow with PKCE became possible for them, and current guidance (RFC 9700, OAuth 2.1) drops the implicit flow, as it drops the **resource owner password credentials** grant, in which the client collected the user's password itself.

### Guarantees and non-guarantees

OAuth guarantees, when implemented to current guidance, that a client receives only the access the user approved, for a limited time, without seeing the user's credentials, and that intercepted codes cannot be redeemed. It does not authenticate the user **to the client** (an access token is not proof of who the user is: Section 20.8), does not define the format of access tokens (JWT or opaque is the AS's choice), and does not make a resource server check scopes and object ownership; that remains the API's job (Section 20.9).

### Failure and diagnosis

- **`invalid_grant` at the token endpoint.** Causes: the code expired or was already used (a double-submitted callback), the `redirect_uri` differs from the one in the authorisation request, or the `code_verifier` does not match. Evidence: the AS's error description and logs.
- **`redirect_uri_mismatch` errors.** A trailing slash, `http` versus `https`, a different port. Fix: register exact values; never allow wildcards.
- **Login CSRF.** Users find themselves logged into someone else's account after "connecting". Evidence: callbacks without `state` validation. Fix: generate, store and check `state` (or rely on PKCE plus `nonce` in OIDC).
- **Over-broad scopes.** Clients request `*` or admin scopes because it is easier. Fix: least privilege per client; the AS should reject undeclared scopes.

### Backend connection

Use a well-tested authorisation server rather than building one, use the authorization code flow with PKCE for every interactive client and client credentials for services, register exact redirect URIs, keep access tokens short-lived, and make every resource server enforce scopes and object-level checks itself.

---

## 20.8 OpenID Connect, Single Sign-On, and SAML

### Definitions

**OpenID Connect** (OIDC, published by the OpenID Foundation) is an identity layer on top of OAuth 2.0. The client asks for the `openid` scope, and the authorisation server, now called the **OpenID Provider** (OP), returns an **ID token** alongside the access token: a JWT that tells the client **who the user is** and **how they authenticated**.

- An **ID token** is a JWT for the **client** (its `aud` is the client ID), with claims such as `sub`, `iss`, `aud`, `exp`, `iat`, `auth_time`, `nonce`, `amr` (authentication methods, Section 20.4), and optionally profile claims (`email`, `email_verified`, `name`).
- The **UserInfo endpoint** returns profile claims for an access token.
- **Discovery** is the document at `https://<issuer>/.well-known/openid-configuration` that lists the provider's endpoints, supported algorithms and the location of its **JWKS** (`jwks_uri`): the public keys for verifying its tokens.
- **Single sign-on** (SSO) is one login at an identity provider giving access to many applications, each of which trusts the provider instead of keeping its own passwords.
- **SAML 2.0** (Security Assertion Markup Language) is an older, XML-based federation standard, still dominant in enterprise SSO: an identity provider sends the application (the **service provider**) a signed XML **assertion** about the user, usually through the browser.

### Why it exists

OAuth tells a client what it may do, not who the user is. Early "login with OAuth" implementations treated "I received an access token that works on the provider's API" as proof of identity, which breaks when a token issued to one application is presented to another. OIDC standardises a token that is explicitly **about authentication** and explicitly **addressed to one client**, plus discovery and key distribution, so that any application can accept federated logins safely with an off-the-shelf library.

### Concrete: ID token versus access token

| | ID token | Access token |
|---|---|---|
| Purpose | tells the **client** who the user is and how they logged in | lets the **client** call an **API** |
| Audience (`aud`) | the client ID | the resource server (API) |
| Format | always a JWT | JWT or opaque, the provider's choice |
| Who validates it | the client | the API |
| Contains | `sub`, `iss`, `auth_time`, `nonce`, `amr`, profile claims | scopes, subject, expiry; whatever the API needs |
| Must be sent to APIs? | **no**: an API must reject it (wrong audience and wrong type) | yes |
| Lifetime | short; used once, at login | short; renewed by refresh tokens |

Checks a client must make on an ID token, beyond the JWT checklist of Section 20.5: `aud` contains its client ID; `nonce` equals the random value it sent in the authorisation request (this binds the token to this login attempt and stops replay); `iss` matches the discovered issuer; and, if it will trust the email for account linking, `email_verified` is true.

### Mechanism: keys and discovery

```text
 client or API, at start-up and periodically:
   GET https://id.shop.example/.well-known/openid-configuration
     → {"issuer":"https://id.shop.example", "authorization_endpoint": "…/authorize",
        "token_endpoint":"…/token", "jwks_uri":"…/jwks.json",
        "id_token_signing_alg_values_supported":["ES256"], …}
   GET https://id.shop.example/jwks.json
     → {"keys":[{"kty":"EC","crv":"P-256","kid":"k2026-10","x":"…","y":"…","use":"sig","alg":"ES256"},
                {"kty":"EC","crv":"P-256","kid":"k2026-07", …}]}        ← old key kept during rotation
 per token: pick the key whose kid matches, from THIS trusted set only (Section 20.5, step 3)
```

### Mechanism: single sign-on

```text
 user opens app A ─► A redirects to the IdP (OIDC authorization code + PKCE)
                     IdP: no IdP session → login with password + MFA → IdP sets its OWN session cookie
                     ◄─ back to A with a code → A gets an ID token → A creates ITS session (Chapter 19)
 user opens app B ─► B redirects to the IdP
                     IdP: session cookie present → no login prompt → straight back to B with a code
                     → B creates its session
 logout: each application ends its own session; ending the IdP session stops new logins; propagating
         logout to every application needs extra mechanisms (front- or back-channel logout notifications)
```

SSO centralises password checks, MFA and account disabling in one place, which is its main security value: disabling an employee at the IdP stops new logins everywhere. Sessions already created in each application continue until they expire or are revoked, so session lifetimes in SSO-backed applications should be modest.

### Mechanism: SAML, briefly

SAML follows the same shape with different technology. The service provider redirects the browser to the IdP with an authentication request; the IdP authenticates the user and returns an XML **assertion** (subject, attributes, conditions such as audience and validity times, and an XML digital signature), usually posted back through the browser in an auto-submitting HTML form. The service provider validates the signature against the IdP's certificate, checks audience, times and that the assertion has not been used before, and creates its own session. SAML's well-known weakness is the complexity of XML signature validation: implementations that validate one part of a document and read the user from another have produced serious authentication bypasses, so SAML should only ever be processed with a mature, current library. New consumer-facing integrations generally use OIDC; enterprise customers frequently require SAML.

### Guarantees and non-guarantees

A validated ID token guarantees that the provider authenticated this user (`sub`, at `auth_time`, with `amr`) for this client and this login attempt (`nonce`). It does not authorise anything at an API, does not guarantee that profile claims such as email are verified unless `email_verified` says so, and does not end application sessions when the IdP session ends.

### Failure and diagnosis

- **Account takeover by email matching.** An application links a federated login to an existing local account because the emails match, without checking `email_verified` (or with a provider that lets users set unverified emails). Fix: link only on verified email, or require the user to log in to the existing account first.
- **ID tokens used as API credentials.** Evidence: APIs accepting tokens whose `aud` is a client ID. Fix: APIs accept only access tokens for their own audience.
- **Login failures after the provider rotates keys.** As in Section 20.5: cached JWKS without the new `kid`.

### Backend connection

Use OIDC (with a library) for federated login and SSO, validate ID tokens with the nonce and audience checks, and keep ID tokens out of API calls. Accept SAML where enterprise customers require it, through a maintained library only. Remember that federation moves authentication to the provider but leaves session lifetime, account linking and authorisation in your application.

---

## 20.9 Authorisation Models and Object-Level Authorisation

### Definitions

- An **authorisation policy** is the set of rules that decide whether a principal may perform an action on a resource. A **policy decision point** evaluates the rules; a **policy enforcement point** is the place in code that asks and obeys.
- An **access control list** (ACL) attaches to each resource a list of principals and what each may do: "document 17: Ana can edit, Ben can read".
- **Role-based access control** (RBAC) grants permissions to **roles** (`support_agent`, `billing_admin`) and roles to users; a check asks whether any of the user's roles carries the permission.
- **Attribute-based access control** (ABAC) evaluates rules over attributes of the principal, the resource, the action and the environment: "a doctor may read a patient record if they are assigned to the patient's ward and it is during their shift".
- **Relationship-based access control** (ReBAC) derives permissions from a graph of relationships between principals and objects: "Ana can edit document 17 because she is a member of team X, which owns folder Y, which contains document 17". Google's Zanzibar paper (2019) popularised it; several open-source systems implement it.
- **Object-level authorisation** is checking access to the **specific object** a request names, not just to the type of operation. Its absence is **IDOR** (insecure direct object reference) or, in the OWASP API Security list, **BOLA** (broken object-level authorisation, Chapter 13, Section 13.13).
- **Multi-tenancy** is one deployment serving several customer organisations (**tenants**) whose data must stay separate. **Row-level security** (RLS) is a database feature that filters rows by a policy on every query, automatically.

### Why it exists

Authentication tells you that the request comes from user 81; it says nothing about whether user 81 may read order 9001. Applications grow many endpoints, and each one that loads an object by an ID from the request is a place where that check can be forgotten. Broken access control heads the OWASP Top 10 (2021) because the failure is silent: the endpoint works perfectly, for everyone, including people who should not see the data.

### Mechanism: the four models side by side

| Model | Rule shape | Strengths | Weaknesses | Typical use |
|---|---|---|---|---|
| ACL | resource → {principal: permissions} | precise per object; easy to explain | hard to manage at scale; "who can access what" is spread over every object | file systems, document sharing |
| RBAC | user → roles → permissions | simple, auditable, matches organisations | **role explosion** when rules depend on context ("support agent for EU customers only"); no notion of ownership by itself | admin consoles, internal tools, coarse API permissions (scopes) |
| ABAC | rule over attributes (user, resource, action, context) | expressive: ownership, tenancy, time, location | rules are harder to reason about and audit; needs attributes available at decision time | regulated domains, multi-tenant SaaS, fine-grained rules |
| ReBAC | paths in a relationship graph | natural for sharing, hierarchies and groups; scales to many objects | a separate system and data model to keep in sync; consistency of the graph matters | collaboration products (documents, folders, organisations) |

Real systems combine them: RBAC for "what kinds of action may this user perform at all" (only `billing_admin` may issue refunds), plus ownership or tenancy rules (ABAC) for "on which objects" (only refunds for orders in the admin's tenant).

### Concrete: an IDOR, and the query-level fix

```http
GET /v1/invoices/52211 HTTP/1.1
Authorization: Bearer <token for user 81, tenant 7>
```

```pseudo
// VULNERABLE: authenticated, but not authorised for THIS object
function getInvoice(request, invoiceId):
    invoice = db.queryOne("SELECT * FROM invoices WHERE id = $1", [invoiceId])
    if invoice is null: return 404
    return 200, toDto(invoice)
```

User 81 can change `52211` to `52212`, `52213` and so on, and read every customer's invoices. Sequential IDs make the enumeration trivial, but random IDs (UUIDs) only make it harder, not impossible: IDs leak through URLs, emails and logs, and "hard to guess" is not authorisation.

The fix puts the authorisation rule **into the query**, so that an object the principal may not see is never loaded:

```pseudo
function getInvoice(request, invoiceId):
    p = request.principal                               // from the authenticated context, never the body
    invoice = db.queryOne(
        "SELECT * FROM invoices WHERE id = $1 AND tenant_id = $2 AND (owner_id = $3 OR $4)",
        [invoiceId, p.tenantId, p.userId, p.hasRole("billing_admin")])
    if invoice is null: return 404                      // same answer for "absent" and "not yours"
    return 200, toDto(invoice)
```

Returning 404 for "exists but not yours" avoids confirming that the object exists. The same scoping applies to every operation: updates (`UPDATE … WHERE id = $1 AND tenant_id = $2`, checking the affected-row count, Chapter 18, Section 18.13), deletes, exports, and searches (`WHERE tenant_id = $2` on every list query).

### Mechanism: enforcing tenancy in the database with row-level security

Scoping every query by hand works until one query forgets. PostgreSQL's row-level security makes the database apply the filter itself:

```sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices FORCE ROW LEVEL SECURITY;            -- apply to the table owner too
CREATE POLICY tenant_isolation ON invoices
    USING (tenant_id = current_setting('app.tenant_id')::bigint)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);

-- the application sets the tenant at the start of each transaction (Chapter 18, Section 18.12):
BEGIN;
SET LOCAL app.tenant_id = '7';                             -- LOCAL: safe with poolers (Chapter 18, 18.5)
SELECT * FROM invoices WHERE id = 52212;                   -- returns nothing if 52212 belongs to tenant 9
COMMIT;
```

`USING` filters which rows are visible; `WITH CHECK` stops inserting or moving rows into another tenant. Superusers and roles with the `BYPASSRLS` attribute ignore policies, so the application must connect as an ordinary role (least privilege, Chapter 18, Section 18.7), and `SET LOCAL` (not `SET`) keeps the setting from leaking to the next user of a pooled connection. RLS is a second line of defence for tenancy, not a replacement for authorisation logic in the application: it knows tenants, not whether user 81 may refund.

### Mechanism: where checks belong

```text
 gateway / middleware   coarse: authenticated? token audience and scopes allow this endpoint?
 application service    function-level: may this principal perform this action at all (role, scope)?
                        object-level: may they perform it on THIS object (owner, tenant, state)?
 query / database       scope every query by tenant and owner; RLS as a backstop
```

Two rules prevent most failures. First, **deny by default**: an endpoint without an explicit policy rejects requests, so a new endpoint cannot ship open by accident. Second, **test authorisation like features**: for every endpoint, a test that a user from another tenant and a user without the role both get 404 or 403.

### Guarantees and non-guarantees

A policy enforced in the query guarantees that objects outside the principal's scope are never loaded or changed by that query. RLS guarantees the filter on every query of a table for non-bypassing roles. Neither guarantees function-level rules (who may refund), protects data reached through a path that skips them (a reporting job connecting as a superuser, a cache that keys only by invoice ID, Chapter 11, Section 11.1), or prevents leaks through side channels such as different error messages or response times.

### Failure and diagnosis

- **IDOR.** Evidence: audit logs showing one principal reading many objects of other tenants, often sequential IDs; a security test that swaps IDs. Diagnosis Lab 4. Fix: scope queries; deny by default; add tests.
- **Function-level gaps.** An admin endpoint reachable by ordinary users because only the UI hides the button. Evidence: requests to admin routes from non-admin principals succeeding. Fix: role checks at the endpoint (BFLA in the OWASP API list).
- **Cross-tenant cache leaks.** A cache key without the tenant ID returns one tenant's data to another. Evidence: cache keys in code. Fix: include every input the value depends on in the key (Chapter 11, Section 11.1).
- **Role explosion.** Hundreds of roles encoding combinations of context. Fix: keep roles coarse, move context into attribute rules.

### Backend connection

Make object-level authorisation part of how data is loaded: every repository method takes the principal (or tenant) and scopes its query, so that a forgotten check becomes a compile-time or review-time omission rather than a silent leak. Use RBAC for coarse permissions, ownership and tenancy rules for objects, ReBAC when sharing graphs dominate, and RLS as a backstop for tenant isolation.

---

## 20.10 Service-to-Service Identity

### Definitions

- **Service identity** (workload identity) is an identity assigned to a running program rather than a person, so that the services it calls can authenticate it and authorise its requests.
- An **API key** is a long random secret issued to a client (often an external customer's integration) and presented on each request.
- **Mutual TLS** (mTLS) authenticates both ends of a TLS connection with certificates (Chapter 8, Section 8.10).
- A **workload identity system** issues short-lived credentials to workloads automatically, based on where they run (a Kubernetes service account, a cloud instance role), so that no long-lived secret is deployed with the code. SPIFFE (Chapter 8, Section 8.10) is a standard format for such identities (`spiffe://prod.shop/ns/billing/sa/invoice-worker`).
- **Token exchange** (RFC 8693) lets a service trade a token it holds for a different one: narrower, for another audience, or representing "service A acting on behalf of user 81".

### Why it exists

Inside a system of many services, "the request came from our private network" is not authentication: one compromised service, a misconfigured network rule or an SSRF bug (below) lets an attacker send requests that look internal. Each service should know which service is calling it and authorise accordingly ("only the checkout service may create payments"), and the credentials that prove it should be short-lived and impossible to copy usefully.

### Mechanism: the options

| Method | How the caller proves itself | Strengths | Weaknesses |
|---|---|---|---|
| API key | sends a static secret in a header | simple; good for external customers' integrations | long-lived; leaks through logs, repos and screenshots; must be stored hashed and rotated |
| OAuth client credentials (Section 20.7) | gets a short-lived access token with a client secret, a private-key JWT, or mTLS | standard tokens with scopes and audiences; central revocation | the client authentication secret still needs protection |
| mTLS | presents a client certificate in the TLS handshake | strong, cryptographic, works below the application; often automated by a service mesh (Chapter 10, Section 10.4) | certificate issuance and rotation infrastructure; identity is per workload, not per user |
| Workload identity | the platform gives the workload a short-lived credential (a signed token or certificate) attesting where it runs | no deployed secrets; automatic rotation | depends on the platform; the metadata endpoint becomes valuable (below) |

**API keys done properly** follow the same rules as session IDs and reset tokens: at least 128 bits from a CSPRNG, stored only as a hash, shown once at creation, with a short non-secret **prefix** (`sk_live_3f9a…`) so that keys can be identified in logs and by secret scanners without revealing them, scoped to minimal permissions, and rotatable without downtime (allow two active keys per client during rotation).

**User context across services.** When the orders service calls the payments service for user 81, payments needs both facts: "the caller is the orders service" (mTLS or a service token) and "on behalf of user 81" (a user token passed along, or a narrower token obtained by token exchange). Forwarding the user's original access token everywhere works only if every service is in its audience, which widens what a leaked token can do; exchanging it for a token scoped to the next service keeps each hop's power minimal.

### Mechanism: SSRF and the metadata endpoint

Chapter 13 (Section 13.14) explained server-side request forgery: a feature that fetches a URL supplied by a user can be made to fetch internal addresses. Workload identity raises the stakes. On most clouds, an **instance metadata service** at a link-local address (Chapter 6, Section 6.4) hands out the instance's temporary cloud credentials to any process on the machine that asks. An SSRF bug in a web application on that machine can therefore turn into the attacker holding the instance's cloud credentials, with whatever access its role has to storage buckets and databases.

The defences, in layers (each detailed in Chapter 13, Section 13.14):

1. **Validate outbound destinations**: allowlist hosts, resolve and check the IP address before connecting, refuse private and link-local ranges, do not follow redirects blindly.
2. **Harden the metadata service**: require the session-token form of the metadata API (on AWS, IMDSv2, which needs a `PUT` request with a special header first, which simple SSRF through a `GET` cannot perform), and limit how many network hops its responses may travel.
3. **Least privilege for the instance role**: the credentials, if stolen, should reach as little as possible.
4. **Egress controls**: the web tier has no route to the metadata service or internal admin endpoints it does not need.

### Guarantees and non-guarantees

mTLS and workload identity guarantee that the caller holds a credential issued to a specific workload, typically valid for minutes to hours. They do not guarantee that the workload has not been compromised (a compromised service holds its own valid credential), and they do not replace authorisation: an authenticated service may still be asking for something it should not. API keys guarantee only that the caller has the key.

### Failure and diagnosis

- **Leaked API keys.** Evidence: secret-scanning alerts on a public repository; requests with a customer's key from unexpected networks. Fix: revoke and reissue; scan repositories and logs continuously (Section 20.11).
- **Expired service certificates.** A whole call path fails with TLS handshake errors at the same moment. Evidence: certificate `notAfter` in the handshake errors (Chapter 8, Section 8.13). Fix: automated rotation and expiry alerts.
- **Cloud credentials used from outside.** Evidence: the cloud audit log shows the instance role's credentials used from an unknown IP. Cause: SSRF to the metadata service, or credentials leaked from a container image. Fix: rotate, harden metadata access, review the SSRF path.

### Backend connection

Authenticate every service call, prefer platform-issued short-lived identities and mTLS over static keys, scope service tokens by audience and permission, and carry user context with narrow exchanged tokens. Treat any feature that fetches user-supplied URLs as a path to your cloud credentials, and defend it accordingly.

---

## 20.11 Secrets Management

### Definitions

A **secret** is any value that grants access or proves identity: database passwords, API keys, private keys, signing keys for tokens and cookies, OAuth client secrets, peppers. **Secrets management** is the practice of storing, distributing, using, rotating and revoking secrets so that they reach only the processes that need them, for as short a time as possible. A **secret manager** (a vault service, a cloud provider's secrets service) stores secrets encrypted, controls and audits access to them, and can issue **dynamic secrets**: credentials created on demand for one consumer with a short lifetime.

### Why it exists

Secrets leak through the same few channels again and again: committed to source control (and kept forever in the history), baked into container images, printed in logs and error pages, pasted into tickets and chat, left in CI variables visible to every job. Once leaked, a long-lived secret is valid until someone notices. Secrets management limits where secrets exist, who can read them, and how long a leaked one is useful.

### Mechanism: the lifecycle of a secret

```text
 CREATE   generated by the secret manager or the system that owns it (random, high entropy)
 STORE    encrypted at rest in the secret manager, under keys held in a KMS (envelope encryption, Chapter 8, 8.3)
 DELIVER  to the workload at run time: fetched with its workload identity (Section 20.10), or mounted as a file;
          never in the image, never in source control (Chapter 12, Section 12.9 compared delivery methods)
 USE      kept in memory; never logged, never returned in responses or error messages
 ROTATE   replace on a schedule and immediately after suspected exposure, without downtime:
            1. create the new secret   2. make consumers accept BOTH old and new
            3. switch producers to the new one   4. revoke the old one after everything has moved
 REVOKE   on decommission or leak; audit who read it and when
```

Rotation needs the "accept both" phase because consumers and producers do not switch at the same instant. For a database password: create a second user (or a second password, where supported), deploy applications with the new one, then remove the old. For a JWT signing key: publish the new public key in the JWKS, start signing with it, and remove the old key after the longest token lifetime has passed (Section 20.5). For a cookie-signing key: verify with old and new, sign with new.

### Concrete: where secrets should and should not live

| Place | Verdict | Why |
|---|---|---|
| Source code or committed config files | never | repository history keeps them forever; every clone has them |
| Container images | never | images are copied to registries and laptops; layers are easy to inspect (Chapter 4, Section 4.10) |
| Environment variables | acceptable, with care | readable by the process and its children, sometimes dumped in crash reports and debug endpoints |
| Mounted files from the platform (Kubernetes Secrets, which are only Base64-encoded unless encryption at rest is configured, Chapter 12, Section 12.9) | acceptable | access controlled by the platform; rotation by re-mounting |
| Secret manager, fetched at run time with workload identity | preferred | encrypted, audited, rotatable, no long-lived secret to deploy |
| Dynamic, short-lived credentials (for example per-instance database users valid for an hour) | best, where available | a leak expires on its own |

### Guarantees and non-guarantees

A secret manager guarantees encrypted storage, access control and an audit trail of reads. It does not guarantee that an application holding a secret will not log it, that a compromised workload cannot use the secrets it is allowed to read, or that rotation happens; those need code discipline, least privilege and automation.

### Failure and diagnosis

- **Secret committed to a repository.** Evidence: a secret-scanning alert. Fix: **rotate first** (removing the commit does not un-leak it), then clean the history if needed, and add pre-commit scanning.
- **Secrets in logs.** Evidence: grep for key prefixes and known patterns; structured-log redaction rules missing. Fix: redaction at the logging layer (Chapter 26); rotate the exposed values.
- **Outage during rotation.** Evidence: authentication failures right after a rotation, from consumers still using the old secret. Fix: the "accept both" phase; rotation tested in staging.

### Backend connection

Assume every secret will leak eventually, and design so that a leak is short-lived and contained: deliver secrets at run time through workload identity, give each service only its own secrets, rotate on a schedule with an overlap phase, prefer dynamic credentials, and scan code, images and logs continuously.

---

## 20.12 Application Security Synthesis

### Definitions

- **Security**, for a system, means that it keeps three properties even against deliberate attack, together called the **CIA triad**. **Confidentiality**: only authorised parties can read the data. **Integrity**: data and actions cannot be changed without authorisation, or at least not without detection. **Availability**: authorised users can use the system when they need it. Chapter 8 (Section 8.1) defined confidentiality and integrity for data in transit, together with authenticity and non-repudiation. **Application security** is the discipline of protecting all three properties across a whole application: its code, its data, its identities and the infrastructure it runs on. Every mechanism in Part V serves one of them: sessions and authentication serve integrity (only the real user acts as that user), authorisation serves confidentiality and integrity, and rate limits and capacity limits on login serve availability.
- A **threat model** is a structured description of what a system must protect, from whom, through which entry points, and with which controls. **Threat modelling** is the activity of producing one, ideally while designing a feature.
- **STRIDE** is a checklist of threat categories, each the violation of a security property: **S**poofing (authenticity), **T**ampering (integrity), **R**epudiation (non-repudiation, auditability), **I**nformation disclosure (confidentiality), **D**enial of service (availability), **E**levation of privilege (authorisation).
- **Defence in depth** is layering independent controls so that one failure does not lead to compromise.
- **Least privilege** is giving every principal (user, service, database role) only the permissions its task needs.
- An **audit trail** is an append-only record of security-relevant events (who did what, to which object, when, from where, with what outcome), kept so that incidents can be investigated and actions attributed.
- The **OWASP Top 10** is a regularly updated awareness list of the most critical web application security risk categories, published by the Open Worldwide Application Security Project. This section uses its 2021 edition, which earlier chapters cite; check whether a newer edition has since been published.

### Why it exists

Security in this book has been taught where each mechanism lives: TLS with networking, XSS with the browser, injection with database integration. Real incidents cross those boundaries: an SSRF in an image fetcher reaches the metadata service, whose credentials read a storage bucket, whose contents include a database backup with weak password hashes. Threat modelling and defence in depth are the tools for seeing a system as an attacker sees it, as connected paths rather than separate topics.

### Mechanism: threat modelling a feature with STRIDE

Feature: "customers can upload a profile photo, by file or by URL".

```text
 data flow:  browser ──HTTPS──► API ──► image service ──► object storage
                                  │            └──► (fetch by URL) the internet / internal network!
                                  └──► database (photo metadata, owner)

 S  spoofing      upload on behalf of another user?        → principal from the session/token only (20.1)
 T  tampering     replace another user's photo?            → object key includes the owner; scoped writes (20.9)
 R  repudiation   "I never uploaded that"                  → audit log of uploads with principal and source
 I  disclosure    other users' photos via guessable URLs?  → random object keys or signed URLs with expiry
                  internal data via fetch-by-URL (SSRF)?   → destination validation, metadata hardening (20.10)
 D  denial        100 MB files, decompression bombs        → size and pixel limits, timeouts (Chapter 12, 12.7)
 E  elevation     image parser vulnerability → code exec   → process images in a sandboxed, low-privilege service
```

The output is a short list of controls attached to the design, each with an owner and a test.

### Concrete: the OWASP Top 10 (2021) mapped to this book

| Risk | Mechanism of the risk | Where the defence is taught |
|---|---|---|
| A01 Broken Access Control | missing object- or function-level checks (IDOR/BOLA); CORS misconfiguration; path manipulation; cross-site request forgery is also in this category | Section 20.9; Chapter 13, Section 13.13; Chapter 9, Section 9.7; Chapter 19, Section 19.6 |
| A02 Cryptographic Failures | data in plaintext in transit or at rest; weak algorithms; fast password hashes; poor key management | Chapter 8 (TLS, AEAD, KMS); Section 20.2; Section 20.11 |
| A03 Injection | untrusted data interpreted as code: SQL, NoSQL, OS commands; XSS is in this category in the 2021 list | Chapter 18, Section 18.7; Chapter 9, Section 9.8 |
| A04 Insecure Design | missing controls in the design itself (no rate limit on reset, no tenancy model) | threat modelling (this section); Section 20.3; Chapter 13 |
| A05 Security Misconfiguration | default credentials, verbose errors, open storage, missing security headers, permissive CORS | Chapter 9, Sections 9.7 and 9.9; Chapter 12, Section 12.9 |
| A06 Vulnerable and Outdated Components | known-vulnerable libraries and images | dependency scanning and patching (Chapter 26); minimal images (Chapter 4, Section 4.10) |
| A07 Identification and Authentication Failures | weak passwords, credential stuffing, broken session handling, missing MFA | Sections 20.2–20.6; Chapter 19 |
| A08 Software and Data Integrity Failures | insecure deserialisation; unsigned updates and pipelines | Chapter 12, Section 12.7; signed artefacts (Chapter 26) |
| A09 Security Logging and Monitoring Failures | attacks not recorded or not noticed | audit trails (below); Chapter 26 |
| A10 Server-Side Request Forgery | server fetches attacker-chosen URLs | Chapter 13, Section 13.14; Section 20.10 |

### Mechanism: defence in depth, on one path

```text
 an injection bug slips into an orders endpoint
   layer 1  parameter binding everywhere (Ch 18, 18.7)              → should have prevented it; failed
   layer 2  least-privilege DB role: no DDL, no access to the users table from the orders service
   layer 3  row-level security: only the current tenant's rows are visible
   layer 4  password verifiers are Argon2id: even if hashes leak, cracking is slow (20.2)
   layer 5  audit log + anomaly alert on unusual query volume → detected within minutes
   result   one failed control; four others limit and expose the damage
```

### Mechanism: security logging and audit trails

Record, as structured events (Chapter 26 covers pipelines):

- authentication events: logins (success and failure, with reason category, not the password), MFA challenges, password changes and resets, token refreshes and reuse detections, logouts;
- authorisation denials, especially object-level ones (a burst of 404s for other tenants' IDs is an IDOR probe);
- administrative and sensitive actions: role changes, refunds, data exports, secret reads;
- for each: the principal, the action, the object, time, source (IP, user agent), request ID (Chapter 10, Section 10.11) and outcome.

Never log secrets, passwords, full tokens, session IDs or unnecessary personal data; log a token's `jti` or a hash prefix instead. Keep the audit trail append-only and out of reach of the services it records (shipped to separate storage), so that an attacker who compromises a service cannot erase its tracks.

### Guarantees and non-guarantees

Threat modelling guarantees that the threats you thought of have owners and controls; it does not find the ones nobody thought of. Defence in depth guarantees that one control's failure is not automatically a breach. The OWASP list guarantees a shared vocabulary, not completeness. Audit logs guarantee evidence only if they are written, protected and read.

### Failure and diagnosis

- **Breach discovered months later.** Evidence: none, because nothing was logged or alerted. Fix: audit events and alerts on authentication anomalies, denials and bulk reads.
- **One bug, total compromise.** Evidence: the incident timeline shows each step needed no second failure (a superuser database role, no tenancy filter, a shared secret). Fix: least privilege and layered controls at each step.

### Backend connection

Threat-model each feature that crosses a trust boundary (Chapter 1, Section 1.3), with STRIDE as the checklist, and attach controls to the design before writing code. Build in layers, give every component least privilege, map your controls against the OWASP Top 10, and keep an audit trail that would let you answer "who did this, and what else did they touch?" within an hour of an incident.

---

## 20.13 Placement-Round Topics: Kerberos, LDAP, and Classic Access-Control Models

> **Placement-round focus:** This section is revision material for campus placement interviews, university security and networking courses, and written tests. It is not part of the main flow. What carries into backend work: Kerberos is the ancestor of ticket-based SSO (Section 20.8) and still runs inside corporate Windows networks, and LDAP directories are what many enterprise identity providers read users and groups from.

### Kerberos

**Kerberos** is a network authentication protocol, developed at MIT, in which a trusted third party, the **Key Distribution Center** (KDC), issues time-limited **tickets** so that users prove their identity to services without sending passwords over the network. The KDC has two parts: the **Authentication Service** (AS) and the **Ticket-Granting Service** (TGS). It uses symmetric keys: each user and each service shares a long-term key with the KDC (the user's is derived from their password).

```text
 1. AS exchange      client ──"I am ana"──► AS
                     AS ──► client: a session key for the TGS (encrypted with ana's key)
                                   + a TICKET-GRANTING TICKET (TGT), encrypted with the TGS's key
                     (only the real Ana can decrypt the session key: her password is never sent)
 2. TGS exchange     client ──TGT + authenticator (timestamp, encrypted with the session key)──► TGS
                     TGS ──► client: a SERVICE TICKET for "fileserver" + a new session key
 3. AP exchange      client ──service ticket + authenticator──► fileserver
                     fileserver decrypts the ticket with its own key, checks the authenticator's
                     timestamp; optionally proves itself back (mutual authentication)
```

The **authenticator** is a fresh timestamp encrypted with the session key, which stops an intercepted ticket from being replayed later; that is why Kerberos requires clocks to be synchronised (commonly within five minutes). The TGT is what gives single sign-on: one password entry at login, then tickets for each service as needed. Microsoft Active Directory uses Kerberos as its main authentication protocol.

### LDAP and directories

A **directory service** stores identities and their attributes (users, groups, devices) in a hierarchical tree, optimised for frequent reads. **LDAP** (Lightweight Directory Access Protocol) is the standard protocol for querying and modifying one. Entries are named by a **distinguished name** (DN), such as `uid=ana,ou=people,dc=shop,dc=example`, and have attributes (`cn`, `mail`, `memberOf`). An application can authenticate a user against LDAP with a **bind** using the user's DN and password, and read group memberships for authorisation. Active Directory is the most widespread directory and speaks LDAP. Modern applications usually reach a directory indirectly, through an identity provider that federates it via OIDC or SAML (Section 20.8). Building LDAP filters by concatenating user input causes **LDAP injection**, with the same cause and cure as SQL injection (Chapter 18, Section 18.7).

### Classic access-control models

| Model | Who decides access | Example |
|---|---|---|
| **Discretionary access control** (DAC) | the owner of an object grants access at their discretion | Unix file permissions; sharing a document |
| **Mandatory access control** (MAC) | a central policy based on security **labels**; users cannot override it | military classification levels; SELinux (Chapter 4, Section 4.10) |
| **Role-based access control** (RBAC) | permissions attach to roles (Section 20.9) | enterprise applications |

Two MAC models are asked often:

- **Bell–LaPadula** protects **confidentiality**: "no read up" (a subject may not read objects above its clearance) and "no write down" (it may not write to objects below its level, so secrets cannot leak downwards).
- **Biba** protects **integrity**, with the reverse rules: "no read down" (do not rely on less trustworthy data) and "no write up" (do not contaminate more trustworthy data).

The **access control matrix** is the abstract model behind all of them: rows for subjects, columns for objects, cells for rights. An ACL is one column of the matrix stored with the object; a **capability list** is one row stored with the subject (a capability is an unforgeable token naming an object and the rights to it, the idea behind bearer tokens).

### Drill for this section

**C1. Explain how Kerberos authenticates a user to a service.**
A strong answer must mention:
- The KDC with its Authentication Service and Ticket-Granting Service; shared symmetric keys.
- AS exchange gives a TGT and a session key decryptable only with the user's password-derived key; the password is never sent.
- TGS exchange gives a service ticket; the service validates it with its own key.
- Authenticators (encrypted timestamps) prevent replay, so clocks must be synchronised; the TGT gives single sign-on.
*Common weak answer:* "Kerberos encrypts the password and sends it to the server." The password never crosses the network.

**C2. Compare DAC, MAC and RBAC. State the Bell–LaPadula and Biba rules.**
A strong answer must mention:
- DAC: owners grant access; MAC: central labels the user cannot override; RBAC: permissions through roles.
- Bell–LaPadula for confidentiality: no read up, no write down.
- Biba for integrity: no read down, no write up.
- ACLs and capability lists as columns and rows of the access control matrix.
*Common weak answer:* swapping the two models' rules; remember that Bell–LaPadula protects secrets from leaking down, Biba protects trusted data from being corrupted from below.

---

## Chapter Summary

**Identity** is what a system knows about an entity; **authentication** verifies that a party is that identity, once per session or token; **authorisation** decides, on every request and for every object, whether the authenticated **principal** may act; **federation** lets one system accept another's authentication. A 401 means "not authenticated", a 403 "not allowed", and identity must always come from the authenticated context, never from the request body.

**Passwords** are stored as salted, slow, memory-hard verifiers (**Argon2id**, scrypt, bcrypt), optionally with a **pepper** outside the database, because fast hashes such as SHA-256 let a stolen database be cracked at billions of guesses per second. Online attacks (**brute force**, **credential stuffing**, **spraying**) are met with rate limits, breached-password screening, MFA and progressive delays rather than hard lockouts, and every response is identical for existing and missing accounts to prevent **enumeration**. Reset and verification flows use random, hashed, short-lived, single-use tokens, links built from configuration, and invalidate sessions on password change. **TOTP** derives six-digit codes from a shared secret and the time with HMAC and dynamic truncation; **WebAuthn passkeys** sign a fresh challenge with a private key bound to the site's origin, which makes them **phishing-resistant** and leaves the server holding only public keys.

**Tokens** are opaque (looked up) or self-contained. A **JWT** is readable, signed claims (`iss`, `sub`, `aud`, `exp`); validation pins the **algorithm** and **keys** per issuer (defeating `alg: none` and RS256/HS256 **key confusion**), checks issuer, audience and time with a small clock leeway, and then authorises. JWTs cannot be revoked before expiry without extra state, so access tokens are short-lived and **refresh tokens** rotate on every use with **reuse detection** that revokes the token family. Sessions and JWTs are a trade-off (revocation, size, freshness, cross-service validation, browser exposure), and for browsers a **BFF** with an `HttpOnly` session cookie keeps tokens out of reach of XSS.

**OAuth 2.0** delegates limited access: the **authorization code flow with PKCE** sends a one-time code through the browser and redeems it with a `code_verifier` whose hash was sent first, with an exact `redirect_uri` and a `state` value; **client credentials** serve services; the **implicit flow** died because it put tokens in URLs. **OpenID Connect** adds the **ID token** (who the user is, for the client), discovery and **JWKS** for key distribution, and enables **SSO**; SAML does the same with XML assertions in enterprises.

**Authorisation** uses ACLs, **RBAC**, **ABAC** and **ReBAC**, usually combined: roles for functions, ownership and tenancy for objects. **IDOR/BOLA** is prevented by scoping every query by tenant and owner, denying by default, testing cross-tenant access, and using **row-level security** as a backstop. **Services** authenticate with short-lived workload identities, mTLS and scoped tokens (exchanged per hop), rather than static API keys, and **SSRF** must be kept away from metadata credentials. **Secrets** live in a secret manager, are delivered at run time, never logged or committed, and rotated with an overlap phase. Across all of it, **threat modelling** with STRIDE, **defence in depth**, **least privilege**, the **OWASP Top 10** as a map, and protected **audit trails** turn individual mechanisms into a secure system.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Authentication | The principal proved the claimed identity, at that time, to the strength of its factors | That the session or token has not been stolen since; any permission |
| Authorisation check | This action on this object was allowed by policy | Anything on paths that skip the check |
| Salted slow KDF | Costly offline guessing per user; no shared cracks for equal passwords | Safety of weak or breached passwords; protection from online attacks or phishing |
| Rate limiting and screening | Online guessing slowed; breached passwords refused | Immunity from distributed attacks; zero friction for real users |
| Reset token (hashed, single-use, short-lived) | Proof of inbox control, once, briefly | Security of the inbox itself |
| TOTP | The user had the shared secret in this time window | That the code was entered on the real site |
| WebAuthn / passkey | A domain-bound private key signed this fresh challenge | Session security after login; safe recovery |
| JWT validation (pinned alg, keys, iss, aud, exp) | The trusted issuer signed these claims for this audience, still in date | Current truth of the claims; confidentiality; revocation |
| Short access-token lifetime | Bounded window for a stolen token | Immediate revocation |
| Refresh rotation with reuse detection | A stolen refresh token ends the family when both parties use it | Detection if the real client never refreshes again |
| OAuth code + PKCE | Codes are useless without the verifier; scoped, limited delegation | User authentication to the client; resource-server checks |
| OIDC ID token (with nonce, aud) | This provider authenticated this user for this client and this login | API authorisation; verified email unless `email_verified` |
| Query-level scoping / RLS | Out-of-scope rows are never read or changed by that query | Function-level rules; paths that bypass the query or the role |
| mTLS / workload identity | The caller holds a credential issued to that workload | That the workload is not compromised; authorisation |
| Secret manager | Encrypted, access-controlled, audited storage | That consumers do not leak what they read; rotation by itself |
| Audit trail | Evidence of what was recorded, if protected | Detection unless someone or something reads it |

---

## Interview Drill

**Q1. What is the difference between authentication and authorisation? What is identity federation?**
A strong answer must mention:
- Authentication verifies who the principal is; authorisation decides whether that principal may perform a specific action on a specific resource.
- Authentication happens once per session or token; authorisation on every request and every object.
- 401 for missing or invalid credentials, 403 (or 404) for valid but insufficient ones.
- Federation: a relying party accepts authentication performed by an identity provider (OIDC, SAML, SSO).
*Common weak answer:* "Authentication is login, authorisation is roles." It misses object-level checks, which is where most real failures are.

**Q2. How should passwords be stored? Why is SHA-256 wrong?**
A strong answer must mention:
- Never plaintext or reversible encryption; a per-user random salt; a slow, memory-hard KDF (Argon2id; scrypt or bcrypt) with tuned parameters stored in the verifier string.
- SHA-256 is fast: GPUs test billions of guesses per second, so human-chosen passwords fall quickly even with salts.
- Salts defeat precomputed tables and identical-password grouping; slowness defeats fast guessing; a pepper outside the database defeats attacks with the database alone.
- Upgrade parameters on login; constant-time comparison.
*Common weak answer:* "Hash it with SHA-256 and a salt." Salting does not make a fast hash slow.

**Q3. How do you defend login against brute force and credential stuffing without locking out real users?**
A strong answer must mention:
- Per-account and per-source rate limits before the KDF; progressive delays and challenges rather than hard lockouts (which attackers can abuse).
- Breached-password screening (k-anonymity range queries), MFA, bot detection, new-device notifications.
- Identical responses and timing for unknown users and wrong passwords (dummy verification).
- Capacity: KDF cost × login rate; a separate pool for login.
*Common weak answer:* "Lock the account after five failed attempts." It turns the defence into a denial-of-service tool and does nothing against stuffing across many accounts.

**Q4. How should a password-reset flow work?**
A strong answer must mention:
- Same response whether or not the account exists.
- A random 128-bit+ token, stored hashed, short expiry, single use, invalidated by later resets.
- Link built from configured URLs, not the `Host` header; no third-party content on the reset page.
- On success: new password hash, all sessions and tokens revoked, notification to the user.
*Common weak answer:* "Email a temporary password." It sends a reusable credential in plaintext and often stays valid.

**Q5. How does TOTP work? How do passkeys work, and why do they resist phishing?**
A strong answer must mention:
- TOTP: shared secret; counter = time / 30 s; HMAC-SHA-1; dynamic truncation to 6 digits; accept ±1 step; reject reused steps; limit attempts.
- WebAuthn: per-site key pair in the authenticator; server sends a random challenge; authenticator signs it with the origin and rpId included; server verifies with the stored public key.
- Phishing resistance: a fake site has a different origin, so no valid signature for the real site can be produced; nothing for the user to retype.
- Server stores only public keys; recovery becomes the weak point.
*Common weak answer:* "TOTP and passkeys are both second factors, so they are equally secure." TOTP codes can be relayed by a phishing site; passkeys cannot.

**Q6. What is a JWT? How is it validated, and what are its pitfalls?**
A strong answer must mention:
- Base64url header.payload.signature; signed claims, readable by anyone; `iss`, `sub`, `aud`, `exp`, `iat`, `nbf`, `jti`.
- Validation: allowlisted algorithm and trusted keys chosen by the server per issuer (`kid` selects among them), signature, issuer, audience, time with small leeway, token type, then authorisation.
- Pitfalls: `alg: none` and RS256/HS256 key confusion when the token's header decides; keys referenced by the token; no audience check; clock skew; secrets in the payload; oversized tokens.
- Revocation is impossible before `exp` without extra state.
*Common weak answer:* "Decode it and check the signature." Without pinning the algorithm and checking audience and expiry, a decoded and "verified" token can be forged or misused.

**Q7. Sessions versus JWTs: what are the trade-offs?**
A strong answer must mention:
- Sessions: a store lookup per request, immediate revocation, tiny cookies, always-current data, `HttpOnly` cookies but CSRF defences needed.
- JWTs: local validation across services and organisations, no shared store for access tokens, but no early revocation, larger requests, claims frozen at issuance, key management.
- In browsers, tokens in script-readable storage are exposed to XSS; cookies are exposed to CSRF.
- Common combination: sessions at the edge (BFF), short-lived JWTs between services.
*Common weak answer:* "JWTs are stateless, so they scale better." Refresh and revocation bring state back, and a session lookup in Redis is rarely the bottleneck.

**Q8. What are refresh tokens, and what is refresh-token rotation?**
A strong answer must mention:
- Short-lived access tokens for APIs; longer-lived refresh tokens presented only to the authorisation server.
- Rotation: each use returns a new refresh token and invalidates the old one.
- Reuse detection: presenting a used token revokes the whole family, ending a theft within one refresh interval.
- Refresh tokens are stored hashed with family, status and absolute expiry; a short grace window for retries.
*Common weak answer:* "A refresh token is a token that lasts longer." Without rotation it is just a long-lived bearer credential.

**Q9. Explain the OAuth 2.0 authorization code flow with PKCE.**
A strong answer must mention:
- Roles: resource owner, client, authorisation server, resource server.
- Client creates `code_verifier` and `code_challenge = BASE64URL(SHA-256(verifier))`, plus `state`; redirects to `/authorize` with `response_type=code`, `client_id`, exact `redirect_uri`, `scope`.
- User authenticates and consents; AS redirects back with a one-time, short-lived `code` and `state`; client checks `state`.
- Client posts the code and `code_verifier` to `/token`; AS checks the hash, client and redirect URI; returns access (and refresh) tokens.
- PKCE makes an intercepted code useless; it is required for all clients in current guidance.
*Common weak answer:* "The user logs in and the app gets a token." It misses the code/verifier binding, which is the security of the flow.

**Q10. Why was the implicit flow abandoned?**
A strong answer must mention:
- It returned access tokens in the redirect URL, exposing them to history, logs, referrers and page scripts.
- No way to bind the token to the client, no safe refresh tokens.
- CORS made the code flow with PKCE possible for browser apps; current guidance (RFC 9700, OAuth 2.1 draft) drops implicit and the password grant.
*Common weak answer:* "It was replaced by JWT." Token format and grant type are unrelated.

**Q11. What is the difference between OAuth 2.0 and OpenID Connect? Between an ID token and an access token?**
A strong answer must mention:
- OAuth is delegated authorisation; OIDC adds authentication of the user to the client.
- ID token: a JWT for the client (`aud` = client ID), with `sub`, `auth_time`, `nonce`, `amr`; never sent to APIs.
- Access token: for the resource server, JWT or opaque, carries scopes.
- OIDC adds discovery, JWKS, UserInfo and standard claims; SSO across applications.
*Common weak answer:* "OIDC is OAuth with JWTs." OAuth access tokens can be JWTs too; the difference is the ID token's purpose and audience.

**Q12. RBAC versus ABAC versus ReBAC: when would you use each?**
A strong answer must mention:
- RBAC: permissions through roles; simple and auditable; role explosion when rules depend on context.
- ABAC: rules over user, resource, action and environment attributes; expresses ownership, tenancy, time; harder to audit.
- ReBAC: permissions from a relationship graph (members, folders, sharing); suits collaboration products.
- Typical combination: roles for functions, attribute or ownership rules for objects.
*Common weak answer:* "ABAC is more advanced, so use it." The right model depends on the shape of the rules.

**Q13. What is IDOR (BOLA), and how do you prevent it?**
A strong answer must mention:
- An endpoint loads an object by a client-supplied ID without checking that the principal may access it.
- Random IDs make guessing harder but are not authorisation.
- Fix: scope the query by tenant and owner from the authenticated principal; return 404 for out-of-scope objects; deny by default.
- Backstops: row-level security, cross-tenant tests, audit alerts on denial bursts.
*Common weak answer:* "Use UUIDs instead of integers." That hides the IDs; it does not check access.

**Q14. How do services authenticate to each other?**
A strong answer must mention:
- Not by network location: internal requests can be forged by a compromised service or SSRF.
- Options: API keys (hashed, prefixed, scoped, rotated), OAuth client credentials, mTLS, platform workload identity (SPIFFE-style), with short-lived credentials preferred.
- User context: forward or exchange tokens (RFC 8693) for narrow audiences.
- Protect metadata credentials from SSRF; least-privilege service roles; authorise per service.
*Common weak answer:* "They are on the same VPC, so they trust each other." Network location is not identity.

**Q15. How should secrets be managed?**
A strong answer must mention:
- Never in code, images or logs; delivered at run time from a secret manager using workload identity (or mounted files).
- Encrypted at rest under KMS keys; access-controlled and audited.
- Rotation with an "accept both" overlap phase; dynamic short-lived credentials where possible.
- Leaks: rotate first, then clean up; continuous scanning.
*Common weak answer:* "Put them in environment variables." Acceptable as a delivery mechanism, but says nothing about storage, access, rotation or leaks.

**Q16. What is threat modelling? Walk through the OWASP Top 10.**
A strong answer must mention:
- Threat modelling: data flows, trust boundaries, threats per element (STRIDE: spoofing, tampering, repudiation, information disclosure, denial of service, elevation of privilege), controls with owners.
- OWASP Top 10 (2021): broken access control, cryptographic failures, injection, insecure design, security misconfiguration, vulnerable components, authentication failures, integrity failures, logging and monitoring failures, SSRF.
- Each mapped to a mechanism and a defence; defence in depth and least privilege across them.
*Common weak answer:* reciting the ten names without a mechanism or defence for any of them.

**Q17. What is the CIA triad, and how do the mechanisms of this chapter map to it?**
A strong answer must mention:
- Confidentiality (only authorised parties read data), integrity (no unauthorised or undetected change), availability (authorised users can use the system when needed).
- Application security protects all three across code, data, identities and infrastructure, not only data in transit (which TLS covers, Chapter 8).
- Mapping: authentication and sessions protect integrity of who acts; authorisation and IDOR prevention protect confidentiality and integrity; KDF capacity planning, rate limits and non-lockout defences protect availability.
- STRIDE derives one threat category per property (information disclosure, tampering, denial of service, plus spoofing, repudiation and elevation of privilege).
*Common weak answer:* "Security means encryption." Encryption serves confidentiality (and, with AEAD, integrity) of data; it does nothing for authorisation or availability.

---

## Scenario Exercises

1. A breach notification says your users table, with SHA-256 password hashes and no salts, was copied a week ago. Plan the next 72 hours (users, storage, sessions) and the long-term fix, explaining why each step is in its order.
2. Your login endpoint sees 300 attempts per second from 40,000 IP addresses, almost all for different email addresses, with a 0.3% success rate. Identify the attack, explain why per-account lockout does not help, and design a defence that keeps login usable.
3. A single-page application stores a 24-hour JWT access token in `localStorage`. After an XSS report in a third-party widget, you must reduce the damage of any future XSS. Propose a target architecture and the migration steps.
4. An API accepts JWTs from your identity provider. A penetration tester reports that tokens issued for your **mobile** app's client ID are accepted by your **admin** API, granting admin functions to some users. Explain what checks are missing and the fix.
5. A B2B SaaS product adds a "download all invoices" export. Design its authorisation so that it cannot leak another tenant's data even if a developer forgets a filter in the export query.
6. A nightly job running in your cloud account calls three internal APIs with a long-lived API key stored in the job's configuration. Redesign its identity and credentials.
7. After moving from a single region to two, about 2% of requests in the new region fail with `401 invalid_token`, and errors spike for a minute after every deploy of the identity service. Give two hypotheses and the evidence that separates them.

---

## Diagnosis Lab

**Lab 1: "Everyone gets 401" (401 versus 403)**
- *Symptom:* After a gateway update, the mobile app shows "Please log in again" in a loop for every user; logging in succeeds, then the next API call fails.
- *Hypotheses:* (a) tokens are not being sent; (b) tokens are rejected by validation; (c) the users are authenticated but not authorised, and the client misreads 403 as 401.
- *Evidence:* Gateway logs show `401` with `WWW-Authenticate: Bearer error="invalid_token", error_description="audience mismatch"`. Decoding a token (its payload is readable) shows `"aud":"orders-api"`; the new gateway configuration expects `"aud":"api.shop.example"` for all routes. Requests that reach services directly with the same token succeed.
- *Root cause:* (b): the gateway's audience setting changed; every token is valid but addressed to a different audience. The client correctly treats 401 as "log in again", so the loop is a consequence, not the cause.
- *Fix:* Restore the audience configuration (or issue tokens with the gateway's audience and accept both during transition). Separately, confirm clients treat 403 as "forbidden" (no re-login) and 401 as "re-authenticate".
- *Prevention:* Contract tests between identity provider configuration and gateway validation; error descriptions in 401s logged and graphed by reason.

**Lab 2: Fresh tokens rejected on some hosts (clock skew)**
- *Symptom:* About 2% of API calls fail with `401` within the first minute after login; retrying a minute later works. Only some instances are affected.
- *Hypotheses:* (a) slow key-set propagation; (b) clock skew; (c) a race in token issuance.
- *Evidence:* Failing instances log `token not yet valid (nbf)`. `chronyc tracking` on those instances shows the system clock 94 seconds **behind**; the validation leeway is 30 seconds. The instances were built from an image whose time synchronisation service was disabled.
- *Root cause:* (b): the verifier's clock is earlier than the token's `nbf`/`iat`, so freshly issued tokens look like they come from the future.
- *Fix:* Enable time synchronisation (NTP), alert on clock offset, and keep a small leeway (30–60 s) for normal drift.
- *Prevention:* Clock offset as a host health check; images tested for NTP.

**Lab 3: A token-expiry storm**
- *Symptom:* Every day at about 09:00, the identity service's token endpoint saturates for two minutes, refresh requests time out, and some users are logged out.
- *Hypotheses:* (a) a login surge as people start work; (b) synchronised token expiry; (c) a slow database under the identity service.
- *Evidence:* Token endpoint traffic is 95% `grant_type=refresh_token`, not logins. Access tokens have a fixed 60-minute lifetime and were all issued at 08:00, when a nightly job revoked and re-issued every session; clients refresh exactly at expiry, with no jitter, and retry immediately on timeout, multiplying the load. Some retried refreshes present an already-rotated token and trigger reuse detection, which revokes the family and logs the user out.
- *Root cause:* (b), amplified by retries: synchronised expiry produces a thundering herd of refreshes, and immediate retries plus strict reuse detection turn overload into logouts.
- *Fix:* Clients refresh at a random point in the last 10–20% of the lifetime; retries use exponential backoff with jitter (Chapter 24); a short reuse grace window returns the same new token pair for a repeated refresh within seconds; remove the mass re-issue job or spread it over hours.
- *Prevention:* Load tests of the refresh path at peak; alerting on refresh rate and reuse-detection rate.

**Lab 4: Privilege escalation through an IDOR**
- *Symptom:* A customer reports seeing another company's invoice PDF in their account's "recent downloads".
- *Hypotheses:* (a) a cache serving one tenant's file to another; (b) an IDOR in the download endpoint; (c) a data-import mistake.
- *Evidence:* The audit log shows user `u_5512` (tenant 31) downloading 2,840 invoices over two days through `GET /v1/invoices/{id}/pdf` with sequential IDs, of which 2,790 belong to other tenants. The endpoint's code loads the invoice by ID and checks only that the user is authenticated; the list endpoint filters by tenant, which is why testing through the UI never showed the problem. The CDN does not cache this path.
- *Root cause:* (b): broken object-level authorisation on one endpoint.
- *Fix (now):* Deploy the scoped query (`WHERE id = $1 AND tenant_id = $2`) returning 404; revoke the user's sessions and tokens pending investigation; determine affected tenants from the audit log and notify them as required. *Fix (lasting):* repository methods that require the principal; row-level security on tenant tables; an automated cross-tenant test for every endpoint that takes an ID; an alert on bursts of cross-tenant denials.
- *Prevention:* Deny-by-default authorisation middleware; threat modelling of new endpoints (Section 20.12).

---

## Solution Sketches

1. Unsalted SHA-256 hashes are crackable quickly for most real passwords, so assume many are already known. Immediately: force a password reset for all users at next login (and invalidate all sessions and refresh tokens), notify users, advise changing reused passwords elsewhere, and screen new passwords against breach corpora. Protect the stored data now by wrapping every existing hash as `argon2id(sha256(password))` in one migration (no plaintext needed), then replace with direct Argon2id hashes as users log in or reset. Long term: a password library with current parameters, a pepper in a secret manager, MFA, and an investigation of how the table was copied (injection, backups, access).
2. Credential stuffing: leaked username–password pairs from other sites, spread across accounts and IPs, so per-account counters rarely trigger and lockout would only hurt the real owners of attacked accounts. Defence: breached-password screening at login and registration, MFA (prompt on risky logins), bot detection and device fingerprinting, per-network rate limits with care for shared NAT, a global anomaly alarm on failure rate, notifications to users on new-device logins, and capacity protection for the KDF (rate limit before verification).
3. Target: a BFF on the same site holds tokens server-side; the browser gets an `HttpOnly`, `Secure`, `SameSite=Lax`, `__Host-` session cookie, with CSRF defences on state-changing routes (Chapter 19, Section 19.6). Short-lived access tokens with rotating refresh tokens stay on the server. Migration: introduce the BFF endpoints, move API calls through it, shorten token lifetimes in the meantime (24 h → 15 min with refresh), revoke existing long-lived tokens, and add a CSP that restricts third-party scripts (Chapter 9, Section 9.8).
4. The admin API is not checking the audience (and perhaps not the client or scopes): any token signed by the provider is accepted. Fix: the admin API accepts only tokens whose `aud` is the admin API, with the admin scope, issued to the admin client; RBAC checks on admin functions; reject ID tokens; add tests for tokens of other audiences.
5. Enforce tenancy at several layers: the export service reads the tenant from the authenticated principal only; repository methods require a tenant parameter; row-level security on invoice tables with `SET LOCAL app.tenant_id` per transaction, connecting as a role without `BYPASSRLS`; the export file is written to a tenant-scoped storage prefix and served through short-lived signed URLs; an automated test that a user of tenant A exporting with RLS active receives only tenant A's rows even if the query omits the filter; an audit event per export.
6. Give the job a workload identity from the platform (a service account or cloud role), obtain short-lived OAuth access tokens with client credentials authenticated by that identity (or mTLS), with separate audiences and minimal scopes for each API; remove the static key from configuration and revoke it; if a static key must remain for one external system, store it in the secret manager, fetch it at run time, and rotate it on a schedule.
7. Hypothesis A: the new region's API instances validate against a cached key set that lags behind the identity service's key rotations, so tokens signed with a new `kid` fail until the cache refreshes; the spike after identity deploys fits if deploys rotate keys or restart with new keys. Evidence: failure reasons logged as "unknown kid", timing relative to JWKS cache refresh, both regions' JWKS contents. Hypothesis B: clock skew between regions makes fresh tokens "not yet valid"; evidence: failure reason `nbf`/`iat`, clock offsets on the new region's hosts. Fixes: refresh JWKS on unknown `kid` (rate-limited) and publish new keys before signing with them; time synchronisation and a small leeway.

---

**END OF CHAPTER 20**
