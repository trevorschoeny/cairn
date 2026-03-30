# Resources & Performance

How the system manages computational resources — concurrency, expensive
object lifecycles, and where work is placed across the system.

**Sources:** Concurrency Models (Erlang/OTP, Go, Node.js communities),
Release It! (Michael Nygard), Twelve-Factor App (Heroku),
A Philosophy of Software Design (Ousterhout, 2018)

---

## 1. Concurrency Model

**When this matters:** When deciding how the system handles multiple
simultaneous operations. This is one of the most constraining decisions
in a codebase — it shapes how every piece of I/O, every request handler,
and every background task is written. Changing concurrency models after
significant code exists is a near-complete rewrite.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Thread-per-request (OS threads, shared memory) | True parallelism, familiar mental model, works for CPU-bound work | Memory overhead (~2MB per thread), shared state requires locks, race conditions and deadlocks |
| Async / event loop (single thread, non-blocking I/O) | Massive I/O concurrency with minimal memory, no locking | CPU-bound work blocks everything, callback/promise complexity, stack traces can be opaque |
| Actor model (isolated entities, message passing) | No shared state, natural fault isolation, scales across machines | Mindset shift, debugging message flows is hard, mailbox ordering is non-trivial |
| CSP / goroutines (lightweight threads, channel communication) | Lightweight concurrency + real parallelism, simple syntax | Language lock-in (Go), channels can deadlock if misused, less natural for distributed systems |
| Virtual threads / fibers (lightweight, runtime-managed) | Thread-per-request simplicity without the memory cost | Requires runtime support (Java 21+, Ruby 3+), ecosystem compatibility varies |

**Guiding questions:**
- Is the workload primarily I/O-bound or CPU-bound? (I/O-bound → async/event loop or actors. CPU-bound → threads, goroutines, or virtual threads. Mixed → hybrid with worker pools for CPU tasks.)
- Does the language constrain the choice? (Node.js → event loop. Go → goroutines. Erlang/Elixir → actors. Java → threads or virtual threads. Python → async or multiprocessing for CPU work due to GIL.)
- How important is fault isolation? (If one bad request shouldn't crash the process → actors or bulkheaded thread pools. If a single-process crash is acceptable → event loop is simpler.)
- Does the system need to scale across multiple machines? (Actors are designed for distribution. Threads and event loops are single-machine primitives — you'll need external coordination.)
- Don't mix concurrency models in the same codebase unless you have a very good reason. The cognitive overhead of two models is more than double the overhead of one.

**Related decisions:** Communication Pattern (Communication & Interfaces #2), Failure Philosophy (Errors & Resilience #6), Scaling approach (implicit — stateless for horizontal)

---

## 2. Resource Pooling & Lifecycle

**When this matters:** When the system uses expensive resources — database
connections, HTTP clients, thread pools, file handles, cryptographic
contexts — that are costly to create or limited in quantity. How these
resources are created, shared, and cleaned up affects both performance
and correctness.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Connection / object pooling (pre-create, check out, return) | Amortized creation cost, bounded resource usage, reuse across requests | Pool sizing is hard to tune, leaked resources exhaust the pool, stale connections |
| On-demand creation (create per use, dispose after) | Simplicity, no pool management, no stale resources | Creation overhead per operation, risk of resource exhaustion under load |
| Singleton (one instance, shared globally) | Zero creation overhead after startup, simple access | Shared mutable state risk, hard to test, lifecycle tied to process |
| Scoped / per-request (created at request start, disposed at request end) | Clean lifecycle, no leaks across requests, testable | Creation overhead per request, can't share across requests |
| Hierarchical scoping (singleton → scoped → transient, DI container manages) | Right lifetime for each resource, framework handles cleanup | DI container complexity, lifetime mismatches cause subtle bugs |

**Guiding questions:**
- How expensive is creating this resource? (Database connections take 10-50ms to establish. HTTP clients need TLS handshakes. If creation is cheap, on-demand is fine. If expensive, pool.)
- What's the maximum number of this resource the system can have? (Database connection limits are real — PostgreSQL defaults to 100. Pool size must respect downstream limits.)
- What happens if the resource leaks? (A leaked database connection eventually exhausts the pool and the system stops. A leaked in-memory object is just a minor GC burden. Pool resources that are dangerous to leak.)
- Does the framework provide lifecycle management? (Most web frameworks have request-scoped dependency injection. Use it rather than managing lifetimes manually.)
- The classic lifetime mismatch bug: injecting a scoped resource into a singleton. The singleton holds the reference forever, but the scoped resource expects to be disposed at request end.

**Related decisions:** Concurrency Model (#1), Data Access Pattern (Data & State #5), Errors & Resilience catalog (resource cleanup on failure)

---

## 3. Computation Placement

**When this matters:** When deciding where expensive or latency-sensitive
work executes. This determines the system's latency profile, cost
structure, and how responsive it feels to users.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Client-side (browser, mobile app) | Immediate responsiveness, reduced server load, works offline | Limited compute power, inconsistent environments, business logic exposed |
| Server-side (API handler, synchronous) | Centralized logic, consistent environment, access to data | Latency for users, server costs scale with traffic, blocking during compute |
| Background worker (queue-driven, async) | Non-blocking for users, retry-friendly, smooths load spikes | Delayed results, queue infrastructure, harder to debug, eventual consistency |
| Edge / CDN (serverless functions at network edge) | Ultra-low latency for global users, reduced origin load | Limited compute time, cold starts, restricted runtime environment, cache invalidation |
| Specialized service (offload to GPU, ML service, search engine) | Hardware-optimized for specific workloads | Network hop, additional infrastructure, another service to operate |

**Guiding questions:**
- Does the user need the result before proceeding? (If yes → client-side or synchronous server. If no → background worker.)
- How long does the computation take? (Under 100ms → synchronous is fine. 100ms-5s → consider async with loading state. Over 5s → background worker with notification.)
- Does the computation need access to server-side data or secrets? (If yes → server-side or background worker. If no → client-side is viable.)
- Is this computation the same for all users? (If yes → edge/CDN can cache the result. If personalized → must run per-request.)
- The common mistake: doing expensive work synchronously in a request handler when the user doesn't need the result immediately. Move report generation, email sending, image processing, and analytics to background workers.

**Related decisions:** Communication Pattern (Communication & Interfaces #2), Caching Strategy (Data & State #9), Recovery Strategy (Errors & Resilience #4)
