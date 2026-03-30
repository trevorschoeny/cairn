<!-- Cairn Decision Record Template v0.1
     Based on MADR 4.0.0 (Kopp, Armbruster & Zimmermann, 2018; adr.github.io/madr)
     Extended with Cairn-specific sections for agent-aware AKM.

     Lineage:
       - Core structure (Context, Decision, Consequences) from Nygard's original ADR (2011)
       - Options + Pros/Cons structure from MADR (Markdown Architectural Decision Records)
       - Y-Statement summary format from Zimmermann's Sustainable Architectural Decisions
       - Anti-Patterns, Scope, and Lineage sections are Cairn originals for agent consumption
       - Confirmation section from MADR 4.0.0 (renamed from "Validation" in 3.0)

     Usage:
       Copy this file to .cairn/decisions/NNNN-short-title.md
       Fill in all mandatory sections. Remove optional sections you don't need.
       Sections marked [AGENT-CRITICAL] are specifically consumed by implementation agents.
-->

---
# YAML frontmatter — machine-readable metadata for tooling and agents.
# Based on MADR 4.0.0 frontmatter, extended with Cairn fields.

status: "proposed"        # proposed | accepted | deprecated | superseded
date: YYYY-MM-DD
decision-makers: ""       # who participated in this decision (MADR 4.0.0 field)

# --- Cairn extensions ---
cairn-origin: ""          # planned | discovered | inferred
                          #   planned = decided during a Cairn planning session (Phase 1)
                          #   discovered = found during brownfield discovery and confirmed by a human
                          #   inferred = recorded by an implementation agent, pending human review
---

# {DECISION-ID} — {Short Descriptive Title}

<!-- The title uses a present-tense verb phrase when possible (Nygard convention).
     Examples: "Use proxy pattern for menu composition"
               "Store sessions in Redis instead of filesystem"
               "Adopt repository pattern for data access"
     The {DECISION-ID} follows the project's cairn config prefix + auto-increment.
-->

## Y-Statement Summary

<!-- One-sentence summary following Zimmermann's Y-Statement format.
     This exists so agents and humans can get the gist without reading the full record.
     Format: "In the context of {situation}, facing {concern},
              we decided for {option} and neglected {alternatives},
              to achieve {quality}, accepting {tradeoff}."
-->

In the context of {use case or component}, facing {specific concern or requirement}, we decided for {chosen option} and neglected {rejected options}, to achieve {desired quality attribute(s)}, accepting {accepted tradeoff(s)}.

## Context and Problem Statement

<!-- MANDATORY. From Nygard's original ADR structure.
     Describe the situation that requires a decision. Include:
     - What you're building or changing
     - What constraints exist (technical, business, timeline)
     - What quality attributes matter most here
     - Why this decision can't be deferred
-->

{Describe the situation, the forces at play, and why a decision is needed now.}

## Decision Drivers

<!-- OPTIONAL but recommended. From MADR 3.0+.
     These are the specific factors that most influenced the decision.
     They help future readers understand what was prioritized.
-->

- {Driver 1 — e.g., "Cross-mod compatibility for menukit API consumers"}
- {Driver 2 — e.g., "Must work with Fabric's mixin system"}
- {Driver 3 — e.g., "Minimize API surface area for v1"}

## Considered Options

<!-- MANDATORY. From MADR's core contribution to the ADR format.
     The MADR project's key insight (Kopp et al., 2018) was that listing options
     with pros and cons is what makes an ADR actually useful for future decisions.
     List all options that were seriously considered. At minimum, list 2.
-->

1. {Option 1 — short name}
2. {Option 2 — short name}
3. {Option 3 — short name}

## Decision Outcome

<!-- MANDATORY. The core of any ADR (present in every format since Nygard 2011).
     State which option was chosen and the primary reason why.
-->

**Chosen option:** "{Option N — short name}", because {primary justification linking back to decision drivers}.

### Confirmation

<!-- OPTIONAL. Introduced in MADR 4.0.0 (renamed from "Validation" in 3.0).
     Describes how compliance with this decision can be verified.
     For Cairn, this is especially valuable — it tells the review skill
     what to check during /cairn-check audits.
-->

{How can we verify this decision is being followed? Examples:
  - "Code review: check that all menu integrations use MenuProxy, never extend Screen directly."
  - "Automated: lint rule X catches direct Screen subclassing in the menukit package."
  - "Test: integration test Y verifies cross-mod compatibility."}

## Pros and Cons of the Options

<!-- OPTIONAL but strongly recommended for decisions at the architecture or design level.
     This is MADR's signature section — the detailed tradeoff analysis per option.
     Use "Good" and "Bad" prefixes (MADR convention) for scanability.
     "Neutral" with (w.r.t. {quality}) for context-dependent properties (MADR 3.0+).
-->

### {Option 1 — short name}

{Brief description of the approach.}

- Good, because {advantage}
- Good, because {advantage}
- Neutral, because {property} (w.r.t. {quality attribute})
- Bad, because {disadvantage}

### {Option 2 — short name}

{Brief description of the approach.}

- Good, because {advantage}
- Bad, because {disadvantage}
- Bad, because {disadvantage}

### {Option 3 — short name}

{Brief description of the approach.}

- Good, because {advantage}
- Bad, because {disadvantage}

## Anti-Patterns

<!-- [AGENT-CRITICAL] Cairn original. Not present in any standard ADR template.

     This is the single most important Cairn extension. Standard ADRs record what
     TO do. This section records what NOT to do, and crucially, WHY NOT.

     Why this matters for agents: An implementation agent that reads only the chosen
     option might find a "better" way that looks correct locally but violates the
     reasoning behind the decision. The anti-pattern section makes the boundaries
     explicit and unambiguous.

     Why this matters for humans: Developers who weren't part of the original decision
     will see seemingly simpler alternatives and be tempted to refactor. This section
     explains why those alternatives were rejected — not just that they were.

     Format each anti-pattern as: Do NOT {action}. Reason: {why this breaks the intent}.
-->

- **Do NOT** {specific prohibited approach}. **Reason:** {why this violates the decision's intent, referencing decision drivers}.
- **Do NOT** {specific prohibited approach}. **Reason:** {why this violates the decision's intent}.

## Consequences

<!-- MANDATORY. From Nygard's original ADR structure, refined by MADR 3.0 which
     merged "Positive Consequences" and "Negative Consequences" into a single section.
     Record what follows from this decision — both good and bad.
-->

- Good, because {positive consequence}
- Bad, because {negative consequence or tradeoff accepted}
- {Neutral or informational consequence}

## Scope

<!-- [AGENT-CRITICAL] Cairn original.

     Standard ADRs don't specify which code they apply to — they're assumed to be
     project-wide or obvious from context. For agent consumption, explicit scoping
     is essential. An implementation agent needs to know: "does this decision affect
     the code I'm about to write?"

     List file paths, module directories, or glob patterns.
     Also list related decisions (by ID) that interact with this one.
-->

**Files and modules governed by this decision:**

- {path/to/module/ or path/to/file.ext or src/*/pattern}

**Related decisions:**

- {DECISION-ID} — {brief description of the relationship}

## Decision Lineage

<!-- OPTIONAL. Cairn original. Used for brownfield evolution.

     When a decision replaces or modifies a previous decision, this section records
     the chain. The key innovation is @preserves — explicitly stating which aspects
     of the old decision's reasoning survive into the new one.

     This prevents a future agent from seeing "superseded" and assuming the old
     decision's reasoning is entirely irrelevant. Some reasoning transfers even when
     the implementation changes.

     Remove this section entirely for greenfield decisions with no predecessors.
-->

**Supersedes:** {DECISION-ID, if this replaces a previous decision}
**Preserves from predecessor:** {Specific reasoning from the old decision that still holds — e.g., "DASH-AUTH-002.security-reasoning — the principle that role escalation must be prevented at the code level, not just the data level"}

## Discovery Notes

<!-- OPTIONAL. Cairn original. Used only for brownfield-discovered decisions
     (cairn-origin: discovered or inferred).

     Records how the decision was found and the confidence level in the
     reconstructed reasoning. This is important because brownfield decisions
     are often reconstructed from code patterns and partial memory — the
     reasoning might be incomplete or speculative.

     Remove this section for planned (greenfield) decisions.
-->

**Discovered by:** {human | agent | collaborative}
**Confidence:** {high | medium | low}
**Source of reasoning:** {e.g., "Original developer confirmed via conversation",
  "Inferred from consistent pattern across 12 files",
  "Git blame shows this pattern introduced in commit abc123 with message '...'"}
**Open questions:** {Any unresolved aspects of the reasoning}

## More Information

<!-- OPTIONAL. From MADR 3.0+.
     Links to related documents, discussions, research, external references,
     or anything that provides additional context beyond what's in this record.
-->

{Links, references, or additional context. Examples:
  - Link to the planning session transcript
  - Link to relevant documentation or specs
  - Link to the PR or commit that implemented this decision
  - Reference to design pattern literature}
