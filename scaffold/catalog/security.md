# Security

How the system authenticates users and services, authorizes access,
defines trust boundaries, and protects sensitive data — the design
decisions that shape where and how security is enforced in code.

**Sources:** OAuth 2.0 (IETF RFC 6749), NIST SP 800-162 (ABAC),
OWASP, Zero Trust Architecture (NIST SP 800-207),
Twelve-Factor App (Heroku)

---

## 1. Authentication Strategy

**When this matters:** When deciding how users and services prove their
identity. This decision constrains the entire system's session management,
scalability model, and integration surface.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Server-side sessions (session ID in cookie, state on server) | Simplicity, easy revocation (delete session), familiar pattern | Requires server-side state (limits horizontal scaling), CSRF vulnerability, cookie-based (cross-domain issues) |
| JWT tokens (stateless, self-contained, signed) | Stateless verification, scales horizontally, works across domains and services | Can't revoke until expiry (without a blocklist), token size grows with claims, must handle refresh flow |
| OAuth 2.0 / OpenID Connect (delegated auth via identity provider) | Third-party login, SSO, delegated access, enterprise integration | Redirect-based flow complexity, dependency on external IdP, more moving parts |
| API keys (static secret per consumer) | Simplest to implement, good for service-to-service or developer APIs | No user-level auth, keys don't expire by default, hard to rotate at scale, leak risk |
| mTLS (mutual TLS, certificate-based) | Zero-trust service-to-service auth, no shared secrets | Certificate management overhead, not suitable for end-user auth, complex infrastructure |

**Guiding questions:**
- Is this a browser app, mobile app, API, or service-to-service? (Browser → sessions or JWT with HttpOnly cookies. Mobile → JWT with secure storage. API for third parties → OAuth. Internal service-to-service → JWT or mTLS.)
- Do you need to support "Sign in with Google/GitHub/etc."? (If yes → OAuth 2.0 / OIDC is required.)
- How important is immediate revocation? (Sessions can be revoked instantly. JWTs live until they expire unless you maintain a blocklist, which reintroduces server state.)
- How many services need to verify identity? (One server → sessions are fine. Many services → JWT or OAuth avoids centralized session lookup on every request.)
- Don't roll your own auth. Use established libraries, IdPs (Auth0, Clerk, Keycloak), or framework-native auth modules.

**Related decisions:** Authorization Model (#2), Trust Boundary Design (#3), Communication & Interfaces catalog (API style)

---

## 2. Authorization Model

**When this matters:** When deciding how the system determines what an
authenticated entity is allowed to do. This is an architectural commitment
that becomes harder to change as the system grows — choosing the wrong
model doesn't break the system today, it breaks your ability to evolve
permissions tomorrow.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| RBAC (Role-Based Access Control — permissions grouped into roles) | Simplicity, mirrors org structure, easy to audit, well-understood | "Role explosion" as org grows, can't express contextual rules (time, location, resource ownership) |
| ABAC (Attribute-Based Access Control — policies evaluate attributes of user, resource, action, environment) | Fine-grained, context-aware, handles dynamic rules (time, location, device) | Complex to implement and audit, requires attribute infrastructure, harder to reason about |
| PBAC (Policy-Based Access Control — externalized policy engine evaluates declarative rules) | Centralized governance, policies versioned and audited independently of code, composes RBAC + ABAC | Requires policy engine infrastructure (OPA, Cedar, Oso), learning curve for policy language |
| ACL (Access Control List — per-resource list of who can do what) | Simple for small systems, fine-grained per-resource control | Doesn't scale — every resource needs its own list, hard to answer "what can user X access?" |
| Ownership-based (resource creator has full control, explicit sharing) | Natural for user-generated content, simple mental model | Limited to ownership semantics, doesn't handle complex organizational hierarchies |

**Guiding questions:**
- How stable is the organizational structure? (Stable roles → RBAC. Fluid, dynamic access needs → ABAC or PBAC.)
- Do access decisions depend on context beyond "who is this user"? (If time of day, device, resource sensitivity, or geographic location matter → ABAC or PBAC. If role alone is sufficient → RBAC.)
- How many distinct permission combinations exist? (Under ~20 roles → RBAC is manageable. Hundreds of role combinations → you have "role explosion" and need ABAC.)
- Does the domain require audit trails for access decisions? (PBAC with a centralized policy engine provides built-in decision logging.)
- Start with RBAC. Add ABAC attributes only when RBAC can't express a real business rule. Consider PBAC when authorization logic is scattered across too many services to govern.

**Related decisions:** Authentication Strategy (#1), Data Ownership (Data & State #10), Error Boundary Design (Errors & Resilience #3)

---

## 3. Trust Boundary Design

**When this matters:** When deciding where in the architecture security
checks are enforced. This determines whether you have a security perimeter
(verify at the edge, trust internally) or zero-trust (verify everywhere).

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Perimeter security (API gateway verifies auth, internal services trust each other) | Simplicity, single enforcement point, reduced internal overhead | One breach past the perimeter exposes everything, doesn't protect against insider threats |
| Zero trust (every service independently verifies identity and authorization) | Defense in depth, lateral movement prevention, secure even if one service is compromised | More infrastructure per service, token verification overhead, harder to develop and test locally |
| Hybrid (gateway handles authn, services handle authz) | Pragmatic — identity verified once, permissions checked where domain knowledge lives | Two systems to maintain, must pass identity context reliably between services |
| BFF / edge auth (authentication at the client-facing edge, backend services receive verified claims) | Clean separation, backend services stay simple, edge handles protocol translation | Edge becomes critical infrastructure, must propagate claims accurately |

**Guiding questions:**
- How many services exist? (Monolith → perimeter is the application boundary. Microservices → trust boundaries between every service pair become important.)
- What's the blast radius of a compromised service? (If one compromised service can access all data → perimeter-only is insufficient. Zero trust limits damage.)
- Where does authorization logic live? (If only the order service understands order permissions, the gateway can't enforce them → hybrid model where gateway handles authn, services handle authz.)
- How are secrets and credentials propagated between services? (JWT claims forwarded in headers, mTLS certificates, or service mesh identity — each has different trust implications.)
- Input validation is a trust boundary too. Never trust data from outside your service boundary — validate and sanitize at every entry point, even between internal services.

**Related decisions:** Authentication Strategy (#1), Authorization Model (#2), Service Boundaries (Structure & Composition #9)

---

## 4. Data Protection Strategy

**When this matters:** When deciding how sensitive data is protected at rest
and in transit. Regulatory requirements (HIPAA, GDPR, PCI-DSS, SOC 2)
often constrain these choices, but even without regulation, protecting
user data is a design responsibility.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| TLS everywhere (encrypt all data in transit) | Baseline protection against eavesdropping, widely supported, often required | Certificate management, slight performance overhead, doesn't protect data at rest |
| Encryption at rest (database-level or disk-level encryption) | Protects against physical theft or unauthorized disk access | Doesn't protect against application-level access, key management complexity |
| Field-level encryption (encrypt specific sensitive fields in application code) | Granular — only sensitive fields are encrypted, even DBAs can't read them | Application complexity, can't query encrypted fields, key rotation affects all records |
| Hashing (one-way, for passwords and verification tokens) | Passwords never stored in reversible form, breach impact reduced | Can't recover original value, must use slow algorithms (bcrypt, argon2) to resist brute force |
| Tokenization (replace sensitive values with opaque tokens, store mapping separately) | PCI-DSS compliance, sensitive data never enters most systems | Requires a token vault, adds a lookup hop, vault becomes critical infrastructure |

**Guiding questions:**
- What data is sensitive? (PII, PHI, financial data, credentials. Classify data before deciding how to protect it — not everything needs the same level of protection.)
- Are there regulatory requirements? (HIPAA → PHI must be encrypted at rest and in transit. PCI-DSS → card data must be tokenized or encrypted. GDPR → right to erasure means you need to know where personal data lives.)
- Who should be able to read sensitive data? (If even database administrators shouldn't see PII → field-level encryption. If only the application needs access → application-level encryption with keys in a secrets manager.)
- Passwords must always be hashed with bcrypt, argon2, or scrypt. Never encrypted (reversible), never SHA-256 (too fast to brute force), never stored in plain text.
- TLS in transit is non-negotiable for any production system. The question is what additional protection layers you add on top.

**Related decisions:** Trust Boundary Design (#3), Configuration Strategy (Identity & Conventions #3), Persistence Strategy (Data & State #4)
