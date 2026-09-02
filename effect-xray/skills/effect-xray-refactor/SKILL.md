---
name: effect-xray-refactor
description: >-
  Behavior-preserving refactor of unnecessary useEffects in React/TSX, run as a
  gate rather than a cleanup pass: every removal must carry an explicit
  preservation claim, and risky ones are adversarially falsified before they
  ship. Reach for this when auditing or removing useEffect hooks where the same
  state is set in several places, where a mount-time event or an accumulator is
  involved, or where a naive "convert to a derived value" would silently break a
  handler or drop data. For a single, plainly-derived effect in a short file, a
  direct edit is fine and you can skip this. It triages each effect off the
  source, changes one at a time, and refuses changes whose behavior it cannot
  vouch for — never batch-ripping, never guessing intent the code doesn't state.
---

# useEffect refactor — a gate, not a cleanup pass

Removing a `useEffect` changes render timing and intermediate state, and the
correct replacement is decided by **intent that isn't in the code**. So the job is
never "delete the effects that look derived." It is: read the wiring, pick the
right move, **state what you are claiming about behavior**, and refuse what you
can't vouch for.

Knowing the patterns is not the hard part — React's
["You Might Not Need an Effect"](https://react.dev/learn/you-might-not-need-an-effect)
is the canonical reference and this skill does not restate it. The hard part is
not shipping a change that looks equivalent, passes the tests, and quietly isn't.
That is the only thing this skill installs.

## Calibrate effort first

The full procedure earns its cost when a wrong cut breaks something elsewhere: a
setter written in several places, a mount-time event, an accumulator, a `deps`
mismatch, an unfamiliar component. **For a single, obviously-derived effect in a
short file, skip the ceremony and make the edit** — ceremony to confirm what is
already plain is wasted motion. Scale up only when the picture isn't clear from
reading.

## Workflow

### 1. Read the component, and grep the setters

Read the whole component, not just the effect body — the wiring that matters
(who *else* writes this state) lives outside it. Then, for each setter the effect
calls:

```bash
grep -n 'setFoo(' path/to/File.tsx
```

Every call site outside the effect is a **co-source** of that state. This one
command is the highest-value fact in the whole procedure, and it is the usual
reason a "safe" removal breaks something.

### 2. Triage off the source

| Signal — read it off the source | Diagnosis | Direction |
|---|---|---|
| One `setState`; every read is reactive; no external touch; not inside a deferred callback; **grep says 0 other call sites** | Derived state | Render-time compute |
| Callback touches fetch / subscription / DOM / storage | External sync | Keep, or `useSyncExternalStore` |
| `setState` sits inside a timer / promise / event callback | Deferred response | Keep, or move into the handler |
| **grep finds the setter called outside the effect too** | Interactive state, not pure derived | Not a plain derived const — reconcile the other writers first |
| `setX(p => …p…)` — functional update reading prior state | Accumulator | Can't be derived from current props — keep the state |
| A reactive read is missing from `deps` | Reactivity mismatch | Find out *why* first — a stale-closure bug and an intentional exclusion look identical |

For the last row, `eslint-plugin-react-hooks`' `exhaustive-deps` is more precise
than reading by eye; use it if the project has it.

### 3. Change one effect, then gate it

Replacement recipes and how to choose are in
[`references/replacement-patterns.md`](references/replacement-patterns.md).

**One effect at a time** — each diff stays reviewable and each regression stays
bisectable. After each change, run the repo's checks (`pnpm test`,
`tsc --noEmit`, …).

Then apply the gate, sized to the risk:

- **Plainly derived, grep clean, no cleanup, no mount-time event** → state the
  claim inline and move on. No agents.
- **Any of: setter written elsewhere · a mount-time event · an accumulator · a
  `deps` mismatch · a cleanup function · an unfamiliar component** → run the
  pair:
  1. `effect-refactor-worker` makes the edit and emits a **preservation claim**
     across six axes (extra render/intermediate state · mount-time single run ·
     other writers · prior-state dependence · intentional deps exclusion ·
     cleanup timing).
  2. `effect-adversarial-reviewer` tries to **falsify** that claim with a
     concrete input sequence where old and new observably differ.

  The claim is the contract between them: it is what makes the review
  falsifiable instead of a matter of taste. A `PASS` from the reviewer is a
  complete answer — it is not required to find something.

  On `CONFIRMED`, revert or fix the axis it named; do not argue with the
  scenario without producing a counter-scenario.

### 4. There is no step 4

There used to be: an AST tool (`effect-xray.mjs`) that mapped each effect's wiring.
It is **archived** in the repo this skill came from —
[`cbcruk/effect-evidence`](https://github.com/cbcruk/effect-evidence/tree/main/archive).
It added no correctness over reading the source, and three of its signals were
worse than the alternatives (`grep`, `exhaustive-deps`, and just reading). Do not
go looking for it; the three steps above are the whole procedure.

## The two rules that matter

- **Preserve behavior, or flag — never silently change it.** If the clean-looking
  removal shifts a semantics (an edit gets clobbered, a mount-time record is lost,
  an accumulation resets), that's not a cleanup, it's a bug. When the
  behavior-preserving move lives outside this file (e.g. a parent `key`), do the
  in-file part and record the rest as a recommendation rather than guessing.
- **Don't guess intent.** A `deps` mismatch, or a setter written in several places,
  means the code isn't telling you why. Ask, or leave it and flag it — don't assume
  the tidy answer. Effects that sync an external system usually aren't removable at
  all; a `setState` inside one doesn't make it derived.
