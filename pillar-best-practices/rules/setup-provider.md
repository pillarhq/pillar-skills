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
      <PillarProvider helpCenter="...">
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
    <PillarProvider helpCenter="your-help-center-slug">
      <YourApp />
    </PillarProvider>
  );
}
```

## Required Props

| Prop | Type | Description |
|------|------|-------------|
| `helpCenter` | `string` | Your help center slug from the Pillar dashboard |

## Optional Props

| Prop | Type | Description |
|------|------|-------------|
| `config` | `object` | Configuration for panel, theme, triggers |
| `onReady` | `() => void` | Callback when SDK is initialized |
| `onError` | `(error: Error) => void` | Callback for initialization errors |
| `cards` | `object` | Custom card components for inline_ui actions |

## Environment Variables

Store your help center slug in an environment variable:

```bash
# .env.local
NEXT_PUBLIC_PILLAR_HELP_CENTER=your-help-center-slug
```

```tsx
<PillarProvider helpCenter={process.env.NEXT_PUBLIC_PILLAR_HELP_CENTER!}>
```
