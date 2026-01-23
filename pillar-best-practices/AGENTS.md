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
    <PillarProvider helpCenter="your-help-center-slug">
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
    <PillarProvider helpCenter={process.env.NEXT_PUBLIC_PILLAR_HELP_CENTER!}>
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
  helpCenter="your-help-center"
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
  helpCenter="..."
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
NEXT_PUBLIC_PILLAR_HELP_CENTER=your-help-center-slug
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
    <PillarProvider helpCenter={process.env.NEXT_PUBLIC_PILLAR_HELP_CENTER!}>
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

## Learn More

- [Pillar SDK Documentation](https://help.trypillar.com/docs)
- [Actions Guide](https://help.trypillar.com/docs/guides/actions)
- [Context API](https://help.trypillar.com/docs/guides/context)
- [Custom Cards](https://help.trypillar.com/docs/guides/custom-cards)
