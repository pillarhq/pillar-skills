# tool-definition

Every WebMCP tool requires four fields: `name`, `description`, `inputSchema`, and `execute`. Missing or malformed fields will cause the tool to fail silently or be ignored by agents.

## Bad

```js
// Missing inputSchema — agent can't know what params to pass
navigator.modelContext.registerTool({
  name: "add-item",
  description: "Add an item",
  execute: ({ name }) => {
    addItem(name);
    return { content: [{ type: "text", text: "Done" }] };
  }
});

// Missing description — agent can't decide when to use it
navigator.modelContext.registerTool({
  name: "add-item",
  inputSchema: {
    type: "object",
    properties: { name: { type: "string" } }
  },
  execute: ({ name }) => {
    addItem(name);
    return { content: [{ type: "text", text: "Done" }] };
  }
});
```

## Good

```js
navigator.modelContext.registerTool({
  name: "add-item",
  description: "Add an item to the list by name",
  inputSchema: {
    type: "object",
    properties: {
      name: { type: "string", description: "Name of the item to add" }
    },
    required: ["name"]
  },
  execute: async ({ name }) => {
    addItem(name);
    refreshUI();
    return {
      content: [{ type: "text", text: `Added "${name}" to the list.` }]
    };
  }
});
```

## Field Requirements

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | `string` | Yes | Kebab-case. Unique per page. |
| `description` | `string` | Yes | Natural language. Agent reads this to decide relevance. |
| `inputSchema` | `object` | Yes | JSON Schema. Every property needs a `description`. |
| `execute` | `function` | Yes | `(params, agent) => ToolResponse`. Can be async. |

## Response Format

Always return a structured response:

```js
return {
  content: [
    { type: "text", text: "Human-readable result" }
  ]
};
```

Don't return raw strings or undefined. The agent needs the `content` array to process the result.

## Feature Detection

Always guard registration:

```js
if ("modelContext" in navigator) {
  navigator.modelContext.registerTool({...});
}
```

## Bulk Registration

Use `provideContext` to replace all tools at once (useful for SPA route changes):

```js
navigator.modelContext.provideContext({
  tools: [toolA, toolB, toolC]
});
```

Use `registerTool`/`unregisterTool` for incremental changes.
