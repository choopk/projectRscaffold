# redomicile

Monorepo scaffolded with [Turborepo](https://turborepo.dev) and [TanStack Start](https://tanstack.com/start).

## Structure

```
apps/
  web/          # TanStack Start app
packages/       # Shared packages (empty for now)
```

## Prerequisites

- Node.js `>=22.12.0`
- pnpm `11.2.2` (declared via `packageManager` in `package.json`)

## Commands

```bash
pnpm install        # Install all workspace dependencies
pnpm dev            # Start all apps in dev mode
pnpm build          # Build all apps and packages
pnpm check-types    # Run TypeScript across workspaces
pnpm lint           # Lint all workspaces
pnpm clean          # Clean build outputs
```
