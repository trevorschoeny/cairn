# Data & State

How data is modeled, stored, accessed, and managed — from system-level
persistence strategies down to individual data structure choices.

**Sources:** Domain-Driven Design (Eric Evans, 2003), CQRS (Greg Young / Martin Fowler),
Event Sourcing (Azure Architecture Center, microservices.io), A Philosophy of Software Design
(Ousterhout, 2018), Software Architecture in Practice (Bass et al.)

---

## 1. Schema Strategy

**When this matters:** When designing how data is structured before storage or
transmission. This decision affects how flexible the system is to change and
how confidently you can reason about data correctness.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Rigid schema (typed, enforced at compile/write time) | Data integrity, early error detection, tooling support (autocomplete, validation) | Schema changes require migrations, less flexible for rapidly evolving requirements |
| Flexible / schemaless (document, JSON blobs, key-value) | Rapid iteration, accommodating unknown or varying structure | Runtime errors, harder to query consistently, schema debt accumulates invisibly |
| Schema-on-read (store flexible, validate at consumption) | Storage flexibility + read-time guarantees | Pushes complexity to consumers, inconsistent data can accumulate in storage |
| Hybrid (rigid core + flexible extensions) | Stable core with room for variation (e.g., typed fields + metadata JSON column) | Two mental models, queries span both paradigms |

**Guiding questions:**
- How often will the data shape change? (Rapid evolution favors flexibility; stable domains favor rigidity.)
- What's the cost of a data integrity bug? (Healthcare, finance, and compliance domains strongly favor rigid schemas.)
- Are multiple consumers reading the same data with different expectations? (Schema-on-read may help, but adds consumer-side complexity.)
- Does the data have a well-understood structure, or is the team still discovering it? (Start flexible during discovery, solidify once the shape stabilizes.)

**Related decisions:** Persistence Strategy (#4), Normalization Level (#7), Communication & Interfaces catalog

---

## 2. Data Structure Selection

**When this matters:** When choosing the in-memory or storage representation
for a collection of items. This is one of the most common code-level design
decisions and one that agents most often make without thinking.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Array / List (ordered, indexed) | Sequential access, ordered data, simple iteration | Slow lookups by value (O(n)), slow insertion/deletion in middle |
| Set (unordered, unique) | Fast membership checks (O(1)), guaranteed uniqueness | No ordering, no duplicates (which may be needed), no index access |
| Map / Dictionary (key-value) | Fast lookup by key (O(1)), associating related data | Memory overhead per entry, no inherent ordering (language-dependent) |
| Stack (LIFO) | Undo/redo, expression parsing, backtracking algorithms | Only top element accessible |
| Queue (FIFO) | Task processing, buffering, breadth-first traversal | Only front element accessible |
| Tree (hierarchical) | Hierarchical data, sorted access (O(log n)), range queries | Complex implementation, rebalancing overhead |
| Graph (nodes + edges) | Relationship modeling, traversal algorithms, network structures | High memory usage, complex algorithms, harder to persist |

**Guiding questions:**
- What are the primary operations? (Frequent lookups → map/set. Ordered iteration → array/list. Hierarchy → tree.)
- Does ordering matter? (If yes, array or sorted tree. If no, set or map may be more efficient.)
- Are duplicates allowed? (If no, use a set. If yes, use a list.)
- What are the performance characteristics that matter? (O(1) lookup vs. O(1) insertion vs. O(log n) sorted access.)
- Does the data structure need to mirror a real-world concept? (People in a queue → queue. File directories → tree. Social network → graph.)

**Related decisions:** Schema Strategy (#1), Resources & Performance catalog

---

## 3. State Management Approach

**When this matters:** When deciding how application state is held, updated,
and observed. This decision has enormous impact on testability, debuggability,
and concurrency safety.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Mutable objects (state lives in objects, updated in place) | Familiarity, simplicity for small state, efficient memory use | Hard to track changes, concurrency bugs, difficult to test history |
| Immutable data (new state = new copy, old state preserved) | Predictability, concurrency safety, easy undo/time-travel, testability | Memory overhead (mitigated by structural sharing), unfamiliar to some teams |
| State machines (explicit states + transitions) | Correctness for complex workflows, impossible states are unrepresentable | Upfront design cost, overkill for simple state, can become unwieldy |
| Reactive / observable (state emits changes, consumers subscribe) | UI responsiveness, decoupled updates, real-time synchronization | Complex data flow debugging, potential for cascade/loop bugs |
| External store (state lives outside the application — DB, Redis, etc.) | Persistence, sharing across instances, survives restarts | Latency, serialization overhead, cache coherence challenges |

**Guiding questions:**
- Does the state have a well-defined set of valid states and transitions? (State machines prevent invalid states by construction.)
- Do multiple components need to observe state changes? (Reactive/observable patterns decouple producers from consumers.)
- Is concurrency a concern? (Immutable data eliminates a whole class of race conditions.)
- Do you need to undo, replay, or audit state changes? (Immutability or event sourcing provide history for free.)
- Is this state UI-local, session-scoped, or persistent? (Local → in-memory is fine. Persistent → external store.)

**Related decisions:** Programming Paradigm (Structure & Composition #1), Temporal Modeling (#8), Errors & Resilience catalog

---

## 4. Persistence Strategy

**When this matters:** When deciding where and how data lives at rest. This
is often the single most constraining data decision — changing a database
type after launch is one of the most expensive refactors.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Relational (SQL — PostgreSQL, MySQL, SQLite) | Data integrity, complex queries, transactions, joins, mature tooling | Rigid schema, vertical scaling limits, ORM impedance mismatch |
| Document (MongoDB, CouchDB, Firestore) | Flexible schema, natural mapping to application objects, horizontal scaling | Weaker consistency guarantees, no joins (or limited), denormalization required |
| Key-value (Redis, DynamoDB, etcd) | Ultra-fast reads/writes, simple data model, caching | No complex queries, limited relationships, data modeling burden on application |
| Graph (Neo4j, Amazon Neptune) | Relationship-heavy queries, traversals, network modeling | Niche tooling, less mature ecosystem, not great for tabular data |
| Time-series (InfluxDB, TimescaleDB) | Temporal data, metrics, IoT, high write throughput | Specialized use case, limited general-purpose querying |
| Embedded (SQLite, LevelDB, RocksDB) | Zero infrastructure, local-first, single-process simplicity | No network access, single-writer limitations, no built-in replication |

**Guiding questions:**
- What's the shape of the data? (Tabular with relationships → relational. Nested documents → document store. Highly connected → graph.)
- What are the query patterns? (Complex joins and aggregations → relational. Key-based lookups → key-value. Traversals → graph.)
- What are the consistency requirements? (Strong consistency → relational. Eventual consistency acceptable → document/key-value.)
- Is this a single-user local app or a multi-user networked service? (Local → embedded. Networked → server-based.)
- What's the team's existing expertise? (The database the team knows well often outperforms the "perfect" choice the team has to learn.)

**Related decisions:** Schema Strategy (#1), Data Ownership (#10), Normalization Level (#7)

---

## 5. Data Access Pattern

**When this matters:** When designing how application code reads and writes
persistent data. This is the bridge between your domain model and your storage.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Repository pattern (interface abstracts persistence) | Testability, swappable storage, clean domain model | More indirection, repository can become a god-interface |
| Active Record (entity knows how to save itself) | Simplicity, rapid development, low ceremony | Couples domain to persistence, hard to test without DB, mixed responsibilities |
| Data Access Object / DAO (dedicated persistence objects) | Separation of concerns, clear responsibility | Boilerplate, another layer to maintain |
| ORM (Object-Relational Mapper) | Productivity, type safety, migration tooling | Impedance mismatch, N+1 queries, magic behavior, hard to optimize |
| Raw queries (SQL, CQL, etc. directly in code) | Full control, optimizable, no abstraction overhead | SQL scattered through codebase, injection risk if not parameterized, hard to change DB |
| Query builder (programmatic query construction) | Type safety + flexibility, composable queries | Still coupled to DB dialect, less readable than raw SQL for complex queries |

**Guiding questions:**
- Could you realistically switch databases in the future? (If yes, repository pattern or DAO creates a clean swap boundary.)
- How complex are your queries? (Simple CRUD → Active Record or ORM. Complex analytics → raw queries or query builders.)
- Is testability a priority? (Repository pattern enables mock/stub persistence in tests without a real database.)
- Does the domain model have complex business logic? (If yes, keep it separate from persistence — repository pattern. If mostly CRUD, Active Record is pragmatic.)

**Related decisions:** Persistence Strategy (#4), Coupling Strategy (Structure & Composition #5), Dependency Direction (Structure & Composition #7)

---

## 6. Identity Strategy

**When this matters:** When deciding how entities are uniquely identified.
This affects URLs, database performance, API design, merge/sync behavior,
and whether IDs leak information.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| UUID v4 (random) | Global uniqueness without coordination, safe to generate client-side, no information leakage | Large (128 bits), poor index locality, not human-readable |
| Auto-increment integer | Compact, fast indexing, natural ordering, human-readable | Sequential (leaks creation order/volume), requires DB coordination, merge conflicts |
| ULID / UUID v7 (time-sortable + random) | Global uniqueness + time ordering + better index locality than UUIDv4 | Leaks approximate creation time, slightly larger than integers |
| Natural key (email, SSN, domain-specific code) | Meaningful, no lookup needed for the key itself | Can change (email), may have privacy implications (SSN), format constraints |
| Slug (human-readable string) | URL-friendly, SEO-friendly, memorable | Uniqueness enforcement, internationalization challenges, renaming complexity |
| Composite key (combination of fields) | No synthetic ID needed, natural for junction tables | Complex joins, harder to reference from other entities, unwieldy in URLs |

**Guiding questions:**
- Will IDs be generated client-side (offline-first, mobile)? (UUIDs or ULIDs don't need server coordination.)
- Will IDs appear in URLs? (Slugs for public-facing, UUIDs for API-facing, avoid auto-increment for security.)
- Do you need to merge data from multiple sources? (UUIDs avoid collision. Auto-increment doesn't.)
- Does the ID need to convey information (time ordering, tenant, type)? (ULIDs, composite keys, or prefixed IDs.)
- What's the database's indexing performance characteristic? (B-tree indexes prefer sequential keys — auto-increment or ULID over random UUID.)

**Related decisions:** Schema Strategy (#1), Security catalog (information leakage through IDs), Communication & Interfaces catalog (API URL design)

---

## 7. Normalization Level

**When this matters:** When designing how data is organized across tables,
collections, or storage units. This is the fundamental tradeoff between
write efficiency and read efficiency.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Fully normalized (3NF+, no duplication) | Data integrity, storage efficiency, single source of truth for each fact | Complex queries require joins, read performance suffers at scale |
| Partially denormalized (strategic duplication) | Read performance for common queries, fewer joins | Duplication means multiple places to update, consistency risk |
| Fully denormalized (all data for a read in one place) | Maximum read speed, simple queries, cache-friendly | Write complexity (update in many places), high storage cost, stale data risk |
| CQRS (separate read and write models) | Optimized reads AND writes, independent scaling | Eventual consistency between models, architectural complexity, synchronization logic |

**Guiding questions:**
- What's the read-to-write ratio? (Read-heavy → denormalize. Write-heavy → normalize. Both extreme → CQRS.)
- How important is immediate consistency? (If stale reads for a few seconds are acceptable, denormalization or CQRS become viable.)
- How many different query shapes do you need to serve? (Few patterns → targeted denormalization. Many diverse patterns → normalized with views or CQRS.)
- Is this a reporting/analytics workload or a transactional one? (Analytics often benefits from denormalized star schemas.)

**Related decisions:** Persistence Strategy (#4), Schema Strategy (#1), Temporal Modeling (#8)

---

## 8. Temporal Modeling

**When this matters:** When deciding how the system represents change over time.
This decision affects auditability, debugging, compliance, and the ability to
understand "what happened and when."

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Current-state-only (overwrite on update) | Simplicity, storage efficiency, straightforward queries | No history, can't answer "what was the value last Tuesday?", no audit trail |
| Soft deletes (mark as deleted, don't remove) | Recoverability, basic audit trail | Queries must filter deleted records, storage grows, "deleted" data persists |
| Audit log (separate log of changes alongside current state) | Auditability + simple current-state queries | Two systems to maintain, log can drift from reality if not transactional |
| Event sourcing (state is derived from a sequence of immutable events) | Complete history, replay, temporal queries, debugging, compliance | Significant complexity, requires CQRS for reads, eventual consistency, schema evolution is hard |
| Bitemporal (tracks both "when it happened" and "when we knew about it") | Regulatory compliance, corrections that preserve original history | Very complex, niche, requires specialized tooling or careful schema design |

**Guiding questions:**
- Does the domain have regulatory or compliance requirements for data history? (Healthcare, finance → event sourcing or bitemporal.)
- How important is debugging production issues? (Being able to replay "what happened" is invaluable — event sourcing or audit logs provide this.)
- Is the team willing to accept the complexity of event sourcing? (If not, an audit log alongside current state is a pragmatic middle ground.)
- Do users need to undo actions or see version history? (Event sourcing or snapshot-based versioning.)
- Could data corrections arrive after the fact? (Bitemporal handles "we learned on March 5th that the January value was wrong" — few other approaches can.)

**Related decisions:** State Management Approach (#3), Normalization Level (#7), Errors & Resilience catalog

---

## 9. Caching Strategy

**When this matters:** When read performance matters and you're willing to trade
some freshness or consistency for speed. Caching decisions are easy to get wrong
and hard to debug — a bad caching strategy causes subtle, intermittent bugs.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| No cache (always read from source) | Simplicity, guaranteed freshness, no invalidation bugs | Higher latency, more load on source, may not meet performance requirements |
| Cache-aside (application checks cache, falls back to source) | Control, flexibility, works with any storage | Application logic must manage cache, cache misses hit source, cold start problem |
| Read-through (cache sits in front of source, auto-populates) | Transparent to application, consistent cache behavior | Cache library/infrastructure dependency, less control over population logic |
| Write-through (write to cache and source simultaneously) | Cache always fresh, no stale reads | Write latency increases (two writes), more complex write path |
| Write-behind (write to cache, asynchronously sync to source) | Fast writes, reduced source load | Risk of data loss if cache fails before sync, eventual consistency |
| CDN / edge cache (cache at network edge) | Global performance, reduced origin load | Cache invalidation delay, different behavior by region, debugging complexity |

**Guiding questions:**
- How stale can the data be? (Real-time dashboards → no cache or very short TTL. Product catalog → hours of staleness may be fine.)
- What's the cache hit rate likely to be? (If most requests are unique, caching adds overhead without benefit.)
- What's your invalidation strategy? (TTL-based, event-driven, or manual? "There are only two hard things in CS: cache invalidation and naming things.")
- Can a stale cache cause correctness issues? (Showing a stale price is annoying. Showing a stale account balance is dangerous.)
- Start with no cache. Add caching only when you have measured evidence that it's needed.

**Related decisions:** Resources & Performance catalog, Normalization Level (#7), Persistence Strategy (#4)

---

## 10. Data Ownership

**When this matters:** When multiple services, modules, or teams need access
to the same data. This decision determines who is the authoritative source
and how others get access.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Shared database (all services read/write the same DB) | Simplicity, strong consistency, easy joins | Tight coupling (schema changes affect everyone), scaling bottleneck, unclear ownership |
| Database-per-service (each service owns its data store) | Loose coupling, independent evolution, team autonomy | No cross-service joins, data duplication, distributed transactions needed |
| Data mesh (domain-oriented, decentralized, data-as-a-product) | Scalable governance, domain team ownership, self-serve data infrastructure | Organizational complexity, requires mature data culture, tooling investment |
| Event-driven replication (services publish changes, others subscribe) | Eventual consistency across services, decoupled, extensible | Complexity, eventual consistency, event schema evolution |
| API-mediated access (services only access others' data through APIs) | Clear contracts, versioning, access control | Latency, availability dependency, chatty service interactions |

**Guiding questions:**
- How many teams work on this system? (Single team → shared DB is pragmatic. Multiple teams → consider per-service ownership to reduce coordination.)
- Do different parts of the system need different consistency guarantees? (Financial transactions need strong consistency; analytics can tolerate eventual.)
- Is there a single authoritative source for each piece of data? (If not, you'll have conflicting writes and no clear resolution strategy.)
- Could one service's schema change break another service? (If yes, coupling is too tight — consider API-mediated access or per-service databases.)
- Does the organization have a data platform team? (Data mesh requires investment in shared infrastructure and governance tooling.)

**Related decisions:** Service Boundaries (Structure & Composition #9), Persistence Strategy (#4), Communication & Interfaces catalog
