# setup-nextjs

When using Next.js App Router, `PillarProvider` must be wrapped in a client component. This is because it uses React context, which requires client-side rendering.

## Bad

```tsx
// app/layout.tsx
// ERROR: PillarProvider uses React context which requires 'use client'
import { PillarProvider } from '@pillar-ai/react';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <PillarProvider productKey="...">
          {children}
        </PillarProvider>
      </body>
    </html>
  );
}
```

This will cause an error because Server Components cannot use React context.

## Good

Create a separate client wrapper component:

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

Then use it in your layout:

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

## Why This Works

- The `'use client'` directive marks `PillarSDKProvider` as a Client Component
- The root layout remains a Server Component (better for performance)
- Children are passed through and can still be Server Components
- Environment variables with `NEXT_PUBLIC_` prefix are available on the client

## File Structure

```
app/
├── layout.tsx          # Server Component (imports PillarSDKProvider)
├── page.tsx
└── ...
providers/
└── PillarSDKProvider.tsx  # Client Component with 'use client'
```
