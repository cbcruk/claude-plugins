# claude-plugins

Personal Claude Code plugin marketplace.

Registering the marketplace does not install anything — it lists what is
available. Install each plugin you want by name.

```bash
claude plugin marketplace add cbcruk/claude-plugins
claude plugin install ts-conventions@cbcruk
```

## Plugins

| Plugin | What it does |
| --- | --- |
| `ts-conventions` | Installs the JSDoc and code-style rule files into a repo, and audits existing JSDoc against them. |

## Layout

```
.claude-plugin/marketplace.json   # registry — the only thing at the repo root
ts-conventions/
├── .claude-plugin/plugin.json    # only plugin.json goes in here
├── rules/                        # rule files the setup skill copies out
└── skills/
    ├── ts-rules-setup/SKILL.md
    └── jsdoc-audit/SKILL.md
```

Component directories (`skills/`, `agents/`, `hooks/`) belong at the plugin
root, never inside `.claude-plugin/`.

## Versioning

No plugin declares a `version`. With the field omitted in both `plugin.json` and
the marketplace entry, the source commit SHA is the version, so every pushed
commit reaches installed users. Adding a `version` would mean updates only ship
when that field is bumped — pushed commits would report "already up to date".

## Development

A folder under `~/.claude/skills/` that contains `.claude-plugin/plugin.json`
loads as `<name>@skills-dir` in the next session — no marketplace, no install
step, discovered in place rather than copied to the cache.

```bash
claude plugin validate ./ts-conventions
```

Do not add `--strict` in CI while the no-version strategy above is in effect —
the missing `version` is a warning, and `--strict` turns warnings into failures.

`SKILL.md` edits take effect in the current session. Changes to `hooks/`,
`agents/`, or `.mcp.json` need `/reload-plugins` or a restart.

## Splitting rules

Split by **when a skill fires**, not by topic. An installed plugin's skill
descriptions sit in context every session; only the body loads on trigger. Group
skills that fire at the same moment into one plugin, so no session pays for
descriptions it will never match.

`claude plugin details <name>` prints the always-on and on-invoke token cost
separately — read those numbers before deciding to split.

Rules that must apply on **every** edit do not belong in a skill body at all;
a trigger that only usually matches leaks. Ship them as rule files and let a
setup skill install them into the target repo's `.claude/rules/`. A plugin-root
`CLAUDE.md` is not loaded as project context, so that route is closed.
