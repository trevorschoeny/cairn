---
name: plan
description: Guide a collaborative planning session for a new project or feature using Cairn. Use when the developer says "cairn plan", "let's plan this project", "help me design this", or "let's make architectural decisions".
user-invocable: true
---

# Cairn Planning Skill

Run a collaborative planning session that produces architectural decision records
a developer (or agent) can reference during implementation.

**Three phases, always in order:**
1. **Calibrate** — Understand the developer, the project, and the depth of collaboration wanted.
2. **Spec** — Collaborate on what the software does before deciding how it's shaped.
3. **Design** — Walk through relevant decision catalogs and record choices.

## Before You Begin

Read the project's `.cairn/config.yaml` to understand:
- `preset` — "developer" (move fast, technical language) or "guided" (explain concepts first)
- `decision_id.prefix` — used when generating decision record filenames
- `catalog.exclude` — skip these catalog files during Phase 3

If no `.cairn/` directory exists, this is a fresh project. Run `cairn init` first
(or create the directory structure manually from the Cairn template).

---

## Phase 1: Calibrate

**Goal:** Set the depth knob for the entire session before doing any work.

Ask three things:

### 1. Involvement Level

"How involved do you want to be in the details?"

| Level | What it means | How the session changes |
|-------|--------------|------------------------|
| **Minimal** | "Here's a sentence. Build it." | Spec phase is a single exchange. Design phase: agent proposes decisions, developer approves/vetoes. |
| **Collaborative** | "Let's think through this together." | Spec phase is iterative. Design phase: agent presents options with tradeoffs, developer chooses. |
| **Deep** | "I want to debate every option." | Spec phase is thorough. Design phase: agent presents full catalog entries, discusses pros/cons in detail. |

Default to **Collaborative** if the developer doesn't have a strong preference.
The developer can change this mid-session ("speed up" / "let's go deeper on this one").

### 2. Project Type

"What kind of software is this?"

Identify from conversation or ask directly. This filters which catalogs are relevant:

| Type | Relevant catalogs | Likely skip |
|------|------------------|-------------|
| Web app (fullstack) | All 7 | — |
| API / backend service | structure, data, communication, errors, resources, security | identity (partially) |
| CLI tool | structure, errors, identity | data (partially), communication (partially), security (partially) |
| Library / package | structure, communication, identity | resources (partially), security (partially) |
| Mobile app | All 7 | — |
| Data pipeline | structure, data, errors, resources | communication (partially), security (partially) |

These are starting filters, not hard rules. Adjust as the spec reveals more.

### 3. Greenfield or Brownfield?

- **Greenfield** — New project. Full planning workflow.
- **Brownfield** — Existing code. Switch to discovery mode (separate skill: cairn-discovery). Exit this skill and point the developer there.
- **Extension** — Existing project with Cairn, adding new features. Skip to Phase 2 (spec) for the new feature, then Phase 3 for any new design decisions.

---

## Phase 2: Spec Collaboration

**Goal:** Agree on what the software does — features, user flows, data, and
boundaries — before making design decisions about how it's shaped internally.

**Why this comes first:** Design decisions without a spec are premature. You can't
choose between REST and GraphQL if you don't know what the API serves. You can't
pick a state management approach if you don't know what state exists.

### How This Phase Works (By Involvement Level)

**Minimal:** Ask for a one-sentence description. Expand it into 3-5 bullet points
of core features. Confirm with the developer. Move on.

**Collaborative:** Walk through these questions iteratively:
1. **What does this software do?** (Elevator pitch — 1-2 sentences)
2. **Who uses it?** (User types, personas, or consuming services)
3. **What are the core features?** (The 3-7 things it must do for v1)
4. **What data does it manage?** (Key entities, relationships, lifecycle)
5. **What are the boundaries?** (What it explicitly does NOT do)
6. **What does it integrate with?** (External APIs, databases, services)
7. **What are the constraints?** (Timeline, team size, hosting, budget, compliance)

Don't ask all seven as a wall of questions. Weave them into conversation.
Let the developer info-dump and then fill gaps with targeted follow-ups.

**Deep:** Same questions as Collaborative, plus:
- Specific user flows ("Walk me through what happens when a user does X")
- API endpoint sketches or screen wireframe descriptions
- Data model drafts (entities, fields, relationships)
- Edge cases and error scenarios

### Spec Output

The spec doesn't need to be a formal document. It's shared context between the
developer and the agent. Options:

- **Minimal/Collaborative:** Keep it in conversation memory. Summarize the key
  points back to the developer before moving to Phase 3.
- **Deep:** Write a lightweight spec file (markdown) to `.cairn/spec.md` or
  wherever the developer prefers. This becomes a reference for implementation.

### Exit Condition

Move to Phase 3 when you can answer: "If I had to start coding this right now,
do I know enough about what it does to make good design decisions?" If yes, proceed.
If no, keep asking.

---

## Phase 3: Design Decisions

**Goal:** Walk through relevant catalog decision points, present options with
tradeoffs, and record the developer's choices as decision records.

### How This Phase Works

**Step 1: Load relevant catalogs.**

Based on the project type from Phase 1, read the relevant catalog files from
`.cairn/catalog/`. Each catalog contains decision points with options, tradeoffs,
and guiding questions.

```
Catalog files live at: .cairn/catalog/{name}.md
  - structure-composition.md  (10 decisions)
  - data-state.md             (10 decisions)
  - communication-interfaces.md (8 decisions)
  - errors-resilience.md      (6 decisions)
  - identity-conventions.md   (5 decisions)
  - resources-performance.md  (3 decisions)
  - security.md               (4 decisions)
```

Read each relevant catalog file before presenting its decisions. Do NOT rely on
memory of catalog contents — read fresh each time.

**Step 2: Filter decision points.**

Not all 46 decision points apply to every project. For each catalog:
- Read the "When this matters" section of each decision point.
- Compare it against the spec from Phase 2.
- Skip decision points that are irrelevant to this project.

Example: A simple CLI tool doesn't need "Event Architecture" or "API Versioning."
A serverless function doesn't need "Resource Pooling & Lifecycle."

**Step 3: Present decisions by catalog, one at a time.**

For each relevant decision point:

1. **State the decision point** — name it and briefly explain why it matters for this project.
2. **Present options** — use the catalog's options table, tailored to the project context.
   - At **Minimal** level: recommend an option with brief justification. Ask for approval.
   - At **Collaborative** level: present 2-4 most relevant options with tradeoffs. Ask which fits.
   - At **Deep** level: present the full options table and guiding questions. Discuss.
3. **Record the choice** — note the decision and reasoning for the record.
4. **Ask about anti-patterns** — "Is there anything you specifically want to avoid or that
   has burned you before?" These go directly into the Anti-Patterns section of the record.
5. **Move on** — don't linger. If the developer says "I don't care, pick one," pick one
   and record your reasoning. That's a valid decision.

**Step 4: Batch related decisions.**

Some decisions are tightly coupled. Present them together rather than forcing
artificial separation:
- API Style + Serialization Format + Communication Pattern (all shape the API)
- Schema Strategy + Persistence Strategy + Normalization Level (all shape the data layer)
- Authentication Strategy + Authorization Model + Trust Boundary Design (all shape security)

**Step 5: Write decision records.**

After completing a catalog (or a batch of related decisions), write decision
records to `.cairn/decisions/` using the template at `.cairn/decisions/_template.md`.

Naming convention: `{prefix}{NNN}-{short-title}.md`
- Use the `decision_id.prefix` from `.cairn/config.yaml`
- Auto-increment the number from the highest existing record
- Use kebab-case short titles: `PEBBLE-001-use-rest-for-public-api.md`

**What goes in each record:**
- **Y-Statement Summary** — one sentence capturing the full decision
- **Context and Problem Statement** — pulled from the spec and the catalog's "when this matters"
- **Considered Options** — from the catalog, filtered to what was actually discussed
- **Decision Outcome** — what was chosen and the primary reason
- **Anti-Patterns** — what NOT to do (from the developer's input + the catalog's guiding questions)
- **Scope** — which files/modules this decision governs (may be "project-wide" for early decisions)
- **cairn-origin** — always `planned` for decisions from this skill

Not every field in the template is required. Use the mandatory sections; include
optional sections only when they add value. A thin decision record that exists
is better than a thorough one that never gets written.

**Step 6: Summarize and confirm.**

After all catalogs are complete, present a summary:
- How many decisions were recorded
- Key themes or architectural direction
- Any decisions that were explicitly deferred (and why)
- Recommendations for what to decide later once implementation reveals more

Ask the developer if anything is missing or if they want to revisit any decision.

---

## Agent Anti-Patterns

These are the mistakes agents make most often during planning sessions.
Guard against all of them.

**Do NOT dump all 46 decisions on the developer at once.**
Present one catalog at a time, filtered to relevant decisions. If the developer
looks overwhelmed, slow down and batch smaller.

**Do NOT skip Phase 2 (Spec) and jump to design decisions.**
Design decisions without a spec are guesswork. Even a one-sentence spec at
Minimal involvement is better than nothing.

**Do NOT present decisions the developer doesn't need to make.**
If the project is a CLI tool, don't ask about API versioning. Filter aggressively.
Err on the side of skipping — the developer can always say "what about X?"

**Do NOT force a choice when the developer says "I don't care."**
Make the choice yourself, document your reasoning, and move on. "Agent chose X
because {reason}" is a valid decision record. The developer can revisit later.

**Do NOT write empty anti-patterns sections.**
Every decision has at least one thing an agent should NOT do. If you can't think
of one, ask: "What's the obvious wrong approach someone might take here?"

**Do NOT treat this as a one-shot process.**
Some decisions genuinely can't be made until implementation reveals more.
Record them as deferred with a reason. The developer can run `cairn plan`
again for a specific decision later.

**Do NOT read catalogs from memory.**
Always read the catalog file fresh from `.cairn/catalog/` before presenting its
decisions. Catalog contents may have been updated since your training data.

---

## Session Management

Planning sessions can be long. Handle interruptions gracefully:

**If the session spans multiple conversations:**
- At the end of each conversation, summarize what's been decided so far and
  what remains. Write completed decision records to disk immediately — don't
  hold them in memory across sessions.
- At the start of a resumed session, read existing decision records from
  `.cairn/decisions/` to rebuild context.

**If the developer wants to pause and come back:**
- Write all pending decisions to disk.
- Note which catalogs have been covered and which remain.
- When resuming, read the existing records and pick up where you left off.

**If the developer wants to revisit a decision:**
- Read the existing record.
- Walk through the options again with the new context.
- Update the record in place (don't create a new one unless the decision
  is truly superseded).

---

## Quick Reference: The Full Flow

```
1. CALIBRATE
   ├── Involvement level (minimal / collaborative / deep)
   ├── Project type (filters catalogs)
   └── Greenfield / brownfield / extension

2. SPEC
   ├── What does it do?
   ├── Who uses it?
   ├── Core features, data, boundaries, integrations, constraints
   └── Exit: "Could I make good design decisions now?" → Yes → proceed

3. DESIGN
   ├── Load relevant catalogs (read fresh from .cairn/catalog/)
   ├── Filter to applicable decision points
   ├── Present options at calibrated depth
   ├── Record choices → .cairn/decisions/{prefix}{NNN}-{title}.md
   └── Summarize and confirm
```
