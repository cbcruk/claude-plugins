# Code Style Rules

Rules for file layout and naming in a TypeScript package. Apply them to every
file you create or move.

## Comments

Do not write inline comments that restate what the code already says.

Do not write banner comments that divide a file into sections — for example
`// ── formatters ─────` or `/* === Utils === */`. A file large enough to need
section dividers should be split into modules instead.

## File names

Use kebab-case for every file name.

```
use-settings.ts
sidebar.tsx
```

## Component layout

Give each component its own folder, and split types and helpers out of the
component file.

```
components/sidebar/
├── sidebar.tsx         # the component
├── sidebar.types.ts    # exported types
└── sidebar.utils.ts    # helpers used by the component
```

## TypeScript

Enable `strict` mode in `tsconfig.json`.

Write an explicit return type on every exported function.

```ts
export function toSlug(value: string): string {
  return value.replaceAll(" ", "-");
}
```

## Strings

Write user-facing and internal strings in English.
