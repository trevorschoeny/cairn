---
name: status
description: Quick overview of Cairn status — decision inventory, coverage, and pending items. Faster than cairn-check. Use when the developer says "cairn status" or "how many decisions do we have".
user-invocable: true
---

Give a quick Cairn status overview for this project. This is a lightweight
inventory, NOT a full compliance check. Do NOT invoke any sub-agents.
Just read the .cairn/ directory directly.

## Instructions

1. Read `.cairn/config.yaml` for project name and settings.

2. List all decision records in `.cairn/decisions/` (excluding _template.md).
   For each, read only the YAML frontmatter and the title line to get:
   - Decision ID and title
   - Status (proposed / accepted / deprecated / superseded)
   - Origin (planned / discovered / inferred)
   - Date

3. Quick-grep the codebase for annotation counts:
   - `@cairn ` (standard annotations)
   - `@cairn-implementation-inferred` (pending review)
   - `@cairn-unknown` (brownfield unknowns)
   - `@cairn-legacy` (legacy patterns)

4. Present:

```
CAIRN STATUS — [project name]

DECISIONS ([count] total):
  [ID] [title]                              [status]  [origin]  [date]
  [ID] [title]                              [status]  [origin]  [date]
  ...

ANNOTATIONS:
  @cairn (standard):              [count] across [N] files
  @cairn-implementation-inferred: [count] (pending review)
  @cairn-unknown:                 [count] (unresolved)
  @cairn-legacy:                  [count]

QUICK ACTIONS:
  [if inferred > 0] → Run cairn plan to promote inferred decisions
  [if unknown > 0]  → Run cairn discover to resolve unknown patterns
  [if any issues]   → Run cairn check for full compliance audit
```

Keep it fast. This should take seconds, not minutes.
