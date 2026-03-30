## Cairn — Architectural Knowledge Management

This project tracks architectural and design decisions using Cairn.

### Before modifying any code:
1. Check `.cairn/decisions/` for decisions scoped to the files you're changing.
2. Check for a DECISIONS.md in the module directory you're working in.
3. Look for `@cairn` annotations in the source files you're editing.
4. Read the **Anti-Patterns** section of relevant decisions — these define what NOT to do and why.
5. If you encounter a pattern you don't understand the reasoning behind (especially one marked `@cairn-unknown`), do not change it. Flag it for human review.

### When creating or modifying code:
- Add `@cairn` annotations referencing governing decisions at the class/module level.
- Create or update the module's DECISIONS.md when entering an undocumented module (if `auto_create` is enabled in `.cairn/config.yaml`).
- If you make an implementation choice with architectural or design significance that isn't covered by an existing decision, record it with a `@cairn-implementation-inferred` annotation and document your reasoning. Flag it for human review.

### Decision records live at:
- **Full records:** `.cairn/decisions/NNNN-short-title.md`
- **Module summaries:** `DECISIONS.md` in each module directory
- **Inline:** `@cairn` comments in source files
- **Configuration:** `.cairn/config.yaml`
