# cli-setup

Use `pillar init` to set up Pillar in a project. It handles framework detection, SDK installation, credential provisioning, tool syncing, and knowledge source creation in a single command.

## Install

Run directly with npx (recommended):

```bash
npx pillar-cli init
```

Or install globally:

```bash
npm install -g pillar-cli
pillar init
```

## What init does

1. **Detects framework** — Next.js (App Router / Pages), Vite+React, Vue, Angular, or vanilla JS
2. **Authenticates** — Opens browser login if not already signed in
3. **Selects product** — Picks from existing products or creates a new one
4. **Installs SDK** — Adds the correct package (`@pillar-ai/react`, `@pillar-ai/vue`, etc.) with the project's package manager
5. **Generates starter code** — Provider wrapper, starter tools file, `.env.local` with agent slug and sync secret
6. **Syncs tools** — Scans for tool definitions and pushes them to the backend
7. **Sets up knowledge** — Detects docs URLs in the project and creates knowledge sources

## Flags

| Flag | Description |
|------|-------------|
| `--agent-slug <slug>` | Skip product selection |
| `--force` | Skip the "already initialized" prompt |
| `--no-validate` | Skip build validation after scaffolding |
| `--debug` | Verbose logging |

## Generated files

For a Next.js App Router project:

| File | Purpose |
|------|---------|
| `components/pillar-provider.tsx` | Client component wrapping `PillarProvider` |
| `lib/pillar-tools.ts` | Starter tools with `navigate_to_page` and `get_current_page` |
| `.env.local` | `NEXT_PUBLIC_PILLAR_SLUG` and `PILLAR_SECRET` |

For other frameworks the paths differ (`src/PillarProvider.tsx`, `src/pillar-tools.ts`), but the structure is the same.

## After init

Init copies an integration prompt to your clipboard. Paste it into your AI editor to wire the provider into your app's layout.

Then:

```bash
pillar doctor        # verify everything is wired up
pillar chat          # test the copilot from the terminal
npm run dev          # start your app and try the panel
```

## Configuration priority

The CLI reads config from multiple sources (highest priority first):

1. **Environment variables** — `PILLAR_SLUG`, `PILLAR_SECRET`, `PILLAR_API_URL`
2. **Config file** — `~/.pillar/config.json` (written by `pillar auth login` and `pillar init`)
3. **Project env** — `.env.local` in the current directory

## Re-initializing

Running `pillar init` in an already-initialized project prompts to re-run or exit. Use `--force` to skip:

```bash
pillar init --force
```

This overwrites the provider, tools, and env files but preserves any custom tools you've added elsewhere.
