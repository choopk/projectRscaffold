# `@repo/typescript-config`

Shared TypeScript configuration for every workspace in this monorepo.

## Why a shared package?

In a monorepo, every app and package needs a `tsconfig.json`. Without a shared base, each one duplicates 20-ish compiler options, and they drift over time — one app gets `strictNullChecks`, another doesn't. Bugs become "works in app A, not in app B."

This package centralizes the rules. One change here updates every workspace.

## The three layers

TypeScript config is split into three files, each answering a different question:

| File | Answers | Lives in |
|---|---|---|
| `base.json` | "What rules should *every* TS project in this repo follow?" | `packages/typescript-config/base.json` |
| `tanstack-start.json` | "What does a *TanStack Start app* need on top of the base?" | `packages/typescript-config/tanstack-start.json` |
| `tsconfig.json` (per app) | "What's specific to *this one* app?" | e.g. `apps/web/tsconfig.json` |

Each layer extends the one above it.

---

### 1. `base.json` — language-level rules

Holds settings that should be true for **any** TypeScript code in this repo, regardless of framework or runtime:

- **Strictness**: `strict`, `noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`, `noUncheckedSideEffectImports`
- **Module system**: `target: ES2022`, `module: ESNext`, `moduleResolution: bundler`, `isolatedModules`, `verbatimModuleSyntax`
- **Bundler ergonomics**: `allowImportingTsExtensions`, `noEmit`, `resolveJsonModule`, `esModuleInterop`
- **Hygiene**: `skipLibCheck`, `forceConsistentCasingInFileNames`
- **Baseline lib**: `["ES2022"]` — no DOM, no Node — kept neutral so backend or library packages can extend it without inheriting browser globals

A future `packages/utils` or `apps/api` would extend `base.json` directly.

### 2. `tanstack-start.json` — preset for the web app shape

Extends `base.json` and adds what a **TanStack Start (React + Vite + SSR)** app specifically needs:

- `jsx: "react-jsx"` — React 17+ automatic JSX transform
- `lib: ["ES2022", "DOM", "DOM.Iterable"]` — browser globals (`document`, `window`, etc.)
- `types: ["vite/client"]` — Vite's import.meta.env types, asset imports

The base stays clean of browser-specific concerns; this preset layers them on. If we later add a second TanStack Start app, it extends the same preset — guaranteed parity.

If we add an `apps/api` (Node-only backend), we'd create a sibling preset `node.json` with `types: ["node"]` and `lib: ["ES2022"]`.

### 3. `tsconfig.json` (per workspace) — local concerns only

Each app or package has its own `tsconfig.json` that **only** contains what's truly local:

```jsonc
// apps/web/tsconfig.json
{
  "extends": "@repo/typescript-config/tanstack-start.json",
  "include": ["**/*.ts", "**/*.tsx"],
  "compilerOptions": {
    "paths": {
      "#/*": ["./src/*"],
      "@/*": ["./src/*"]
    }
  }
}
```

That's it. Nine lines. Three things only this workspace cares about:

- **`extends`** — which preset to inherit
- **`include`** — what files belong to this project
- **`paths`** — its own import aliases (`#/*`, `@/*`)

Everything else (target, strictness, JSX, libs) comes from the preset. Drift becomes impossible.

---

## How they're wired together

`extends` chains compose like CSS:

```
apps/web/tsconfig.json
  └── extends "@repo/typescript-config/tanstack-start.json"
        └── extends "./base.json"
```

When `tsc` reads `apps/web/tsconfig.json`, it walks the chain top-down, merging compiler options. Local options in `apps/web/tsconfig.json` win over the preset; the preset wins over the base.

The `@repo/typescript-config` package is resolved through pnpm — `apps/web/package.json` declares `"@repo/typescript-config": "workspace:*"` in `devDependencies`, which symlinks the local package into `node_modules`. No publishing required.

## Adding a new app

1. `mkdir apps/foo` with its own `package.json` declaring `"@repo/typescript-config": "workspace:*"` in devDeps
2. Create `apps/foo/tsconfig.json` extending the relevant preset
3. Run `pnpm install` to link the workspace

If `foo` is also a TanStack Start app, it extends `tanstack-start.json` and inherits the same TypeScript behavior as `apps/web` for free.

## Adding a new preset

If a new app type doesn't fit any existing preset (e.g. a Node service, a Cloudflare Worker, a CLI tool), create a sibling file next to `tanstack-start.json`:

```jsonc
// packages/typescript-config/node.json
{
  "extends": "./base.json",
  "compilerOptions": {
    "types": ["node"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext"
  }
}
```

Then add it to the `files` array in `packages/typescript-config/package.json` so consumers can extend it.

## Why not put everything in one big `tsconfig.base.json` at the root?

That works for a single-flavor repo, but breaks down once you have multiple runtime targets. A Node backend doesn't want `lib: ["DOM"]`. A browser app needs it. A worker needs `WebWorker`. Splitting into `base.json` + per-runtime presets lets each project pick its target without duplicating language-level rules.
