# Cairn — Technical Plan

**Architectural Knowledge Management for AI-Assisted Development**

*"Stacked stones marking a trail. You're leaving markers for whoever comes through this code next."*

---

## 1. What Cairn Is

Cairn is an npm package that installs a set of Claude Code skills, slash commands, directory conventions, and a decision reference catalog into any project. It gives AI coding agents the ability to:

- **Plan** collaboratively with a human, systematically surfacing architectural and design decisions
- **Record** those decisions in structured formats co-located with code
- **Discover** implicit decisions in existing brownfield codebases
- **Review** whether implementation agents respected recorded decisions
- **Implement** code with decision-awareness baked into the agent's operating instructions

Cairn is harness-agnostic. It composes with Ralph, Task Master, Ruflo, or a plain Claude Code session. It doesn't replace any workflow — it adds an AKM layer to whatever workflow you're already using.

---

## 2. Installation & Initialization

### Install

```bash
npx cairn init
```

### What `cairn init` Does

1. Creates the `.cairn/` directory structure:

```
.cairn/
├── decisions/              # Decision records (numbered .md files)
│   └── _template.md        # MADR-based template for new decisions
├── catalog/                # Decision reference catalog
│   ├── architecture.md     # System-level decision categories
│   ├── design.md           # Code-level decision categories
│   ├── data.md             # Data structure & modeling decisions
│   ├── integration.md      # API shape, protocols, boundaries
│   ├── error-handling.md   # Error philosophy decisions
│   └── naming.md           # Naming & convention decisions
├── config.yaml             # Cairn configuration
└── coverage.yaml           # Tracks which modules have been through discovery
```

2. Installs Claude Code skills into `.claude/skills/`:

```
.claude/skills/
├── cairn-planning/
│   └── SKILL.md            # Planning phase skill
├── cairn-implementation/
│   └── SKILL.md            # Implementation phase skill
├── cairn-discovery/
│   └── SKILL.md            # Brownfield discovery skill
└── cairn-review/
    └── SKILL.md            # Decision compliance review skill
```

3. Installs Claude Code slash commands into `.claude/commands/`:

```
.claude/commands/
├── cairn-plan.md           # Start a planning session
├── cairn-discover.md       # Run brownfield discovery on a module
├── cairn-check.md          # Audit decision compliance
└── cairn-status.md         # Show AKM coverage across the project
```

4. Adds a `CAIRN.md` to the project root (brief pointer file for agents):

```markdown
# Cairn — Architectural Knowledge Management

This project uses Cairn to track architectural and design decisions.

- Decision records: `.cairn/decisions/`
- Module-level decisions: `DECISIONS.md` in each module directory
- Inline annotations: `@cairn` comments in source files

Before modifying any code:
1. Check `.cairn/decisions/` for decisions scoped to the files you're changing.
2. Check the module's `DECISIONS.md` if one exists.
3. Look for `@cairn` annotations in the source files.
4. If you encounter a pattern you don't understand, flag it — don't change it.

See `.cairn/config.yaml` for project-specific Cairn settings.
```

5. Appends a reference to `CAIRN.md` in the project's `CLAUDE.md` (creates one if it doesn't exist):

```markdown
## Architectural Knowledge Management
This project uses Cairn. Read CAIRN.md before making any changes.
@CAIRN.md
```

### Configuration: `.cairn/config.yaml`

```yaml
version: 1

# Planning preset: "developer" or "guided"
#   developer — assumes familiarity with design/architecture terms
#   guided — explains each concept and its implications
preset: developer

# Annotation style for inline code annotations
annotation:
  format: comment           # "comment" for inline code comments
  prefix: "@cairn"           # The annotation prefix in source files
  link_to_decisions: true    # Include path to full decision record

# Module-level decision files
module_decisions:
  filename: DECISIONS.md     # Name of co-located decision files
  auto_create: true          # Agent creates DECISIONS.md when entering undocumented module

# Coverage tracking
coverage:
  track: true
  file: .cairn/coverage.yaml

# Decision ID format
decision_id:
  prefix: ""                 # Optional project prefix (e.g., "PEBBLE-")
  auto_increment: true
```

---

## 3. Decision Record Format

Based on MADR, extended for Cairn's needs. Stored in `.cairn/decisions/` as numbered markdown files.

### Template: `.cairn/decisions/_template.md`

```markdown
# {DECISION_ID} — {Title}

**Status:** proposed | accepted | superseded | deprecated
**Date:** {YYYY-MM-DD}
**Supersedes:** {ID, if applicable}
**Superseded by:** {ID, if applicable}

## Context

What is the problem or situation that requires a decision?
What constraints exist? What are we optimizing for?

## Options Considered

### Option 1: {Name}

{Description}

**Pros:**
- {advantage}

**Cons:**
- {disadvantage}

**Optimizes for:** {quality attribute — e.g., maintainability, performance, compatibility}

### Option 2: {Name}

{...}

### Option 3: {Name}

{...}

## Decision

{Which option was chosen and why.}

## Anti-Patterns

{What NOT to do, and why not. This section is critical for agent consumption.
An agent that only reads the decision might find a "better" way that violates
the reasoning. This section makes the boundaries explicit.}

- Do NOT {specific anti-pattern}. Reason: {why this breaks the decision's intent}.
- Do NOT {specific anti-pattern}. Reason: {why this breaks the decision's intent}.

## Consequences

**Positive:**
- {benefit}

**Negative:**
- {tradeoff accepted}

## Scope

**Files/modules affected:**
- {path or pattern}

**Related decisions:**
- {DECISION_ID} — {brief relationship description}
```

---

## 4. Inline Annotation Format

Cairn annotations are structured comments placed in source files at the point where a decision is most visible — class declarations, function signatures, data structure definitions, module entry points.

### Standard Annotation

```
// @cairn {DECISION_ID} {short-name}
// @reason {One-line summary of WHY, not what}
// @see .cairn/decisions/{DECISION_ID}.md
```

### Extended Annotation (for critical decisions)

```
// @cairn {DECISION_ID} {short-name}
// @choice {The specific option chosen}
// @reason {Why this choice was made — the business/architectural logic}
// @anti-pattern Do NOT {what not to do}. {Why not.}
// @see .cairn/decisions/{DECISION_ID}.md
```

### Brownfield-Specific Annotations

```
// @cairn-unknown {short-name}
// @observation {What the pattern is and why it looks deliberate}
// @status undocumented — do not refactor without human review
```

```
// @cairn-legacy {DECISION_ID} {short-name}
// @status legacy-accidental | legacy-deliberate | acknowledged-debt
// @constraint {What agents must do or avoid regarding this pattern}
```

### Decision Lineage Annotations (for brownfield evolution)

```
// @cairn {NEW_ID} {short-name}
// @supersedes {OLD_ID}
// @preserves {OLD_ID}.{specific-reasoning}
// @reason {Why the old decision was changed and what was kept}
```

---

## 5. Module-Level Decision Files

Each module directory can have a `DECISIONS.md` that summarizes the decisions governing that module.

### Format: `src/{module}/DECISIONS.md`

```markdown
# Decisions: {Module Name}

**Coverage:** full | partial | undiscovered
**Last reviewed:** {date}

## Active Decisions

| ID | Decision | Key Constraint |
|----|----------|----------------|
| {ID} | {title} | {most important anti-pattern or constraint} |

## Decision Details

### {DECISION_ID} — {Title}

**Summary:** {1-2 sentence summary}
**Anti-patterns:** {key things NOT to do in this module}
**Full record:** .cairn/decisions/{DECISION_ID}.md

### {DECISION_ID} — {Title}

{...}

## Undocumented Patterns

{Patterns observed during discovery that haven't been confirmed as deliberate.
Listed here so future agents know to be cautious.}

- {pattern description} — status: unconfirmed
```

---

## 6. Skills — Detailed Design

### 6.1 Planning Skill (`cairn-planning`)

**Trigger:** Auto-discovers when the user starts discussing a new feature, component, or project. Also triggered by `/cairn-plan`.

**Behavior:**

1. Loads the decision reference catalog from `.cairn/catalog/`.
2. Determines the preset from `.cairn/config.yaml` (developer vs. guided).
3. For each relevant catalog category, systematically presents decision points:
   - Names the decision category
   - Lists 3-7 options with tradeoffs
   - Asks the user to weigh in
   - Records the decision using the template
4. Proactively identifies decisions the user hasn't mentioned:
   - "You described the data model but haven't mentioned error handling. Here's what I'd want to decide before implementation..."
   - "This feature will need to integrate with {existing module}. Before we build, we should decide how that boundary works."
5. For each decision, generates both the positive choice AND the anti-patterns.
6. At the end, produces:
   - Numbered decision records in `.cairn/decisions/`
   - A task list annotated with decision constraints (for handoff to any harness)
   - A summary of decisions made, for human review before implementation begins

**Developer preset vs. Guided preset:**

Developer preset speaks in terms like "proxy vs. extender," "command-query separation," "event sourcing." It assumes familiarity and moves fast.

Guided preset explains each concept: "There are two ways to connect your menu system to existing screens. One approach (extending) means your menu IS a type of screen — simple, but it means other mods that also extend the same screen will conflict with yours. The other approach (proxying) means your menu WRAPS a screen — more code, but other mods can wrap the same screen independently without conflicts. Given that you want menukit to be compatible with other mods, which approach fits better?"

Both presets produce identical output formats. The difference is purely in the conversation.

### 6.2 Implementation Skill (`cairn-implementation`)

**Trigger:** Auto-discovers when the agent is writing or modifying code in a project that has a `.cairn/` directory.

**Behavior:**

1. Before modifying any file, checks:
   - `.cairn/decisions/` for decisions scoped to the target file/module
   - The module's `DECISIONS.md` if one exists
   - Existing `@cairn` annotations in the file
2. While implementing, follows recorded decisions and respects anti-patterns.
3. After creating or modifying a file:
   - Adds `@cairn` annotations for any decisions that govern the code
   - Creates or updates the module's `DECISIONS.md`
4. When encountering an implementation choice not covered by existing decisions:
   - Records it as `@cairn-implementation-inferred` with reasoning
   - Flags it for human review in the module's `DECISIONS.md`
5. Never changes code that has a `@cairn-unknown` annotation without human approval.

**Integration with harnesses:**

For Ralph: The implementation skill gets referenced in PROMPT.md or AGENTS.md. The agent reads it at the start of each iteration loop.

For Task Master: Decision constraints are included in task metadata. The agent reads them before starting each task.

For bare Claude Code: The skill auto-loads when Claude detects it's writing code in a Cairn-enabled project.

### 6.3 Discovery Skill (`cairn-discovery`)

**Trigger:** Invoked by `/cairn-discover` or auto-discovers when the planning skill detects brownfield context (existing code in the module being planned).

**Behavior:**

1. Scans the target module(s) for patterns that look architecturally significant:
   - Consistent structural patterns (all classes use X, except one that uses Y)
   - Data access patterns (repository vs. direct, ORM vs. raw SQL)
   - Error handling patterns (exceptions vs. result types vs. null returns)
   - API shape patterns (REST vs. RPC style, versioned vs. unversioned)
   - Naming conventions (and inconsistencies)
   - Dependency patterns (what depends on what, and how)
2. Presents candidate decisions as questions, not assertions:
   - "I see X. Was this deliberate? If so, what was the reasoning?"
3. Based on human response, records each as:
   - Full decision record (confirmed deliberate)
   - `@cairn-legacy` with status `legacy-deliberate` (deliberate but needs revision)
   - `@cairn-legacy` with status `legacy-accidental` (not deliberate, refactoring candidate)
   - `@cairn-legacy` with status `acknowledged-debt` (known technical debt)
   - `@cairn-unknown` (human doesn't know, preserve until investigated)
4. Updates `.cairn/coverage.yaml` with the module's new coverage status.

### 6.4 Review Skill (`cairn-review`)

**Trigger:** Invoked by `/cairn-check` or used as a QA pass in a generator/evaluator harness.

**Behavior:**

1. Scans all files modified since the last review (or since a given commit):
   - Do modified files have `@cairn` annotations?
   - Do the changes respect the anti-patterns in relevant decision records?
   - Are there new patterns that should have decision records but don't?
2. Cross-references module `DECISIONS.md` files with actual code:
   - Do the listed decisions still match the implementation?
   - Are there decisions listed that no longer apply (dead decisions)?
3. Checks for `@cairn-implementation-inferred` annotations that need human review.
4. Produces a report:
   - **Compliant:** Changes that follow recorded decisions
   - **Violations:** Changes that appear to violate anti-patterns
   - **Gaps:** Changes that lack decision coverage
   - **Stale:** Decision records that reference code that no longer exists
   - **Pending review:** Implementation-inferred decisions awaiting human confirmation

---

## 7. Decision Reference Catalog

The catalog is the knowledge base Claude draws from during planning. It's organized by concern area, and each entry lists common decision points with their options and the quality attributes they optimize for.

Stored in `.cairn/catalog/`. Initially curated by Trevor, later open to contributions.

### Catalog Structure

Each catalog file covers a concern area. Within each file, decision points are organized as:

```markdown
## {Decision Category}

### {Decision Point Name}

**When this matters:** {Situations where this decision has architectural impact}

**Common options:**

| Option | Optimizes for | Trades off |
|--------|--------------|------------|
| {option A} | {quality attribute} | {what you give up} |
| {option B} | {quality attribute} | {what you give up} |
| {option C} | {quality attribute} | {what you give up} |

**Questions to ask:**
- {Question that helps determine the right choice for this project}
- {Question that surfaces constraints the user might not have considered}

**Related decisions:** {Other decision points this interacts with}
```

### Planned Catalog Files

**`architecture.md`** — System-level decisions
- Service boundaries (monolith, modular monolith, microservices, serverless)
- Communication patterns (sync/async, events/commands, request/response)
- Data ownership (shared database, database-per-service, event sourcing)
- Deployment topology (single region, multi-region, edge)
- Caching strategy (none, read-through, write-behind, CDN)

**`design.md`** — Code-level structural decisions
- Composition model (inheritance, composition, delegation, mixins, proxies)
- Coupling strategy (direct dependency, interface/protocol, events, DI)
- State management (stateful objects, immutable data, state machines)
- Abstraction level (thin wrappers, deep modules, leaky abstractions)
- Design patterns (and when each is appropriate vs. over-engineering)

**`data.md`** — Data structure and modeling decisions
- Collection types (array, set, map, tree, graph — and when each matters)
- Identity strategy (UUID, auto-increment, natural keys, composite keys)
- Schema approach (rigid schema, flexible/document, schema-on-read)
- Normalization level (fully normalized, denormalized for reads, CQRS)
- Temporal modeling (current-state-only, bitemporal, event-sourced)

**`integration.md`** — API and boundary decisions
- API style (REST, GraphQL, RPC, message queue)
- API versioning (URL, header, content negotiation, none)
- Serialization format (JSON, protobuf, MessagePack, XML)
- Authentication model (API keys, OAuth, JWT, mTLS)
- Error contract (HTTP status codes, error objects, result types)

**`error-handling.md`** — Error philosophy decisions
- Error representation (exceptions, result types, error codes, null/optional)
- Error granularity (broad categories, specific codes, structured errors)
- Recovery strategy (fail fast, retry, circuit breaker, fallback)
- Error boundaries (where errors are caught and translated)
- Logging philosophy (structured, unstructured, correlation IDs)

**`naming.md`** — Convention decisions
- Naming style (camelCase, snake_case, PascalCase — per context)
- Domain vocabulary (ubiquitous language, abbreviation policy)
- File organization (by feature, by layer, by type)
- Module naming (noun-based, verb-based, domain-aligned)

---

## 8. Coverage Tracking

`.cairn/coverage.yaml` tracks which modules have been through the discovery process and their AKM coverage level.

```yaml
modules:
  src/menukit:
    coverage: full
    last_reviewed: 2026-03-28
    decision_count: 7
    discovery_date: 2026-03-28
  src/observability/ingestion:
    coverage: partial
    last_reviewed: 2026-03-25
    decision_count: 3
    discovery_date: 2026-03-25
    notes: "Error handling decisions still pending review"
  src/auth:
    coverage: undiscovered
```

---

## 9. npm Package Structure

```
cairn/
├── package.json
├── bin/
│   └── cairn.js              # CLI entry point
├── src/
│   ├── init.js               # Initialization logic
│   ├── scaffold.js           # Creates .cairn/ directory structure
│   └── install-skills.js     # Copies skills into .claude/skills/
├── templates/
│   ├── cairn/                # .cairn/ directory template
│   │   ├── decisions/
│   │   │   └── _template.md
│   │   ├── catalog/
│   │   │   ├── architecture.md
│   │   │   ├── design.md
│   │   │   ├── data.md
│   │   │   ├── integration.md
│   │   │   ├── error-handling.md
│   │   │   └── naming.md
│   │   ├── config.yaml
│   │   └── coverage.yaml
│   ├── skills/               # Claude Code skill templates
│   │   ├── cairn-planning/
│   │   │   └── SKILL.md
│   │   ├── cairn-implementation/
│   │   │   └── SKILL.md
│   │   ├── cairn-discovery/
│   │   │   └── SKILL.md
│   │   └── cairn-review/
│   │       └── SKILL.md
│   ├── commands/             # Claude Code slash command templates
│   │   ├── cairn-plan.md
│   │   ├── cairn-discover.md
│   │   ├── cairn-check.md
│   │   └── cairn-status.md
│   ├── CAIRN.md              # Root pointer file template
│   └── claude-md-append.md   # Text to append to CLAUDE.md
└── README.md
```

### `package.json`

```json
{
  "name": "cairn-akm",
  "version": "0.1.0",
  "description": "Architectural Knowledge Management for AI-assisted development",
  "bin": {
    "cairn": "./bin/cairn.js"
  },
  "keywords": [
    "ai",
    "architecture",
    "decisions",
    "adr",
    "claude-code",
    "agent",
    "akm"
  ],
  "license": "MIT"
}
```

### CLI Commands (v1)

```bash
npx cairn init                    # Initialize Cairn in current project
npx cairn init --preset guided    # Initialize with guided preset
npx cairn init --prefix PEBBLE   # Initialize with project-specific decision ID prefix
```

Future CLI commands (post-v1):
```bash
npx cairn status                  # Show coverage summary
npx cairn upgrade                 # Update skills and catalog to latest version
```

The CLI is intentionally minimal. Most interaction happens through Claude Code skills and slash commands, not through the CLI.

---

## 10. Build Plan — Phased Approach

Even though the scope is "full system," the build should be iterative. Each phase is usable on its own.

### Phase 1: Foundation (Week 1)

**Goal:** Decision format, directory conventions, and the init command work.

- [ ] Create npm package scaffold with bin entry point
- [ ] Implement `cairn init` — creates `.cairn/`, installs skills, creates CAIRN.md
- [ ] Write the decision record template (`_template.md`)
- [ ] Write `config.yaml` template with defaults
- [ ] Write `CAIRN.md` root pointer file
- [ ] Write the CLAUDE.md append text
- [ ] Define the inline annotation format (documented in a reference file)

**Deliverable:** You can run `npx cairn init` and get a working directory structure.

### Phase 2: Planning Skill + Catalog (Week 2)

**Goal:** The planning skill works in a Claude Code session.

- [ ] Write `cairn-planning/SKILL.md` — the full planning skill with both presets
- [ ] Write `/cairn-plan` slash command
- [ ] Curate `catalog/architecture.md` (first catalog file)
- [ ] Curate `catalog/design.md` (second catalog file)
- [ ] Curate `catalog/data.md`
- [ ] Curate `catalog/integration.md`
- [ ] Curate `catalog/error-handling.md`
- [ ] Curate `catalog/naming.md`
- [ ] Test: run a full planning session on a real project (use Pebble or trevor-mc-mod)

**Deliverable:** You can start a Claude Code session, run `/cairn-plan`, and get a structured decision elicitation conversation that produces numbered decision records.

### Phase 3: Implementation Skill (Week 3)

**Goal:** Implementation agents are decision-aware.

- [ ] Write `cairn-implementation/SKILL.md`
- [ ] Define the `DECISIONS.md` module-level format
- [ ] Define the `@cairn-implementation-inferred` convention
- [ ] Test: run a Ralph loop or extended Claude Code session with the implementation skill active
- [ ] Verify: do generated files have correct annotations? Do module DECISIONS.md files get created?

**Deliverable:** An implementation agent writes code with inline `@cairn` annotations and maintains module-level `DECISIONS.md` files.

### Phase 4: Discovery Skill (Week 4)

**Goal:** Brownfield discovery works.

- [ ] Write `cairn-discovery/SKILL.md`
- [ ] Write `/cairn-discover` slash command
- [ ] Define `@cairn-unknown` and `@cairn-legacy` annotation formats
- [ ] Implement `coverage.yaml` tracking
- [ ] Test: run discovery on an existing module in trevor-mc-mod or another brownfield project
- [ ] Verify: does Claude surface meaningful candidate decisions? Does it categorize them correctly?

**Deliverable:** You can run `/cairn-discover src/menukit` and get a structured archaeology conversation that produces categorized decision records and coverage tracking.

### Phase 5: Review Skill (Week 5)

**Goal:** Decision compliance auditing works.

- [ ] Write `cairn-review/SKILL.md`
- [ ] Write `/cairn-check` slash command
- [ ] Write `/cairn-status` slash command
- [ ] Define the review report format
- [ ] Test: make a code change that violates a recorded anti-pattern, run `/cairn-check`, verify it catches it
- [ ] Test: make a code change without annotations, verify the gap is flagged

**Deliverable:** You can run `/cairn-check` after an implementation session and get a compliance report showing violations, gaps, and pending reviews.

### Phase 6: Polish & Dogfood (Week 6)

**Goal:** Use Cairn on a real project end-to-end and refine.

- [ ] Run the full workflow on a real feature (planning → implementation → review)
- [ ] Refine skill prompts based on what works and what doesn't
- [ ] Refine catalog entries based on which decisions actually come up
- [ ] Update README with real examples
- [ ] Publish v0.1.0 to npm

---

## 11. Open Questions

Things to figure out during implementation:

1. **Annotation density:** How many `@cairn` annotations per file before it becomes noise? Need a heuristic — maybe annotate at the module/class level but not individual functions unless the decision is function-specific.

2. **Decision granularity:** Where's the line between "architecturally significant" and "just a coding choice"? The catalog helps, but we'll need to tune this through dogfooding.

3. **Catalog evolution:** How do contributed catalog entries get reviewed? Need a contribution format and review process before opening it up.

4. **Multi-file decisions:** Some decisions span many files (e.g., "we use the repository pattern everywhere"). The scope field in the decision record handles this, but the inline annotations could get repetitive. Maybe a convention like "annotate the interface/protocol file, not every implementation"?

5. **Decision conflicts:** What happens when two decisions contradict each other? The review skill should catch this, but need a resolution protocol.

6. **Catalog versioning:** When the catalog gets updated (new decision categories, refined options), how does that affect projects already using Cairn? Probably `npx cairn upgrade` that merges new catalog entries without overwriting customizations.
