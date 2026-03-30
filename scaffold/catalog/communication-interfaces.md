# Communication & Interfaces

How components talk to each other — from system-level protocols
down to function signatures and API shapes.

**Sources:** REST (Roy Fielding, 2000), GraphQL (Facebook, 2015), gRPC (Google),
CQRS (Greg Young / Martin Fowler), Event-Driven Architecture (Azure Architecture Center),
API Design-First (OpenAPI / AsyncAPI communities), Postel's Law (RFC 793)

---

## 1. API Style

**When this matters:** When designing how consumers interact with your service
or module. This is one of the most visible and durable decisions — changing an
API style after clients depend on it is extremely expensive.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| REST (resource-oriented, HTTP verbs) | Universality, cacheability, browser compatibility, massive tooling ecosystem | Over/under-fetching, chatty for complex data, no built-in real-time |
| GraphQL (query language, single endpoint) | Client-controlled data fetching, fewer round trips, strongly typed schema | Server complexity, caching difficulty, N+1 query risk, potential for expensive queries |
| gRPC (binary RPC, protobuf, HTTP/2) | Performance, strong contracts, streaming, code generation | No native browser support, binary not human-readable, steeper learning curve |
| WebSocket (persistent bidirectional connection) | Real-time bidirectional communication, low latency | Stateful connections harder to scale, no built-in request/response semantics |
| Message queue / event bus (async, broker-mediated) | Full decoupling, resilience, buffering, fan-out to multiple consumers | No immediate response, eventual consistency, debugging complexity |

**Guiding questions:**
- Who are the consumers? (Browsers → REST or GraphQL. Internal microservices → gRPC. IoT devices → gRPC or MQTT. Third-party developers → REST is safest.)
- What are the data access patterns? (Predictable resources → REST. Complex, nested, variable → GraphQL. High-throughput, low-latency → gRPC.)
- Do you need real-time communication? (If yes → WebSocket, gRPC streaming, or SSE.)
- Does the consumer need an immediate response? (If no → message queue.)
- Can you mix styles? (Often the best answer: REST for public, gRPC internal, WebSocket for real-time, queue for background.)

**Related decisions:** Communication Pattern (#2), Serialization Format (#5), Service Boundaries (Structure & Composition #9)

---

## 2. Communication Pattern

**When this matters:** When deciding the temporal relationship between sender
and receiver. Determines whether components are coupled in time — whether
the sender must wait for the receiver.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Synchronous request-response | Simplicity, immediate feedback, strong consistency | Temporal coupling, cascading failures, sender blocks waiting |
| Asynchronous fire-and-forget | Maximum decoupling, resilience, sender never blocks | No confirmation of processing, harder to debug, eventual consistency |
| Asynchronous request-reply (correlation ID) | Decoupling + eventual response via callback | Complex correlation logic, timeout handling, harder to reason about |
| Streaming (uni- or bidirectional) | Continuous data flow, real-time, efficient for large datasets | Stateful connections, backpressure management, lifecycle complexity |
| Hybrid (sync for reads, async for writes) | Fast reads + resilient writes | Two models to maintain, potential inconsistency between paths |

**Guiding questions:**
- Does the caller need the result before proceeding? (If yes → synchronous. If no → async.)
- What happens if the receiver is temporarily down? (Sync fails immediately. Async with a queue buffers and retries.)
- How many services are in the call chain? (Each sync hop multiplies latency and reduces availability. Ten services at 99.9% = ~99% chain availability.)
- Can the user tolerate a brief delay before seeing the result? (If yes → async with notification.)
- Does the team have experience debugging distributed async systems? (Async requires tracing, correlation IDs, dead-letter queues.)

**Related decisions:** API Style (#1), Event Architecture (#3), Errors & Resilience catalog

---

## 3. Event Architecture

**When this matters:** When deciding whether and how components communicate
through events. The complexity ramp from "no events" to "full event sourcing"
is enormous — each step adds capability and cost.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| No events (direct calls only) | Simplicity, strong consistency, easy debugging | Tight coupling, hard to add consumers, no audit trail |
| Simple pub/sub (publish events, subscribers react) | Loose coupling, extensibility without changing publisher | Eventual consistency, ordering challenges, harder to trace |
| Event sourcing (state = immutable event log) | Complete audit trail, temporal queries, replay, debugging | Significant complexity, requires CQRS for reads, schema evolution is hard |
| Choreography (services react independently, no coordinator) | Maximum autonomy, no coordination single-point-of-failure | Hard to see overall flow, distributed debugging nightmare |
| Orchestration (central coordinator directs workflow) | Visible process flow, centralized error handling, easier to reason about | Coordinator is a bottleneck/SPOF, tighter coupling to coordinator logic |

**Guiding questions:**
- Do multiple independent consumers need to react to the same state change? (If yes → events. Single consumer → direct call is simpler.)
- Do you need a complete history of what happened and when? (Event sourcing provides this, but at significant cost.)
- Is the business process a well-defined workflow? (Orchestration makes flow visible. Choreography distributes it.)
- Can you tolerate eventual consistency? (If not → events alone won't suffice, you'll need sagas or distributed transactions.)
- Start with no events. Add pub/sub when a second consumer appears. Consider event sourcing only when audit trail or temporal queries are genuine requirements, not speculative.

**Related decisions:** Communication Pattern (#2), Temporal Modeling (Data & State #8), Coupling Strategy (Structure & Composition #5)

---

## 4. Contract Strategy

**When this matters:** When deciding how the interface between components is
defined and enforced. Determines whether breaking changes are caught before
deployment or discovered in production.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Schema-first / design-first (OpenAPI, protobuf, AsyncAPI → generate code) | Parallel development, early feedback, auto-generated docs + clients, schema is source of truth | Upfront design effort, tooling setup, risk of schema drift if discipline lapses |
| Code-first (write code, generate schema from annotations/types) | Speed to v1, code is single source of truth, natural for small teams | Consumers can't start until code exists, framework can influence API design, schema is afterthought |
| Consumer-driven contracts (consumers define needs, provider verifies) | Prevents breaking specific consumers, each consumer only tests what it uses | Requires contract testing infra (Pact, etc.), coordination overhead |
| Informal (README, wiki, Postman collection, verbal agreements) | Lowest ceremony, fastest to start | No enforcement, docs drift from reality, integration bugs found in production |

**Guiding questions:**
- How many teams consume this API? (One team → code-first is fine. Multiple → schema-first enables parallel work.)
- Is this public or internal? (Public APIs almost always need schema-first. Internal can start code-first.)
- Does the framework force a choice? (GraphQL and gRPC are inherently schema-first. REST can go either way.)
- Can you afford integration bugs in production? (Consumer-driven contracts catch them in CI.)
- How fast does the API need to ship? (Code-first is faster for v1. Schema-first pays dividends from v2 onward.)

**Related decisions:** API Versioning (#6), Backward Compatibility (#8), API Style (#1)

---

## 5. Serialization Format

**When this matters:** When deciding how data is encoded for transmission.
Affects performance, debuggability, interoperability, and schema evolution.
Often constrained by the API style choice.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| JSON | Human readability, universal support, browser-native, flexible | Verbose, no built-in schema enforcement, slower to parse than binary |
| Protocol Buffers (protobuf) | Compact binary, fast serialization, strong typing, backward-compatible evolution | Not human-readable, requires code generation, tooling setup |
| XML | Strictness, namespaces, mature validation (XSD), enterprise integration | Extremely verbose, complex parsing, largely superseded |
| MessagePack | JSON-like semantics in compact binary form | Less tooling than JSON or protobuf, smaller community |
| Avro (schema-embedded binary) | Schema evolution, compact, native to Kafka/Hadoop ecosystems | Requires schema registry, uncommon outside data engineering |

**Guiding questions:**
- Will humans need to read payloads for debugging? (JSON is readable anywhere. Protobuf requires specialized tools.)
- Is bandwidth or parsing speed a constraint? (Binary formats are 3-10x smaller and faster.)
- Does the API style constrain the choice? (gRPC requires protobuf. GraphQL returns JSON. REST typically uses JSON.)
- Do consumers span diverse languages and platforms? (JSON is universally supported. Protobuf has broad but not universal support.)
- Default to JSON unless you have measured evidence that binary formats are needed.

**Related decisions:** API Style (#1), Contract Strategy (#4), Resources & Performance catalog

---

## 6. API Versioning

**When this matters:** When you anticipate breaking changes to a published API.
The versioning strategy determines how consumers discover, adopt, and migrate
between API versions.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| URL path (/v1/users, /v2/users) | Visibility, cacheability, simple routing, easy testing (paste URL in browser) | Violates REST principle (same resource, different URLs), versions entire API at once |
| Custom header (Api-Version: 2, Stripe-Version: 2024-01-01) | Clean URLs, per-request version control, granular migration | Not visible in URLs or logs, harder to test casually, caching needs Vary header |
| Content negotiation (Accept: application/vnd.api.v2+json) | REST-purist, versions representations not resources, fine-grained | Complex implementation, less accessible, harder to test and cache |
| Query parameter (/users?version=2) | Simple to implement | Pollutes caching, non-standard, easy to forget |
| No versioning (additive-only changes, never break) | Simplicity, no version management overhead | Constrains API evolution forever, legacy fields accumulate |

**Guiding questions:**
- Is this a public API consumed by third parties? (URL versioning is most accessible. Stripe's header approach is the gold standard for granular control.)
- How often do you expect breaking changes? (Rarely → no versioning with additive-only evolution. Regularly → explicit versioning.)
- How many versions will coexist? (Support at most 2: current and previous. More becomes a maintenance burden.)
- Does your infrastructure support header-based routing? (Not all CDNs and proxies handle Vary headers well.)
- Give clients 6-12 months to migrate, then sunset old versions with clear deprecation notices.

**Related decisions:** Backward Compatibility (#8), Contract Strategy (#4), API Style (#1)

---

## 7. Interface Granularity

**When this matters:** When designing how much data and functionality is
exposed per API call. This determines the chattiness of the API, the
payload sizes, and how well the API fits different consumers' needs.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Coarse-grained (few calls, large payloads) | Fewer round trips, simpler client logic, good for high-latency networks | Over-fetching, wasted bandwidth, harder to cache granularly |
| Fine-grained (many calls, small payloads) | Precision, cacheability, composability, client flexibility | Chatty, more round trips, client must orchestrate multiple calls |
| Backend-for-Frontend / BFF (custom API per consumer type) | Each consumer gets exactly what it needs, optimized payloads | Multiple APIs to maintain, duplication of logic across BFFs |
| GraphQL-style (client specifies shape per request) | Maximum client flexibility without server-side BFFs | Server complexity, query cost analysis needed, caching is harder |
| Pagination (offset, cursor, or keyset for collections) | Bounded response sizes, predictable performance | Client must handle multi-page flows, cursor-based is harder to implement but scales better |

**Guiding questions:**
- How diverse are the consumers? (Single consumer → coarse-grained tailored to its needs. Many diverse consumers → fine-grained or GraphQL or BFF.)
- What's the network latency between client and server? (High latency, like mobile → fewer round trips, coarse-grained. Low latency, like same datacenter → fine-grained is fine.)
- For collections: how large can the result set get? (Unbounded → you must paginate. Cursor-based scales better than offset-based for large datasets. Offset allows random access.)
- Is the API primarily read-heavy or write-heavy? (Read-heavy with diverse consumers → GraphQL or BFF. Write-heavy → coarse-grained commands.)

**Related decisions:** API Style (#1), Data & State catalog (Caching Strategy, Normalization Level), Resources & Performance catalog

---

## 8. Backward Compatibility Strategy

**When this matters:** When evolving an API that has existing consumers.
This decision determines your contract with consumers about what kinds
of changes you'll make and how you'll communicate them.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Additive-only (never remove or rename, only add) | Consumer stability, no migration needed, simplest for consumers | Legacy fields accumulate forever, API becomes bloated over time, constrains design evolution |
| Tolerant reader / Postel's Law (be liberal in what you accept) | Resilience to minor changes on either side, graceful degradation | Can mask real errors, makes strict validation harder, consumers may depend on undocumented behavior |
| Strict versioning with deprecation windows | Clean breaks when needed, old API sunsets on schedule | Migration burden on consumers, multiple versions to maintain during transition |
| Consumer-driven contract testing (Pact, Spring Cloud Contract) | Only break what consumers actually use, fine-grained compatibility checking | Requires testing infrastructure, consumer participation, coordination overhead |
| Immutable APIs (never change, deploy new service for new version) | Maximum stability per version, zero migration | Service proliferation, operational overhead per version, harder to share infrastructure |

**Guiding questions:**
- How many consumers exist, and can you coordinate with all of them? (Few, known consumers → strict versioning with direct communication. Many unknown consumers → additive-only or tolerant reader.)
- What's the cost to consumers of a breaking change? (Enterprise clients on 6-month release cycles → long deprecation windows. Internal services you control → faster migration.)
- Can you define what "breaking" means precisely? (Adding a field is usually safe. Removing, renaming, or changing types breaks consumers. Changing behavior without changing the schema is the sneakiest kind of breakage.)
- Does your CI pipeline test compatibility? (Consumer-driven contracts automate this. Without automation, breaking changes slip through.)
- Stripe's approach is worth studying: they version via header with date-based versions, maintain backward compatibility within a version, and let consumers opt into new versions per-request.

**Related decisions:** API Versioning (#6), Contract Strategy (#4), Security catalog (deprecation of auth methods)
