# action-descriptions

Write specific, descriptive action descriptions that the AI can match to user intent. Vague descriptions make it harder for the AI to suggest the right action.

## Bad

```tsx
// Too generic - AI can't distinguish intent
open_settings: {
  description: 'Go to settings',
  type: 'navigate',
  path: '/settings',
}

// Single word - no context
billing: {
  description: 'Billing',
  type: 'navigate',
  path: '/billing',
}
```

## Good

```tsx
// Specific and includes context about when to suggest
open_settings: {
  description: 'Navigate to the settings page. Suggest when user asks about preferences, account settings, or configuration.',
  type: 'navigate',
  path: '/settings',
}

// Includes related topics
view_billing: {
  description: 'Navigate to billing and subscription settings. Suggest when user asks about payments, invoices, pricing, or subscription.',
  type: 'navigate',
  path: '/billing',
}
```

## Add Example Phrases

The `examples` array helps the AI understand different ways users might phrase requests:

```tsx
invite_member: {
  description: 'Open the invite team member modal',
  examples: [
    'invite someone to my team',
    'add a new team member',
    'how do I add users?',
    'share access with someone',
    'invite a colleague',
  ],
  type: 'trigger_action',
}
```

## Best Practices

### 1. Be Specific About the Action

```tsx
// Good
description: 'Open the invite team member modal to add someone to your organization'

// Bad
description: 'Invite'
```

### 2. Include When to Suggest

```tsx
// Good
description: 'Navigate to API settings. Suggest when user asks about API keys, webhooks, or integrations.'

// Bad
description: 'API settings page'
```

### 3. Use Natural Language

```tsx
// Good
description: 'Create a new project with a name and optional description'

// Bad
description: 'POST /projects create'
```

### 4. Cover Related Topics

```tsx
description: 'Export data to CSV or Excel format. Suggest when user asks about downloading, exporting, or backing up their data.'
```

## Template

```tsx
{
  description: '[Action verb] [what it does]. Suggest when user asks about [related topics].',
  examples: [
    'natural phrase 1',
    'natural phrase 2',
    'question form',
  ],
  type: 'navigate' | 'trigger_action' | 'inline_ui',
}
```
