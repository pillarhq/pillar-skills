# Pillar SDK Complete Reference

This is the complete reference for integrating the Pillar SDK. It covers installation, configuration, actions, handlers, and advanced features.

## Overview

Pillar SDK provides an AI-powered assistant panel for your application. Users can ask questions, and the AI can suggest actions that execute directly in your app.

**What you'll build:**
- A slide-out assistant panel with AI chat
- A sidebar trigger (or custom button) that opens the panel
- Actions the AI can suggest to users

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

## Defining Actions

Actions are things users can do in your app that the AI can suggest.

### Basic Action Definition

```tsx
// lib/pillar/actions.ts
import { defineActions } from '@pillar-ai/sdk';

export const actions = defineActions({
  open_settings: {
    description: 'Navigate to the settings page',
    type: 'navigate',
    path: '/settings',
    autoRun: true,
  },

  invite_member: {
    description: 'Open the invite team member modal',
    examples: [
      'invite someone to my team',
      'add a new user',
      'how do I invite people?',
    ],
    type: 'trigger_action',
  },
});
```

### Action Types

| Type | Description | Use Case |
|------|-------------|----------|
| `navigate` | Navigate to a page in your app | Settings, dashboard, detail pages |
| `trigger_action` | Run custom logic | Open modals, start wizards, toggle features |
| `query` | Fetch data and return to the agent | List items, get details, lookups (auto-sets `returns: true`) |
| `inline_ui` | Show interactive UI in chat | Forms, confirmations, previews |
| `external_link` | Open URL in new tab | Documentation, external resources |
| `copy_text` | Copy text to clipboard | API keys, code snippets |

### Writing Good Descriptions

The AI matches user queries to your action descriptions. Be specific:

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
  type: 'trigger_action',
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

## Action Handlers

### Centralized Handler Pattern

Create a dedicated component for all your handlers:

```tsx
// components/PillarActionHandlers.tsx
'use client';

import { usePillar } from '@pillar-ai/react';
import { useRouter } from 'next/navigation';
import { useEffect } from 'react';

export function PillarActionHandlers() {
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
  <PillarActionHandlers />
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

Pillar provides fallback handlers for common action types:

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
| `userRole` | `string` | User's role for action filtering |
| `userState` | `string` | User's current state (onboarding, trial, active) |
| `errorState` | `object` | Current error with code and message |
| `custom` | `object` | Any additional context data |

### Action Filtering with Context

Use `requiredContext` to control which actions the AI suggests:

```tsx
delete_user: {
  description: 'Delete a user from the organization',
  type: 'trigger_action',
  requiredContext: { userRole: 'admin' },
}
```

## Plans (Automatic)

The AI automatically creates multi-step plans when a user's request requires multiple actions. Your handlers work the same way - no special code needed.

**Example:** When a user asks "Help me set up my workspace", the AI might create a plan:
1. Create a new project
2. Add your first knowledge source
3. Configure the assistant
4. Invite your team

Each step executes your registered action handlers automatically.

## Custom Cards (Advanced)

For `inline_ui` actions, you can render custom React components in the chat:

```tsx
// Define the action
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

Both packages include TypeScript definitions. Get full type inference for your actions:

```tsx
import { usePillar } from '@pillar-ai/react';
import type { actions } from '@/lib/pillar/actions';

function ActionHandlers() {
  const { onTask } = usePillar<typeof actions>();

  useEffect(() => {
    // TypeScript knows the data shape for each action
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

// components/PillarActionHandlers.tsx
'use client';

import { usePillar } from '@pillar-ai/react';
import { useRouter } from 'next/navigation';
import { useEffect } from 'react';

export function PillarActionHandlers() {
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
import { PillarActionHandlers } from '@/components/PillarActionHandlers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <PillarSDKProvider>
          <PillarActionHandlers />
          {children}
        </PillarSDKProvider>
      </body>
    </html>
  );
}
```

## Action Decomposition

When an API or feature has many modes, don't model it as one action with a large schema and conditional fields. Split it into smaller actions that each do one thing.

### Why

Your `dataSchema` is converted directly into the AI tool's parameter schema. Large schemas with optional/conditional fields cause:
- The AI guessing at fields it doesn't need
- Ambiguous intent matching (which mode did the user mean?)
- Harder debugging when a tool call fails

Smaller schemas mean the AI fills in fewer fields, picks the right tool more often, and errors are easier to trace.

### Pattern

```tsx
// Before: one action with an operation switch
manage_report: {
  description: 'Create, schedule, or export a report',
  type: 'trigger_action',
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

// After: three actions, each with only the fields it needs
create_report: {
  description: 'Create a new report with filters',
  type: 'trigger_action',
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
  type: 'trigger_action',
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
  type: 'trigger_action',
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

Split when an action has:
- An "operation" or "mode" field that changes which other fields are required
- More than 5-6 properties in the schema
- Conditional required fields (field X is required only when field Y is "foo")
- Multiple distinct user intents mapping to one action name

Keep it as one action when:
- All fields are always relevant regardless of input
- The action is simple (1-3 properties)
- Splitting would create nearly identical actions

### Verification Loop Pattern

For actions that create or modify resources via API, add a companion query action that validates input before committing. Without this, the agent creates broken configurations and has no way to detect or fix them.

The pattern:

1. A `test_*` or `preview_*` query action that validates the input and returns sample data
2. A `create_*` action with `returns: true` that commits the change
3. Agent guidance that tells the AI to always test before creating

```tsx
// Step 1: Query action -- agent validates the data first
test_chart_query: {
  description: 'Test a chart query and return sample results',
  type: 'query',
  guidance: 'Use this before any create_*_chart action to verify the query returns data.',
  dataSchema: {
    type: 'object',
    properties: {
      datasource: { type: 'object', properties: { id: { type: 'number' }, type: { type: 'string' } }, required: ['id', 'type'] },
      queries: { type: 'array', items: { type: 'object', properties: { columns: { type: 'array', items: { type: 'object' } }, metrics: { type: 'array', items: { type: 'object' } } }, required: ['columns', 'metrics'] } },
    },
    required: ['datasource', 'queries'],
  },
  autoRun: true,
  autoComplete: true,
}

// Step 2: Creation action -- only called after query is verified
create_bar_chart: {
  description: 'Create a bar chart via API with full configuration',
  type: 'trigger_action',
  returns: true,
  guidance: 'Always test the query with test_chart_query first. Only create after it returns valid data.',
  dataSchema: { /* tight schema for bar chart params */ },
  autoRun: true,
  autoComplete: true,
}
```

This is the pattern used in the Superset copilot (`test_chart_query` -> `create_bar_chart`) and Grafana copilot (`test_datasource_query` -> `create_timeseries_panel`). The agent guidance then encodes the workflow:

```tsx
export const agentGuidance = `
VERIFY BEFORE COMMITTING:
When creating visualizations, always test the query against the data source first.
If it returns data, proceed with creating the visualization.
If it returns no data or errors, adjust the query and re-test.
Never create a visualization with an unverified query.
`;
```

### Reusable Schema Components

When multiple actions share the same schema shapes (metrics, filters, columns), define them as shared objects and reference them across actions. This keeps schemas consistent and reduces duplication.

```tsx
// Shared schema components
const metricSchema = {
  type: 'object',
  description: 'Metric object for aggregation',
  properties: {
    expressionType: { type: 'string', enum: ['SQL', 'SIMPLE'] },
    sqlExpression: { type: 'string', description: 'SQL expression like "SUM(amount)"' },
    label: { type: 'string', description: 'Display label for the metric' },
  },
  required: ['expressionType', 'label'],
};

const filterSchema = {
  type: 'object',
  properties: {
    column: { type: 'string', description: 'Column name to filter' },
    operator: { type: 'string', description: 'Comparison operator' },
    value: { type: 'string', description: 'Filter value' },
  },
  required: ['column', 'operator', 'value'],
};

// Used in multiple actions
create_bar_chart: {
  dataSchema: {
    type: 'object',
    properties: {
      metrics: { type: 'array', items: metricSchema },
      filters: { type: 'array', items: filterSchema },
      // ...
    },
  },
}

create_line_chart: {
  dataSchema: {
    type: 'object',
    properties: {
      metrics: { type: 'array', items: metricSchema },  // same schema
      filters: { type: 'array', items: filterSchema },  // same schema
      // ...
    },
  },
}
```

This pattern is used in the Superset copilot where `metricSchema`, `adhocFilterSchema`, and `xAxisColumnSchema` are shared across all chart creation actions.

## How Actions Become Tools

When the agent finds your actions via search, they are registered as native tools in the LLM's tool-calling API. This means:

1. Your action's `description` becomes the tool description
2. Your action's `dataSchema` becomes the tool's `parameters` JSON Schema
3. The AI calls the tool by name with structured arguments matching your schema
4. The SDK executes your handler with those arguments and returns the result

This is why `dataSchema` quality matters so much. The schema _is_ the tool definition the model sees.

```
Client defines action          Server converts to tool         AI calls tool
+------------------+          +---------------------+         +-----------------+
| name: invite_user|   sync   | name: invite_user   |  call   | invite_user({   |
| description: ... | -------> | description: ...    | ------> |   email: "...", |
| dataSchema: {    |          | parameters: {       |         |   role: "admin" |
|   email, role    |          |   email, role       |         | })              |
| }                |          | }                   |         |                 |
+------------------+          +---------------------+         +-----------------+
```

Actions with `returns: true` (or `type: 'query'`) send the handler's return value back to the agent for further reasoning.

## Schema Compatibility

Pillar routes tool calls to multiple LLM providers (OpenAI, Anthropic, Google Gemini via OpenRouter). Gemini validates schemas more strictly than other providers and will reject tool calls with a 400 error if the schema is invalid. Follow these rules in every `dataSchema`:

1. **Arrays need `items`** - `{ type: 'array' }` is invalid. Use `{ type: 'array', items: { type: 'string' } }` or similar.
2. **`type` must be a single string** - `type: ['string', 'null']` is invalid. Use `type: 'string'` and note nullability in the description.
3. **Objects need `properties`** - `{ type: 'object' }` without properties is invalid. Add at least one property, or use `type: 'string'` for freeform data.
4. **`required` must match `properties`** - Every entry in `required` must be a key in `properties`.

See `rules/schema-compatibility.md` for the full rules, examples, and a review checklist.

## Agent Guidance

Agent Guidance is custom instructions injected into the AI agent's prompt at runtime. Use it to tell the agent how to choose between actions, describe multi-step workflows, and provide domain knowledge.

### Two Ways to Configure

**1. Admin dashboard** (no deploy required):

1. Go to the admin dashboard
2. Navigate to **Configure** > **AI Assistant**
3. Enter instructions in the **Agent Guidance** textarea
4. Save changes

**2. Code sync via `agentGuidance` export** (deployed with your actions):

Export an `agentGuidance` string alongside your actions. `pillar-sync` sends it to the backend automatically:

```tsx
// lib/pillar/actions/index.ts
export const actions = { /* ... */ };

export const agentGuidance = `
PREFER API OVER NAVIGATION:
- When you can accomplish a task via API, prefer that over navigating to a form
- API actions execute instantly and return structured results
- Only navigate when no API alternative exists

ORDER FULFILLMENT:
When a user asks to process an order:
1. Look up the order details first
2. Verify inventory is available
3. Generate the shipment
4. Notify the customer when complete
`;
```

Then sync:

```bash
PILLAR_SLUG=your-product PILLAR_SECRET=xxx npx pillar-sync --actions ./lib/pillar/actions/index.ts
```

The code-sync path is useful when your guidance references specific action names and should stay in version control alongside those actions.

### Tips for Writing Guidance

1. Tell the agent what to prefer and when, referencing exact action names
2. Describe multi-step workflows as numbered sequences
3. Include verification steps where relevant ("test first, then create")
4. Keep it focused on common requests (not an exhaustive manual)
5. Update as you add or rename actions

### Verification Workflow Example

For copilots that create resources via API, agent guidance should encode the verify-then-commit loop:

```tsx
export const agentGuidance = `
PREFER API OVER NAVIGATION:
- API actions execute instantly and return structured results
- Navigation requires the user to fill out forms manually
- Only navigate when no API alternative exists

CREATING VISUALIZATIONS:
When building charts or dashboards:
1. First discover what data sources are available
2. Inspect the data to understand columns and types
3. Test the query to confirm it returns data before creating anything
4. If the test fails, adjust the query and re-test
5. Only create the visualization after validation succeeds
6. Show the user the result when done
`;
```

This ensures the agent validates before committing. The same pattern applies to any create workflow: test first, then commit.

## Parameter Examples (Advanced)

For complex schemas, you can provide concrete examples of valid parameter objects. These are stored alongside the action and shown to the AI when it needs to fill in the schema.

```tsx
create_report: {
  description: 'Create a new report with filters',
  type: 'trigger_action',
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
- [Actions Guide](https://trypillar.com/docs/guides/actions)
- [Context API](https://trypillar.com/docs/guides/context)
- [Custom Cards](https://trypillar.com/docs/guides/custom-cards)
