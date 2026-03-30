# Errors & Resilience

How errors are represented, validated against, handled across layers,
recovered from, and made visible — from code-level error types to
system-level fault tolerance.

**Sources:** Release It! (Michael Nygard, 2007/2018), Enterprise Craftsmanship (Vladimir Khorikov),
Domain-Driven Design (Eric Evans, 2003), Circuit Breaker (Martin Fowler), Resilience4j,
OpenTelemetry, Three Pillars of Observability (IBM/industry standard)

---

## 1. Error Representation

**When this matters:** When deciding how functions and methods signal failure.
This is one of the most pervasive decisions in a codebase — it shapes how
every caller handles every operation that can fail.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Exceptions (try/catch/finally) | Clean happy path, automatic propagation up call stack, familiar to most developers | Hidden control flow, unclear which functions throw, performance overhead when thrown, easy to over-catch |
| Result / Either types (Ok/Error) | Explicit in signatures, forces callers to handle both paths, composable, no hidden flow | Verbose, requires language/library support, can be tedious for deep call chains |
| Error codes / return values (C, Go style) | Simple, no runtime magic, visible in function signatures | Easy to ignore, no automatic propagation, mixes error and success in same channel |
| Optionals / nullable (Option, Maybe, null) | Simple for "absence" cases, lightweight | Can't convey why something failed, only that it did |
| Hybrid (Result for expected failures, exceptions for unexpected) | Best of both — explicit handling for business errors, automatic propagation for bugs | Two mental models, team must agree on which errors are "expected" vs "unexpected" |

**Guiding questions:**
- Does the language have strong Result type support? (Rust, Haskell, F# → Result types are idiomatic. Java, Python, JS → exceptions are idiomatic. Go → error return values.)
- Can the caller meaningfully recover from this failure? (If yes → Result type or error code, so recovery logic is explicit. If no → exception, let it propagate to a top-level handler.)
- How typed should errors be? (A single Error string is easy but uninformative. A hierarchy of typed error classes enables pattern-matching on failure kind. Error codes as strings enable cross-boundary communication without coupling to types.)
- The hybrid rule of thumb: Result for expected business failures (validation failed, resource not found, insufficient funds). Exceptions for unexpected infrastructure failures (database unreachable, null pointer, out of memory).

**Related decisions:** Error Boundary Design (#3), Validation Strategy (#2), Communication & Interfaces catalog (API error contracts)

---

## 2. Validation Strategy

**When this matters:** When deciding where and how input and state are checked
for correctness. Validation prevents errors from occurring in the first place —
it's the first line of defense.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Always-valid domain model (entities enforce their own invariants, cannot be constructed invalid) | Correctness by construction, impossible to have invalid objects in the system | Upfront design effort, may need separate "draft" entities for partial input, constructor complexity |
| Layered validation (presentation checks format, application checks business rules, domain guards invariants) | Defense in depth, each layer handles its own concern, clear error messages per layer | Validation logic can be duplicated across layers, harder to keep in sync |
| Schema validation at boundary (validate at API/input edge, trust internally) | Simple, single validation point, fast feedback to consumers | Internal code trusts that validation already happened — if bypassed, invalid state can creep in |
| Deferred validation (accept input tentatively, validate before committing) | Flexible for complex workflows, wizard-style forms, draft states | Risk of operating on invalid data, must remember to validate before commit |

**Guiding questions:**
- What's the cost of invalid data reaching the database or a downstream system? (High cost, like healthcare or finance → always-valid domain model + layered validation. Low cost → schema validation at boundary may suffice.)
- Does the domain have complex invariants that span multiple fields? (If yes → domain model should enforce them, not the presentation layer. "A delivery must have both an address and a time window" is a domain rule.)
- Do users need to save partial/draft state before completing a form? (If yes → deferred validation or separate "draft" entity with weaker constraints.)
- Where should error messages originate? (Domain layer should signal what's wrong via codes. Presentation layer should translate codes into user-facing messages.)

**Related decisions:** Error Representation (#1), Schema Strategy (Data & State #1), Module Depth (Structure & Composition #4)

---

## 3. Error Boundary Design

**When this matters:** When deciding where in the architecture errors are caught,
translated, and surfaced. Boundaries define who is responsible for handling
what kind of failure.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Per-layer translation (domain throws → application catches and translates → presentation formats) | Clean separation, each layer speaks its own error language, domain stays pure | More indirection, translation logic can become boilerplate |
| Centralized global handler (single top-level catch that maps all errors to responses) | Simple, one place to maintain, consistent error formatting | Loses context-specific recovery opportunities, can become a god-handler |
| Per-operation handling (each endpoint/handler manages its own errors) | Maximum control per use case, context-aware recovery | Inconsistent error handling across endpoints, duplication |
| Middleware / interceptor chain (errors flow through a pipeline of handlers) | Composable, cross-cutting concerns (logging, metrics) applied uniformly | Pipeline ordering matters, debugging through middleware layers is harder |

**Guiding questions:**
- How many distinct error types does the system produce? (Few → centralized handler is manageable. Many → per-layer translation gives better structure.)
- Do different consumers need different error formats? (API returns JSON error objects. CLI returns text. UI shows toast notifications. Per-layer translation enables this.)
- Should domain code know about HTTP status codes? (No. The presentation/API layer maps domain errors to HTTP codes. "Customer not found" → 404 is a presentation concern.)
- Don't catch exceptions you don't know what to do about. If you can't add value by catching, let it propagate.

**Related decisions:** Error Representation (#1), Responsibility Allocation (Structure & Composition #6), Communication & Interfaces catalog

---

## 4. Recovery Strategy

**When this matters:** When designing how the system responds to failures,
especially in distributed systems where partial failure is the norm.
These patterns determine whether a failing dependency cascades into a
system-wide outage.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Retry (with exponential backoff + jitter) | Handling transient failures (network blips, brief unavailability) | Can worsen overload if failure isn't transient, requires idempotent operations |
| Circuit breaker (closed → open → half-open state machine) | Preventing cascading failures, fast failure when dependency is down | Added state management, threshold tuning, stale circuit state |
| Bulkhead (isolated resource pools per dependency) | Blast radius containment — one slow dependency can't starve others | Resource overhead from maintaining separate pools, capacity planning complexity |
| Timeout (upper bound on wait time) | Preventing indefinite blocking, freeing resources | Too aggressive → false failures. Too lenient → slow cascading. Tuning is hard. |
| Fallback / graceful degradation (serve cached, default, or reduced data) | User experience continuity during partial failure | Fallback must be simpler and more reliable than what it replaces, stale data risk |
| Fail fast (reject immediately when preconditions aren't met) | Protecting resources, fast feedback, prevents wasted work | Aggressive — no chance of recovery, bad UX if overused |

**Guiding questions:**
- Is the operation idempotent? (If yes → retry is safe. If no → retry can cause duplicates or side effects. Make it idempotent first.)
- What's the expected failure mode? (Transient → retry. Persistent → circuit breaker. Overload → bulkhead + backoff.)
- What's acceptable degraded behavior? (Can you show cached data? Default recommendations? A "temporarily unavailable" message? If no fallback exists, fail fast is better than hanging.)
- These patterns compose: timeout + retry + circuit breaker is a common stack. Start with timeouts on all external calls, then add retry for transient failures, then circuit breaker when a dependency is chronically failing.
- A retry without an idempotency strategy can cause duplicates and inconsistent side effects. Design idempotency before adding retries.

**Related decisions:** Communication Pattern (Communication & Interfaces #2), Caching Strategy (Data & State #9), Resources & Performance catalog

---

## 5. Observability Approach

**When this matters:** When deciding how the system's behavior — especially
its failures — is made visible to operators and developers. You can't fix
what you can't see. Observability decisions made at design time determine
how debuggable the system is at 2 AM during an incident.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Structured logging (JSON or key-value, machine-parseable) | Queryable, filterable, correlatable across services, automation-friendly | Requires discipline and schema agreement across teams, slightly more verbose to emit |
| Unstructured logging (plain text, printf-style) | Simple, human-readable in terminal, low ceremony | Impossible to query at scale, brittle parsing, can't correlate across services |
| Metrics (counters, gauges, histograms — numeric time-series) | Fast dashboards, trend detection, alerting, SLO tracking | Limited context (tells you "what" but not "why"), high-resolution metrics are expensive to store |
| Distributed tracing (spans + trace IDs across service boundaries) | Visualizes request flow, pinpoints bottlenecks, maps dependencies | Instrumentation overhead, sampling decisions, storage cost |
| Three pillars combined (logs + metrics + traces, correlated) | Complete picture — "what" (metrics), "where" (traces), "why" (logs) | Significant infrastructure investment, requires team discipline, data volume management |

**Guiding questions:**
- Is this a monolith or distributed system? (Monolith → structured logging + metrics is usually sufficient. Distributed → you need tracing or debugging cross-service issues becomes guesswork.)
- Do you have a correlation ID strategy? (Every request should carry a trace/correlation ID through all services it touches. Plan this from day one — retrofitting observability into a decoupled system is much harder than building it in.)
- What will you need to know during an incident? (Think backwards from "the system is broken at 2 AM" — what information do you wish you had? Log that information now.)
- Structured logging is non-negotiable for any system beyond a toy project. The question is what else you add on top.
- Logs explain decisions and failures. Metrics tell you something is wrong. Traces show you where. You need all three to debug production systems efficiently.

**Related decisions:** Recovery Strategy (#4), Event Architecture (Communication & Interfaces #3), Resources & Performance catalog

---

## 6. Failure Philosophy

**When this matters:** When establishing the team's default posture toward
failure. This is a meta-decision that shapes all the others — it determines
whether your code assumes things will go right and handles exceptions, or
assumes things will go wrong and designs for it.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Defensive programming (validate everything, trust nothing) | Catching errors early, robustness against unexpected input | Verbose, can obscure business logic with checks, performance overhead from redundant validation |
| Fail fast (crash immediately on unexpected state) | Early detection, clean error signals, prevents operating on corrupt state | Aggressive — service goes down rather than limping. Requires good monitoring and restart infrastructure. |
| Tolerant / resilient (absorb failures, degrade gracefully) | User experience continuity, system stays up even when parts fail | Can mask real problems, complexity in fallback logic, risk of serving stale or wrong data |
| Let it crash (Erlang/OTP philosophy — don't recover, restart from known good state) | Simple error handling code, supervisor trees manage recovery, process isolation | Requires actor model or equivalent supervision infrastructure, unfamiliar to most teams |

**Guiding questions:**
- What's the cost of a wrong answer vs. no answer? (If wrong data is dangerous — healthcare, finance — fail fast. If availability matters more than precision — social media feed, recommendations — tolerate and degrade.)
- Does the system have supervision/restart infrastructure? (Kubernetes, systemd, Erlang supervisors → "let it crash" is viable. No auto-restart → fail fast means manual intervention.)
- How experienced is the team with distributed failure modes? (Defensive programming is the safest default. Tolerant/resilient requires understanding partial failure deeply.)
- These philosophies aren't mutually exclusive — different layers can adopt different postures. Domain logic might fail fast on invariant violations while the API layer degrades gracefully for consumers.
- The goal isn't to prevent all failures, but to handle them so gracefully that your users never notice.

**Related decisions:** Recovery Strategy (#4), Validation Strategy (#2), all other catalogs (this philosophy colors every decision)
