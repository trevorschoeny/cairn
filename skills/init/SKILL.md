---
name: init
description: Initialize Cairn in the current project. Creates .cairn/ directory with config, catalogs, decision template, and annotation reference. Use when the developer says "cairn init", "set up cairn", "initialize cairn", or "add cairn to this project".
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Bash(*)
---

# Cairn Init

Set up Architectural Knowledge Management in the current project by copying
scaffold files from the Cairn plugin into the project directory.

## Pre-flight Check

1. Check if `.cairn/config.yaml` already exists in the project root.
   - If it does: tell the developer "This project already has Cairn. To
     reinitialize, remove `.cairn/` first. Existing decision records will
     NOT be affected by reinit." Then stop.
   - If it doesn't: proceed.

## Step 1: Create .cairn/ directory

Copy the scaffold files from the plugin into the project:

```
Source: ${CLAUDE_PLUGIN_ROOT}/scaffold/
Target: .cairn/
```

Create these directories and copy these files:

1. Create `.cairn/decisions/` directory
2. Create `.cairn/catalog/` directory
3. Copy `${CLAUDE_PLUGIN_ROOT}/scaffold/config.yaml` → `.cairn/config.yaml`
4. Copy `${CLAUDE_PLUGIN_ROOT}/scaffold/annotations.md` → `.cairn/annotations.md`
5. Copy `${CLAUDE_PLUGIN_ROOT}/scaffold/decisions/_template.md` → `.cairn/decisions/_template.md`
6. Copy all files from `${CLAUDE_PLUGIN_ROOT}/scaffold/catalog/` → `.cairn/catalog/`
   - structure-composition.md
   - data-state.md
   - communication-interfaces.md
   - errors-resilience.md
   - identity-conventions.md
   - resources-performance.md
   - security.md

**Do NOT overwrite existing files.** If a file already exists at the target
path, skip it and note that it was skipped.

## Step 2: Update CLAUDE.md

Read `${CLAUDE_PLUGIN_ROOT}/scaffold/claude-md-append.md` for the content to append.

- If `CLAUDE.md` exists and already contains the word "Cairn": skip, note it was skipped.
- If `CLAUDE.md` exists but doesn't mention Cairn: append the content (with two blank lines before it).
- If `CLAUDE.md` doesn't exist: create it with the append content.

## Step 3: Setup Wizard

After copying files, walk the developer through configuring their project.
Ask these questions one at a time — don't dump them all at once. After each
answer, update `.cairn/config.yaml` with their choice before moving on.

### 3a. Project name

"What's this project called?"

Use whatever they say. If they give a short name, use it. If they say
something like "it's the Hugo static site generator," use "Hugo".
Write it to `project.name` in config.yaml.

### 3b. Decision ID prefix

"Pick a short prefix for decision IDs — usually a project abbreviation.
For example, if this is called Pebble, decisions would be PEBBLE-001,
PEBBLE-002, etc. What prefix do you want?"

Write it to `decision_id.prefix` in config.yaml. Uppercase it.

### 3c. Planning preset

"When we plan decisions together, do you want me to speak in technical
shorthand (developer mode) or explain each concept first (guided mode)?"

- If they say technical/developer/fast/shorthand → `developer`
- If they say explain/guided/beginner/thorough → `guided`
- Default to `developer` if unclear

Write it to `preset` in config.yaml.

### 3d. Project type

"Is this a new project (greenfield) or does it have existing code?"

This doesn't go in config — it determines the next step suggestion:
- Greenfield → suggest "cairn plan"
- Existing code → suggest "cairn discover"

## Step 4: Report

After the wizard completes, show:

```
Cairn initialized!

Created:
  .cairn/config.yaml          — Project configuration (configured)
  .cairn/annotations.md       — Inline annotation reference
  .cairn/decisions/_template.md — Decision record template
  .cairn/catalog/             — 7 decision catalogs (46 decision points)
  CLAUDE.md                   — Updated with Cairn instructions

Ready to go! Say "cairn plan" or "cairn discover" to get started.
```

## Rules

- Use the Read tool to read scaffold files from `${CLAUDE_PLUGIN_ROOT}/scaffold/`.
- Use the Write tool to create files in the project.
- Never overwrite existing files without asking.
- If `${CLAUDE_PLUGIN_ROOT}` doesn't resolve or the scaffold files aren't found,
  tell the developer the plugin may not be installed correctly.
- During the wizard, ask ONE question at a time. Wait for the answer before
  asking the next question.
