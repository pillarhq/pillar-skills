# Pillar SDK Complete Reference

This is the complete reference for integrating the Pillar SDK. It covers installation, configuration, tools, handlers, and advanced features.

## Overview

Pillar SDK provides an AI-powered assistant panel for your application. Users can ask questions, and the AI can suggest tools that execute directly in your app.

**What you'll build:**
- A slide-out assistant panel with AI chat
- A sidebar trigger (or custom button) that opens the panel
- Tools the AI can suggest to users

## Installation

```bash
npm install @pillar-ai/sdk @pillar-ai/react
```

Or with other package managers:

```bash
# yarn
yarn add @pillar-ai/sdk @pillar-ai/react

# pnpm
pnpm add @pillar-ai/sdk @pillar-ai/react
```

## Provider Setup

### Basic Setup

Wrap your app with `PillarProvider`:

```tsx
import { PillarProvider } from '@pillar-ai/react';

function App() {
  return (
    <PillarProvider productKey="your-product-key">
      <YourApp />
    </PillarProvider>
  );
}
```

### Next.js App Router

**Important:** Since `PillarProvider` uses React context, create a client wrapper component:

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

Then use in your root layout:

```tsx
// app/layout.tsx
import { PillarSDKProvider } from '@/providers/PillarSDKProvider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <PillarSDKProvider>
          {children}
        </PillarSDKProvider>
      </body>
    </html>
  );
}
```

### Configuration Options

All configuration is optional with sensible defaults:

```tsx
<PillarProvider
  productKey="your-product-key"
  config={{
    edgeTrigger: {
      enabled: true,  // Show sidebar tab on screen edge
    },
    mobileTrigger: {
      enabled: true,
      position: 'bottom-right',
      icon: 'sparkle',
      size: 'medium',
    },
    panel: {
      position: 'right',  // 'left' | 'right'
      mode: 'push',       // 'push' | 'overlay'
      width: 380,
    },
    theme: {
      mode: 'auto',  // 'light' | 'dark' | 'auto'
      colors: {
        primary: '#2563eb',
      },
    },
    textSelection: {
      enabled: true,
      label: 'Ask AI',
    },
  }}
/>
```

## Defining Tools

Tools are things users can do in your app that the AI can suggest.

### Basic Tool Definition

```tsx
// lib/pillar/tools.ts
import { PillarProvider, usePillarTool } from '@pillar-ai/react';

function useAppTools() {
  const router = useRouter();

  usePillarTool({
    name: 'open_settings',
    description: 'Navigate to the settings page',
    type: 'navigate',
    autoRun: true,
    execute: () => router.push('/settings'),
  });

  usePillarTool({
    name: 'invite_member',
    description: 'Open the invite team member modal',
    examples: [
      'invite someone to my team',
      'add a new user',
      'how do I invite people?',
    ],
    type: 'trigger_tool',
    execute: () => openInviteModal(),
  });
}
```

### Tool Types

| Type | Description | Use Case |
|------|-------------|----------|
| `navigate` | Navigate to a page in your app | Settings, dashboard, detail pages |
| `trigger_tool` | Run custom logic | Open modals, start wizards, toggle features |
| `query` | Fetch data and return to the agent | List items, get details, lookups (auto-sets `returns: true`) |
| `inline_ui` | Show interactive UI in chat | Forms, confirmations, previews |
| `external_link` | Open URL in new tab | Documentation, external resources |
| `copy_text` | Copy text to clipboard | API keys, code snippets |

### Writing Good Descriptions

The AI matches user queries to your tool descriptions. Be specific:

```tsx
// Good - specific about when to use
description: 'Navigate to billing page. Suggest when user asks about payments, invoices, or subscription.'

// Less helpful - too generic
description: 'Go to billing'
```

### Adding Example Phrases

Help the AI understand different ways users might phrase requests:

```tsx
examples: [
  'invite someone to my team',
  'add a new team member',
  'how do I add users?',
  'share access with someone',
]
```

### Data Extraction with Schema

Define a schema to have the AI extract structured data from user messages:

```tsx
add_source: {
  description: 'Add a new knowledge source',
  type: 'trigger_tool',
  dataSchema: {
    type: 'object',
    properties: {
      url: {
        type: 'string',
        description: 'URL of the source to add',
      },
      name: {
        type: 'string',
        description: 'Display name for the source',
      },
    },
    required: ['url'],
  },
}
```

## defineTool() — Preferred API

For new code, use `pillar.defineTool()` to co-locate the tool definition and handler in one call. This is the recommended pattern — it prevents drift between definition and handler, and the `guidance` field is extracted automatically by `pillar-sync --scan`.

```tsx
// tools/dashboardCrud.ts
import type { PillarInstance } from './types';

export function registerDashboardTools(pillar: PillarInstance): Array<() => void> {
  return [
    pillar.defineTool({
      name: 'create_dashboard',
      description: 'Create a new empty dashboard, returning its UID for panel creation.',
      guidance:
        'First step in any dashboard workflow. Returns dashboard_uid needed by all create_*_panel tools.',
      type: 'trigger_tool',
      autoRun: true,
      autoComplete: true,
      inputSchema: {
        type: 'object',
        properties: {
          title: { type: 'string', description: 'Dashboard title' },
        },
        required: ['title'],
      },
      execute: async (data: { title: string }) => {
        const response = await api.createDashboard(data.title);
        return { success: true, data: { uid: response.uid } };
      },
    }),
  ];
}
```

Key properties of `defineTool()`:
- Handler is co-located with the definition (`execute` field)
- Returns an unsubscribe function for cleanup
- `guidance` field is supported and synced via `--scan`
- All `defineTool` tools automatically return data to the agent

## Tool Syncing

### Automatic scanning (recommended)

```bash
PILLAR_SLUG=my-app PILLAR_SECRET=xxx npx pillar-sync --scan ./src/tools
```

The scanner uses the TypeScript compiler API to statically extract metadata from `defineTool()` and `usePillarTool()` calls. It extracts: `name`, `description`, `guidance`, `type`, `inputSchema`, `examples`, `autoRun`, `autoComplete`.

No barrel file or manifest needed. The scanner finds all tool definitions recursively.

### What the scanner cannot extract

- `execute` functions (runtime only)
- Variable references (must be inline literals)
- Computed values

If a field can't be resolved statically, the scanner skips it with a warning.

## Tool Handlers (Legacy Pattern)

### Centralized Handler Pattern

Create a dedicated component for all your handlers:

```tsx
// components/PillarToolHandlers.tsx
'use client';

import { usePillar } from '@pillar-ai/react';
import { useRouter } from 'next/navigation';
import { useEffect } from 'react';

export function PillarToolHandlers() {
  const { pillar } = usePillar();
  const router = useRouter();

  useEffect(() => {
    if (!pillar) return;

    const handlers = [
      pillar.onTask('navigate', ({ path }) => {
        router.push(path);
        return { success: true };
      }),

      pillar.onTask('invite_member', ({ email, role }) => {
        openInviteModal({ email, role });
        return { success: true };
      }),
    ];

    return () => handlers.forEach(unsub => unsub?.());
  }, [pillar, router]);

  return null;
}
```

Include it in your layout:

```tsx
// app/layout.tsx
<PillarSDKProvider>
  <PillarToolHandlers />
  {children}
</PillarSDKProvider>
```

### Handler Return Values

```tsx
// Success
return { success: true };

// Success with message
return { success: true, message: 'Invite sent!' };

// Failure
return { success: false, message: 'Something went wrong' };
```

### Built-in Handlers

Pillar provides fallback handlers for common tool types:

| Type | Default Behavior |
|------|------------------|
| `navigate` | `window.location.href = path` |
| `external_link` | `window.open(url, '_blank')` |
| `copy_text` | `navigator.clipboard.writeText(text)` |

Register your own handlers to override these defaults.

## React Hooks

### usePillar

Access the SDK instance and panel state:

```tsx
import { usePillar } from '@pillar-ai/react';

function MyComponent() {
  const { pillar, open, close, toggle, isPanelOpen } = usePillar();

  return (
    <button onClick={() => toggle()}>
      {isPanelOpen ? 'Close' : 'Open'} Assistant
    </button>
  );
}
```

### useHelpPanel

Simplified panel controls:

```tsx
import { useHelpPanel } from '@pillar-ai/react';

function HelpButton() {
  const { open, isOpen } = useHelpPanel();

  return <button onClick={open}>Get Help</button>;
}
```

## Context API (Enhancement)

Pass contextual information to help the AI provide better assistance:

```tsx
// Sync current route
import { usePillar } from '@pillar-ai/react';
import { usePathname } from 'next/navigation';

function RouteContextSync() {
  const { pillar } = usePillar();
  const pathname = usePathname();

  useEffect(() => {
    pillar?.setContext({ currentPage: pathname });
  }, [pillar, pathname]);

  return null;
}
```

### Context Properties

| Property | Type | Description |
|----------|------|-------------|
| `currentPage` | `string` | Current URL path |
| `currentFeature` | `string` | Human-readable feature name |
| `userRole` | `string` | User's role for tool filtering |
| `userState` | `string` | User's current state (onboarding, trial, active) |
| `errorState` | `object` | Current error with code and message |
| `custom` | `object` | Any additional context data |

### Tool Filtering with Context

Use `requiredContext` to control which tools the AI suggests:

```tsx
delete_user: {
  description: 'Delete a user from the organization',
  type: 'trigger_tool',
  requiredContext: { userRole: 'admin' },
}
```

## Plans (Automatic)

The AI automatically creates multi-step plans when a user's request requires multiple tools. Your handlers work the same way - no special code needed.

**Example:** When a user asks "Help me set up my workspace", the AI might create a plan:
1. Create a new project
2. Add your first knowledge source
3. Configure the assistant
4. Invite your team

Each step executes your registered tool handlers automatically.

## Custom Cards (Advanced)

For `inline_ui` tools, you can render custom React components in the chat:

```tsx
// Define the tool
show_product: {
  type: 'inline_ui',
  description: 'Show a product preview card',
}

// Register the card component
<PillarProvider
  productKey="..."
  cards={{
    show_product: ({ data, onComplete }) => (
      <ProductCard product={data} onSelect={() => onComplete({ success: true })} />
    ),
  }}
>
```

## Environment Variables

```bash
# .env.local
NEXT_PUBLIC_PILLAR_PRODUCT_KEY=your-product-key
```

## TypeScript Support

Both packages include TypeScript definitions. Get full type inference for your tools:

```tsx
import { usePillar } from '@pillar-ai/react';
import type { tools } from '@/lib/pillar/tools';

function ToolHandlers() {
  const { onTask } = usePillar<typeof tools>();

  useEffect(() => {
    // TypeScript knows the data shape for each tool
    onTask('invite_member', (data) => {
      console.log(data.email); // Typed!
    });
  }, [onTask]);

  return null;
}
```

## Complete Example

Here's a minimal complete setup:

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

// components/PillarToolHandlers.tsx
'use client';

import { usePillar } from '@pillar-ai/react';
import { useRouter } from 'next/navigation';
import { useEffect } from 'react';

export function PillarToolHandlers() {
  const { pillar } = usePillar();
  const router = useRouter();

  useEffect(() => {
    if (!pillar) return;
    const handlers = [
      pillar.onTask('navigate', ({ path }) => {
        router.push(path);
        return { success: true };
      }),
    ];
    return () => handlers.forEach(unsub => unsub?.());
  }, [pillar, router]);

  return null;
}

// app/layout.tsx
import { PillarSDKProvider } from '@/providers/PillarSDKProvider';
import { PillarToolHandlers } from '@/components/PillarToolHandlers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <PillarSDKProvider>
          <PillarToolHandlers />
          {children}
        </PillarSDKProvider>
      </body>
    </html>
  );
}
```

## Tool Decomposition

When an API or feature has many modes, don't model it as one tool with a large schema and conditional fields. Split it into smaller tools that each do one thing.

### Why

Your `dataSchema` is converted directly into the AI tool's parameter schema. Large schemas with optional/conditional fields cause:
- The AI guessing at fields it doesn't need
- Ambiguous intent matching (which mode did the user mean?)
- Harder debugging when a tool call fails

Smaller schemas mean the AI fills in fewer fields, picks the right tool more often, and errors are easier to trace.

### Pattern

```tsx
// Before: one tool with an operation switch
manage_report: {
  description: 'Create, schedule, or export a report',
  type: 'trigger_tool',
  dataSchema: {
    type: 'object',
    properties: {
      operation: { type: 'string', enum: ['create', 'schedule', 'export'] },
      name: { type: 'string' },
      format: { type: 'string', enum: ['pdf', 'csv'] },
      schedule: { type: 'string' },
      filters: { type: 'object' },
    },
  },
}

// After: three tools, each with only the fields it needs
create_report: {
  description: 'Create a new report with filters',
  type: 'trigger_tool',
  dataSchema: {
    type: 'object',
    properties: {
      name: { type: 'string', description: 'Report name' },
      filters: { type: 'object', description: 'Filter criteria' },
    },
    required: ['name'],
  },
}

schedule_report: {
  description: 'Set a recurring schedule for an existing report',
  type: 'trigger_tool',
  dataSchema: {
    type: 'object',
    properties: {
      reportId: { type: 'string' },
      schedule: { type: 'string', description: 'Cron expression or preset like "daily", "weekly"' },
    },
    required: ['reportId', 'schedule'],
  },
}

export_report: {
  description: 'Export a report to PDF or CSV',
  type: 'trigger_tool',
  dataSchema: {
    type: 'object',
    properties: {
      reportId: { type: 'string' },
      format: { type: 'string', enum: ['pdf', 'csv'] },
    },
    required: ['reportId', 'format'],
  },
}
```

The handlers can share internal logic. The point is that the AI sees three clearly scoped tools instead of one ambiguous one.

### When to Split

Split when a tool has:
- An "operation" or "mode" field that changes which other fields are required
- More than 5-6 properties in the schema
- Conditional required fields (field X is required only when field Y is "foo")
- Multiple distinct user intents mapping to one tool name

Keep it as one tool when:
- All fields are always relevant regardless of input
- The tool is simple (1-3 properties)
- Splitting would create nearly identical tools

## How Tools Work

When the agent finds your tools via search, they are registered as native tools in the LLM's tool-calling API. This means:

1. Your tool's `description` becomes the tool description
2. Your tool's `dataSchema` becomes the tool's `parameters` JSON Schema
3. The AI calls the tool by name with structured arguments matching your schema
4. The SDK executes your handler with those arguments and returns the result

This is why `dataSchema` quality matters so much. The schema _is_ the tool definition the model sees.

```
Client defines tool            Server registers tool           AI calls tool
+------------------+          +---------------------+         +-----------------+
| name: invite_user|   sync   | name: invite_user   |  call   | invite_user({   |
| description: ... | -------> | description: ...    | ------> |   email: "...", |
| dataSchema: {    |          | parameters: {       |         |   role: "admin" |
|   email, role    |          |   email, role       |         | })              |
| }                |          | }                   |         |                 |
+------------------+          +---------------------+         +-----------------+
```

Tools with `returns: true` (or `type: 'query'`) send the handler's return value back to the agent for further reasoning.

## Schema Compatibility

Pillar routes tool calls to multiple LLM providers (OpenAI, Anthropic, Google Gemini via OpenRouter). Gemini validates schemas more strictly than other providers and will reject tool calls with a 400 error if the schema is invalid. Follow these rules in every `dataSchema`:

1. **Arrays need `items`** - `{ type: 'array' }` is invalid. Use `{ type: 'array', items: { type: 'string' } }` or similar.
2. **`type` must be a single string** - `type: ['string', 'null']` is invalid. Use `type: 'string'` and note nullability in the description.
3. **Objects need `properties`** - `{ type: 'object' }` without properties is invalid. Add at least one property, or use `type: 'string'` for freeform data.
4. **`required` must match `properties`** - Every entry in `required` must be a key in `properties`.

See `rules/schema-compatibility.md` for the full rules, examples, and a review checklist.

## Agent Guidance

Agent Guidance is custom instructions injected into the AI agent's prompt at runtime. Use it to tell the agent how to choose between tools, describe multi-step workflows, and provide domain knowledge.

### Two Ways to Configure

**1. Admin dashboard** (no deploy required):

1. Go to the admin dashboard
2. Navigate to **Configure** > **AI Assistant**
3. Enter instructions in the **Agent Guidance** textarea
4. Save changes

**2. Code sync via `AGENT_GUIDANCE.md`** (deployed with your tools):

Place an `AGENT_GUIDANCE.md` file in the directory you pass to `--scan`. The CLI picks it up automatically -- no JS export or barrel file needed:

```markdown
<!-- src/tools/AGENT_GUIDANCE.md -->
PREFER API TOOLS OVER NAVIGATION:
- When both an API tool and a navigation tool can accomplish a task, prefer the API tool
- API tools execute instantly; navigation requires user to complete forms manually

ORDER FULFILLMENT WORKFLOW:
When a user asks to process an order:
1. Use get_order to fetch order details
2. Use validate_inventory to check stock
3. Use create_shipment to generate shipping label
4. Use notify_customer to send confirmation
```

Then sync:

```bash
PILLAR_SLUG=your-product PILLAR_SECRET=xxx npx pillar-sync --scan ./src/tools
```

The scanner reads `AGENT_GUIDANCE.md` from the root of the scan directory and includes it in the manifest. This keeps your guidance in version control alongside the tool files it references.

### Tips for Writing Guidance

1. Tell the agent what to prefer and when, referencing exact tool names
2. Describe multi-step workflows as numbered sequences
3. Keep it focused on common requests (not an exhaustive manual)
4. Update as you add or rename tools

## Parameter Examples (Advanced)

For complex schemas, you can provide concrete examples of valid parameter objects. These are stored alongside the tool and shown to the AI when it needs to fill in the schema.

```tsx
create_report: {
  description: 'Create a new report with filters',
  type: 'trigger_tool',
  dataSchema: {
    type: 'object',
    properties: {
      name: { type: 'string' },
      filters: { type: 'object' },
    },
    required: ['name'],
  },
  parameterExamples: [
    {
      description: 'Monthly sales report filtered by region',
      parameters: {
        name: 'Monthly Sales - West',
        filters: { region: 'west', period: 'last_30_days' },
      },
    },
  ],
}
```

## Learn More

- [Pillar SDK Documentation](https://trypillar.com/docs)
- [Tools Guide](https://trypillar.com/docs/guides/tools)
- [Context API](https://trypillar.com/docs/guides/context)
- [Custom Cards](https://trypillar.com/docs/guides/custom-cards)
