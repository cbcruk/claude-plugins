# Table-Driven Rules

Rules for any change to game logic or data tables in this repo. Apply them on
every edit, including bulk content generation.

## Numbers live in tables

Read every tunable number from `data/tuning.ts` or a content table under
`data/`. When a new number is needed, add a named key to the table first and
reference it from the code. Only `0`, `1`, and `-1` may appear as literals
outside `data/`.

Do the arithmetic on the table value directly. Copying `TUNING.foo` into a local
and multiplying it by a literal hides a magic number from the linter.

## References use IDs

Reference other rows by string ID (`'lisboa'`), typed by the table's ID union
(`PortId`). Resolve IDs to indexes once at load time.

## Derived values are computed

Compute anything that follows from other columns — distances from coordinates,
totals from parts. When a value must be overridden by hand, name the column for
the intent (`distanceOverride`).

## Every table has invariants

A change that adds a table also adds its rules to the validator. Each new rule
is seen failing once before it is trusted.

## Done means validate passes

A data or logic change is done when the validate script exits 0. When it fails,
fix the data and run it again.

## Shape changes need approval

`docs/shape.md` lists the rules that tables cannot change. Ask the user before
editing anything on that list.
