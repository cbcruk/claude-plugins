---
name: table-driven-tuning
description: >-
  Move game balance out of code into data tables and close the tuning loop with
  a validator, seeded headless simulation, bot metrics, and a metrics baseline.
  Use when balance changes keep requiring code edits, when LLM-generated content
  is too large to review by hand, or when a game's economy, pacing, or choice
  space needs measuring rather than guessing.
---

# Table-driven tuning

Numbers split into three layers — **Shape**, **Knob**, **Content** — and the
loop *edit table → validate → sim → metrics diff* replaces reading code. Tables
are cheap; the loop is not. The worked example throughout is a trade-voyage game
(ports, trade goods, discoveries); swap in the target game's nouns.

Order: `01 → 02 → 03 → 04` are the foundation, `05 → 06 → 07` the measurement
rig, `08–11` the metrics (any order), `12` and `13` any time. Start where the
symptom points.

## Symptom index

| Symptom | Suspect | Recipe |
| --- | --- | --- |
| Rebalancing means editing code | Layers not separated | 01, 02 |
| A tuning table exists but nobody touches it | No way to measure | 06, 11 |
| The late game is boring | Choice space collapses | 08, 09 |
| Money suddenly goes infinite | Arbitrage cycle | 10 |
| Every table edit breaks something | Derived values stored | 03 |
| LLM data errors go unnoticed | Producing without a validator | 04, 13 |
| Sim results differ every run | RNG not injected | 05 |
| Can't tell better from worse | No baseline | 11 |
| Tables are perfect, still not fun | Shape problem | 12 |

Every recipe ends on a **check** — an observable the agent sees with its own
eyes. A recipe is done when its check passes, not when the code is written.

## 01. Map the tunable surface

Stop adding features and inventory every numeric literal:

```bash
rg -n --type ts '(?<![\w.])[0-9]+(\.[0-9]+)?(?![\w])' src/ --pcre2 \
  | rg -v 'test|spec|\.d\.ts' | rg -v '\b(0|1|-1|2)\b'
```

Sort each into a layer:

| Layer | Test | Goes to |
| --- | --- | --- |
| **Shape** | Changing it changes the *form* of the rules | Stays in code, listed in `docs/shape.md` |
| **Knob** | Changing it changes only the *feel* | `data/tuning.ts` |
| **Content** | It differs per instance | That concept's table |

Borderline test: *set it to 0 or infinity — is it a different genre now?* A
tailwind multiplier of `1.3` at 0 is still a sailing game: Knob. A fleet cap of
`4` at 1 is a different game: Shape.

When unsure, classify as Shape. Promoting Shape to Knob later is cheap; finding
out a Knob was Shape means the whole table was tuning the wrong axis.

**Check** — `docs/shape.md` exists and lists every Shape value. The document is
the deliverable: without it, weeks go into tuning an axis tables cannot move.

## 02. Knob table + literal lint

```ts
export const TUNING = {
  baseSpeedPerSail: 0.8,
  windTailwindMul: 1.3,
  scurvyOnsetDays: 45,
  priceVolatility: 0.12,
  priceMeanReversion: 0.05,
  bulkPriceImpact: 0.008,
} as const
```

Enforce it — an unenforced rule lasts about two days against an LLM. Exempt
`data/`, where a block of numbers is the point:

```js
'no-magic-numbers': ['error', { ignore: [0, 1, -1], ignoreArrayIndexes: true, enforceConst: true }]
```

The lint misses `const eff = TUNING.baseSpeedPerSail * 1.2`; that one is for
review.

**Check** — insert `* 1.5` into any function and watch CI go red.

## 03. Table schema — IDs in, indexes resolved once

Author with string IDs; run on indexes. Hard-coded indexes shift when a row is
inserted, and untyped strings break silently on a typo.

```ts
export const PORTS = [
  { id: 'lisboa', name: 'Lisboa', x: 120, y: 340, size: 3, specialty: 'wine' },
  { id: 'sevilla', name: 'Sevilla', x: 138, y: 352, size: 3, specialty: 'olive' },
] as const satisfies readonly PortDef[]

export type PortId = (typeof PORTS)[number]['id']
```

Constraining other tables by `PortId` makes the type checker catch references to
ports that don't exist — half of referential integrity for free. Resolve once at
init:

```ts
const portIndex = new Map(PORTS.map((p, i) => [p.id, i]))
const routes = ROUTES.map((r) => ({ ...r, from: portIndex.get(r.from)!, to: portIndex.get(r.to)! }))
```

Compute derived values (distance from coordinates); a stored copy drifts
silently when the source column moves. A hand override gets an intent-naming
column such as `distanceOverride`.

**Check** — insert a port in the middle of the table. Nothing breaks.

## 04. Validator — the ceiling on table growth

How fast tables can grow is capped by the validator. Write each table's
invariants as code that exits 1 on failure:

1. ID uniqueness
2. Coordinate collisions
3. Route connectivity (BFS from the start port)
4. Discovery reachability
5. Ranges (`basePrice > 0`)
6. Progressability (at least one profitable route on starting funds)

```ts
const reach = bfs(START_PORT, routes)
for (const p of PORTS) if (!reach.has(p.id)) fail('unreachable-port', p.id)
```

A PR that adds a table adds its validate rules in the same change.

**Check** — move a port inland and see `unreachable-port` fire. Every rule has
been seen failing at least once; a rule that never failed may be lying.

## 05. Seeded RNG

An unreproducible sim cannot be debugged. Replace `Math.random()` with an
injectable PRNG carried on the game state — a global singleton breaks under
parallel runs. Give each system its own stream (voyage / prices / events) from
the start, so changing one leaves the others' sequences aligned for clean
comparison; retrofitting this later hurts.

**Check** — `rg -n 'Math\.random' src/` returns nothing, and
`hash(run(42)) === hash(run(42))`.

## 06. Headless simulator

An entry point that runs the game loop for N turns without UI. Emergent problems
don't show up from reading code; some only appear after decades of game time.

```ts
for (let t = 0; t < turns && !state.over; t++) {
  const actions = legalActions(state)
  const chosen = bot(state, actions, rng)
  state = advanceTurn(applyAction(state, chosen, rng), rng)
  trace.push({
    turn: t,
    gold: state.gold,
    action: chosen.kind,
    port: state.currentPort,
    choiceCount: actions.filter((a) => isMeaningful(state, a)).length,
  })
}
```

This requires `legalActions` and `applyAction` as pure functions separated from
UI. If they are tangled with UI, separating them is the first task.

**Check** — one full loop (edit table → validate → sim → metrics diff) finishes
in a few minutes. That time is the real ceiling on development speed.

## 07. Three bots — measure the floor and the ceiling

| Bot | Implementation | Reading |
| --- | --- | --- |
| Random | Uniform over legal actions | Survives here → **no tension** |
| Greedy | Maximize one-step expected gain | Wins here → **no strategic depth** |
| Optimizer | Beam search / 3–5 turn lookahead | Hunts dominant strategies |

Keep the Optimizer shallow; a stronger one turns the work into tuning the bot.

**Check** — the three results separate clearly. Random ≈ Greedy means choices
don't matter; Greedy ≈ Optimizer means planning doesn't matter. Either way the
game is dead.

## 08–11. Metrics

Choice width, lock-in turn, arbitrage detection, and the regression baseline.
Read [`references/metrics.md`](references/metrics.md) when implementing any of
them or interpreting their output.

## 12. Shape spike

Tables all filled, still not fun, and tuning can't fix it — the most expensive
failure, because Shape is out of the tables' reach. Test Shape **before** filling
content.

Spec: 2 ports, 3 goods, 5 discoveries, no UI, sim only, disposable within 24
hours. Look at curve *shapes*, not fun: does choice width avoid converging to 0,
is the money curve sub-exponential, do the three bots separate?

The spike stays outside the main project. A prototype that can't be thrown away
is legacy.

**Check** — the spike passes before any content is filled. Boring with 2 ports
means boring with 40.

## 13. Mass content production

Install the always-on rules into the target repo so they hold during bulk
generation, when this skill may not be loaded:

```bash
mkdir -p .claude/rules
cp "${CLAUDE_PLUGIN_ROOT}/rules/table-driven.md" .claude/rules/table-driven.md
```

Then add `@.claude/rules/table-driven.md` under a `## Rules` section in the
repo's `CLAUDE.md` — the import is what loads it. If the file already exists,
diff it and show the user before overwriting. Adjust the paths in the rule
(`data/tuning.ts`, `docs/shape.md`) and the validate script name to the repo's.

Make validate passing the completion condition of every content request:

```
"Add 40 ports and get `pnpm validate` passing. If it fails, fix and rerun."
```

Slice work as **one table + its invariants + one metric**, not by feature.
Sliced by feature, an LLM writes code; sliced by data, it writes tables.

Left alone, an LLM hard-codes numbers. Output grows while nothing stays tunable —
that is usually what "not making progress" turns out to be. The 02 lint is the
only reliable defense.
