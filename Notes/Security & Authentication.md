# Security & Authentication — Complete Deep-Dive Revision Guide
## System Design Interview Preparation | Product-Based Companies

---

**Prepared for:** Yash | AI/GenAI Engineer transitioning to Product Company System Design Interviews
**Coverage:** Authentication vs Authorization · JWT & OAuth 2.0 · SSO & SAML · Encryption · DDoS Protection · Zero Trust Architecture

---

## Table of Contents

1. **Authentication vs Authorization** — AuthN vs AuthZ, factors, RBAC/ABAC/ReBAC, session management
2. **JWT & OAuth 2.0** — JWT structure, HS256 vs RS256, OAuth flows, PKCE, OIDC, scopes
3. **SSO & SAML** — IdP/SP roles, SAML assertion flow, SAML vs OIDC, Single Logout, SCIM
4. **Encryption** — Symmetric vs asymmetric, TLS/forward secrecy, encryption at rest, KMS, envelope encryption, hashing
5. **DDoS Protection** — Attack types, layered defense, AWS Shield, application-layer DDoS, Cloudflare
6. **Zero Trust Architecture** — Never trust always verify, mTLS, service mesh, BeyondCorp, JIT access
7. **Appendix** — Cross-topic reference, complete secure system architecture, BFSI study tips

---

# Security & Authentication — Deep-Dive System Design Notes
### For Product-Based Company Interviews | Beginner → Advanced

---

> **How to use these notes:** Same structure as all previous guides.
> What is it → Why does it exist → How it works step by step → Diagrams → Internals
> → Tradeoffs → Real-world → Failures → Interview tips.
> Every concept is explained from scratch — no prior security knowledge assumed.

---

# TOPIC 1: Authentication vs Authorization

---

## 1. What Problem Do These Concepts Solve?

Every system that has users or services calling its APIs must answer two fundamental questions:

```
QUESTION 1: "WHO ARE YOU?"
→ This is AUTHENTICATION (AuthN)
→ Verifying identity: prove you are who you claim to be

QUESTION 2: "WHAT ARE YOU ALLOWED TO DO?"
→ This is AUTHORIZATION (AuthZ)
→ Verifying permissions: given I know who you are,
  what actions and resources can you access?

THESE ARE COMPLETELY SEPARATE CONCERNS:
A valid, authenticated user may be UNAUTHORIZED to access a resource.
An authorization system has NO MEANING without authentication first.

THE CLASSIC CONFUSION (401 vs 403 — recall Networking Fundamentals):
401 Unauthorized → MISLEADINGLY named. Actually means:
                   "Not Authenticated" — prove who you are.
403 Forbidden     → Correctly named. Means:
                   "Not Authorized" — I know who you are,
                   but you can't do this.
```

**Analogy:**
- Authentication = showing your passport at airport security ("I am Yash Sharma")
- Authorization = your boarding pass says you can enter Gate 23 and sit in Row 12B ("You may board this flight, in this seat")

---

## 2. Authentication — Deep Dive

### Authentication Factors

```
THE THREE FACTORS OF AUTHENTICATION:

Something you KNOW (knowledge factor):
  Password, PIN, security questions, passphrase
  WEAKNESS: Can be guessed, phished, leaked in breaches,
            reused across sites, keylogged

Something you HAVE (possession factor):
  Phone (OTP via SMS/authenticator app), hardware security
  key (YubiKey), smart card, email access (magic link)
  WEAKNESS: Device can be stolen, SIM can be swapped
            (SIM swapping attack on SMS OTP)

Something you ARE (inherence factor):
  Fingerprint, face recognition, iris scan, voice recognition
  WEAKNESS: Can't be changed if compromised (your fingerprint
            is permanent), spoofing attacks on poor implementations

MULTI-FACTOR AUTHENTICATION (MFA):
Requires 2 or more factors from DIFFERENT categories.
"Password + SMS OTP" = knowledge + possession = 2FA.
Even if password is leaked, attacker also needs the phone.
DRAMATICALLY reduces account takeover risk.

PHISHING-RESISTANT MFA:
SMS OTP can be phished (fake site captures the OTP in real-time
and replays it to the real site). 

PHISHING-RESISTANT alternatives:
- Hardware security keys (FIDO2/WebAuthn): cryptographic
  challenge-response tied to the specific domain — even if
  the user visits a phishing site, the key refuses to respond
  because the domain doesn't match.
- Passkeys (FIDO2 in software): same protocol, stored in
  device secure enclave / password manager. Becoming the
  industry standard (Apple, Google, Microsoft all support).
```

### Authentication Methods — The Spectrum

```
1. PASSWORD-BASED (most common, weakest):
   Client → sends {username, password}
   Server → hashes password (bcrypt/Argon2), compares to stored hash
   Server → returns session cookie or token

   STORAGE: NEVER store plaintext passwords. Always:
   - Hash with a SLOW, salted algorithm: bcrypt (cost factor 12),
     Argon2id (winner of Password Hashing Competition 2015),
     scrypt (memory-hard)
   - SALT prevents rainbow table attacks (precomputed hash lookups)
   - SLOW algorithms make brute force impractical

   WHY SLOW HASH FOR PASSWORDS?
   For file integrity (SHA-256): FAST is good (checking many files)
   For passwords: SLOW is good (brute-force tries millions/sec with
   fast hash; bcrypt limits to ~100/sec per core — massively raises
   the cost of cracking)

2. TOKEN-BASED (stateless — see JWT section):
   Client authenticates once → receives a TOKEN
   Every subsequent request carries the TOKEN in headers
   Server VERIFIES token (not looks it up in a DB) → stateless

3. CERTIFICATE-BASED (mTLS — mutual TLS):
   Both client AND server present X.509 certificates.
   Client proves identity via certificate (not username/password).
   Used for: service-to-service authentication in microservices
   (Zero Trust Architecture — Topic 6), API clients with high
   security requirements (banking APIs), IoT devices.

4. OAUTH / SOCIAL LOGIN (delegated authentication):
   "Login with Google / GitHub / LinkedIn"
   User authenticates to a THIRD PARTY (Google) who vouches
   for them to YOUR application. You never see the password.
   Full deep dive in Topic 2 (JWT & OAuth 2.0).

5. MAGIC LINKS / EMAIL OTP:
   User provides email → server sends a time-limited URL/OTP
   → user clicks link → authenticated. No password at all.
   Used by: Slack (for new device login), Notion, Linear.
   Advantage: no password to steal. Disadvantage: email
   account becomes the single point of failure.

6. PASSKEYS (FIDO2 / WebAuthn — the future):
   Cryptographic key pair: private key in device's secure
   enclave, public key on server.
   Authentication: server sends challenge → device signs with
   private key (requires biometric/PIN confirmation) → server
   verifies signature with public key.
   PHISHING RESISTANT: private key never leaves device,
   challenge response is domain-specific.
   PASSWORDLESS: no password to steal, phish, or forget.
```

---

## 3. Authorization — Deep Dive

### Authorization Models

```
MODEL 1: ACL (Access Control List)
List of specific users/groups with their specific permissions
for a specific resource.

Resource: /documents/financial-report-2026.pdf
ACL:
  yash@company.com → [READ, WRITE]
  priya@company.com → [READ]
  rahul@company.com → [READ, WRITE, DELETE]
  finance-team@company.com → [READ]

PROS: Granular, easy to understand for simple cases
CONS: Doesn't scale — managing ACLs for millions of resources
      and thousands of users becomes unmanageable.

MODEL 2: RBAC (Role-Based Access Control)
Users are assigned ROLES. Roles have PERMISSIONS.

Roles:
  admin: [CREATE, READ, UPDATE, DELETE] on ALL resources
  editor: [CREATE, READ, UPDATE] on content resources
  viewer: [READ] on public resources
  billing-admin: [READ, UPDATE] on billing resources

User → Role Assignment:
  yash → [admin, billing-admin]
  priya → [editor]
  rahul → [viewer]

PROS: Manageable at scale. "Assign user to role" is simple.
      Audit: "who has admin access?" is a single role lookup.
CONS: Role explosion — as permissions become more fine-grained,
      you need more and more roles. "Senior editor who can publish
      but not delete?" → another role. RBAC becomes unwieldy
      when permissions are highly contextual.

MODEL 3: ABAC (Attribute-Based Access Control)
Policies evaluate ATTRIBUTES of the user, resource, and context.

Policy example:
ALLOW if:
  user.department == resource.department AND
  user.clearanceLevel >= resource.classificationLevel AND
  request.time >= "09:00" AND request.time <= "18:00" AND
  request.ipCountry == "IN"

PROS: Extremely flexible. Can express complex context-sensitive rules.
CONS: Complex to implement, audit, and debug. Policies become hard
      to understand. More suitable for government/military/enterprise.
      AWS IAM is essentially ABAC.

MODEL 4: ReBAC (Relationship-Based Access Control)
Authorization is based on GRAPH RELATIONSHIPS between users
and resources.

Google's Zanzibar (the authorization system behind Google Docs):
"Can user Yash READ document 456?"
→ Is Yash a direct viewer of 456? No.
→ Is Yash a member of group that is viewer of 456? No.
→ Is Yash a viewer of a folder that contains 456? YES!
   (folder:2 viewer includes yash, document:456 is in folder:2)
→ ALLOW

PROS: Natural for hierarchical/nested resources (folders, orgs,
      repos). Efficient for "who has access?" lookups via graph.
CONS: Complex query execution (graph traversal). Requires careful
      schema design.
USED BY: Google Drive/Docs (Zanzibar), GitHub (repo permissions),
         Dropbox, Airbnb (OpenFGA is the open-source implementation).
```

### The Authorization Decision Flow

```
Every protected resource access:

Request arrives:
  User: yash@company.com
  Action: DELETE
  Resource: /documents/financial-report-2026.pdf
  Context: time=14:30, IP=203.0.113.5, country=IN

Authorization service checks policy:
1. Who is the user? (resolved from JWT/session)
2. What roles/attributes does this user have?
3. What policies apply to this resource + action?
4. Does any policy ALLOW this? (if no ALLOW → implicit DENY)
5. Does any policy explicitly DENY this? (explicit DENY overrides ALLOW)

Decision:
  ALLOW → request proceeds to the resource
  DENY  → 403 Forbidden returned immediately

SEPARATION OF CONCERNS (the standard pattern):
- Application code should NOT contain authorization logic inline
  ("if user.role == 'admin' and resource.type == 'financial'...")
- EXTERNALIZE authorization to a dedicated policy engine:
  OPA (Open Policy Agent) — REGO language, K8s native
  Casbin — multi-model authorization library
  OpenFGA — Google Zanzibar open implementation
  AWS IAM — for AWS resources
  Azure RBAC — for Azure resources

WHY EXTERNALIZE?
- Centralized audit: all authorization decisions logged in one place
- Single point of update: change a policy → applies everywhere
- Testable: authorization policies become code with unit tests
```

---

## 4. Session Management — Stateful vs Stateless

```
STATEFUL SESSIONS (traditional, server-side):

1. User authenticates → server creates SESSION in DB/Redis:
   session_id: "abc123xyz"
   data: {user_id: 456, roles: ["admin"], created: "2026-06-14", expires: "2026-06-15"}

2. Server sets cookie: Set-Cookie: session_id=abc123xyz; HttpOnly; Secure; SameSite=Strict

3. Browser sends cookie on every request:
   Cookie: session_id=abc123xyz

4. Server LOOKS UP session_id in DB/Redis for every request
   → Validates it exists and hasn't expired
   → Retrieves user context

PROS: Immediate revocation (delete from Redis → user instantly logged out)
CONS: Requires a shared session store (Redis) accessible by ALL servers
      → adds latency (Redis round trip on every request)
      → Redis becomes a dependency/bottleneck/SPOF if not clustered

STATELESS TOKENS (modern — JWT — see Topic 2):
No server-side session storage. Token itself contains user data.
Server just VERIFIES the token's signature on every request.

PROS: No shared session store needed. Horizontally scalable.
      Tokens can be used across multiple services (microservices).
CONS: Cannot be immediately revoked without additional infrastructure
      (token is valid until expiry, even if "logged out").
      Token size adds overhead to every request header.

HYBRID (best of both):
Use short-lived JWTs (15-minute expiry) + long-lived refresh tokens.
Revocation: maintain a small Redis "blocklist" of revoked JWT IDs
(jti claim). Check blocklist only on refresh, not on every API call.
Most production systems use this hybrid approach.
```

---

## 5. Real-World Usage

**Google:** Uses Zanzibar for authorization across all Google products. A user's access to a Google Doc is computed from a graph of user→group→folder→document relationships. Zanzibar serves millions of authorization decisions per second with ~5ms p99 latency.

**GitHub:** RBAC for repositories (owner, collaborator, triage, write, maintain, admin roles), with repository-level permissions inherited from organization membership. Actions like "allow user to push to main branch" require specific role assignments checked on every git push.

**AWS IAM:** A sophisticated ABAC system. Every AWS API call includes the identity (IAM user, role, or service) and the requested action (s3:GetObject, rds:DescribeDBInstances). IAM evaluates all applicable policies (identity-based, resource-based, permission boundaries, SCPs in Organizations) and either allows or explicitly denies. "Deny always wins" — an explicit Deny anywhere overrides any number of Allows.

---

## 6. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Privilege escalation: regular    │ Authorization logic has a flaw;  │ Defense in depth: multiple auth   │
│ user accesses admin resource     │ role check missing; IDOR         │ checks; principle of least       │
│                                  │ (Insecure Direct Object          │ privilege; test with role-specific│
│                                  │ Reference — accessing            │ automated tests; use externalized │
│                                  │ /api/users/123 without           │ authorization (OPA) not inline    │
│                                  │ checking ownership)              │ code                              │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Password stored in plaintext     │ Developer used MD5 or no hash    │ Use bcrypt/Argon2id; automated    │
│ (data breach → all accounts      │                                  │ security testing; OWASP Top 10   │
│ immediately compromised)         │                                  │ training for developers           │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Session fixation attack           │ Server reuses the same session   │ ALWAYS regenerate session ID      │
│ (attacker sets victim's session  │ ID before and after auth         │ upon successful authentication;   │
│ ID before login)                 │                                  │ invalidate old session ID         │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Broken access control: API       │ Server-side authorization        │ NEVER trust client-sent role      │
│ trusts client-sent role claims   │ trusts role in request body      │ claims; enforce authorization     │
│ ("role=admin" in request)        │ or query parameter               │ based ONLY on verified session/   │
│                                  │                                  │ JWT / server-side session         │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 7. Interview Quick-Fire Answers

**Q: What's the difference between authentication and authorization?**
A: Authentication (AuthN) verifies IDENTITY — "who are you?" The system checks credentials (password, token, certificate) to confirm you are who you claim. Authorization (AuthZ) verifies PERMISSIONS — "what are you allowed to do?" Given I know who you are, can you perform this action on this resource? They're sequential: authentication must succeed before authorization is meaningful. The HTTP status codes reflect this: 401 means "not authenticated" (prove yourself), 403 means "authenticated but not authorized" (I know who you are, but you can't do this).

**Q: Compare RBAC and ABAC — when would you choose each?**
A: RBAC (Role-Based) assigns users to roles, and roles have permissions. It's simple, auditable, and scales well for clear organizational hierarchies ("admin can do everything, editor can write, viewer can read"). Choose RBAC for most web applications and organizational systems. ABAC (Attribute-Based) evaluates dynamic attributes of user, resource, and context in policy rules. It's more flexible (time-of-day restrictions, department-based access, clearance levels) but more complex to implement and audit. Choose ABAC when permissions are highly contextual and can't be captured in a fixed set of roles — AWS IAM is a real-world ABAC example.

**Q: What is the principle of least privilege?**
A: Every user, service, or system component should have the MINIMUM permissions necessary to perform its function — nothing more. A service that only reads from a database should have read-only credentials, not full read-write-delete access. A Lambda function that writes to one S3 bucket should only have s3:PutObject permission on that specific bucket, not s3:* on all buckets. Least privilege limits the blast radius when credentials are compromised — an attacker with stolen read-only credentials can't delete data.

---
---

# TOPIC 2: JWT & OAuth 2.0

---

## 1. What Problem Does JWT Solve?

After a user logs in (authentication), every subsequent API request needs to prove "I am the authenticated user from earlier." Two approaches:

```
APPROACH 1: STATEFUL SESSIONS (server-side)
Server stores session data in Redis. Every request → Redis lookup.
Problems at scale: Redis round trip on every request, Redis
must be shared across all app servers.

APPROACH 2: STATELESS TOKENS — JWT
The server signs a token containing user identity data.
Client sends this token with every request.
Server VERIFIES the signature (cryptographic math, no DB lookup).
State lives IN THE TOKEN, not on the server.

JWT is a standard (RFC 7519) for creating these tokens.
```

---

## 2. JWT Structure — Every Bit Explained

```
A JWT looks like this (three Base64URL-encoded parts, separated by dots):

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJ1c2VyX2lkIjoiMTIzIiwicm9sZXMiOlsiYWRtaW4iXSwiZW1haWwiOiJ5YXNoQGNvLmNvbSIsImlhdCI6MTcxODAwMDAwMCwiZXhwIjoxNzE4MDAwOTAwfQ
.
[signature bytes]

PART 1: HEADER (Base64URL-decoded):
{
  "alg": "RS256",    ← signing algorithm (RS256 = RSA + SHA256)
  "typ": "JWT"       ← token type
}

PART 2: PAYLOAD / CLAIMS (Base64URL-decoded):
{
  "user_id": "123",
  "roles": ["admin"],
  "email": "yash@company.com",
  "iat": 1718000000,  ← issued at (Unix timestamp)
  "exp": 1718000900,  ← expires at (15 minutes later)
  "jti": "a3f8b2c1",  ← JWT ID (unique, for revocation checks)
  "iss": "auth.company.com",  ← issuer
  "aud": "api.company.com"    ← audience (intended recipient)
}

PART 3: SIGNATURE:
base64url(sign(algorithm, header + "." + payload, secret/private_key))

CRITICAL: BASE64URL IS NOT ENCRYPTION!
The header and payload are READABLE BY ANYONE who has the JWT.
They're only BASE64URL-ENCODED (reversible, no key needed).
NEVER put sensitive data (SSN, card numbers, passwords) in a JWT payload.
The SIGNATURE only proves the payload HASN'T BEEN TAMPERED WITH —
it doesn't hide the contents.
```

### JWT Signing Algorithms — HS256 vs RS256

```
HS256 (HMAC + SHA256) — SYMMETRIC:
  ONE shared secret key for BOTH signing AND verification.
  SIGN: signature = HMAC_SHA256(header.payload, secret)
  VERIFY: recalculate HMAC, compare to signature in token

  PROBLEM: Any service that needs to VERIFY tokens must know
  the secret. If you have 20 microservices all verifying tokens,
  all 20 must have the secret. A compromise of ANY ONE leaks
  the signing capability — an attacker can forge tokens!

RS256 (RSA + SHA256) — ASYMMETRIC:
  PRIVATE KEY: used ONLY by the auth server to SIGN tokens.
               Must be kept secret on the auth server.
  PUBLIC KEY:  distributed freely to ALL services that need to VERIFY tokens.
               Even if all 20 microservices have the public key,
               they CANNOT forge tokens (can't sign without the private key).

  SIGN: signature = RSA_SIGN(header.payload, private_key)
  VERIFY: RSA_VERIFY(signature, header.payload, public_key) → true/false

RULE: Use RS256 (or ES256 — elliptic curve, same asymmetric principle)
for production systems with multiple services verifying tokens.
HS256 only when you have a single service doing everything.

KEY ROTATION:
RS256 enables safe key rotation: publish new public key alongside old
(via JWKS endpoint — JSON Web Key Set), sign new tokens with new private
key, allow verification of old tokens with old public key until they
expire. Then decommission old key. Zero downtime rotation.
```

---

## 3. JWT Security — Common Vulnerabilities

```
ATTACK 1: "alg: none" VULNERABILITY (historical, now mostly fixed)
Early JWT libraries had a bug where setting alg="none" in the header
caused the library to accept tokens WITHOUT verifying the signature.

Attacker: Takes a valid JWT, decodes payload, changes user_id to admin's,
          sets alg="none", removes the signature.
Vulnerable library: "alg is none, no signature check needed" → accepts it!

MITIGATION: ALWAYS explicitly specify the EXPECTED algorithm in your
verification code. Never accept "none". Modern libraries fixed this.

ATTACK 2: WEAK SECRETS for HS256
If the HS256 secret is short or guessable (e.g., "secret123"), an
attacker can brute-force it offline (no rate limiting — they have the
token, can try trillions of secrets locally).

MITIGATION: HS256 secrets must be cryptographically random, at least 256 bits.
Better: use RS256 (no secret to brute-force).

ATTACK 3: NOT VALIDATING CLAIMS
A JWT can be STRUCTURALLY VALID (signature correct) but still be:
- Expired (exp claim is in the past)
- For a different audience (aud claim doesn't match your service)
- Issued by a different issuer (iss claim not trusted)

Lazy implementations skip these checks: "signature is valid, must be fine."

MITIGATION: ALWAYS validate: exp (not expired), iss (trusted issuer),
aud (intended for this service), iat (not issued in the future).

ATTACK 4: JWT in localStorage (XSS)
Storing JWT in JavaScript's localStorage → accessible to ANY script
running on the page. An XSS attack (injecting malicious script into
the page) can steal the JWT.

MITIGATION: Store JWT in HttpOnly cookies (not accessible to JavaScript at all).
Or use short-lived JWTs in memory (lost on page refresh — requires
re-auth flow on reload, but secure).

ATTACK 5: JWT REPLAY after logout
User logs out. JWT still valid until expiry (15 min). Attacker who
captured the token can still use it for 15 minutes.

MITIGATION:
- Short expiry (15 minutes) limits the window
- Maintain a REVOCATION LIST (blocklist) in Redis for logged-out tokens:
  SADD revoked_tokens {jti_claim}
  On each request: SISMEMBER revoked_tokens {token_jti}
  → But this adds a Redis lookup on every request (removes statelessness benefit!)
- Better: short-lived access tokens (15 min) + long-lived refresh tokens
  (stored securely, can be revoked immediately in DB)
```

---

## 4. OAuth 2.0 — Delegated Authorization

### The Problem OAuth Solves

```
SCENARIO WITHOUT OAUTH:
You want to use app "TripPlanner" which needs to read your
Google Calendar to suggest times for trips.

NAIVE APPROACH: Give TripPlanner your Google password.
PROBLEMS:
  - TripPlanner now has your Google password for EVERYTHING
  - If TripPlanner is hacked, your Google account is compromised
  - TripPlanner could read your Gmail, Drive, anything
  - You can't revoke TripPlanner's access without changing your password

OAUTH 2.0 SOLUTION:
TripPlanner gets a LIMITED ACCESS TOKEN that allows ONLY reading
your calendar — not your email, not your files, not your contacts.
You grant this through Google's consent screen — TripPlanner never
sees your Google password. You can REVOKE the token at any time
from Google's security settings without changing your password.
```

### The Roles in OAuth 2.0

```
RESOURCE OWNER: The USER who owns the protected data (you).

CLIENT: The APPLICATION requesting access (TripPlanner).
  NOT to be confused with the browser! This is the app.

AUTHORIZATION SERVER: Issues access tokens (Google's auth server).
  Authenticates the user, obtains consent, issues tokens.

RESOURCE SERVER: The API protecting the user's data (Google Calendar API).
  Validates access tokens presented by the client.

SCOPE: The specific permissions being requested.
  "calendar.readonly" = only read calendar, nothing else.
```

### The Authorization Code Flow — The Main Flow

```
MOST SECURE and MOST COMMON flow. Used for:
- Web applications with a backend server
- Mobile apps (with PKCE — Proof Key for Code Exchange)

STEP 1: Client redirects user to Authorization Server
Browser → GET https://accounts.google.com/o/oauth2/auth
  ?client_id=tripplanner_client_id
  &redirect_uri=https://tripplanner.com/callback
  &response_type=code
  &scope=https://www.googleapis.com/auth/calendar.readonly
  &state=random_csrf_token_abc123    ← CSRF protection

STEP 2: User authenticates and consents
Google shows: "TripPlanner wants to read your Calendar. Allow?"
User logs in (if not already) and clicks "Allow".

STEP 3: Authorization Server returns Authorization Code
Browser → redirects to:
https://tripplanner.com/callback
  ?code=AUTHORIZATION_CODE_xyz        ← short-lived (10 minutes), single-use
  &state=random_csrf_token_abc123     ← verify matches what we sent!

TripPlanner backend verifies state matches → CSRF check passed.

STEP 4: Client exchanges Code for Tokens (server-to-server, not browser!)
TripPlanner backend → POST https://accounts.google.com/o/oauth2/token
  Body:
  client_id=tripplanner_client_id
  client_secret=tripplanner_secret    ← backend has the secret, browser doesn't!
  code=AUTHORIZATION_CODE_xyz
  grant_type=authorization_code
  redirect_uri=https://tripplanner.com/callback

STEP 5: Authorization Server returns tokens
{
  "access_token": "ya29.xxx",        ← use to call Google Calendar API
  "token_type": "Bearer",
  "expires_in": 3600,                ← valid 1 hour
  "refresh_token": "1//xxx",         ← use to get new access tokens
  "scope": "https://www.googleapis.com/auth/calendar.readonly"
}

STEP 6: Client calls Resource Server with Access Token
TripPlanner → GET https://www.googleapis.com/calendar/v3/events
Authorization: Bearer ya29.xxx

STEP 7: Refresh Token Flow (when access token expires)
TripPlanner → POST https://accounts.google.com/o/oauth2/token
  grant_type=refresh_token
  refresh_token=1//xxx
  client_id=...
  client_secret=...
→ Returns new access_token (without user interaction!)

WHY THE TWO-STEP (code → tokens)?
The authorization code travels through the BROWSER (URL redirect).
If the browser's history, HTTP referrer header, or network was
compromised, only the SHORT-LIVED, SINGLE-USE code is exposed.
The TOKENS are exchanged server-to-server (never touch the browser),
requiring the client_secret to complete. This protects the tokens.
```

### PKCE — OAuth for Mobile and SPAs

```
Mobile apps and Single Page Applications (SPAs) CANNOT safely store
a client_secret (the code is on the user's device — a determined
attacker can extract it from the app binary).

PKCE (Proof Key for Code Exchange, RFC 7636) replaces the client_secret
with a cryptographic challenge:

STEP 1: App generates a random string (code_verifier):
  code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

STEP 2: App hashes it to get code_challenge:
  code_challenge = base64url(SHA256(code_verifier))
  code_challenge = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

STEP 3: Send code_challenge in the auth request (step 1 of main flow)
  ?code_challenge=E9Melhoa2...
  &code_challenge_method=S256

STEP 4: When exchanging the code for tokens, send code_verifier:
  code_verifier=dBjftJeZ4CVP...
  (Auth server verifies: SHA256(code_verifier) == code_challenge from step 3)

WHY THIS IS SECURE:
An attacker who intercepts the authorization CODE can't exchange it
for tokens because they don't have the code_verifier (it was never
transmitted — only its SHA256 hash was). The app proves it initiated
the flow by revealing the original verifier.
```

### OAuth Scopes — The Permission Boundary

```
Scopes define EXACTLY what the access token can do.
The principle of least privilege (Topic 1) applied to OAuth.

Google Calendar API scopes:
calendar.readonly         → read only, all calendars
calendar                  → read + write, all calendars
calendar.events           → read + write events only
calendar.events.readonly  → read events only

REQUESTING MINIMAL SCOPES:
"TripPlanner only needs to see your free/busy times,
not the event titles and details."

→ Request ONLY calendar.freebusy scope, not calendar.readonly
→ User sees a less scary consent screen
→ Less data exposed if TripPlanner is compromised
→ Google can detect if TripPlanner requests more than claimed

INCREMENTAL AUTHORIZATION:
Request only what you need IMMEDIATELY. Ask for more scopes
when the user actually tries to use that feature.
Google's best practice for better user experience.
```

---

## 5. OpenID Connect (OIDC) — Authentication on Top of OAuth

```
OAuth 2.0 is an AUTHORIZATION framework (access to resources).
But developers started using it for AUTHENTICATION too ("Login with Google").
PROBLEM: OAuth access tokens don't tell you WHO THE USER IS —
they just say "this token can access calendar.readonly."

OpenID Connect (OIDC) extends OAuth 2.0 to add AUTHENTICATION:
Same flow as OAuth, but with an additional ID TOKEN (a JWT)
that contains user identity claims.

ID TOKEN (JWT) payload:
{
  "iss": "https://accounts.google.com",
  "sub": "110248495820838343983",   ← stable unique user ID (use this, not email!)
  "email": "yash@company.com",
  "email_verified": true,
  "name": "Yash Sharma",
  "picture": "https://lh3.googleusercontent.com/...",
  "iat": 1718000000,
  "exp": 1718003600,
  "aud": "tripplanner_client_id"    ← this token is FOR TripPlanner
}

ACCESS TOKEN: "I can access these Google APIs on behalf of this user"
ID TOKEN: "This is who the user IS (authenticated by Google)"

OIDC + OAuth together:
- Use ACCESS TOKEN to call APIs (calendar, etc.)
- Use ID TOKEN to know who the user is (for your own DB/session)
- Use sub claim (not email!) as the stable user identifier
  (emails can change; Google's sub is permanent for that user)

OIDC is what powers "Sign in with Google/Apple/Microsoft" —
you're using OAuth infrastructure but getting authentication information.
```

---

## 6. Real-World Usage

**Stripe:** Uses OAuth 2.0 for "Connect" — allowing merchants to authorize Stripe to charge their customers on their behalf, or allowing platforms to act on behalf of their connected merchant accounts. Access tokens are tied to specific scopes (read-only reports, full payment processing, etc.) and can be revoked instantly.

**GitHub:** Uses OAuth 2.0 with fine-grained personal access tokens (PATs). Scopes control exactly which repositories and actions a token can perform. GitHub's new fine-grained PATs can be restricted to specific repositories, specific branches, and specific read/write permissions per resource type.

**Okta/Auth0 (Identity Providers):** Implement the authorization server role — companies integrate Okta as their identity provider. Applications redirect to Okta for authentication (OIDC) and authorization (OAuth 2.0). Okta manages user accounts, MFA, SSO, and token issuance centrally — removing this complexity from individual applications.

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ JWT secret leaked → attacker     │ HS256 secret in environment      │ Rotate to RS256 (no shared       │
│ can forge ANY token              │ variable, logged, in git repo    │ secret); use secrets manager;    │
│                                  │                                  │ rotate keys immediately on leak  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ OAuth CSRF attack (state not     │ Client doesn't generate or       │ ALWAYS generate random state     │
│ validated → attacker tricks       │ validate state parameter         │ parameter per request; verify    │
│ victim into linking attacker's    │                                  │ on callback BEFORE exchanging    │
│ account)                         │                                  │ the code                          │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Open redirect in OAuth callback   │ redirect_uri not strictly        │ Register exact redirect URIs     │
│ (attacker steals auth code)      │ validated by auth server;        │ with auth server; reject any     │
│                                  │ accepts redirects to             │ URI not on the registered list   │
│                                  │ attacker-controlled domains      │                                   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Long-lived access tokens stolen   │ Access tokens valid for hours/   │ Short-lived access tokens (15-60 │
│ → persistent unauthorized access │ days; no revocation mechanism    │ min); refresh tokens with         │
│                                  │                                  │ revocation capability; token      │
│                                  │                                  │ binding to device/session         │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: What is a JWT and why is it called "stateless"?**
A: A JWT (JSON Web Token) is a self-contained token that encodes user identity and claims directly in the token itself (header.payload.signature). It's stateless because the server doesn't need to look up session data — it VERIFIES the cryptographic signature (using the public key for RS256 or shared secret for HS256) to confirm the payload hasn't been tampered with. Any server with the verification key can validate any JWT without a database call, enabling horizontal scaling without a shared session store.

**Q: What's the difference between OAuth 2.0 and OpenID Connect?**
A: OAuth 2.0 is an AUTHORIZATION framework — it answers "what can this application do on behalf of this user?" It issues access tokens that grant specific API permissions (scopes). OpenID Connect (OIDC) is an AUTHENTICATION layer built on top of OAuth 2.0 — it adds an ID Token (a JWT) that answers "who is this user?" and provides verified identity claims (sub, email, name). Use OAuth for API authorization; use OIDC (which includes OAuth) for "Sign in with Google/GitHub" scenarios where you need both identity AND API access.

**Q: Why is the Authorization Code flow preferred over Implicit flow?**
A: The Implicit flow (now deprecated in OAuth 2.1) returned the access token directly in the URL fragment after redirect — exposing it in browser history, HTTP referrer headers, and potentially to malicious JavaScript on the page. The Authorization Code flow returns only a short-lived, single-use CODE in the redirect URL. The actual token exchange happens server-to-server (backend to auth server), requiring the client_secret — which never touches the browser. If the code is intercepted, it's useless without the client_secret. For mobile/SPAs without a backend, PKCE provides similar protection without requiring a client_secret.

**Q: What is PKCE and when is it required?**
A: PKCE (Proof Key for Code Exchange) is required for OAuth clients that CAN'T securely store a client_secret — specifically mobile apps and single-page applications where the code is on the user's device and can potentially be extracted. PKCE uses a cryptographic challenge: the client generates a random code_verifier, sends SHA256(code_verifier) as the code_challenge with the auth request, and then proves it initiated the flow by revealing the original code_verifier when exchanging the authorization code for tokens. An intercepted code is useless without the verifier, which was never transmitted.


---
---

# TOPIC 3: SSO & SAML

---

## 1. What Problem Does SSO Solve?

A typical enterprise employee uses dozens of applications: Gmail, Salesforce, Slack, Jira, GitHub, Confluence, AWS Console, HR portal, expense system. Without SSO, each application has its OWN username and password — a nightmare for both users AND IT administrators.

```
WITHOUT SSO:
User has 20 different accounts:
  - Different passwords (or same password everywhere — security risk!)
  - HR system: jira.company.com has yash.sharma@company.com
  - Slack: yash@company.com
  - AWS: yash.sharma
  - Salesforce: yash_sharma@company.com
  → Password reset flood when policies change
  → When employee leaves: IT must manually revoke 20 accounts
    (often miss some → ex-employee still has access!)
  → Each app independently manages identity → security audit nightmare

WITH SSO:
ONE login (Identity Provider / IdP, e.g., Okta, Azure AD)
grants access to ALL applications.
  → User logs in ONCE, accesses everything
  → When employee leaves: disable ONE account in IdP → instantly
    locked out of ALL applications simultaneously
  → Centralized audit log of all logins across all applications
  → Enforce MFA in ONE place → applies to all applications
```

**Analogy:** SSO is like a hotel key card. One card (your IdP identity) unlocks your room, the gym, the parking garage, the business center — all without separate keys for each. The front desk (IdP) issued the card; each facility (application) trusts the card without needing to know your personal details.

---

## 2. How SSO Works — Core Concepts

```
THE THREE PARTIES:

IDENTITY PROVIDER (IdP): The authority on user identity.
  Examples: Okta, Azure Active Directory, Google Workspace,
            OneLogin, Auth0, Ping Identity, AWS IAM Identity Center
  Stores: user accounts, passwords, group memberships, attributes
  Responsibilities: authenticate users, issue tokens/assertions,
                    enforce MFA, manage sessions

SERVICE PROVIDER (SP): The application the user wants to access.
  Examples: Salesforce, Slack, GitHub, your custom SaaS app
  Does NOT store passwords — delegates authentication to IdP
  Trusts assertions from the IdP about who the user is

USER (Principal): The human trying to access a Service Provider

THE TRUST RELATIONSHIP:
SP and IdP must establish trust BEFORE SSO works:
  - SP registers with IdP (gets entity ID, certificate)
  - IdP issues SP a signing certificate to verify assertions
  - Both configure the other's metadata
  This is done ONCE during setup, not per-user.
```

---

## 3. SAML 2.0 — The Enterprise SSO Standard

SAML (Security Assertion Markup Language) is an XML-based open standard for exchanging authentication and authorization data between an IdP and an SP. It's the dominant protocol in enterprise SSO (especially for legacy enterprise applications).

### SAML SP-Initiated Flow (Most Common)

```
USER              SERVICE PROVIDER (Salesforce)    IDENTITY PROVIDER (Okta)
  │                        │                               │
  │─ Access salesforce.com─▶│                              │
  │                        │                               │
  │                        │(user not authenticated)       │
  │                        │Generate SAML AuthnRequest:    │
  │                        │<samlp:AuthnRequest            │
  │                        │  ID="abc123"                  │
  │                        │  Destination="https://okta/sso"│
  │                        │  AssertionConsumerServiceURL= │
  │                        │  "https://salesforce.com/acs" │
  │                        │  />                           │
  │                        │                               │
  │◀─ Redirect to Okta ────│                               │
  │   (with base64-encoded  │                              │
  │   AuthnRequest in URL)  │                              │
  │                        │                               │
  │─ GET https://okta/sso ─────────────────────────────────▶│
  │◀─ Login page ───────────────────────────────────────────│
  │─ POST {username,pw,OTP}────────────────────────────────▶│
  │                        │(authenticate user, check MFA) │
  │                        │                               │
  │                        │     Generate SAML Assertion:  │
  │                        │<samlp:Response>               │
  │                        │  <Assertion>                  │
  │                        │    <Subject>                  │
  │                        │      yash@company.com         │
  │                        │    </Subject>                 │
  │                        │    <AttributeStatement>       │
  │                        │      <Attribute Name="groups">│
  │                        │        <Value>sales-team</V>  │
  │                        │      </Attribute>             │
  │                        │    </AttributeStatement>      │
  │                        │    <Conditions NotBefore=..   │
  │                        │       NotOnOrAfter=../>       │
  │                        │  </Assertion>                 │
  │                        │  <Signature>...</Signature>   │
  │                        │</samlp:Response>              │
  │                        │                               │
  │◀──── Redirect back ─────────────────────────────────────│
  │  POST form with SAML   │                               │
  │  Response (base64) to  │                               │
  │  salesforce.com/acs    │                               │
  │                        │                               │
  │─ POST /acs ────────────▶│                              │
  │  SAMLResponse=base64   │                               │
  │                        │Verify Okta's signature        │
  │                        │Check: NotBefore/NotOnOrAfter  │
  │                        │Check: Audience matches SF     │
  │                        │Extract: user email, groups    │
  │                        │Create local session for user  │
  │◀─ Redirect to dashboard│                               │
  │                        │                               │
```

### SAML Assertion — The Security Token

```
The SAML Assertion is the XML document that the IdP signs and delivers.
It contains three types of statements:

1. AUTHENTICATION STATEMENT: "This user authenticated at this time by this method"
   <AuthnStatement AuthnInstant="2026-06-14T10:30:00Z">
     <AuthnContext>
       <AuthnContextClassRef>
         urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
         (means: password + HTTPS)
       </AuthnContextClassRef>
     </AuthnContext>
   </AuthnStatement>

2. ATTRIBUTE STATEMENT: "This user has these attributes"
   <AttributeStatement>
     <Attribute Name="email">
       <AttributeValue>yash@company.com</AttributeValue>
     </Attribute>
     <Attribute Name="groups">
       <AttributeValue>sales-team</AttributeValue>
       <AttributeValue>admin</AttributeValue>
     </Attribute>
     <Attribute Name="department">
       <AttributeValue>Engineering</AttributeValue>
     </Attribute>
   </AttributeStatement>

3. AUTHORIZATION STATEMENT (less common): "This user is authorized to do X"

SIGNATURE:
The entire assertion is SIGNED using the IdP's private key.
The SP verifies using the IdP's public certificate.
Prevents: any tampering with the assertion in transit.
(Even if someone intercepts the POST form and modifies the assertion,
the signature won't match → SP rejects it.)
```

---

## 4. SAML vs OAuth/OIDC — When to Use Each

```
┌──────────────────────────┬────────────────────────────┬────────────────────────────┐
│ Factor                    │ SAML 2.0                    │ OAuth 2.0 + OIDC            │
├──────────────────────────┼────────────────────────────┼────────────────────────────┤
│ Format                    │ XML (verbose, complex)       │ JSON (compact, modern)      │
│ Primary use               │ Enterprise SSO, federated    │ API authorization, social   │
│                          │ identity (B2B)               │ login, mobile apps          │
│ Token format              │ XML Assertion (large)        │ JWT (compact, readable)     │
│ Transport                 │ HTTP POST (browser-based)    │ HTTP redirect + API calls   │
│ Service-to-service?       │ Not designed for it          │ Yes (client credentials     │
│                          │                              │ flow, machine-to-machine)   │
│ Mobile support            │ Poor (browser-dependent)     │ Excellent (PKCE for mobile) │
│ Modern web apps (SPA)     │ Awkward                      │ Natural fit                 │
│ Enterprise B2B SSO        │ Dominant (nearly universal)  │ Growing adoption            │
│ Typical IdPs              │ Okta, Azure AD, ADFS,        │ Google, GitHub, Auth0,      │
│                          │ PingFederate                  │ Okta (supports both)        │
│ Complexity                │ HIGH (XML, certificates,     │ MEDIUM (JSON, REST-based,   │
│                          │ metadata exchange)            │ broader tooling)            │
└──────────────────────────┴────────────────────────────┴────────────────────────────┘

RULE OF THUMB:
- Enterprise/corporate SSO with existing infrastructure → likely SAML
- New app integrating with consumer identity (Google, GitHub) → OIDC
- API-to-API authorization → OAuth 2.0 client credentials
- Building new identity infrastructure → OIDC + OAuth 2.0 (modern stack)
- Many IdPs (Okta, Azure AD) support BOTH SAML and OIDC simultaneously
```

---

## 5. SSO Session Management — Single Logout (SLO)

```
SINGLE SIGN-ON: Login once → access all apps. ✅
SINGLE LOGOUT: Logout once → logged out of ALL apps. (harder!)

PROBLEM:
User logs out of Salesforce.
Salesforce deletes its local session.
BUT: Okta still has an active session. Gmail, Slack, GitHub
still have active sessions based on that Okta session.
Is the user "logged out"? Only from Salesforce.

SINGLE LOGOUT FLOW (SAML SLO):
1. User clicks logout in Salesforce (SP)
2. Salesforce sends LogoutRequest to Okta (IdP)
3. Okta invalidates the IdP session
4. Okta sends LogoutRequests to ALL other SPs that had sessions
   based on this IdP session (Slack, Gmail, GitHub, etc.)
5. Each SP invalidates its local session
6. Okta redirects user to a "logged out" page

COMPLEXITY: SLO requires all SPs to implement the LogoutRequest
handler AND to be reachable by the IdP (no browser needed for
back-channel logout — IdP calls SPs directly, not via user browser).
Many implementations only do FRONT-CHANNEL logout (via browser
redirects, which requires the browser to visit each SP) — unreliable
if user closes the browser mid-way.

PRACTICAL REALITY: Most enterprise SSO implementations don't
implement SLO properly. The fix: short SP session lifetimes
(30-60 minutes), so even if SLO fails, sessions naturally expire.
```

---

## 6. Real-World Usage

**Okta (largest enterprise IdP):** Serves as the IdP for thousands of enterprise customers. Employees log in to Okta once and access all configured SPs (Salesforce, Workday, AWS, GitHub, etc.) via SAML or OIDC. Okta handles MFA enforcement, conditional access policies ("require MFA from untrusted networks"), and provides a centralized audit log.

**Google Workspace (IdP + SP):** Google acts as both an IdP (for "Sign in with Google" — OIDC) and as an SP consuming IdP assertions (enterprises can configure their ADFS or Okta as the IdP for Google Workspace). Large enterprises often have all Google Workspace accounts federated through their corporate Active Directory via SAML.

**AWS IAM Identity Center (formerly SSO):** Allows employees to access multiple AWS accounts with a single login, using the corporate IdP (Azure AD, Okta) as the SAML identity source. One Okta login → access to dev, staging, and production AWS accounts with appropriate IAM roles, without separate AWS credentials for each.

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ SAML replay attack: attacker     │ SP doesn't track used           │ Track assertion IDs in Redis;     │
│ captures and resubmits           │ assertion IDs; no InResponseTo  │ verify NotOnOrAfter strictly;    │
│ a valid assertion                │ validation; clock skew allows   │ use short assertion validity      │
│                                  │ reuse within validity window    │ windows (1-5 minutes)             │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ IdP outage → ALL applications    │ SSO creates a single point       │ Build emergency "break-glass"    │
│ inaccessible simultaneously      │ of failure for authentication    │ local admin accounts; IdP HA     │
│                                  │                                  │ (Okta/Azure AD are SaaS with     │
│                                  │                                  │ 99.99% SLA); have fallback IdP   │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ User not offboarded from SP      │ Employee leaves; IdP account     │ SCIM (System for Cross-domain    │
│ after IdP account disabled       │ disabled but SP still has local  │ Identity Management) — IdP       │
│                                  │ session or local account         │ automatically deprovisions       │
│                                  │                                  │ users in all SPs; short SP       │
│                                  │                                  │ session lifetimes                 │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ XML signature wrapping attack     │ SAML assertion XML parsed        │ Use well-maintained SAML         │
│ (attacker wraps valid signature   │ incorrectly — signature covers   │ libraries; validate signature    │
│ around malicious assertion)       │ part of XML but code uses        │ covers the ENTIRE assertion;     │
│                                  │ different part                   │ canonicalize XML before           │
│                                  │                                  │ signature verification           │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: What is SSO and what problem does it solve?**
A: SSO (Single Sign-On) allows users to authenticate ONCE with a central Identity Provider and access multiple applications without re-entering credentials. It solves three problems: user experience (one password instead of dozens), security (enforce MFA in one place, applies everywhere), and administration (offboard a leaving employee by disabling one account, instantly revoking access to all connected applications instead of manually deprovisioning 20+ systems).

**Q: What is SAML and how does it differ from OIDC for SSO?**
A: SAML (Security Assertion Markup Language) is an XML-based protocol where the IdP issues signed XML assertions containing user identity and attributes. It's browser-redirect-based and dominant in enterprise B2B SSO. OIDC (OpenID Connect, built on OAuth 2.0) uses JSON/JWT tokens and is more modern, mobile-friendly, and API-native. SAML is typically the choice when integrating with existing enterprise infrastructure (Azure AD, ADFS); OIDC is preferred for new applications, consumer identity, and API-driven architectures. Most major IdPs (Okta, Azure AD) support both protocols simultaneously.

**Q: What is SCIM and how does it relate to SSO?**
A: SCIM (System for Cross-domain Identity Management) is a protocol for automatically provisioning and deprovisioning users in Service Providers based on changes in the Identity Provider. When SSO alone handles only authentication — if an employee is terminated, you disable their IdP account (blocking future logins) but their SP accounts may persist. SCIM closes this gap: when an IdP account is deactivated, SCIM automatically deactivates or deletes the corresponding accounts in all connected SPs, ensuring complete offboarding across the entire application estate.

---
---

# TOPIC 4: Encryption

---

## 1. What Problem Does Encryption Solve?

Data exists in two states, each needing different protection:

```
DATA IN TRANSIT: Moving across a network
  Risk: EAVESDROPPING (man-in-the-middle) — someone on the
  network captures your packets and reads them.
  EXAMPLE: Connecting to your bank over unencrypted HTTP at a
  coffee shop WiFi — the barista (or anyone on the network)
  can see your username, password, and account details.

DATA AT REST: Stored on disk, in databases, in backups
  Risk: PHYSICAL THEFT or UNAUTHORIZED DB ACCESS — someone
  steals a hard drive, or gets DB access, or your backup S3
  bucket is accidentally made public.
  EXAMPLE: Database breach — if credit card numbers are stored
  in plaintext, all 10 million cards are immediately usable.
  If encrypted, the attacker has useless ciphertext without keys.

ENCRYPTION: Mathematical transformation of plaintext data into
ciphertext that is unreadable without the correct KEY.
The key is a secret that controls encryption and decryption.
```

---

## 2. Symmetric vs Asymmetric Encryption

### Symmetric Encryption — One Key for Both Operations

```
ALGORITHM: AES (Advanced Encryption Standard) — THE standard.
  AES-128: 128-bit key (secure)
  AES-256: 256-bit key (extremely secure, NSA-approved for top secret)
  Modes: AES-GCM (Galois/Counter Mode) — provides encryption AND
         integrity authentication simultaneously. THE recommended mode.

HOW IT WORKS:
  Encrypt: plaintext + key → ciphertext
  Decrypt: ciphertext + key → plaintext
  The SAME KEY does both.

SPEED: EXTREMELY FAST. Hardware AES acceleration built into:
  Intel (AES-NI instructions), ARM (crypto extensions).
  Encrypt/decrypt GBs per second on modern hardware.

THE FUNDAMENTAL PROBLEM: KEY DISTRIBUTION
  If Alice wants to send Bob an encrypted message:
  Alice must SHARE THE KEY with Bob FIRST.
  But how does Alice share the key SECURELY?
  She can't encrypt it (Bob doesn't have a key yet!).
  She can't send it plaintext (anyone could intercept it).

  This is the "key distribution problem" — SYMMETRIC ENCRYPTION
  ALONE CANNOT SOLVE IT.
  SOLUTION: Use asymmetric encryption to EXCHANGE a symmetric key.
  Then use the symmetric key for all subsequent communication.
  (This is EXACTLY what TLS does — recall Networking Fundamentals!)
```

### Asymmetric Encryption — Public/Private Key Pairs

```
ALGORITHMS: RSA (2048+ bit keys), ECC/ECDSA (256+ bit keys,
  much shorter keys for same security level), Ed25519 (modern,
  fast, used in SSH).

HOW IT WORKS:
  KEY PAIR: mathematically related pair of keys
  PUBLIC KEY: share with EVERYONE. Anyone can encrypt with it.
              Cannot decrypt what it encrypted.
  PRIVATE KEY: keep SECRET. Only you have it.
               Can decrypt what the public key encrypted.
               Can create SIGNATURES that prove ownership.

  Encrypt with public key → only private key holder can decrypt.
  Sign with private key → anyone with public key can verify.

ENCRYPTION USE:
  Alice wants to send secret to Bob:
  1. Alice gets Bob's public key (from Bob's website, cert, etc.)
  2. Alice encrypts message with Bob's PUBLIC KEY
  3. Ciphertext transmitted — safe to send anywhere
  4. Bob decrypts with his PRIVATE KEY (only he has it)
  KEY DISTRIBUTION SOLVED: Bob's public key is PUBLIC — no secret channel needed!

DIGITAL SIGNATURES USE:
  Bob wants to prove he wrote a document:
  1. Bob signs document hash with his PRIVATE KEY → Signature
  2. Anyone verifies: did Bob's PUBLIC KEY produce this signature from this hash?
  3. If YES → document was signed by the holder of that private key (Bob)
     AND document hasn't been modified (hash would differ)
  USE IN: Code signing, TLS certificates, JWT RS256, Git commit signing

SPEED: 100-1000x SLOWER than AES symmetric encryption.
RULE: NEVER use asymmetric encryption for bulk data.
  Instead: "hybrid encryption" — use asymmetric to EXCHANGE a
  symmetric key, then use the symmetric key for all data.
  TLS does exactly this (recall TLS handshake from Networking Fundamentals!).
```

---

## 3. Encryption in Transit — TLS Deep Dive

```
(Covered in Networking Fundamentals — HTTP/HTTPS topic.
Key points for Security interviews:)

TLS 1.3 IMPROVEMENTS OVER TLS 1.2:
  ✅ 1-RTT handshake (vs 2-RTT for TLS 1.2)
  ✅ 0-RTT session resumption for returning clients
  ✅ Forward secrecy MANDATORY (Ephemeral DH key exchange always)
  ✅ Removed weak cipher suites (RC4, 3DES, RSA key exchange)
  ✅ Encrypted more of the handshake (server cert now encrypted)

FORWARD SECRECY (Perfect Forward Secrecy — PFS):
  If an attacker records encrypted traffic TODAY and obtains
  the private key LATER — can they decrypt the old traffic?

  WITHOUT PFS: YES. Old RSA key exchange: same private key
  used to derive ALL session keys → recording + later key theft
  → decrypt all past sessions.

  WITH PFS (Ephemeral DH — ECDHE): NO.
  Each session generates a NEW, TEMPORARY key pair just for
  that session (ephemeral). The session key is DERIVED from
  this temporary key + DH exchange — the server's long-term
  private key is NOT used to derive the session key directly.
  After the session, the temporary key is DISCARDED.
  Future private key theft = cannot decrypt past sessions.

  TLS 1.3 mandates forward secrecy — all cipher suites use
  ephemeral key exchange.

HSTS (HTTP Strict Transport Security):
  Server header: Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
  Browser remembers for 1 year: ONLY contact this domain via HTTPS.
  Prevents SSL stripping attacks (attacker downgrades HTTPS→HTTP).
  HSTS Preload: submit domain to hardcoded browser list → browsers
  refuse HTTP connections even on FIRST visit (no trust-on-first-use).

CERTIFICATE TRANSPARENCY (CT):
  All publicly trusted TLS certificates must be logged in public
  CT logs (RFC 6962). Browsers reject certificates not in CT logs.
  Benefit: any misissued certificate (e.g., attacker gets a CA to
  issue a cert for google.com) is publicly visible within seconds.
  Google, CloudFlare monitor CT logs and can detect certificate abuse.
```

---

## 4. Encryption at Rest

```
LAYERS OF ENCRYPTION AT REST:

LAYER 1: FULL DISK ENCRYPTION (FDE)
  Entire disk encrypted. Even if someone physically steals
  the server's hard drive, data is unreadable without the key.
  Examples: AWS EBS encryption, BitLocker (Windows), LUKS (Linux)
  Keys managed by: AWS KMS (Key Management Service), TPM chip

  LIMITATION: Doesn't protect data from an authorized but
  compromised OS/application — the disk is decrypted for anyone
  who boots the OS. Protects against PHYSICAL THEFT only.

LAYER 2: DATABASE ENCRYPTION (TDE — Transparent Data Encryption)
  Database engine encrypts data files on disk automatically.
  Examples: PostgreSQL pgcrypto, MySQL InnoDB encryption, RDS encryption
  Application sees plaintext (the DB decrypts transparently);
  the underlying FILES are encrypted.

  LIMITATION: Same as FDE — protects the files, but anyone with
  DB access sees plaintext. A SQL injection attack that runs as
  the DB user gets plaintext data.

LAYER 3: APPLICATION-LEVEL ENCRYPTION (field-level encryption)
  The APPLICATION encrypts specific sensitive fields BEFORE
  storing in the database. The database stores ciphertext.
  Even a DBA with full DB access cannot read these fields.

  EXAMPLE: Credit card numbers encrypted before storage:
  plaintext: "4532 0151 0000 0001"
  stored in DB: "enc:AES256:iv=abc123:ciph=xyz789..."
  Only the application (with the encryption key) can decrypt.

  USE FOR: PAN data (Payment Card Numbers — PCI DSS requirement),
           SSN, biometric data, healthcare records (PHI under HIPAA)

  TRADEOFFS: Cannot query/search/index encrypted fields directly
  (can't do WHERE card_number = ? if it's encrypted differently
  each time). Solutions: encrypt with a deterministic cipher
  (same key + same plaintext = same ciphertext, but weaker) or
  maintain a secure index (Blind Index pattern).
```

### Key Management — The Critical Challenge

```
"The security of your encryption is only as good as the security of your keys."

If you encrypt data with a key stored in the same database as the data,
an attacker who gets DB access gets BOTH the key AND the ciphertext → useless encryption!

KEY MANAGEMENT BEST PRACTICES:

1. USE A KEY MANAGEMENT SERVICE (KMS):
   AWS KMS, Google Cloud KMS, Azure Key Vault, HashiCorp Vault
   The MASTER KEY never leaves the KMS hardware (HSM — Hardware
   Security Module). You send plaintext, KMS returns ciphertext.
   You send ciphertext, KMS decrypts and returns plaintext.
   The key itself is never exported — the HSM does the crypto.

2. ENVELOPE ENCRYPTION:
   Encrypting KEYS with keys. Used by AWS KMS and most KMS systems.

   DATA KEY: A random AES-256 key generated per file/record.
   Used to encrypt the actual data.

   KEY ENCRYPTION KEY (KEK) / MASTER KEY: Stored in KMS/HSM.
   Used to ENCRYPT THE DATA KEY.

   What's stored in your database/S3:
   - Encrypted data (data_ciphertext)
   - Encrypted data key (data_key_ciphertext)

   Decryption flow:
   1. Get data_key_ciphertext from DB
   2. Send to KMS → KMS decrypts using master key → returns data_key
   3. Use data_key to decrypt data_ciphertext → plaintext
   4. DISCARD data_key from memory when done

   BENEFITS:
   ✅ Master key NEVER leaves KMS hardware
   ✅ Different data key per record → compromise of one key ≠ all data
   ✅ Key rotation: rotate master key in KMS, re-encrypt data keys
      (not all data — just the small data keys). Much faster.

3. KEY ROTATION:
   Regularly rotate encryption keys (annually for KEKs minimum).
   Envelope encryption makes rotation manageable.
   Old data remains decryptable (keep old key in KMS marked "for decryption only")
   New data uses new key. Gradually re-encrypt old data as accessed.

4. SEPARATION OF DUTIES:
   The team managing encryption keys should NOT be the same team
   that manages the application. KMS access policies enforce this.
   Developers shouldn't have KMS key access in production.
```

---

## 5. Hashing vs Encryption — A Critical Distinction

```
ENCRYPTION: REVERSIBLE. Ciphertext → plaintext (with key).
  Use for: data you need to RETRIEVE LATER (credit cards, docs)

HASHING: ONE-WAY. Hash → cannot reverse to original (no key).
  Use for: data where you only need to VERIFY, not retrieve.
  Examples: passwords (verify a password matches the hash),
            file integrity (verify file hasn't changed),
            digital signatures (sign the hash, not the whole file)

CRYPTOGRAPHIC HASH PROPERTIES:
  Deterministic: same input → always same output
  Fast to compute: efficient to calculate
  Collision resistant: infeasible to find two inputs with same hash
  Pre-image resistant: given hash H, infeasible to find input M where hash(M) = H
  Avalanche effect: tiny input change → completely different hash

COMMON HASH FUNCTIONS:
  SHA-256 / SHA-3: For file integrity, digital signatures, general use
  bcrypt / Argon2id / scrypt: For PASSWORD HASHING (intentionally SLOW)
  MD5 / SHA-1: BROKEN (collision attacks found) — never use for security

PASSWORD HASHING — SALTING:
  PROBLEM: Same password → same hash.
  If two users have password "Password123", their DB entries are identical.
  RAINBOW TABLE: Pre-compute hash("Password123") → attacker sees the hash
  → looks up in table → immediately knows the password.

  SALT: A random value appended to the password before hashing.
  Each user gets a UNIQUE, RANDOM SALT stored alongside their hash.
  hash = bcrypt("Password123" + "randomsalt_abc") → unique per user
  Even two users with identical passwords have different stored hashes.
  Rainbow tables are defeated: attacker would need one table PER UNIQUE SALT.

  bcrypt example:
  stored_value = "$2b$12$randomsalt_abc123/hashvaluexyz..."
  The stored value INCLUDES the salt and cost factor (12 = 2^12 iterations).
  bcrypt.verify(input_password, stored_value) → True/False
```

---

## 6. Real-World Usage

**WhatsApp / Signal (End-to-End Encryption):** Messages encrypted on sender's device with recipient's public key — WhatsApp servers only ever see ciphertext. The Signal Protocol uses a combination of: Curve25519 for key agreement, AES-256-GCM for message encryption, HMAC-SHA256 for integrity. Even WhatsApp cannot read your messages (though metadata — who talked to whom and when — is visible).

**AWS (KMS + Envelope Encryption):** Every EBS volume, RDS database, and S3 bucket can be encrypted using AWS KMS. S3 uses envelope encryption: each S3 object gets a unique data key (encrypted by your KMS master key and stored alongside the object). Rotating your KMS master key only requires re-encrypting the data keys, not re-encrypting petabytes of data.

**PCI-DSS compliance (BFSI — directly relevant):** Payment Card Industry Data Security Standard mandates: encrypt cardholder data in transit (TLS 1.2+), encrypt at rest (AES-256), never store CVV (not even encrypted), truncate or mask PANs in logs. Field-level encryption with KMS is the standard pattern for payment systems — even the database administrator cannot see raw card numbers.

---

## 7. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Key stored next to encrypted     │ "Security by configuration"      │ External KMS (AWS KMS, Vault);   │
│ data — both stolen together       │ not enforced; developer          │ separation of key material from  │
│                                  │ convenience over security        │ encrypted data; automated checks  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Using ECB mode for AES            │ AES-ECB encrypts identical       │ ALWAYS use AES-GCM or AES-CBC    │
│ (reveals data patterns)           │ plaintext blocks to identical    │ with random IV; never ECB.        │
│                                  │ ciphertext — patterns visible    │ Lint/security scan for ECB usage  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ TLS downgrade attack (POODLE,     │ Server supports old weak          │ Disable TLS 1.0, 1.1, SSL 3.0;  │
│ BEAST): attacker forces old       │ protocols for "backwards          │ enforce TLS 1.2 minimum,          │
│ vulnerable TLS version            │ compatibility"                    │ TLS 1.3 preferred; HSTS           │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Secrets/API keys in git repo      │ Developer hardcodes credentials   │ Pre-commit hooks (gitleaks,       │
│ (scanned by attackers within       │ in source code; accidentally      │ trufflehog); GitHub secret       │
│ seconds of a public push)         │ committed to public repo          │ scanning; secrets manager         │
│                                  │                                  │ (AWS Secrets Manager, Vault)      │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 8. Interview Quick-Fire Answers

**Q: What's the difference between symmetric and asymmetric encryption, and how does TLS use both?**
A: Symmetric encryption uses ONE key for both encrypting and decrypting (AES-256) — extremely fast but requires securely exchanging the key. Asymmetric encryption uses a key PAIR — the public key encrypts, the private key decrypts (RSA, ECDH) — solves key distribution but is 100-1000x slower. TLS combines both: the handshake uses asymmetric cryptography (ECDHE) to securely agree on a session key, then all actual data is encrypted using that session key with fast symmetric AES. This "hybrid" approach gives both security (key exchange without prior shared secret) and performance (fast bulk encryption).

**Q: What is forward secrecy and why does it matter?**
A: Forward secrecy (PFS) means that even if an attacker records encrypted traffic NOW and later obtains the server's private key, they CANNOT decrypt the recorded traffic. This is achieved by generating ephemeral (temporary, per-session) key pairs for the key exchange (ECDHE), then discarding them after the session ends. Without PFS (RSA key exchange), all past sessions could be decrypted if the private key is ever compromised. TLS 1.3 mandates forward secrecy by removing non-ephemeral key exchange cipher suites.

**Q: What is envelope encryption and why is it used in cloud KMS systems?**
A: Envelope encryption uses two levels of keys: a DATA KEY (unique per record/file, never stored in plaintext) encrypts the actual data; a MASTER KEY in the KMS (Hardware Security Module, never exportable) encrypts the data key. The database stores encrypted data + encrypted data key. Decryption requires calling KMS to decrypt the data key, then using it locally. Benefits: the master key never leaves the HSM; each record has a unique data key so compromise of one key doesn't affect all data; key rotation only requires re-encrypting the small data keys, not gigabytes of data.

**Q: Why should you use bcrypt or Argon2 for password hashing instead of SHA-256?**
A: SHA-256 is designed to be FAST — computers can compute billions of SHA-256 hashes per second, making brute-force attacks trivial. bcrypt and Argon2 are designed to be SLOW (configurable cost factor) — making each guess take milliseconds, limiting brute force to thousands of attempts per second. Additionally, bcrypt/Argon2 automatically include a SALT (unique random value per user) in the stored hash, defeating precomputed rainbow tables. SHA-256 has none of these properties — using it for passwords leaves users vulnerable to offline cracking after a database breach.

---
---

# TOPIC 5: DDoS Protection

---

## 1. What Problem Does DDoS Protection Solve?

A **DDoS (Distributed Denial of Service) attack** overwhelms a system with traffic from thousands or millions of compromised machines (a "botnet"), making the service unavailable to legitimate users — without needing to actually "hack" anything. The attack doesn't need to break in; it just needs to flood the door.

```
THE ANATOMY OF A DDoS ATTACK:

NORMAL TRAFFIC:
Legitimate users → 10,000 req/sec → Your server handles it fine

DDoS ATTACK:
Botnet (500,000 compromised machines, each sending 100 req/sec):
  500,000 × 100 = 50,000,000 req/sec → Your server is overwhelmed
  CPU maxed, memory exhausted, network saturated
  → Legitimate users cannot connect
  → Your service appears "down"
  → Business impact: revenue loss, SLA violations, reputational damage

THE "DISTRIBUTED" PART:
Source IPs are spread across millions of different addresses globally.
You can't just block one IP range — the attack comes from everywhere.

TYPES OF DDoS:

VOLUMETRIC (Layer 3/4): Flood with raw bandwidth/packets
  UDP Flood: Send millions of UDP packets to random ports
  ICMP Flood (Ping flood): Overwhelm with ping requests
  Amplification attacks: Send small requests to reflectors
    (DNS, NTP, memcached) that generate large responses
    targeting the victim (100-50,000x amplification factor!)
  → Measured in Gbps or Tbps. Largest recorded: 3.47 Tbps (2022)

PROTOCOL (Layer 3/4): Exploit protocol weaknesses
  SYN Flood: millions of TCP SYN packets → server creates
    half-open connections until table full → no new connections
    (recall from Networking Fundamentals — TCP topic!)
  ACK Flood, RST Flood

APPLICATION (Layer 7): Target specific application logic
  HTTP Flood: millions of legitimate-looking HTTP GET/POST requests
  Slowloris: open many connections, send headers very slowly,
    keeping server worker threads occupied indefinitely
  Cache-busting: requests with unique query strings bypass CDN cache,
    forcing each to hit the origin server
  → Harder to detect (looks like real traffic), needs fewer packets
```

---

## 2. DDoS Mitigation Strategies — Layered Defense

```
PRINCIPLE: No single technique stops all DDoS. Use LAYERS.

LAYER 1: UPSTREAM / ISP-LEVEL (Volumetric)
  Your ISP or transit provider absorbs or blocks volumetric
  attacks BEFORE they reach your network. BGP blackholing:
  "route all traffic to victim IP to null" — drops attack AND
  legitimate traffic (nuclear option, buys time).
  BGP FlowSpec: more surgical traffic filtering at ISP level.

LAYER 2: SCRUBBING CENTERS (Cloud-based)
  Specialized DDoS protection providers (Cloudflare, Akamai,
  AWS Shield Advanced) have globally distributed SCRUBBING CENTERS
  with MASSIVE bandwidth (Cloudflare: 100+ Tbps aggregate capacity).
  ALL traffic is routed through scrubbing centers → malicious
  traffic filtered → clean traffic forwarded to origin.
  Anycast routing (recall CDN topic, Networking Fundamentals!)
  absorbs the attack volume globally.

LAYER 3: CDN AS A SHIELD
  If origin IP is hidden behind a CDN:
  - Attackers must attack the CDN (which has global DDoS absorption)
  - Origin IP stays secret (never expose in DNS)
  - CDN cache absorbs most HTTP flood (cache hits never reach origin)
  This is the cheapest, most effective protection for most websites.

LAYER 4: RATE LIMITING (Application Layer)
  (Recall Rate Limiting topic from Scalability notes!)
  Limit requests per IP per second. First line of defense against
  smaller floods. Combine with CAPTCHA for suspected bots.
  ChallengeL Serve JavaScript challenge — bots fail, humans pass.

LAYER 5: WAF (Web Application Firewall)
  Layer 7 traffic inspection. Block:
  - Known bot signatures (User-Agent patterns, behavior)
  - Malformed requests (oversized headers, invalid methods)
  - Geographic blocking (if your service is India-only, block traffic
    from IPs outside India — reduces attack surface dramatically)
  - IP reputation lists (known botnet C&C, Tor exit nodes)

LAYER 6: ANYCAST + GLOBAL DISTRIBUTION
  Distribute your infrastructure globally (multi-region, recall
  Geo-distribution from Scalability notes). An attack targeting
  one region is absorbed by that region; other regions unaffected.
  Anycast means a global flood is split across all PoPs —
  each PoP sees a manageable fraction of the total.
```

### AWS Shield — Managed DDoS Protection

```
AWS SHIELD STANDARD (free, automatic for all AWS customers):
  - Automatic protection against most common DDoS attacks
  - Layer 3/4 protection (volumetric, protocol attacks)
  - AWS's global network absorbs the traffic
  - SYN flood protection (SYN cookies at AWS network edge)

AWS SHIELD ADVANCED ($3,000/month + data transfer costs):
  - Always-on advanced DDoS detection
  - Automatic Layer 7 protection (application attacks)
  - 24/7 DDoS Response Team (DRT) — AWS experts help during attacks
  - Financial protection: AWS credits costs caused by DDoS attacks
    (if Auto Scaling scales up due to attack → credits for extra charges)
  - Advanced monitoring and reporting

CLOUDFLARE MAGIC TRANSIT / SPECTRUM:
  BGP-based DDoS protection: announce your IP ranges through
  Cloudflare's network → all traffic scrubbed before reaching you.
  Used by large enterprises that need to protect non-HTTP traffic.
```

---

## 3. Application-Layer DDoS — The Hardest to Stop

```
PROBLEM: Modern Layer 7 DDoS uses legitimate-looking requests
  (valid HTTP, valid URLs, valid headers) from real-looking clients.
  Traditional rate-limiting by IP fails: botnet spreads across
  millions of IPs, each sending only 1-2 req/sec.

DETECTION TECHNIQUES:
  Behavioral analysis: legitimate users browse randomly; bots tend
    to have repetitive, structured request patterns (same path,
    no referrer, no cookies, identical User-Agents).
  Browser fingerprinting: JavaScript challenge to verify browser
    behavior (JS execution, cookie handling, header ordering).
  ML-based anomaly detection: baseline "normal" traffic patterns,
    flag deviations in real-time (Cloudflare Bot Management,
    AWS WAF with fraud control rules).

SLOWLORIS PROTECTION:
  Slowloris sends HTTP headers slowly (1 byte per ~15 seconds)
  keeping server worker threads alive indefinitely.
  FIX: Minimum request rate timeout (Nginx: client_header_timeout 10s)
       Maximum concurrent connections per IP (LimitConn in Nginx)
       Proxy/reverse proxy (Nginx buffers connections, app server
       only sees COMPLETE requests — Nginx absorbs Slowloris).
       (Recall Reverse Proxy topic, Scalability notes!)

CACHE-BUSTING PROTECTION:
  Attackers add random query params (?r=randomstring) to bypass CDN cache.
  FIX: Normalize cache keys — strip or ignore arbitrary query params
       at CDN/WAF level; only cache based on known query params.
```

---

## 4. Real-World DDoS Incidents

**GitHub (2018 — 1.35 Tbps Memcached Amplification):** The largest DDoS attack at that time. Attackers sent requests to memcached servers (UDP port 11211) with GitHub's IP as source — memcached servers sent responses (up to 50,000x the request size) to GitHub. Cloudflare and Akamai helped absorb traffic; GitHub's BGP routing redirected traffic through scrubbing centers, and the attack ended within 20 minutes.

**Cloudflare (2022 — 26 million RPS HTTP/3 DDoS):** A 26 million HTTP requests/second Layer 7 DDoS — all from 5,067 compromised cloud server IPs (not home IPs). Cloudflare's infrastructure absorbed the entire attack without any impact to customers, illustrating the value of global distributed network capacity for DDoS absorption.

**AWS Shield (2020 — 2.3 Tbps):** AWS reported absorbing a 2.3 Tbps attack (the largest ever at the time) on AWS Shield Advanced. The attack was reflected UDP traffic — AWS's global infrastructure absorbed it with no impact to the targeted customer.

---

## 5. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Origin IP exposed — CDN         │ DNS leakage (old DNS records,    │ Never expose origin IP in DNS;   │
│ protection bypassed              │ email headers, cert transparency │ use CF Tunnel/AWS PrivateLink;   │
│                                  │ logs show real IP)               │ rotate origin IP after setup     │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Auto-scaling scales up under     │ DDoS looks like a traffic spike  │ AWS Shield Advanced has          │
│ DDoS → massive AWS bill          │ → ASG launches hundreds of       │ financial protection; set ASG    │
│ (the "bill bomb")                │ instances → $10K+ surprise bill  │ max-instances conservatively;   │
│                                  │                                  │ rate limit BEFORE origin server  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Rate limiting blocks legitimate  │ Rate limit set too aggressively; │ Rate limit by authenticated user │
│ users (e.g., enterprise behind   │ corporate NAT shares one IP;     │ or API key, not just IP; higher  │
│ a NAT all hit same IP limit)     │ all employees share one IP       │ limits for known corporate ranges│
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 6. Interview Quick-Fire Answers

**Q: What are the main types of DDoS attacks and how do mitigation strategies differ?**
A: Volumetric attacks (Layer 3/4) flood with raw bandwidth/packets — mitigated by upstream scrubbing centers and CDNs with massive global capacity (Cloudflare's 100+ Tbps absorbs even the largest attacks). Protocol attacks exploit TCP/IP weaknesses (SYN flood) — mitigated by SYN cookies at the network edge. Application layer attacks (Layer 7) send legitimate-looking HTTP requests — hardest to mitigate, require behavioral analysis, JavaScript challenges, ML-based bot detection, and rate limiting. A layered defense using CDN + WAF + rate limiting + upstream scrubbing covers all three attack types.

**Q: Why does hiding your origin IP matter for DDoS protection?**
A: CDN-based DDoS protection only works if attackers target the CDN (which has massive DDoS absorption capacity) rather than your origin server directly. If the origin's real IP is known (discoverable via old DNS records, SSL certificate history in CT logs, email headers, etc.), attackers bypass the CDN entirely and flood the origin — which has no special DDoS capacity. Keeping the origin IP secret and routing all traffic through the CDN is the most cost-effective DDoS mitigation for most applications.

---
---

# TOPIC 6: Zero Trust Architecture

---

## 1. What Problem Does Zero Trust Solve?

Traditional "castle and moat" network security assumes: **everything INSIDE the corporate network is trusted; everything OUTSIDE is untrusted.**

```
TRADITIONAL PERIMETER SECURITY:

INTERNET (Untrusted)
    │
    ▼
[FIREWALL / VPN] ← the "castle wall"
    │
    ▼
CORPORATE NETWORK (Trusted) → employees, servers, databases
"Once you're through the firewall/VPN, you can access everything."

PROBLEMS WITH THIS MODEL:
1. LATERAL MOVEMENT: Attacker compromises ONE machine inside
   the perimeter → now "trusted" → can access EVERYTHING.
   Most major breaches follow this pattern.
   (The SolarWinds attack: compromised a build server INSIDE
   thousands of companies' networks → then moved laterally to
   government systems, financial institutions, etc.)

2. CLOUD AND REMOTE WORK: Modern infrastructure spans AWS,
   GCP, Azure, SaaS apps, home networks, coffee shops.
   "Inside the corporate network" barely means anything anymore
   when your data is in AWS and employees work from anywhere.

3. INSIDER THREATS: A malicious employee IS inside the perimeter.
   Perimeter security provides zero protection against insiders.

4. VPN FRICTION: Forcing all traffic through a VPN creates
   latency, bottlenecks, and poor user experience — yet provides
   only perimeter-level security.
```

**Zero Trust** is a security model based on the principle: **"Never trust, always verify."** No user, device, or service is trusted by default — regardless of whether they're on the corporate network or not. Every access request is AUTHENTICATED, AUTHORIZED, and ENCRYPTED, every time.

---

## 2. The Zero Trust Principles

```
PRINCIPLE 1: VERIFY EXPLICITLY
  Every access request must be authenticated AND authorized
  using ALL available signals:
  - User identity (is this a valid, active employee?)
  - Device health (is the device managed, patched, compliant?)
  - Location (is this a known location / new country?)
  - Time (is this an unusual access time?)
  - Application/resource being accessed (what's the sensitivity?)
  - Behavioral anomaly (does this match the user's normal pattern?)

  Critically: "being on the VPN" is NOT a sufficient signal.

PRINCIPLE 2: USE LEAST PRIVILEGE ACCESS (connect to Auth Topic 1)
  Grant minimum necessary permissions.
  Time-limited access (JIT — Just-In-Time):
  "Engineer needs DB access for 2 hours to debug incident"
  → Grant temporary access → automatically revoked after 2 hours.
  Eliminates standing privileged access (permanent admin rights).

PRINCIPLE 3: ASSUME BREACH
  Design systems ASSUMING an attacker is already inside.
  Segment networks so a compromise of one service doesn't
  mean a compromise of all services.
  Monitor ALL traffic (east-west, not just north-south).
  Implement detection so when breach happens (not if), you
  catch it fast and limit blast radius.
```

---

## 3. Zero Trust Architecture Components

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER / DEVICE                            │
│  Employee on laptop, employee on phone, partner API, service     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Access Request
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      POLICY ENGINE                                │
│  Evaluates: identity + device + context → access decision        │
│  "Should this user, from this device, in this context,           │
│   access this resource right now?"                                │
│  Components:                                                      │
│    - Identity verification (IdP: Okta, Azure AD — SSO Topic!)    │
│    - Device compliance check (MDM: Jamf, Intune)                 │
│    - Conditional Access policies                                  │
│    - Risk scoring (ML-based behavioral analytics)                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │ ALLOW / DENY decision
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    POLICY ENFORCEMENT POINT (PEP)                │
│  The gateway that actually allows or blocks the request          │
│  Examples: API Gateway, Envoy Proxy (service mesh), Cloudflare   │
│            Access, Google BeyondCorp, Zscaler Private Access     │
│                                                                   │
│  If ALLOWED: proxy the request to the protected resource         │
│  If DENIED: return 403, log the attempt, trigger alert           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    PROTECTED RESOURCE                              │
│  Application, API, database, server                               │
│  The resource itself ALSO verifies the identity of its callers   │
│  (defense in depth — don't just rely on the PEP)                │
└─────────────────────────────────────────────────────────────────┘
```

### mTLS — Mutual TLS for Service-to-Service Zero Trust

```
In a Zero Trust microservices architecture, EVERY service-to-
service call must be authenticated — not just user-to-service.

STANDARD TLS: Server proves its identity to the client.
  Client: anonymous (server doesn't know who's calling).
  This is what HTTPS does — you authenticate the server you're
  connecting to, but the server doesn't authenticate you via TLS
  (uses JWT/cookies in the application layer instead).

mTLS (Mutual TLS): BOTH sides present X.509 certificates.
  Client proves: "I am payment-service, cert issued by company CA"
  Server proves: "I am inventory-service, cert issued by company CA"
  Both verify: "Is this cert issued by my trusted CA? Is it valid?"

RESULT: Every service-to-service call is:
  ✅ Encrypted (TLS encryption)
  ✅ Authenticated (both sides verified via certificates)
  ✅ Cannot be spoofed (requires the CA-issued certificate)

CERTIFICATE LIFECYCLE:
  Issuing and rotating certificates to hundreds of services is
  complex. SERVICE MESHES automate this:

  Istio / Linkerd / Cilium (service mesh sidecars):
  Each pod gets an Envoy sidecar proxy.
  The mesh's CA (e.g., Istio's Citadel) automatically:
  - Issues short-lived certificates (24 hours) to each pod
  - Rotates them before expiry (no manual cert management!)
  - Enforces mTLS between all service calls transparently
  - The application code doesn't need to handle TLS at all

  SPIFFE / SPIRE: Standards for workload identity in Zero Trust.
  SPIFFE defines a URI-based identity for workloads:
  spiffe://company.com/ns/payment/sa/payment-service
  SPIRE implements SPIFFE: issues SVIDs (SPIFFE Verifiable
  Identity Documents) as certificates. Workloads use their
  SPIFFE identity for mTLS — no hardcoded service accounts needed.
```

### Google BeyondCorp — Zero Trust for Human Access

```
Google pioneered Zero Trust with BeyondCorp (2011).
Instead of VPN, employees access internal apps through a
context-aware access proxy based on:
  1. User identity (Google account + MFA)
  2. Device certificate (is it Google-managed, with encryption,
     latest OS, no known vulnerabilities?)
  3. Access level (what's the sensitivity of what they're accessing?)

ACCESS TIERED BY RISK:
  LOW SENSITIVITY (public docs, email):
    Any Google account, any device → access granted
  MEDIUM SENSITIVITY (internal wikis, code):
    Google account + managed device + no recent security alerts
  HIGH SENSITIVITY (production DBs, security configs):
    Google account + managed device + hardware security key MFA
    + no recent suspicious activity + within office hours

THE EMPLOYEE EXPERIENCE:
  No VPN. Open a browser, go to the internal app URL.
  BeyondCorp evaluates the context → grants or denies access.
  From a coffee shop, the same evaluation applies.
  VPN vs no VPN is irrelevant — the REQUEST is evaluated, not the network.

GOOGLE'S RESULT: In 2017, when NotPetya ransomware swept through
corporate networks globally, Google suffered minimal damage because
their internal architecture already assumed untrusted networks.
```

---

## 4. Zero Trust in Practice — Implementation Path

```
MOST ORGANIZATIONS DON'T DO ZERO TRUST IN ONE STEP.
It's a JOURNEY, not a binary state.

MATURITY LEVELS:

LEVEL 0 (Traditional):
  VPN for all internal access. IP-based trust.
  Flat network — everything accessible once on VPN.

LEVEL 1 (Beginning Zero Trust):
  MFA for all user access.
  Conditional access (deny from unmanaged devices).
  Network segmentation (DMZ, VLANs between sensitive zones).
  Inventory and classify all assets.

LEVEL 2 (Advancing):
  Device compliance enforcement (MDM required for access).
  Application-level access control (replace network access with
  app-specific access — Cloudflare Access, Zscaler ZPA).
  Service accounts given least privilege (no overprivileged SA).
  mTLS between microservices (service mesh deployed).
  Continuous monitoring of all access patterns.

LEVEL 3 (Optimizing):
  Just-In-Time privileged access (no standing admin access).
  ML-based anomaly detection on all access patterns.
  Automated response (suspicious behavior → automatic challenge or block).
  Full visibility across cloud, on-premise, SaaS, all endpoints.
  SPIFFE/SPIRE workload identity for non-human access.

RECOMMENDED STARTING POINTS (high ROI):
1. Enforce MFA everywhere — immediate, huge security impact
2. Deploy Conditional Access — block non-compliant devices
3. Segment networks — contain blast radius of any breach
4. Implement privileged access management (PAM) — control
   who has admin access and require approval + audit trail
```

---

## 5. Real-World Usage

**Google BeyondCorp:** Described above — the original Zero Trust implementation. Open-sourced as BeyondCorp Enterprise (now part of Google Cloud). The NotPetya incident in 2017 validated their approach — Google's business continuity despite the attack was directly attributed to their Zero Trust network design.

**Cloudflare Access + Tunnel:** Used by many companies as their Zero Trust replacement for VPN. Employees access internal apps via Cloudflare's network (authenticated via IdP + device check) without exposing those apps to the internet. Cloudflare Tunnel establishes an outbound connection from the private server to Cloudflare — no inbound firewall rules needed, the server IP is never exposed.

**BFSI adoption:** RBI and SEBI guidelines increasingly reference Zero Trust principles — especially the "assume breach" principle given the frequency of supply-chain attacks on financial institutions. mTLS between microservices (especially for payment flows between originator banks, NPCI's UPI switch, and beneficiary banks) is becoming a regulatory expectation for new infrastructure.

---

## 6. Failure Scenarios

```
┌────────────────────────────────┬────────────────────────────────┬──────────────────────────────────┐
│ Failure                         │ Root Cause                      │ Mitigation                        │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ mTLS certificate expiry causes   │ Short-lived certs (24h) not      │ Use service mesh with auto-       │
│ service-to-service failures      │ rotated before expiry;           │ rotation; monitor cert expiry;    │
│                                  │ certificate rotation logic fails │ alert on rotation failures;       │
│                                  │                                  │ grace period overlapping certs    │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Zero Trust adds too much         │ Every request requires policy     │ Cache policy decisions (seconds,  │
│ latency for high-throughput       │ engine evaluation, IdP check,    │ not indefinitely); pre-authorize  │
│ internal services                 │ certificate verification         │ service-to-service with           │
│                                  │                                  │ workload certificates (fast        │
│                                  │                                  │ TLS verify vs slow IdP call)      │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Over-permissive service accounts │ Default service accounts         │ Audit service account permissions │
│ — one compromised SA gives       │ created with wide permissions     │ regularly; apply least privilege  │
│ blast radius across all services │ for "convenience"                │ to all; use SPIFFE workload        │
│                                  │                                  │ identity with scoped permissions  │
├────────────────────────────────┼────────────────────────────────┼──────────────────────────────────┤
│ Policy engine outage blocks       │ Policy enforcement point has     │ HA policy engine (clustered OPA/  │
│ all service access                │ no fallback; single point         │ Sentinel); fail-open vs fail-     │
│                                  │ of failure                        │ closed decision (context-         │
│                                  │                                  │ dependent; security-critical       │
│                                  │                                  │ systems: fail-closed always)      │
└────────────────────────────────┴────────────────────────────────┴──────────────────────────────────┘
```

---

## 7. Interview Quick-Fire Answers

**Q: What is Zero Trust and how does it differ from perimeter security?**
A: Perimeter security assumes everything INSIDE the corporate network is trusted — once through the firewall/VPN, a user or service can access most internal resources. Zero Trust assumes NO implicit trust based on network location — "never trust, always verify." Every access request (user, device, service) must be explicitly authenticated, authorized using current context (device health, location, behavior), and the session encrypted, regardless of whether the requester is on-premises, on VPN, or in the cloud. This matters because modern attacks (supply chain, insider threats, lateral movement) defeat perimeter security once they're inside the perimeter.

**Q: What is mTLS and why is it important for microservices security?**
A: Standard TLS authenticates the SERVER to the client. mTLS (mutual TLS) requires BOTH sides to present certificates — the client proves its identity to the server AND the server proves its identity to the client. In microservices, this means service A calling service B must prove it's actually service A (using a certificate issued by the company's CA), and service B can verify and enforce "only payment-service is allowed to call my /charge endpoint." Service meshes (Istio, Linkerd) automate certificate issuance and rotation, making mTLS transparent to application code. Without mTLS, a compromised service or an attacker who gained network access could impersonate legitimate services.

**Q: What is Just-In-Time (JIT) access and why is it a Zero Trust best practice?**
A: JIT access means granting elevated permissions ONLY when needed, for a SPECIFIC DURATION, rather than maintaining standing privileged access permanently. Example: "This engineer needs read access to the production database to debug an incident — grant access for 2 hours, then automatically revoke." This eliminates the risk of persistent high-privilege credentials being stolen, compromised in a supply chain attack, or misused by insiders. If an engineer's laptop is compromised, the attacker gets the engineer's normal access but not elevated privileges (which would have required an explicit, time-limited approval).

---
---

# APPENDIX: Cross-Topic Quick Reference

## Master Comparison — All Security Concepts

```
┌──────────────────────────┬───────────────────────────────────────────────────────────┐
│ Topic                     │ Core Question It Answers                                    │
├──────────────────────────┼───────────────────────────────────────────────────────────┤
│ Authentication vs          │ "Who are you, and what are you allowed to do? —            │
│ Authorization              │ These are separate questions requiring separate answers"   │
│ JWT & OAuth 2.0            │ "How do I issue unforgeable, stateless identity tokens,    │
│                           │ and how do I let users grant limited API access to         │
│                           │ third-party applications without sharing passwords?"        │
│ SSO & SAML                 │ "How do enterprises give employees one login for dozens    │
│                           │ of applications, with central control and audit?"           │
│ Encryption                 │ "How do I ensure data cannot be read in transit or at      │
│                           │ rest, even if intercepted or stolen?"                       │
│ DDoS Protection            │ "How do I prevent volumetric network attacks from making   │
│                           │ my service unavailable to legitimate users?"                │
│ Zero Trust Architecture    │ "How do I secure access when I can no longer rely on       │
│                           │ network location as a trust signal?"                        │
└──────────────────────────┴───────────────────────────────────────────────────────────┘
```

## How All Six Topics Interconnect

```
A COMPLETE SECURE SYSTEM — all topics in one architecture:

USER ACCESSES api.company.com:

1. ZERO TRUST CHECK (Topic 6):
   Is this a managed device? MFA verified? Normal location?
   → Cloudflare Access / BeyondCorp evaluates context.
   → Policy decision: ALLOW (proceed) / DENY (block)

2. AUTHENTICATION (Topic 1 + Topic 2):
   User authenticates via SSO (Topic 3 — Okta SAML/OIDC)
   → OAuth 2.0 Authorization Code flow
   → JWT issued: {user_id, roles, exp: +15min, jti: unique}
   → Signed with RS256 (private key at auth server only)

3. AUTHORIZATION (Topic 1):
   JWT presented to API Gateway.
   Gateway verifies RS256 signature (using public key).
   OPA policy: "Does user's role allow this API + method?"
   → RBAC check: admin can DELETE, editor cannot.
   → ALLOW: request forwarded to service.

4. ENCRYPTION IN TRANSIT (Topic 4):
   All communication over TLS 1.3 (forward secrecy via ECDHE).
   Service-to-service: mTLS (service mesh — Istio) (Topic 6).
   No plaintext traffic anywhere in the system.

5. SERVICE-TO-SERVICE (Topic 6):
   payment-service calls inventory-service.
   mTLS certificates verify both sides' identities.
   SPIFFE workload identity: no hardcoded service accounts.

6. DATA AT REST (Topic 4):
   Sensitive fields (card numbers, SSN) encrypted at application level.
   Envelope encryption: data keys encrypted by AWS KMS master key.
   Database files encrypted (RDS encryption — EBS AES-256).

7. DDOS PROTECTION (Topic 5):
   Cloudflare absorbs volumetric attacks at edge.
   WAF blocks Layer 7 anomalies and bot traffic.
   Rate limiting (per user + per IP) at API Gateway.
   Origin IP never exposed in DNS.

8. SSO OFFBOARDING (Topic 3):
   Employee leaves → Okta account disabled.
   SCIM deprovisions user from ALL applications.
   JWT refresh tokens revoked.
   Access revoked within seconds, not hours.
```

## Final Study Tips

```
1. LEARN THE ATTACK TO UNDERSTAND THE DEFENSE.
   For every security control, understand WHAT ATTACK it prevents:
   - MFA prevents: credential stuffing, phishing (stolen passwords)
   - RS256 JWT: prevents: token forgery if secret leaked
   - PKCE: prevents: authorization code interception
   - mTLS: prevents: service impersonation, MITM between services
   - Forward secrecy: prevents: retroactive decryption after key theft
   This "attack → defense" framing shows you understand WHY,
   not just WHAT.

2. THE SEQUENCE FOR SECURE API DESIGN:
   Authentication → who are you? (JWT/OAuth)
   Authorization → what can you do? (RBAC/ABAC/OPA)
   Input validation → is the request valid? (prevent SQLi, XSS)
   Rate limiting → not too fast? (prevent DoS/brute-force)
   Audit logging → what happened? (compliance, incident response)
   Encryption → data protected in transit and at rest?
   This 6-step checklist covers the majority of "how would you
   secure this system?" interview questions.

3. CONNECT SECURITY TO PRIOR NOTES:
   - TLS handshake → Networking Fundamentals (HTTP/HTTPS topic)
   - Rate limiting → Scalability notes (Rate Limiting topic)
   - Redis for session tokens/blocklists → Caching notes
   - JWT → Stateless (horizontal scaling) from Scalability notes
   - DDoS → CDN absorption → Networking Fundamentals (CDN topic)
   - mTLS → Envoy sidecar → DevOps notes (Kubernetes topic)
   - Audit logs → structured logging → DevOps (Observability topic)
   - KMS + S3 encryption → DevOps (AWS notes)

4. FOR BFSI/FINTECH INTERVIEWS (highly relevant):
   - PCI-DSS: encrypt card data at rest (AES-256), in transit (TLS 1.2+),
     never store CVV, truncate PANs in logs, field-level encryption
   - RBI: data localization → all auth and encryption keys must be
     in India (KMS in ap-south-1); audit trail for all transactions
   - OAuth for open banking (Account Aggregator framework in India):
     OIDC-based consent framework — FIPs (Financial Information
     Providers) and FIUs (Financial Information Users) exchange
     data via standardized OAuth consent flows
   - mTLS between banking systems: NPCI, payment gateways, and
     banks increasingly mandate mTLS for API integrations
   - Zero Trust for remote SOC (Security Operations Center):
     security analysts access production systems via context-aware
     proxy (no direct VPN to production), all commands logged
   - Incident response: DDoS on payment gateway = revenue loss +
     regulatory scrutiny → AWS Shield Advanced + CloudFront mandatory

5. SECURITY IS NOT AN AFTERTHOUGHT.
   In every system design answer, include security as a FIRST-CLASS
   concern, not a footnote. "And for security, we'd add TLS and auth"
   is weak. Instead:
   "For authentication, we use OAuth 2.0 with OIDC, RS256-signed JWTs
   with 15-minute expiry and refresh token rotation. Service-to-service
   uses mTLS via our Istio service mesh with SPIFFE workload identity.
   Sensitive fields are encrypted at the application layer using
   envelope encryption with AWS KMS, giving defense in depth even
   against database admin access."
   This level of specificity distinguishes senior candidates.
```
