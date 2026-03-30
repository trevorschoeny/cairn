# Cairn

**Architectural Knowledge Management for AI-Assisted Development**

Cairn records the *why* behind your code's structure so AI agents
don't "improve" away your deliberate design decisions.

## The Problem

Code shows WHAT was built. The WHY — the reasoning behind architectural
and design choices — exists in conversations, memory, or nowhere. When
agents encounter code later, they may refactor it in ways that violate
deliberate decisions they don't know about.

## The Solution

Cairn provides:
- **Decision records** — structured markdown files recording what was
  decided, why, and what NOT to do (anti-patterns)
- **Inline annotations** — `@cairn` comments linking code to decisions
- **Planning skill** — collaborative sessions to make design decisions
  before coding
- **Implementation skill** — behavioral overlay ensuring agents respect
  decisions during coding
- **Discovery skill** — archaeological tool for surfacing decisions
  already embedded in existing code

## Install

Cairn is a Claude Code plugin. Install it from GitHub:

```
/plugin install github:trevorschoeny/cairn
```

Then initialize it in any project:

```
/cairn:init
```

This creates:

| What | Where | Purpose |
|------|-------|---------|
| Decision infrastructure | `.cairn/` | Config, catalogs (46 decision points across 7 areas), decision template, annotation reference |
| Agent instructions | `CLAUDE.md` | Tells agents about Cairn (appended to existing file) |

The plugin also provides sub-agents (preflight, annotator, reviewer, scanner)
and skills (planning, implementation, discovery) that are available automatically
once the plugin is installed.

Then edit `.cairn/config.yaml` to set your project name and decision ID prefix.

## Usage

### New Project (Greenfield)

Say **"cairn plan"** — runs a collaborative planning session:
1. Calibrates to your preferred depth (minimal / collaborative / deep)
2. Collaborates on a lightweight spec of what you're building
3. Walks through relevant design decisions from the catalog
4. Records choices as decision records in `.cairn/decisions/`

### Existing Project (Brownfield)

Say **"cairn discover"** — runs architectural archaeology:
1. Scans the codebase to identify patterns
2. Asks you to confirm which patterns were deliberate
3. Records confirmed decisions with confidence levels
4. Tags unexplained patterns with `@cairn-unknown` (agents won't touch these)

### During Implementation

The implementation skill activates automatically when agents code in a
Cairn-enabled project:
- **Before coding:** Preflight sub-agent briefs the agent on applicable decisions
- **During coding:** Anti-patterns are treated as hard constraints
- **After coding:** Annotator sub-agent links code to governing decisions

### Commands

```
/cairn:status    → Quick inventory of decisions and annotations
/cairn:check     → Full compliance audit against recorded decisions
```

## Design Catalogs

Cairn ships with 46 decision points organized into 7 concern areas:

| Catalog | Decisions | Covers |
|---------|-----------|--------|
| Structure & Composition | 10 | Paradigm, module design, coupling, extension, boundaries |
| Data & State | 10 | Schema, persistence, state management, caching, ownership |
| Communication & Interfaces | 8 | API style, sync/async, events, contracts, versioning |
| Errors & Resilience | 6 | Error types, validation, recovery, observability, failure philosophy |
| Identity & Conventions | 5 | Code organization, naming, config, tests, versioning |
| Resources & Performance | 3 | Concurrency, resource pooling, computation placement |
| Security | 4 | Authentication, authorization, trust boundaries, data protection |

Not all 46 apply to every project. The planning skill filters by project type
and only presents what's relevant.

## Works With Any Agent Harness

Cairn's core interface is the filesystem. Decision records are markdown files.
Annotations are code comments. Any agent that can read files can consume Cairn:

- **Claude Code** — Full support with plugin, sub-agents, skills, and commands
- **Loop harnesses** — Agents read `.cairn/decisions/` on each fresh iteration
- **Multi-agent orchestrators** — Shared `.cairn/` directory = shared architectural contract
- **Any future harness** — If it reads files before coding, Cairn works

## The Name

A cairn is a stack of stones marking a trail for whoever comes through next.
That's what this does for code — it marks the reasoning trail so the next
developer or agent knows why the path goes this way.

## License

MIT
