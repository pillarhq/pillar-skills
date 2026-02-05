# action-handlers

Create a centralized handler component with proper cleanup. Handlers must be registered when the SDK is ready and cleaned up on unmount to prevent memory leaks.

## Bad

```tsx
// Missing cleanup - will cause memory leaks
function ActionHandlers() {
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
  <PillarActionHandlers />
  {children}
</PillarSDKProvider>
```

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

If the action returns data to the agent, make sure it can be JSON serialized.

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
