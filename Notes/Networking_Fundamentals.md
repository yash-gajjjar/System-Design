# Networking Fundamentals — Complete Deep-Dive Revision Guide
## System Design Interview Preparation | Product-Based Companies

---

**Prepared for:** Yash | AI/GenAI Engineer transitioning to Product Company System Design Interviews
**Coverage:** HTTP/HTTPS · DNS · TCP vs UDP · REST vs GraphQL vs gRPC · WebSockets · HTTP/2 & HTTP/3 · CDN · OSI Model

---

## How This Guide Is Organized

Each topic follows the same structure:
1. **What Problem Does It Solve?** — the "why" before the "what"
2. **Core Intuition** — beginner-friendly explanations with analogies
3. **Step-by-step Diagrams** — ASCII diagrams of every protocol flow
4. **Deep Dive** — internals, algorithms, frame structures
5. **Comparison Tables** — tradeoffs for every design decision
6. **Real-World Usage** — how Google, Netflix, Uber, Stripe, Discord etc. actually use these
7. **Failure Scenarios** — what breaks in production and how to fix it
8. **Interview Quick-Fire Answers** — ready-to-use answers for common questions

---

## Table of Contents

1. **HTTP / HTTPS** — Request/response anatomy, methods, status codes, headers, TLS handshake, caching
2. **DNS** — Resolution flow, record types, TTL tradeoffs, load balancing strategies, security
3. **TCP vs UDP** — Handshakes, congestion control, when to use each, real-world protocol choices
4. **REST vs GraphQL vs gRPC** — API design philosophies, N+1 problem, protobuf, decision framework
5. **WebSockets** — Upgrade handshake, frame structure, scaling architecture, heartbeats
6. **HTTP/2 & HTTP/3** — Multiplexing, HPACK, QUIC, connection migration, 0-RTT
7. **CDN** — Architecture, cache invalidation strategies, edge compute, Netflix Open Connect
8. **OSI Model** — All 7 layers, encapsulation, L4 vs L7 load balancers, troubleshooting methodology
9. **Appendix** — Cross-topic quick reference, master comparison tables, study tips

---

# Networking Fundamentals — Deep-Dive System Design Notes
### For Product-Based Company Interviews | Beginner → Advanced

---

> **How to use these notes:** Read each section top to bottom the first time. The structure always goes:
> What is it → Why does it exist → How it works step by step → Diagrams → Internals → Tradeoffs → Real-world → Failures → Interview tips.
> Every concept is explained from scratch — no prior networking knowledge assumed.

---

# TOPIC 1: HTTP / HTTPS

---

## 1. What Problem Does HTTP Solve?

Imagine you want to load `www.google.com` in your browser. Your browser is a program running on your laptop. Google's server is a computer sitting in a data center in another country. These two programs need to communicate — but how?

They need a **shared language** — a protocol that defines:
- How does the browser say "please give me this page"?
- How does the server say "here it is" or "it doesn't exist"?
- How does the browser know when the message starts and ends?
- How do they handle errors?

**HTTP (HyperText Transfer Protocol)** is that shared language. It was invented by Tim Berners-Lee in 1989 at CERN, originally just to share scientific documents. Today it is the foundation of the entire web — every website, every API, every mobile app backend uses HTTP.

**HTTPS** is HTTP with a security envelope — all communication is encrypted so nobody in the middle can read or tamper with it.

---

## 2. Core Intuition — How HTTP Actually Works

### The Basic Idea: Request and Response

HTTP follows a very simple model called **request-response**:

```
CLIENT (your browser / mobile app / API consumer)
    │
    │──── "I want something" (HTTP Request) ────▶  SERVER
    │
    │◀─── "Here it is" or "Error" (HTTP Response) ──
```

That's it. Every single HTTP interaction follows this pattern. The client always initiates. The server always responds. This is fundamentally different from a phone call (where either side can speak) — HTTP is more like sending a letter and waiting for a reply.

### What Travels Over the Wire

HTTP messages are **plain text** (in HTTP/1.1). You could literally read them if you intercepted them on the wire.

Here is a real HTTP request your browser makes when you type `google.com`:

```
GET / HTTP/1.1
Host: www.google.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)
Accept: text/html,application/xhtml+xml
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

And the response Google sends back:

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Date: Thu, 11 Jun 2026 10:30:00 GMT
Cache-Control: private, max-age=0
Content-Encoding: gzip
Transfer-Encoding: chunked

<!DOCTYPE html>
<html>...
(the actual HTML page content)
```

### HTTP is Stateless

This is one of the most important properties to understand. **Stateless means the server remembers nothing between requests.** Each request is completely independent.

```
Request 1:  "Give me the homepage"
Server:     "Here it is" ← server forgets this happened

Request 2:  "Give me my profile"
Server:     "Who are you? I don't know you" ← has no memory of Request 1
```

This is why websites need **cookies** and **sessions** — they're tools layered on top of HTTP to fake statefulness, because HTTP itself has no memory.

**Why stateless?** Because it makes servers massively scalable. Any server in a cluster of 1,000 machines can handle any request — it doesn't need to hold your "state." If Request 1 went to Server A and Request 2 goes to Server B, that's fine — neither server needs to know about the other.

---

## 3. Anatomy of an HTTP Request — Every Part Explained

```
┌─────────────────────────────────────────────────────────────────┐
│  REQUEST LINE                                                   │
│  POST /api/v1/orders HTTP/1.1                                   │
│  ─────────────────────────────                                  │
│  │     │              │                                         │
│  │     │              └─── Protocol version                     │
│  │     └──────────────── Path (resource being requested)        │
│  └────────────────────── HTTP Method (verb — what to do)        │
├─────────────────────────────────────────────────────────────────┤
│  HEADERS (key: value pairs, one per line)                       │
│                                                                 │
│  Host: api.mystore.com          ← Which server (required!)     │
│  Content-Type: application/json ← What format is my body?      │
│  Authorization: Bearer eyJ...   ← Prove who I am               │
│  Content-Length: 47             ← How many bytes in body?       │
│  Accept: application/json       ← What format do I want back?  │
│  Connection: keep-alive         ← Don't close TCP after this   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  BLANK LINE (mandatory separator between headers and body)      │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  BODY (optional — only for POST, PUT, PATCH)                    │
│                                                                 │
│  {"item": "Nike Air Max", "qty": 2, "size": "42"}               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Anatomy of an HTTP Response — Every Part Explained

```
┌─────────────────────────────────────────────────────────────────┐
│  STATUS LINE                                                    │
│  HTTP/1.1 201 Created                                           │
│  ────────────────────                                           │
│  │       │   │                                                  │
│  │       │   └─── Human-readable status text                    │
│  │       └─────── Status code (machine-readable number)         │
│  └─────────────── Protocol version                              │
├─────────────────────────────────────────────────────────────────┤
│  RESPONSE HEADERS                                               │
│                                                                 │
│  Content-Type: application/json     ← What format is body?     │
│  Content-Length: 83                 ← How many bytes?           │
│  Location: /api/v1/orders/ORD-789  ← Where is new resource?    │
│  Cache-Control: no-store            ← Don't cache this!         │
│  X-Request-Id: f47ac10b-58cc        ← Trace ID for debugging    │
│  Date: Thu, 11 Jun 2026 10:30:01   ← When server responded     │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  BLANK LINE                                                     │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  BODY                                                           │
│                                                                 │
│  {"orderId": "ORD-789", "status": "confirmed", "total": 4999}   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. HTTP Methods — Deep Explanation

Think of HTTP methods as **verbs** that describe what operation you want to perform on a **resource** (noun).

### GET — Read/Fetch

Used to retrieve a resource. Should NEVER modify anything on the server. Safe to call multiple times — the 100th GET should return the same result as the 1st (assuming nothing changed server-side).

```
GET /users/123          → Get user with ID 123
GET /products?page=2    → Get page 2 of products
GET /orders/456/items   → Get items in order 456
```

### POST — Create

Used to create a new resource. Calling it twice creates two resources (not idempotent). The server decides the new resource's ID/URL.

```
POST /users              → Create a new user
Body: {"name": "Yash", "email": "yash@example.com"}
Response: 201 Created, Location: /users/789
```

### PUT — Replace (Full Update)

Replace an ENTIRE resource. If you PUT a user with only `name` in the body, the `email` field gets deleted — you must send all fields. Idempotent — calling PUT twice with the same data leaves the same result.

```
PUT /users/123
Body: {"name": "Yash Sharma", "email": "yash@new.com", "age": 28}
→ Replaces the entire user object
```

### PATCH — Partial Update

Only update the fields you send. Other fields are untouched.

```
PATCH /users/123
Body: {"email": "newemail@example.com"}
→ Only email is updated; name, age etc. unchanged
```

### DELETE — Remove

Delete a resource. Idempotent — deleting something that's already deleted returns 404 or 204, but doesn't cause an error in most designs.

```
DELETE /users/123    → Delete user 123
Response: 204 No Content
```

### Key Properties Table

```
┌──────────┬─────────────┬──────────────┬────────────────────────────────────────┐
│ Method   │ Idempotent? │ Safe?        │ Meaning                                │
├──────────┼─────────────┼──────────────┼────────────────────────────────────────┤
│ GET      │ YES         │ YES          │ Calling N times = same result, no      │
│          │             │              │ side effects                           │
├──────────┼─────────────┼──────────────┼────────────────────────────────────────┤
│ POST     │ NO          │ NO           │ Creates new resource each call         │
├──────────┼─────────────┼──────────────┼────────────────────────────────────────┤
│ PUT      │ YES         │ NO           │ Same body PUT twice = same server      │
│          │             │              │ state (but modifies server)            │
├──────────┼─────────────┼──────────────┼────────────────────────────────────────┤
│ PATCH    │ MAYBE       │ NO           │ Depends on implementation              │
├──────────┼─────────────┼──────────────┼────────────────────────────────────────┤
│ DELETE   │ YES         │ NO           │ Delete twice: first=204, second=404    │
├──────────┼─────────────┼──────────────┼────────────────────────────────────────┤
│ HEAD     │ YES         │ YES          │ Like GET but response has no body —    │
│          │             │              │ used to check resource existence/size  │
├──────────┼─────────────┼──────────────┼────────────────────────────────────────┤
│ OPTIONS  │ YES         │ YES          │ Ask server: "what methods do you       │
│          │             │              │ support for this URL?" (CORS preflight)│
└──────────┴─────────────┴──────────────┴────────────────────────────────────────┘

Idempotent = calling N times has same effect as calling once
Safe       = does not modify server state (read-only)
```

---

## 6. HTTP Status Codes — Complete Interview Guide

Status codes are grouped in hundreds. The hundred tells you the category.

```
1xx — Informational    (rare, mostly internal protocol use)
2xx — Success          (your request worked)
3xx — Redirection      (go look somewhere else)
4xx — Client Error     (YOU made a mistake in the request)
5xx — Server Error     (the SERVER made a mistake)
```

### The Codes You WILL Be Asked About

```
┌──────┬──────────────────────────┬──────────────────────────────────────────────┐
│ Code │ Name                     │ When it happens & interview relevance        │
├──────┼──────────────────────────┼──────────────────────────────────────────────┤
│ 200  │ OK                       │ GET/PATCH/DELETE succeeded with body         │
│ 201  │ Created                  │ POST succeeded; new resource created         │
│      │                          │ Best practice: include Location header       │
│ 204  │ No Content               │ Success but no body (DELETE, some PATCHes)  │
├──────┼──────────────────────────┼──────────────────────────────────────────────┤
│ 301  │ Moved Permanently        │ Resource permanently at new URL; browser     │
│      │                          │ & search engines update their links          │
│ 302  │ Found (Temp Redirect)    │ Temporarily at different URL; don't update  │
│ 304  │ Not Modified             │ Client's cached version is still fresh       │
│      │                          │ KEY: sent when ETag/Last-Modified match      │
│      │                          │ — server sends NO body, saves bandwidth      │
├──────┼──────────────────────────┼──────────────────────────────────────────────┤
│ 400  │ Bad Request              │ Request is malformed / invalid JSON          │
│ 401  │ Unauthorized             │ "Who are you?" — authentication needed       │
│      │                          │ Common mistake: people confuse 401 vs 403   │
│ 403  │ Forbidden                │ "I know who you are, but you can't do this" │
│      │                          │ Authenticated but lacks permission          │
│ 404  │ Not Found                │ Resource doesn't exist at this URL          │
│ 405  │ Method Not Allowed       │ URL exists but not with this HTTP verb       │
│ 409  │ Conflict                 │ Race condition — concurrent modification     │
│      │                          │ Used in optimistic locking                  │
│ 422  │ Unprocessable Entity     │ JSON valid but business logic failed        │
│      │                          │ e.g., "age must be > 0"                     │
│ 429  │ Too Many Requests        │ RATE LIMITING — critical for system design  │
│      │                          │ Must include Retry-After header             │
├──────┼──────────────────────────┼──────────────────────────────────────────────┤
│ 500  │ Internal Server Error    │ Generic server crash — something broke       │
│ 502  │ Bad Gateway              │ LB/proxy got invalid response from upstream │
│ 503  │ Service Unavailable      │ Server overloaded or in maintenance         │
│      │                          │ Should include Retry-After header           │
│ 504  │ Gateway Timeout          │ LB/proxy timed out waiting for upstream     │
│      │                          │ KEY: check circuit breaker, increase timeout│
└──────┴──────────────────────────┴──────────────────────────────────────────────┘
```

### 401 vs 403 — The Confusion

```
You enter a hotel:

401 = The security guard stops you at the door.
      "Do you have a room key? Show it to me."
      → You need to authenticate first.

403 = You show your key, guard scans it.
      "Sorry, you have a standard room key, 
       but this is the Executive Lounge — you don't have access."
      → You authenticated, but lack permission.
```

---

## 7. HTTP Headers — Deep Dive

Headers are key-value pairs that provide metadata about the request or response. They don't carry the actual content — they describe it, control caching, handle security, etc.

### Caching Headers (Critical for System Design)

```
SCENARIO: Browser fetches image.png from server

First request:
┌─ Server Response ─────────────────────────────────────────┐
│ HTTP/1.1 200 OK                                           │
│ ETag: "33a64df551425fcc55e"     ← fingerprint of content │
│ Cache-Control: max-age=3600     ← cache for 3600 seconds  │
│ Last-Modified: Thu, 11 Jun 2026 09:00:00 GMT              │
│                                                           │
│ [image binary data]                                       │
└───────────────────────────────────────────────────────────┘

Browser caches image + ETag + timestamp.
3600 seconds later, cache expires. Browser needs image again.

Second request (CONDITIONAL GET):
┌─ Browser Request ────────────────────────────────────────┐
│ GET /image.png HTTP/1.1                                  │
│ If-None-Match: "33a64df551425fcc55e"   ← send ETag back │
│ If-Modified-Since: Thu, 11 Jun 2026 09:00:00 GMT         │
└──────────────────────────────────────────────────────────┘

If unchanged → Server says:
HTTP/1.1 304 Not Modified    ← No body! Saves bandwidth.

If changed → Server says:
HTTP/1.1 200 OK
ETag: "newfingerprint"
[new image data]
```

### Cache-Control Directives Explained

```
Cache-Control: max-age=3600
    └─ Cache this for 3600 seconds (1 hour) — then it's "stale"

Cache-Control: no-cache
    └─ MISLEADING NAME. Does NOT mean "don't cache".
       Means "cache it, but ALWAYS validate with server before using"
       → Every request gets a conditional GET (304 if unchanged)

Cache-Control: no-store
    └─ Don't cache at all. Ever. (Used for sensitive data like banking)

Cache-Control: private
    └─ Only the user's browser can cache this (not CDN/proxies)
       (Used for user-specific data: profile pages, dashboards)

Cache-Control: public
    └─ Anyone can cache this (CDN edge servers, proxies, browsers)

Cache-Control: immutable
    └─ This resource will NEVER change — don't even bother validating
       (Used with content-hashed filenames like main.a3f8.js)

Cache-Control: s-maxage=86400
    └─ Override max-age for shared caches (CDNs) only
       Browser uses max-age, CDN uses s-maxage
```

### Security Headers

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
    └─ HSTS: "For the next 1 year, ONLY contact me via HTTPS"
       Browser refuses to even try HTTP — prevents downgrade attacks

Content-Security-Policy: default-src 'self'; script-src 'self' cdn.example.com
    └─ CSP: Prevents XSS — only load scripts from approved sources

X-Frame-Options: DENY
    └─ Prevents your page from being embedded in <iframe>
       Stops clickjacking attacks

X-Content-Type-Options: nosniff
    └─ Browser must use Content-Type header, not guess file type
       Prevents MIME-type sniffing attacks

Access-Control-Allow-Origin: https://app.example.com
    └─ CORS: Which domains can call this API from browser JS?
       * = anyone (dangerous for authenticated APIs!)
```

---

## 8. How HTTPS Works — TLS Explained From Scratch

### Why Do We Need HTTPS?

Without HTTPS, HTTP traffic is plain text on the wire. Anyone on your network (coffee shop WiFi, your ISP, a hacker between you and the server) can:
- **Read** everything you send/receive (passwords, credit cards)
- **Modify** the content (inject ads, malware, redirect links)
- **Impersonate** the server (send you a fake page)

HTTPS prevents all three by using **TLS (Transport Layer Security)**.

### What TLS Provides

```
┌─────────────────────────────────────────────────────────────┐
│  TLS provides THREE guarantees:                             │
│                                                             │
│  1. ENCRYPTION — Data is scrambled. Only the intended       │
│     recipient can unscramble it.                            │
│     (symmetric encryption: AES-256-GCM)                    │
│                                                             │
│  2. AUTHENTICATION — You can prove the server is who it     │
│     claims to be (not an impersonator).                     │
│     (digital certificates: X.509 + PKI)                    │
│                                                             │
│  3. INTEGRITY — Data cannot be tampered with in transit.    │
│     Any modification is detected and rejected.             │
│     (HMAC: Hash-based Message Authentication Code)         │
└─────────────────────────────────────────────────────────────┘
```

### TLS Handshake — Step by Step (TLS 1.3)

```
CLIENT (Browser)                        SERVER (google.com)
       │                                        │
       │── ClientHello ────────────────────────▶│
       │   "I support TLS 1.3"                  │
       │   "My random number: abc123"            │
       │   "I support: AES-256, ChaCha20"        │
       │   "My key share: (ECDHE public key)"    │
       │                                        │
       │◀─ ServerHello ─────────────────────────│
       │   "Let's use TLS 1.3 + AES-256-GCM"   │
       │   "My random number: xyz789"            │
       │   "My key share: (ECDHE public key)"    │
       │   "My certificate: [signed by CA]"      │
       │   "Certificate Verify: [signature]"     │
       │   "Finished: [HMAC of handshake]"       │
       │                                        │
       │── (Verify certificate) ───────────────▶│
       │   Browser checks: Is cert signed by     │
       │   a trusted CA (DigiCert, Let's         │
       │   Encrypt, etc.)? Is CN = google.com?  │
       │   Is cert expired?                      │
       │                                        │
       │── Finished ───────────────────────────▶│
       │   "I verified everything. Ready."       │
       │                                        │
       │════════ ENCRYPTED HTTP TRAFFIC ════════│
       │   (symmetric key derived from both     │
       │   ECDHE key shares + random numbers)   │
```

Key insight: **TLS 1.3 does this in 1 Round Trip Time (RTT)** — the handshake takes just one back-and-forth. TLS 1.2 needed 2 RTTs. This matters enormously for mobile users.

### What is a Certificate? (PKI Explained Simply)

```
PROBLEM: When browser connects to bank.com, how does it know
         it's really talking to the real bank and not an attacker?

SOLUTION: Digital Certificates + Certificate Authorities (CA)

Chain of Trust:
┌────────────────────────────────────────────────────────────┐
│  Root CA (DigiCert, Let's Encrypt, etc.)                  │
│  "We are the ultimate trust anchor"                        │
│  Browsers come pre-installed with ~100 trusted root CAs    │
│           │                                                │
│           └─ Intermediate CA                               │
│              "We were authorized by Root CA"               │
│                       │                                    │
│                       └─ bank.com's Certificate            │
│                          "We were authorized by             │
│                           Intermediate CA"                 │
│                          Contains: domain name,            │
│                          public key, expiry date,          │
│                          CA signature                      │
└────────────────────────────────────────────────────────────┘

Browser verifies the entire chain:
- bank.com cert signed by → Intermediate CA ✓
- Intermediate CA signed by → Root CA ✓  
- Root CA is in browser's trust store ✓
- Domain name matches ✓
- Not expired ✓
→ TRUSTED!
```

---

## 9. HTTP Caching — The Complete Mental Model

Caching is one of the most tested topics in system design interviews. Understanding HTTP caching well is essential.

### Where Caches Live

```
REQUEST JOURNEY (with caches):

Browser ─▶ Browser Cache ─▶ CDN Edge Cache ─▶ Reverse Proxy Cache ─▶ Origin Server
          (your laptop)    (nearest PoP)       (nginx/Varnish)

RESPONSE JOURNEY (the response fills caches on the way back):

Origin ─▶ Reverse Proxy caches it ─▶ CDN caches it ─▶ Browser caches it

Next request at the SAME browser:
Browser Cache HIT → no network call at all (0ms!)

Next request from DIFFERENT user, SAME CDN region:
CDN Cache HIT → served from edge, no origin call
(typically 5-20ms instead of 200ms cross-country)
```

### Cache Decision Flow

```
Browser makes request:

Is there a cached response?
├── NO → Fetch from server, cache the response, return it
└── YES → Is it fresh? (current time < cached time + max-age?)
    ├── YES (Cache HIT, fresh) → Return cached response immediately. Done!
    └── NO (Cache STALE) → Do I have an ETag or Last-Modified?
        ├── YES → Send conditional GET to server
        │         Server says 304? → Refresh timestamp, return cached body
        │         Server says 200? → Replace cache with new response
        └── NO → Fetch fresh from server, cache it, return it
```

---

## 10. Key HTTP Headers Reference Table

```
┌─────────────────────────┬───────────┬──────────────────────────────────────────┐
│ Header                  │ Direction │ Purpose & Interview Notes                │
├─────────────────────────┼───────────┼──────────────────────────────────────────┤
│ Host                    │ Request   │ Which virtual host? REQUIRED in HTTP/1.1  │
│ Content-Type            │ Both      │ application/json, multipart/form-data     │
│ Content-Length          │ Both      │ Body size in bytes                        │
│ Authorization           │ Request   │ Bearer <JWT>, Basic <base64>              │
│ Cookie                  │ Request   │ Stored cookies sent to server             │
│ Set-Cookie              │ Response  │ Server instructs browser to store cookie  │
│ Accept                  │ Request   │ "I want JSON back" (content negotiation)  │
│ Accept-Encoding         │ Request   │ "I can handle gzip, br compression"       │
│ Content-Encoding        │ Response  │ "Body is gzip compressed"                 │
│ Cache-Control           │ Both      │ Caching directives (max-age, no-cache…)   │
│ ETag                    │ Response  │ Resource version fingerprint              │
│ If-None-Match           │ Request   │ "Send 304 if ETag still matches"          │
│ Last-Modified           │ Response  │ When resource last changed                │
│ If-Modified-Since       │ Request   │ "Send 304 if unchanged since this date"   │
│ Location                │ Response  │ Redirect URL (3xx) or new resource (201)  │
│ Retry-After             │ Response  │ Seconds to wait before retrying (429/503) │
│ Connection              │ Both      │ keep-alive (persistent) vs close          │
│ Transfer-Encoding       │ Response  │ chunked = body sent in pieces             │
│ X-Request-Id            │ Both      │ Trace ID for distributed tracing          │
│ Strict-Transport-Security│ Response │ HSTS — force HTTPS                        │
│ Access-Control-*        │ Response  │ CORS headers                              │
│ Content-Security-Policy │ Response  │ Prevent XSS — approved script sources     │
└─────────────────────────┴───────────┴──────────────────────────────────────────┘
```

---

## 11. Tradeoffs and Design Decisions

### HTTP vs HTTPS

```
┌──────────────────┬─────────────────────────┬────────────────────────────────┐
│ Concern          │ HTTP                    │ HTTPS                          │
├──────────────────┼─────────────────────────┼────────────────────────────────┤
│ Security         │ Plaintext. Anyone on    │ Encrypted end-to-end.          │
│                  │ network can intercept.  │ MitM attacks prevented.        │
│ Performance      │ No TLS overhead.        │ ~1ms extra for TLS resumption. │
│                  │ (Negligible in practice)│ TLS 1.3 = 1 RTT handshake.    │
│ SEO              │ Google penalizes sites. │ Positive ranking signal.       │
│ Trust            │ Browser shows warning.  │ Padlock icon = user trust.     │
│ Cost             │ Free                    │ Cert free (Let's Encrypt).     │
│                  │                         │ CPU overhead negligible.       │
│ HTTP/2           │ Not supported (spec      │ Required for HTTP/2 in all    │
│                  │ allows but browsers     │ browsers.                      │
│                  │ refuse in practice)     │                                │
└──────────────────┴─────────────────────────┴────────────────────────────────┘
VERDICT: Always use HTTPS. No exceptions for public traffic.
```

### REST Verbs: PUT vs PATCH

```
Scenario: Update user's email only

PUT /users/123
Body: {"name": "Yash", "email": "new@email.com", "age": 28}
→ Must send ALL fields. Missing fields get nulled/deleted.
→ Safe if client always has full object

PATCH /users/123  
Body: {"email": "new@email.com"}
→ Only email updated. Other fields untouched.
→ Safer for partial updates. Less bandwidth.
→ NOT idempotent by default (depends on implementation)

INTERVIEW: PATCH is preferred for partial updates in REST APIs.
PUT is used when you want to guarantee full replacement (file uploads, config).
```

---

## 12. Failure Scenarios & Mitigations

```
┌─────────────────────┬───────────────────────────┬──────────────────────────────┐
│ Failure             │ Root Cause                │ Mitigation                   │
├─────────────────────┼───────────────────────────┼──────────────────────────────┤
│ 504 Gateway Timeout │ Upstream service too slow;│ Tune timeout thresholds;     │
│                     │ load balancer gives up    │ circuit breaker pattern;     │
│                     │                           │ async + polling for slow ops │
├─────────────────────┼───────────────────────────┼──────────────────────────────┤
│ TLS cert expiry     │ Certificate not renewed   │ Auto-renew (Let's Encrypt);  │
│ (site shows warning)│ before expiry date        │ monitor cert expiry -30 days │
├─────────────────────┼───────────────────────────┼──────────────────────────────┤
│ CORS blocked        │ Missing                   │ Configure CORS on API        │
│                     │ Access-Control-Allow-     │ gateway; return correct       │
│                     │ Origin header             │ headers for OPTIONS preflight│
├─────────────────────┼───────────────────────────┼──────────────────────────────┤
│ Cache serving stale │ max-age too high; content │ Use content-hash filenames   │
│ content after deploy│ changed before TTL expired│ for static assets; purge CDN │
├─────────────────────┼───────────────────────────┼──────────────────────────────┤
│ Large payloads slow │ Uncompressed JSON/HTML     │ Enable gzip/Brotli           │
│ down responses      │                           │ compression on server        │
├─────────────────────┼───────────────────────────┼──────────────────────────────┤
│ Client overloads    │ No rate limiting on API    │ Implement 429 Too Many       │
│ server with requests│                           │ Requests with Retry-After    │
└─────────────────────┴───────────────────────────┴──────────────────────────────┘
```

---

## 13. Real-World Company Usage

**Google:** Introduced HTTPS-as-ranking-signal in 2014. Uses HSTS preloading — all google.com subdomains are in browser hardcoded lists, cannot be accessed via HTTP even on first visit.

**Netflix:** Uses HTTP/2 multiplexing for API calls to backend services. Every API response carries `X-Netflix-*` custom headers for internal tracing. Cache-Control tuned per content type: static assets (max-age=31536000 + immutable), manifests (max-age=10), user data (no-store).

**Stripe:** Their public REST API returns consistent error objects with `type`, `code`, `message`, `param` fields — best-in-class error design. Uses `Idempotency-Key` header on POST requests — submit same request twice with same key and it's safe (no duplicate charge).

**Twitter/X:** Uses rate limiting extensively — 429 with `X-Rate-Limit-Remaining: 0` and `X-Rate-Limit-Reset: <timestamp>` headers tell clients exactly when to retry.

---

## 14. Interview Quick-Fire Answers

**Q: What is the difference between 401 and 403?**
A: 401 means "not authenticated — prove who you are." 403 means "I know who you are, but you don't have permission."

**Q: What does idempotent mean? Which HTTP methods are idempotent?**
A: Idempotent means calling the operation N times has the same effect as calling it once. GET, PUT, DELETE, HEAD, OPTIONS are idempotent. POST is not.

**Q: How does HTTP caching work?**
A: Server includes Cache-Control (max-age) and ETag in responses. Browser caches content. When cache expires, browser sends If-None-Match with ETag — server returns 304 if unchanged (no body, saves bandwidth) or 200 with new content.

**Q: Why is HTTPS not just about encryption?**
A: HTTPS provides three things: encryption (nobody can read data in transit), authentication (certificate proves server identity), and integrity (HMAC detects tampering). Without authentication, a hacker could still intercept and impersonate the server with their own encryption.

**Q: What is CORS and why does it exist?**
A: CORS (Cross-Origin Resource Sharing) is a browser security mechanism. By default, JavaScript on `app.mysite.com` cannot call APIs on `api.othersite.com` — the browser blocks it. The API server opts in by returning `Access-Control-Allow-Origin` headers. This prevents malicious websites from making unauthorized API calls using a logged-in user's cookies.

---
---

# TOPIC 2: DNS — Domain Name System

---

## 1. What Problem Does DNS Solve?

Computers communicate using IP addresses — numerical labels like `142.250.80.46`. But humans are terrible at remembering numbers. Nobody wants to type `142.250.80.46` to visit Google.

DNS is the internet's **phonebook** — a system that translates human-readable names like `google.com` into machine-readable IP addresses like `142.250.80.46`.

But DNS is far more than just a phonebook. At scale, it's also a:
- **Load balancer** (return different IPs for different users)
- **Geographic router** (send European users to European servers)
- **Failover mechanism** (if primary server dies, return backup IP)
- **Service discovery system** (in Kubernetes, microservices find each other via DNS)

---

## 2. Core Intuition — The Phonebook Analogy

```
Imagine you want to call "Pizza Palace" restaurant.
You don't know their number, so you use a phonebook.

DNS works in a hierarchy of phonebooks:

┌─────────────────────────────────────────────────────────────┐
│  MASTER LIBRARIAN (Root DNS — 13 root server clusters)      │
│  "I don't have the number, but I know which regional         │
│   phonebook office handles .com names — go ask them."       │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  REGIONAL LIBRARIAN (TLD DNS — e.g., Verisign for .com)     │
│  "I don't have google.com's number, but I know Google's      │
│   own nameserver handles their records — go ask them."       │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  GOOGLE'S OWN LIBRARIAN (Authoritative DNS — ns1.google.com)│
│  "Yes! google.com's IP address is 142.250.80.46"            │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. DNS Resolution — Complete Step-by-Step Walk-Through

Let's trace exactly what happens when you type `www.google.com` in your browser and hit Enter:

```
YOUR BROWSER                  YOUR OS                RECURSIVE RESOLVER        ROOT NS             .COM TLD NS         GOOGLE AUTH NS
    │                            │                    (8.8.8.8 or ISP)              │                    │                    │
    │                            │                           │                      │                    │                    │
    ├─ Step 1: Check browser cache ─┤                        │                      │                    │                    │
    │  "Did I look up google.com    │                        │                      │                    │                    │
    │   recently?" If yes → done.   │                        │                      │                    │                    │
    │                            │                           │                      │                    │                    │
    ├─ Step 2: Ask OS ──────────▶│                           │                      │                    │                    │
    │                            ├─ Check /etc/hosts ──────▶│                       │                    │                    │
    │                            │  (local override file)   │                      │                    │                    │
    │                            │                           │                      │                    │                    │
    │                            ├─ Check OS DNS cache ─────▶│                      │                    │                    │
    │                            │                           │                      │                    │                    │
    │                            ├─ Ask Recursive Resolver──▶│                      │                    │                    │
    │                            │                           │                      │                    │                    │
    │                            │                           ├── Step 3: Ask Root ─▶│                    │                    │
    │                            │                           │   "Who handles .com?"│                    │                    │
    │                            │                           │◀──────────────────── │                    │                    │
    │                            │                           │   "Ask 192.5.6.30    │                    │                    │
    │                            │                           │    (Verisign TLD NS)"│                    │                    │
    │                            │                           │                      │                    │                    │
    │                            │                           ├── Step 4: Ask TLD ─────────────────────▶│                    │
    │                            │                           │   "Who handles       │                    │                    │
    │                            │                           │    google.com?"      │                    │                    │
    │                            │                           │◀───────────────────────────────────────── │                    │
    │                            │                           │   "Ask ns1.google.com│                    │                    │
    │                            │                           │    (216.239.32.10)"  │                    │                    │
    │                            │                           │                      │                    │                    │
    │                            │                           ├── Step 5: Ask Auth ──────────────────────────────────────────▶│
    │                            │                           │   "What is the IP    │                    │                    │
    │                            │                           │    of google.com?"   │                    │                    │
    │                            │                           │◀─────────────────────────────────────────────────────────────  │
    │                            │                           │   "142.250.80.46,    │                    │                    │
    │                            │                           │    TTL=300 seconds"  │                    │                    │
    │                            │                           │                      │                    │                    │
    │                            │                           ├─ Step 6: Cache result│                    │                    │
    │                            │                           │  for 300 seconds     │                    │                    │
    │                            │                           │                      │                    │                    │
    │                            │◀── IP: 142.250.80.46 ─── │                      │                    │                    │
    │◀─── IP: 142.250.80.46 ─────│                           │                      │                    │                    │
    │                            │                           │                      │                    │                    │
    ├─ Step 7: Connect to 142.250.80.46:443 (HTTPS)          │                      │                    │                    │
```

**Performance note:** Steps 3-5 each require a network round trip. On first visit, DNS resolution can take 100-200ms. This is why caching (Steps 1-2-6) is so important.

---

## 4. DNS Record Types — Every One Explained

### A Record (The Most Common)

Maps a domain to an IPv4 address. The fundamental DNS record.

```
google.com.    300    IN    A    142.250.80.46
│              │      │     │    │
│              │      │     │    └── IPv4 address
│              │      │     └─────── Record type
│              │      └───────────── Class (IN = Internet, always)
│              └──────────────────── TTL in seconds
└─────────────────────────────────── Domain name (trailing dot = FQDN)
```

You can have MULTIPLE A records for one domain → round-robin DNS load balancing:

```
google.com.    300    IN    A    142.250.80.46
google.com.    300    IN    A    142.250.80.47
google.com.    300    IN    A    142.250.80.48
→ DNS resolver returns all 3; client typically picks the first one
```

### AAAA Record (IPv6)

```
google.com.    300    IN    AAAA    2607:f8b0:4004:c07::65
                                   └── IPv6 address (128-bit, 8 groups of 4 hex digits)
```

### CNAME Record (Canonical Name — Alias)

Maps one domain name to another domain name (not to an IP directly).

```
www.google.com.    3600    IN    CNAME    google.com.
                                          └── The "real" name — resolver then looks up google.com's A record

shop.company.com.  300     IN    CNAME    company.myshopify.com.
                                          └── Delegated to Shopify's infrastructure
```

**Critical restriction:** CNAME records cannot exist on the "apex" (root) domain. You cannot do:

```
company.com.    IN    CNAME    company.myshopify.com.    ← INVALID!
```

This is because company.com also needs MX records (for email), SOA records, etc., and the CNAME would replace all of them. This is called the "CNAME at apex" problem. Solutions: ALIAS records (AWS Route 53), ANAME records (some providers), or CNAME Flattening (Cloudflare).

### MX Record (Mail Exchange)

```
company.com.    3600    IN    MX    10    mail1.google.com.
company.com.    3600    IN    MX    20    mail2.google.com.
                                   │     └── Mail server hostname
                                   └── Priority (lower = higher priority)
```

The priority value determines which mail server receives email first. If priority 10 server is down, email goes to priority 20 server.

### TXT Record (Text)

Stores arbitrary text. Used for verification and security.

```
company.com.    3600    IN    TXT    "v=spf1 include:_spf.google.com ~all"
                                     └── SPF record: authorized mail servers

company.com.    3600    IN    TXT    "google-site-verification=abc123xyz"
                                     └── Domain ownership proof for Google

_dmarc.company.com.    IN    TXT    "v=DMARC1; p=reject; rua=mailto:dmarc@company.com"
                                     └── DMARC policy for email authentication
```

### NS Record (Name Server)

Delegates a zone to specific nameservers.

```
google.com.    172800    IN    NS    ns1.google.com.
google.com.    172800    IN    NS    ns2.google.com.
google.com.    172800    IN    NS    ns3.google.com.
google.com.    172800    IN    NS    ns4.google.com.
→ All DNS queries for *.google.com go to these servers
→ High TTL (48h) because nameservers rarely change
```

### SRV Record (Service)

Specifies host AND port for a service. Used in Kubernetes and microservices.

```
_grpc._tcp.userservice.default.svc.cluster.local.    IN    SRV    10 20 50051 pod-abc.userservice.
                                                              │  │  │    └── hostname
                                                              │  │  └─────── port number
                                                              │  └─────────── weight (load balancing)
                                                              └────────────── priority
```

---

## 5. TTL (Time To Live) — The Critical Tradeoff

TTL controls how long resolvers and browsers cache a DNS record before re-querying. It is measured in seconds.

```
TTL = 300   → Cache for 5 minutes
TTL = 3600  → Cache for 1 hour
TTL = 86400 → Cache for 24 hours

TRADEOFF:

┌─────────────────────────────────┬──────────────────────────────────┐
│ LOW TTL (60–300 seconds)        │ HIGH TTL (3600–86400 seconds)    │
├─────────────────────────────────┼──────────────────────────────────┤
│ ✅ Fast incident recovery        │ ✅ Fewer DNS queries              │
│    Change IP → takes effect in  │    Lower cost, lower load on     │
│    60-300 seconds               │    authoritative nameservers     │
│ ✅ Easy A/B traffic splits       │ ✅ Better performance             │
│ ✅ Good for canary deployments   │    Cached IP = no DNS round trip │
│ ❌ More DNS queries              │    on subsequent connections     │
│    More load on auth nameservers│ ❌ Slow recovery during incidents│
│ ❌ Higher latency                │    IP change takes 24h to fully  │
│    More round trips             │    propagate worldwide           │
└─────────────────────────────────┴──────────────────────────────────┘

MIGRATION BEST PRACTICE:
1. Normal operation:     TTL = 86400 (1 day)
2. 48 hours BEFORE migration: Drop TTL to 300 (5 min)
   → Wait for old TTL to expire worldwide
3. During migration:     Change IP record
   → New IP propagates in 5 minutes
4. After migration stable: Raise TTL back to 86400
```

---

## 6. DNS-Based Load Balancing and Traffic Routing

DNS is not just for name resolution — it's a powerful traffic routing layer.

### Round-Robin DNS (Simplest)

```
api.company.com.    60    IN    A    10.0.0.1
api.company.com.    60    IN    A    10.0.0.2
api.company.com.    60    IN    A    10.0.0.3

Different users get different IPs:
User A → resolver returns [10.0.0.1, 10.0.0.2, 10.0.0.3] → uses 10.0.0.1
User B → resolver returns [10.0.0.2, 10.0.0.3, 10.0.0.1] → uses 10.0.0.2
User C → resolver returns [10.0.0.3, 10.0.0.1, 10.0.0.2] → uses 10.0.0.3

PROBLEM: No health checking. If 10.0.0.1 dies, 1/3 of users hit dead server.
SOLUTION: Use a smarter DNS service (AWS Route 53 with health checks).
```

### Weighted Routing (Canary Deployments)

```
Route 90% traffic to v1 (stable), 10% to v2 (new version):

api.company.com. → 90% → 10.0.0.1 (v1 servers)
api.company.com. → 10% → 10.0.0.2 (v2 servers)

Gradual rollout:
Week 1: 90/10
Week 2: 70/30
Week 3: 50/50
Week 4: 0/100 → v2 fully deployed
```

### Geolocation / Latency-Based Routing

```
User in Mumbai     → DNS returns 10.0.1.1 (Mumbai data center)
User in Frankfurt  → DNS returns 10.0.2.1 (Frankfurt data center)
User in Virginia   → DNS returns 10.0.3.1 (US-East data center)

Benefits:
- Lower latency (serve from nearest DC)
- Data residency compliance (EU users served from EU = GDPR)
- Content localization (language, currency)

Used by: AWS Route 53, Cloudflare, Netflix, Uber
```

### Anycast (Used by CDNs and 1.1.1.1 DNS)

```
UNICAST (normal DNS):
One IP → One server in one location
Query from Mumbai → travels to Virginia → high latency

ANYCAST:
Same IP announced from 300 locations via BGP
Query from Mumbai → BGP routing chooses NEAREST location → served from Mumbai
Query from Frankfurt → BGP routing → served from Frankfurt

The SAME IP (e.g., 1.1.1.1) is served by 300+ machines worldwide.
BGP automatically routes to nearest one.

Used by: Cloudflare (1.1.1.1 DNS), Google (8.8.8.8), all major CDNs
```

---

## 7. DNS Security — Attacks and Defenses

### DNS Cache Poisoning

```
ATTACK:
Attacker wants users typing bank.com to reach attacker's server.

Attacker sends thousands of fake DNS responses to a resolver:
"bank.com's IP is 1.2.3.4 (attacker's server)"

If the attacker's response arrives before the real auth server's response
AND the transaction ID (a 16-bit number) matches:
→ Resolver caches the fake record!
→ All users hit attacker's server

SCALE OF PROBLEM:
16-bit transaction ID = only 65,536 possible values
Attacker with fast connection can try all 65,536 in < 1 second!
```

```
DEFENSES:

1. Source Port Randomization (RFC 5452)
   Also randomize the UDP source port (65,536 ports × 65,536 IDs = 4B combinations)
   Makes brute-force infeasible

2. DNSSEC (DNS Security Extensions)
   Auth nameservers SIGN all records with private key
   Resolvers VERIFY signature with public key
   Forged records fail signature verification
   
   DNSSEC Record Chain:
   ROOT has RRSIG (signed by root key)
   .COM has RRSIG (signed by .com key)
   google.com has RRSIG (signed by google's key)
   → Tamper-evident chain of trust

3. DNS over HTTPS (DoH) and DNS over TLS (DoT)
   Normal DNS: plaintext UDP — anyone can see what you're looking up
   DoH/DoT: DNS queries are encrypted — ISPs/attackers can't spy on queries
   DoH used by Firefox, Chrome by default (sends to Cloudflare/Google)
```

### DNS DDoS Amplification Attack

```
ATTACK:
DNS queries are small (30-40 bytes)
DNS responses can be large (up to 4000 bytes with EDNS0)
Amplification factor: up to 100×

Attacker sends spoofed queries (victim's IP as source):
Attacker → DNS resolver: "Give me ALL records for example.com?" (40 bytes)
                         using victim's IP as source address
DNS resolver → Victim:  Response (4000 bytes) × millions of queries
→ Victim gets flooded with 100× amplified traffic it never requested

DEFENSES:
- BCP38: ISPs filter packets with spoofed source IPs
- Response Rate Limiting (RRL): limit identical responses to same IP
- Disable ANY query type (returns all records — most useful for attackers)
- Rate limit DNS query rates per source IP
```

---

## 8. DNS in Kubernetes and Microservices

Modern service discovery uses DNS internally. Understanding this is important for system design interviews involving microservices.

```
KUBERNETES DNS (CoreDNS):

Each pod and service gets a DNS name automatically:

Service name: userservice
Namespace: production
Cluster domain: cluster.local

Full DNS name: userservice.production.svc.cluster.local

Other pods query this name:
order-service → DNS lookup → userservice.production.svc.cluster.local
             ← 10.96.0.12 (service ClusterIP)
             → Load balanced across pods

Short names work within same namespace:
In production namespace: just use "userservice"
Cross-namespace: "userservice.production"

SRV records for port discovery:
_grpc._tcp.userservice.production.svc.cluster.local → host:port
```

---

## 9. Failure Scenarios and Mitigations

```
┌──────────────────────────┬────────────────────────────┬──────────────────────────────┐
│ Failure                  │ Root Cause                 │ Mitigation                   │
├──────────────────────────┼────────────────────────────┼──────────────────────────────┤
│ Authoritative NS down    │ Single nameserver failure  │ Always configure ≥2 auth NS  │
│                          │                            │ in different data centers    │
├──────────────────────────┼────────────────────────────┼──────────────────────────────┤
│ TTL too high during      │ Changed IP but old IP still│ Pre-lower TTL 48h before     │
│ incident/migration       │ cached in millions of      │ planned changes; TTL=60 for  │
│                          │ resolvers                  │ critical services            │
├──────────────────────────┼────────────────────────────┼──────────────────────────────┤
│ Subdomain takeover       │ CNAME points to deleted    │ Audit all CNAME records      │
│ (attacker owns your      │ cloud resource (S3 bucket, │ regularly; delete unused DNS │
│  subdomain)              │ Heroku app, Azure service) │ records immediately          │
├──────────────────────────┼────────────────────────────┼──────────────────────────────┤
│ DNS outage takes down    │ Single DNS provider; no    │ Multi-provider DNS (Route53  │
│ entire service           │ redundancy                 │ + Cloudflare); low TTL       │
│ (Fastly outage 2021)     │                            │ for quick failover           │
├──────────────────────────┼────────────────────────────┼──────────────────────────────┤
│ DNS amplification DDoS   │ Open resolvers exploited   │ BCP38, RRL, block ANY        │
├──────────────────────────┼────────────────────────────┼──────────────────────────────┤
│ Negative caching storm   │ App generating random      │ Negative caching (NXDOMAIN   │
│ (NXDOMAIN flood)         │ nonexistent hostnames      │ TTL in SOA), fix the app     │
└──────────────────────────┴────────────────────────────┴──────────────────────────────┘
```

---

## 10. Real-World DNS Usage

**Netflix:** Uses AWS Route 53 with latency-based routing across 3+ AWS regions. TTL kept at 60 seconds for API endpoints — allows sub-60s failover. Static content CDN uses different DNS (Netflix Open Connect). Internal service discovery uses Eureka (not DNS) for microservices.

**Cloudflare:** Operates the world's fastest public DNS resolver at 1.1.1.1. Uses Anycast — the IP 1.1.1.1 is served from 300+ data centers. Average response time: 11ms globally vs. 57ms for Google's 8.8.8.8 (Cloudflare's claim). Offers DoH and DoT for privacy.

**Uber:** Internal microservices use Kubernetes service discovery (CoreDNS). External traffic uses AWS Route 53 with health checks and weighted routing for canary deployments. Geolocation routing to serve drivers/riders from nearest regional cluster.

---

## 11. Interview Quick-Fire Answers

**Q: What is DNS and why do we need it?**
A: DNS maps human-readable domain names to IP addresses. Computers communicate via IPs, but humans remember names. Without DNS, we'd have to memorize IP addresses for every website.

**Q: What is the difference between recursive and authoritative DNS?**
A: A recursive resolver (like 8.8.8.8) does the work of querying root, TLD, and authoritative servers on your behalf — it recurses through the hierarchy. An authoritative nameserver holds the actual records for a domain and gives definitive answers. The recursive resolver is like a research assistant; the authoritative server is the source of truth.

**Q: What is a TTL and what happens if it's too high during an incident?**
A: TTL (Time To Live) is how many seconds resolvers cache a DNS record. If TTL is 86400 (24 hours) and your server goes down, you change the IP record, but resolvers worldwide still serve the old IP for up to 24 hours — users can't reach you. Best practice: lower TTL to 60-300 seconds before planned changes.

**Q: How do CDNs use DNS?**
A: CDNs use a combination of CNAME records and Anycast. Your domain CNAMEs to the CDN's domain. The CDN uses Anycast or latency-based DNS routing to return the IP of the nearest edge server. This means users get served from a nearby PoP without you managing any of the routing complexity.

**Q: What is DNSSEC?**
A: DNSSEC adds cryptographic signatures to DNS records. Each zone signs its records with a private key. Resolvers verify signatures with the public key. This prevents cache poisoning — forged records fail signature verification. The chain of trust runs: Root → TLD → Domain.

---
---

# TOPIC 3: TCP vs UDP

---

## 1. What Problem Do These Solve?

You've got two computers that want to exchange data. At the IP layer (Layer 3), packets can:
- Arrive out of order
- Get lost (dropped by routers under congestion)
- Get duplicated
- Arrive corrupted

The **Transport Layer (Layer 4)** sits above IP and answers: "How do I reliably (or unreliably-but-fast) get data from one application to another?"

Two answers emerged:
- **TCP** — "I will guarantee every byte arrives, in order, exactly once. I'll retransmit what's lost."
- **UDP** — "I'll do my best, but if something gets lost, tough luck. Here's your raw speed."

Neither is better — they serve different use cases.

---

## 2. What is a Port? (Prerequisite)

Before understanding TCP/UDP, you need to understand ports.

```
Your laptop has ONE IP address: 192.168.1.10
But it runs MANY programs simultaneously:
  - Browser (talking to google.com)
  - Slack (talking to slack.com)
  - VS Code (talking to github.com)
  - Spotify (streaming music)

How does the OS know which incoming packets belong to which program?

PORTS! Each program binds to a unique port number (0–65535).

192.168.1.10:52341 ──▶ 142.250.80.46:443   (browser → google HTTPS)
192.168.1.10:52342 ──▶ 52.204.120.50:443   (slack → slack.com)
192.168.1.10:52343 ──▶ 140.82.112.4:443    (VS Code → github.com)

A "connection" is identified by a 4-tuple:
(source IP, source port, dest IP, dest port)

Well-known ports:
80   → HTTP
443  → HTTPS
53   → DNS
22   → SSH
25   → SMTP (email)
5432 → PostgreSQL
6379 → Redis
3306 → MySQL
```

---

## 3. TCP — Transmission Control Protocol — Deep Dive

### The Core Promise

TCP guarantees that:
1. Every byte sent will be received (no loss)
2. Bytes will be received in the same order they were sent (no reordering)
3. Bytes will be received exactly once (no duplicates)
4. The receiver won't be overwhelmed (flow control)
5. The network won't be overwhelmed (congestion control)

### The TCP Segment Structure

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌─────────────────────────────┬─────────────────────────────────────┐
│         Source Port (16)    │       Destination Port (16)         │
├─────────────────────────────┴─────────────────────────────────────┤
│                    Sequence Number (32)                           │
├───────────────────────────────────────────────────────────────────┤
│                 Acknowledgment Number (32)                        │
├────────┬────────┬─┬─┬─┬─┬─┬─┬──────────────────────────────────┤
│ Data   │Reserved│U│A│P│R│S│F│           Window Size (16)        │
│ Offset │        │R│C│S│S│Y│I│                                    │
│ (4)    │  (6)   │G│K│H│T│N│N│                                    │
├────────┴────────┴─┴─┴─┴─┴─┴─┴──────────────────────────────────┤
│           Checksum (16)           │      Urgent Pointer (16)      │
├────────────────────────────────────────────────────────────────── ┤
│                    Options (variable)                             │
├───────────────────────────────────────────────────────────────────┤
│                         Data (variable)                           │
└───────────────────────────────────────────────────────────────────┘

Key flags:
SYN = Synchronize (start connection)
ACK = Acknowledge (confirm receipt)
FIN = Finish (close connection)
RST = Reset (abort connection)
PSH = Push (deliver data to application immediately)
```

### The 3-Way Handshake — Every Step Explained

Before any data can be sent, TCP establishes a connection. This takes 1 Round Trip Time (RTT).

```
CLIENT                              SERVER
  │                                   │
  │  Why sequence numbers?            │
  │  Both sides pick random starting  │
  │  sequence numbers (ISN) to        │
  │  prevent old packets from a       │
  │  previous connection being        │
  │  confused with new ones.          │
  │                                   │
  │──── SYN (seq=1000) ─────────────▶ │
  │     "I want to connect.           │
  │      My first byte will be #1001" │
  │      (ISN chosen randomly: 1000)  │
  │                                   │
  │ ◀── SYN-ACK (seq=5000, ack=1001)── │
  │     "OK! I got your SYN.          │
  │      My first byte will be #5001. │
  │      I ACK byte 1001 — I'm ready  │
  │      for your first data byte."   │
  │                                   │
  │──── ACK (seq=1001, ack=5001) ───▶ │
  │     "Got it! I'm ready for your   │
  │      first data byte too."        │
  │                                   │
  │════════ DATA TRANSFER ════════════│
```

### Sequence Numbers and Acknowledgments — How Reliability Works

```
CLIENT sends 3 segments:

Segment 1: seq=1001, len=500, data="Hello..."
Segment 2: seq=1501, len=500, data="World..."
Segment 3: seq=2001, len=500, data="How are..."

SERVER receives them and sends ACKs:
ACK 1501  → "Received up to byte 1500, send byte 1501 next"
ACK 2001  → "Received up to byte 2000, send byte 2001 next"
ACK 2501  → "Received up to byte 2500, all good"

SCENARIO: Segment 2 is LOST

CLIENT sends: Seg1, Seg2, Seg3
SERVER receives: Seg1, _____, Seg3

SERVER ACKs: ACK 1501 (got Seg1, waiting for Seg2)
             ACK 1501 (got Seg3, but can't acknowledge past Seg2!)
             ACK 1501 (still waiting...)

CLIENT sees 3 duplicate ACKs → "Fast Retransmit!"
CLIENT resends Seg2

SERVER receives Seg2 → can now acknowledge 1501, 2001, 2501 all at once
SERVER sends: ACK 2501 (cumulative ACK — "got everything up to 2500")
```

### TCP Connection Teardown (4-Way Close)

```
CLIENT (initiates close)         SERVER
  │                                │
  │──── FIN ──────────────────────▶│   "I'm done sending. Closing my side."
  │◀─── ACK ───────────────────────│   "Got your FIN."
  │                                │   (Server may still be sending data)
  │◀─── FIN ───────────────────────│   "I'm done sending too."
  │──── ACK ──────────────────────▶│   "Got your FIN."
  │                                │
  │ CLIENT waits in TIME_WAIT state│
  │ for 2×MSL (60-120 seconds)     │
  │ Why? In case the final ACK     │
  │ was lost and server retransmits│

TIME_WAIT: At high connection rates (thousands/second), TIME_WAIT sockets
           accumulate and exhaust the port space (65536 ports).
           Solution: SO_REUSEADDR, tcp_tw_reuse, or connection pooling.
```

### TCP Flow Control — Preventing Receiver Overflow

```
PROBLEM: What if sender is sending at 1Gbps but receiver can only
         process at 100Mbps? Receiver's buffer fills up → overflow → data loss.

SOLUTION: Receive Window (advertised in TCP header)

TCP header has a 16-bit "Window Size" field.
Receiver tells sender: "You can send at most X bytes without waiting for ACK"

┌─────────────────────────────────────────────────────┐
│ Receiver's buffer: [====FULL====] [    EMPTY    ]   │
│ Available space: 8KB                                │
│ Receiver sends: Window = 8192                       │
│ Sender: "OK, I'll only send 8KB max without ACK"    │
│                                                     │
│ As receiver's app reads data, buffer drains:        │
│ Receiver's buffer: [  EMPTY  ] [      EMPTY    ]   │
│ Receiver sends: Window = 32768                      │
│ Sender: "Now I can send 32KB without ACK"           │
└─────────────────────────────────────────────────────┘

Window Scaling: Original TCP has 16-bit window = max 64KB.
Too small for modern high-bandwidth links!
Solution: Window Scale option (RFC 7323) allows scaling by 2^14
= up to 1GB receive window.
```

### TCP Congestion Control — The Key Algorithms

This is where TCP gets clever — it detects network congestion and backs off.

```
PROBLEM: The internet has no central coordinator. Routers just drop packets
         when overloaded. TCP must detect this and self-regulate.

INDICATORS OF CONGESTION:
1. Packet loss (timeout or 3 duplicate ACKs)
2. Increased RTT (packets taking longer → congested routers)

ALGORITHMS:

1. SLOW START (misleading name — it's actually exponential growth!)
   ┌─────────────────────────────────────────────────────────┐
   │ Start with cwnd (congestion window) = 1 MSS (1460 bytes)│
   │ Each ACK received → cwnd += 1 MSS                       │
   │ → cwnd doubles each RTT (exponential growth)            │
   │                                                         │
   │ cwnd: 1 → 2 → 4 → 8 → 16 → ... (until ssthresh)       │
   └─────────────────────────────────────────────────────────┘

2. CONGESTION AVOIDANCE (linear growth after slow start threshold)
   ┌─────────────────────────────────────────────────────────┐
   │ Once cwnd reaches ssthresh, switch to linear growth     │
   │ Each full RTT (all ACKs received) → cwnd += 1 MSS      │
   │                                                         │
   │ cwnd: ... 16 → 17 → 18 → 19 → ... (linear)            │
   └─────────────────────────────────────────────────────────┘

3. ON PACKET LOSS:
   TCP Reno (classic):
   - Triple duplicate ACK → ssthresh = cwnd/2, cwnd = cwnd/2 (halved!)
   - Timeout → ssthresh = cwnd/2, cwnd = 1 (reset to start!)
   
   TCP CUBIC (Linux default):
   - Uses cubic function for window growth, less aggressive on fast networks
   
   TCP BBR (Google, 2016 — Linux 4.9+):
   - NOT loss-based! Models bottleneck bandwidth and RTT
   - Measures actual throughput, not packet loss as signal
   - Much better on satellite, mobile, lossy networks
   - Used by Google for YouTube, Google.com

VISUAL OF CWND OVER TIME (TCP Reno):

cwnd
 |
64 |              /           /
   |            /           /
32 |          /           /
   |        /           /
16 |      /  \ssthresh /
   |    /     \      /
 8 |   /       \   /
   |  /         \ /
 4 | /           X ← packet loss! (duplicate ACK)
   |/            |\
 1 |             | \___
   └──────────────────────────── time
   [slow start][CA]  [slow start][CA]
```

---

## 4. UDP — User Datagram Protocol — Deep Dive

### The Core Properties

UDP provides:
- Connectionless (no handshake needed)
- Unreliable (no retransmit, no ACKs)
- Unordered (packets may arrive out of order)
- No flow control
- No congestion control
- FAST (minimal overhead)
- Supports broadcast and multicast (TCP cannot)

### UDP Datagram Structure

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌─────────────────────────────┬─────────────────────────────────────┐
│         Source Port (16)    │       Destination Port (16)         │
├─────────────────────────────┼─────────────────────────────────────┤
│           Length (16)       │          Checksum (16)              │
├─────────────────────────────┴─────────────────────────────────────┤
│                         Data (variable)                           │
└───────────────────────────────────────────────────────────────────┘

That's it! 8 bytes of header total (vs 20+ bytes for TCP).
No sequence numbers, no ACK numbers, no flags, no window.
Just: source port, destination port, length, checksum, data.
```

### Why is UDP Faster?

```
TCP sending one HTTP request:
1. SYN (1 RTT for handshake)
2. SYN-ACK
3. ACK + data
4. ACK (server acknowledges data)
→ Minimum 2 RTT before data is received

UDP sending one DNS query:
1. Query packet sent immediately
2. Response received
→ 1 RTT, no setup overhead

For a 100ms RTT link:
TCP minimum latency: 200ms (before first byte of response)
UDP minimum latency: 100ms

At 10ms RTT (LAN):
TCP: 20ms handshake + data transfer
UDP: 10ms flat

For high-frequency small messages (DNS, game state updates):
This difference is dramatic.
```

---

## 5. The Key Comparison — When to Use Each

```
┌─────────────────────────┬───────────────────────────┬──────────────────────────────┐
│ Property                │ TCP                       │ UDP                          │
├─────────────────────────┼───────────────────────────┼──────────────────────────────┤
│ Connection setup        │ 3-way handshake (1 RTT)   │ None — send immediately      │
│ Delivery guarantee      │ Yes — every byte delivered│ Best-effort — may lose       │
│ Ordering                │ Yes — in-order delivery   │ No — may reorder             │
│ Duplicate prevention    │ Yes                       │ No                           │
│ Flow control            │ Yes (receive window)      │ No                           │
│ Congestion control      │ Yes (slow start, CUBIC)   │ No                           │
│ Header overhead         │ 20–60 bytes               │ 8 bytes                      │
│ Latency                 │ Higher (handshake + ACKs) │ Lower (no setup, no ACK)     │
│ Throughput (reliable)   │ High                      │ Variable (no retransmit)     │
│ Broadcast/Multicast     │ No                        │ Yes                          │
│ Use cases               │ HTTP, HTTPS, SSH, FTP,   │ DNS, DHCP, live video,        │
│                         │ SMTP, databases           │ VoIP, gaming, QUIC           │
└─────────────────────────┴───────────────────────────┴──────────────────────────────┘
```

### Decision Framework

```
Use TCP when:
├── Data must arrive completely (file download, financial transaction)
├── Data must arrive in order (HTTP response body, SSH commands)  
├── You cannot afford to implement reliability yourself
└── Correctness > low latency

Use UDP when:
├── Latency is critical (VoIP call quality degrades with retransmit delays)
├── Old data is useless (game position from 500ms ago is worthless)
├── You implement reliability at application layer (QUIC does this)
├── Small, frequent messages (DNS queries, sensor data)
└── Broadcasting/multicasting required (service discovery, IPTV)
```

---

## 6. Real-World Usage With Reasoning

### DNS (UDP port 53)

```
DNS queries are tiny (30–60 bytes) and responses are small (typically < 512 bytes).
The overhead of a TCP handshake (1 RTT) would double query latency.
If a DNS response is lost, the client just re-sends the query — simple!

UDP gives us:
- Single packet query/response (1 RTT)
- No connection state on DNS servers (millions of clients can query at once)

Exception: DNS uses TCP when:
- Response > 512 bytes (EDNS0 can extend this to 4KB, but still)
- Zone transfers between nameservers (must be complete and reliable)
```

### Live Video Streaming — YouTube Live, Twitch (WebRTC, UDP)

```
Video frames at 30fps = new frame every 33ms

If Frame #15 is lost:
TCP: Retransmit Frame #15 → Frame #16, #17, #18 all wait → video FREEZES
     (even 200ms freeze is very noticeable to users)

UDP: Skip Frame #15 → display Frame #16, #17, #18 immediately
     One slightly blurry moment, barely noticeable

For LIVE video, old data is worthless. 
A 200ms retransmit delay is far worse than one dropped frame.
```

### Voice over IP (VoIP) — Discord, Teams, WhatsApp

```
Human voice: sampled 8,000 or 16,000 times per second
Each packet: typically 10–20ms of audio (160–320 bytes)

If packet #42 is lost:
TCP: Wait for retransmit. Audio freezes for 100–300ms. 
     Conversation sounds like "Hel─ ─ ─lo? Can you hear me?"
     (RTT + retransmit delay)

UDP: Skip packet #42. Play silence or extrapolate from #41.
     "Hel-lo?" (small glitch, barely audible)
     
Packet concealment: good VoIP apps interpolate missing 20ms
from surrounding audio. Sounds better than a 300ms freeze.
```

### Online Gaming

```
Game tick rate: 64 ticks/second (CS:GO) = 15ms between ticks

Player position update every 15ms.
If position update #1000 is lost:
TCP: Wait for retransmit. Player movement freezes → "rubber banding"
     (player snaps back to old position when connection recovers)
     
UDP + custom protocol:
  - Send: position, velocity, timestamp with every packet
  - If #1000 is lost, #1001 has enough info to extrapolate
  - Client-side prediction fills the gap
  - Game feels smooth even with 5% packet loss

Games implement their own:
- Reliability (for critical events: player killed, item picked up)
- Ordering (sequence numbers)
- Congestion control (reduce update rate under load)
...all on top of UDP, customized for game requirements.
```

### QUIC — The Best of Both Worlds (HTTP/3)

```
QUIC is a protocol built on top of UDP that implements:
- Reliable delivery (like TCP)
- Stream multiplexing (like HTTP/2)
- Congestion control (like TCP)
- 0-RTT connection establishment
- Per-stream reliability (no HOL blocking!)

Why build on UDP instead of TCP?
- TCP is implemented in OS kernels — you can't change it without OS updates
- UDP is just "send packets" — you can build any logic on top in userspace
- Faster iteration: Google shipped QUIC updates without waiting for OS vendors
- No HOL blocking: QUIC stream 1's loss doesn't block stream 2

QUIC is what HTTP/3 uses. More in the HTTP/2 & HTTP/3 section.
```

---

## 7. Failure Scenarios

### SYN Flood DDoS Attack

```
ATTACK:
Attacker sends millions of SYN packets to server.
Each SYN causes server to allocate memory for "half-open" connection state.
Server sends SYN-ACK and waits. Attacker never sends ACK.
Server's connection table fills up → legitimate users can't connect.

Normal: Client ──SYN──▶ Server: allocates state ──SYN-ACK──▶ ──ACK──▶ connection
Attack: Attacker ──SYN──▶ Server: allocates state ──SYN-ACK──▶ [silence]
        (repeated millions of times → server OOM)

DEFENSE: SYN Cookies
Server doesn't allocate state on SYN.
Instead: encodes connection info in the sequence number (a cryptographic hash).
When (if) ACK arrives, server verifies hash and allocates state.
No memory wasted on half-open connections!
```

### TIME_WAIT Exhaustion

```
After closing a connection, the OS keeps it in TIME_WAIT for 60-120 seconds.
(Reason: ensure any delayed packets from old connection don't corrupt new one)

High-traffic systems (thousands of connections/second):
Thousands of ports stuck in TIME_WAIT
Eventually: can't open new connections (no free ports!)

SOLUTIONS:
1. Connection pooling — reuse connections instead of open/close rapidly
2. SO_REUSEADDR — allow reusing TIME_WAIT ports immediately
3. tcp_tw_reuse — kernel setting to allow reuse for outbound connections
4. Increase port range — default is 32768–60999, extend to 1024–65535
```

---

## 8. Interview Quick-Fire Answers

**Q: Why do we need both TCP and UDP? Why not just use TCP everywhere?**
A: TCP's reliability comes at a cost — handshake latency, retransmit delays, head-of-line blocking. For applications where low latency matters more than guaranteed delivery (voice, video, gaming), or where the application can handle loss better than TCP can (DNS), UDP is superior. The key insight: for live voice or video, a 300ms retransmit delay is FAR worse than a 20ms gap in audio/video. Old data is worthless; fresh data is everything.

**Q: Explain the TCP 3-way handshake.**
A: Client sends SYN with its starting sequence number. Server responds with SYN-ACK — acknowledging client's SYN and including its own starting sequence number. Client sends ACK acknowledging server's sequence number. After this 1 RTT, both sides have agreed on sequence numbers and the connection is established.

**Q: What is TCP head-of-line blocking?**
A: TCP guarantees in-order delivery. If one packet is lost, all subsequent packets must wait for the retransmit, even if they've already been received. On an HTTP/2 connection multiplexing 10 streams over one TCP connection, losing one packet blocks ALL 10 streams — even those that don't need the lost packet. HTTP/3 on QUIC solves this with per-stream reliability.

**Q: What is the SYN flood attack and how is it prevented?**
A: Attacker sends millions of SYN packets, each causing the server to allocate memory for a half-open connection, then never completing the handshake. Server memory exhausts. Prevention: SYN cookies — server encodes connection info cryptographically in the sequence number instead of allocating memory. Memory is only allocated when the ACK arrives with the valid cookie.

---
---

# TOPIC 4: REST vs GraphQL vs gRPC

---

## 1. What Problem Do These Solve?

Your e-commerce application has a backend that stores products, users, orders, payments. Your frontend (web, iOS, Android apps) needs to read and write this data. You need a standard way for the frontend to communicate with the backend.

Three main approaches exist, each with fundamentally different philosophies:
- **REST** — "Model your data as resources and use HTTP verbs to act on them"
- **GraphQL** — "Let the client specify exactly what data it needs"
- **gRPC** — "Define a service contract and call remote methods like local functions, with maximum performance"

---

## 2. REST — Representational State Transfer

### Core Philosophy

REST is an **architectural style** (not a protocol) defined by Roy Fielding in his 2000 PhD thesis. It's built around these constraints:

```
1. STATELESS       Every request contains all information needed to process it.
                   Server stores no session state between requests.
                   
2. RESOURCE-BASED  Everything is a "resource" identified by a URL.
                   Resources are nouns (users, orders, products).
                   HTTP verbs act on them.
                   
3. UNIFORM INTERFACE
                   Standard HTTP methods: GET, POST, PUT, PATCH, DELETE
                   Standard status codes: 200, 201, 404, etc.
                   
4. CACHEABLE       Responses should indicate whether they're cacheable.

5. LAYERED SYSTEM  Client doesn't know if it's talking to the real server
                   or a proxy/cache in front of it.
```

### REST URL Design — Beginner to Advanced

```
BASIC RESOURCE NAMING:
Bad:  /getUser, /createOrder, /deleteProduct  ← verbs in URL (wrong!)
Good: /users,   /orders,      /products       ← nouns only

COLLECTIONS AND ITEMS:
GET    /users              → List all users (collection)
POST   /users              → Create a new user
GET    /users/123          → Get specific user (item)
PUT    /users/123          → Replace user 123 completely
PATCH  /users/123          → Partially update user 123
DELETE /users/123          → Delete user 123

NESTED RESOURCES (relationships):
GET    /users/123/orders         → All orders belonging to user 123
POST   /users/123/orders         → Create order for user 123
GET    /users/123/orders/456     → Specific order 456 of user 123
DELETE /users/123/orders/456     → Cancel order 456

QUERY PARAMETERS (filtering, sorting, pagination):
GET /products?category=electronics&minPrice=1000&maxPrice=50000
GET /orders?status=pending&sortBy=createdAt&order=desc
GET /users?page=2&limit=20&search=yash
```

### REST Response Design — Best Practices

```json
GOOD ERROR RESPONSE (clear, structured):
HTTP/1.1 422 Unprocessable Entity
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Email must be a valid email address"
      },
      {
        "field": "age",
        "code": "OUT_OF_RANGE",
        "message": "Age must be between 18 and 120"
      }
    ],
    "requestId": "req_abc123xyz"
  }
}

GOOD SUCCESS RESPONSE with pagination:
HTTP/1.1 200 OK
{
  "data": [
    {"id": 1, "name": "Product A", "price": 999},
    {"id": 2, "name": "Product B", "price": 1499}
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 847,
    "totalPages": 43
  },
  "links": {
    "self":  "/products?page=1&limit=20",
    "next":  "/products?page=2&limit=20",
    "last":  "/products?page=43&limit=20"
  }
}
```

### REST's Problems at Scale — Over-fetching and Under-fetching

This is the fundamental motivation for GraphQL.

```
SCENARIO: Building a mobile app that shows a user's profile with
          their last 3 orders and the first product of each order.

OVER-FETCHING (getting too much data):

GET /users/123
Response: {
  "id": 123,
  "name": "Yash",
  "email": "yash@email.com",
  "phone": "+91...",           ← not needed on this screen
  "address": {...},             ← not needed
  "preferences": {...},         ← not needed
  "createdAt": "2023-...",      ← not needed
  "lastLogin": "2026-...",      ← not needed
  ...50 more fields...
}
Mobile app only needed: name and email. Got 50 fields. Wasted bandwidth.

UNDER-FETCHING (not enough data — need multiple requests):

To show "user profile + last 3 orders + first product of each order":

Request 1: GET /users/123         → user data
Request 2: GET /users/123/orders?limit=3  → 3 order IDs
Request 3: GET /orders/456        → first product ID
Request 4: GET /orders/789        → first product ID
Request 5: GET /orders/012        → first product ID
Request 6: GET /products/111      → product details
Request 7: GET /products/222      → product details
Request 8: GET /products/333      → product details

8 HTTP round trips! On mobile with 100ms latency = 800ms minimum.
This is called the N+1 problem.
```

---

## 3. GraphQL — Query Language for APIs

### Core Philosophy

GraphQL was created by Facebook (2012, open-sourced 2015) specifically to solve REST's over-fetching and under-fetching problems for their mobile apps. The philosophy:

"The **client** should specify exactly what it needs. The server should deliver exactly that — no more, no less."

### The Single Endpoint

```
REST:  Multiple endpoints:
       GET  /users/123
       GET  /users/123/orders
       GET  /products/111
       ...

GraphQL: Single endpoint:
         POST /graphql
         (method always POST, body contains the query)
```

### GraphQL Query — How It Works

```graphql
# Client sends this query:
query GetUserProfile {
  user(id: "123") {
    name                    ← only these fields
    email                   ← exactly these fields
    recentOrders(limit: 3) {
      id
      total
      status
      firstItem {
        name
        imageUrl
        price
      }
    }
  }
}

# Server responds with exactly this shape:
{
  "data": {
    "user": {
      "name": "Yash",
      "email": "yash@email.com",
      "recentOrders": [
        {
          "id": "ORD-456",
          "total": 4999,
          "status": "delivered",
          "firstItem": {
            "name": "Nike Air Max",
            "imageUrl": "https://...",
            "price": 4999
          }
        },
        ...2 more orders...
      ]
    }
  }
}
```

One request. Exactly the data needed. The shape of the response mirrors the shape of the query.

### GraphQL Schema — The Contract

```graphql
# The server defines its schema (the type system):

type User {
  id: ID!              # ! = non-nullable (required)
  name: String!
  email: String!
  phone: String        # nullable (optional)
  orders(limit: Int, offset: Int): [Order!]!
  createdAt: DateTime!
}

type Order {
  id: ID!
  user: User!          # relationships are declared!
  total: Float!
  status: OrderStatus!
  items: [OrderItem!]!
  createdAt: DateTime!
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

# Queries = read operations
type Query {
  user(id: ID!): User
  users(page: Int, limit: Int): [User!]!
  order(id: ID!): Order
}

# Mutations = write operations
type Mutation {
  createUser(name: String!, email: String!): User!
  updateUser(id: ID!, name: String, email: String): User!
  placeOrder(userId: ID!, items: [OrderItemInput!]!): Order!
}

# Subscriptions = real-time (WebSocket under the hood)
type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}
```

### GraphQL's Problems

```
1. N+1 QUERY PROBLEM (the biggest one!)

   Query:
   { orders(limit: 100) { user { name } } }
   
   Naive implementation:
   - Fetch 100 orders (1 DB query)
   - For each order, fetch its user (100 DB queries!)
   = 101 total DB queries!
   
   SOLUTION: DataLoader (batching + caching)
   - Collect all unique user IDs from all orders
   - Fetch them in ONE batch query
   - Cache and distribute to each order
   = 2 DB queries total

2. NO HTTP CACHING
   All GraphQL requests are POST (by convention).
   POST responses are never cached by browsers/CDNs.
   REST GET requests can be cached naturally.
   
   Workaround: Use GET for queries (append query string), POST for mutations.
   Or: Persisted queries (client pre-registers queries, server caches results).

3. OVERLY COMPLEX QUERIES (security risk!)
   Malicious client:
   {
     users {
       orders {
         user {
           orders {
             user {
               orders { ... }  ← depth 100 levels
             }
           }
         }
       }
     }
   }
   This could trigger millions of DB queries!
   
   SOLUTION: Query depth limiting, query cost analysis, query complexity scoring.

4. FILE UPLOADS
   GraphQL has no native file upload. Need multipart extensions.
   REST handles file uploads naturally with multipart/form-data.
```

### When to Use GraphQL

```
GOOD FIT:
✅ Mobile apps (save bandwidth, reduce round trips)
✅ Complex, nested data needs (social graphs, content trees)
✅ Multiple frontend clients needing different data shapes
   (web wants 20 fields, mobile wants 5 fields)
✅ Rapidly changing frontend requirements (no need for new API versions)
✅ Backend for Frontend (BFF) layer

BAD FIT:
❌ Simple CRUD APIs (overkill)
❌ File upload/download heavy use cases
❌ High-performance caching required (no HTTP cache)
❌ Non-hierarchical data
❌ Team not familiar with GraphQL tooling
```

---

## 4. gRPC — Remote Procedure Calls

### Core Philosophy

gRPC was developed by Google (2015) based on their internal RPC framework "Stubby." The philosophy:

"Services should call each other like calling local functions. Define a strict contract using Protocol Buffers. Get type-safe client libraries in any language. Get maximum performance with binary serialization."

### Protocol Buffers — The Heart of gRPC

```protobuf
// user.proto — the contract between client and server

syntax = "proto3";

package user.v1;

// The service definition
service UserService {
  // Unary RPC — one request, one response (like REST)
  rpc GetUser(GetUserRequest) returns (User);
  
  // Server streaming — one request, stream of responses
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming — stream of requests, one response
  rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateResponse);
  
  // Bidirectional streaming — stream of requests AND responses
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

// Request/Response messages (strongly typed!)
message GetUserRequest {
  string user_id = 1;  // field number (1) used in binary encoding
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  repeated string tags = 5;  // array
  google.protobuf.Timestamp created_at = 6;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
  string search = 3;
}
```

### Why Protobuf is Faster Than JSON

```
JSON (text):
{"id":"123","name":"Yash Sharma","email":"yash@example.com","age":28}
= 65 bytes
Parsing: String parsing, key matching, type coercion — slow!

Protobuf (binary):
0A 03 31 32 33 12 0C 59 61 73 68 20 53 68 61 72 6D 61 1A 10 79 61...
= ~25 bytes (60% smaller!)
Parsing: Field number lookup, direct memory copy — very fast!

HOW PROTOBUF BINARY WORKS:
Each field has a field number (in .proto file).
Wire format: field_number + wire_type + value

For string "Yash" in field 2 (name):
Binary: 0x12 (field 2, wire type 2 = length-delimited) 0x04 (length=4) 59 61 73 68 (bytes "Yash")

No field names in the binary! "name" is just "field 2".
This saves significant bandwidth and parsing time.

BENCHMARK (typical numbers):
Operation          JSON        Protobuf    Speedup
Serialization      540 ns      90 ns       6x faster
Deserialization    1,400 ns    180 ns      8x faster
Payload size       65 bytes    24 bytes    63% smaller
```

### gRPC Streaming — 4 Types

```
1. UNARY RPC (like REST request-response):
   Client ──request──▶ Server
   Client ◀──response── Server
   Use: Most API calls (get user, create order)

2. SERVER STREAMING RPC:
   Client ──request──▶ Server
   Client ◀──response─ Server (keeps sending...)
   Client ◀──response─ Server
   Client ◀──response─ Server
   Client ◀── done ─── Server
   Use: Large data downloads, real-time feeds (stock prices),
        progress updates for long-running jobs

3. CLIENT STREAMING RPC:
   Client ──data────▶ Server
   Client ──data────▶ Server
   Client ──data────▶ Server
   Client ──done────▶ Server
   Client ◀──result── Server
   Use: File uploads, bulk inserts, log streaming to server

4. BIDIRECTIONAL STREAMING RPC:
   Client ──data────▶ Server ──data────▶ Client (both directions simultaneously!)
   Client ◀──data─── Server
   Client ──data────▶ Server ◀──data─── Client
   Use: Real-time chat, collaborative editing, game state sync
```

### gRPC in Practice — Code Generation

```bash
# From one .proto file, generate client/server code in any language:

protoc --go_out=. --go-grpc_out=. user.proto        # Go
protoc --java_out=. user.proto                        # Java
protoc --python_out=. user.proto                      # Python
protoc --js_out=. user.proto                          # JavaScript

# Generated Go client — type-safe, no manual HTTP/JSON:
conn, _ := grpc.Dial("userservice:50051", grpc.WithInsecure())
client := pb.NewUserServiceClient(conn)

user, err := client.GetUser(ctx, &pb.GetUserRequest{UserId: "123"})
// user is a strongly-typed *pb.User struct — compile-time type safety!
```

---

## 5. The Big Three-Way Comparison

```
┌─────────────────────┬─────────────────────────┬─────────────────────────┬─────────────────────────┐
│ Feature             │ REST                    │ GraphQL                 │ gRPC                    │
├─────────────────────┼─────────────────────────┼─────────────────────────┼─────────────────────────┤
│ Protocol            │ HTTP/1.1, HTTP/2         │ HTTP/1.1, HTTP/2        │ HTTP/2 (required)       │
│ Data format         │ JSON/XML (text)          │ JSON (text)             │ Protobuf (binary)       │
│ Schema/Contract     │ Optional (OpenAPI)       │ Built-in (SDL)          │ Mandatory (proto files) │
│ Type safety         │ None native              │ Yes (schema types)      │ Yes (proto types)       │
│ Code generation     │ Optional (OpenAPI gen)  │ Optional                │ Mandatory               │
│ HTTP caching        │ Excellent (GET cached)   │ Poor (all POST)         │ Not applicable          │
│ Browser support     │ Native                  │ Native                  │ Needs grpc-web proxy    │
│ Streaming           │ SSE (server only)        │ WebSocket subscriptions │ Native (4 types)        │
│ Performance         │ Medium                  │ Medium                  │ High (binary, HTTP/2)   │
│ Learning curve      │ Low                     │ Medium                  │ High                    │
│ Debugging           │ Easy (curl, browser)     │ Medium (GraphiQL)       │ Hard (binary)           │
│ Versioning          │ URL (/v1/, /v2/)         │ Schema evolution        │ Field numbers           │
│ Best for            │ Public APIs, CRUD, web  │ Mobile BFF, complex UIs │ Internal microservices  │
│ Companies using     │ Stripe, Twitter, AWS     │ GitHub, Facebook, Shopify│ Google, Uber, Netflix  │
└─────────────────────┴─────────────────────────┴─────────────────────────┴─────────────────────────┘
```

---

## 6. The Decision Framework

```
DESIGNING AN API — DECISION TREE:

Who will consume it?
├── External developers (public API)?
│   └── Use REST
│       - Developer familiarity
│       - Easy to test with curl/Postman
│       - HTTP caching works
│       - OpenAPI/Swagger documentation
│       Examples: Stripe, Twilio, GitHub v3
│
└── Internal services / microservices?
    ├── Performance critical (high RPS, low latency)?
    │   └── Use gRPC
    │       - Binary protobuf: 5-10x smaller, 6-8x faster parsing
    │       - HTTP/2 multiplexing
    │       - Streaming support
    │       Examples: Google internal, Uber microservices
    │
    └── Multiple frontend clients with different data needs?
        ├── YES → Use GraphQL (or GraphQL Federation)
        │         - Mobile gets exactly what it needs
        │         - Web gets a different shape
        │         - BFF pattern
        │         Examples: GitHub v4, Facebook, Netflix
        │
        └── NO → Use REST (simpler) or gRPC (faster)
```

---

## 7. Real-World Usage — How Companies Actually Do It

### Netflix — GraphQL Federation

```
Netflix has 500+ microservices. If a mobile app wants to show a
movie with its ratings, cast, and recommendations, it needs to
call MovieService, RatingService, CastService, RecommendationService.

Without federation:
Mobile → MovieService (1 call)
Mobile → RatingService (1 call)  
Mobile → CastService (1 call)
Mobile → RecommendationService (1 call)
= 4 network round trips, mobile coordinates everything

With GraphQL Federation (Apollo):
Each service owns its portion of the graph:
  MovieService defines: type Movie { id, title, year }
  RatingService extends: Movie { rating, reviewCount }
  CastService extends: Movie { cast: [Actor] }

A gateway stitches these into ONE unified graph.
Mobile → Gateway (1 GraphQL query, gets everything)
Gateway → orchestrates service calls internally

Mobile makes 1 request. Gets all data. Services stay independent.
```

### Uber — gRPC Internally, REST Externally

```
Rider app → REST API Gateway → Uber backend (gRPC mesh)

External API (REST):
POST /requests (request a ride)
GET  /requests/current
GET  /products (available ride types)

Internal services communicate via gRPC:
DispatchService.FindNearbyDrivers(location, radius)
→ DriverLocationService.GetDriverLocations([driverIds])
→ PricingService.CalculatePrice(origin, destination, rideType)
→ ETAService.EstimateArrival(driverLocation, riderLocation)

gRPC is used internally because:
- Polyglot (services in Go, Java, Python all use same proto definitions)
- Performance (thousands of inter-service calls per second)
- Streaming (real-time driver location updates)
```

---

## 8. Schema Versioning — How to Evolve Your API

### REST Versioning Strategies

```
URL versioning (most common):
GET /api/v1/users  → old behavior
GET /api/v2/users  → new behavior (breaking changes in v2)

Header versioning:
GET /api/users
API-Version: 2024-01-01   ← Stripe uses this pattern

Query parameter:
GET /api/users?version=2

PROBLEM WITH URL VERSIONING:
- Clients don't upgrade automatically
- You end up maintaining v1, v2, v3 all simultaneously forever
- Deprecation is hard (enterprise clients stuck on v1 for years)
```

### Protobuf Schema Evolution (gRPC's Approach)

```
// v1 proto:
message User {
  string id = 1;
  string name = 2;
}

// v2 proto — add a new field:
message User {
  string id = 1;
  string name = 2;
  string email = 3;    // NEW — old clients ignore unknown fields!
}

RULES for backward compatibility:
✅ ADD new fields (old clients ignore them, new clients use them)
✅ RENAME fields (field numbers are what matter in binary, not names)
✅ REMOVE optional fields (replace with placeholder to preserve field number)

❌ CHANGE field type of existing field number
❌ REUSE field numbers (old clients may misinterpret data)
❌ Change required fields to optional (proto3 has no required, less issue)

This "field number" system means you rarely need to version gRPC APIs!
```

---

## 9. Interview Quick-Fire Answers

**Q: What is REST and what are its constraints?**
A: REST is an architectural style for building APIs using HTTP. Its key constraints are: stateless (no server-side session), resource-based (nouns in URLs), uniform interface (standard HTTP verbs and status codes), cacheable responses, and layered system (client doesn't know about intermediaries).

**Q: What is the N+1 problem in GraphQL?**
A: When fetching a list of N items, each requiring a sub-query, the naive implementation fires N+1 database queries (1 for the list + N for sub-resources). Solved using DataLoader, which batches all N sub-queries into one query and caches results within a request.

**Q: Why does Netflix use GraphQL Federation?**
A: Netflix has 500+ microservices. Without federation, mobile clients would need multiple HTTP round trips to assemble data from different services. GraphQL Federation lets each service define its own portion of the graph; a gateway stitches them together, so clients make one request and get exactly the data they need.

**Q: Why is gRPC faster than REST?**
A: Three reasons. First, Protobuf binary serialization is 5-10x smaller and 6-8x faster to parse than JSON. Second, gRPC uses HTTP/2 which multiplexes multiple calls on one TCP connection — no connection setup overhead for each call. Third, strong typing via proto files allows code generation — no runtime type checking needed.

**Q: When would you choose REST over GraphQL?**
A: REST when: building a public API (developer familiarity, easy curl testing, HTTP caching for GET requests), simple CRUD with straightforward data shapes, teams unfamiliar with GraphQL tooling, or when HTTP caching is critical (CDN caching REST GET responses is very effective). GraphQL when: multiple clients needing different data shapes, mobile apps needing to minimize bandwidth, complex nested data requirements.


---
---

# TOPIC 5: WebSockets

---

## 1. What Problem Does WebSocket Solve?

HTTP is a request-response protocol — the **client always speaks first**. The server can only respond; it cannot initiate communication.

This creates a fundamental problem for real-time applications:

```
REAL-TIME SCENARIOS where server needs to PUSH data to client:
- Chat app: "New message from Priya"
- Stock trading: "INFY just dropped ₹50"
- Live sports: "GOAL! India scores in 88th minute"
- Multiplayer game: "Enemy player moved to position (x=423, y=891)"
- Collaborative doc: "Rahul is typing in paragraph 3"
- Ride-sharing: "Your driver is 2 minutes away"

PROBLEM WITH HTTP:
Server cannot send data unless client asks first.
To get "real-time" updates, clients had to POLL:

Client: "Any new messages?" → Server: "No"  (every 1 second)
Client: "Any new messages?" → Server: "No"  (wasted request)
Client: "Any new messages?" → Server: "No"  (wasted request)
Client: "Any new messages?" → Server: "Yes! Here's 1 message"

This is inefficient:
- Most polls return empty responses (wasted bandwidth)
- High server load (1000 users × 1 request/second = 1000 req/sec for nothing)
- Latency = polling interval (1s poll = up to 1s delay for new messages)
```

WebSocket solves this by upgrading a standard HTTP connection into a **persistent, full-duplex, bidirectional channel** — both sides can send at any time.

---

## 2. Core Intuition

```
HTTP (walkie-talkie):
  You press button → speak → release → wait for reply
  Only one person can speak at a time
  You must initiate every exchange

WebSocket (phone call):
  Connection is established ONCE
  Either side can speak at any moment
  Both sides can speak simultaneously
  Connection persists until someone hangs up
```

---

## 3. How WebSocket Works — The Upgrade Process

WebSocket is not a separate protocol from scratch — it **starts as HTTP** and then upgrades. This is clever: it works on port 80/443, passes through firewalls and proxies (which understand HTTP), and then switches to a lightweight frame-based protocol.

### Step 1: HTTP Upgrade Handshake

```
CLIENT sends a regular HTTP request with special upgrade headers:

GET /chat HTTP/1.1
Host: chat.company.com
Connection: Upgrade              ← "I want to change protocols"
Upgrade: websocket               ← "Specifically, to WebSocket"
Sec-WebSocket-Version: 13        ← WebSocket protocol version
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
                   └── Random base64-encoded 16-byte value
                       (prevents caching proxies from replaying responses)
Origin: https://app.company.com  ← Security check on server side

SERVER responds if it supports WebSocket:

HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
                      └── SHA-1(client_key + GUID) base64 encoded
                          Proves server understood the WS handshake
                          (not a security mechanism — just protocol check)
```

### Step 2: Connection is Now WebSocket

```
After the 101 response, the TCP connection is "hijacked":
- No more HTTP headers on this connection
- The connection is now a raw TCP stream
- Both sides communicate using WebSocket FRAMES (not HTTP messages)
- Connection stays open indefinitely (until one side closes it)

TIMELINE:
┌──────────────────────────────────────────────────────────────────┐
│ 0ms:    Client sends HTTP GET with Upgrade header                │
│ 50ms:   Server responds 101 Switching Protocols                  │
│ 50ms:   Connection is now WebSocket                              │
│ 51ms:   Server sends: {"type":"welcome","userId":"abc"}          │
│ 200ms:  Client sends: {"type":"join","room":"general"}           │
│ 205ms:  Server sends: {"type":"message","text":"Hello Yash!"}    │
│ ...                                                              │
│ ∞:      Connection stays open (days, weeks even)                 │
│ ...                                                              │
│ N:      Either side sends Close frame → TCP connection closes    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. WebSocket Frame Structure — Deep Dive

Once the connection is established, data is sent in **frames** — very lightweight binary structures.

```
WebSocket Frame Layout:

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌─┬───┬─┬───────────┬─┬─────────────┬───────────────────────────┐
│F│R R│R│  OPCODE   │M│ PAYLOAD LEN │  Extended Payload Length   │
│I│S S│S│  (4 bits) │A│   (7 bits)  │     (if needed)           │
│N│V V│V│           │S│             │                           │
│ │1 2│3│           │K│             │                           │
└─┴───┴─┴───────────┴─┴─────────────┴───────────────────────────┘
│                  Masking Key (32 bits, if MASK=1)               │
├───────────────────────────────────────────────────────────────────┤
│                       Payload Data                               │
└───────────────────────────────────────────────────────────────────┘

FIELD EXPLANATIONS:

FIN (1 bit):   1 = this is the final (or only) fragment of a message
               0 = more fragments coming (for large messages split across frames)

RSV1-3 (3 bits): Reserved for extensions (e.g., per-message compression)

OPCODE (4 bits): What kind of frame is this?
  0x0 = Continuation frame (fragment of previous message)
  0x1 = Text frame (UTF-8 text data)
  0x2 = Binary frame (raw binary data)
  0x8 = Connection close
  0x9 = Ping (keepalive check — "are you still there?")
  0xA = Pong (keepalive response — "yes I'm here!")

MASK (1 bit):  Client → Server: MUST be 1 (data must be masked)
               Server → Client: MUST be 0 (server never masks)
               WHY MASKING? Prevents cache poisoning attacks on HTTP proxies
               that might confuse WebSocket frames with HTTP responses.

PAYLOAD LEN:   0-125: actual length
               126: next 2 bytes are the real length (up to 65,535 bytes)
               127: next 8 bytes are the real length (up to 2^63 bytes)

MASKING KEY (32 bits): If MASK=1, a random 4-byte XOR key
               Each byte of payload is XORed with masking_key[i % 4]
               Simple XOR — not encryption, just obfuscation for proxies
```

### Overhead Comparison

```
HTTP/1.1 request (sending "Hello"):
GET /send HTTP/1.1\r\n
Host: api.chat.com\r\n
Content-Type: application/json\r\n
Authorization: Bearer eyJhbGci...\r\n
Content-Length: 18\r\n
\r\n
{"message":"Hello"}
= ~200-400 bytes of overhead for 5 bytes of data!

WebSocket frame (sending "Hello"):
[1 byte header][1 byte length][4 bytes mask][5 bytes data]
= 11 bytes overhead for 5 bytes of data!
= ~20x less overhead than HTTP

This matters enormously for high-frequency messages (games, trading).
```

---

## 5. Heartbeat — Keeping Connections Alive

This is a critical production concern that beginners often miss.

```
PROBLEM: "Phantom connections"

Network infrastructure (NAT devices, load balancers, firewalls) 
automatically close idle TCP connections after 30-300 seconds.

Your WebSocket connection might appear "open" in your code,
but the underlying TCP connection was silently closed by a 
NAT device or load balancer after 60 seconds of inactivity.

SYMPTOMS:
- Users don't receive messages after being idle
- Connection appears open but messages are dropped silently
- Reconnect storms when many users' connections expire simultaneously

SOLUTION: Ping/Pong frames (WebSocket RFC 6455 §5.5.2)

SERVER sends Ping frame every 20-30 seconds:
Server → Client: [Ping frame, payload: "heartbeat-1234"]

Client MUST respond with Pong:
Client → Server: [Pong frame, same payload echoed back]

If server doesn't receive Pong within timeout (e.g., 10 seconds):
→ Client is dead or disconnected
→ Server closes connection and cleans up resources

IMPLEMENTATION PATTERN:
                                                     
Every 25 seconds:                                    
Server → Ping ────────────────────────────▶ Client   
         ◀─── Pong ─────────────────────────         
         (connection confirmed alive)                
                                                     
After 10 seconds with no Pong:                       
Server: "Client is gone, closing socket"             
Server → Remove from room member lists               
Server → Notify other users "User went offline"      
```

---

## 6. WebSocket vs Alternatives — Complete Comparison

This is crucial for interviews. You need to know when NOT to use WebSocket.

```
OPTION 1: SHORT POLLING
How: Client sends HTTP request every N seconds

Timeline:
t=0s:   Client → "Any messages?"  Server → "No"
t=1s:   Client → "Any messages?"  Server → "No"
t=2s:   Client → "Any messages?"  Server → "No"
t=2.3s: New message arrives at server
t=3s:   Client → "Any messages?"  Server → "Yes! Here it is"

Latency:    Up to N seconds (polling interval)
Overhead:   Very high (mostly empty responses)
Server load: High (constant requests even with no data)
Use when:   Checking status every few minutes (email, background jobs)
            Very simple implementation needed
            Low frequency updates (>30 seconds between updates)


OPTION 2: LONG POLLING
How: Client sends HTTP request, server holds it open until data arrives

Timeline:
t=0s:   Client → "Any messages?" (server holds connection open)
t=2.3s: New message arrives
t=2.3s: Server → "Yes! Here it is" (responds immediately)
t=2.4s: Client → "Any messages?" (immediately asks again)

Latency:    Near real-time (server responds the moment data arrives)
Overhead:   Moderate (connections held open, but fewer empty responses)
Server load: Moderate (many open connections consuming file descriptors)
Use when:   Simple real-time updates, legacy browser support needed
            One-directional push (server→client) infrequently


OPTION 3: SERVER-SENT EVENTS (SSE)
How: One-way HTTP streaming from server to client

Client opens one persistent HTTP connection:
GET /events HTTP/1.1
Accept: text/event-stream

Server keeps the HTTP response body open and streams events:
data: {"type":"message","text":"Hello"}\n\n
data: {"type":"typing","user":"Priya"}\n\n
data: {"type":"message","text":"How are you?"}\n\n

Latency:    Real-time (server pushes immediately)
Direction:  Server → Client ONLY (client can't send via same connection)
Browser:    Automatic reconnect built in! (EventSource API)
HTTP/2:     Can multiplex many SSE streams on one connection
Use when:   Live dashboards, news feeds, notifications
            Server pushes, client rarely sends back
            Simple implementation needed


OPTION 4: WEBSOCKET
How: Full-duplex persistent connection

Direction:  BOTH ways simultaneously
Latency:    Real-time
Overhead:   Minimal (8-14 byte frame headers)
Complexity: Higher (connection management, scaling challenges)
Use when:   Both sides need to send frequently
            Very low latency critical (gaming, trading, chat)
            High message frequency (>10 messages/second per user)
```

### Decision Table

```
┌─────────────────────────┬──────────┬──────────┬──────────┬────────────┐
│ Feature                 │ Polling  │ Long Poll│   SSE    │ WebSocket  │
├─────────────────────────┼──────────┼──────────┼──────────┼────────────┤
│ Direction               │ C→S      │ C→S      │ S→C      │ Both       │
│ Real-time latency       │ ❌ No    │ ✅ Yes   │ ✅ Yes   │ ✅ Yes     │
│ HTTP caching friendly   │ ✅       │ ❌       │ ❌       │ ❌         │
│ Auto-reconnect          │ ✅       │ Manual   │ ✅ Built-in│ Manual    │
│ Works behind all proxies│ ✅       │ ✅       │ ✅       │ ⚠️ Most    │
│ Server push             │ ❌       │ ✅       │ ✅       │ ✅         │
│ Client push (frequent)  │ Via POST │ Via POST │ Via POST │ ✅ Native  │
│ Bandwidth efficiency    │ ❌ Poor  │ ⚠️ OK   │ ✅ Good  │ ✅ Best    │
│ Implementation simplicity│ ✅ Easy │ ⚠️ Med  │ ✅ Easy  │ ❌ Hard    │
│ Horizontal scaling      │ ✅ Easy  │ ✅ Easy  │ ✅ Easy  │ ❌ Hard    │
└─────────────────────────┴──────────┴──────────┴──────────┴────────────┘
```

---

## 7. Scaling WebSockets — The Hard Problem

This is the most important interview topic for WebSocket. It's where most candidates struggle.

### The Core Scaling Problem

```
SCENARIO: 100,000 concurrent users in a chat app

NAIVE APPROACH (WRONG):

Load Balancer
    │
    ├── WS Server 1 (users: Alice, Bob, Carol)
    ├── WS Server 2 (users: Dave, Eve, Frank)
    └── WS Server 3 (users: George, Hannah, Ivan)

Alice (on Server 1) sends message to Dave (on Server 2).
Server 1 has Alice's socket.
Server 1 does NOT have Dave's socket.
Server 1 CANNOT push to Dave!

This is the fundamental challenge: WebSocket connections are STATEFUL.
Unlike REST where any server can handle any request,
WebSocket connections are pinned to specific servers.
```

### Solution 1: Pub/Sub Message Broker (Industry Standard)

```
ARCHITECTURE WITH REDIS PUB/SUB:

              Alice                          Dave
                │                             │
                ▼                             ▼
           WS Server 1                   WS Server 2
                │                             │
                │── SUBSCRIBE room:general ───┘ (both servers subscribe to channel)
                │                             │
                ▼                             ▼
        ┌───────────────────────────────────────────┐
        │         Redis Pub/Sub Broker               │
        │                                           │
        │  Channel: room:general                    │
        │  Channel: room:tech                       │
        │  Channel: user:dave                       │
        └───────────────────────────────────────────┘

Alice sends "Hello Dave!" to Server 1:

Step 1: Server 1 receives message from Alice's socket
Step 2: Server 1 publishes to Redis: PUBLISH room:general {"from":"Alice","text":"Hello Dave!"}
Step 3: Redis delivers to ALL subscribers of room:general
Step 4: Server 2 receives from Redis (it subscribed to room:general)
Step 5: Server 2 finds Dave's socket in its local connection map
Step 6: Server 2 pushes message to Dave's WebSocket connection

RESULT: Any server can handle any message. Servers are stateless in terms of logic.
State (which user is in which room) lives in Redis, not the servers.
```

### Solution 2: Sticky Sessions (Simpler but Limited)

```
Load Balancer with sticky sessions (affinity):
- Hash user_id → always route to same server
- Alice's user_id → always goes to Server 1
- Dave's user_id → always goes to Server 2

PROBLEM: Server 1 goes down → Alice's session is lost
         Server 1 has 10,000 users → Server 2 has 1,000 → uneven load
         Hard to scale up/down without re-routing all connections

USE WHEN: Small scale, simple architecture, Redis not available
```

### Solution 3: Dedicated Connection Gateway (Netflix, Slack pattern)

```
ARCHITECTURE:

Mobile/Web Clients
    │
    ▼
WebSocket Gateway Layer (stateless connection handler)
    │  Holds raw WS connections
    │  Routes messages to/from message broker
    │
    ▼
Kafka / Redis Pub-Sub / RabbitMQ (message broker)
    │  Fan-out messages to correct gateway instances
    │
    ▼
Business Logic Services (stateless REST/gRPC microservices)
    │  Process messages, update databases
    │  Publish responses to broker

BENEFITS:
- Gateway layer scales independently (just add more instances)
- Business logic is standard HTTP/gRPC services (easy to scale)
- Message broker handles delivery guarantees
- Failures in business logic don't drop connections

USED BY: Slack (RTM API), Discord (Gateway API), PubNub
```

---

## 8. Connection Management at Scale — Production Details

### The C10K Problem

```
"C10K" = "10,000 concurrent connections" — a famous systems challenge.

HISTORICAL PROBLEM (pre-2000s):
Traditional servers: one thread or process per connection
10,000 connections = 10,000 threads
Each thread: 1-8MB stack = 10-80GB RAM just for stacks!
Context switching 10,000 threads = huge CPU overhead

MODERN SOLUTION: Event-loop / non-blocking I/O

Node.js: Single thread, event loop handles all connections
Go: Goroutines (lightweight threads, ~2KB each) → millions of connections
Java: Netty (NIO event loop) → millions of connections
Nginx: Event-loop based → million+ connections per instance

REAL NUMBERS:
- Discord: 5 million concurrent WebSocket connections per machine (Go)
- Slack: ~million connections per Gateway instance (Java/Netty)
- WhatsApp: 2 billion users, millions of concurrent connections (Erlang)
```

---

## 9. Real-World Usage — Deep Dive

### Slack — WebSocket Architecture

```
Each Slack client maintains ONE persistent WebSocket connection.
This connection carries: messages, typing indicators, presence updates,
channel membership changes, file uploads progress, app events.

Slack's approach:
- WebSocket Gateway servers: ~100 instances, each holding ~100K connections
- Redis Pub/Sub for fanout between gateway instances
- Each workspace maps to a set of gateway servers
- Message delivery guaranteed: ACK system on top of WebSocket
  (client ACKs each message; server retransmits if no ACK in 5s)

Scale: ~10M daily active users → peaks ~2M concurrent connections
```

### Discord — Sharding WebSocket at Scale

```
Discord has servers (guilds) with up to 250,000 members.
One guild change event → notify 250,000 WebSocket connections!

SHARDING:
Discord splits connections across "shards" (independent gateway clusters)
Each shard handles connections for a range of guild_ids
guild_id % total_shards = shard_number

Large bot with 100,000 guilds:
- Uses 50 shards (2,000 guilds per shard)
- Each shard = separate WebSocket connection

Discord's scale:
- 1B+ events/day
- ~19M concurrent users at peak
- ~15 million WebSocket connections per DC
- Written in Elixir (Actor model → perfect for millions of lightweight processes)
```

### Binance — Financial WebSocket Streams

```
Binance operates live market data streams via WebSocket:

wss://stream.binance.com:9443/stream

Streams available:
- Trade stream: every trade as it happens
- Depth stream: full order book updates (100ms)
- Ticker stream: 24hr rolling window statistics
- Candlestick stream: OHLCV data at 1m, 5m, 1h, 1d

Scale: millions of connected clients
Key insight: data is PUSHED to clients as fast as possible
REST API used for: placing orders, getting account info (infrequent)
WebSocket used for: market data (continuous, high frequency)

Performance requirement: < 5ms from trade execution to client notification
```

---

## 10. Failure Scenarios and Mitigations

```
┌─────────────────────────┬─────────────────────────────┬────────────────────────────────┐
│ Failure                 │ Root Cause                  │ Mitigation                     │
├─────────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Silent connection death │ NAT/firewall closes idle TCP│ Ping/pong heartbeat every 25s  │
│ (messages disappear)    │ connection without notifying│ Client: reconnect on error     │
├─────────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Server crash loses all  │ No message persistence      │ Message queue (Redis/Kafka)    │
│ in-flight messages      │ In-memory connection state  │ Client sequence numbers + ACK  │
├─────────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Reconnect storm         │ Server restart → all clients│ Exponential backoff + jitter   │
│ (thundering herd)       │ try to reconnect at once    │ on client reconnection         │
│                         │ → overwhelms server         │ Randomize reconnect delay      │
├─────────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Memory leak             │ Connections not cleaned up  │ Track connections in registry  │
│ (server OOM over time)  │ on disconnect               │ Heartbeat to detect dead conns │
│                         │                             │ Periodic connection audit      │
├─────────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Load balancer 502 on    │ LB doesn't support HTTP     │ Configure nginx proxy_read_    │
│ WebSocket upgrade       │ Upgrade / short timeout     │ timeout 3600s; use sticky LB   │
├─────────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Cross-shard message     │ User A on Server 1 messages │ Redis Pub/Sub fanout           │
│ delivery failure        │ User B on Server 2          │ Persistent message queue       │
└─────────────────────────┴─────────────────────────────┴────────────────────────────────┘
```

---

## 11. Interview Quick-Fire Answers

**Q: What is the WebSocket handshake?**
A: WebSocket starts as a standard HTTP GET request with two special headers: `Connection: Upgrade` and `Upgrade: websocket`. The server responds with `101 Switching Protocols`. After this, the TCP connection is repurposed as a WebSocket channel — no more HTTP framing, just lightweight WebSocket frames in both directions.

**Q: How do you scale WebSocket servers horizontally?**
A: The core problem is that WebSocket connections are stateful — a message for User B (on Server 2) might arrive at Server 1 (where User A is connected). Solution: use a pub/sub message broker like Redis. Each WS server subscribes to relevant channels. When Server 1 gets a message for a room, it publishes to Redis; all servers subscribed to that room receive it and push to their local connections. Servers become stateless routers.

**Q: When would you use SSE instead of WebSocket?**
A: SSE (Server-Sent Events) when: communication is one-directional (server pushes to client), you need automatic reconnection (built into EventSource API), you want to stay on standard HTTP (works easily with HTTP/2 multiplexing and all proxies), or simplicity is important. WebSocket when: both client and server need to send frequently (chat, gaming, collaborative editing), or very high message frequency where HTTP overhead of SSE becomes noticeable.

**Q: What is the heartbeat/ping-pong mechanism in WebSocket?**
A: NAT devices and load balancers close idle TCP connections after 30-120 seconds. Without heartbeats, a WebSocket connection that hasn't exchanged data appears dead to the network and gets silently closed. WebSocket defines Ping/Pong control frames. The server sends a Ping every 20-30 seconds; the client responds with a Pong. If no Pong arrives within a timeout, the server closes the connection and cleans up resources.

---
---

# TOPIC 6: HTTP/2 and HTTP/3

---

## 1. Why Were HTTP/2 and HTTP/3 Created?

### HTTP/1.1 — The Original Problems

HTTP/1.1 (1997) was designed for a simpler web. The average 1997 webpage: ~10KB, 5 resources. A 2024 webpage: ~2.5MB, 100+ resources (JS bundles, images, fonts, API calls).

HTTP/1.1 has two fundamental problems at this scale:

```
PROBLEM 1: HEAD-OF-LINE BLOCKING (HOL Blocking)

HTTP/1.1 processes ONE request per connection at a time.
Request 2 must WAIT for Request 1's response to fully arrive.

Timeline (fetching 3 resources on 100ms RTT connection):
                                                        
Conn 1: [──── GET /index.html ──────][response 50ms]
                                            [── GET /style.css ──][response 20ms]
                                                                         [GET /app.js][response 80ms]
Total: 50 + 20 + 80 = 150ms (purely sequential)

BROWSER WORKAROUND: Open 6 parallel connections to same domain!
But this is wasteful (6 TCP handshakes, 6 TLS handshakes, OS resource usage).

PROBLEM 2: HEADER OVERHEAD

HTTP/1.1 headers are plain text, uncompressed, repeated on EVERY request.
A typical request has 400-800 bytes of headers.
For 100 resources: 100 × 600 bytes = 60,000 bytes of pure header overhead!

Typical repeated headers on every request to same server:
Host: api.company.com                 (same every time!)
Cookie: session=abc123; pref=...       (same every time! often 500+ bytes)
User-Agent: Mozilla/5.0 (Mac...)       (same every time!)
Accept-Encoding: gzip, deflate, br     (same every time!)
Authorization: Bearer eyJhbGci...      (same every time!)

60KB just to say "give me these files" when the actual files are maybe 500KB total.
12% overhead just in headers!
```

---

## 2. HTTP/2 — Complete Deep Dive

HTTP/2 was standardized in 2015 (RFC 7540) by Google (based on their SPDY protocol). It addresses both HTTP/1.1 problems without changing HTTP semantics — same methods, headers, status codes, URLs. Only the "wire format" changes.

### Feature 1: Binary Framing Layer

```
HTTP/1.1: Text-based messages
GET /index.html HTTP/1.1\r\n
Host: example.com\r\n
\r\n

HTTP/2: Binary frames
┌────────────────────────────────────────────────────┐
│ Length (24 bits) │ Type (8 bits) │ Flags (8 bits)  │
├────────────────────────────────────────────────────┤
│ Reserved (1 bit) │   Stream ID (31 bits)            │
├────────────────────────────────────────────────────┤
│                  Payload                            │
└────────────────────────────────────────────────────┘

Frame Types:
HEADERS (0x1)  - HTTP headers (compressed with HPACK)
DATA    (0x0)  - Request/response body
SETTINGS(0x4)  - Configuration parameters
PING    (0x6)  - Keepalive
RST_STREAM(0x3)- Cancel a specific stream
WINDOW_UPDATE  - Flow control
PUSH_PROMISE   - Server push announcement

BENEFITS of binary:
✅ More compact (no text parsing, no \r\n delimiters)
✅ Less error-prone (no ambiguity in parsing)
✅ Easier to multiplex (each frame has stream ID)
✅ Faster to parse (bitmask operations vs string parsing)
```

### Feature 2: Multiplexing — The Big Win

This is the key feature. Multiple HTTP requests/responses are interleaved on a SINGLE TCP connection, each tagged with a Stream ID.

```
HTTP/1.1 (6 connections, sequential on each):
                                                        
Conn1: [GET /html....response....][GET /css...response...]
Conn2: [GET /js1....response....][GET /js2...response...]
Conn3: [GET /img1...response....][GET /img2..response...]
Conn4: [GET /font...response....][GET /api...response...]
Conn5: [GET /icon...response....][...                  ]
Conn6: [GET /bg.....response....][...                  ]

Problems: 6 TCP handshakes, 6 TLS handshakes, OS resources for 6 sockets


HTTP/2 (1 connection, all multiplexed):
                                                        
Stream 1: [─ GET /html ─][───────── response data ─────────]
Stream 3: [─ GET /css  ─][── response ──]
Stream 5: [─ GET /js1  ─][──────── response data ──────────]
Stream 7: [─ GET /img1 ─][── response ──]
Stream 9: [─ GET /font ─][─────────── response data ───────]
         All happening simultaneously on ONE TCP connection!

On the wire (interleaved frames):
[S1:HEADERS][S3:HEADERS][S5:HEADERS][S7:HEADERS]
[S3:DATA chunk1][S1:DATA chunk1][S7:DATA full][S5:DATA chunk1]
[S3:DATA last][S1:DATA chunk2][S5:DATA chunk2]
[S9:HEADERS][S9:DATA chunk1][S1:DATA last][S5:DATA last]
[S9:DATA last]

RESULT:
- 1 TCP connection (1 handshake, 1 TLS setup)
- All resources fetched concurrently
- No head-of-line blocking at HTTP layer
- Typical improvement: 30-50% faster page loads
```

### Feature 3: HPACK Header Compression

```
PROBLEM: Same headers sent with every request

REQUEST 1 (first request):
Headers sent: 872 bytes
:method: GET
:path: /index.html
:scheme: https
:authority: www.example.com
accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
accept-encoding: gzip, deflate, br
accept-language: en-US,en;q=0.9
cookie: _ga=GA1.2.123456.789; session=abcdef123456...
user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36

REQUEST 2 (fetching /style.css, same host):
Without compression: still 872 bytes (all headers repeated!)
With HPACK: only ~50 bytes — just ":path: /style.css" as a delta!

HOW HPACK WORKS:
1. Static Table (61 pre-defined common header/value pairs)
   Index 2  = :method: GET
   Index 4  = :path: /
   Index 6  = :scheme: https
   Index 24 = cache-control: no-cache

2. Dynamic Table (headers seen in this connection, added as sent)
   Index 62 = :authority: www.example.com  (added after first request)
   Index 63 = user-agent: Mozilla/5.0...   (added after first request)

3. Huffman Encoding (compresses string values)
   "www.example.com" → binary Huffman code → ~60% smaller

RESULT:
First request:  Headers compressed from 872 → 300 bytes (65% smaller)
Second request: Headers compressed to ~50 bytes (94% smaller!)
                Only send the diff: new path, changed headers

SAVINGS: 50-90% header size reduction on typical websites
```

### Feature 4: Stream Prioritization

```
When a browser fetches 100 resources simultaneously, some matter more:
CSS (render-blocking) > HTML > above-fold images > JS > below-fold images > analytics

HTTP/2 allows assigning priorities to streams:
Stream 1 (CSS):          weight=256, exclusive dependency on Stream 0
Stream 3 (HTML):         weight=220, depends on Stream 1
Stream 5 (above-fold img):weight=183, depends on Stream 3
Stream 7 (JS bundles):   weight=147
Stream 9 (other images): weight=20

Server should allocate bandwidth proportionally to weights.
Critical path resources arrive first → faster first render (FCP/LCP metrics).

REALITY: Many HTTP/2 implementations ignore priorities or implement them poorly.
HTTP/3 removed prioritization from the spec (moved to HTTP Priority header).
```

### Feature 5: Server Push (Deprecated in Practice)

```
CONCEPT: Server sends resources the client will need BEFORE client asks.

Browser requests /index.html:
Server: "They'll need /style.css and /app.js. I'll push them immediately!"

SERVER PUSH:
┌── Client Request ──────────────────────────────────────────────────┐
│ GET /index.html                                                    │
└────────────────────────────────────────────────────────────────────┘
                ┌── Server Response ─────────────────────────────────┐
                │ PUSH_PROMISE stream 2: /style.css                  │
                │ PUSH_PROMISE stream 4: /app.js                     │
                │ DATA stream 1: (index.html content)                │
                │ HEADERS stream 2: 200 OK                           │
                │ DATA stream 2: (style.css content)                 │
                │ HEADERS stream 4: 200 OK                           │
                │ DATA stream 4: (app.js content)                    │
                └────────────────────────────────────────────────────┘

PROBLEM: Server can't know if client already has /style.css in cache!
Client may already have it → server is pushing data client doesn't need!
Wasted bandwidth.

REALITY: Server Push was removed from Chrome in 2022 (rarely helped in practice).
Alternative: 103 Early Hints HTTP status code (server hints which resources to preload).
```

---

## 3. HTTP/2's Remaining Problem — TCP Head-of-Line Blocking

HTTP/2 solved **application-layer** HOL blocking with multiplexing. But TCP itself still has HOL blocking.

```
HTTP/2 MULTIPLEXING OVER TCP:

On the wire, HTTP/2 streams are interleaved as TCP segments.

Segment 1: [Stream 1 data][Stream 3 data][Stream 5 data]
Segment 2: [Stream 1 data][Stream 7 data]              ← LOST!
Segment 3: [Stream 5 data][Stream 9 data]

TCP guarantees in-order delivery.
Segment 3 arrives at receiver → TCP: "Wait! Segment 2 is missing!"
TCP HOLDS Segment 3 and waits for Segment 2 to be retransmitted.

ALL STREAMS on this connection are blocked, waiting for Segment 2's retransmit!
Even streams 1, 5, 9 that weren't in Segment 2 are blocked!

TCP doesn't understand HTTP/2 streams — it just sees bytes.
One lost packet = all 100 multiplexed streams stall!

ON A GOOD NETWORK (fiber, wired): 0.01% packet loss → barely matters
ON A BAD NETWORK (mobile 4G, WiFi): 2% packet loss + retransmit latency
                                     HOL blocking is a real problem!
Facebook measured: HTTP/2 was SLOWER than HTTP/1.1 in high packet loss conditions!
```

This is the motivation for QUIC and HTTP/3.

---

## 4. HTTP/3 and QUIC — From the Ground Up

### The Problem with Changing TCP

```
Why not just fix TCP?

TCP lives in the OS kernel. To change TCP, you need:
1. Linux kernel developers to implement changes
2. Linus Torvalds to merge them
3. Linux distribution maintainers to ship new kernel
4. Cloud providers to update their instances
5. ISPs to update network equipment
6. End users to update their OS

Timeline: 5-10 YEARS for widespread deployment.

Google's insight: "What if we build a new protocol on top of UDP?"
- UDP is just "send datagrams" — no logic
- We can implement all of TCP's reliability ourselves, in userspace
- Ship updates to QUIC without waiting for OS vendors
- Iterate fast

Result: QUIC (Quick UDP Internet Connections) — 2012 at Google, RFC 9000 (2021)
HTTP/3 = HTTP/2 semantics over QUIC (RFC 9114, 2022)
```

### What QUIC Is — A New Transport Protocol

```
QUIC = UDP + (reliability + ordering + flow control + congestion control
              + multiplexing + TLS 1.3 + connection migration)

All of TCP's features, reimplemented at the application layer on UDP.

KEY INNOVATION: Per-Stream Reliability

QUIC maintains separate reliability for each stream.
Lost packet only blocks the stream it belongs to.
Other streams proceed normally.

HTTP/2 over TCP (packet loss affects ALL streams):
Stream A: [frame1][frame2][frame3 ← LOST!][frame4 blocked][frame5 blocked]
Stream B: [frame1][frame2][frame3 BLOCKED TOO!          ][frame4 blocked ]
Stream C: [frame1][BLOCKED TOO!                                           ]
All streams wait for Stream A's retransmit!

HTTP/3 over QUIC (packet loss only affects that stream):
Stream A: [frame1][frame2][frame3 ← LOST!][frame4 blocked][retransmit frame3]
Stream B: [frame1][frame2][frame3][frame4][frame5]  ← unaffected!
Stream C: [frame1][frame2][frame3][frame4]  ← unaffected!
Only Stream A is blocked. B and C continue at full speed.
```

### QUIC Connection Establishment — The Latency Revolution

```
CONNECTION LATENCY COMPARISON:

HTTP/1.1 over TLS 1.2 (old, still common):

Client          Server
  │─ TCP SYN ──▶│             RTT 0
  │◀─ SYN-ACK ──│             (TCP handshake: 1 RTT)
  │─ ACK ───────▶│
  │─ ClientHello▶│             RTT 1
  │◀─ServerHello+│             (TLS 1.2 handshake: 2 RTT)
  │  Certificate │
  │─ Finished ──▶│             RTT 2
  │◀─ Finished ──│
  │─ HTTP GET ──▶│             RTT 3
  │◀─ Response ──│
Total: 4 RTT before data arrives!
On 100ms RTT: 400ms before first byte of content!

HTTP/2 over TLS 1.3 (modern, common today):
  │─ TCP SYN ──▶│             RTT 0 (TCP)
  │◀─ SYN-ACK ──│
  │─ ACK+ClientHello▶│        RTT 1 (TLS 1.3 = 1 RTT)
  │◀─ServerHello+Cert+Fin──│
  │─ HTTP GET ──▶│             RTT 2
  │◀─ Response ──│
Total: 3 RTT (improved!)
On 100ms RTT: 300ms before first byte.

HTTP/3 over QUIC (new):
  │─ QUIC Initial (includes TLS ClientHello) ──▶│  RTT 0
  │◀─ QUIC Handshake (Server's TLS) ────────────│  (QUIC+TLS combined: 1 RTT)
  │─ HTTP/3 GET ──▶│                              RTT 1
  │◀─ Response ────│
Total: 2 RTT!
On 100ms RTT: 200ms before first byte.

0-RTT RESUMPTION (repeat connections — most real traffic!):
  │─ QUIC + HTTP/3 request (with cached session key) ──▶│  RTT 0
  │◀─ Response ───────────────────────────────────────────│
Total: 1 RTT!! Data in the very first packet!
On 100ms RTT: 100ms — same as just the network latency!
```

### QUIC Connection Migration — Mobile Game Changer

```
SCENARIO: You're on your phone, streaming video on WiFi.
You leave the house → phone switches from WiFi to 4G.

HTTP/2 over TCP:
TCP connection is identified by (src_ip, src_port, dst_ip, dst_port).
Your IP address changes when switching WiFi → 4G.
→ Old TCP connection is DEAD (different src_ip).
→ New TCP connection needed: 3-way handshake + TLS = 2-3 RTT.
→ Video buffering, stream interrupted.

HTTP/3 over QUIC:
QUIC connections are identified by a CONNECTION ID (random 64-bit number).
The connection ID is chosen by client, sent in every QUIC packet.
When your IP changes (WiFi → 4G):
→ Client continues sending packets with same CONNECTION ID.
→ Server receives packets with same CONNECTION ID.
→ Server updates its internal (client_id → new_ip) mapping.
→ Stream continues without interruption!

This is called "connection migration." Massive for mobile users.
```

---

## 5. HTTP/2 vs HTTP/3 — The Full Comparison

```
┌─────────────────────────┬────────────────────────────┬─────────────────────────────────┐
│ Feature                 │ HTTP/2                     │ HTTP/3                          │
├─────────────────────────┼────────────────────────────┼─────────────────────────────────┤
│ Transport protocol      │ TCP                        │ QUIC (on UDP)                   │
│ TLS                     │ Optional (browsers require)│ Always-on (built into QUIC)     │
│ Connection setup        │ 2-3 RTT (TCP+TLS)          │ 1 RTT (first), 0-RTT (repeat)   │
│ App-layer HOL blocking  │ Solved                     │ Solved                          │
│ Transport-layer HOL     │ Still has it (TCP)         │ Solved (per-stream reliability) │
│ Connection migration    │ No (IP tied to TCP conn)   │ Yes (CONNECTION ID survives IP  │
│                         │                            │ change, WiFi→4G seamless)       │
│ Header compression      │ HPACK                      │ QPACK (HPACK adapted for QUIC)  │
│ Encrypted headers       │ No (TLS encrypts payload)  │ Yes (QUIC encrypts everything   │
│                         │                            │ including header fields)        │
│ Firewall compatibility  │ Excellent (standard TCP)   │ Good (UDP sometimes blocked)    │
│                         │                            │ ~5-8% of networks block UDP     │
│ Server push             │ Yes (but deprecated)       │ Yes (but deprecated)            │
│ Packet loss sensitivity │ High (all streams stall)   │ Low (only affected stream stalls│
│ Mobile performance      │ Degrades with packet loss  │ Excellent (loss + migration)    │
│ Adoption               │ ~65% of web traffic         │ ~30%+ and growing (2024)        │
│ Best for                │ Most web traffic            │ Mobile, high packet loss, low   │
│                         │ Low-loss reliable networks  │ latency requirements            │
└─────────────────────────┴────────────────────────────┴─────────────────────────────────┘
```

---

## 6. Real-World Impact Numbers

```
Google (2020 study, HTTP/3 vs HTTP/2):
- Search result latency: 2% improvement on desktop, 5% on mobile
- YouTube rebuffering: 9% reduction on desktop, 15% on mobile
- YouTube quality: 8-11% improvement
(Numbers seem small but at Google's scale = millions of better user experiences)

Facebook (2020 internal data):
- HTTP/3 reduced request error rate by 8%
- Reduced tail latency (p95, p99) by 10%
- Most improvement in high packet-loss conditions (developing markets, rural)

Cloudflare internal analysis:
- 8-12% faster Time to First Byte (TTFB) with HTTP/3
- Up to 30% improvement on mobile networks with >2% packet loss
```

---

## 7. Deployment Considerations

```
SERVING HTTP/3 TODAY:

Server support:
✅ Nginx (1.25.x + ngtcp2/OpenSSL 3 or nginx-quic patch)
✅ Caddy (native QUIC support out of the box)
✅ LiteSpeed Web Server (native)
✅ Cloudflare (HTTP/3 on all PoPs)
✅ AWS CloudFront (HTTP/3 supported)
✅ Google Cloud Load Balancer (HTTP/3 supported)

Client support:
✅ Chrome 87+ (Jan 2021)
✅ Firefox 88+ (Apr 2021)
✅ Safari 14+ (2020)
✅ Edge 87+ (Jan 2021)
❌ Older browsers → automatic fallback to HTTP/2 or HTTP/1.1

Alt-Svc header tells clients HTTP/3 is available:
alt-svc: h3=":443"; ma=86400, h3-29=":443"; ma=86400
         └── "HTTP/3 (h3) is available on port 443, cached for 24 hours"
         
Client receives this, upgrades to HTTP/3 on next connection.
Graceful: if QUIC fails, falls back to HTTP/2 automatically.

UDP Firewall Problem:
Some corporate firewalls, ISPs, and networks block or throttle UDP.
HTTP/3 falls back gracefully to HTTP/2 on these networks.
Chrome shows "h3" in DevTools Network tab if HTTP/3 is being used.
```

---

## 8. Interview Quick-Fire Answers

**Q: What is head-of-line blocking and how does HTTP/2 solve it?**
A: In HTTP/1.1, one connection handles one request at a time — Request 2 waits for Response 1 to finish. HTTP/2 solves this with multiplexing: multiple streams share one TCP connection, interleaved as binary frames tagged with stream IDs. Response to Request 1 and Response to Request 2 can arrive simultaneously, in any order.

**Q: HTTP/2 uses multiplexing, so why is HTTP/3 still needed?**
A: HTTP/2 solved head-of-line blocking at the HTTP application layer, but TCP itself still has HOL blocking. If one TCP packet is lost, ALL HTTP/2 streams on that connection wait for the retransmit — even streams that didn't need that packet. HTTP/3 uses QUIC on UDP with per-stream reliability — a lost packet only blocks the stream it belongs to. Also, HTTP/3 has 0-RTT connection resumption and connection migration (surviving WiFi→4G switches), which HTTP/2 over TCP cannot do.

**Q: What is HPACK and why does HTTP/2 need it?**
A: HTTP/1.1 headers are plain text, uncompressed, and repeated in full on every request. Cookies, user-agent, authorization headers can total 400-800 bytes per request. HPACK uses a combination of static table (61 common headers), dynamic table (headers seen in this connection), and Huffman encoding to compress headers. After the first request, subsequent requests only send changed headers as deltas — typical savings of 50-90%.

**Q: What is 0-RTT in QUIC/HTTP/3?**
A: On a first connection, QUIC takes 1 RTT to establish (combined QUIC+TLS 1.3 handshake). On subsequent connections (returning users), the client can include application data in the very first packet using a cached session ticket. The server can process this data immediately — 0 round trips before data. This dramatically reduces latency for returning users. Trade-off: 0-RTT data is vulnerable to replay attacks, so it should only be used for safe, idempotent requests like GET.


---
---

# TOPIC 7: CDN — Content Delivery Network

---

## 1. What Problem Does a CDN Solve?

### The Physics Problem

Data cannot travel faster than the speed of light. In fiber optic cable, light travels at about 200,000 km/second (slower than vacuum due to the refractive index of glass).

```
DISTANCE-BASED LATENCY (one-way, theoretical minimum):

Mumbai to Virginia, USA: ~13,000 km
13,000 km ÷ 200,000 km/s = 65ms ONE WAY
Round trip (RTT) = 130ms minimum — and that's the THEORETICAL BEST CASE
                                    (real-world: routing hops, congestion add more)

Real-world measured RTT, Mumbai → US-East: ~230-280ms

Now imagine your website has:
- 1 HTML file
- 3 CSS files  
- 10 JS files
- 30 images

Even with HTTP/2 multiplexing (1 connection), every byte still has to
travel 13,000 km and back. For a user in Mumbai loading a US-hosted site:

Page load time floor = network RTT × (number of round trips needed)
                     = 250ms × 3-4 round trips (DNS, TCP, TLS, HTTP)
                     = 750-1000ms BEFORE the page even starts rendering!
```

### The CDN Solution

```
A CDN places COPIES of your content on servers physically close
to your users — at "Points of Presence" (PoPs) or "edge locations" 
around the world.

WITHOUT CDN:
User in Mumbai ──────────── 13,000 km ──────────── Origin server (Virginia)
                         (250ms RTT)

WITH CDN:
User in Mumbai ── 5 km ── CDN Edge Server (Mumbai PoP)
                (2ms RTT)
                     │
                     │ (only on cache MISS, rarely)
                     ▼
              Origin server (Virginia)
```

### Analogy

```
Without CDN: Like having ONE massive warehouse in the US, and every
time someone in India orders a product, it gets shipped individually
from the US warehouse. Slow, expensive, doesn't scale.

With CDN: Like Amazon having regional fulfillment centers. 
Popular products are pre-stocked in the Mumbai warehouse.
When you order, it ships from 5km away, not 13,000km away.
Only rare/unusual items need to come from the central US warehouse.
```

---

## 2. Core Intuition — How a CDN Actually Works

### Step-by-Step: First Request (Cache MISS)

```
USER (Mumbai) wants: https://cdn.shop.com/product-images/shoe123.jpg

STEP 1: DNS RESOLUTION
User's DNS query for cdn.shop.com
→ CDN's DNS infrastructure (often using Anycast or GeoDNS)
→ Returns IP of NEAREST edge server (Mumbai PoP: 104.18.x.x)

STEP 2: CONNECTION TO EDGE
User connects to Mumbai edge server (2-5ms RTT — very close!)
TCP + TLS handshake completes quickly

STEP 3: CACHE CHECK AT EDGE
Mumbai edge server checks its local cache:
"Do I have shoe123.jpg cached?"
→ NO (first time anyone requested this in Mumbai region)
→ CACHE MISS

STEP 4: ORIGIN FETCH
Mumbai edge server makes a request to ORIGIN SERVER (Virginia)
"GET /product-images/shoe123.jpg"
→ This trip takes 250ms RTT (Mumbai ↔ Virginia)
→ Origin returns the image + cache headers (Cache-Control: max-age=86400)

STEP 5: CACHE AND SERVE
Mumbai edge server:
  a) Stores image in its local cache (valid for 86400 seconds)
  b) Serves the image to the user

Total time for FIRST user: ~5ms (to edge) + 250ms (edge to origin) + transfer time
                          ≈ 255ms+ (similar to no CDN, for THIS user)
```

### Step-by-Step: Subsequent Requests (Cache HIT)

```
SECOND USER (also in Mumbai) wants the SAME image: shoe123.jpg

STEP 1: DNS RESOLUTION → Mumbai edge server (same as before)

STEP 2: CONNECTION TO EDGE → 2-5ms RTT

STEP 3: CACHE CHECK AT EDGE
"Do I have shoe123.jpg cached?"
→ YES! (cached from previous user, TTL still valid)
→ CACHE HIT

STEP 4: SERVE IMMEDIATELY FROM CACHE
No call to origin needed!

Total time for SECOND user: ~5ms + transfer time
≈ 50x faster than the first user, and 50x faster than without CDN!

THIS IS THE CDN VALUE PROPOSITION:
- First user "pays" the origin round trip (cache miss, rare)
- All subsequent users (could be millions) get edge speed (cache hit)
- 95-99% of requests for popular content are cache hits
```

---

## 3. CDN Architecture Diagram

```
                          ┌─────────────────────────┐
                          │   ORIGIN SERVER          │
                          │   (Your application,     │
                          │   e.g., Virginia, USA)   │
                          └────────────┬─────────────┘
                                       │
                          (Origin pulls — only on cache miss)
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
┌───────▼────────┐            ┌────────▼────────┐           ┌────────▼────────┐
│  Mumbai PoP     │            │  Frankfurt PoP   │           │  Singapore PoP   │
│  (Edge Server)  │            │  (Edge Server)   │           │  (Edge Server)   │
│  ┌───────────┐  │            │  ┌───────────┐   │           │  ┌───────────┐   │
│  │ Cache:    │  │            │  │ Cache:    │   │           │  │ Cache:    │   │
│  │ -images   │  │            │  │ -images   │   │           │  │ -images   │   │
│  │ -css/js   │  │            │  │ -css/js   │   │           │  │ -css/js   │   │
│  │ -api(some)│  │            │  │ -api(some)│   │           │  │ -api(some)│   │
│  └───────────┘  │            │  └───────────┘   │           │  └───────────┘   │
└────────┬────────┘            └────────┬─────────┘           └────────┬─────────┘
         │                              │                              │
   ┌─────▼─────┐                  ┌─────▼─────┐                  ┌─────▼─────┐
   │ Indian    │                  │ European  │                  │ SEA       │
   │ Users     │                  │ Users     │                  │ Users     │
   └───────────┘                  └───────────┘                  └───────────┘

Each PoP serves users in its geographic region from local cache.
Origin server only receives traffic for cache misses (typically <5% of total requests).
```

---

## 4. What CDNs Cache — Static vs Dynamic

### Static Content (The Classic Use Case)

```
PERFECT FOR CDN CACHING (rarely or never changes):
┌──────────────────────────────────────────────────────────────┐
│ Resource Type      │ Example                │ Typical TTL    │
├──────────────────────────────────────────────────────────────┤
│ Images             │ product-photo.jpg      │ 7-30 days      │
│ Videos             │ tutorial.mp4           │ 7-30 days      │
│ Fonts              │ roboto-regular.woff2   │ 1 year (immutable)│
│ JS/CSS bundles     │ main.a3f8b2.js         │ 1 year (immutable, content-hashed)│
│ PDFs/Documents     │ user-manual.pdf        │ 7 days         │
│ Software downloads │ app-installer.exe      │ 30 days        │
└──────────────────────────────────────────────────────────────┘

Why content-hashed filenames matter:
main.js → if content changes, new deploy creates main.a8f3c1.js
The OLD filename main.a3f8b2.js NEVER changes → cache forever (immutable)
The NEW filename is a NEW URL → CDN treats it as new content (cache miss once, then cached)
No invalidation needed! The URL itself encodes the version.
```

### Dynamic Content (More Nuanced)

```
SOMETIMES CACHEABLE:
┌──────────────────────────────────────────────────────────────────┐
│ Resource Type        │ Caching Strategy                          │
├──────────────────────────────────────────────────────────────────┤
│ Product listing page │ Cache for 60s with stale-while-revalidate │
│ (same for all users) │ Vary: Accept-Language for localization    │
│                       │                                            │
│ API responses        │ Short TTL (5-30s) for non-personalized    │
│ (e.g., trending      │ data. NEVER cache personalized API         │
│ items, public stats) │ responses (user-specific data)             │
│                       │                                            │
│ HTML pages           │ Edge-Side Includes (ESI) — cache the       │
│ (mostly static, some │ static shell, dynamically fill in          │
│ personalized parts)  │ personalized fragments at edge             │
└──────────────────────────────────────────────────────────────────┘

NEVER CACHE (or cache=private, no CDN):
- User's account/profile page
- Shopping cart contents  
- Real-time data (stock prices, live scores) — use WebSocket instead
- Anything behind authentication that's user-specific
- Payment/checkout pages

THE Vary HEADER:
Cache-Control: public, max-age=3600
Vary: Accept-Language, Accept-Encoding

→ CDN caches SEPARATE versions for each combination of
  Accept-Language (en, hi, fr...) and Accept-Encoding (gzip, br)
→ English user gets English cached version
→ Hindi user gets Hindi cached version
→ Both versions cached independently at edge
```

---

## 5. Cache Invalidation — The Hardest Problem in Computer Science (Joke, but True)

"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### Strategy 1: TTL Expiry (Time-Based)

```
Cache-Control: max-age=3600

Edge server caches response with a timestamp.
After 3600 seconds, cache entry is "stale."
Next request → edge re-fetches from origin (cache miss).

PROS: Simple, automatic, no manual intervention
CONS: Up to `max-age` seconds of staleness after content changes
      (your news article update might not show for up to 1 hour)
```

### Strategy 2: Versioned URLs / Cache Busting

```
BEFORE deploy: <script src="/main.js"></script>
                CDN caches /main.js with hash ABC123

You change the JS code.

AFTER deploy: <script src="/main.a8f3c1.js"></script>
              ↑ Build tool generates new hash from new content
              
NEW URL = NEW cache entry. Old URL (main.a3f8b2.js) still cached,
but nobody requests it anymore (HTML references new URL).

Old cached entry simply expires naturally — no invalidation needed!
This is THE BEST PRACTICE for JS/CSS/images with build pipelines.
```

### Strategy 3: Explicit Purge / Invalidation API

```
Used when content changes UNEXPECTEDLY and you can't wait for TTL.

Example: A news article had a factual error, needs immediate correction.

curl -X POST "https://api.cloudflare.com/zones/{zone}/purge_cache" \
  -H "Authorization: Bearer {token}" \
  -d '{"files":["https://news.com/article/12345"]}'

→ Cloudflare immediately removes this URL from ALL edge caches worldwide
→ Next request = cache miss = fresh fetch from origin

PROS: Immediate
CONS: API call needed for every change; doesn't scale to "purge 10,000 product pages"
      Purge propagation can take a few seconds across all PoPs
```

### Strategy 4: Surrogate Keys / Cache Tags (Most Powerful)

```
PROBLEM: An e-commerce category page shows 50 products.
One product's price changes. Do you purge the category page URL?
What if the product appears on 20 different category/search pages?

SOLUTION: Tag responses with logical keys (Surrogate-Key header)

Response for /category/shoes:
Surrogate-Key: category-shoes product-123 product-456 product-789

Response for /search?q=nike:
Surrogate-Key: search-nike product-123 product-999

When product-123's price changes:
PURGE BY TAG: "product-123"
→ CDN invalidates ALL responses tagged with "product-123"
→ Both /category/shoes AND /search?q=nike get invalidated
→ One API call, regardless of how many pages reference this product

USED BY: Fastly (Surrogate-Key), Cloudflare (Cache-Tag) — common in 
headless CMS and e-commerce architectures (Shopify, commercetools)
```

### Strategy 5: Stale-While-Revalidate

```
Cache-Control: max-age=60, stale-while-revalidate=300

What happens:
0-60s:    Cache is FRESH. Serve from cache immediately. No origin call.
60-360s:  Cache is STALE but still served! 
          AND a background request goes to origin to refresh the cache.
          User gets fast (stale) response immediately,
          next user gets the FRESH response (already refreshed).
360s+:    Cache fully expired. Must wait for origin (cache miss, blocking).

This means users almost NEVER experience the "cache miss" latency spike —
they always get a fast response, while freshness is maintained in the background.

USED BY: Most modern CDNs and frameworks (Next.js ISR uses this concept)
```

---

## 6. CDN Beyond Caching — Modern Edge Capabilities

CDNs have evolved from "dumb caches" to full edge computing platforms.

### DDoS Mitigation

```
A CDN's massive distributed capacity acts as a shock absorber.

Without CDN: Attacker sends 1 Tbps of traffic directly at your origin.
             Your origin (maybe 10 Gbps capacity) is instantly overwhelmed.

With CDN: Attack traffic hits CDN edge servers FIRST.
          CDN has aggregate capacity of 100+ Tbps across all PoPs.
          CDN absorbs and filters malicious traffic.
          Only legitimate traffic reaches your origin.

Real example: Cloudflare mitigated a 71 million requests/second DDoS attack 
(2023) — the largest HTTP DDoS recorded at the time. Origin never saw it.
```

### Web Application Firewall (WAF)

```
WAF rules run AT THE EDGE, before requests reach your origin:

- SQL Injection detection: blocks requests with ' OR 1=1-- patterns
- XSS detection: blocks <script> tags in form inputs
- Rate limiting: max 100 requests/minute per IP for /login
- Bot detection: blocks known bad user-agents, challenges suspicious traffic
- Geo-blocking: block traffic from specific countries

BENEFIT: Malicious requests never reach your application servers.
         Your origin only processes legitimate traffic.
```

### Edge Compute (Serverless at the Edge)

```
Run actual application code AT the edge location (closest to user):

Cloudflare Workers / Fastly Compute@Edge / AWS Lambda@Edge

USE CASES:
- A/B testing: rewrite responses based on cookie at edge (no origin call)
- Authentication: verify JWT tokens at edge, reject invalid before origin
- Image transformation: resize/format images on-the-fly at edge
- Personalization: inject user-specific data into cached HTML shell
- API composition: combine multiple API calls into one edge function

EXAMPLE (Cloudflare Worker, simplified):
addEventListener('fetch', event => {
  const country = event.request.headers.get('CF-IPCountry')
  if (country === 'IN') {
    return fetch('https://origin.com/in/homepage')
  }
  return fetch('https://origin.com/global/homepage')
})

This runs in <1ms at 300+ locations worldwide — no origin round trip needed
for routing decisions.
```

### Image Optimization

```
ORIGIN stores: product-photo.jpg (4000x3000px, 5MB, JPEG)

CDN serves different versions based on requesting device:
- iPhone (Retina, narrow viewport): 800x600px, WebP, 80KB
- Desktop (wide viewport): 1600x1200px, WebP, 250KB
- Old Android (no WebP support): 800x600px, JPEG, 150KB

All generated ON-THE-FLY at edge, cached per variant.
URL: https://cdn.shop.com/product-photo.jpg?width=800&format=webp

Origin stores ONE high-quality master image.
CDN generates and caches all needed variants automatically.
```

---

## 7. CDN Real-World Architectures

### Cloudflare — The Anycast Giant

```
- 300+ cities, 100+ countries
- ~20% of all internet websites use Cloudflare (as of 2024)
- Uses Anycast: the IP 104.16.x.x is announced from ALL 300+ locations
  via BGP. Internet routing automatically sends traffic to the
  topologically nearest location.
- 1.1.1.1 DNS resolver — fastest public DNS, 11ms median globally
- Argo Smart Routing: routes traffic over Cloudflare's private backbone
  instead of public internet for the origin-pull leg (cache miss path)
```

### Netflix Open Connect — Building Their Own CDN

```
Netflix made an unusual choice: instead of using a third-party CDN
(Akamai, Cloudflare), they built their OWN CDN called "Open Connect."

WHY?
- Video streaming = massive, predictable bandwidth (35% of US internet
  traffic during peak hours at one point!)
- Third-party CDN costs at this scale would be enormous
- Video content is HIGHLY cacheable (same files watched by millions)

HOW IT WORKS:
- Netflix builds physical appliances (servers with huge SSDs)
- These are placed DIRECTLY INSIDE ISP data centers (free of charge to ISPs —
  reduces ISP's own bandwidth costs to internet backbone)
- Every night, Netflix PRE-POSITIONS popular content onto these appliances
  based on predictions of what will be watched tomorrow in that region
- ~95%+ of Netflix streams are served from these local appliances —
  literally inside your ISP's network, often <1ms away

This is "predictive pre-caching" rather than reactive cache-miss-then-cache.
```

### E-commerce — Cache Tags in Action (Shopify pattern)

```
Shopify stores serve millions of product pages.

Product page /products/nike-air-max:
Surrogate-Key: product_123 collection_shoes vendor_nike

When merchant updates the product price:
→ Shopify's backend fires: PURGE Surrogate-Key product_123
→ All cached pages referencing product_123 invalidated:
  - The product page itself
  - The "shoes" collection page (shows this product)
  - Any "related products" widgets featuring it
  - Search results pages

One event → precise invalidation across the entire cached surface area.
```

---

## 8. Failure Scenarios — CDN-Specific

```
┌────────────────────────────┬──────────────────────────────┬─────────────────────────────────┐
│ Failure                    │ Root Cause                   │ Mitigation                       │
├────────────────────────────┼──────────────────────────────┼─────────────────────────────────┤
│ Thundering herd            │ Popular content's cache       │ Request collapsing: edge server  │
│ ("cache stampede")         │ entry expires; millions of    │ sends ONLY ONE request to origin │
│                            │ simultaneous requests all      │ even if 10,000 users request the │
│                            │ become cache misses → all hit │ same expired resource            │
│                            │ origin simultaneously         │ simultaneously; others wait for  │
│                            │                               │ that one response                │
├────────────────────────────┼──────────────────────────────┼─────────────────────────────────┤
│ Cache poisoning            │ Attacker crafts a request     │ Normalize cache keys carefully;  │
│                            │ with unusual headers that     │ explicitly define Vary headers;  │
│                            │ causes CDN to cache a BAD     │ validate/sanitize headers used   │
│                            │ response and serve it to      │ in cache key                     │
│                            │ everyone                       │                                   │
├────────────────────────────┼──────────────────────────────┼─────────────────────────────────┤
│ CDN provider outage        │ Your CDN provider has a       │ Multi-CDN strategy (use 2+ CDN   │
│ (e.g., Fastly June 2021    │ regional or global incident   │ providers, switch via DNS); low  │
│ outage took down Reddit,   │                               │ TTL DNS for fast failover;       │
│ GitHub, Amazon, NYT,        │                               │ origin shield as fallback        │
│ Spotify simultaneously)    │                               │                                   │
├────────────────────────────┼──────────────────────────────┼─────────────────────────────────┤
│ Stale content after deploy │ New HTML deployed, but old     │ Cache-bust HTML aggressively     │
│ (users see old JS/CSS      │ JS/CSS still cached at edge    │ (short TTL or no-cache for HTML);│
│ referenced by new HTML, or │ with longer TTL                │ content-hash filenames for       │
│ vice versa)                │                               │ JS/CSS so URLs always match      │
├────────────────────────────┼──────────────────────────────┼─────────────────────────────────┤
│ Origin overload on cache   │ TTLs too short, or too much   │ Increase TTLs for cacheable      │
│ miss spike                 │ uncacheable dynamic content    │ content; use "origin shield"     │
│                            │                               │ (one mid-tier cache layer between│
│                            │                               │ edge PoPs and origin, reduces    │
│                            │                               │ origin requests further)         │
└────────────────────────────┴──────────────────────────────┴─────────────────────────────────┘
```

---

## 9. Interview Quick-Fire Answers

**Q: How does a CDN reduce latency?**
A: A CDN places cached copies of content on servers ("edge locations" or "PoPs") geographically close to users. Instead of every request traveling to a single origin server (potentially thousands of km away), most requests are served from a nearby edge — reducing round-trip time from hundreds of milliseconds to single digits. Only cache misses (typically <5% of requests) travel to the origin.

**Q: What is cache invalidation and what strategies exist?**
A: Cache invalidation is removing or refreshing stale cached content. Key strategies: TTL expiry (automatic, time-based, simple but can serve stale data temporarily), versioned/content-hashed URLs (new content = new URL, old cache naturally expires, no invalidation needed — best for JS/CSS), explicit purge APIs (immediate but doesn't scale to bulk changes), and surrogate keys/cache tags (tag responses with logical identifiers, purge by tag to invalidate many related pages at once).

**Q: What is the thundering herd problem and how do CDNs solve it?**
A: When a popular cached resource's TTL expires, potentially thousands of simultaneous requests become cache misses at the same moment, all hitting the origin simultaneously — overwhelming it. CDNs solve this with "request collapsing": the edge server detects multiple concurrent requests for the same expiring resource and sends only ONE request to origin; all other requests wait for that single response and share it.

**Q: Why did Netflix build their own CDN instead of using Cloudflare/Akamai?**
A: Video streaming is extremely bandwidth-intensive and highly cacheable (millions of users watch the same files). At Netflix's scale, third-party CDN costs would be enormous. They built "Open Connect" — physical appliances placed directly inside ISP data centers, pre-loaded nightly with predicted popular content. This achieves ~95%+ cache hit rates with content served literally from inside the user's ISP network.

---
---

# TOPIC 8: OSI Model

---

## 1. What Problem Does the OSI Model Solve?

Networking involves many different concerns operating simultaneously: physical signals on a wire, addressing on a local network, routing across the internet, ensuring reliable delivery, and the actual application logic (like HTTP).

If all of this was one giant tangled system, it would be impossible to:
- Debug problems (where exactly is the failure?)
- Develop independently (Ethernet engineers and HTTP engineers don't need to coordinate)
- Replace one piece without breaking everything (switch from WiFi to Ethernet without changing your browser)

The **OSI (Open Systems Interconnection) Model** (1984, ISO standard) is a conceptual framework that divides networking into **7 layers**, each with a specific, well-defined responsibility. Each layer only talks to the layer directly above and below it.

> **Important nuance for interviews:** The OSI model is a *conceptual teaching framework*. The internet actually runs on the **TCP/IP model** (4 layers), which is simpler. But OSI vocabulary ("Layer 4 load balancer", "Layer 7 firewall") is used constantly in industry, so you must know both.

---

## 2. The Analogy — International Mail

```
Imagine sending a contract to a business partner in another country.

LAYER 7 (Application):  You write the contract (the actual content/meaning)
LAYER 6 (Presentation): You translate it to their language, encode in PDF
LAYER 5 (Session):      You establish a "conversation" — this is contract #47,
                         expect a signed reply
LAYER 4 (Transport):    You choose: registered mail (guaranteed, tracked) 
                         or regular mail (faster, no tracking)
LAYER 3 (Network):       You write the destination COUNTRY and CITY address
LAYER 2 (Data Link):     You hand it to the LOCAL POST OFFICE, which knows
                         how to get it to the next post office (local routing)
LAYER 1 (Physical):      The actual truck/plane/ship physically carrying
                         the envelope

Each layer adds its own "envelope" around what the layer above gave it.
The recipient's layers each "open" their corresponding envelope.
```

---

## 3. The 7 Layers — Deep Explanation, Bottom to Top

### Layer 1: Physical Layer

```
RESPONSIBILITY: Transmit raw bits (0s and 1s) over a physical medium.

WHAT IT DEALS WITH:
- Electrical signals (voltage levels on copper wire)
- Light pulses (fiber optic cables)
- Radio waves (WiFi, Bluetooth, cellular)
- Physical connectors (RJ45, USB, fiber connectors)
- Cable specifications (Cat5e, Cat6, fiber types)

EXAMPLE: Your Ethernet cable physically carries electrical pulses.
A "1" might be represented as +0.85V, a "0" as -0.85V (in Ethernet's
differential signaling).

DEVICES OPERATING HERE: Hubs, repeaters, network cables, NICs (physical part)

NO CONCEPT OF "DATA" YET — just raw signal transmission.
```

### Layer 2: Data Link Layer

```
RESPONSIBILITY: Reliable transfer of data frames between two nodes
ON THE SAME NETWORK SEGMENT (i.e., devices that can "see" each other
directly, like all computers plugged into the same switch).

KEY CONCEPTS:

MAC ADDRESS: A unique 48-bit hardware identifier burned into every
network card. Format: AA:BB:CC:DD:EE:FF

Example: 3C:22:FB:45:6A:01 (this is YOUR laptop's unique address)

FRAME STRUCTURE (Ethernet):
┌────────────┬────────────┬──────┬─────────────┬──────┐
│ Dest MAC   │ Src MAC    │ Type │ Payload     │ CRC  │
│ (6 bytes)  │ (6 bytes)  │(2byte)│(46-1500 byte)│(4byte)│
└────────────┴────────────┴──────┴─────────────┴──────┘

ARP (Address Resolution Protocol):
"I know the IP address (192.168.1.5), but what's the MAC address?"

Device broadcasts: "Who has 192.168.1.5? Tell 192.168.1.10"
                    (broadcast to FF:FF:FF:FF:FF:FF — everyone on segment)
Device with that IP responds: "192.168.1.5 is at MAC AA:BB:CC:DD:EE:FF"
Requester caches this mapping (ARP cache/table)

VLAN (Virtual LAN):
Logically segments a physical network into multiple isolated networks.
VLAN 10 = Engineering, VLAN 20 = Finance — same physical switches,
but devices in VLAN 10 cannot directly see VLAN 20 traffic.

DEVICES OPERATING HERE: Switches, bridges, network interface cards (logical part)
```

### Layer 3: Network Layer

```
RESPONSIBILITY: Routing data ACROSS DIFFERENT NETWORKS — getting a
packet from your laptop in Mumbai to a server in Virginia, through
potentially dozens of intermediate networks/routers.

KEY CONCEPTS:

IP ADDRESS: Logical address (unlike MAC, which is physical/hardware)
IPv4: 192.168.1.10 (32-bit, ~4.3 billion addresses — exhausted!)
IPv6: 2001:0db8:85a3::8a2e:0370:7334 (128-bit, 340 undecillion addresses)

IP PACKET STRUCTURE (simplified IPv4):
┌──────────┬──────────┬─────┬──────────────┬──────────────┬─────────┐
│ Version  │ Header   │ TTL │ Source IP    │ Dest IP      │ Payload │
│ (4 bits) │ Length   │     │ (32 bits)    │ (32 bits)    │         │
└──────────┴──────────┴─────┴──────────────┴──────────────┴─────────┘

TTL (Time To Live) — NOT the same as DNS TTL!
This is a HOP COUNT. Each router that forwards the packet decrements TTL by 1.
If TTL reaches 0, the packet is DROPPED (prevents infinite routing loops).
Default TTL ~64 or 128. This is how `traceroute` works — sends packets
with TTL=1, 2, 3... and sees which router responds with "TTL exceeded"
at each hop, mapping the entire route.

ROUTING:
Routers maintain ROUTING TABLES — "to reach network X, send via interface Y"
BGP (Border Gateway Protocol): how ISPs and large networks exchange
routing information — "I can reach these IP ranges, route through me"
This is the protocol that makes Anycast (CDN) possible.

NAT (Network Address Translation):
Your home has ONE public IP (assigned by ISP).
Your laptop, phone, smart TV all have PRIVATE IPs (192.168.1.x).
Your router translates: 192.168.1.10:54321 ↔ Public IP:30000
                          192.168.1.11:54322 ↔ Public IP:30001
To the internet, all your devices appear to come from ONE public IP.

ICMP: Used for diagnostics — `ping` sends ICMP Echo Request,
gets ICMP Echo Reply. "TTL exceeded" messages also use ICMP.

DEVICES OPERATING HERE: Routers, Layer 3 switches
```

### Layer 4: Transport Layer

```
RESPONSIBILITY: End-to-end communication between APPLICATIONS
(not just devices) — this is where TCP and UDP live.
Covered in deep detail in Topic 3 (TCP vs UDP).

KEY ADDITION AT THIS LAYER: PORTS
IP address gets you to the right MACHINE.
Port number gets you to the right APPLICATION on that machine.

(source_ip, source_port, dest_ip, dest_port) = unique connection identifier

DEVICES OPERATING HERE: Layer 4 Load Balancers (route based on IP:port only,
don't look at HTTP content)
```

### Layer 5: Session Layer

```
RESPONSIBILITY: Managing "sessions" — establishing, maintaining,
and terminating connections between applications. Handles 
authentication and reconnection.

In practice, most modern protocols (HTTP, TLS) absorb session
management into themselves rather than using a distinct Layer 5
protocol. This is why Layer 5 is often considered "merged" into
Layer 7 in the TCP/IP model.

EXAMPLES: 
- NetBIOS sessions (older Windows networking)
- RPC (Remote Procedure Call) session establishment
- SQL database connection sessions

In modern web stacks: HTTP cookies/sessions, TLS session 
resumption tickets serve this role.
```

### Layer 6: Presentation Layer

```
RESPONSIBILITY: Translation, encryption, compression — ensuring
data is in a format the receiving application can understand,
regardless of how the sending application encoded it.

EXAMPLES:
- Character encoding: ASCII, UTF-8, UTF-16
- Encryption/Decryption: TLS/SSL (often considered Layer 6, though
  it spans 4-7 in practice)
- Compression: gzip, Brotli
- Data format conversion: JPEG, PNG, MPEG (image/video encoding)

In modern stacks: TLS handles encryption (technically operates 
across layers), Content-Type and Content-Encoding HTTP headers 
handle format/compression negotiation — these are Layer 6 concerns
implemented within Layer 7 protocols.
```

### Layer 7: Application Layer

```
RESPONSIBILITY: The protocols that applications directly use to
communicate. This is the layer most developers work with daily.

PROTOCOLS:
HTTP/HTTPS  — Web browsing, APIs
DNS         — Name resolution
SMTP/IMAP/POP3 — Email
FTP/SFTP    — File transfer
SSH         — Secure remote shell
gRPC        — RPC over HTTP/2
WebSocket   — Bidirectional communication
MQTT        — IoT messaging

DEVICES OPERATING HERE: Layer 7 Load Balancers (read HTTP headers, URLs,
cookies — can route based on /api vs /static), Web Application Firewalls,
API gateways
```

---

## 4. The Complete 7-Layer Table

```
┌─────┬───────────────┬──────────┬────────────────────────────┬─────────────────────────────────┬──────────────────────┐
│  #  │ Layer         │ PDU Name │ Key Protocols              │ Responsibility                   │ Example Device       │
├─────┼───────────────┼──────────┼────────────────────────────┼─────────────────────────────────┼──────────────────────┤
│  7  │ Application   │ Data     │ HTTP, DNS, SMTP, FTP, gRPC │ User-facing protocols             │ L7 Load Balancer, WAF│
│  6  │ Presentation  │ Data     │ TLS/SSL, JPEG, Compression │ Encryption, encoding, compression │ —                    │
│  5  │ Session       │ Data     │ NetBIOS, RPC sessions       │ Session establishment/management  │ —                    │
│  4  │ Transport     │ Segment  │ TCP, UDP, QUIC              │ End-to-end delivery, ports        │ L4 Load Balancer     │
│  3  │ Network       │ Packet   │ IP, ICMP, BGP, OSPF         │ Routing across networks            │ Router               │
│  2  │ Data Link     │ Frame    │ Ethernet, WiFi, ARP, MAC    │ Node-to-node on same network       │ Switch               │
│  1  │ Physical      │ Bit      │ Ethernet PHY, fiber, radio  │ Raw bit transmission                │ Hub, cable, NIC      │
└─────┴───────────────┴──────────┴────────────────────────────┴─────────────────────────────────┴──────────────────────┘

PDU = Protocol Data Unit (the "chunk" of data at that layer)
```

---

## 5. Data Encapsulation — How It All Fits Together

This is the single most important diagram to understand for OSI.

```
SENDING SIDE (top-down, each layer wraps data from above):

Layer 7 (App):     [ HTTP Request: "GET /index.html" ]
                              │
                              │ Layer 4 wraps it:
                              ▼
Layer 4 (TCP):     [ TCP Header | HTTP Request                          ]
                    (src port, dst port, seq#, ack#, flags...)
                              │
                              │ Layer 3 wraps it:
                              ▼
Layer 3 (IP):      [ IP Header | TCP Header | HTTP Request              ]
                    (src IP, dst IP, TTL, protocol...)
                              │
                              │ Layer 2 wraps it:
                              ▼
Layer 2 (Ethernet):[Eth Header| IP Header | TCP Header | HTTP Request |Eth Trailer]
                    (src MAC, dst MAC)                                  (CRC checksum)
                              │
                              │ Layer 1 transmits:
                              ▼
Layer 1 (Physical): 1011010010110101001011010010110101...
                    (actual electrical/optical signal on the wire)


RECEIVING SIDE (bottom-up, each layer unwraps and passes up):

Layer 1: Receives bits, reconstructs the Ethernet frame
Layer 2: Reads Ethernet header, checks "is this MAC address mine?"
         Strips Ethernet header/trailer, passes IP packet up
Layer 3: Reads IP header, checks "is this IP address mine?"
         Strips IP header, passes TCP segment up
Layer 4: Reads TCP header, checks sequence numbers, sends ACK
         Reassembles in-order data, passes to correct PORT/application
Layer 7: Application (web server) receives "GET /index.html"
         and processes the HTTP request
```

---

## 6. TCP/IP Model — What's Actually Used

```
The OSI model has 7 layers. The TCP/IP model (what the internet 
actually implements) has 4 layers — layers 5, 6, 7 of OSI collapse
into one "Application" layer.

┌─────────────────────┬─────────────────┬───────────────────────────────┐
│ TCP/IP Model        │ OSI Equivalent  │ Examples                       │
├─────────────────────┼─────────────────┼───────────────────────────────┤
│ Application         │ Layers 5, 6, 7  │ HTTP, HTTPS, DNS, TLS, gRPC,   │
│                     │                 │ WebSocket, SSH                │
│ Transport           │ Layer 4         │ TCP, UDP, QUIC                 │
│ Internet            │ Layer 3         │ IPv4, IPv6, ICMP, BGP, OSPF    │
│ Link (Network       │ Layers 1, 2     │ Ethernet, WiFi (802.11), ARP   │
│ Access)             │                 │                                │
└─────────────────────┴─────────────────┴───────────────────────────────┘

WHY THE DIFFERENCE?
OSI was designed as a theoretical, complete framework (1984, by committee).
TCP/IP was designed practically, evolving from real implementation needs
(ARPANET, 1970s-80s) — it grew organically and "Application" absorbed
session/presentation concerns as needed by each individual protocol
(e.g., TLS handles both session and presentation concerns for HTTPS).

INTERVIEW TIP: When asked "which layer is X", interviewers often expect
OSI terminology for vocabulary (L4 load balancer, L7 firewall) but
understand the internet runs on TCP/IP's 4-layer model in practice.
```

---

## 7. L4 vs L7 Load Balancers — The Most Important Practical Application

This single concept gets asked in nearly every system design interview.

### Layer 4 Load Balancer

```
OPERATES AT: Transport layer (TCP/UDP)
SEES: Source IP, Source Port, Destination IP, Destination Port
DOES NOT SEE: HTTP headers, URLs, cookies, request body (it's encrypted
              anyway if HTTPS, and L4 LB doesn't decrypt)

HOW IT ROUTES:
"Connections to IP:Port X → forward to backend pool [A, B, C]"
Routing decision based on: round-robin, least connections, IP hash
ALL based on connection-level info, not content.

EXAMPLE: AWS Network Load Balancer (NLB), HAProxy in TCP mode, LVS (Linux 
Virtual Server)

      Client ──TCP SYN──▶ L4 LB ──TCP SYN──▶ Backend Server A
                          (just forwards packets,
                           doesn't terminate TCP)

PROS:
✅ Extremely fast (minimal processing — just packet forwarding)
✅ Can handle millions of connections (lower CPU overhead)
✅ Works for non-HTTP protocols (databases, game servers, raw TCP)
✅ Preserves client IP more easily

CONS:
❌ Cannot route based on URL path (/api vs /static)
❌ Cannot inspect or modify HTTP content
❌ Cannot do SSL termination meaningfully (just passes through)
❌ Cannot do content-based features (WAF rules, cookie-based routing)

USE WHEN: Extreme throughput needs, non-HTTP protocols, 
simple "spread load evenly" requirements
```

### Layer 7 Load Balancer

```
OPERATES AT: Application layer (HTTP/HTTPS)
SEES: Full HTTP request — method, URL path, headers, cookies, body
      (after TLS termination — LB decrypts, inspects, can re-encrypt)

HOW IT ROUTES:
"/api/*    → backend pool [API servers]"
"/static/* → backend pool [Static file servers] or redirect to CDN"
"/admin/*  → backend pool [Admin servers, with extra auth check]"
Cookie-based: "if cookie 'beta=true' → route to [Canary servers]"
Header-based: "if User-Agent contains 'Mobile' → route to [Mobile-optimized servers]"

EXAMPLE: AWS Application Load Balancer (ALB), Nginx, Envoy, Istio (service mesh),
HAProxy in HTTP mode

      Client ──HTTPS request──▶ L7 LB (terminates TLS, reads request)
                                    │
                  Decides based on URL/headers/cookies:
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              /api → API pool   /static → CDN   /admin → Admin pool

PROS:
✅ Content-based routing (microservices architecture!)
✅ SSL/TLS termination (offload encryption from backend servers)
✅ Can inject/modify headers, rewrite URLs
✅ WAF integration (inspect for SQL injection, XSS at LB level)
✅ A/B testing, canary deployments via cookie/header routing

CONS:
❌ More CPU overhead (TLS decryption, HTTP parsing)
❌ Slightly higher latency per request
❌ More complex configuration

USE WHEN: Microservices (route by path to different services),
need SSL termination, content-based routing, WAF requirements
```

### Combined Architecture (Common in Production)

```
Internet
   │
   ▼
┌──────────────────┐
│  L4 Load Balancer │  ← handles raw connection-level load distribution
│  (AWS NLB)        │     across multiple L7 LB instances (for HA)
└─────────┬─────────┘
          │
          ▼
┌──────────────────┐
│  L7 Load Balancer │  ← TLS termination, path-based routing,
│  (AWS ALB/Nginx)  │     WAF, cookie-based canary routing
└─────────┬─────────┘
          │
   ┌──────┼──────┬───────────┐
   ▼      ▼      ▼           ▼
[API]  [Web]  [Static]   [Admin]
```

---

## 8. Troubleshooting Network Issues Using OSI Layers

This is an extremely practical interview/real-world skill — debugging "systematically, bottom-up."

```
SCENARIO: "Users report they can't reach our API at api.company.com"

THE BOTTOM-UP DEBUGGING METHODOLOGY:

┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 1 (Physical) — Is there a connection at all?                    │
│ Check: Is the server instance running? Is the network interface up?  │
│ Tool: `ip link show` / cloud provider instance status                │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 2 (Data Link) — Can devices on the same network talk?           │
│ Check: ARP table correct? VLAN configuration correct?                 │
│ Tool: `arp -a`                                                        │
│ (Mostly relevant for on-premise / data center debugging)             │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 3 (Network) — Is the IP address reachable?                       │
│ Check: Can you ping the server's IP? Firewall/security group rules?  │
│ Tool: `ping api.company.com` or `ping 1.2.3.4`                       │
│       `traceroute api.company.com` (find where packets get dropped)  │
│                                                                       │
│ Result: "ping times out" → likely firewall blocking ICMP, or         │
│         routing issue, or server genuinely unreachable               │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 4 (Transport) — Is the specific PORT open?                       │
│ Check: Is the application listening on port 443?                      │
│ Tool: `telnet api.company.com 443` or `nc -zv api.company.com 443`   │
│                                                                       │
│ Result: "Connection refused" → nothing listening on that port,        │
│         or firewall blocking that specific port                       │
│ Result: "Connection timed out" → packet sent but no response          │
│         (firewall silently dropping, not rejecting)                   │
├─────────────────────────────────────────────────────────────────────┤
│ LAYER 7 (Application) — Is the HTTP server responding correctly?      │
│ Check: Does the application return correct HTTP response?            │
│ Tool: `curl -v https://api.company.com/health`                       │
│                                                                       │
│ Result: TCP connects but HTTP times out → application is hung/       │
│         overloaded (process running but not processing requests)     │
│ Result: HTTP 502/503/504 → load balancer can't reach backend, or     │
│         backend is unhealthy                                          │
│ Result: HTTP 200 but wrong data → application logic bug (not network!)│
└─────────────────────────────────────────────────────────────────────┘

KEY INSIGHT FOR INTERVIEWS: Always debug bottom-up. If Layer 3 (ping) 
fails, don't waste time debugging Layer 7 (application code) — the 
problem is more fundamental. This systematic approach demonstrates 
senior-level debugging maturity.
```

---

## 9. Critical Interview Vocabulary Table

```
┌────────────────────────────┬───────┬────────────────────────────────────────────────────────────┐
│ Term                       │ Layer │ Meaning & Why It Matters                                    │
├────────────────────────────┼───────┼────────────────────────────────────────────────────────────┤
│ MTU (Maximum Transmission  │ L2/3  │ Max size of a single frame/packet (Ethernet default: 1500   │
│ Unit)                      │       │ bytes). If data exceeds MTU, it's FRAGMENTED — extra        │
│                            │       │ overhead and potential performance issues. "Jumbo frames"   │
│                            │       │ (9000 bytes) used in data centers to reduce overhead.       │
├────────────────────────────┼───────┼────────────────────────────────────────────────────────────┤
│ ARP (Address Resolution    │ L2    │ Resolves IP address → MAC address for devices on the same   │
│ Protocol)                  │       │ network segment. "ARP spoofing" is a common LAN attack.      │
├────────────────────────────┼───────┼────────────────────────────────────────────────────────────┤
│ NAT (Network Address       │ L3    │ Translates private IPs (192.168.x.x) to a public IP. Allows  │
│ Translation)               │       │ many devices to share one public IP. Why your home network  │
│                            │       │ "looks like one device" to the internet.                     │
├────────────────────────────┼───────┼────────────────────────────────────────────────────────────┤
│ BGP (Border Gateway         │ L3    │ The protocol that makes the internet's routing work — ISPs  │
│ Protocol)                  │       │ announce "I own these IP ranges" to each other. THE basis    │
│                            │       │ of Anycast (how CDNs route to nearest PoP).                   │
├────────────────────────────┼───────┼────────────────────────────────────────────────────────────┤
│ VXLAN (Virtual Extensible  │ L2/3  │ Tunnels Layer 2 frames over Layer 3 (IP) networks. Used by   │
│ LAN)                       │       │ Kubernetes CNI plugins (Calico, Flannel) and cloud VPC overlay│
│                            │       │ networks to create virtual networks across physical hosts.   │
├────────────────────────────┼───────┼────────────────────────────────────────────────────────────┤
│ Subnet / CIDR              │ L3    │ A range of IP addresses, written as 192.168.1.0/24 (means    │
│                            │       │ first 24 bits are network, last 8 bits are host = 256        │
│                            │       │ addresses). Used heavily in VPC design (AWS/GCP/Azure).       │
├────────────────────────────┼───────┼────────────────────────────────────────────────────────────┤
│ Gateway / Default Gateway  │ L3    │ The router that a device sends traffic to when the           │
│                            │       │ destination is outside its local network.                    │
├────────────────────────────┼───────┼────────────────────────────────────────────────────────────┤
│ Socket                     │ L4    │ The combination of (IP address + port) — the actual          │
│                            │       │ programming abstraction used to send/receive data.            │
└────────────────────────────┴───────┴────────────────────────────────────────────────────────────┘
```

---

## 10. How OSI Connects to System Design

```
DEPENDENCY MAP — How OSI layers relate to system design topics:

┌─────────────────────────────────────────────────────────────────────┐
│ L7 (Application)                                                      │
│   ├── HTTP/HTTPS design choices → API design, caching, CDN rules     │
│   ├── WebSocket → real-time architecture                             │
│   ├── gRPC → microservices internal communication                    │
│   └── DNS → service discovery, traffic routing                       │
├─────────────────────────────────────────────────────────────────────┤
│ L4 (Transport)                                                        │
│   ├── TCP vs UDP choice → reliability vs latency tradeoffs           │
│   ├── L4 Load Balancers → high-throughput non-HTTP routing           │
│   ├── Connection pooling → database connections, HTTP keep-alive     │
│   └── Port management → microservice port allocation, K8s services  │
├─────────────────────────────────────────────────────────────────────┤
│ L3 (Network)                                                          │
│   ├── VPC/Subnet design → multi-region architecture                  │
│   ├── BGP/Anycast → CDN routing, global load balancing               │
│   ├── NAT → private subnet internet access (NAT gateways)            │
│   └── IP allocation → Kubernetes pod networking, service mesh        │
├─────────────────────────────────────────────────────────────────────┤
│ L2 (Data Link)                                                        │
│   ├── VLAN → network segmentation/isolation (security zones)         │
│   └── VXLAN → container networking overlays                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 11. Interview Quick-Fire Answers

**Q: What is the OSI model and why is it useful if the internet doesn't actually use it?**
A: The OSI model is a 7-layer conceptual framework for understanding networking, dividing concerns into physical transmission, local addressing, routing, transport, session, presentation, and application. While the actual internet uses the simpler 4-layer TCP/IP model, OSI provides a shared vocabulary ("Layer 4 load balancer", "Layer 7 firewall") used universally in industry, and its bottom-up structure provides a systematic debugging methodology.

**Q: What's the difference between a Layer 4 and Layer 7 load balancer?**
A: An L4 load balancer operates on TCP/UDP — it sees only IP addresses and ports, and forwards packets without understanding their content. It's extremely fast but can't make routing decisions based on URL paths, headers, or cookies. An L7 load balancer terminates HTTP/TLS, reads the actual request (URL, headers, cookies, body), and can route based on content — e.g., /api/* to one service, /static/* to another. L7 enables microservices routing, SSL offload, and WAF integration, at the cost of more CPU overhead.

**Q: Explain data encapsulation.**
A: As data moves down the OSI layers for transmission, each layer wraps ("encapsulates") the data from the layer above with its own header (and sometimes trailer). HTTP data gets a TCP header (becoming a segment), which gets an IP header (becoming a packet), which gets an Ethernet header and trailer (becoming a frame), which is finally transmitted as raw bits. The receiving side reverses this — each layer strips its corresponding header and passes the payload up.

**Q: What is BGP and why does it matter for CDNs?**
A: BGP (Border Gateway Protocol) is how autonomous networks (ISPs, large companies, CDNs) announce which IP address ranges they can route traffic to. This is the foundation of Anycast — a CDN can announce the SAME IP address from 300+ locations worldwide via BGP, and the internet's routing automatically sends each user's traffic to the topologically nearest announcing location, without any DNS-based geo-routing needed.

---
---

# APPENDIX: Cross-Topic Quick Reference

## Master Comparison — All Protocols at a Glance

```
┌─────────────┬───────┬────────────┬─────────────┬──────────────────────────┐
│ Protocol    │ Layer │ Transport  │ Connection  │ Primary Use Case          │
├─────────────┼───────┼────────────┼─────────────┼──────────────────────────┤
│ HTTP/1.1    │ L7    │ TCP        │ Per-request │ Legacy web, simple APIs   │
│ HTTP/2      │ L7    │ TCP        │ Persistent  │ Modern web (multiplexed)  │
│ HTTP/3      │ L7    │ QUIC (UDP) │ Persistent  │ Mobile, high packet-loss  │
│ WebSocket   │ L7    │ TCP        │ Persistent  │ Real-time bidirectional   │
│ gRPC        │ L7    │ HTTP/2     │ Persistent  │ Internal microservices    │
│ GraphQL     │ L7    │ HTTP       │ Per-request │ Flexible client queries   │
│ DNS         │ L7    │ UDP (mostly)│ Stateless  │ Name resolution           │
│ TCP         │ L4    │ —          │ Yes         │ Reliable transport        │
│ UDP         │ L4    │ —          │ No          │ Fast, lossy transport     │
│ TLS 1.3     │ L5/6  │ TCP/QUIC   │ Negotiated  │ Encryption layer           │
└─────────────┴───────┴────────────┴─────────────┴──────────────────────────┘
```

## A Complete Request — All Topics in One Flow

```
User types https://shop.com in browser:

1. DNS (Topic 2):     shop.com → resolve to IP via recursive resolver,
                       possibly Anycast-routed to nearest CDN PoP

2. CDN (Topic 7):     Edge server checks cache — if static assets, 
                       serve directly. If dynamic, forward to origin.

3. TCP (Topic 3):     3-way handshake establishes connection
                       (or QUIC handshake if HTTP/3)

4. TLS (Topic 1):     TLS 1.3 handshake — 1 RTT, certificate verification

5. HTTP/2 or 3        Multiplexed request sent: GET /
   (Topic 6):         HPACK-compressed headers

6. HTTP (Topic 1):    Server processes request, returns 200 OK with HTML

7. Browser parses HTML, discovers more resources (CSS/JS/images)
   → fetched via same multiplexed connection (Topic 6)
   → served from CDN cache (Topic 7)

8. App makes API calls (Topic 4):
   → REST for general CRUD
   → GraphQL if mobile app needs flexible queries
   → Internal services communicate via gRPC

9. Real-time updates (Topic 5):
   → WebSocket connection opened for live notifications/chat

10. Everything sits on top of OSI's Network/Transport/Link layers
    (Topic 8) — IP routing, Ethernet framing, physical transmission
```

---

## Final Study Tips

```
1. DRAW everything. If you can draw the TLS handshake, the TCP 
   handshake, and the OSI encapsulation diagram from memory, you 
   understand them deeply enough for interviews.

2. ALWAYS mention TRADEOFFS. Interviewers care less about "what is X"
   and more about "when would you choose X over Y, and why."

3. CONNECT topics. "We use HTTP/2 multiplexing AND a CDN AND DNS 
   geo-routing together" shows systems thinking, not memorized facts.

4. Know REAL NUMBERS where possible (Netflix's 95% cache hit rate, 
   QUIC's 1-RTT vs TCP's 3-RTT, HPACK's 90% header reduction). 
   Numbers make answers memorable and credible.

5. For BFSI/fintech interviews (relevant to your prep): pay extra 
   attention to idempotency (HTTP methods, Stripe's Idempotency-Key),
   TLS/mTLS for service-to-service auth, and reliable delivery 
   guarantees (TCP, gRPC streaming with ACKs) — these come up 
   constantly in payment system design discussions.
```
