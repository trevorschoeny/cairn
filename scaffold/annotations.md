# Cairn Inline Annotation Reference

This document defines the `@cairn` annotation syntax used in source code files.
Annotations are structured comments that link code to decision records, making
the reasoning behind implementation choices visible at the point where they matter.

**Core principle:** Annotate at the level where the decision is most visible —
class declarations, module entry points, data structure definitions, or interface
signatures. Do not annotate every function unless the decision is function-specific.

---

## Standard Annotation

Use this for code that implements a known, planned decision.

```
// @cairn {DECISION-ID} {short-name}
// @reason {One-line summary of WHY — the business/architectural logic, not the what}
// @see .cairn/decisions/{DECISION-ID}.md
```

**Example:**
```java
// @cairn MENUKIT-003 proxy-over-extender
// @reason Cross-mod compatibility — extenders bind to concrete classes, breaking when
//         other mods also extend the same screen. Proxies delegate through interfaces.
// @see .cairn/decisions/MENUKIT-003-use-proxy-pattern-for-menu-composition.md
public class MenuProxy implements MenuProvider {
```

**When to use:** Any code that directly implements a recorded decision.
The `@see` line is included when `annotations.link_to_decisions` is `true` in config.yaml.

---

## Extended Annotation

Use this for critical decisions where an agent (or future developer) might be
tempted to refactor. The anti-pattern line acts as a guardrail directly in the code.

```
// @cairn {DECISION-ID} {short-name}
// @choice {The specific option chosen from the decision record}
// @reason {Why — the business/architectural logic}
// @anti-pattern Do NOT {what not to do}. {Why not.}
// @see .cairn/decisions/{DECISION-ID}.md
```

**Example:**
```python
# @cairn PEBBLE-001 hybrid-layered-event-sourced
# @choice Layered data model with event-sourced transformation pipeline
# @reason Healthcare compliance requires full auditability of every data transformation.
#         Detection rules need a stable normalized schema to query against.
# @anti-pattern Do NOT store only the normalized form. The raw EHR data must always be
#               preserved for regulatory and debugging needs.
# @see .cairn/decisions/PEBBLE-001-data-model-architecture.md
class NormalizationPipeline:
```

**When to use:** Decisions where the anti-pattern is non-obvious — where a reasonable
developer or agent might "improve" the code by doing exactly the wrong thing.

---

## Implementation-Inferred Annotation

Used by implementation agents when they make a design choice that isn't covered
by an existing decision record. This flags the choice for human review while
documenting the agent's reasoning.

```
// @cairn-implementation-inferred {short-name}
// @reason {Agent's reasoning for the choice}
// @alternatives-considered {What else the agent considered}
// @review-status pending
```

**Example:**
```typescript
// @cairn-implementation-inferred batch-event-writes
// @reason Writing events individually caused N+1 database calls per normalization run.
//         Batching in groups of 100 balances throughput with memory usage.
// @alternatives-considered Individual writes (too slow), unbounded batch (memory risk)
// @review-status pending
private async flushEventBatch(events: TransformationEvent[]): Promise<void> {
```

**When to use:** The implementation agent encounters a choice with design significance
that no existing decision covers. Rather than silently choosing, it documents the
choice and flags it. During the next human review, the annotation can be promoted
to a full decision record or acknowledged as acceptable.

---

## Unknown Pattern Annotation (Brownfield)

Used during discovery when a pattern looks deliberate but nobody can confirm
why it exists. This makes the *absence of knowledge* visible — a future agent
sees this and knows: someone noticed this looks intentional, but the reasoning
hasn't been confirmed yet.

```
// @cairn-unknown {short-name}
// @observation {What the pattern is and why it looks deliberate}
// @status undocumented — do not refactor without human review
```

**Example:**
```python
# @cairn-unknown redis-session-ttl-override
# @observation Session TTL is set to 7200s (2hr) rather than the Redis default of 1800s.
#              This is explicitly configured in three places, suggesting it was deliberate,
#              but no documentation or commit message explains the reasoning.
# @status undocumented — do not refactor without human review
SESSION_TTL = 7200
```

**When to use:** During brownfield discovery (Phase: archaeology). A pattern
appears intentional but the original reasoning is lost. The annotation protects
it from being "cleaned up" by an agent that doesn't understand its significance.

**Important:** Agents encountering `@cairn-unknown` annotations MUST NOT modify
the annotated code without explicit human approval.

---

## Legacy Pattern Annotation (Brownfield)

Used for patterns that have been through discovery and categorized. The status
field distinguishes between patterns that are deliberately preserved, accidentally
present, or recognized as technical debt.

```
// @cairn-legacy {DECISION-ID} {short-name}
// @status {legacy-deliberate | legacy-accidental | acknowledged-debt}
// @constraint {What agents must do or avoid regarding this pattern}
```

**Status values:**
- `legacy-deliberate` — The pattern was intentional and the reasoning holds.
  Treat it like an active decision. Accompanied by a full decision record.
- `legacy-accidental` — The pattern wasn't intentional. Safe to refactor when
  working in this area, but don't go out of your way to change it.
- `acknowledged-debt` — Known technical debt. The pattern is wrong, everyone
  knows it, and fixing it is on the roadmap. Don't build on top of it.

**Example:**
```python
# @cairn-legacy DASH-AUTH-001 filesystem-sessions
# @status acknowledged-debt
# @constraint Do not add features that depend on session persistence across
#             server restarts. This will be replaced with Redis sessions
#             during the multi-tenancy migration.
SESSION_STORE = FileSystemSessionStore("/tmp/sessions")
```

---

## Decision Lineage Annotation (Brownfield Evolution)

Used when code implements a decision that replaced a previous one. The
`@supersedes` and `@preserves` tags create a traceable chain of reasoning
through the codebase's history.

```
// @cairn {NEW-DECISION-ID} {short-name}
// @supersedes {OLD-DECISION-ID}
// @preserves {OLD-DECISION-ID}.{specific-reasoning-that-still-holds}
// @reason {Why the old decision was changed and what was kept}
// @see .cairn/decisions/{NEW-DECISION-ID}.md
```

**Example:**
```python
# @cairn DASH-MT-AUTH-001 two-layer-tenant-roles
# @supersedes DASH-AUTH-002
# @preserves DASH-AUTH-002.security-reasoning
# @reason Original hardcoded roles prevented escalation via data corruption.
#         Multi-tenancy requires configurable roles, but the security constraint
#         survives: TenantRoleValidator enforces cross-tenant escalation boundaries
#         in code, not data — preserving the original security property.
# @see .cairn/decisions/DASH-MT-AUTH-001-two-layer-tenant-roles.md
class TenantRoleValidator:
```

**When to use:** Any brownfield refactor where a new decision replaces an old one
but some of the original reasoning still applies. The `@preserves` tag is the key
innovation — it prevents future agents from seeing "superseded" and assuming the
old reasoning is entirely irrelevant.

---

## Quick Reference

| Annotation | When to use | Agent behavior |
|-----------|-------------|----------------|
| `@cairn` | Code implements a planned decision | Follow the decision, respect anti-patterns |
| `@cairn` (extended) | Critical decisions with non-obvious anti-patterns | Same as above, with explicit guardrail in-code |
| `@cairn-implementation-inferred` | Agent made an unplanned design choice | Document reasoning, flag for human review |
| `@cairn-unknown` | Brownfield pattern with unknown reasoning | **Do not modify** without human approval |
| `@cairn-legacy` | Brownfield pattern with categorized status | Behavior depends on status (see above) |
| `@cairn` with `@supersedes`/`@preserves` | Refactored decision with lineage | Follow new decision, respect preserved reasoning |

## Annotation Density Guidelines

- **Annotate at the module/class level**, not every function.
- **Always annotate** the file where the decision is most visible:
  the interface definition, the entry point, the data structure declaration.
- **Skip annotations** for code that merely follows a pattern established
  elsewhere — annotate the pattern's origin, not every instance of it.
- **Exception:** If a specific function makes a surprising choice that differs
  from the module-level pattern (and there's a decision for why), annotate that function.
- **When in doubt:** One annotation explaining "why" is better than ten
  annotations repeating "what."
