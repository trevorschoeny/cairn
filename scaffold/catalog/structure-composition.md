# Structure & Composition

How components relate to each other — from system-level service boundaries
down to class-level composition models.

**Sources:** Gang of Four (Gamma et al., 1994), SOLID Principles (Robert C. Martin, 2000),
A Philosophy of Software Design (John Ousterhout, 2018), Software Architecture in Practice
(Bass, Clements & Kazman, 4th ed.)

---

## 1. Programming Paradigm

**When this matters:** At the start of any project, or when adding a major new subsystem.
This is the most foundational structural decision — it constrains how you think about
composition, state, coupling, and extension for everything downstream.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Object-Oriented (OOP) | Encapsulation, modeling real-world entities, polymorphism | Can lead to deep hierarchies, shared mutable state, over-abstraction |
| Functional | Predictability, testability, composability, concurrency safety | Steeper learning curve, can be verbose for stateful I/O, less intuitive for domain modeling |
| Procedural | Simplicity, directness, low abstraction overhead | Poor encapsulation at scale, harder to manage complexity as system grows |
| Data-Oriented | Performance, cache efficiency, batch processing | Less intuitive modeling, tooling support varies, harder to express business rules |
| Hybrid / Multi-Paradigm | Flexibility — use the right tool per context | Inconsistency risk, requires team discipline to avoid paradigm soup |

**Guiding questions:**
- Does the problem domain map naturally to entities with behavior (OOP), data transformations (functional), or step-by-step procedures (procedural)?
- How important is concurrency safety? (Functional's immutability helps significantly here.)
- What does the team know? A paradigm the team is fluent in often beats a theoretically better one they're learning.
- Does the platform or framework strongly favor a paradigm? (e.g., React's shift toward functional components, Unity's data-oriented tech stack)
- Will this codebase be maintained by people with diverse backgrounds? (Hybrid can be pragmatic but needs clear conventions about when to use which style.)

**Related decisions:** Composition Model (#2), State management (see Data & State catalog), Extension Strategy (#8)

---

## 2. Composition Model

**When this matters:** Any time you're designing how components share or reuse behavior.
This is the decision that your Minecraft mod's proxy-vs-extender choice falls under.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Inheritance (is-a) | Code reuse, polymorphism, expressing taxonomies | Tight coupling to parent, fragile base class problem, rigid hierarchies |
| Composition (has-a) | Flexibility, modularity, runtime reconfiguration | More boilerplate (forwarding methods), no automatic polymorphism |
| Delegation | Same as composition, with explicit forwarding | Verbose, but makes control flow explicit |
| Mixins / Traits | Reuse across unrelated types, multiple "inheritance" without diamond problem | Ordering conflicts, implicit behavior, harder to trace execution |
| Proxies / Wrappers | Interception, cross-cutting concerns, compatibility across consumers | Indirection makes debugging harder, performance overhead for hot paths |

**Guiding questions:**
- Is the relationship truly "is-a" (a Dog is an Animal) or "has-a" (a Car has an Engine)? If you have to squint to see the "is-a," use composition.
- Will other codebases or third parties extend or wrap your components? (Proxies and composition are safer for public APIs.)
- Do you need runtime flexibility — swapping behavior without recompilation? (Composition and delegation win here.)
- How deep would the inheritance tree get? More than 2-3 levels is a strong signal to switch to composition.
- Does the language or framework nudge toward a specific model? (e.g., React hooks favor composition, Fabric modding favors mixins)

**Related decisions:** Extension Strategy (#8), Coupling Strategy (#5), Programming Paradigm (#1)

---

## 3. Module Decomposition Strategy

**When this matters:** When designing the top-level structure of a codebase —
deciding how to organize source files, directories, packages, or services.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| By feature / domain | Cohesion — everything related to a feature lives together | Cross-cutting concerns (logging, auth) don't have an obvious home |
| By layer | Separation of concerns — clear boundaries between presentation, logic, data | A single feature's code is scattered across many directories |
| By type / role | Discoverability for a specific kind of artifact (all controllers in one place) | Same scattering problem as by-layer, weaker cohesion |
| Hybrid (feature-first, layers within) | Pragmatic balance — features are self-contained, layers exist within each | Requires discipline to keep consistent; can drift toward one style |

**Guiding questions:**
- If a new developer joins and is asked to "add a filter to the search feature," how many directories would they need to touch? (Fewer is better.)
- Do features change independently? (Feature-based decomposition aligns code boundaries with change boundaries.)
- Is the codebase a monorepo with multiple apps? (Feature-based within each app, shared libraries by type/concern.)
- How large is the team? Larger teams benefit from feature-based decomposition because it reduces merge conflicts.

**Related decisions:** Service Boundaries (#9), Responsibility Allocation (#6)

---

## 4. Module Depth

**When this matters:** Every time you create a new module, class, function, or service.
This is Ousterhout's core insight: the interface-to-implementation ratio determines
whether a module reduces or increases system complexity.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Deep modules (simple interface, complex implementation) | Information hiding, reduced cognitive load for consumers, fewer dependencies | Harder to test internals, risk of god-modules if taken too far |
| Shallow modules (complex interface, simple implementation) | Granularity, unit testability, explicit control flow | Many small components increase total interface surface, can obscure the big picture |
| Context-dependent (deep for stable abstractions, shallow for volatile logic) | Balance — use depth where stability matters, shallowness where change is frequent | Requires judgment; no single rule applies everywhere |

**Guiding questions:**
- Is this module's interface simpler than its implementation? If the interface is as complex as the implementation, the module isn't hiding anything.
- Would a consumer of this module need to understand its internals to use it correctly? If yes, the abstraction is leaking.
- Are you creating many pass-through methods that just forward calls to another layer? That's a signal of shallow, unnecessary indirection.
- Is the Unix file I/O model (5 functions hiding thousands of lines) a good analogy for this module? If so, go deep.

**Related decisions:** Abstraction Design (#10), Composition Model (#2)

---

## 5. Coupling Strategy

**When this matters:** Every time one component needs to use another. The coupling
strategy determines how tightly bound components are, how far changes propagate,
and how independently components can evolve.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Direct dependency (concrete references) | Simplicity, performance, type safety | High coupling — changes propagate, hard to substitute |
| Interface / protocol-based | Substitutability, testability (mocking), independent evolution | Added abstraction layer, can be over-engineered for stable internals |
| Event-based / message-passing | Maximum decoupling, async-friendliness, extensibility | Hard to trace execution, eventual consistency, debugging complexity |
| Dependency injection | Testability, configurability, inversion of control | Framework overhead, implicit wiring can obscure what depends on what |
| Service calls (HTTP, RPC, IPC) | Process-level isolation, independent deployment | Network latency, failure modes, serialization overhead |

**Guiding questions:**
- How often does the depended-upon component change? (Stable components can tolerate direct coupling; volatile ones need abstraction.)
- Do you need to substitute this dependency in tests? (If yes, interface-based or DI.)
- Should the consumer know about the producer, or should they be fully decoupled? (Events for full decoupling, interfaces for partial.)
- Is this an internal module boundary or a system boundary? (System boundaries almost always need looser coupling.)
- Would the Law of Demeter (only talk to your immediate neighbors) be violated? If A reaches through B to call C, coupling is too tight.

**Related decisions:** Dependency Direction (#7), Communication & Interfaces catalog, Service Boundaries (#9)

---

## 6. Responsibility Allocation

**When this matters:** When deciding what logic belongs in which module. Poor
allocation leads to god-modules (too much responsibility) or anemic modules
(no real logic, just data containers).

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Single Responsibility Principle (one reason to change) | Focused modules, easy to understand and test individually | Can lead to many tiny modules if interpreted too strictly |
| Feature cohesion (everything for one use case together) | Self-contained features, minimal cross-module coordination | Duplication of shared logic across features |
| Domain-driven (aggregate roots, bounded contexts) | Business alignment, rich domain model, ubiquitous language | Higher upfront design cost, overkill for simple CRUD |
| Data-centric (modules organized around data they own) | Clear data ownership, fewer shared-state bugs | Business logic can get scattered across data-owning modules |
| Actor-based (each module owns its state and mailbox) | Concurrency safety, fault isolation, location transparency | Messaging overhead, harder to reason about synchronous flows |

**Guiding questions:**
- If this module's requirements change, how many other modules would need to change too? (The answer should ideally be zero or one.)
- Does this module have a single, clear answer to "what is this for?" If you need "and" to describe it, it might have too many responsibilities.
- Is there logic duplicated across multiple modules that should be extracted into a shared responsibility?
- Are there data fields that are always read/written together? They probably belong in the same module.
- Would a new team member be able to guess what this module does from its name alone?

**Related decisions:** Module Decomposition (#3), Module Depth (#4), Programming Paradigm (#1)

---

## 7. Dependency Direction

**When this matters:** When structuring how modules reference each other. The direction
of dependencies determines which modules can change independently and which are
coupled to others' evolution.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Top-down (high-level depends on low-level) | Intuitive, simple, matches call flow | High-level policy is coupled to low-level details; hard to reuse or test policy independently |
| Inverted (both depend on abstractions — DIP) | High-level policy is independent of implementation details, testable, swappable | More interfaces/abstractions to maintain, indirection |
| Acyclic / layered (strict one-way dependency flow) | Predictable build order, no circular dependencies, clear layering | Requires discipline; sometimes natural relationships are bidirectional |
| Plugin architecture (core defines interfaces, plugins implement) | Extensibility without modifying core, third-party contributions | Plugin API becomes a stability contract, versioning complexity |

**Guiding questions:**
- If you draw the dependency graph, are there cycles? Cycles make it impossible to build, test, or reason about modules independently.
- Does your business logic depend on infrastructure details (database, HTTP, filesystem)? If yes, consider inverting those dependencies.
- Could the low-level module be swapped out (e.g., switching databases, changing a vendor SDK)? If that swap would require changes to business logic, dependencies are flowing the wrong way.
- Is this a library meant for reuse? (Libraries should have dependencies flowing inward, never outward toward consumers.)

**Related decisions:** Coupling Strategy (#5), Extension Strategy (#8), Composition Model (#2)

---

## 8. Extension Strategy

**When this matters:** When you need to plan for future functionality that doesn't
exist yet. How will new capabilities be added without breaking existing ones?

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Modification (change existing code) | Simplicity for small systems | Violates Open-Closed Principle, risk of regression, no parallel development |
| Subclassing / overriding | Language-level support, familiar pattern | Tight coupling to parent, fragile base class, deep hierarchies |
| Interface implementation (new implementations of existing contracts) | Open-Closed compliance, parallel development, substitutability | More upfront abstraction work, interface must be designed for extension |
| Configuration / feature flags | Runtime flexibility, A/B testing, gradual rollout | Configuration sprawl, dead code accumulation, testing combinatorial explosion |
| Plugin / hook system | Third-party extensibility, core stability | API surface becomes a contract, versioning complexity, security considerations |
| Composition (new capabilities assembled from existing components) | Maximum flexibility, no existing code changes | Requires well-designed composable components upfront |

**Guiding questions:**
- Will third parties need to extend this system, or only your own team? (Third-party extension almost always requires plugin/hook/interface approaches.)
- How frequently will new capabilities be added? (High frequency favors configuration or composition; low frequency can tolerate modification.)
- Does the extension need to happen at runtime (feature flags, plugins) or is compile-time sufficient (new implementations)?
- What's the blast radius if an extension introduces a bug? (Plugins and composition isolate blast radius better than subclassing.)

**Related decisions:** Dependency Direction (#7), Composition Model (#2), Coupling Strategy (#5)

---

## 9. Service / Component Boundaries

**When this matters:** When deciding whether a system should be one deployable unit
or many, and where the boundaries fall. This is the highest-level structural decision
and has cascading effects on team structure, deployment, and operational complexity.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Monolith | Simplicity, low operational overhead, easy debugging, transactional consistency | Scaling requires scaling everything, deployment couples all features, codebase can become unwieldy |
| Modular monolith | Monolith benefits + internal module boundaries enforced in code | Requires discipline to maintain boundaries, can degrade into a big ball of mud |
| Microservices | Independent deployment, team autonomy, technology flexibility, granular scaling | Network complexity, distributed transactions, operational overhead, debugging across services |
| Serverless / Functions | Pay-per-use, auto-scaling, zero server management | Cold start latency, vendor lock-in, hard to test locally, limited execution duration |
| Hybrid (monolith core + extracted services) | Pragmatic — start simple, extract when pain justifies complexity | Boundary between monolith and services needs careful management |

**Guiding questions:**
- How many developers will work on this system simultaneously? (Conway's Law: system boundaries tend to mirror team boundaries.)
- Do different parts of the system have fundamentally different scaling needs? (If yes, those are natural service boundary candidates.)
- Can you afford the operational complexity of distributed systems? (Monitoring, tracing, deployment pipelines per service.)
- Is this a new system or an existing one? (Start monolithic, extract services when you have evidence of where the boundaries should be.)
- Does the domain have natural bounded contexts (DDD) where the same word means different things? (Those are strong boundary candidates.)

**Related decisions:** Module Decomposition (#3), Coupling Strategy (#5), Communication & Interfaces catalog

---

## 10. Abstraction Design

**When this matters:** When designing any public interface — a function signature,
a class API, a module's exports, or a service contract. The abstraction is the
promise you make to consumers about what your module does without revealing how.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| General-purpose (broad, reusable interface) | Reusability, stability (fewer reasons to change), information hiding | Harder to design upfront, may not fit any specific use case perfectly |
| Special-purpose (narrow, specific interface) | Fits one use case perfectly, easier to design | Low reusability, leaks assumptions about the caller, changes with each new use case |
| Progressive disclosure (simple default, advanced options available) | Ease of use for common cases + power for advanced cases | API surface is larger, documentation burden, risk of inconsistency between simple and advanced paths |
| Convention-based (implicit contracts via naming, structure, or documentation) | Minimal boilerplate, flexibility | Fragile — nothing enforces the contract, easy to violate accidentally |

**Guiding questions:**
- What would a consumer need to know to use this correctly? That's your interface. Everything else is implementation.
- Is the abstraction hiding the right things? (Omitting important details creates a "false abstraction" — Ousterhout's term for one that looks simple but forces consumers to know implementation details anyway.)
- Could this interface serve more than one consumer? If the interface is tightly shaped to one caller, it's probably too special-purpose.
- Would a new consumer be able to use this interface from the documentation alone, without reading the implementation? If not, the abstraction is leaking.
- Does the interface use domain vocabulary (meaningful to the problem) or implementation vocabulary (meaningful to the code)? Domain vocabulary creates more durable abstractions.

**Related decisions:** Module Depth (#4), Coupling Strategy (#5), Identity & Conventions catalog
