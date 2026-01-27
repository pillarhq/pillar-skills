# setup-provider

Always wrap your application with `PillarProvider` at the root level. Without it, the SDK won't initialize and no Pillar features will work.

## Bad

```tsx
// Missing PillarProvider - SDK won't work
function App() {
  return (
    <div>
      <YourApp />
    </div>
  );
}
```

```tsx
// Provider too deep in the tree - components outside won't have access
function App() {
  return (
    <div>
      <Header />
      <PillarProvider productKey="...">
        <MainContent />
      </PillarProvider>
    </div>
  );
}
```

## Good

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

## Required Props

| Prop | Type | Description |
|------|------|-------------|
| `productKey` | `string` | Your product key from the Pillar app |

## Optional Props

| Prop | Type | Description |
|------|------|-------------|
| `config` | `object` | Configuration for panel, theme, triggers |
| `onReady` | `() => void` | Callback when SDK is initialized |
| `onError` | `(error: Error) => void` | Callback for initialization errors |
| `cards` | `object` | Custom card components for inline_ui actions |

## Environment Variables

Store your product key in an environment variable:

```bash
# .env.local
NEXT_PUBLIC_PILLAR_PRODUCT_KEY=your-product-key
```

```tsx
<PillarProvider productKey={process.env.NEXT_PUBLIC_PILLAR_PRODUCT_KEY!}>
```
