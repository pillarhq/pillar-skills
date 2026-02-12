# user-interaction

Use `agent.requestUserInteraction()` to pause tool execution and get user confirmation or input. The agent object is the second parameter passed to `execute`.

## When to Use

- Purchases, payments, fund transfers
- Deleting accounts, data, or resources
- Sending emails, messages, or notifications
- Modifying security settings (passwords, MFA)
- Any action the user would expect to confirm in normal UI

## When Not to Use

- Read-only queries (order status, search results)
- Filtering or sorting displayed content
- Reversible UI state changes
- Fetching data to return to the agent

## Pattern

```js
execute: async ({ productId, quantity }, agent) => {
  // Show confirmation to the user
  const confirmed = await agent.requestUserInteraction(async () => {
    return confirm(`Add ${quantity} of product ${productId} to cart and checkout?`);
  });

  if (!confirmed) {
    throw new Error("Purchase cancelled by user");
  }

  await checkout(productId, quantity);
  return { content: [{ type: "text", text: `Order placed for ${quantity} items.` }] };
}
```

## Multiple Interactions

`requestUserInteraction` can be called multiple times in a single tool execution:

```js
execute: async ({ items }, agent) => {
  // First: confirm items
  const itemsOk = await agent.requestUserInteraction(() =>
    confirm(`Order ${items.length} items?`)
  );
  if (!itemsOk) throw new Error("Cancelled");

  // Second: confirm payment method
  const paymentOk = await agent.requestUserInteraction(() =>
    confirm("Charge to card ending 4242?")
  );
  if (!paymentOk) throw new Error("Payment cancelled");

  await processOrder(items);
  return { content: [{ type: "text", text: "Order complete." }] };
}
```

## Custom UI in Interactions

The callback can render any UI, not just `confirm()`:

```js
const address = await agent.requestUserInteraction(() => {
  return new Promise((resolve) => {
    showAddressPickerModal({
      onSelect: (addr) => resolve(addr),
      onCancel: () => resolve(null)
    });
  });
});

if (!address) throw new Error("No address selected");
```

## Error Handling

Throw an error when the user declines. This signals to the agent that the action was not completed:

```js
if (!confirmed) {
  throw new Error("Action cancelled by user");
}
```

Don't return a success response when the action was cancelled.
