# dynamic-registration

Register and unregister tools as your SPA state changes. Only expose tools that are relevant to the current page state.

## Why

A flat list of every tool on the site overwhelms the agent's context window and increases the chance of the wrong tool being selected. Tools should match what the user can currently do.

## Bad

```js
// Registers everything on page load, never updates
navigator.modelContext.provideContext({
  tools: [addTool, editTool, deleteTool, undoTool, checkoutTool, ...]
});
```

## Good

### Route-level swap

Replace all tools when the view changes:

```js
function onRouteChange(route) {
  navigator.modelContext.provideContext({
    tools: getToolsForRoute(route)
  });
}
```

### Component lifecycle (React)

Register on mount, unregister on unmount:

```jsx
function CheckoutPage({ cart }) {
  useEffect(() => {
    if (!("modelContext" in navigator)) return;

    navigator.modelContext.registerTool({
      name: "place-order",
      description: "Submit the current cart as an order. Requires user confirmation.",
      inputSchema: {
        type: "object",
        properties: {
          shippingMethod: {
            type: "string",
            enum: ["standard", "express"],
            description: "Shipping speed"
          }
        }
      },
      execute: async ({ shippingMethod }, agent) => {
        const ok = await agent.requestUserInteraction(() =>
          confirm(`Place order with ${shippingMethod} shipping?`)
        );
        if (!ok) throw new Error("Cancelled");
        const order = await submitOrder(cart.id, shippingMethod);
        return { content: [{ type: "text", text: `Order #${order.id} placed.` }] };
      }
    });

    return () => navigator.modelContext.unregisterTool("place-order");
  }, [cart.id]);
}
```

### State-dependent tools

Only register tools when their preconditions are met:

```js
// "undo" only exists when there are undoable actions
useEffect(() => {
  if (!("modelContext" in navigator)) return;
  if (undoStack.length === 0) return;

  navigator.modelContext.registerTool({
    name: "undo",
    description: "Undo the last editing action",
    inputSchema: { type: "object", properties: {} },
    execute: () => {
      const action = undoStack.pop();
      revert(action);
      return { content: [{ type: "text", text: `Undid: ${action.description}` }] };
    }
  });

  return () => navigator.modelContext.unregisterTool("undo");
}, [undoStack.length]);
```

## `provideContext` vs `registerTool`/`unregisterTool`

| Method | Use when |
|--------|----------|
| `provideContext({ tools })` | Swapping the entire tool set (route changes, major state transitions) |
| `registerTool` / `unregisterTool` | Adding or removing individual tools incrementally |

`provideContext` clears all existing tools before registering the new set. Don't mix it with `registerTool` unless you understand the clearing behavior.
