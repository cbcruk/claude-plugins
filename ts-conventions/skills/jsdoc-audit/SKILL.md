---
name: jsdoc-audit
description: Audit existing JSDoc comments on exported symbols against the project's JSDoc rules and report or fix the gaps. Use when the user asks to review, audit, or fix JSDoc or doc comments.
---

# Audit JSDoc against the rules

Read `${CLAUDE_PLUGIN_ROOT}/rules/jsdoc.md` first — it is the standard this
audit checks against. If the target repo has its own `.claude/rules/jsdoc.md`,
that copy wins where the two disagree.

## Scope

Ask the user for the scope if they did not give one. Default to the package's
public entry points and the modules they re-export — the symbols a consumer
sees. Do not audit non-exported symbols.

## What to check

Go symbol by symbol and record a finding only where the rule is actually broken:

- **Missing.** An exported function, class, interface, or type alias with no
  JSDoc. For classes and interfaces, each constructor, method, and property
  counts separately.
- **Summary line.** The first paragraph must be one sentence saying what the
  symbol does. Flag a first paragraph that runs into implementation detail,
  rationale, or caveats — those belong in later paragraphs.
- **Redundant tags.** A `@param` or `@returns` that only restates the type the
  signature already carries. Keep the tag when it states something the signature
  cannot: a valid range, a sentinel return such as `-1`, a unit, an ownership
  rule.
- **Examples.** Symbols with several parameters or non-obvious behaviour need an
  `@example`, and its code block must include the `import` so it runs when
  pasted. Check the import specifier against the one the repo actually
  publishes.
- **Module comments.** A package exposing several modules needs a `@module`
  comment at the top of each module file, with a summary paragraph and a usage
  example.
- **Renderer-dependent syntax.** `> [!IMPORTANT]` blocks and `@example`
  title/description splitting render on JSR and not in plain tooltips. Flag them
  only if the repo does not publish to a renderer that supports them.
- **Consistency.** `@template` vs `@typeParam` — flag whichever the repo uses
  less, not a fixed choice.

## Output

Report findings grouped by file, each as `path:line — what rule, what is wrong`.
Lead with missing comments, then summary-line problems, then the rest.

Apply fixes only if the user asked for them. When fixing, edit the JSDoc in the
same change as any code it describes, then run the repo's doc check
(`docs:check`, `deno doc --lint`, `deno test --doc`, or the equivalent) and
report its output. If no such script exists, say so rather than inventing one.
