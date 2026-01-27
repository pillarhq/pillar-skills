---
name: pillar-best-practices
description: Best practices for integrating the Pillar SDK into React and Next.js applications. Use when adding an AI assistant panel, help center widget, or embedded product assistant.
---

# Pillar SDK Best Practices

Best practices for integrating the Pillar SDK into React and Next.js applications. Use when adding an AI assistant panel, help center widget, or embedded product assistant.

## When to Apply

Reference these guidelines when:

- Adding Pillar SDK to a React or Next.js project
- Setting up `PillarProvider` in your app
- Defining actions for the AI assistant to suggest
- Creating action handlers to execute user requests

## Essential Rules

| Priority | Rule | Description |
|----------|------|-------------|
| CRITICAL | `setup-provider` | Always wrap your app with PillarProvider |
| CRITICAL | `setup-nextjs` | Next.js App Router requires a 'use client' wrapper |
| HIGH | `action-descriptions` | Write specific, AI-matchable descriptions |
| HIGH | `action-handlers` | Use centralized handlers with proper cleanup |

## Quick Reference

### 1. Provider Setup (CRITICAL)

Always wrap your app with `PillarProvider`:

```tsx
import { PillarProvider } from '@pillar-ai/react';

<PillarProvider productKey="your-product-key">
  {children}
</PillarProvider>
```

### 2. Next.js App Router (CRITICAL)

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

### 3. Action Descriptions (HIGH)

Write specific descriptions the AI can match:

```tsx
// Good - specific and includes context
description: 'Navigate to billing settings. Suggest when user asks about payments, invoices, or subscription.'

// Bad - too generic
description: 'Go to billing'
```

Add example phrases:

```tsx
examples: [
  'how do I update my payment method?',
  'where can I see my invoices?',
  'change my subscription',
]
```

### 4. Action Handlers (HIGH)

Use a centralized handler component with cleanup:

```tsx
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
```

## How to Use

Read individual rule files for detailed explanations and code examples:

```
rules/setup-provider.md
rules/setup-nextjs.md
rules/action-descriptions.md
rules/action-handlers.md
```

For the complete guide with all patterns expanded: `AGENTS.md`
