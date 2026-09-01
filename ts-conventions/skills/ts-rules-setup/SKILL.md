---
name: ts-rules-setup
description: Install the TypeScript JSDoc and code-style rule files into this repo's .claude/rules/ and wire them into CLAUDE.md. Use when the user asks to set up, install, or update the TS conventions in a project.
---

# Install the TypeScript convention rules

These rules must apply on every edit, not on a trigger phrase. A skill body only
loads when its description matches, so the rules are shipped as files that the
target repo loads as project context instead.

## Files this skill installs

- `${CLAUDE_PLUGIN_ROOT}/rules/jsdoc.md` — JSDoc comments on exported symbols.
- `${CLAUDE_PLUGIN_ROOT}/rules/code-style.md` — file names, component folders,
  comment policy, strict mode.

## Steps

1. Confirm the target. Default to the current working directory; if the user
   named a path, use that instead.

2. Copy the rule files:

   ```bash
   mkdir -p .claude/rules
   cp "${CLAUDE_PLUGIN_ROOT}/rules/jsdoc.md" .claude/rules/jsdoc.md
   cp "${CLAUDE_PLUGIN_ROOT}/rules/code-style.md" .claude/rules/code-style.md
   ```

   If a file already exists, diff it against the plugin copy and show the user
   what would change before overwriting. Local edits win unless the user says
   otherwise.

3. Wire them into `CLAUDE.md` at the repo root. `.claude/rules/*.md` is not
   loaded on its own — the import is what loads it. Append, or create the file:

   ```markdown
   ## Rules

   @.claude/rules/jsdoc.md
   @.claude/rules/code-style.md
   ```

   If a `## Rules` section already exists, add only the missing lines to it.

4. Reconcile with what the repo already does. Read `CLAUDE.md` and any existing
   rules for conventions that contradict the installed files — a different
   `@template`/`@typeParam` choice, a docs-check command, an entry point that
   `@example` blocks import from. Report the conflicts to the user and let them
   pick; do not silently rewrite either side.

5. Report what changed: files written, files skipped, `CLAUDE.md` lines added,
   conflicts found.
