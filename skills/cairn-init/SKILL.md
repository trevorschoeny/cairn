---
name: cairn-init
description: >
  Initialize Cairn in the current project. Creates the .cairn/ directory with
  config, catalogs, decision template, and annotation reference. Also appends
  Cairn instructions to CLAUDE.md. Use when the developer says "cairn init",
  "set up cairn", "initialize cairn", "add cairn to this project", or any
  phrasing that implies wanting to set up Cairn in a project for the first time.
disable-model-invocation: true
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

## Step 3: Report

After completing the setup, tell the developer:

```
Cairn initialized!

Created:
  .cairn/config.yaml          — Project configuration (edit this first)
  .cairn/annotations.md       — Inline annotation reference
  .cairn/decisions/_template.md — Decision record template
  .cairn/catalog/             — 7 decision catalogs (46 decision points)
  CLAUDE.md                   — Updated with Cairn instructions

Next steps:
  1. Edit .cairn/config.yaml to set your project name and decision ID prefix
  2. New project?      Say "cairn plan" to run a collaborative planning session
  3. Existing project? Say "cairn discover" to surface embedded decisions
```

## Rules

- Use the Read tool to read scaffold files from `${CLAUDE_PLUGIN_ROOT}/scaffold/`.
- Use the Write tool to create files in the project.
- Never overwrite existing files without asking.
- If `${CLAUDE_PLUGIN_ROOT}` doesn't resolve or the scaffold files aren't found,
  tell the developer the plugin may not be installed correctly.
