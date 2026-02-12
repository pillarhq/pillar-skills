# WebMCP Complete Reference

This is the complete reference for implementing WebMCP tools — JavaScript functions that AI agents, browser assistants, and assistive technologies can discover and call on your web page.

## Overview

WebMCP is a W3C proposal (from Microsoft and Google) that adds `navigator.modelContext` to the browser. Web developers register tools with a name, description, JSON Schema, and an `execute` callback. Any connected agent can discover and call them.

A web page with WebMCP tools is functionally an MCP server that runs in the browser instead of on a backend.

**What you'll build:**
- Tools that agents can call to interact with your app
- Structured responses that agents can reason about
- User confirmation flows for sensitive operations
- Dynamic tool sets that change with your app state

## API Surface

### `navigator.modelContext`

| Method | Purpose |
|--------|---------|
| `registerTool(descriptor)` | Add a single tool |
| `unregisterTool(name)` | Remove a tool by name |
| `provideContext({ tools })` | Replace all tools with a new set |

### Tool Descriptor

```ts
interface ToolDescriptor {
  name: string;           // kebab-case, unique per page
  description: string;    // Natural language — the LLM reads this
  inputSchema: JSONSchema; // Parameters the tool accepts
  execute: (params: object, agent: Agent) => Promise<ToolResponse> | ToolResponse;
}
```

### Agent Object

Passed as the second argument to `execute`. Scoped to the lifetime of that tool call.

| Method | Purpose |
|--------|---------|
| `requestUserInteraction(callback)` | Pause execution for user input. Returns the callback's return value. Can be called multiple times. |

### Tool Response

```ts
interface ToolResponse {
  content: Array<{ type: "text"; text: string }>;
}
```

## Feature Detection

Always check before registering. The API is not in stable browsers yet.

```js
if ("modelContext" in navigator) {
  navigator.modelContext.registerTool({...});
}
```

Until native support ships, the `@mcp-b/global` polyfill adds `navigator.modelContext` to any browser. Feature-detect native first, polyfill second.

## Registering Tools

### Single tool

```js
navigator.modelContext.registerTool({
  name: "add-to-cart",
  description: "Add a product to the shopping cart by product ID",
  inputSchema: {
    type: "object",
    properties: {
      productId: { type: "string", description: "The product ID" },
      quantity: { type: "number", description: "Number of items (1-100)" }
    },
    required: ["productId"]
  },
  execute: async ({ productId, quantity }) => {
    await cartApi.add(productId, quantity ?? 1);
    return {
      content: [{ type: "text", text: `Added ${quantity ?? 1} items to cart` }]
    };
  }
});
```

### Bulk registration

`provideContext` replaces all previously registered tools. Use it for SPAs that swap tool sets on route changes:

```js
navigator.modelContext.provideContext({
  tools: [
    { name: "tool-a", description: "...", inputSchema: {...}, execute: fnA },
    { name: "tool-b", description: "...", inputSchema: {...}, execute: fnB }
  ]
});
```

Each tool name in the list must be unique.

### Unregistering

```js
navigator.modelContext.unregisterTool("add-to-cart");
```

## Writing Descriptions

The `description` field is the most important part of a tool definition. Agents use it to decide whether the tool is relevant to the user's request.

### Be specific about what the tool does

```js
// Good
description: "Filter the product list based on a natural language query. Updates the page to show matching products."

// Bad
description: "Filter products"
```

### Describe what it returns

```js
// Good
description: "Search available flights. Returns an array of {id, airline, price, departure, arrival}. Does not book — use book-flight to complete a booking."

// Bad
description: "Search flights"
```

### Include example invocations for complex tools

```js
description: `Make changes to the current design based on instructions. Examples:
  editDesign("Change the title font to red")
  editDesign("Rotate each background image slightly for asymmetry")`
```

### Describe every parameter

Every property in `inputSchema` should have a `description`:

```js
inputSchema: {
  type: "object",
  properties: {
    query: {
      type: "string",
      description: "Natural language search query (e.g. 'red shoes under $50 in size 9')"
    },
    maxResults: {
      type: "number",
      description: "Maximum results to return. Default 20, max 100."
    }
  }
}
```

## Input Schema Patterns

### Enums

```js
properties: {
  priority: {
    type: "string",
    enum: ["low", "medium", "high"],
    description: "Task priority level"
  }
}
```

### Arrays

```js
properties: {
  tags: {
    type: "array",
    items: { type: "string" },
    description: "Labels to apply"
  }
}
```

Always include `items` when using `type: "array"`.

### Nested objects

```js
properties: {
  address: {
    type: "object",
    properties: {
      street: { type: "string" },
      city: { type: "string" },
      zip: { type: "string" }
    },
    required: ["street", "city"]
  }
}
```

### Required fields

```js
inputSchema: {
  type: "object",
  properties: {
    name: { type: "string" },
    email: { type: "string" }
  },
  required: ["name", "email"]
}
```

Every entry in `required` must be a key in `properties`.

## User Interaction

Use `agent.requestUserInteraction()` when a tool needs user confirmation or input during execution.

### When to use it

- Purchases and payments
- Account deletion or data removal
- Sending messages or emails
- Any irreversible action

### When not to use it

- Read-only queries
- Filtering or searching
- UI state changes that are easily reversible

### Pattern

```js
execute: async ({ productId }, agent) => {
  const confirmed = await agent.requestUserInteraction(async () => {
    return confirm(`Purchase product ${productId}?`);
  });

  if (!confirmed) {
    throw new Error("Purchase cancelled by user");
  }

  await completePurchase(productId);
  return { content: [{ type: "text", text: `Purchased ${productId}` }] };
}
```

`requestUserInteraction` can be called multiple times per tool execution — for example, confirm shipping address, then confirm payment.

## Dynamic Registration (SPAs)

### Only expose relevant tools

Register tools that match the current page state. An "undo" tool should only exist when there are undoable actions. A "checkout" tool should only exist when there's a cart.

### React pattern

```jsx
function DocumentEditor({ doc }) {
  useEffect(() => {
    if (!("modelContext" in navigator)) return;

    navigator.modelContext.registerTool({
      name: "edit-document",
      description: `Edit "${doc.title}". Accepts natural language editing instructions.`,
      inputSchema: {
        type: "object",
        properties: {
          instructions: {
            type: "string",
            description: "What to change (e.g. 'make the heading larger')"
          }
        },
        required: ["instructions"]
      },
      execute: async ({ instructions }) => {
        const result = await applyEdit(doc.id, instructions);
        return { content: [{ type: "text", text: result.summary }] };
      }
    });

    return () => navigator.modelContext.unregisterTool("edit-document");
  }, [doc.id]);
}
```

### Route-level swap

For SPAs with distinct pages, swap the entire tool set on navigation:

```js
function updateToolsForRoute(route) {
  const tools = getToolsForRoute(route);
  navigator.modelContext.provideContext({ tools });
}

// Call on route change
router.on("change", (route) => updateToolsForRoute(route));
```

## Security

### Validate parameters

Agent-provided parameters are untrusted input. Validate like any user input:

```js
execute: async ({ quantity }) => {
  if (typeof quantity !== "number" || quantity < 1 || quantity > 100) {
    throw new Error("quantity must be a number between 1 and 100");
  }
  // proceed
}
```

### Don't leak sensitive data in responses

Tool responses go to the agent. Don't include:
- Auth tokens or API keys
- Internal system IDs the user shouldn't see
- PII that wasn't explicitly requested

```js
// Bad — leaks internal data
return { content: [{ type: "text", text: `Order created. Internal ref: ${internalId}, auth: ${token}` }] };

// Good — only user-facing data
return { content: [{ type: "text", text: `Order #${orderId} created. Estimated delivery: ${eta}` }] };
```

### Confirm destructive actions

Use `agent.requestUserInteraction()` before purchases, deletions, sends, or account changes.

### Match description to behavior

A tool described as "view cart" that triggers a purchase is a misrepresentation vulnerability. Agents trust descriptions to decide what to call. The description must accurately reflect what `execute` does.

### Avoid over-parameterization

Don't request more data than the tool needs. Agents try to fill every parameter from user context. Excessive parameters become a privacy fingerprinting vector:

```js
// Bad — asks for data the tool doesn't need
properties: {
  size: { type: "string" },
  age: { type: "number", description: "For age-appropriate styling" },
  pregnant: { type: "boolean", description: "For maternity options" },
  location: { type: "string", description: "For weather-appropriate suggestions" },
  skinTone: { type: "string", description: "For color matching" },
  previousPurchases: { type: "array", description: "For style consistency" }
}

// Good — only what's needed
properties: {
  size: { type: "string" },
  style: { type: "string", description: "Style preference (e.g. 'formal', 'casual')" }
}
```

## Service Worker Tools

Tools can run in service workers without a visible page. Use for background operations like notifications, data sync, or simple CRUD.

```js
// service-worker.js
self.agent.provideContext({
  tools: [{
    name: "add-todo",
    description: "Add an item to the user's todo list",
    inputSchema: {
      type: "object",
      properties: {
        text: { type: "string", description: "The todo item" },
        priority: { type: "string", enum: ["low", "medium", "high"] }
      },
      required: ["text"]
    },
    execute: async (params, clientInfo) => {
      const list = await getTodoList(clientInfo.sessionId);
      list.add(params.text, params.priority ?? "medium");
      await syncToBackend(list);
      return { content: [{ type: "text", text: `Added: ${params.text}` }] };
    }
  }]
});
```

Service worker tools use `clientInfo.sessionId` to differentiate concurrent sessions.

### When to use service workers vs. page tools

| Use case | Where to register |
|----------|-------------------|
| Needs to update visible UI | Page (`navigator.modelContext`) |
| Background CRUD, notifications | Service worker (`self.agent`) |
| Works with or without the page open | Service worker |
| Context-dependent (current document, cart) | Page |

## Event-Based Alternative

The spec also proposes an event-based pattern for handling tool calls. This is useful when tools are declared in a manifest separate from the implementation:

```js
window.agent.addEventListener("toolcall", async (e) => {
  if (e.name === "add-todo") {
    addTodoItem(e.params.text);
    e.respondWith({
      content: [{ type: "text", text: `Added "${e.params.text}"` }]
    });
  }
});
```

In the hybrid approach, the `toolcall` event fires before `execute`. If the handler calls `e.preventDefault()`, the `execute` callback is skipped.

## Complete Examples

### Read-only tool

```js
navigator.modelContext.registerTool({
  name: "get-order-status",
  description: "Check order status by ID. Returns status, ETA, and tracking URL.",
  inputSchema: {
    type: "object",
    properties: {
      orderId: { type: "string", description: "Order ID (e.g. ORD-12345)" }
    },
    required: ["orderId"]
  },
  execute: async ({ orderId }) => {
    const order = await fetchOrder(orderId);
    return {
      content: [{
        type: "text",
        text: `Order ${orderId}: ${order.status}. ETA: ${order.eta}. Track: ${order.trackingUrl}`
      }]
    };
  }
});
```

### Mutation with confirmation

```js
navigator.modelContext.registerTool({
  name: "delete-account",
  description: "Permanently delete the user's account and all data. Requires user confirmation.",
  inputSchema: {
    type: "object",
    properties: {
      reason: { type: "string", description: "Reason for deletion (optional)" }
    }
  },
  execute: async ({ reason }, agent) => {
    const confirmed = await agent.requestUserInteraction(() =>
      confirm("This will permanently delete your account. Are you sure?")
    );
    if (!confirmed) throw new Error("Account deletion cancelled");

    await deleteAccount({ reason });
    return { content: [{ type: "text", text: "Account deleted." }] };
  }
});
```

### Tool with UI side effects

```js
navigator.modelContext.registerTool({
  name: "filter-templates",
  description: "Filter the template list based on a visual description. Updates the page to show matching templates.",
  inputSchema: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "Natural language filter (e.g. 'spring themed, white background')"
      }
    },
    required: ["query"]
  },
  execute: async ({ query }) => {
    const results = await searchTemplates(query);
    renderTemplateGrid(results);
    return {
      content: [{
        type: "text",
        text: `Showing ${results.length} templates matching "${query}".`
      }]
    };
  }
});
```

### Multi-step workflow with shared context

```js
// Tool 1: search (available always)
navigator.modelContext.registerTool({
  name: "search-products",
  description: "Search products. Returns array of {id, name, price, description, imageUrl}.",
  inputSchema: {
    type: "object",
    properties: {
      query: { type: "string", description: "Search query" },
      size: { type: "string", description: "Size filter (e.g. 'M', '10')" }
    },
    required: ["query"]
  },
  execute: async ({ query, size }) => {
    const products = await searchProducts({ query, size });
    return {
      content: [{
        type: "text",
        text: JSON.stringify(products.map(p => ({ id: p.id, name: p.name, price: p.price })))
      }]
    };
  }
});

// Tool 2: show selected products (available always)
navigator.modelContext.registerTool({
  name: "show-products",
  description: "Display specific products to the user by ID. Updates the page UI.",
  inputSchema: {
    type: "object",
    properties: {
      productIds: {
        type: "array",
        items: { type: "string" },
        description: "Product IDs to display"
      }
    },
    required: ["productIds"]
  },
  execute: async ({ productIds }) => {
    await showProductsInGrid(productIds);
    return {
      content: [{ type: "text", text: `Showing ${productIds.length} products.` }]
    };
  }
});
```

## Spec Status

As of February 2026:

- **W3C proposal**: Incubating in the Web Machine Learning Community Group. Not a standard yet.
- **Chrome**: Early preview behind a flag (Chrome 146). Not in stable.
- **Polyfill**: `@mcp-b/global` adds `navigator.modelContext` to any browser today.
- **API shape**: May change. Wrap registration in a helper so you can update without rewriting all consumers.

## Links

- [W3C WebMCP explainer](https://github.com/webmachinelearning/webmcp)
- [W3C WebMCP API proposal](https://github.com/webmachinelearning/webmcp/blob/main/docs/proposal.md)
- [Security and privacy considerations](https://github.com/webmachinelearning/webmcp/blob/main/docs/security-privacy-considerations.md)
- [Service workers explainer](https://github.com/webmachinelearning/webmcp/blob/main/docs/service-workers.md)
- [MCP-B polyfill](https://github.com/WebMCP-org)
