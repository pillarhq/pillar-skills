---
name: pillar-best-practices
description: Best practices for integrating the Pillar SDK and CLI into web applications. Use when adding an AI assistant panel, setting up Pillar with the CLI, syncing tools, managing knowledge sources, or designing tool workflows.
---

# Pillar SDK & CLI Best Practices

Best practices for integrating the Pillar SDK and CLI into web applications. Covers project setup with the CLI, tool syncing, knowledge source management, and SDK integration patterns.

## When to Apply

Reference these guidelines when:

- Setting up Pillar in a new project (`pillar init`)
- Syncing tools to the Pillar backend (`pillar sync`)
- Managing knowledge sources (`pillar knowledge`)
- Diagnosing integration issues (`pillar doctor`)
- Adding Pillar SDK to a React or Next.js project
- Setting up `PillarProvider` in your app
- Defining tools for the AI assistant to discover and call
- Creating tool handlers to execute user requests
- Designing multi-tool workflows for agentic operations

## Essential Rules

### CLI

| Priority | Rule | Description |
|----------|------|-------------|
| CRITICAL | `cli-setup` | Use `pillar init` to scaffold a project — handles framework detection, SDK install, credentials, and first sync |
| HIGH | `cli-sync` | Use `pillar sync --scan` to push tool definitions; use `--watch` in development and CI for deploys |
| HIGH | `cli-knowledge` | Use `pillar knowledge` to manage docs and help content the copilot uses to answer questions |

### SDK

| Priority | Rule | Description |
|----------|------|-------------|
| CRITICAL | `setup-provider` | Always wrap your app with PillarProvider |
| CRITICAL | `setup-nextjs` | Next.js App Router requires a 'use client' wrapper |
| CRITICAL | `schema-compatibility` | inputSchema must follow cross-model formatting rules (arrays need items, no type unions) |
| HIGH | `tool-descriptions` | Write specific, AI-matchable descriptions and keep tools focused |
| HIGH | `tool-return-values` | Return flat data from execute -- never `{ success: true }` without the actual data |
| HIGH | `tool-handlers` | Use centralized handlers with proper cleanup |
| HIGH | `guidance-field` | Use the guidance field for agent-facing disambiguation and prerequisites |
| HIGH | `workflow-patterns` | Design multi-tool workflows using the distributed guidance pattern |
| HIGH | `tool-overlap-audit` | Audit existing tools for overlap before creating new ones |
| HIGH | `codebase-verification` | Verify API shapes against the actual codebase -- never guess |
| HIGH | `form-queue-pattern` | For form-opening tools, defer completion until user submits; queue multiple forms for sequential execution |

## Quick Reference — CLI

### 1. Project Setup (CRITICAL)

Install and initialize Pillar in one command:

```bash
npx pillar-cli init
```

This detects your framework, installs the SDK, generates a provider wrapper and starter tools, creates credentials, syncs tools, and sets up knowledge sources. See `rules/cli-setup.md`.

If you already have a product key:

```bash
npx pillar-cli init --product-key your-slug
```

### 2. Tool Syncing (HIGH)

Push tool definitions to the backend:

```bash
pillar sync --scan ./src          # one-time sync
pillar sync --scan ./src --watch  # re-sync on file changes
pillar sync status --scan ./src   # compare local vs remote
```

In CI/CD:

```bash
npx pillar-cli sync --scan ./src
```

Set `PILLAR_SLUG` and `PILLAR_SECRET` as environment variables. See `rules/cli-sync.md`.

### 3. Knowledge Sources (HIGH)

Add docs and help content the copilot uses to answer questions:

```bash
pillar knowledge add https://docs.myapp.com
pillar knowledge list
pillar knowledge status
pillar knowledge sync <source-id>
pillar knowledge remove <source-id>
```

See `rules/cli-knowledge.md`.

### 4. Diagnostics

Verify the integration is healthy:

```bash
pillar doctor
```

Checks product key, sync secret, SDK version, tool sync status, knowledge sources, and embed config reachability.

### 5. Testing

Test the copilot without starting your app:

```bash
pillar chat "how do I export data?"   # single question
pillar chat                            # interactive session
```

### 6. Authentication

```bash
pillar auth login     # opens browser to authenticate
pillar auth status    # check current session
pillar auth logout    # clear credentials
```

Credentials are stored in `~/.pillar/config.json`.

## Quick Reference — SDK

### 7. Provider Setup (CRITICAL)

Always wrap your app with `PillarProvider`:

```tsx
import { PillarProvider } from '@pillar-ai/react';

<PillarProvider productKey="your-product-key">
  {children}
</PillarProvider>
```

### 8. Next.js App Router (CRITICAL)

Create a client wrapper component:

```tsx
// providers/PillarSDKProvider.tsx
'use client';

import { PillarProvider } from '@pillar-ai/react';

export function PillarSDKProvider({ children }: { children: React.ReactNode }) {
  return (
    <PillarProvider productKey={process.env.NEXT_PUBLIC_PILLAR_PRODUCT_KEY!}>
      {children}
    </PillarProvider>
  );
}
```

### 9. Tool Descriptions (HIGH)

Write specific descriptions the AI can match:

```tsx
// Good - specific and includes context
description: 'Navigate to billing settings. Suggest when user asks about payments, invoices, or subscription.'

// Bad - too generic
description: 'Go to billing'
```

### 10. Tool Registration (HIGH)

In React components, use the `usePillarTool` hook — it auto-registers on mount and cleans up on unmount:

```tsx
import { usePillarTool } from '@pillar-ai/react';

usePillarTool({
  name: 'create_dashboard',
  description: 'Create a new empty dashboard.',
  guidance: 'First step in dashboard workflow. Returns dashboard_uid needed by create_*_panel tools.',
  type: 'trigger_tool',
  autoRun: true,
  inputSchema: { type: 'object', properties: { title: { type: 'string' } }, required: ['title'] },
  execute: async (data) => {
    const result = await api.createDashboard(data.title);
    return { uid: result.uid };
  },
});
```

Outside React (or for imperative registration), use `pillar.defineTool()` with the same schema shape. It returns an unsubscribe function for cleanup.

Sync tools after defining them: `pillar sync --scan ./src`

### 11. Guidance Field (HIGH)

Use `guidance` for agent-facing instructions that help the LLM choose and chain tools:

```tsx
get_available_datasources: {
  description: 'Get datasources available for creating visualizations.',
  guidance: 'Call BEFORE creating dashboards or panels. If zero results, ask user to create one.',
}
```

### 12. Audit for Overlap (HIGH)

Before creating a new tool, search existing tools for semantic overlap:

```tsx
// Found existing: save_dashboard handles "create a dashboard"
// Don't create a second create_dashboard -- extend or disambiguate instead
```

### 13. Decompose Large Tools (HIGH)

Prefer smaller tools with tight schemas over one large tool with many modes:

```tsx
// Instead of one "manage_user" with an operation enum,
// split into focused tools:
invite_user: { description: 'Invite a new user by email', type: 'trigger_tool' }
remove_user: { description: 'Remove a user from the org', type: 'trigger_tool' }
change_user_role: { description: 'Change a user role', type: 'trigger_tool' }
```

## How to Use

Read individual rule files for detailed explanations and code examples:

```
rules/cli-setup.md
rules/cli-sync.md
rules/cli-knowledge.md
rules/setup-provider.md
rules/setup-nextjs.md
rules/tool-descriptions.md
rules/tool-handlers.md
rules/schema-compatibility.md
rules/guidance-field.md
rules/workflow-patterns.md
rules/tool-overlap-audit.md
rules/codebase-verification.md
rules/form-queue-pattern.md
```
