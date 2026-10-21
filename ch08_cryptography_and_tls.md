# Chapter 8: Cryptography Essentials and HTTPS/TLS

## Where We Are

Chapter 6 traced bytes across switches, routers, NAT gateways and TCP connections, and Chapter 7 gave those bytes meaning as HTTP messages. Look again at the wire traces in Section 7.16: every header, every cookie, every `Authorization` token and every JSON body crossed the network as readable text. Anyone on the path (the Wi-Fi access point in a café, a router at an ISP, a compromised machine in a data centre) could have **read** them, **changed** them (the TCP checksum of Chapter 6 detects accidents, not deliberate edits, because an attacker simply recomputes it), or **impersonated** the server by redirecting the client with forged ARP replies or DNS answers (Sections 6.2 and 6.6).

This chapter is about the tools that stop that. **Cryptography** provides a small set of building blocks: random numbers, hashes, message authentication codes, symmetric encryption, and public-key operations (key exchange and signatures). **TLS** (Transport Layer Security) combines them into the protocol that turns `http://` into `https://`. The same building blocks reappear all over backend systems: webhook signatures, signed cookies, JSON Web Tokens, encrypted database columns, cloud key management, password storage and service-to-service authentication. Understanding what each one proves, and what it does not, is what lets you use them correctly.

There is no heavy mathematics here. The goal is intuition: why each primitive has its property, and which attack it stops.

**After this chapter you can:**
- state the security goals (confidentiality, integrity, authenticity, non-repudiation) and what passive and active attackers can do;
- distinguish encoding, hashing, encryption, MACs and signatures by whether they use a key, whether they can be reversed, and what they prove;
- explain why backend code needs a cryptographically secure random number generator;
- explain block and stream ciphers, why ECB mode leaks patterns, how AEAD (AES-GCM, ChaCha20-Poly1305) combines encryption and authentication, and why a nonce must never repeat;
- state the properties of a cryptographic hash, and explain why fast hashes are wrong for passwords;
- explain how HMAC proves integrity and authenticity, and where it is used (webhooks, cookies, JWT HS256);
- explain public-key cryptography, and the difference between encrypting and signing;
- work through Diffie–Hellman with small numbers, and explain ECDHE and forward secrecy;
- read an X.509 certificate, trace a chain of trust, list the validation steps, and explain revocation, Certificate Transparency and ACME;
- trace the TLS 1.3 handshake, and explain resumption, 0-RTT and its replay risk, SNI and ALPN;
- compare TLS termination at a load balancer, re-encryption, passthrough and mutual TLS;
- explain HSTS, mixed content and certificate-expiry outages;
- list what HTTPS does **not** protect;
- diagnose expired, incomplete, mismatched and untrusted certificates and protocol mismatches with `openssl s_client`.

**Interview questions this chapter prepares you for:**
1. What is the difference between encoding, hashing and encryption? Is Base64 encryption?
2. What is the difference between symmetric and asymmetric encryption? Why doesn't TLS encrypt everything with RSA?
3. Why is ECB mode insecure? What is AEAD, and why must a nonce never be reused?
4. What properties does a cryptographic hash have? Why is SHA-256 wrong for storing passwords?
5. What is a MAC or HMAC? How is it different from a digital signature?
6. How can two parties agree on a secret key over a network an attacker is recording?
7. What is forward secrecy?
8. What is in a certificate, why are there intermediate CAs, and how does a client validate a certificate?
9. Walk through the TLS 1.3 handshake. How is it faster than TLS 1.2? What is the risk of 0-RTT?
10. What are SNI and ALPN?
11. What is the difference between terminating TLS at a load balancer, passing it through, and mutual TLS?
12. Why does an HTTP-to-HTTPS redirect still leave the first request exposed, and how does HSTS fix it?
13. Why does a browser accept a certificate that your backend client rejects with "unable to get local issuer certificate"?
14. What does HTTPS not protect against?

**Scope.** Here: security goals, the cryptographic primitives and what each proves, random numbers, PKI and certificates, the TLS 1.3 protocol, TLS deployment topologies including mTLS, HSTS and related browser behaviour, and TLS diagnosis. Not here: the mathematics behind the primitives. Browser security policies (same-origin policy, CORS, XSS, CSP) are Chapter 9. Load balancers and proxies as components are Chapter 10. Password storage in full (Argon2id, bcrypt, salts, peppers), JWT validation, OAuth and service identity are Chapter 20; sessions and CSRF are Chapter 19. Secrets management is Chapter 20.

---

## 8.1 Security Goals and the Threat Model

### Definitions

- **Confidentiality**: unauthorised parties cannot read the data.
- **Integrity**: any unauthorised change to the data (modification, insertion, deletion, reordering) is detected.
- **Authenticity**: the data, or the party you are talking to, really comes from who it claims to.
- **Non-repudiation**: the author of a message cannot later credibly deny, to a third party, having produced it.

A **threat model** is an explicit statement of what an attacker can do, which decides which goals a design must meet. Two network attackers matter here:

- A **passive attacker** (eavesdropper) can record and read traffic but not change it.
- An **active attacker**, also called a **man-in-the-middle** (MITM), controls a point on the path and can read, change, drop, delay, replay and inject traffic in both directions.

### Why it exists

When people say "secure this connection", they usually mean only confidentiality. In backend systems integrity and authenticity matter at least as much:

1. **Confidentiality without integrity is broken.** Some encryption schemes (stream ciphers, Section 8.3) encrypt by combining each plaintext byte with a secret byte. An attacker who cannot read the message, but knows where the amount sits in it, can flip bits in the ciphertext and thereby flip the corresponding bits of the decrypted amount. Without an integrity check, the server accepts the altered value.
2. **Encryption without authenticity is useless.** A client that sets up a perfectly encrypted channel *with the attacker* (because nothing proved who was at the other end) hands the attacker its password, encrypted. The attacker decrypts it, re-encrypts it on a second channel to the real server, and relays everything in between.

Security design therefore starts from the **active attacker**: assume the network is hostile, every byte can be read and changed, and any party can claim any identity.

### Mechanism: four attacks and what stops each

```text
 1. EAVESDROPPING
    client ────── "Authorization: Bearer eyJ..." ──────► server
                            │ attacker records it
    stopped by: CONFIDENTIALITY (encryption, Section 8.3)

 2. TAMPERING
    client ── "qty": 1 ──► [attacker changes it to "qty": 999] ──► server
    stopped by: INTEGRITY (authentication tags, MACs, signatures; Sections 8.3, 8.5, 8.6)

 3. IMPERSONATION
    client ── "I think you are the bank" ──► [attacker] ── second connection ──► bank
    stopped by: AUTHENTICITY (certificates and signatures, Sections 8.6, 8.8)

 4. REPLAY AND DOWNGRADE
    replay:    the attacker records a valid encrypted "pay Bob 50" and sends it again
    downgrade: the attacker edits the opening messages so both sides pick an old, weak option
    stopped by: sequence numbers and nonces inside the authenticated data, and a signed or
                MAC-protected record of the whole handshake (Section 8.9)
```

Replay is easy to overlook: a message can be both encrypted and authenticated and still be harmful when it arrives twice. That is why TLS includes a sequence number in every record's authentication (Section 8.9), and why application tokens carry expiry times and one-time identifiers.

### Guarantees and non-guarantees

Together, confidentiality, integrity, authenticity and replay protection guarantee that two endpoints can exchange data across a hostile network such that no one on the path can read, alter, reorder, inject or replay it undetected, and that the peer is who its credentials say. They do **not** say anything about the endpoints themselves: once data is decrypted inside a server, a bug or a compromised process can still leak or misuse it (Section 8.12).

### Failure and diagnosis

**Home-made cryptography.** A team "protects" a mobile API by XOR-ing payloads with a secret string, or by inventing its own shuffling scheme, instead of using TLS and standard primitives. Anyone who decompiles the app recovers the scheme, and then the data. The principle violated is **Kerckhoffs's principle**: a cryptographic system must remain secure even if everything about it except the key is public. Evidence in review: any custom transformation described as "encryption", any key embedded in client code. Fix: standard primitives from a maintained library, keys kept out of code (Chapter 20).

### Backend connection

For any security-sensitive design (an API, a webhook, a signed URL, a token), ask four questions in order. Who must not read it (confidentiality)? Who could change it (integrity)? How do I know who produced it (authenticity: a shared secret or a public-key signature)? What happens if a valid copy is sent again five seconds or five days later (replay: expiry, nonces, idempotency keys)?

---

## 8.2 Encoding, Hashing, Encryption, MACs, Signatures, and Randomness

### Definitions

- **Encoding** is a public, keyless, reversible change of representation (Base64, hexadecimal, UTF-8, percent-encoding) for transport or storage. It provides **no security**.
- A **cryptographic hash function** is a keyless, one-way function that maps input of any length to a fixed-size **digest** (32 bytes for SHA-256), such that finding an input for a given digest, or two inputs with the same digest, is infeasible.
- **Encryption** is a keyed, reversible transformation of **plaintext** into **ciphertext** so that only holders of the right key can recover the plaintext. It provides confidentiality.
- A **MAC** (message authentication code) is a short **tag** computed from a message and a **shared secret key**; anyone with the key can recompute it to check that the message is unchanged and came from a key holder (integrity and authenticity).
- A **digital signature** is a tag computed from a message with a **private key** that anyone can verify with the matching **public key** (integrity, authenticity and non-repudiation).
- A **CSPRNG** (cryptographically secure pseudo-random number generator) produces random bytes that an attacker cannot predict, even after seeing many earlier outputs.

### Why this taxonomy matters

Saying "we encrypt passwords with SHA-256", "the token is Base64-encrypted" or "anyone can verify the HMAC with the public key" in an interview signals that the speaker does not know which tool does what. Each of these operations takes different inputs, uses a different kind of key (or none), and answers a different question.

### Mechanism 1: the comparison table

| Operation | Key | Reversible? | Proves or protects | Does **not** | Examples and uses |
|---|---|---|---|---|---|
| Encoding | none | yes, by anyone | safe transport of bytes through text channels | hide anything, or detect changes | Base64 in `Authorization: Basic`, JWT segments, hex digests |
| Hash | none | no | a fingerprint: equal inputs give equal digests | prove who produced the data (an attacker can hash their own data) | SHA-256: file checksums, container image digests, Git object IDs |
| Password hash | none, but a per-user random **salt** | no, and deliberately slow | resistance to guessing if a password database leaks | serve as a fast general-purpose hash | Argon2id, scrypt, bcrypt (Chapter 20) |
| MAC (HMAC) | one shared secret | no | integrity and authenticity between key holders | hide the message; let third parties verify without the secret | HMAC-SHA256: webhook signatures, signed cookies, JWT HS256 |
| Signature | private key signs, public key verifies | no | integrity, authenticity, non-repudiation; anyone can verify | hide the message | Ed25519, ECDSA, RSA-PSS: certificates, JWT RS256/ES256, software packages |
| Symmetric (AEAD) encryption | one shared secret, plus a nonce | yes, with the key | confidentiality, integrity and authenticity, fast | help two parties who do not already share a key | AES-GCM, ChaCha20-Poly1305: TLS records, encrypted columns |
| Public-key encryption / key agreement | public and private key pair | yes, with the private key (or derives a shared secret) | lets parties without a shared secret establish one | encrypt bulk data efficiently | ECDHE (X25519) in TLS; RSA-OAEP to wrap small keys |

### Concrete: Base64 is not encryption

```text
 input (28 bytes):  {"userId":42,"role":"admin"}
 Base64 (40 chars): eyJ1c2VySWQiOjQyLCJyb2xlIjoiYWRtaW4ifQ==
 decoding:          echo 'eyJ1c2VySWQiOjQyLCJyb2xlIjoiYWRtaW4ifQ==' | base64 -d
                    → {"userId":42,"role":"admin"}
```

Base64 maps every 3 bytes to 4 characters from a 64-character alphabet; it needs no key and anyone can reverse it. It is why so many tokens start with `eyJ`: a JSON object that begins with `{"` followed by a letter always encodes to those three characters. A **JSON Web Token** (JWT, RFC 7519) is, by default, three Base64url segments, header, payload and signature, and its payload is readable by anyone who has the token. The signature (a MAC or a digital signature) stops the holder from *changing* `"role":"user"` to `"role":"admin"`; it does not hide anything. Never put secrets in a JWT payload or in a Base64 cookie. (Chapter 20 covers JWT validation.)

### Mechanism 2: secure randomness

Every key, nonce, salt, session identifier and password-reset token depends on one property: an attacker must not be able to predict it.

Programming languages offer two kinds of random number generator:

1. **General-purpose PRNGs** (`java.util.Random`, JavaScript's `Math.random()`, Python's `random`) are designed for speed in simulations and for things like retry jitter. They are deterministic formulas with a small internal state. From a handful of outputs, an attacker can often reconstruct the state and then compute every past and future output. Using one for tokens lets an attacker predict the next password-reset link.
2. **CSPRNGs** are built from cryptographic primitives and seeded by the operating system from unpredictable events (hardware noise sources, interrupt timing). On Linux, the kernel's generator is reached through the `getrandom` system call or `/dev/urandom`; once seeded at boot, it does not block. Seeing any number of outputs gives no useful information about the next one.

> **Java lens:** Use `java.security.SecureRandom` for keys, tokens, nonces and salts, never `java.util.Random` or `ThreadLocalRandom`. `UUID.randomUUID()` (version 4) is generated with `SecureRandom` and carries 122 random bits. Old advice to point the JVM at `/dev/random` (`-Djava.security.egd=...`) dates from kernels where it could block; on current Linux kernels both devices use the same non-blocking generator once it is initialised.
>
> **Other runtimes:** Node.js: `crypto.randomBytes()` or `crypto.randomUUID()`. Python: the `secrets` module. Go: `crypto/rand`, not `math/rand`.

### Guarantees and non-guarantees

A CSPRNG guarantees unpredictable output. It does not make a short value strong: a 6-digit code has only 1,000,000 possibilities however good the generator, so it must be rate-limited and expire quickly. And no generator protects a token that is logged in plain text.

### Failure and diagnosis

**Predictable tokens.** Reset or invitation tokens are generated with a general-purpose PRNG, or are UUID version 1 (built from a timestamp and the machine's MAC address, not from randomness), or are a timestamp hashed with SHA-256 (a hash of something guessable is guessable). Evidence in review: grep the code that creates tokens, sessions, keys and nonces for `new Random(`, `Math.random`, `uuid1`, `System.currentTimeMillis`. Fix: 16–32 bytes from a CSPRNG, encoded with Base64url or hex.

### Backend connection

A one-line rule that prevents a whole class of incidents: in any code path that creates something an attacker must not guess, the only acceptable source of randomness is the platform's CSPRNG.

---

## 8.3 Symmetric Encryption and AEAD

### Definitions

**Symmetric encryption** uses the **same secret key** to encrypt and to decrypt. Most modern schemes also take a **nonce** ("number used once"): a value, not secret, that must be different for every message encrypted under the same key, so that encrypting the same plaintext twice gives different ciphertexts. Some modes call it an **IV** (initialisation vector).

A **block cipher** is a keyed permutation of fixed-size blocks: it maps each possible 16-byte input block to a 16-byte output block, in a way that looks random without the key. **AES** (Advanced Encryption Standard) is the standard block cipher: 128-bit blocks, with 128-, 192- or 256-bit keys. A **mode of operation** defines how to use a block cipher on messages longer than one block.

A **stream cipher** generates a long pseudo-random **keystream** from a key and a nonce, and combines it with the plaintext using XOR (exclusive or: each output bit is 1 when the two input bits differ). **ChaCha20** is the standard stream cipher.

**AEAD** (authenticated encryption with associated data) is a single operation that encrypts the plaintext *and* computes an **authentication tag** over the ciphertext and over optional **associated data** (AAD): data that stays unencrypted but is protected against change. Decryption checks the tag first and releases no plaintext if anything was altered. The two AEADs in common use are **AES-GCM** and **ChaCha20-Poly1305**.

### Why it exists

Symmetric encryption is the workhorse: it encrypts every byte of HTTPS traffic after the handshake, and most data at rest. It is fast: modern CPUs have AES instructions (AES-NI on x86, cryptography extensions on ARM) that encrypt at gigabytes per second per core, far faster than public-key operations (Section 8.6). Its limitation is that both sides must already share the key, which is the problem Section 8.7 solves.

AEAD exists because encryption alone gives confidentiality only. Section 8.1 showed that an attacker who can change ciphertext can change plaintext; AEAD makes every change detectable.

### Mechanism 1: why ECB leaks patterns

The most naive mode, **ECB** (electronic codebook), cuts the message into 16-byte blocks and encrypts each one separately with the same key. Since a block cipher is deterministic, **equal plaintext blocks give equal ciphertext blocks**:

```text
 plaintext blocks (16 bytes each)         ciphertext blocks
 "role":"admin",.                ──AES─►   8f a2 19 04 c1 ...
 "dept":"sales",.                ──AES─►   3b 77 e0 11 9a ...
 "role":"admin",.                ──AES─►   8f a2 19 04 c1 ...   identical to block 1
```

Two consequences:

1. **Patterns survive encryption.** Encrypt an uncompressed image with ECB, and every region of identical pixels becomes a region of identical ciphertext blocks. The shapes in the picture stay visible in the encrypted file; this is the well-known "ECB penguin" illustration. The same is true of structured records: an observer can see which records have equal fields.
2. **Blocks can be moved.** Because blocks are independent, an attacker can cut and paste ciphertext blocks between messages, assembling a new message that decrypts to valid-looking data, without knowing the key.

Never use ECB. Beware of APIs that choose it silently: in Java, `Cipher.getInstance("AES")` with the default provider means `AES/ECB/PKCS5Padding`.

### Mechanism 2: modes with nonces, and why they still need authentication

Modes such as **CBC** (cipher block chaining, which mixes each plaintext block with the previous ciphertext block, starting from a random IV) and **CTR** (counter mode, which encrypts a nonce plus a counter to produce a keystream and XORs it with the plaintext, turning a block cipher into a stream cipher) fix the pattern leak. But by themselves they provide **no integrity**:

- In CTR mode and in any stream cipher, ciphertext = plaintext XOR keystream. Flipping a bit of the ciphertext flips the same bit of the decrypted plaintext. An attacker who knows the layout of a message can change it predictably.
- CBC decryption ends with a check that the last block's padding is well formed. If a server reveals, through different error messages or measurable timing, whether the padding of a tampered ciphertext was valid, that one bit of information per attempt can be enough to decrypt messages without the key: the **padding oracle attack**, which broke real systems, including older versions of TLS.

The lesson the industry drew is that encryption must always be authenticated, and the authentication checked before anything else is done with the data. That is AEAD.

### Mechanism 3: how AEAD works

```text
 ENCRYPT(key K, nonce N, plaintext P, associated data AAD)
   K    32 bytes (AES-256) or 16 bytes (AES-128); secret
   N    12 bytes; unique for every message under K; sent in the clear
   P    any length; will be hidden
   AAD  any length; stays readable, but is covered by the tag
        (for example a record header, a database row ID)
            │
            ▼
   ciphertext C (same length as P)   +   tag T (16 bytes), computed over AAD and C

 DECRYPT(K, N, C, AAD, T)
   1. recompute the tag over AAD and C, and compare it with T in constant time
   2. if they differ, because any bit of C, AAD, N or T changed: fail, release NOTHING
   3. otherwise decrypt C and return P
```

- **AES-GCM** (Galois/counter mode) is CTR-mode encryption plus a fast polynomial MAC called GHASH; with hardware support it is very fast.
- **ChaCha20-Poly1305** (RFC 8439) is the ChaCha20 stream cipher plus the Poly1305 MAC. It is fast in plain software and naturally resistant to timing side channels, so it is preferred on devices without AES hardware (many phones and small devices). TLS 1.3 offers both.

**Associated data in practice.** Suppose each row of a `users` table stores an encrypted bank account number. What stops someone with database access from copying the encrypted value from the CEO's row into their own row, so that the application decrypts the CEO's number onto their screen? Encrypt with the row's ID as AAD (`"user_id:42"`). The value is not secret and is not stored inside the ciphertext, but it is bound into the tag: decrypting the copied value with AAD `"user_id:99"` fails the tag check.

### Mechanism 4: never reuse a nonce

The one rule of AES-GCM and ChaCha20-Poly1305: **never encrypt two different messages with the same key and nonce.**

Both are stream-cipher constructions: the keystream depends only on the key and the nonce. With the same pair:

```text
 C1 = P1 XOR KS(K, N)
 C2 = P2 XOR KS(K, N)
 C1 XOR C2 = P1 XOR P2          the keystream cancels out
```

An observer of both ciphertexts learns the XOR of the two plaintexts, from which structured text such as JSON is usually easy to recover. For AES-GCM it is worse: a repeated nonce also lets an attacker recover the internal authentication key and **forge valid tags** for messages of their choosing under that key.

Safe ways to choose nonces:

- **Counters**, in protocols that keep state. TLS derives each record's nonce from a per-connection secret IV combined with the record's 64-bit sequence number, so a nonce never repeats within a connection (Section 8.9).
- **Random 12-byte nonces**, for independent encryptions such as database fields, stored next to the ciphertext. With random 96-bit nonces the chance of a repeat grows with the number of messages, so guidance (from NIST, for GCM) limits one key to about 2^32 messages; rotate keys, or use one data key per object (Concrete, below).
- **Constructions with larger or misuse-resistant nonces**: XChaCha20-Poly1305 uses a 24-byte nonce, which makes random repeats negligibly unlikely; AES-GCM-SIV (RFC 8452) limits the damage if a nonce does repeat.

### Concrete: envelope encryption with a key management service

How do backends encrypt files and records with keys held in a **KMS** (key management service: AWS KMS, Google Cloud KMS, HashiCorp Vault) when a KMS call is a network round trip and accepts only small payloads? With **envelope encryption**, a two-level hierarchy of a **data encryption key** (DEK) and a **key encryption key** (KEK):

```text
 ENCRYPT a 50 MB file
 1. ask the KMS for a new data key under master key "prod-kek"
    the KEK never leaves the KMS; the KMS returns:
      (a) the plaintext DEK (32 bytes), kept only in the application's memory, briefly
      (b) the same DEK encrypted under the KEK ("wrapped DEK")
 2. encrypt the file locally with AES-256-GCM using the plaintext DEK and a fresh nonce
 3. discard the plaintext DEK; store:
      ┌──────────────┬──────────┬──────────────────────────┬──────────┐
      │ wrapped DEK  │ nonce    │ ciphertext (50 MB)       │ tag (16) │
      └──────────────┴──────────┴──────────────────────────┴──────────┘

 DECRYPT
 1. send only the small wrapped DEK to the KMS, which checks permissions and unwraps it
 2. decrypt the file locally with the DEK
```

Why it is the standard design for data at rest:

1. **Performance.** The file never travels to the KMS; it is encrypted locally at hardware speed.
2. **Fresh keys.** Every object gets its own DEK, so no single key encrypts enough data to approach nonce limits.
3. **Control.** Access to data now depends on permission to use the KEK, which the KMS checks and logs on every unwrap. Rotating the KEK means re-wrapping the small DEKs, not re-encrypting all the data, and revoking access to the KEK makes stolen copies of the data useless.

> **Java lens:** AES-GCM in the JDK is `Cipher.getInstance("AES/GCM/NoPadding")` initialised with a `GCMParameterSpec(128, nonce)` (a 128-bit tag and a 12-byte nonce from `SecureRandom`); `updateAAD` adds associated data. A failed tag check throws `AEADBadTagException`.

### Guarantees and non-guarantees

AEAD with a unique nonce guarantees confidentiality of the plaintext and integrity and authenticity of the ciphertext and the associated data. It does not hide the **length** of the plaintext (the ciphertext has the same length, plus the tag), does not protect against replay of a whole valid message by itself (that needs sequence numbers or identifiers in the AAD), and provides nothing if the key leaks or a nonce repeats.

### Failure and diagnosis

- **"Tag mismatch" on every decryption** (`AEADBadTagException: Tag mismatch` in Java). Wrong key, wrong nonce, wrong AAD, or a truncated ciphertext. A common bug: the code generated a random nonce for encryption but did not store it, and generates a new one for decryption. Fix: store the nonce with the ciphertext (`nonce || ciphertext || tag`), and pass identical AAD on both sides.
- **Keys in source code or configuration files.** An encryption key committed to a repository protects nothing once the repository is copied. Fix: keys in a KMS or secrets manager (Chapter 20), envelope encryption, rotation.

### Backend connection

To encrypt a sensitive field (a refresh token, a tax identifier, a third party's API secret), use AES-256-GCM or ChaCha20-Poly1305 from a standard library, a fresh random nonce stored with the value, the row's primary key as associated data, and a key managed through a KMS with envelope encryption.

---

## 8.4 Cryptographic Hash Functions

### Definition

A **cryptographic hash function** H maps an input of any length to a fixed-length digest (**SHA-256**: 256 bits = 32 bytes, usually shown as 64 hexadecimal characters). It is deterministic and keyless, and it is secure if three things are infeasible:

1. **Preimage resistance**: given a digest h, find any input m with H(m) = h.
2. **Second-preimage resistance**: given an input m1, find a different m2 with H(m2) = H(m1).
3. **Collision resistance**: find *any* two different inputs with the same digest.

Changing a single input bit changes about half of the output bits, unpredictably (the **avalanche** property).

### Why it exists

Hashes give a short, unforgeable-in-practice fingerprint of data. If you know the correct digest of a 2 GB container image from a trustworthy source, you can verify a downloaded copy by hashing it and comparing 32 bytes. Hashes are also building blocks: MACs (Section 8.5), signatures (which sign a digest rather than the whole message, Section 8.6), certificate fingerprints and key derivation.

### Mechanism: what the properties mean in practice

```text
 SHA-256("order:42:amount:100") = 4f0c...e1     (32 bytes)
 SHA-256("order:42:amount:101") = b97a...03     (completely different)
```

- For a good n-bit hash, finding a preimage takes about 2^n attempts (2^256 for SHA-256, far beyond any computation).
- Finding a collision takes only about 2^(n/2) attempts, because of the **birthday bound**: among many random values, a repeat appears much sooner than intuition suggests (in a room of 23 people, two probably share a birthday). For SHA-256 that is 2^128, still infeasible.
- **Broken hashes.** MD5 collisions can be produced cheaply, and a SHA-1 collision was publicly demonstrated in 2017. Neither may be used where collisions matter (signatures, certificates, deduplication of untrusted data). They survive only as non-security checksums. Use SHA-256, SHA-384/512, SHA-3 or BLAKE2/BLAKE3.
- **Non-cryptographic hashes** (CRC32, MurmurHash, xxHash, a language's `hashCode`) are designed for speed in hash tables, partitioning and error detection. They have no resistance against an attacker who chooses inputs.

### Why fast hashes are wrong for passwords

A hash is one-way, so storing `SHA-256(password)` looks safe. It is not, because passwords are not random: people choose from a small space of likely values. An attacker who steals the table does not reverse the hash; they **guess**, hashing candidate passwords and comparing. SHA-256 is designed to be fast, and a single modern GPU computes it on the order of billions of times per second, so dictionary words, common passwords and their variations fall quickly. Adding a per-user random **salt** stops the attacker from cracking all users at once with precomputed tables, but each password can still be guessed at full speed.

Password storage therefore uses functions that are **deliberately slow and memory-hungry**, with a tunable cost: **Argon2id**, **scrypt** or **bcrypt**. Chapter 20 covers how to use them.

### Guarantees and non-guarantees

A secure hash guarantees that equal digests mean, in practice, equal inputs, and that the digest reveals nothing useful about an unpredictable input. It does **not** prove who produced the data: anyone can hash anything, so an attacker who replaces a file can publish its new digest too. Proving origin needs a key: a MAC (Section 8.5) or a signature (Section 8.6). And a hash does not hide a *predictable* input: hashing an email address, a phone number or a card number does not anonymise it, because the attacker can hash every candidate.

### Failure and diagnosis

- **"Anonymised" identifiers that are not.** A data export replaces phone numbers with their SHA-256 digests. Since there are only about ten billion possible phone numbers in a national format, an attacker hashes them all and reverses the export. Fix: a keyed function (HMAC with a secret key, Section 8.5) if values must be matchable, or real removal of the data.
- **Integrity checks that check nothing.** A download page lists a file's SHA-256 next to the file on the same server. An attacker who can replace the file can replace the digest. Fix: a signature over the file, or a digest obtained from a separate trusted channel.

### Backend connection

Use SHA-256 for fingerprints, content addressing and integrity of data from trusted sources; HMAC when the producer must be authenticated; signatures when anyone must be able to verify; and a password-hashing function, never a plain hash, for passwords.

---

## 8.5 Message Authentication Codes and HMAC

### Definition

A **MAC** is a function MAC(K, m) that takes a secret key K and a message m and produces a short tag. Without K, it is infeasible to produce a valid tag for any message you have not seen tagged. A receiver who shares K recomputes the tag and compares; a match proves the message is unchanged and was tagged by someone holding K.

**HMAC** (RFC 2104) is the standard way to build a MAC from a hash function, such as **HMAC-SHA256**.

### Why it exists

A hash proves nothing about origin (Section 8.4). Many systems need to prove origin between parties who share a secret: a payment provider calling your webhook endpoint, your own server receiving back a cookie it issued, a service checking a token issued by another service in the same organisation. A MAC does exactly that, cheaply.

### Mechanism 1: why not just hash the secret with the message?

The obvious construction, tag = SHA-256(secret || message) (where `||` means concatenation), is broken for SHA-256 and other hashes of the same family. These hashes process input in blocks and output their internal state at the end, so anyone who knows a valid tag (and the message length) can **continue** the computation: they can compute a valid tag for the message with extra data appended, without knowing the secret. This is a **length-extension attack**.

HMAC avoids it by hashing twice, with the key mixed in at both ends:

```text
 HMAC(K, m) = H( (K' XOR opad) || H( (K' XOR ipad) || m ) )

 K'          the key, padded (or first hashed, if long) to the hash's block size
 ipad, opad  two fixed constants
```

The outer hash, keyed again, hides the inner hash's state, so length extension is impossible. HMAC remains secure even with hash functions whose collision resistance has weakened, which is why it is used everywhere.

### Mechanism 2: verifying a webhook

A provider sends events to your endpoint, and both sides hold a shared webhook secret. Many providers sign a timestamp together with the body, to prevent replay:

```text
 provider                                              your endpoint
 body = {"id":"evt_91","type":"payment.succeeded",...}
 t    = 1792240000                                     (Unix time)
 sig  = hex(HMAC-SHA256(secret, t + "." + body))
 POST /webhooks/payments
 Signature: t=1792240000,v1=5257a869e7ec... ─────────► 1. read the raw body bytes, exactly as received
                                                       2. reject if |now − t| > 5 minutes (replay window)
                                                       3. expected = HMAC-SHA256(secret, t + "." + raw body)
                                                       4. compare expected with v1 in CONSTANT TIME
                                                       5. only then parse and act; also de-duplicate by
                                                          event id, because providers redeliver (Chapter 25)
```

(The header names and formats differ by provider; Stripe and GitHub, for example, document their own. The structure is the same.)

### Mechanism 3: constant-time comparison

A normal string comparison stops at the first differing byte, so comparing a wrong tag that matches the expected tag in its first 10 bytes takes slightly longer than comparing one that differs in the first byte. Over many requests, an attacker who can measure those small timing differences can discover a valid tag one byte at a time: a **timing side channel**. MAC verification must use a comparison whose time does not depend on where the inputs differ.

> **Java lens:** `Mac.getInstance("HmacSHA256")` with a `SecretKeySpec` computes HMACs; compare tags with `MessageDigest.isEqual(a, b)`, which runs in constant time, never with `Arrays.equals` or by comparing hex strings with `equals`.
>
> **Other runtimes:** Python: `hmac.compare_digest`. Node.js: `crypto.timingSafeEqual`. Go: `hmac.Equal`.

### Where HMACs appear

- **Webhooks**, as above.
- **Signed cookies and signed URLs**: the server appends HMAC(K, value) to a value it gives the client, and rejects anything returned with a tag that does not verify. The client can read the value (a MAC is not encryption) but cannot change it.
- **JWT with HS256**: the token's signature is HMAC-SHA256 over its header and payload. Everyone who can verify such a token also holds the key and can **create** tokens, which is why tokens that many services must verify use public-key signatures instead (RS256, ES256; Section 8.6, Chapter 20).
- **Request signing** for APIs, where each request carries an HMAC over its method, path, timestamp and body (several cloud providers sign API requests this way).

### Guarantees and non-guarantees

A MAC guarantees integrity and authenticity **among the holders of the key**. It does not provide confidentiality, does not prevent replay unless the tagged data includes a timestamp or unique identifier, and does not provide non-repudiation: since both sides hold the key, either could have produced the tag, so a third party cannot tell which.

### Failure and diagnosis

- **Webhook signatures that never verify.** The handler computes the HMAC over the *parsed and re-serialised* JSON instead of the raw bytes received. Re-serialisation changes whitespace, key order or number formatting, so the bytes, and the tag, differ. Fix: verify against the exact raw body before any parsing.
- **Verification with a plain comparison**, or verification that is skipped when the signature header is missing. Both turn an authentication check into a suggestion. Fix: constant-time comparison; reject missing signatures.
- **A leaked HS256 key.** If the key used for HS256 tokens is in every service's configuration, any compromised service can mint tokens for any user. Fix: public-key signatures for tokens verified by many parties.

### Backend connection

Use HMAC-SHA256 whenever two parties you control (or a provider and you) share a secret and need to authenticate messages: verify on the raw bytes, in constant time, with a timestamp or identifier against replay. When the verifiers are many or untrusted, use signatures.

---

## 8.6 Asymmetric Cryptography and Digital Signatures

### Definitions

**Asymmetric** (public-key) cryptography uses a **key pair**: a **private key** that its owner keeps secret, and a mathematically related **public key** that can be given to anyone. Deriving the private key from the public key is infeasible.

Two different operations use key pairs:

- **Public-key encryption**: anyone encrypts with the public key; only the private-key holder can decrypt.
- **Digital signature**: only the private-key holder can **sign** a message; anyone with the public key can **verify** the signature.

**RSA** and **elliptic-curve cryptography** (ECC) are the two families in use. Common signature schemes are **RSA-PSS** and the older **RSA PKCS#1 v1.5** (RSA family), **ECDSA** and **Ed25519** (elliptic-curve family).

### Why it exists

Symmetric cryptography requires both sides to share a key first. On the internet, a browser meets a server it has never talked to, and there is no secure channel through which they could have shared one. Public keys can be published openly; what must be secret never travels. Signatures add something MACs cannot: verification by parties who cannot create tags themselves. A million browsers can verify a server's signature without any of them being able to forge it, and one authentication service can issue tokens that fifty other services verify without being able to issue their own.

### Mechanism 1: the intuition behind RSA and ECC

Public-key schemes rest on a **trapdoor**: a computation that is easy in one direction and infeasible to reverse without a secret.

- **RSA**: multiplying two large primes is easy; factoring their product back into the primes is, for numbers of 2,048 bits and more, infeasible with known methods. The public key contains the product n and an exponent e; the private key is an exponent d that can only be computed by someone who knows the primes. Encryption is "raise to the power e, modulo n"; decryption is "raise to the power d, modulo n", which undoes it.

  A toy example with tiny numbers (insecure, for intuition only): primes 61 and 53 give n = 3233; take e = 17, for which the matching private exponent is d = 2753. Encrypting the message 65 gives 65^17 mod 3233 = 2790, and decrypting gives 2790^2753 mod 3233 = 65. Real RSA uses primes hundreds of digits long, and a **padding** scheme (OAEP for encryption, PSS for signatures) that adds randomness and structure; "textbook" RSA without padding is insecure.

- **ECC**: the public key is a point on an elliptic curve obtained by "multiplying" a fixed base point by the private number; computing the point is fast, recovering the number from the point is infeasible. ECC reaches the same security as RSA with much smaller keys: a 256-bit curve key (P-256, or Curve25519 used by X25519 and Ed25519) gives roughly the security of a 3,072-bit RSA key, with faster operations and smaller signatures. New systems usually prefer ECC.

Public-key operations are far slower than symmetric encryption, and RSA encryption can only handle a message smaller than its key. That answers a classic question, "why doesn't TLS just encrypt all traffic with RSA?": it would be slow and awkward, and (Section 8.7) it would lose forward secrecy. Public-key cryptography is used briefly, to agree on and authenticate symmetric keys, and AEAD does the bulk work.

(Large quantum computers, if built, would break both RSA and ECC. Post-quantum key-exchange schemes such as **ML-KEM** are being deployed alongside classical ones in TLS, often in "hybrid" combinations; certificates and signatures are following more slowly.)

### Mechanism 2: signing and verifying

```text
 SIGN (private key holder)                    VERIFY (anyone with the public key)
 message m                                    message m', signature s
   │                                            │
   ▼ hash                                       ▼ hash
 h = SHA-256(m)                               h' = SHA-256(m')
   │                                            │
   ▼ private-key operation                      ▼ public-key check
 s = Sign(private key, h)        ─────────►   Verify(public key, h', s) → valid / invalid
```

The signature covers the message's digest, which is why a secure hash is part of every signature scheme. Any change to the message changes the digest, and the signature no longer verifies.

Encryption and signing are opposite uses of a key pair. With encryption, the *recipient's* public key locks and their private key unlocks: it is about who can read. With signing, the *author's* private key signs and their public key checks: it is about who wrote. Using one where the other is needed is a common conceptual error; signing does not hide the message.

### Concrete: where signatures appear

- **Certificates.** A certificate authority signs each certificate it issues (Section 8.8).
- **The TLS handshake.** The server signs the handshake transcript with its certificate's private key, proving it holds that key right now (Section 8.9).
- **Tokens.** A JWT signed with RS256 (RSA PKCS#1 v1.5 with SHA-256) or ES256 (ECDSA P-256 with SHA-256) can be verified by any service holding the issuer's public key, which is commonly published at a well-known URL as a JSON Web Key Set (Chapter 20). Only the issuer can create one.
- **Software supply chain.** Packages, container images and commits are signed so that installers can verify their origin.

### Guarantees and non-guarantees

A valid signature guarantees that the holder of the private key signed exactly this message, which also gives non-repudiation to the extent that only the owner could use the key. It does not guarantee *who* owns the key: a public key is just numbers, and binding it to a name ("this key belongs to api.example.com") is the job of certificates (Section 8.8). It does not hide the message, and it does not prevent replay of a signed message unless the message contains an expiry or a unique identifier.

### Failure and diagnosis

- **Accepting the wrong algorithm.** A token-verification library trusts the algorithm named in the token itself. An attacker sends a token whose header says `"alg": "none"`, or says HS256 and is "signed" with HMAC using the server's *public* key as the secret, and a careless verifier accepts it. Fix: the verifier must fix the expected algorithm and key in configuration, never take them from the token (Chapter 20).
- **Weak randomness in ECDSA.** ECDSA needs a fresh secret random value for every signature; reusing one, or generating it predictably, reveals the private key from two signatures. This has caused real key compromises. Fix: use maintained libraries (which generate this value deterministically from the key and message, per RFC 6979, or use Ed25519, which is deterministic by design).

### Backend connection

Use signatures when verifiers are many, external or less trusted than the signer: public APIs' tokens, webhooks verified by third parties, anything installed by others. Keep private keys in a KMS or hardware module where possible, and plan for rotation by publishing several public keys at once with key identifiers.

---

## 8.7 Key Exchange and Forward Secrecy

### Definitions

A **key exchange** (key agreement) is a protocol by which two parties derive the same secret value over a channel that an attacker can read, without ever sending the secret.

**Diffie–Hellman** (DH) is the classic key exchange. **ECDHE** (elliptic-curve Diffie–Hellman, ephemeral) is its modern form, using elliptic curves and fresh keys for every exchange; **X25519** is the most common curve for it.

**Forward secrecy** (often called perfect forward secrecy) is the property that compromise of a party's long-term private key does not reveal the keys, and so the contents, of past sessions.

### Why it exists

A client and a server that have never met must end up with a shared symmetric key for AEAD (Section 8.3), while an attacker records everything they send. Encrypting a chosen key with the server's public RSA key (the "RSA key transport" of TLS 1.2 and earlier) works, but has a dangerous property: anyone who later obtains the server's private key (stolen from a backup, demanded by someone, extracted from a decommissioned disk) can decrypt every recorded session from the past years. Diffie–Hellman with fresh keys removes that risk.

### Mechanism 1: Diffie–Hellman with small numbers

Both sides agree publicly on a prime p and a base g. Each picks a private number, sends a public value derived from it, and combines its own private number with the other's public value.

```text
 public parameters:  p = 23, g = 5

 ALICE                                         BOB
 private a = 6                                 private b = 15
 A = g^a mod p = 5^6 mod 23 = 8                B = g^b mod p = 5^15 mod 23 = 19
              ───────── A = 8 ─────────►
              ◄──────── B = 19 ─────────
 secret = B^a mod p = 19^6 mod 23 = 2          secret = A^b mod p = 8^15 mod 23 = 2

 the attacker sees p = 23, g = 5, A = 8, B = 19, but not a, b or the secret 2
```

Both sides compute g^(a·b) mod p = 2, because (g^b)^a = (g^a)^b. An eavesdropper who sees A and B must recover a or b to get there, which is the **discrete logarithm problem**: easy with p = 23 (try every exponent), infeasible with the sizes used in practice. The shared value is then fed into a **key derivation function** (HKDF in TLS 1.3) to produce the actual encryption keys.

**ECDHE in words.** The same idea on an elliptic curve: each side picks a random private number, multiplies the curve's base point by it to get a public point, exchanges points, and multiplies the other's point by its own number. Both arrive at the same point, the shared secret. With X25519, each public key is 32 bytes and the computation takes microseconds.

### Mechanism 2: the man in the middle, and how authentication stops it

Diffie–Hellman alone protects against a **passive** attacker only. An active attacker can sit between the parties and run two exchanges:

```text
 Alice ── A ──► Mallory (replaces A with M1) ── M1 ──► Bob
 Alice ◄── M2 ── Mallory (replaces B with M2) ◄── B ── Bob
 Alice shares a secret with Mallory; Bob shares another secret with Mallory;
 Mallory decrypts, reads and re-encrypts everything in between
```

The fix is to **authenticate** the exchange. In TLS, the server signs the handshake, including its Diffie–Hellman public value, with the private key of its certificate (Section 8.9). Mallory can substitute values but cannot produce that signature, so the client detects the substitution. This is why TLS needs both key exchange (to create a secret) and signatures plus certificates (to know with whom).

### Mechanism 3: forward secrecy

With **ephemeral** Diffie–Hellman, each side generates its private number for one handshake and **deletes it** afterwards. The server's long-term certificate key is used only to *sign*, never to encrypt keys. Consequently:

```text
 2026: attacker records an encrypted session          (ECDHE keys used, then deleted)
 2028: attacker steals the server's certificate private key
       → can now impersonate the server in NEW sessions (until the certificate is revoked)
       → cannot decrypt the 2026 recording: the ephemeral keys that produced its secret no
         longer exist anywhere, and the certificate key never protected them
```

With RSA key transport, the 2028 theft would decrypt the 2026 recording. TLS 1.3 removed RSA key transport entirely; its full handshakes always use (EC)DHE, so they always provide forward secrecy.

### Guarantees and non-guarantees

An authenticated ECDHE exchange guarantees that the two endpoints, and only they, share a fresh secret, and that compromise of long-term keys later does not expose this session. It does not protect a session whose endpoint is compromised *while* it runs (the keys are in its memory), does not authenticate anyone by itself, and forward secrecy is weaker for session-resumption keys kept for a while on the server (Section 8.9).

### Failure and diagnosis

**Legacy configurations without forward secrecy.** A server, or a TLS-inspecting middlebox, still allows TLS 1.2 cipher suites with RSA key transport (names starting `TLS_RSA_WITH_...`). Evidence: a scan with `openssl s_client` (Section 8.13) or a TLS scanner shows them accepted. Fix: allow only (EC)DHE suites in TLS 1.2, and prefer TLS 1.3.

### Backend connection

Forward secrecy is why stealing a server's key is bad but not catastrophic for past traffic, and why "record everything now, decrypt later" does not work against modern TLS without breaking the key exchange itself. Keep it by allowing only TLS 1.3, or TLS 1.2 with ECDHE, everywhere you terminate TLS, including internal hops.

---

## 8.8 Certificates and Public Key Infrastructure

### Definitions

A **certificate** (X.509) is a signed document that binds a **public key** to an **identity**, such as the domain name `api.example.com`, for a limited period. The signature is made by a **certificate authority** (CA), an organisation trusted to check identities before signing.

**PKI** (public key infrastructure) is the whole system: CAs, the certificates they issue, the **trust stores** that list which CAs a client trusts, and the mechanisms for revoking and auditing certificates.

A **chain of trust** is the sequence leaf certificate → one or more **intermediate** CA certificates → a **root** CA certificate, in which each certificate is signed by the next one's key, and the root is in the client's trust store.

### Why it exists

Section 8.7 ended with the server signing its handshake. The client can check the signature with the server's public key, but how does it know that this public key belongs to `api.example.com` and not to an attacker? It cannot meet every server in person. Instead it trusts a small set of CAs, and each server presents a certificate in which a trusted CA states, with its signature, "this key belongs to `api.example.com`". Trust in a few root keys, shipped with the operating system or browser, scales to every server on the internet.

### Mechanism 1: what is in a certificate

```text
$ openssl x509 -in leaf.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 04:a1:9c:...                       unique per issuing CA
        Signature Algorithm: ecdsa-with-SHA256            how the CA signed it
        Issuer: C = US, O = Example Trust, CN = Example Issuing CA R1
        Validity
            Not Before: Sep  1 00:00:00 2026 GMT
            Not After : Nov 30 23:59:59 2026 GMT          valid for about 90 days
        Subject: CN = api.example.com
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey          the key being certified
                Public-Key: (256 bit)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature                         may sign (handshakes)
            X509v3 Extended Key Usage:
                TLS Web Server Authentication             may be used as a TLS server
            X509v3 Basic Constraints: critical
                CA:FALSE                                  may NOT sign other certificates
            X509v3 Subject Alternative Name:
                DNS:api.example.com, DNS:www.example.com  the names it is valid for
            Authority Information Access:
                CA Issuers - URI:http://ca.example-trust.test/r1.der   where to fetch the issuer
            CT Precertificate SCTs: ...                   proof of public logging (Mechanism 4)
    Signature Algorithm: ecdsa-with-SHA256
    Signature Value: 30:45:02:21:...                      the CA's signature over all of the above
```

The fields that matter most: **Subject Alternative Name** (SAN), which lists the names the certificate is valid for (modern clients check only the SAN, not the older Common Name); **validity**; **issuer**; the **public key**; the usage constraints; and the CA's **signature** over everything.

### Mechanism 2: the chain, and why intermediates exist

```text
 ┌───────────────────────────────┐
 │ ROOT CA certificate           │   self-signed; ships in the trust store of the OS,
 │ "Example Root CA"             │   browser or language runtime; private key kept offline
 └──────────────┬────────────────┘
                │ signs
 ┌──────────────▼────────────────┐
 │ INTERMEDIATE CA certificate   │   CA:TRUE; its key signs leaf certificates every day
 │ "Example Issuing CA R1"       │   sent by the SERVER, together with the leaf
 └──────────────┬────────────────┘
                │ signs
 ┌──────────────▼────────────────┐
 │ LEAF certificate              │   CA:FALSE; SAN: api.example.com
 │ "api.example.com"             │   the server holds its private key
 └───────────────────────────────┘
```

Roots are long-lived and hard to replace: every device in the world must already trust them. So their private keys are kept offline in hardware and used rarely, to sign intermediates. Intermediates do the daily signing; if one is compromised, it can be revoked and replaced without touching the root stores of billions of devices. The consequence for operators: **the server must send the leaf and the intermediates**. Clients have the roots, not the intermediates.

### Mechanism 3: how a client validates a certificate

1. **Build a chain** from the leaf, through the intermediates the server sent (some clients can also fetch missing ones using the Authority Information Access URL; many cannot), to a root in the client's trust store.
2. **Verify each signature**: each certificate's signature must verify with the public key of the one above it.
3. **Check validity periods**: the current time must be within Not Before and Not After for every certificate in the chain.
4. **Check the name**: the hostname the client meant to reach must match a SAN entry. A wildcard `*.example.com` matches exactly one label (`api.example.com`, not `example.com` and not `a.b.example.com`).
5. **Check constraints and usages**: intermediates must be CAs (`CA:TRUE`), the leaf must allow TLS server authentication, path-length and name constraints must hold.
6. **Check revocation**, where the client does (Mechanism 4).
7. **Check transparency policy**, where the client enforces it: major browsers require proof that the certificate was publicly logged (Mechanism 4).

Then, during the handshake, the server proves it holds the leaf's private key by signing (Section 8.9). A certificate alone proves nothing, since certificates are public; possession of the private key is what authenticates.

### Mechanism 4: revocation, transparency, and automated issuance

**Revocation** is how a certificate is cancelled before it expires (because its key leaked, or it was issued wrongly). A **CRL** (certificate revocation list) is a signed list of revoked serial numbers that clients download. **OCSP** (Online Certificate Status Protocol) lets a client ask the CA about one certificate; with **OCSP stapling**, the server fetches a fresh signed OCSP answer and includes it in its handshake, sparing the client the extra lookup. In practice revocation checking is unreliable: many clients "soft-fail" (accept the certificate if the check cannot be completed), and browsers use their own compressed revocation lists. The industry's main answer is **short certificate lifetimes**: a stolen key is useful only until the certificate expires, and lifetimes allowed by industry rules are being shortened step by step, toward periods of weeks rather than months.

**Certificate Transparency** (CT) addresses a different risk: a CA, through error or compromise, issuing a certificate for your domain to someone else. CAs submit every certificate to public, append-only **CT logs**, which return a signed timestamp (**SCT**) that is embedded in the certificate. Browsers require SCTs, so an unlogged certificate is rejected, and domain owners can monitor the logs for any certificate issued for their names.

**ACME** (RFC 8555), the protocol used by Let's Encrypt and other CAs, automates issuance. The client proves it controls the domain with a **challenge**:

- **HTTP-01**: serve a token at `http://<domain>/.well-known/acme-challenge/<token>` on port 80.
- **DNS-01**: publish a token in a `TXT` record at `_acme-challenge.<domain>` (Chapter 6, Section 6.6); this is the only method that can obtain wildcard certificates, and it works for hosts not reachable from the internet.
- **TLS-ALPN-01**: answer a special TLS handshake on port 443.

The CA checks the token from the internet and issues the certificate; the client renews automatically before expiry.

### Guarantees and non-guarantees

A validated chain guarantees that a CA trusted by the client stated that this public key belongs to this name, and the handshake signature proves the server holds the matching private key. It does not guarantee that the organisation behind the name is honest (anyone can obtain a valid certificate for `paypa1-login.example`), that the certificate has not been revoked (if the client does not check), or that every CA in the trust store is careful (CT exists to detect the ones that are not).

### Failure and diagnosis

- **Missing intermediate.** The server sends only its leaf. Browsers often succeed anyway, because they have cached the intermediate or fetch it automatically; `curl`, Java, Python and Go clients fail with "unable to get local issuer certificate" or "PKIX path building failed". Evidence: `openssl s_client -showcerts` (Section 8.13) lists only one certificate. Fix: configure the server with the full chain file (leaf followed by intermediates).
- **Hostname mismatch.** A client connects to `api.example.com` but the certificate's SAN lists only `www.example.com`, or a client connects by IP address. Evidence: the SAN list in `openssl x509 -ext subjectAltName`. Fix: a certificate with the right names, and clients that connect by name.
- **Private or corporate CA not trusted.** Internal services use certificates from an internal CA, or a corporate proxy inspects TLS with its own CA; a container image or a JVM does not have that CA in its trust store. Fix: add the CA to the relevant trust store (and only that CA); never disable verification.

### Backend connection

Automate issuance and renewal (ACME or your cloud's certificate service), monitor expiry dates of every certificate including internal ones, serve full chains, and know which trust store each of your clients uses: the operating system's, the browser's, the JVM's own `cacerts`, or a language runtime's bundled list. Different stores are why "it works in the browser" proves little.

---

## 8.9 The TLS 1.3 Handshake

### Definitions

**TLS** (Transport Layer Security) is a protocol that provides an encrypted, integrity-protected, authenticated channel between two endpoints over a reliable transport. TLS 1.3 is specified in **RFC 8446**. It has two parts: a **handshake**, which negotiates parameters, authenticates the server (and optionally the client) and derives keys; and a **record layer**, which carries application data in encrypted records.

**SNI** (Server Name Indication) is a field in the client's first handshake message that names the host the client wants, so that one IP address can serve many certificates.

**ALPN** (Application-Layer Protocol Negotiation) is a handshake extension by which client and server agree on the protocol that will run inside TLS, such as `h2` (HTTP/2) or `http/1.1`.

### Why it exists

TLS assembles the earlier primitives into a protocol: ECDHE for a fresh shared secret (Section 8.7), certificates and signatures to authenticate the server (Sections 8.6 and 8.8), a key derivation function to turn the secret into keys, and AEAD to protect every record (Section 8.3). Version 1.3 redesigned the handshake to take **one round trip** instead of two and removed every option that had led to attacks.

### Mechanism 1: the full TLS 1.3 handshake

```text
 client                                                         server
   │ ClientHello                                                  │
   │   random (32 bytes)                                          │
   │   supported versions: TLS 1.3                                │
   │   cipher suites: TLS_AES_128_GCM_SHA256, TLS_CHACHA20_...    │
   │   key_share: X25519 public key (client guesses the group)    │
   │   signature_algorithms, server_name (SNI): api.example.com   │
   │   ALPN: h2, http/1.1                                         │
   ├─────────────────────────────────────────────────────────────►│
   │                                                              │ picks suite and group,
   │                                                              │ computes the ECDHE secret
   │                                             ServerHello      │ → handshake keys
   │                                  random, key_share (X25519)  │
   │◄─────────────────────────────────────────────────────────────┤
   │ ═══ everything below is encrypted with handshake keys ═══    │
   │                                  {EncryptedExtensions}       │ ALPN result: h2
   │                                  {Certificate}               │ leaf + intermediates
   │                                  {CertificateVerify}         │ signature over the transcript
   │                                  {Finished}                  │ MAC over the transcript
   │◄─────────────────────────────────────────────────────────────┤
 client: computes the ECDHE secret, validates the chain (Section 8.8),
         checks the signature and the Finished MAC → application keys
   │ {Finished}                                                   │
   │ [application data: GET /v1/orders HTTP/2 ...] ──────────────►│
   │◄──────────────────────────────── [application data: response]│
```

Step by step:

1. **ClientHello.** The client proposes versions, AEAD cipher suites, signature algorithms, and the groups it supports, and **sends a key share right away** for the group it guesses the server will pick. It names the host (SNI) and the application protocols it can speak (ALPN).
2. **ServerHello.** The server chooses, and returns its own key share. Now both sides can compute the ECDHE shared secret, and from it, through the key schedule (a chain of HKDF steps), **handshake traffic keys**.
3. **Encrypted server messages.** From here on, everything is encrypted. **EncryptedExtensions** carries the rest of the server's choices, including the ALPN result. **Certificate** carries the chain. **CertificateVerify** is a signature, made with the leaf certificate's private key, over a hash of every handshake message so far (the **transcript**): this proves possession of the key and binds it to this handshake's key shares, which defeats the man in the middle of Section 8.7. **Finished** is a MAC over the transcript with a key derived from the secret: it proves the server derived the same keys and that nobody altered any handshake message, which also prevents downgrade attacks (Section 8.1).
4. **Client checks and Finished.** The client validates the certificate chain and hostname, verifies the signature and the Finished MAC, and sends its own Finished. It can send application data immediately after, in the same flight.

Total: **one round trip** from ClientHello to the first request. If the client guessed the wrong group, the server answers with a **HelloRetryRequest** naming the group it wants, costing an extra round trip.

**What TLS 1.3 removed**: RSA key transport (no forward secrecy), static Diffie–Hellman, CBC-mode and other non-AEAD ciphers, compression (which leaked secrets through sizes), renegotiation, and many other options. The three main cipher suites are `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384` and `TLS_CHACHA20_POLY1305_SHA256`.

### Mechanism 2: compared with TLS 1.2

```text
 TLS 1.2 full handshake: two round trips before application data
 client                                        server
   │ ClientHello ─────────────────────────────►│
   │◄──────── ServerHello, Certificate,         │   round trip 1
   │          ServerKeyExchange, ServerHelloDone│
   │ ClientKeyExchange, ChangeCipherSpec,       │
   │ Finished ─────────────────────────────────►│   round trip 2
   │◄──────── ChangeCipherSpec, Finished        │
   │ application data ─────────────────────────►│
```

In TLS 1.2 the client waits to learn the server's choices before sending its key exchange; and the certificate travels unencrypted. TLS 1.3 saves a round trip by sending a key share speculatively in the first message, and encrypts everything after ServerHello. At 50 ms RTT, that is 50 ms saved on every new connection (Chapter 6, Section 6.12).

### Mechanism 3: the record layer

After the handshake, application bytes are cut into **records** of at most 16 KB of plaintext, and each record is protected with AEAD. Each record's nonce is the per-connection IV combined with a **64-bit sequence number** that counts records, so a nonce never repeats, and because the sequence number is part of the authentication, a record that is **replayed, dropped or reordered** fails to decrypt (Section 8.1). In TLS 1.3 even the record's real content type is inside the encryption.

> **Java lens:** Java's TLS implementation (JSSE) is what `HttpClient`, JDBC drivers and most frameworks use. Setting the system property `javax.net.debug=ssl:handshake` prints the handshake messages, which is useful when a negotiation fails. Trust decisions use the JVM's trust store (the `cacerts` file, or one given with `javax.net.ssl.trustStore`), which is separate from the operating system's.

### Mechanism 4: resumption and 0-RTT

After a handshake, the server can send a **NewSessionTicket**: an encrypted blob from which it can recover a **pre-shared key** (PSK) derived from this session. On a later connection, the client offers the ticket in its ClientHello, and the server can skip certificate verification. Resumption normally still performs a fresh ECDHE exchange alongside the PSK, keeping forward secrecy for the new session.

```text
 FIRST CONNECTION (full handshake, as in Mechanism 1)
 client                                                         server
   │ ... full handshake ...                                       │
   │◄──────────────── {NewSessionTicket: ticket T, ~PSK} ─────────┤ after the handshake
   │ client stores T and the PSK it derives from this session     │

 LATER CONNECTION: resumption with 0-RTT early data
 client                                                         server
   │ ClientHello                                                  │
   │   pre_shared_key: ticket T                                   │
   │   key_share: new X25519 public key                           │
   │   early_data                                                 │
   │ (early data, encrypted with a key from the PSK):             │
   │   GET /v1/catalog ... ──────────────────────────────────────►│ decrypts T → PSK; may accept
   │                                                              │ early data → handles GET at once
   │                         ServerHello (pre_shared_key, key_share)
   │                         {EncryptedExtensions: early_data accepted}
   │                         {Finished}                           │ no Certificate, no
   │◄─────────────────────────────────────────────────────────────┤ CertificateVerify needed
   │ {EndOfEarlyData} {Finished}                                  │
   │ [application data, now with keys that also mix in the new ECDHE secret]
   │◄──────────────────────── [response to GET /v1/catalog] ──────┤

 the request travelled in the client's FIRST packet: zero round trips before it was sent.
 an attacker who copies that first flight can send it again (replay): see below.
```

**0-RTT ("early data").** A resuming client may also send application data in its **first flight**, encrypted with a key derived from the PSK, before the handshake completes. That saves the round trip entirely, but early data has two weaknesses:

- **Replay.** An attacker who records the client's first flight can send it to the server again. The server cannot reliably tell the copy from the original (servers can limit this with single-use tickets or replay caches, which are hard to make perfect across many servers). A replayed `GET` is usually harmless; a replayed `POST /payments` is not.
- **Weaker forward secrecy**: early data is protected only by the PSK, not by a fresh ECDHE exchange.

So servers should accept early data only for requests that are **safe and idempotent** (Chapter 7, Section 7.4), and many disable it. HTTP has machinery for this: a proxy that forwards early data marks the request with `Early-Data: 1`, and an origin that does not want to risk it answers `425 Too Early` so the client retries after the handshake (RFC 8470).

### Mechanism 5: SNI and ALPN

- **SNI.** Many sites share one IP address on a CDN or load balancer. The server must choose which certificate to present before the client has said anything else, so the client names the host in the ClientHello. In standard TLS, the ClientHello is **not encrypted**, so the SNI reveals to the network which site the client is visiting. **ECH** (Encrypted Client Hello) encrypts it using a key published in DNS, where both sides support it.
- **ALPN.** The client lists the protocols it can speak (`h2`, `http/1.1`; for QUIC, `h3`), and the server picks one in EncryptedExtensions. This is how HTTP/2 is negotiated without an extra round trip (Chapter 7, Section 7.10). A server that cannot agree on any of the offered protocols fails the handshake.

### Guarantees and non-guarantees

A completed TLS 1.3 handshake guarantees that the client is talking to the holder of a private key for a certificate valid for the name it asked for (if it validated properly), that both sides share fresh keys known to no one else, that the handshake was not tampered with, and that subsequent records are confidential, integrity-protected and in order, with no replay or reordering, until the connection ends. It does not authenticate the client (unless mutual TLS is used, Section 8.10), does not protect 0-RTT data from replay, and does not hide the SNI (without ECH), IP addresses, timing or sizes.

### Failure and diagnosis

- **Handshake failures after a "hardening" change.** A server is restricted to TLS 1.3 only, and older clients (an old JDK, an embedded device, a legacy partner) fail with a protocol-version alert. Evidence: `openssl s_client -tls1_2` versus `-tls1_3` against the server (Section 8.13). Fix: keep TLS 1.2 with strong (ECDHE plus AEAD) suites until the clients are upgraded.
- **Slow handshakes.** New connections take several round trips. Causes: a HelloRetryRequest because the client's first key share was for a group the server does not prefer, a large certificate chain that does not fit in TCP's initial congestion window (Chapter 6, Section 6.12), or OCSP fetches by the client. Evidence: a capture, or `curl -w` showing a large `time_appconnect` (Chapter 7, Section 7.16). Fixes: align key-share groups, trim the chain, staple OCSP, reuse connections and session tickets.

### Backend connection

Every HTTPS request your service makes or receives runs this handshake once per connection, which is one more reason to reuse connections (Chapter 6, Section 6.11). Enable TLS 1.3, keep 0-RTT off unless you have made the affected endpoints replay-safe, and remember that SNI is what a load balancer uses to choose a certificate and often a backend.

---

## 8.10 Where TLS Terminates, and Mutual TLS

### Definitions

**TLS termination** is the point where a TLS connection ends: the component that holds the certificate's private key, completes the handshake and decrypts the traffic. Behind it, traffic is plaintext unless a new TLS connection begins.

- **Termination at the edge** (at a load balancer or reverse proxy): the balancer decrypts, and forwards plain HTTP to the application over the internal network.
- **Re-encryption** (also called TLS bridging): the balancer decrypts, inspects and routes, then opens a **new** TLS connection to the application.
- **Passthrough**: an L4 balancer forwards the encrypted bytes untouched (Chapter 6, Section 6.13), and the application terminates TLS itself; the connection is end-to-end.

**Mutual TLS** (mTLS) is TLS in which the **client also presents a certificate** and proves possession of its private key, so that both sides authenticate each other.

### Why it exists

Terminating TLS requires CPU, certificate management and private keys. Concentrating it at the edge means few places hold keys, certificates are renewed in one place, and the balancer can read HTTP to route on paths and headers, add headers, and cache (Chapter 10). But traffic behind the edge is then readable by anything on the internal network, and internal networks are not as trusted as diagrams suggest: shared clouds, compromised machines and misconfigured peering exist. Many organisations, and many compliance regimes, therefore require encryption on internal hops as well, and increasingly authentication of *which service* is calling. mTLS provides that authentication at the transport layer.

### Mechanism 1: the topologies

```text
 1. EDGE TERMINATION
    client ══TLS══► [ load balancer: certificate + key ] ──plain HTTP──► app
                    decrypts; can route on HTTP           internal hop readable on the network

 2. RE-ENCRYPTION
    client ══TLS══► [ load balancer ] ══new TLS══► app (internal certificate)
                    decrypts, routes, re-encrypts      both hops encrypted; LB still sees plaintext

 3. PASSTHROUGH (L4)
    client ══════════════ TLS end to end ═════════════► app (public certificate + key)
                    [ L4 balancer forwards bytes ]       no HTTP-level routing at the balancer

 4. MUTUAL TLS between services
    service A ══TLS, both sides present certificates══► service B
    (client cert: "spiffe://prod/orders")               B authorises A by its certificate identity
```

| | Edge termination | Re-encryption | Passthrough | mTLS |
|---|---|---|---|---|
| Internal hop encrypted | no | yes | yes | yes |
| Balancer can read and route HTTP | yes | yes | no | depends where it terminates |
| Private keys held by | balancer | balancer and application | application | every service |
| Caller identity proven at transport level | no | no | no | **yes, both directions** |

**How the application learns about the client.** After termination, the application's TCP peer is the balancer, not the client, and the application did not see the handshake. The balancer passes the original information in headers: the client address in `X-Forwarded-For`, the scheme in `X-Forwarded-Proto` (Chapter 10), and, for mTLS terminated at the balancer, details of the client certificate. The application must trust these headers only from its own balancer (Chapter 7, Section 7.6).

### Mechanism 2: mutual TLS

In an mTLS handshake, the server adds a **CertificateRequest** message (listing acceptable CAs), and the client responds with its own **Certificate** and **CertificateVerify** before its Finished. The server validates the client's chain against its own trust store of client-issuing CAs, exactly as in Section 8.8, and then knows the client's identity, typically from a name in the SAN.

Inside organisations, mTLS is commonly run by a **service mesh** or a workload-identity system: a proxy next to each service (or the service's own library) obtains a **short-lived certificate** from an internal CA, often valid for hours, naming the workload (for example with a SPIFFE identity such as `spiffe://prod/orders`, a URI in the SAN). Certificates rotate automatically, so no human handles private keys. Authorisation then uses that identity: "only `orders` may call `payments` on `POST /charges`" (Chapter 20 covers service identity and authorisation).

### Guarantees and non-guarantees

Each TLS connection guarantees confidentiality and integrity **between its two endpoints**. With edge termination, that means client to balancer only; the internal hop has whatever protection the network gives it. Re-encryption protects both hops but still lets the balancer, which sees plaintext, read everything. mTLS guarantees that each side holds a private key for a certificate issued by a CA the other trusts; it does not say what the caller is *allowed* to do (that is authorisation), and it is only as strong as the internal CA and the protection of the workloads' keys.

### Failure and diagnosis

- **Redirect loops and wrong URLs after edge termination.** The application sees plain HTTP from the balancer, concludes the request was insecure and redirects to HTTPS forever, or builds `http://` links. Fix: honour `X-Forwarded-Proto` from the trusted balancer (Chapter 7, Section 7.11).
- **mTLS failures after a CA or certificate rotation.** Services start rejecting each other with "unknown CA" or "certificate required" alerts. Cause: the new client certificates come from a CA that the servers' trust stores do not include yet, or a client was not configured to present a certificate at all. Fix: roll out new trust (add the new CA everywhere) before new certificates, then remove the old CA after the old certificates expire.
- **Expired short-lived certificates.** A workload's certificate-renewal agent stops working, and the service fails all calls a few hours later. Evidence: handshake alerts naming an expired certificate; the agent's logs. Fix: alert on certificate age, not only on expiry.

### Backend connection

Choose the topology deliberately: edge termination for simplicity where the internal network is trusted and policy allows it; re-encryption or passthrough where internal traffic must be encrypted; mTLS where services must prove who they are to each other. Wherever TLS terminates, that component holds keys and sees plaintext, which makes it part of your security boundary.

---

## 8.11 HTTPS in Practice: HSTS, Mixed Content, Certificate Expiry

### Definitions

**HSTS** (HTTP Strict Transport Security, RFC 6797) is a response header, `Strict-Transport-Security`, by which a site tells browsers to use only HTTPS for it, for a stated period.

**Mixed content** is a page loaded over HTTPS that includes resources (scripts, stylesheets, images, API calls) loaded over plain HTTP.

A **certificate-expiry outage** is an outage caused by a certificate passing its Not After date without being replaced.

### Why they exist

A user who types `example.com` into a browser, or follows an old `http://` link, sends the first request over plain HTTP. The server's redirect to HTTPS (Chapter 7, Section 7.11) comes too late: an active attacker on the network can intercept that first plaintext request and keep the user on an HTTP connection that the attacker relays to the real HTTPS site (an attack known as **SSL stripping**). The user never reaches HTTPS at all. HSTS makes the browser refuse to use HTTP for the site in the first place.

Mixed content matters because one plain-HTTP script on an HTTPS page gives a network attacker control of the whole page. Certificate expiry matters because certificates expire by design (Section 8.8), and every one of them is a scheduled outage unless it is renewed.

### Mechanism 1: HSTS

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

- On receiving this header **over HTTPS**, the browser records that, for `max-age` seconds (here one year), it must turn every `http://` URL for this host into `https://` *before sending anything*, and must refuse to continue past a certificate error for it, without the usual option to click through the warning.
- `includeSubDomains` applies the rule to every subdomain.
- The first visit is still exposed, because the browser has not yet seen the header. The **preload list**, a list of HSTS hosts built into browsers, closes that gap; `preload` signals consent to be added, and sites submit themselves.

Because HSTS is cached by browsers, it is hard to undo: a site with `includeSubDomains` and a long `max-age` must be sure that every subdomain, including internal tools, serves valid HTTPS, or those subdomains become unreachable in browsers that saw the header. Roll it out with a short `max-age`, then raise it.

### Mechanism 2: mixed content

Browsers block **active** mixed content (scripts, stylesheets, frames, `fetch` and XHR calls to `http://`) on HTTPS pages, and upgrade or block **passive** mixed content such as images. The fixes are to load every resource over HTTPS (protocol-relative or `https://` URLs), and to let a Content Security Policy directive (`upgrade-insecure-requests`, Chapter 9) rewrite stragglers.

### Mechanism 3: certificate-expiry outages

Expiry outages have recurring shapes:

- A certificate renewed by a person, from a calendar reminder, that the person forgot or left the company.
- Automated renewal that silently broke (DNS credentials rotated, port 80 blocked so the HTTP-01 challenge fails, a renewal job that renews the file but never reloads the server process, which keeps serving the old certificate from memory).
- An **intermediate** or **root** that expired or was replaced, breaking older clients whose trust stores were not updated (public CAs have caused widespread failures this way when old roots expired).
- Internal certificates (databases, message brokers, mTLS, admin tools) that no one monitors because they are not on the public website.

### Guarantees and non-guarantees

HSTS guarantees that a browser that has seen it (or has the host preloaded) never contacts the site over HTTP, and that certificate errors for it cannot be clicked through. It does nothing for clients that are not browsers, and nothing on a browser's first contact without preload. Blocking mixed content guarantees that HTTPS pages do not run scripts fetched without integrity. Neither protects against a valid certificate for a look-alike domain.

### Failure and diagnosis

- **Expired certificate.** Symptom: all clients fail at once; browsers show a full-page warning; logs say `certificate has expired`. Evidence: `openssl s_client` (Section 8.13) or `openssl x509 -noout -enddate`. Fix: renew, deploy the full chain, **reload the process**, and verify from outside.
- **Renewed but still expired.** The file on disk is new, but the server still presents the old certificate, because the process never reloaded it. Evidence: compare the serial number on disk with the one presented on the wire. Fix: make reload part of renewal.
- **HSTS lockout.** After enabling `includeSubDomains`, an internal HTTP-only tool on a subdomain stops loading in browsers. Fix: serve it over HTTPS; browsers keep the old policy until it expires.

### Backend connection

Monitor certificate expiry from outside (what clients actually receive), for every endpoint including internal ones, with alerts well before expiry. Automate renewal end to end, including the reload. Turn on HSTS for user-facing sites once every subdomain is HTTPS-ready.

---

## 8.12 What HTTPS Guarantees, and What It Does Not

### Definition

HTTPS is HTTP carried over TLS. Its guarantee is precise: **data in transit between the two TLS endpoints** is confidential and integrity-protected, and the client has authenticated the server's name (and, with mTLS, the server has authenticated the client).

### Why the boundary matters

"We use HTTPS, so it is secure" is one of the most common overstatements in design reviews. HTTPS protects the pipe. Most real breaches happen at the ends of the pipe, or in what the pipe does not cover.

### Mechanism: what HTTPS does not protect

1. **Compromised endpoints.** Data is plaintext inside the client and the server. Malware on the user's device, a compromised server, a malicious browser extension or a leaked server-side log sees everything.
2. **Application bugs.** Injection, broken access control, insecure direct object references, cross-site scripting and server-side request forgery all travel happily over HTTPS (Chapters 9, 13, 18, 20).
3. **Hops after termination.** If TLS ends at a load balancer, the hop behind it is unprotected unless re-encrypted (Section 8.10), and every component that terminates TLS sees plaintext.
4. **Metadata.** The network still sees the IP addresses and ports (Chapter 6), the server name in the SNI unless ECH is used (Section 8.9), DNS lookups unless DNS is encrypted (Chapter 6, Section 6.6), and the timing and sizes of messages, from which an observer can sometimes infer which page or action was requested.
5. **The identity behind a name.** A valid certificate proves control of a domain name, not honesty. A phishing site at a look-alike name can have a perfectly valid certificate.
6. **Data at rest.** HTTPS says nothing about how the server stores what it receives; database encryption, backups and logs are separate problems (Section 8.3).
7. **Mistakes in validation.** A client that disables certificate verification ("trust all certificates", `curl -k`, `verify=False`), or that skips hostname checking, has encryption with an unknown party: an active attacker can impersonate the server. This is the most common way HTTPS is defeated in backend code.
8. **Replay of 0-RTT data** (Section 8.9).

### Guarantees and non-guarantees

| HTTPS guarantees | HTTPS does not guarantee |
|---|---|
| Confidentiality of data between the two TLS endpoints | Protection inside compromised endpoints, or after termination |
| Integrity: tampering, reordering and replay of records are detected | Correctness or security of the application's logic |
| Server authentication: the server holds a key certified for this name (if the client validates) | That the name belongs to someone trustworthy |
| Forward secrecy (with TLS 1.3 or ECDHE in TLS 1.2) | Hidden metadata: IPs, SNI (without ECH), sizes, timing |
| Client authentication, with mTLS | Authorisation: what the authenticated party may do |

### Failure and diagnosis

**Disabled verification left in production.** Code written for a test environment with self-signed certificates sets a "trust everything" option that ships to production. Nothing fails, which is the problem: the client would accept an attacker's certificate. Evidence: search the code and configuration for trust-all managers, `InsecureSkipVerify`, `verify=False`, `-k`, `NODE_TLS_REJECT_UNAUTHORIZED=0`. Fix: add the test CA to the test trust store instead, and fail builds that disable verification.

### Backend connection

When a design says "encrypted in transit", ask: between which two endpoints, who terminates, who holds the keys, which hops are left, and what protects the data at the endpoints and at rest.

---

## 8.13 Diagnosing TLS: `openssl s_client` and the Classic Failures

### Why a systematic approach

TLS errors arrive in many dialects: a browser warning page, a curl message, a Java stack trace, a load balancer's "handshake failure" counter. Underneath there are only a handful of failures: expired, incomplete chain, hostname mismatch, untrusted issuer, and protocol or cipher mismatch. `openssl s_client` connects as a client and prints what the server actually sends, which identifies the failure regardless of which client reported it.

### Concrete 1: an annotated `openssl s_client` session

The output format varies between OpenSSL versions; this is representative of OpenSSL 3.

```text
$ openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts </dev/null
CONNECTED(00000003)                                                    ① TCP connection made
depth=2 C = US, O = Example Trust, CN = Example Root CA                ② chain built up to a root
verify return:1                                                           in the local trust store
depth=1 C = US, O = Example Trust, CN = Example Issuing CA R1
verify return:1
depth=0 CN = api.example.com
verify return:1
---
Certificate chain                                                      ③ what the SERVER sent
 0 s:CN = api.example.com                                                 0 = leaf (s = subject,
   i:C = US, O = Example Trust, CN = Example Issuing CA R1                 i = issuer)
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: ecdsa-with-SHA256
   v:NotBefore: Sep  1 00:00:00 2026 GMT; NotAfter: Nov 30 23:59:59 2026 GMT  ④ validity
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
 1 s:C = US, O = Example Trust, CN = Example Issuing CA R1                1 = intermediate: the
   i:C = US, O = Example Trust, CN = Example Root CA                       chain is complete
...
---
Peer signature type: ECDSA                                             ⑤ CertificateVerify signature
Server Temp Key: X25519, 253 bits                                      ⑥ ECDHE group: forward secrecy
---
SSL handshake has read 2876 bytes and written 393 bytes
Verification: OK                                                       ⑦ chain validated
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384                        ⑧ version and AEAD suite
...
Verify return code: 0 (ok)
```

Reading it: ② shows the chain the client could build; ③ shows the certificates the server actually sent (if only `0` appears, the intermediate is missing); ④ the validity window; ⑥ the key exchange; ⑦ and the final return code the verdict; ⑧ the negotiated version and cipher.

Two cautions. Without `-servername`, older versions send no SNI, and a server hosting many names may answer with its default certificate, which then appears to be "wrong". And `s_client` does **not** check the hostname unless asked: add `-verify_hostname api.example.com` (and `-verify_return_error` to stop on any error).

Useful variants:

```bash
# expiry date of what the server presents
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# does the server accept TLS 1.2? TLS 1.3?
openssl s_client -connect api.example.com:443 -servername api.example.com -tls1_2 </dev/null
openssl s_client -connect api.example.com:443 -servername api.example.com -tls1_3 </dev/null

# which application protocol does ALPN choose?
openssl s_client -connect api.example.com:443 -servername api.example.com -alpn h2,http/1.1 </dev/null
```

### Concrete 2: the classic failures and their signatures

| Failure | `openssl s_client` | `curl` | Java | Typical cause and fix |
|---|---|---|---|---|
| Expired certificate | `verify error:num=10:certificate has expired` | `SSL certificate problem: certificate has expired` | `CertificateExpiredException` inside a "PKIX path validation failed" error | renewal failed, or the process was not reloaded; renew and reload |
| Missing intermediate | `num=20:unable to get local issuer certificate`, then `num=21:unable to verify the first certificate`; only certificate `0` in the chain | `unable to get local issuer certificate` | `PKIX path building failed ... unable to find valid certification path to requested target` | server configured with the leaf only; serve the full chain |
| Hostname mismatch | `num=62:hostname mismatch` (with `-verify_hostname`) | `no alternative certificate subject name matches target host name` | `No subject alternative DNS name matching api.example.com found` | wrong certificate, missing SAN, connecting by IP or an alias, or no SNI sent; fix the certificate or the name |
| Untrusted or self-signed issuer | `num=18:self-signed certificate` or `num=19:self-signed certificate in certificate chain` | `self-signed certificate in certificate chain` | `PKIX path building failed` | internal or corporate CA not in the client's trust store; add that CA to the right store |
| Protocol or cipher mismatch | handshake fails with an alert such as `protocol version` or `handshake failure` | `alert protocol version` or similar | `SSLHandshakeException: Received fatal alert: protocol_version` or `No appropriate protocol` | client and server share no version or suite; upgrade the old side, or temporarily allow TLS 1.2 with strong suites |

A browser that works where `curl` or Java fails is almost always the missing-intermediate case (the browser had the intermediate cached or fetched it) or a trust-store difference (the browser uses the operating system's store, the JVM uses its own).

### Guarantees and non-guarantees

`openssl s_client` shows exactly what the server sends to a client that connects from where you run it, with the SNI and options you choose. It does not use your application's trust store or settings: a test that succeeds with `openssl` can still fail in a JVM whose trust store lacks a CA, and the reverse. Reproduce with the same name, the same network path and, where possible, the same trust store as the failing client.

### Backend connection

Keep a TLS check in every runbook: `openssl s_client` with `-servername`, `-showcerts` and `-verify_hostname`, plus the expiry pipeline above. Most TLS incidents are diagnosed in the first minute by reading the chain and the verify code.

---

## Chapter Summary

Security on a hostile network means four goals: **confidentiality** (no one else reads it), **integrity** (changes are detected), **authenticity** (it comes from who it claims), and **non-repudiation** (the author cannot deny it), plus protection against **replay**. The attacker to design for is **active**: able to read, change, drop, inject and replay traffic.

The tools are distinct. **Encoding** (Base64) is public and reversible: no security at all, which is why a JWT's payload is readable by anyone. **Hashes** (SHA-256) give keyless, one-way fingerprints that resist preimages and collisions but prove nothing about who produced the data; **fast hashes are wrong for passwords**, which need slow, salted functions (Chapter 20). **MACs** (HMAC-SHA256) prove integrity and authenticity between holders of a shared key; they must be verified on raw bytes, in constant time. **Signatures** (Ed25519, ECDSA, RSA-PSS) are made with a private key and verified by anyone with the public key, giving non-repudiation. **Symmetric encryption** is fast and needs a shared key; it must be **authenticated** (AEAD: AES-GCM, ChaCha20-Poly1305), and a nonce must **never repeat** under one key. **ECB** leaks patterns; unauthenticated modes let attackers alter data. All of these need keys and nonces from a **CSPRNG**.

**Public-key cryptography** solves the problem of strangers. **Diffie–Hellman** lets two parties derive a shared secret from values exchanged in the open; using **ephemeral** keys (**ECDHE**) gives **forward secrecy**, so stealing a server's long-term key later does not decrypt recorded sessions. Unauthenticated Diffie–Hellman falls to a man in the middle, so the server **signs** the exchange. **Certificates** bind public keys to names: a **CA** signs them, a **chain** runs from the leaf through intermediates to a root in the client's **trust store**, and validation checks signatures, dates, the **SAN** against the hostname, constraints and (unreliably) revocation; **Certificate Transparency** makes misissuance visible, short lifetimes limit damage, and **ACME** automates issuance.

**TLS 1.3** assembles all of this in **one round trip**: the ClientHello carries a key share, SNI and ALPN; the ServerHello returns a key share; the server's certificate, its signature over the transcript (**CertificateVerify**) and a **Finished** MAC follow encrypted; then AEAD records with sequence-numbered nonces carry the data. **Resumption** reuses a pre-shared key, and **0-RTT** sends data immediately at the price of possible **replay**, so it suits only safe, idempotent requests. Where TLS **terminates** (at the edge, with re-encryption, passed through, or as **mTLS** between services) decides which hops are protected and who sees plaintext. **HSTS** stops browsers from ever using HTTP for a site, browsers block **mixed content**, and **certificate expiry** must be monitored and automated. Finally, HTTPS protects data **in transit between the TLS endpoints** and nothing else: not compromised endpoints, application bugs, metadata such as IPs and SNI, data at rest, or clients that disable verification.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Encoding (Base64) | Safe transport of bytes as text | Any confidentiality or integrity |
| CSPRNG | Unpredictable output | Strength of short values; secrecy of logged tokens |
| Hash (SHA-256) | Practical uniqueness of digests; one-wayness for unpredictable inputs | Origin of the data; hiding of guessable inputs; safe password storage |
| HMAC | Integrity and authenticity among key holders | Confidentiality; non-repudiation; replay protection without a timestamp or ID |
| Digital signature | Integrity, authenticity and non-repudiation, verifiable by anyone | Who owns the key (needs a certificate); confidentiality |
| AEAD | Confidentiality plus integrity of ciphertext and associated data, with a unique nonce | Hidden length; replay protection on its own; anything after nonce reuse or key leak |
| ECB mode | Nothing beyond block-by-block secrecy | Hiding of patterns; integrity |
| Ephemeral ECDHE | A fresh shared secret; forward secrecy | Authentication (needs signatures); protection while an endpoint is compromised |
| Certificate chain | A trusted CA bound this key to this name | Honesty of the name's owner; revocation status if unchecked |
| Certificate Transparency | Public visibility of issued certificates | Prevention of misissuance (it enables detection) |
| TLS 1.3 handshake | Authenticated server, fresh keys, untampered handshake, in one round trip | Client authentication (without mTLS); hidden SNI (without ECH) |
| TLS records | Confidentiality, integrity, order; no replay within a connection | Protection of 0-RTT data from replay |
| Edge termination | TLS from client to balancer; HTTP routing at the balancer | Protection of the internal hop |
| mTLS | Both peers hold keys certified by a trusted CA | Authorisation decisions; security of the internal CA |
| HSTS | Browsers never use HTTP for the site once known; no click-through of errors | Protection on the first visit without preload; protection for non-browser clients |
| HTTPS overall | Security of data in transit between TLS endpoints | Endpoint security, application security, metadata privacy, data at rest |

---

## Interview Drill

**Q1. What is the difference between encoding, hashing and encryption? Is Base64 encryption?**
A strong answer must mention:
- Encoding: keyless, reversible by anyone, for representation (Base64, hex, UTF-8); no security.
- Hashing: keyless, one-way, fixed-size digest; fingerprints, not secrecy of guessable inputs.
- Encryption: keyed and reversible only with the key; confidentiality.
- Base64 is encoding; a JWT's payload is Base64url and readable by anyone; its signature prevents changes, not reading.
*Common weak answer:* "Base64 encrypts the token." It has no key and anyone can decode it.

**Q2. What is the difference between symmetric and asymmetric cryptography? Why doesn't TLS encrypt everything with RSA?**
A strong answer must mention:
- Symmetric: one shared key, very fast (AES with hardware support), but the key must be shared first.
- Asymmetric: key pairs; public keys can be published; enables key agreement and signatures; much slower, and RSA can only encrypt small messages.
- TLS uses asymmetric operations only in the handshake (ECDHE plus a signature) to set up symmetric AEAD keys for the data.
- RSA key transport also lacked forward secrecy, which is why TLS 1.3 removed it.
*Common weak answer:* "asymmetric is more secure", which ignores speed, purpose and forward secrecy.

**Q3. Why is ECB insecure? What is AEAD, and why must a nonce never be reused?**
A strong answer must mention:
- ECB encrypts each block independently, so equal plaintext blocks give equal ciphertext blocks: patterns leak and blocks can be cut and pasted.
- Unauthenticated modes (CTR, CBC alone) let attackers alter plaintext by altering ciphertext; padding oracles can even reveal it.
- AEAD (AES-GCM, ChaCha20-Poly1305) encrypts and computes a tag over the ciphertext and associated data, and releases nothing if the tag fails.
- Reusing a key and nonce XORs away the keystream (P1 XOR P2 leaks) and, for GCM, enables forgeries; use counters, random nonces within limits, or larger-nonce constructions.
*Common weak answer:* "ECB is old." The answer needs the deterministic-block mechanism.

**Q4. What properties does a cryptographic hash have, and why is SHA-256 wrong for passwords?**
A strong answer must mention:
- Preimage, second-preimage and collision resistance; avalanche; collisions cost about 2^(n/2) by the birthday bound.
- MD5 and SHA-1 are broken for collision resistance.
- Passwords are guessable, and SHA-256 is fast (billions of guesses per second on GPUs), so a stolen table falls to dictionary attacks; salts stop precomputation but not guessing.
- Use slow, salted, memory-hard functions: Argon2id, scrypt, bcrypt (Chapter 20).
*Common weak answer:* "SHA-256 cannot be reversed, so it is safe." Attackers guess, they do not reverse.

**Q5. What is a MAC or HMAC, and how does it differ from a digital signature?**
A strong answer must mention:
- A MAC is a keyed tag proving integrity and authenticity to holders of the shared secret; HMAC builds one from a hash with two keyed passes, which defeats length extension.
- Verification on raw bytes, in constant time; timestamps or IDs against replay.
- Uses: webhooks, signed cookies, JWT HS256, request signing.
- A signature uses a private key to sign and a public key to verify: anyone can verify, only the owner can sign, giving non-repudiation; a MAC's verifiers can also forge.
*Common weak answer:* "HMAC is a hash with a password", missing who can verify versus who can create.

**Q6. How do two parties agree on a secret key over a network an attacker records?**
A strong answer must mention:
- Diffie–Hellman: public p and g; each sends g^secret mod p; each combines the other's value with its own secret to reach g^(ab) mod p.
- An eavesdropper sees only the public values and faces the discrete-logarithm problem; a worked example with small numbers.
- ECDHE does the same on elliptic curves (X25519), with fresh keys per handshake.
- Alone it falls to an active man in the middle; TLS authenticates it by having the server sign the handshake.
*Common weak answer:* "the client encrypts a key with the server's public key", which is the older method without forward secrecy.

**Q7. What is forward secrecy?**
A strong answer must mention:
- Compromise of a long-term private key does not expose past sessions' keys.
- Achieved with ephemeral key exchange: per-handshake keys deleted afterwards; the certificate key only signs.
- RSA key transport lacks it: recorded traffic can be decrypted once the key is stolen.
- TLS 1.3 full handshakes always provide it; 0-RTT data has weaker protection.
*Common weak answer:* "the keys change often", without the ephemeral mechanism or what is protected. Rotating certificates does not stop a stolen key from decrypting recorded sessions that used RSA key transport; only ephemeral key exchange does.

**Q8. What is in a certificate, why are there intermediate CAs, and how does a client validate a certificate?**
A strong answer must mention:
- Subject public key, SAN names, validity, issuer, usages and constraints, extensions, and the CA's signature over all of it.
- Roots live in trust stores and are kept offline; intermediates do daily signing and can be replaced; servers must send intermediates.
- Validation: build the chain to a trusted root, verify signatures, check dates, match the hostname against the SAN, check constraints and usages, check revocation and CT where enforced.
- The handshake signature proves the server holds the leaf's private key.
*Common weak answer:* "the browser checks the certificate is valid", with no chain, no name check and no proof of key possession. A certificate is public; without the name check and the handshake signature, anyone could present it.

**Q9. Walk through the TLS 1.3 handshake. How is it faster than TLS 1.2, and what is the risk of 0-RTT?**
A strong answer must mention:
- ClientHello with versions, suites, a key share, SNI and ALPN; ServerHello with the server's key share; both derive handshake keys.
- Encrypted EncryptedExtensions, Certificate, CertificateVerify (signature over the transcript), Finished (MAC over the transcript); client verifies and sends Finished, then data.
- One round trip, because the client sends its key share speculatively (TLS 1.2 needed two); HelloRetryRequest adds one if the guess was wrong.
- Resumption uses a PSK from a session ticket; 0-RTT early data can be replayed, so only safe, idempotent requests should use it (`425 Too Early`).
*Common weak answer:* "the server sends its certificate and they agree on a key", with no key share, no signature over the transcript, and no replay discussion. Those details are what make it one round trip and what stop a man in the middle.

**Q10. What are SNI and ALPN, and what breaks without them?**
A strong answer must mention:
- SNI names the host in the ClientHello so a shared IP can pick the right certificate (and often backend).
- Without SNI, the server presents a default certificate and the client reports a hostname mismatch.
- SNI is visible on the network unless ECH is used.
- ALPN negotiates the protocol inside TLS (`h2`, `http/1.1`) without extra round trips; a failed negotiation fails the handshake or falls back to HTTP/1.1.
*Common weak answer:* naming the acronyms without the shared-IP and protocol-negotiation problems they solve. Those problems are what explain the failures you will actually debug: default certificates, and unexpected fallback to HTTP/1.1.

**Q11. Compare terminating TLS at a load balancer, re-encryption, passthrough and mutual TLS.**
A strong answer must mention:
- Edge termination: the balancer decrypts and routes on HTTP; the internal hop is plaintext; few places hold keys.
- Re-encryption: a second TLS connection to the backend; both hops encrypted; the balancer still sees plaintext.
- Passthrough: L4 forwarding of encrypted bytes; end-to-end TLS; no HTTP routing at the balancer.
- mTLS: the client also presents a certificate, giving service identity at the transport layer; usually automated with short-lived certificates; authorisation is still separate.
*Common weak answer:* "terminate at the load balancer because it is faster", without the trade-off of plaintext hops. The internal hop is then unprotected, which may violate policy and exposes traffic to anything on that network.

**Q12. Why does an HTTP-to-HTTPS redirect still leave the first request exposed, and how does HSTS fix it?**
A strong answer must mention:
- The first request goes over plain HTTP; an active attacker can intercept it and keep the user on HTTP (SSL stripping).
- HSTS makes the browser rewrite `http://` to `https://` before sending, for `max-age`, and forbids clicking through certificate errors.
- The first visit is still exposed unless the domain is on the preload list.
- `includeSubDomains` requires every subdomain to support HTTPS; roll out with a short `max-age` first.
*Common weak answer:* "the redirect forces HTTPS", missing that the redirect itself travels over HTTP.

**Q13. Why does a browser accept a certificate that your backend client rejects with "unable to get local issuer certificate"?**
A strong answer must mention:
- The server probably sends only the leaf; browsers often have the intermediate cached or fetch it, while most backend clients do not.
- Alternatively the issuing CA is in the operating system's trust store but not in the JVM's or runtime's own store.
- Confirm with `openssl s_client -showcerts` (only one certificate in the chain) and by checking the client's trust store.
- Fix the server's chain or the client's trust store; never disable verification.
*Common weak answer:* "the backend's certificate library is buggy; turn off verification." Disabling verification removes server authentication entirely, turning a configuration problem into a security hole.

**Q14. What does HTTPS not protect against?**
A strong answer must mention:
- Compromised endpoints, and anything after TLS terminates.
- Application vulnerabilities (injection, broken access control, XSS, SSRF).
- Metadata: IPs, ports, SNI without ECH, DNS without encryption, sizes and timing.
- Phishing domains with valid certificates; data at rest; clients that disable verification; replay of 0-RTT data.
*Common weak answer:* "HTTPS makes the site secure." It protects only data in transit between the two TLS endpoints.

**Q15. How would you verify a webhook from a payment provider?**
A strong answer must mention:
- Compute HMAC-SHA256 over the exact raw body (and the signed timestamp) with the shared secret.
- Compare with the provided signature in constant time; reject missing or invalid signatures.
- Reject timestamps outside a small window to limit replay.
- De-duplicate by event ID, because providers deliver at least once (Chapter 25).
*Common weak answer:* "check that the request comes from the provider's IP address", which is weaker and brittle, and says nothing about integrity.

**Q16. How would you encrypt a sensitive column in a database?**
A strong answer must mention:
- An AEAD cipher (AES-256-GCM or ChaCha20-Poly1305) from a standard library.
- A fresh random nonce per value, stored with the ciphertext; the row's ID as associated data to prevent moving values between rows.
- Keys from a KMS with envelope encryption (per-object or per-tenant data keys wrapped by a master key), never in code.
- Plan for key rotation and for which queries become impossible (no indexing or searching on ciphertext without extra design).
*Common weak answer:* "AES with a key in the config file", with no mode, nonce, authentication or key management. The default mode may be ECB, and the key leaks with every copy of the configuration.

**Q17. Why do certificate-expiry outages keep happening, and how do you prevent them?**
A strong answer must mention:
- Certificates expire by design, and lifetimes are getting shorter.
- Manual renewal is forgotten; automated renewal fails silently (challenges blocked, credentials rotated, process not reloaded); internal certificates go unmonitored; intermediates and roots change.
- Monitor what clients actually receive, from outside, for every endpoint, with early alerts.
- Automate renewal and reload end to end, and test it.
*Common weak answer:* "set a calendar reminder." Manual steps are exactly what fails; renewal and monitoring must be automated and checked from outside.

---

## Scenario Exercises

1. A team stores users' third-party API tokens in a `credentials` column, "encrypted with AES" using a key in the application's configuration file and `Cipher.getInstance("AES")`. Review the design: what is wrong, what could an attacker with read access to the database do, and what would you change?
2. Thirty internal services verify access tokens signed with HS256 and a single shared secret stored in each service's configuration. One low-priority service is compromised. What can the attacker do, and how would you redesign token signing?
3. A company's compliance team requires "encryption in transit everywhere". Today, TLS terminates at the public load balancer and traffic to the services is plain HTTP inside the private network. Describe two designs that satisfy the requirement, and their trade-offs for routing, key management and service identity.
4. After a security team enables `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` on `example.com`, several internal tools at `*.corp.example.com` stop loading in browsers. Explain why, why it will not go away quickly, and what should have been done.
5. A server's TLS private key was found in a public code repository, where it had been for three months. What is exposed (past traffic, future traffic, impersonation), how does the answer depend on the TLS configuration, and what do you do now, in order?
6. A product manager says: "We use HTTPS everywhere, so customer data is secure." Write the five most important caveats you would raise, each with a concrete example from a typical backend.
7. An API team wants to enable TLS 1.3 0-RTT on its edge to shave 50 ms off mobile requests. The API includes `GET /catalog`, `POST /orders` and `POST /payments`. What do you advise, and what must the edge and the application do for it to be safe?

---

## Diagnosis Lab

**Lab 1: Every client fails at 00:00 on a Sunday**
- *Symptom:* At midnight UTC, all mobile apps and partner integrations fail to reach `api.example.com`. Browsers show a full-page certificate warning. The application servers are healthy and their own logs show no errors, because no requests reach them.
- *Hypotheses:* (a) a DNS change sent clients elsewhere; (b) the certificate expired; (c) the load balancer failed.
- *Evidence:* `dig api.example.com` returns the usual address, against (a). `openssl s_client -connect api.example.com:443 -servername api.example.com` reports `verify error:num=10:certificate has expired`, with Not After equal to that midnight. The load balancer's metrics show handshake failures rising and backend health checks passing. The renewal job's log shows it has failed for the past 30 days: the HTTP-01 challenge could not be reached since a new firewall rule closed port 80, and nobody watched the job's alerts.
- *Root cause:* Automated renewal broke silently a month earlier, and nothing monitored the certificate's actual expiry.
- *Fix:* Issue a certificate by a method that works now (DNS-01, or reopen port 80 for the challenge), install the full chain on the balancer, confirm with `openssl s_client` from outside, and restore port 80 for challenges or move permanently to DNS-01.
- *Prevention:* External monitoring of presented certificates with alerts 30, 14 and 7 days before expiry; alerts on renewal-job failure; a test that renewal works after any firewall change.

**Lab 2: Works in the browser, fails in the Java service**
- *Symptom:* After a partner rotated its certificate, your Java integration fails with `PKIX path building failed: ... unable to find valid certification path to requested target`. Opening the partner's API in a browser shows a valid padlock. `curl` from the same container fails with `unable to get local issuer certificate`.
- *Hypotheses:* (a) the partner's new certificate is from a CA missing from the JVM's trust store; (b) the partner's server is not sending its intermediate; (c) a proxy is intercepting TLS.
- *Evidence:* `openssl s_client -connect partner.example.net:443 -servername partner.example.net -showcerts` lists only certificate `0` (the leaf), issued by an intermediate that the server does not send, which supports (b). The intermediate's issuing root is present in the JVM's `cacerts`, against (a). The certificate presented has the partner's expected issuer, not a corporate proxy's, against (c). The browser succeeds because it fetched the intermediate using the Authority Information Access URL (or had it cached).
- *Root cause:* The partner deployed the leaf certificate without the intermediate chain.
- *Fix:* Ask the partner to serve the full chain. As a temporary measure, adding the intermediate to your trust store is possible, but it must be removed later and must never become "trust all certificates".
- *Prevention:* Synthetic checks against partners' endpoints with `openssl s_client -verify_return_error`, and an agreed notification process for certificate changes.

**Lab 3: Hostname mismatch after moving behind a new load balancer**
- *Symptom:* After `billing.example.com` moves to a shared load balancer that also serves 40 other domains, most clients work, but one old batch job fails with `No subject alternative DNS name matching billing.example.com found`.
- *Hypotheses:* (a) the certificate for `billing.example.com` is missing from the balancer; (b) the batch job does not send SNI, so the balancer presents its default certificate; (c) the job connects by a different name.
- *Evidence:* `openssl s_client -connect lb.example.com:443 -servername billing.example.com` shows the correct certificate, against (a). The same command without `-servername` (behaving like a client that sends no SNI) shows the balancer's default certificate for `www.example.com`. A capture of the batch job's ClientHello shows no `server_name` extension; the job uses a very old HTTP library. The job's configured URL is `https://billing.example.com`, against (c).
- *Root cause:* A client without SNI support could work against the old dedicated server (one certificate per address) but not against a shared balancer that selects certificates by SNI.
- *Fix:* Upgrade the job's HTTP library (any current TLS stack sends SNI). If that is impossible right away, give the old client a dedicated listener whose default certificate is `billing.example.com`.
- *Prevention:* Inventory legacy clients before consolidating TLS endpoints; monitor handshakes without SNI on shared balancers.

**Lab 4: Handshake failures after a TLS policy change**
- *Symptom:* After the edge's TLS policy is changed to "TLS 1.3 only", a partner's payment terminals fail every connection with a handshake error; everything else works.
- *Hypotheses:* (a) the terminals do not support TLS 1.3; (b) the terminals' trust store lacks the edge's CA; (c) a network change.
- *Evidence:* `openssl s_client -tls1_3` against the edge succeeds and `-tls1_2` fails with a protocol-version alert, confirming the policy. The partner's logs from a terminal show it offering TLS 1.2 only, with ECDHE and AES-GCM suites. The failures began within a minute of the policy change, and connectivity tests (TCP connect to port 443) succeed, against (c). The terminals never got far enough to validate a certificate, against (b).
- *Root cause:* The policy removed the only protocol version the terminals support.
- *Fix:* Restore TLS 1.2 limited to ECDHE with AEAD suites (forward secrecy and authenticated encryption), possibly on a separate endpoint for the partner, and agree a deadline for the terminals' upgrade.
- *Prevention:* Before tightening TLS policy, review the protocol versions and suites clients actually negotiate (edge logs record them), and roll out per endpoint.

---

## Solution Sketches

1. `Cipher.getInstance("AES")` with the default provider is AES in ECB mode with no authentication: equal tokens produce equal ciphertexts (revealing which users share a token or which tokens are identical), blocks can be moved between rows, and tampering goes undetected. The key in the configuration file means anyone who can read the configuration, a backup of it or the repository can decrypt everything. An attacker with database read access can compare ciphertexts and, if they ever obtain the configuration, decrypt all tokens. Change to: AES-256-GCM (or ChaCha20-Poly1305) with a fresh random 12-byte nonce per value stored alongside it, the row's ID as associated data, and envelope encryption with data keys wrapped by a KMS master key; re-encrypt existing values, and rotate the old key and the third-party tokens themselves if exposure is possible.
2. With HS256, verification and creation use the same secret, so the attacker can mint valid tokens for any user and any role, accepted by all thirty services, until the secret is rotated everywhere. Redesign: sign tokens with an asymmetric algorithm (ES256 or RS256, or EdDSA) at one issuer whose private key sits in a KMS or hardware module; services verify with public keys fetched from the issuer's key set, with key identifiers to support rotation; keep tokens short-lived. A compromised service can then read tokens it receives but cannot create new ones.
3. (a) Re-encryption at the load balancer: the balancer keeps routing on HTTP and opens TLS to each service, using certificates from an internal CA. Simple for applications, but the balancer still sees plaintext and holds keys, and services do not learn which service called them. (b) mTLS between services, through a service mesh or workload-identity system with short-lived certificates issued and rotated automatically: every hop is encrypted and each service can authorise callers by identity, at the cost of operating the mesh or CA and of debugging handshake failures. Passthrough is a third option where the balancer need not read HTTP.
4. `includeSubDomains` told every browser that visited `example.com` to use only HTTPS for every subdomain, including `*.corp.example.com`, for two years, and to refuse to click through certificate errors. Tools that serve plain HTTP, or HTTPS with internal certificates the browsers do not trust, become unreachable. Browsers cache the policy until `max-age` expires, and if the domain was added to the preload list, the policy ships inside browsers and removal takes months. It should have been preceded by an inventory of subdomains, moving all of them to HTTPS with trusted certificates (or to a separate domain for internal tools), and a gradual rollout starting with a short `max-age` and without `preload`.
5. Anyone who copied the key can impersonate the server to clients until the certificate is revoked and replaced (given a way onto the network path or DNS). Past traffic: if every handshake used ECDHE (TLS 1.3, or TLS 1.2 with ECDHE suites), recorded sessions stay safe because of forward secrecy; any sessions that used RSA key transport can be decrypted by someone who recorded them. Session-ticket keys are a separate secret, but rotate them too. Actions in order: generate a new key pair and certificate, deploy it with the full chain, revoke the old certificate, rotate session-ticket keys, remove the key from the repository and its history (and treat it as permanently public anyway), check CT logs for unexpected certificates for your names, review which TLS versions and suites were allowed, and find out how the key reached the repository.
6. For example: (1) TLS ends at the load balancer; behind it, traffic and the balancer itself see plaintext. (2) Application bugs such as broken access control (one customer reading another's order by changing an ID) travel over HTTPS perfectly. (3) Data at rest (databases, backups, logs that record request bodies) is not protected by HTTPS. (4) Endpoints: a compromised server, or malware on a customer's phone, sees everything. (5) Metadata: IP addresses, the server name in SNI, and traffic patterns remain visible. A sixth worth adding: any client code that disables certificate verification has encryption with no authentication.
7. Enable 0-RTT, if at all, only for requests that are safe to replay: `GET /catalog` qualifies; `POST /orders` and `POST /payments` do not, because an attacker could replay a recorded first flight. The edge must mark requests received as early data (`Early-Data: 1`) and either refuse early data for anything but safe methods or forward the marker to the application, which must answer `425 Too Early` for anything not replay-safe so that the client retries after the handshake completes. Measure the real gain: 0-RTT only helps resumed connections, and connection reuse often already removes the handshake from most requests.

---

**END OF CHAPTER 8**
