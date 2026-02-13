# tool-handlers

**Preferred pattern:** Use `pillar.defineTool()` for new code — it co-locates the definition and handler in a single call, preventing drift. See the `defineTool()` section in AGENTS.md.

**Legacy pattern (`onTask`):** Use the centralized handler component below when working with existing codebases that use `defineActions` + `onTask()`. Handlers must be registered when the SDK is ready and cleaned up on unmount to prevent memory leaks.

## Bad

```tsx
// Missing cleanup - will cause memory leaks
function ToolHandlers() {
  const { pillar } = usePillar();

  useEffect(() => {
    if (!pillar) return;
    
    pillar.onTask('navigate', ({ path }) => {
      router.push(path);
    });
    // No cleanup!
  }, [pillar]);

  return null;
}
```

```tsx
// Handlers scattered across multiple components - hard to maintain
function SettingsPage() {
  const { pillar } = usePillar();

  useEffect(() => {
    pillar?.onTask('open_settings', () => { ... });
  }, [pillar]);
}

function BillingPage() {
  const { pillar } = usePillar();

  useEffect(() => {
    pillar?.onTask('view_billing', () => { ... });
  }, [pillar]);
}
```

## Good

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

    // Collect all handler subscriptions
    const handlers = [
      pillar.onTask('navigate', ({ path }) => {
        router.push(path);
        return { success: true };
      }),

      pillar.onTask('open_modal', ({ modalId }) => {
        openModal(modalId);
        return { success: true };
      }),

      pillar.onTask('copy_api_key', () => {
        navigator.clipboard.writeText(apiKey);
        return { success: true, message: 'API key copied!' };
      }),
    ];

    // Cleanup on unmount
    return () => handlers.forEach(unsub => unsub?.());
  }, [pillar, router]);

  return null;
}
```

Add to your layout once:

```tsx
// app/layout.tsx
<PillarSDKProvider>
  <PillarToolHandlers />
  {children}
</PillarSDKProvider>
```

## Tool Definitions with guidance

When defining tools, the `guidance` field provides agent-facing instructions that help the LLM choose and chain tools. Include it alongside the description:

```tsx
const tools = {
  get_available_datasources: {
    description: 'Get datasources available for creating visualizations.',
    guidance: 'Call BEFORE creating dashboards or panels. If zero results, ask user to create one.',
    type: 'trigger_action',
  },
  create_dashboard: {
    description: 'Create a new empty dashboard.',
    guidance: 'First step when building a new dashboard. Returns dashboard_uid needed by all create_*_panel tools.',
    type: 'trigger_action',
    inputSchema: {
      type: 'object',
      properties: {
        title: { type: 'string', description: 'Dashboard title' },
      },
      required: ['title'],
    },
  },
};
```

See `rules/guidance-field.md` for full details on when and how to use guidance.

## Handler Return Values

Always return a result object:

```tsx
// Success
return { success: true };

// Success with user message
return { success: true, message: 'Settings saved!' };

// Failure with error message
return { success: false, message: 'Failed to save. Please try again.' };
```

## Return JSON-Serializable Data

If the tool returns data to the agent, make sure it can be JSON serialized.

```tsx
// Good - plain data
return { success: true, id: result.id, name: result.name };

// Bad - non-serializable values
return { success: true, element: document.body, handler: () => {} };
```

## Error Handling

Wrap handlers in try-catch for graceful error handling:

```tsx
pillar.onTask('export_data', async (data) => {
  try {
    await exportData(data);
    return { success: true, message: 'Export complete!' };
  } catch (error) {
    console.error('Export failed:', error);
    return { success: false, message: 'Export failed. Please try again.' };
  }
});
```

## Built-in Handlers

Pillar provides default handlers for common types. Override them when you need custom behavior:

| Type | Default Behavior | Override When |
|------|------------------|---------------|
| `navigate` | `window.location.href = path` | Using client-side routing (Next.js, React Router) |
| `external_link` | `window.open(url, '_blank')` | Need custom link handling |
| `copy_text` | `navigator.clipboard.writeText(text)` | Need custom feedback |

## Dependency Array

Include all dependencies used in handlers:

```tsx
useEffect(() => {
  if (!pillar) return;

  const handlers = [
    pillar.onTask('navigate', ({ path }) => {
      router.push(path);  // Uses router
      return { success: true };
    }),
  ];

  return () => handlers.forEach(unsub => unsub?.());
}, [pillar, router]);  // Include router in deps
```
