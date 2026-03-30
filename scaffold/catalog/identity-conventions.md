# Identity & Conventions

How things are named, organized, configured, tested, and versioned —
the decisions where consistency matters more than which option you pick,
but agents must know the convention to follow it.

**Sources:** Clean Code (Robert C. Martin, 2008), Implementing Domain-Driven Design
(Vaughn Vernon, 2013), Vertical Slice Architecture (Jimmy Bogard),
Semantic Versioning (semver.org), Twelve-Factor App (Heroku)

---

## 1. Code Organization Strategy

**When this matters:** When deciding how files and modules are grouped in the
project structure. This determines where a developer (or agent) puts a new
file, and how quickly anyone can find code related to a specific feature
or concern.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Layer-first (group by technical role: controllers/, services/, models/) | Clear separation of concerns, easy to find all code of a given type, familiar to most developers | Features are scattered across directories, touching one feature requires changes in many folders, hard to extract or delete a feature |
| Feature-first (group by business capability: users/, orders/, billing/) | High cohesion, easy to find/add/remove entire features, natural path to microservice extraction | Shared code needs a separate "common" or "shared" directory, can duplicate layer structure inside each feature |
| Hybrid: feature-first with layers inside (orders/service.ts, orders/controller.ts) | Best of both — features are cohesive, layers are visible within each feature | Deeper nesting, layer names repeated across features |
| Domain-based (DDD bounded contexts as top-level grouping) | Reflects business language, strong encapsulation per context, explicit boundaries | Requires upfront domain modeling, overkill for simple CRUD apps |

**Guiding questions:**
- How many features does the project have? (Few → layer-first is manageable. Many → feature-first prevents directory sprawl per layer.)
- How many developers work on the project simultaneously? (Multiple teams → feature-first reduces merge conflicts because teams work in separate directories.)
- Is there a realistic path to microservices? (Feature-first makes extraction straightforward — each feature is already self-contained.)
- Layer-first works well for small projects and tutorials. Most production systems that start layer-first eventually migrate toward feature-first as they grow.

**Related decisions:** Module Decomposition (Structure & Composition #3), Service Boundaries (Structure & Composition #9), Test Organization (#4)

---

## 2. Naming Patterns

**When this matters:** When establishing how files, modules, functions, variables,
and types are named. Agents silently get naming wrong more than almost any
other convention — they default to their training data's majority pattern,
which may not match your project.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Domain-concept names (Patient, Appointment, ClaimSubmission) | Readability, ubiquitous language (DDD), code mirrors business conversations | Requires domain knowledge, names may be long, domain terms can be ambiguous across contexts |
| Technical-role names (PatientController, AppointmentService, ClaimRepository) | Instantly clear what a file does technically, easy to locate by role | Can obscure what the code is *for*, encourages thinking in layers rather than features |
| Hybrid: domain noun + technical suffix (PatientService, ClaimValidator) | Both intent and role are visible, most common in practice | Verbose, suffix conventions must be agreed on and followed consistently |
| Action-based names for functions (createPatient, submitClaim, validateAddress) | Clear intent, reads like documentation, good for APIs | Naming consistency across team is harder — create vs add vs register vs insert |

**Sub-decisions to record (these are the conventions agents most often violate):**
- **Casing:** camelCase, PascalCase, snake_case, kebab-case — and which applies to files, variables, classes, constants, database columns, API fields. (Often language-determined, but not always — e.g., a TypeScript project might use camelCase for variables but kebab-case for file names.)
- **Singular vs plural:** Is the directory `model/` or `models/`? Is the table `user` or `users`? Is the API endpoint `/user` or `/users`?
- **Abbreviation policy:** Is it `authentication` or `auth`? `configuration` or `config`? `repository` or `repo`? Pick a line and document it.
- **Prefix/suffix conventions:** Do interfaces start with `I`? Do implementations end with `Impl`? Do test files end with `.test.ts` or `.spec.ts`?
- **Verb vocabulary:** Standardize the verbs: create/read/update/delete? Or add/get/set/remove? Or insert/fetch/modify/purge? Pick one set.

**Guiding questions:**
- What does the language community convention dictate? (Follow it unless you have a strong reason not to. Python uses snake_case. Java uses camelCase. Go uses PascalCase for exports.)
- Would a new team member know where to put a new file and what to name it without asking? (If not, the naming conventions aren't documented well enough.)
- Can you enforce naming conventions with linters? (ESLint, Pylint, etc. can catch casing violations. Automate what you can.)

**Related decisions:** Code Organization Strategy (#1), all catalogs (naming affects how agents interpret and create code everywhere)

---

## 3. Configuration Strategy

**When this matters:** When deciding how application settings, secrets, and
environment-specific values are managed. This affects deployability,
security, and how easily the application adapts to different environments.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Environment variables (12-Factor App style) | Portability, no secrets in code, container-friendly, language-agnostic | No structure or typing, hard to see all config at a glance, easy to misspell a key |
| Config files (JSON, YAML, TOML, .env) | Structured, readable, version-controllable (for non-secrets), easy to see all settings | Must not contain secrets in version control, format parsing overhead, env-specific files can proliferate |
| Feature flags (LaunchDarkly, Unleash, or custom) | Runtime toggling without deploys, gradual rollouts, A/B testing | External dependency, flag debt accumulates, adds indirection to code |
| Hardcoded defaults with overrides | Simple, works without external config, sensible defaults reduce config burden | Changing defaults requires code changes and redeployment |
| Hierarchical / layered config (defaults → env file → env vars → CLI args) | Flexible, predictable precedence, works across all environments | Debugging "where did this value come from?" can be hard |

**Guiding questions:**
- Where do secrets live? (Never in version control. Environment variables, a secrets manager like Vault/AWS Secrets Manager, or encrypted .env files that are gitignored.)
- How many environments do you have? (Dev, staging, production at minimum. Each needs its own configuration without code changes.)
- Do non-developers need to change configuration? (If yes → feature flags or admin UI. If no → config files or env vars are sufficient.)
- The Twelve-Factor App rule: store config in the environment. Config is anything that varies between deploys — credentials, resource handles, per-deploy values. Code does not vary between deploys.

**Related decisions:** Security catalog (secrets management), Errors & Resilience catalog (feature flags for graceful degradation)

---

## 4. Test Organization

**When this matters:** When deciding where tests live, how they're named, and
what kind of testing happens at which level. Agents need to know this to
create tests in the right place with the right naming pattern.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Co-located (tests next to source: user.ts + user.test.ts) | Easy to find the test for any file, tests move when source moves, visible coverage gaps | Test files mixed with source can feel cluttered, harder to run "all tests" in some toolchains |
| Separate test tree (mirrored directory: src/user.ts + test/user.test.ts) | Clean source directories, test infrastructure isolated, familiar to Java/C# developers | Tests can drift from source structure, harder to spot missing tests, more navigation |
| Hybrid (unit tests co-located, integration/e2e tests in separate directory) | Unit tests stay close to code, broader tests have their own space with shared fixtures | Two conventions to remember, boundary between "unit" and "integration" can be fuzzy |

**Sub-decisions to record:**
- **File naming:** `.test.ts` vs `.spec.ts` vs `_test.go` — this is the single most common thing agents get wrong about tests.
- **Test naming pattern:** `describe/it` (BDD), `test_function_name_scenario_expected` (verbose), or free-form.
- **What gets tested at each level:** Unit = single function/class in isolation. Integration = multiple components together. E2E = full user flow through real infrastructure.

**Guiding questions:**
- Does the framework dictate test location? (Go requires `_test.go` files next to source. Jest can be configured either way.)
- How do you run a single feature's tests? (Co-located makes this trivial. Separate tree requires path mapping.)
- Is test coverage measured and enforced? (If yes, co-located tests make coverage gaps more visible during code review.)

**Related decisions:** Code Organization Strategy (#1), Naming Patterns (#2)

---

## 5. Project Versioning

**When this matters:** When deciding how the project communicates its
release history and compatibility promises. This affects package
publishing, changelog generation, deployment strategies, and how
consumers (or dependent services) know what changed.

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| Semantic Versioning / semver (MAJOR.MINOR.PATCH) | Clear compatibility signal, widely understood, tooling support (npm, cargo, etc.) | Requires discipline to classify changes correctly, major bumps can cause consumer anxiety |
| Calendar Versioning / calver (YYYY.MM.DD or YYYY.MM.PATCH) | Human-readable release dates, no need to classify change severity | No compatibility signal, consumers can't tell if upgrade is safe |
| Git-tag-only (no formal version number, tags mark releases) | Simple, no version file to maintain, works for internal services with few consumers | No version metadata in the artifact itself, harder for consumers to pin dependencies |
| No formal versioning (trunk-based, continuous deployment) | Maximum deployment velocity, every commit is a release | No rollback target, hard to communicate what changed, not viable for published packages |

**Guiding questions:**
- Is this a published package that others depend on? (If yes → semver is nearly mandatory. Consumers need compatibility signals.)
- Is this an internal service with continuous deployment? (If yes → calver or git-tag-only may be simpler. Nobody pins to a version of your internal API.)
- Do you need to maintain multiple release branches? (Semver + git tags makes this manageable. No versioning makes it chaotic.)
- How do you generate changelogs? (Conventional Commits + semver enables automated changelog generation. Without structured commits, changelogs are manual.)
- Semver rule of thumb: if you're not sure whether a change is breaking, it probably is. Bump major.

**Related decisions:** Backward Compatibility (Communication & Interfaces #8), API Versioning (Communication & Interfaces #6)
