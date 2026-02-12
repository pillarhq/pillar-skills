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

### 5. Decompose Large Actions

If one action would need a huge schema or many conditional modes, split it into smaller, focused actions. Each action should do one thing with a tight schema the AI can fill reliably.

```tsx
// Too broad - one action trying to handle everything
manage_user: {
  description: 'Manage users in the organization',
  type: 'trigger_action',
  dataSchema: {
    type: 'object',
    properties: {
      operation: { type: 'string', enum: ['invite', 'remove', 'change_role'] },
      email: { type: 'string' },
      role: { type: 'string' },
      userId: { type: 'string' },
    },
  },
}

// Better - one action per operation, each with only the fields it needs
invite_user: {
  description: 'Invite a new user to the organization by email',
  type: 'trigger_action',
  dataSchema: {
    type: 'object',
    properties: {
      email: { type: 'string', description: 'Email address to invite' },
      role: { type: 'string', enum: ['admin', 'member', 'viewer'] },
    },
    required: ['email'],
  },
}

remove_user: {
  description: 'Remove a user from the organization',
  type: 'trigger_action',
  dataSchema: {
    type: 'object',
    properties: {
      userId: { type: 'string', description: 'ID of the user to remove' },
    },
    required: ['userId'],
  },
}
```

Why this matters: your `dataSchema` becomes the tool's parameter schema in the AI's tool-calling API. Smaller schemas mean fewer required fields, less room for the AI to guess wrong, and clearer intent matching.

Schemas must also follow cross-model formatting rules. Pillar routes to multiple LLM providers, and Gemini rejects schemas that use array type unions (`type: ['string', 'null']`) or arrays without `items`. See `rules/schema-compatibility.md` for the full list.

### Design for Verification Loops

For actions that create or modify resources via API, add a companion query action that lets the agent validate first. This prevents the agent from committing broken configurations it can't recover from.

The pattern:

1. A `test_*` or `preview_*` query action that validates the input and returns sample data
2. A `create_*` action with `returns: true` that commits the change
3. Agent guidance that tells the AI to always test before creating

```tsx
// Query action: agent validates the data first
test_report_query: {
  description: 'Test a report query and return sample results',
  type: 'query',
  guidance: 'Use this before create_report to verify the query returns data. If it returns 0 rows, the query is wrong.',
  dataSchema: {
    type: 'object',
    properties: {
      datasource_id: { type: 'number', description: 'Dataset to query' },
      filters: { type: 'object', properties: {} },
    },
    required: ['datasource_id'],
  },
}

// Creation action: only called after query is verified
create_report: {
  description: 'Create a report from a validated query',
  type: 'trigger_action',
  returns: true,
  guidance: 'Always test the query with test_report_query first. Only create after it returns valid data.',
  dataSchema: {
    type: 'object',
    properties: {
      name: { type: 'string' },
      datasource_id: { type: 'number' },
      filters: { type: 'object', properties: {} },
    },
    required: ['name', 'datasource_id'],
  },
}
```

The agent guidance then tells the AI: "Always test queries with test_report_query before calling create_report."

This is the same pattern used in the Superset copilot (`test_chart_query` -> `create_bar_chart`) and the Grafana copilot (`test_datasource_query` -> `create_timeseries_panel`). Without verification, the agent creates broken panels and has no way to detect or fix them.

### 6. Use the `guidance` Field

`description` and `guidance` serve different purposes:

- **`description`** is for intent matching -- it helps the AI find the right action when the user asks something. Keep it short and matchable.
- **`guidance`** is for usage instructions -- it tells the AI *how* to use the action correctly once it's been selected. Put caveats, chaining requirements, and pitfalls here.

```tsx
// Bad - everything crammed into description
create_chart: {
  description: 'Create a chart on a dashboard. Always test the query first with test_query. Returns chart_id. Use dashboards array to link. Pie charts use singular metric not array.',
}

// Good - description for matching, guidance for instructions
create_chart: {
  description: 'Create a chart on a dashboard',
  guidance:
    'Always test the query with test_datasource_query first. ' +
    'Returns chart_id which can be used with add_chart_to_dashboard. ' +
    'Pie charts use a singular metric field, not the metrics array.',
}
```

Use `guidance` for:
- Chaining requirements ("call X before Y")
- What the return value contains and how to use it
- Common pitfalls ("field Z must be in format ABC")
- When to prefer this action over a similar one

## Template

```tsx
{
  description: '[Action verb] [what it does]. Suggest when user asks about [related topics].',
  guidance: '[When to use, prerequisites, pitfalls, what it returns]',
  examples: [
    'natural phrase 1',
    'natural phrase 2',
    'question form',
  ],
  type: 'navigate' | 'trigger_action' | 'query' | 'inline_ui',
}
```
