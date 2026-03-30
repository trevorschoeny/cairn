---
name: cairn-discovery
description: >
  Run architectural discovery on an existing codebase to surface and record the
  design decisions already embedded in the code. Use when the developer says
  "cairn discover", "cairn-discover", "document the architecture", "what decisions
  are in this codebase", "add cairn to an existing project", or any phrasing that
  implies wanting to understand and record the architectural choices in code that
  already exists. Also trigger when cairn-planning detects a brownfield project
  and redirects here. This is archaeological — it reads the code and asks the
  developer to confirm what's intentional vs accidental.
user-invocable: true
---

# Cairn Discovery Skill

Surface the architectural and design decisions already embedded in an existing
codebase. The code already answered these questions — the answers are just
implicit in the patterns rather than recorded anywhere. Your job is to make
them explicit.

**This is NOT planning.** Planning presents options for the developer to choose.
Discovery presents observations for the developer to confirm or correct.

## Before You Begin

If the project doesn't have a `.cairn/` directory yet, create one with the
standard structure (config.yaml, decisions/, catalog/). The developer is
adopting Cairn for an existing project — set up the infrastructure first.

Read `.cairn/config.yaml` if it exists for project settings and excluded catalogs.

---

## Phase 1: Scan

**Goal:** Build a structural profile of the codebase without flooding your context.

IF the `cairn-scanner` sub-agent is available:

> You MUST use the cairn-scanner agent. The scanner reads dozens of files
> to build a structural profile. Doing this in your main context would
> waste most of your context window on raw code you don't need to keep.
> The scanner returns a concise 60-80 line profile.

```
Use cairn-scanner to profile the project at [project root]
```

IF the sub-agent is NOT available:

Do the scan yourself, but be strategic:
1. Read the top-level directory listing (2 levels deep).
2. Read package/dependency files and framework configs.
3. Sample 3-5 structurally important source files (entry points, models, routes).
4. Compile a mental profile of the patterns you observe.

**Save the profile.** Write it to `.cairn/discovery-profile.md` so it survives
context resets and can be referenced later.

---

## Phase 2: Calibrate

**Goal:** Understand how thorough the developer wants to be.

Ask: "How deep do you want to go?"

| Depth | What it covers | Time estimate |
|-------|---------------|---------------|
| **Quick pass** | The 8-10 biggest structural decisions (paradigm, organization, API style, data layer, auth). Good for getting the major guardrails recorded fast. | 15-30 minutes |
| **Full archaeology** | Every relevant catalog decision point. Thorough but longer. Best for projects where you want complete coverage before agents start working. | 1-2 hours |
| **Module-by-module** | Focus on one module at a time, go deep. Good for very large codebases where full archaeology is overwhelming. | Variable — per module |

Default to **Quick pass** if the developer doesn't have a strong preference.
They can always do another pass later for deeper coverage.

---

## Phase 3: Surface Decisions

**Goal:** Walk through relevant catalog areas, present what the code chose,
and get the developer to confirm whether each choice was deliberate.

### How This Works

For each relevant catalog area (filtered by what the scanner found):

1. **Read the catalog file** from `.cairn/catalog/` for reference.

2. **Present the observation, not options.** This is the key difference from
   planning. Instead of "here are your options for error handling," say:

   > "Looking at your codebase, you're using **exceptions** for error handling
   > in the service layer, but I see **Result types** in `src/domain/order.ts`
   > and `src/domain/payment.ts`. The rest of the codebase uses try/catch.
   > Was the Result type usage in the domain layer a deliberate choice?"

3. **Wait for the developer's response.** Their answer determines how to record it:

| Developer says | What it means | How to record |
|----------------|--------------|---------------|
| "Yes, that was deliberate because..." | Confirmed intentional decision | `cairn-origin: discovered`, confidence: high, include their reasoning |
| "Yeah we do that, not sure why" | Pattern exists but reasoning is lost | `cairn-origin: discovered`, confidence: medium, open questions noted |
| "Huh, I didn't realize we did that" | Accidental pattern, no deliberate choice | `@cairn-legacy` annotation (legacy-accidental). May or may not warrant a decision record. |
| "That's tech debt we need to fix" | Known debt | `@cairn-legacy` annotation (acknowledged-debt). Note in decision record if recorded. |
| "Some files do X, some do Y — that's a mess" | Inconsistency, no governing decision | Ask: "Which approach should be the standard going forward?" Then record THAT as the decision. |

4. **Ask about anti-patterns.** For confirmed decisions, ask:
   "Is there a specific wrong approach you've seen people try, or that
   you want to make sure agents avoid?"

5. **Move to the next observation.** Don't linger. If the developer doesn't
   have strong feelings, record what you observed and move on.

### What to Surface (By Depth)

**Quick pass — surface these ~8-10 decisions:**
- Code organization strategy (feature vs layer)
- Primary programming paradigm (OOP, functional, mixed)
- API style (REST, GraphQL, gRPC)
- Persistence strategy (which database, which ORM)
- Error handling approach (exceptions, Result types)
- Authentication mechanism (JWT, sessions, OAuth)
- Test organization (co-located vs separate)
- Naming conventions (casing, singular/plural)
- Any notable inconsistencies from the scanner profile

**Full archaeology — additionally surface:**
- All remaining decision points from each relevant catalog
- Walk through the scanner's inconsistencies one by one
- Check for patterns the scanner might have missed by reading deeper

**Module-by-module — for each module:**
- All decision points visible within that module
- Cross-module patterns (does this module follow the project conventions?)
- Module-specific decisions that differ from project defaults

---

## Phase 4: Record

**Goal:** Write decision records for confirmed decisions and annotate the code.

### Writing Decision Records

For each confirmed decision, write a record to `.cairn/decisions/` using
the template at `.cairn/decisions/_template.md`.

Key differences from planning-generated records:
- **cairn-origin:** `discovered` (not `planned`)
- **Discovery Notes section:** ALWAYS fill this in. Include:
  - `Discovered by: collaborative` (human + agent)
  - `Confidence: high | medium | low` (based on developer's response)
  - `Source of reasoning:` "Developer confirmed in discovery session" or
    "Inferred from consistent pattern across N files" or
    "Partially confirmed — developer unsure of original reasoning"
  - `Open questions:` anything unresolved

### Annotating Existing Code

After recording decisions, use the cairn-annotator sub-agent (or annotate
manually) to add `@cairn` annotations to key files. For discovery, also use:

- `@cairn-unknown` — for patterns the developer can't explain the reasoning
  behind. This is the brownfield safety net. It tells implementation agents:
  "This pattern exists for unknown reasons. Do NOT modify it."

- `@cairn-legacy` with a qualifier:
  - `@cairn-legacy(deliberate)` — old approach, kept intentionally
  - `@cairn-legacy(accidental)` — emerged without deliberate choice
  - `@cairn-legacy(acknowledged-debt)` — known technical debt, will be addressed

### Coverage Tracking

If `coverage.enabled` is true in `.cairn/config.yaml`, track which modules
have been through discovery. After completing a module or the full project:

Update a `.cairn/coverage.md` file listing:
- Modules scanned with dates
- Decision count per module
- Modules not yet scanned

This lets the developer see AKM progress over time and know which parts
of the codebase still have implicit decisions.

---

## Agent Anti-Patterns

**Do NOT assume inconsistency means a mistake.** Two modules using different
error handling may be a deliberate boundary decision. Always ask before
recording it as drift or debt.

**Do NOT try to read the entire codebase.** Use the scanner sub-agent. Sample
strategically. You need a map, not a photocopy.

**Do NOT recommend changes during discovery.** Your job is to surface and
record what exists, not to propose improvements. If you see something that
looks like a problem, note it in the decision record's open questions — don't
suggest a fix.

**Do NOT skip the developer confirmation step.** You are reconstructing
reasoning from code patterns, which is inherently speculative. The developer's
confirmation (or correction) is what turns a guess into a decision record.

**Do NOT create high-confidence records for things the developer is unsure about.**
If they say "I guess we do that but I'm not sure why," that's medium confidence
at best. Be honest about uncertainty — it's more valuable than false precision.

---

## Quick Reference

```
1. SCAN
   └── Use cairn-scanner → structural profile → save to .cairn/discovery-profile.md

2. CALIBRATE
   └── Quick pass (~10 decisions) | Full archaeology (all) | Module-by-module

3. SURFACE
   ├── Present observations, not options
   ├── "Your code does X. Was that deliberate?"
   ├── Developer confirms → discovered decision (high confidence)
   ├── Developer unsure → discovered decision (medium confidence)
   ├── Developer says accidental → @cairn-legacy annotation
   └── Inconsistency found → ask which approach should be standard going forward

4. RECORD
   ├── Decision records → .cairn/decisions/ (cairn-origin: discovered)
   ├── Annotations → @cairn, @cairn-unknown, @cairn-legacy in source
   └── Coverage → .cairn/coverage.md
```
