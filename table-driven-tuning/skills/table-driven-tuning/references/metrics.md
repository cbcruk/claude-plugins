# Metrics

Each metric reads the trace from the headless simulator (recipe 06) under the
three bots (recipe 07). Run it across several seeds before reading anything into
it.

## 08. Choice width — the death metric of a sandbox

The number of *meaningful* actions open each turn: those within 70% of the best
action's estimated gain.

```ts
const isMeaningful = (s, a) =>
  estimateGain(s, a) >
  Math.max(...legalActions(s).map((x) => estimateGain(s, x))) * TUNING.meaningfulRatio
```

From the turn this stays below 3, the player is a spectator.

`estimateGain` is shared with the Greedy bot, so a dumb bot makes a dumb metric.
An inaccurate estimate makes the whole curve lie.

**Check** — compare three seeds. Similar curves point to a structural problem;
divergent curves point to initial conditions.

## 09. Lock-in turn — when a dominant strategy takes over

"Eventually you just shuttle Lisboa ↔ Sevilla." Take the entropy of the Greedy
bot's `action:port` distribution over a 100-turn sliding window and find the turn
where `H < 1.0 bit`.

Lock-in by itself is fine. What matters is **content exhaustion** at lock-in: if
the player has seen 60% of discoveries when it happens, the other 40% go unseen.
Read the two numbers together.

**Check** — lock-in at 20% of total turns means 80% of the game is repetition.

## 10. Infinite arbitrage

Detect **positive-ratio cycles** in the trade graph. Weight each edge by
`-log(round-trip return)`, turning it into negative-cycle detection, and run
Bellman-Ford. `bestTradeRatio` must include sailing days, supply consumption,
and risk; leave any out and routes that actually lose money show up as arbitrage.

Set a threshold rather than aiming for zero — some advantaged routes are the fun
of discovery. For example, fail only above a 1.5× round-trip return.

**Check** — it runs inside validate, so every new port or good is checked
automatically. This is the safety net for mass content production.

## 11. Regression baseline — diff the metrics

Version the *results*, not just the code. Commit `metrics/baseline.json` as the
median over 20 seeds and diff against it, flagging only changes above 15%:

```
⚠ lockInTurn: 1840 → 620 (-66.3%) ← intended?
```

A single-seed baseline mistakes variance for change. Update the baseline only
deliberately; auto-updating hides slow decay.

**Check** — push one tuning value in an obviously bad direction and see the diff
flag it.
