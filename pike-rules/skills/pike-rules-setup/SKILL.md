---
name: pike-rules-setup
description: Install Rob Pike's 5 Rules of Programming (measure before optimizing, simple algorithms, data first) into this repo's .claude/rules/ and wire them into CLAUDE.md. Use when the user asks to set up, install, or update Pike's rules in a project.
---

# Install Pike's rules

These rules must apply on every edit, not on a trigger phrase. A skill body only
loads when its description matches, so the rules are shipped as a file that the
target repo loads as project context instead.

## Files this skill installs

- `${CLAUDE_PLUGIN_ROOT}/rules/pike-rules.md` — when to optimize, how simple
  the algorithm should be, and designing data before code.

## Steps

1. Confirm the target. Default to the current working directory; if the user
   named a path, use that instead.

2. Copy the rule file:

   ```bash
   mkdir -p .claude/rules
   cp "${CLAUDE_PLUGIN_ROOT}/rules/pike-rules.md" .claude/rules/pike-rules.md
   ```

   If the file already exists, diff it against the plugin copy and show the
   user what would change before overwriting. Local edits win unless the user
   says otherwise.

3. Wire it into `CLAUDE.md` at the repo root. `.claude/rules/*.md` is not
   loaded on its own — the import is what loads it. Append, or create the file:

   ```markdown
   ## Rules

   @.claude/rules/pike-rules.md
   ```

   If a `## Rules` section already exists, add only the missing line to it.

4. Reconcile with what the repo already does. Read `CLAUDE.md` and any existing
   rules for guidance that contradicts the installed file — a mandated cache
   layer, a performance budget that requires a specific structure, a benchmark
   command the "measure" rule should name. Report the conflicts to the user and
   let them pick; do not silently rewrite either side.

5. Report what changed: file written or skipped, `CLAUDE.md` lines added,
   conflicts found.
