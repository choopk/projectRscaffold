# Type checking (split: app vs. Storybook)

Type-checking for the application and Storybook are run as **two independent `tsc` invocations against two `tsconfig` files**. They never share a single combined pass, and a type error in a story will not block the app's type-check.

## Configs

| Config | Scope | Used by |
|---|---|---|
| `tsconfig.json` | App only — `src/**/*.{ts,tsx}` minus `src/stories/**` and `.storybook/**` | `pnpm check-types`, the IDE |
| `tsconfig.storybook.json` | Storybook surface — `.storybook/**`, `src/stories/**`, plus `src/routeTree.gen.ts` for TanStack Router's module augmentation | `pnpm check-types:storybook` |

`tsconfig.storybook.json` extends `tsconfig.json`, so `compilerOptions` (paths, JSX, types, strictness) stay in sync.

## Commands

From this app (`apps/web`):

```bash
pnpm check-types               # tsc --noEmit                                — app only
pnpm check-types:storybook     # tsc --noEmit -p tsconfig.storybook.json    — Storybook only
```

From the repo root:

```bash
pnpm check-types                              # via turbo, app only across all workspaces
pnpm turbo run check-types:storybook          # via turbo, Storybook only
pnpm --filter @redomicile/web check-types[:storybook]   # direct invocation
```

`pnpm check` at the repo root runs only the **app-level** `check-types` (alongside lint + format). Storybook type-checking is opt-in.

## Behavior, in one table

Error injected in… | `check-types` (app) | `check-types:storybook`
---|---|---
A story file (`src/stories/*.tsx`) | ✅ passes | ❌ fails
A `.storybook/*.ts` config file | ✅ passes | ❌ fails
An app route (`src/routes/*.tsx`) | ❌ fails | ❌ also fails *(see asymmetry below)*
`src/router.tsx`, `src/env.ts` | ❌ fails | ❌ also fails *(same reason)*

### Why the asymmetry

The Storybook compilation includes `src/routeTree.gen.ts` so TanStack Router's `declare module` augmentation is in scope — without it, `createFileRoute('/')` in the scaffolded `Page.stories.ts` route-mocking demo would not type-check. `routeTree.gen.ts` imports the route files, which pulls them transitively into the Storybook compilation graph.

Net effect:
- **App errors propagate to both checks** (a real route bug is caught everywhere).
- **Story errors stay contained in `check-types:storybook`** — they do *not* affect the app check. ← this is the property the split was made to provide.

If you ever need *fully* disjoint checks (route errors must not surface in the Storybook check), the path is to drop `src/routeTree.gen.ts` from `tsconfig.storybook.json` and stop using `createFileRoute` in scaffolded story files. Not worth doing today.

## Why not TypeScript project references?

A "solution" `tsconfig.json` with `references: [...]` and `composite: true` per project is the canonical way to formalize multiple projects. It's deliberately *not* used here because:

1. The goal is **separation of invocations**, not incremental build performance — `tsc -b` isn't needed.
2. References add `composite: true`, `rootDir` constraints, and `.tsbuildinfo` files that this small app doesn't benefit from.
3. The IDE works fine with the current two-config setup: app files resolve to `tsconfig.json`; story / `.storybook/*` files fall back to inferred-project mode (VS Code is reasonable about this) or can be opened with `tsconfig.storybook.json` selected explicitly.

The upgrade to project references is a one-line frontmatter change if needed later.

## How this was verified

Two negative tests confirmed the split (see git history if reproducing):

1. Append `const _x: number = "bad";` to `src/stories/Button.tsx`. `pnpm check-types` passes; `pnpm check-types:storybook` fails on that file. Revert.
2. Append the same line to `src/routes/index.tsx`. `pnpm check-types` fails; `pnpm check-types:storybook` also fails (transitive). Revert.

## Files touched by this split

- `apps/web/tsconfig.json` — narrowed `include`, added `exclude`, added explicit `types: ["vite/client", "node"]`
- `apps/web/tsconfig.storybook.json` — new
- `apps/web/package.json` — added `check-types:storybook` script
- `turbo.json` — registered the new task
