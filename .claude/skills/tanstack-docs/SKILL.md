---
name: tanstack-docs
description: Look up live TanStack docs, libraries, add-ons, and ecosystem partners via the @tanstack/cli (JSON mode). Use whenever the user asks about anything in the TanStack ecosystem so answers come from current docs, not stale training data. TRIGGER when: user asks about TanStack Router, Start, Query, Form, Table, Store, Virtual, Ranger, DB, Pacer, or Optimistic; user asks about scaffolding a TanStack app or its add-ons (drizzle, clerk, etc.); user asks about ecosystem providers (auth, database, deployment) for a TanStack project. SKIP for: generic React/TypeScript questions with no TanStack API involved.
---

# TanStack lookups

Default framework is `react` (this repo is TanStack Start + React). Run the
matching command, parse the JSON, then answer in prose. Do NOT dump raw JSON
to the user.

## Command picker

| User's question                         | Command                                                                                 |
| --------------------------------------- | --------------------------------------------------------------------------------------- |
| Conceptual / "how does X work"          | `pnpm dlx @tanstack/cli search-docs "<query>" --library <lib> --framework react --json` |
| Specific doc page by slug               | `pnpm dlx @tanstack/cli doc query <slug> --json`                                        |
| "What TanStack libraries exist"         | `pnpm dlx @tanstack/cli libraries --json`                                               |
| "What add-ons can I scaffold"           | `pnpm dlx @tanstack/cli create --list-add-ons --framework React --json`                 |
| Details on one add-on (e.g. drizzle)    | `pnpm dlx @tanstack/cli create --addon-details <id> --framework React --json`           |
| Auth / DB / deployment provider options | `pnpm dlx @tanstack/cli ecosystem --category <cat> --json`                              |

`<lib>` ∈ {router, start, query, form, table, store, virtual, ranger, db, pacer, optimistic}

## Fallback

If `search-docs` returns nothing relevant, `WebFetch` the docs site directly:
`https://tanstack.com/<lib>/latest/docs`.

## Don'ts

- Don't answer TanStack API questions from training data without running a lookup first — APIs churn.
- Don't paste the raw JSON to the user; synthesize.
- Don't run the CLI for non-TanStack questions just because TanStack is mentioned in passing.
