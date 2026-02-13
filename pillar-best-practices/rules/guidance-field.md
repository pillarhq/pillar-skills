# guidance-field

The `guidance` field provides agent-facing instructions that help the LLM choose the right tool and chain tools correctly. It's appended to the tool description in the LLM's tool list during agentic reasoning.

## When to Use guidance

Add `guidance` when:
- Two tools have similar descriptions and the LLM might pick the wrong one
- A tool requires output from another tool (prerequisite chain)
- A tool should NOT be used in certain contexts
- A tool is the entry point or exit point of a multi-tool workflow

## guidance vs description

| Field | Audience | Purpose | Example |
|-------|----------|---------|---------|
| `description` | User + AI search | Semantic matching to user queries | "Get datasources available for visualizations" |
| `guidance` | Agent only | Disambiguation, prerequisites, chaining | "Call BEFORE creating panels. If zero results, ask user to create one." |

## Templates

### Disambiguation
```tsx
navigate_to_dashboards: {
  description: 'Browse existing dashboards.',
  guidance: 'Use ONLY for browsing existing dashboards. Do NOT use when the user wants to create a new dashboard -- use create_dashboard instead.',
}
```

### Prerequisites
```tsx
create_timeseries_panel: {
  description: 'Add a timeseries panel to a dashboard.',
  guidance: 'Requires dashboard_uid from create_dashboard. MUST call test_datasource_query first to validate the query.',
}
```

### Entry Point
```tsx
get_available_datasources: {
  description: 'Get datasources available for creating visualizations.',
  guidance: 'Call BEFORE creating dashboards or panels to discover datasources. If zero results, ask the user to create one. If multiple, ask which to use.',
}
```

### Exit Point
```tsx
navigate_to_dashboard: {
  description: 'Open a dashboard in the browser.',
  guidance: 'Use as the final step after creating a dashboard and adding panels. Shows the user the completed dashboard.',
}
```

## Rules

1. Keep guidance concise -- it's injected into the LLM system prompt, so tokens matter
2. Use imperative voice: "Call BEFORE", "Requires", "Use ONLY when"
3. Reference other tools by name: "Requires dashboard_uid from create_dashboard"
4. Specify negative conditions: "Do NOT use when..."
5. Don't duplicate the description -- guidance adds information, not restates it

## Syncing guidance to the backend

When using `defineTool()`, the `guidance` field is extracted automatically by the AST scanner:

```bash
npx pillar-sync --scan ./src/actions
```

The scanner reads `guidance` as a static string from the `defineTool()` call and includes it in the manifest sent to the backend. No separate sync file or manual configuration needed.

On the backend, the `guidance` value is stored on the `Action` model and concatenated with the `description` when building the LLM's tool list, so the agent sees both at tool-selection time.
