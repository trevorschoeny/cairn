---
name: cairn-check
description: >
  Run a Cairn health check across the project or a specific module. Reports
  decision compliance, annotation coverage, staleness, and gaps. Use when the
  developer says "cairn check", "cairn-check", "audit the architecture",
  "are we following our decisions", or wants a full compliance report.
user-invocable: true
---

Run a Cairn architectural health check on this project.

## Scope

If an argument was provided, check only that path: $ARGUMENTS
If no argument was provided, check the entire project.

## Instructions

You MUST use the `cairn-reviewer` sub-agent for this task. Do NOT do the
review yourself in the main context — the reviewer reads all decision records
in its own context window and produces a structured report.

### Step 1: Run the review

If checking the full project:
```
Use cairn-reviewer to check the entire project
```

If checking a specific path:
```
Use cairn-reviewer to check $ARGUMENTS
```

### Step 2: Add project-wide health metrics

After the reviewer returns its report, supplement it with these
project-level metrics that require reading .cairn/ directly:

1. **Decision inventory**: Count files in `.cairn/decisions/` (excluding _template.md).
   List any with `status: deprecated` or `status: superseded`.

2. **Catalog coverage**: Compare the decision records against the 7 catalog areas.
   Which areas have decisions recorded? Which have zero?

3. **Staleness check**: For each decision, compare its `date` field against
   the last-modified date of files in its scope. If the files have been heavily
   modified since the decision was recorded, flag it as potentially stale.

4. **Inferred decisions pending review**: Grep the codebase for
   `@cairn-implementation-inferred`. These are unplanned decisions that an
   agent flagged during implementation but that haven't been promoted to
   full decision records yet.

5. **Unknown patterns**: Grep for `@cairn-unknown`. These are brownfield
   patterns with unknown reasoning that should eventually be resolved
   through discovery.

### Step 3: Present the health report

Combine the reviewer's report with the project-level metrics into a
single summary. Format:

```
CAIRN HEALTH CHECK — [project name]
Date: [today]

INVENTORY
  Decision records: [count]
  Active: [count] | Deprecated: [count] | Superseded: [count]

CATALOG COVERAGE
  Structure & Composition: [N] decisions
  Data & State: [N] decisions
  Communication & Interfaces: no decisions recorded
  ... (all 7 areas)

VIOLATIONS: [count]
  [summary from reviewer]

GAPS: [count]
  [summary from reviewer]

POTENTIALLY STALE: [count]
  - [DECISION-ID] last updated [date], but scoped files modified [date]

PENDING REVIEW: [count]
  - [count] @cairn-implementation-inferred annotations
  - [count] @cairn-unknown annotations

COMPLIANT: [count] files checked, [count] fully compliant
```

Keep it concise. The developer wants a dashboard, not a novel.
