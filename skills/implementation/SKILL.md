---
name: implementation
description: Behavioral overlay for coding sessions in Cairn-enabled projects. Ensures architectural decisions are respected by running preflight checks, adding annotations, and flagging violations. Active whenever a .cairn/ directory exists.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Bash(*)
---

# Cairn Implementation Skill

This skill makes you Cairn-aware during coding. It is not a conversation —
it is a behavioral overlay. Follow these rules automatically whenever you are
implementing code in a Cairn-enabled project.

## How to Detect a Cairn-Enabled Project

Any of these signals mean Cairn is active:
- A `.cairn/` directory exists in the project root
- The CLAUDE.md references Cairn
- Source files contain `@cairn` annotations
- `.claude/agents/cairn-preflight.md` exists

If you see any of these, this skill applies to your entire coding session.

---

## The Three Checkpoints

Cairn adds three checkpoints to your normal coding workflow. They wrap
around your existing process — they don't replace it.

```
Normal coding workflow:
  [understand task] → [plan approach] → [write code] → [test] → [commit]

With Cairn:
  [understand task] → [PREFLIGHT] → [plan approach] → [write code] → [test] → [ANNOTATE] → [commit]
                                          ↑                              ↓
                                    [ASK-BEFORE]                   [REVIEW] (optional)
                                 (if decision gap found)
```

### Checkpoint 1: PREFLIGHT (Before Coding)

**When:** After you understand the task but BEFORE you plan your approach or
write any code. Every time. No exceptions.

**What it does:** Reads decision records and returns a briefing of what
applies to the files you're about to touch.

**How to invoke it:**

IF the `cairn-preflight` sub-agent is available (`.claude/agents/cairn-preflight.md` exists):

> You MUST use the cairn-preflight agent. Do NOT spin up your own ad-hoc
> sub-agent to read decision files. Do NOT read decision files yourself
> in the main context. The preflight agent exists specifically to keep
> your context clean. Use it.

Invoke it with a description of what you're about to modify:
```
Use cairn-preflight to check decisions for src/api/orders/ and src/models/order.ts
```

IF the sub-agent is NOT available (non-Claude-Code environment, Ralph harness, etc.):

Read the decision files directly. This is the fallback path:
1. Read `.cairn/config.yaml` for the project setup.
2. List files in `.cairn/decisions/`.
3. For each decision, read the **Scope** and **Anti-Patterns** sections.
4. Check for a `DECISIONS.md` in the target module directory.
5. Grep target files for existing `@cairn` annotations.
6. Mentally compile the same briefing the sub-agent would produce.

**What to do with the briefing:**

The briefing tells you:
- Which decisions govern the files you're touching
- What anti-patterns to avoid (the most critical information)
- What `@cairn-unknown` patterns exist (DO NOT MODIFY these)

Incorporate this into your approach. If the briefing says "Do NOT use direct
Screen extension" and your plan was to extend Screen, change your plan.
The decision was made deliberately. If you genuinely believe the decision
is wrong, flag it to the developer — do not silently violate it.

---

### Checkpoint 2: ANNOTATE (After Implementation)

**When:** After you've written and tested the code, but BEFORE committing.
Run this once per coding task, not after every file save.

**What it does:** Scans your new/modified files, adds `@cairn` annotations
linking code to governing decisions, and flags any unplanned design choices.

**How to invoke it:**

IF the `cairn-annotator` sub-agent is available:

> You MUST use the cairn-annotator agent. Do NOT add @cairn annotations
> yourself in the main context. The annotator reads all decision records
> in its own context window and applies annotations consistently. You
> will get annotations wrong if you do it from memory.

```
Use cairn-annotator on src/api/orders/ and src/models/order.ts
```

IF the sub-agent is NOT available:

Add annotations yourself following these rules:
1. Read `.cairn/annotations.md` for the annotation format reference.
2. For each file you created or significantly modified:
   - Identify which decision records govern it (from the preflight briefing).
   - Add a `@cairn DECISION-ID — brief reason` comment at the class/module level.
   - Include the most important anti-pattern inline.
3. If you made an architectural choice not covered by any existing decision,
   add `@cairn-implementation-inferred — [describe choice and reasoning]`.
4. Do NOT add annotations to code you didn't change.

---

### Checkpoint 3: REVIEW (On Demand)

**When:** This one is NOT automatic. Invoke it when:
- The developer asks for a Cairn review (`cairn check`, `cairn review`)
- Before a significant commit or PR
- When you're unsure if your changes comply with recorded decisions
- Periodically during long implementation sessions

**What it does:** Compares changes against decision records. Reports
violations, drift, and coverage gaps.

**How to invoke it:**

IF the `cairn-reviewer` sub-agent is available:

> Use the cairn-reviewer agent. It has git access and will diff your
> changes against decisions systematically.

```
Use cairn-reviewer to check my changes in src/api/
```

IF the sub-agent is NOT available:

Self-review by reading the relevant decision records and comparing your
changes against each decision's chosen option and anti-patterns. Report
any concerns to the developer.

---

## Behavioral Rules (Always Active)

These rules apply throughout your entire coding session, not just at
the checkpoints.

### Rule 1: Never Modify @cairn-unknown Code

If you encounter code with a `@cairn-unknown` annotation, it means the
pattern's reasoning is unknown. Do NOT refactor, optimize, or "improve" it.
Even if it looks wrong. Even if you're sure you know a better way. The
annotation exists because someone explicitly decided this code should not
be touched until a human reviews it.

If the task requires modifying that code, tell the developer:
"This code is marked @cairn-unknown, which means its design reasoning
hasn't been documented yet. I shouldn't modify it without your review."

### Rule 2: Anti-Patterns Are Hard Constraints

When a decision record's Anti-Patterns section says "Do NOT X," treat
it as a hard constraint, not a suggestion. Anti-patterns exist because
someone thought carefully about what could go wrong.

If your implementation would violate an anti-pattern, you have two options:
1. Change your approach to avoid the anti-pattern.
2. Tell the developer you believe the anti-pattern is wrong, explain why,
   and ask whether to proceed or update the decision.

You do NOT have the option of silently violating an anti-pattern.

### Rule 3: Ask Before Making Unplanned Decisions

During implementation, you will encounter choices not covered by existing
decision records. Most are trivial (variable names, loop style) — just
pick the reasonable option and move on. But some are architecturally
significant. For those: **stop and ask before implementing.**

**Significance threshold:** A choice is significant if:
- It affects how other files should be written
- Changing it later would require touching multiple files
- It introduces a new pattern, dependency, or abstraction
- There are two or more reasonable approaches with real tradeoffs

**When you hit a significant choice:**

1. **Stop.** Do not pick an approach and keep going.
2. **Present everything in one prompt.** Tell the developer:
   - What choice you've encountered
   - What the options are (at least 2)
   - What tradeoffs you see
   - Which option you'd lean toward and why
   - "Should we record this as an official decision?"
3. **Wait.** Do not proceed until they respond.
4. **After they respond:** implement their choice. If they want an official
   decision, hand off to the planning skill after implementation is done.
   If not, annotate with `@cairn-implementation-inferred`.

The developer should only need to respond once — "option B, and yes make
it official" or "go with A, no need for a record" — and you have everything
you need to proceed.

**For trivial choices** (no real tradeoffs, one obviously correct approach):
just implement, annotate with `@cairn-implementation-inferred` if it's
non-obvious, and move on. Don't interrupt the developer's flow for things
that don't matter.

### Rule 4: Use the Right Sub-Agent for the Job

This is critical. Claude Code makes it easy to spin up ad-hoc sub-agents.
Do NOT do this for Cairn tasks. The Cairn sub-agents have specific
instructions, output formats, and tool restrictions that ensure consistency.

| Task | Use this | NOT this |
|------|----------|----------|
| Check decisions before coding | `cairn-preflight` agent | Ad-hoc sub-agent or reading files yourself |
| Add annotations after coding | `cairn-annotator` agent | Adding annotations from memory |
| Review changes against decisions | `cairn-reviewer` agent | Ad-hoc "review my code" sub-agent |
| Plan new decisions | `cairn-planning` skill | Making up decisions during implementation |

If the sub-agents aren't available (wrong environment), use the fallback
paths described in each checkpoint above. But if they ARE available, use them.

### Rule 5: Don't Create Full Decisions During Implementation

When Rule 3 surfaces a significant choice and the developer decides, do NOT
write a full decision record on the fly. Instead:
1. Implement the developer's choice.
2. Annotate with `@cairn-implementation-inferred`.
3. After the implementation task is complete, offer to formalize it via
   the planning skill if the developer wants a full record.

Full decision records involve presenting options, discussing tradeoffs,
and recording anti-patterns properly. Doing that mid-implementation
breaks flow and produces thin records. The `@cairn-implementation-inferred`
annotation captures the choice now; the planning skill can flesh it out later.

---

## Edge Cases

**New files with no governing decisions:** This is normal, especially early in
a project. Write the code following the project's general patterns. If you
notice a pattern forming that should be a decision (you're making the same
structural choice across multiple new files), flag it to the developer.

**Conflicting decisions:** If two decision records give contradictory guidance
for the same file, stop and flag this to the developer. Do not try to reconcile
them yourself. This usually means one decision needs to be superseded.

**Decision says one thing, developer says another in this session:** The
developer wins. They may be intentionally deviating. Follow their instruction,
but mention: "This differs from decision [DECISION-ID]. Want me to update the
decision record after we're done?"

**Massive refactor touching many governed files:** Run preflight once with the
full scope of the refactor, not per-file. The briefing may be longer than
usual — that's fine. For very large refactors, suggest running cairn-reviewer
partway through rather than only at the end.

**Context window pressure:** If your context is getting full and you haven't
run preflight yet, this is exactly when sub-agents pay off. The preflight
agent reads 20 decision files in its own context and returns a 15-line
briefing. If sub-agents aren't available and context is tight, at minimum
grep for anti-patterns in decision files scoped to your target files.

---

## Quick Reference

```
BEFORE coding:  Run cairn-preflight (or read decisions directly)
DURING coding:  Respect anti-patterns. Don't touch @cairn-unknown.
                Significant uncovered choice? STOP → present options → wait for developer.
                Trivial uncovered choice? Just do it, annotate if non-obvious.
AFTER coding:   Run cairn-annotator (or annotate manually)
ON DEMAND:      Run cairn-reviewer for compliance check

Sub-agents available?  USE THEM. Always. For every Cairn task.
Sub-agents missing?    Read .cairn/ files directly. Same information, different loading path.
```
