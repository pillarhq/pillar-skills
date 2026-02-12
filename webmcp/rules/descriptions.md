# descriptions

Write specific tool descriptions that tell the agent what the tool does, what it returns, and when to use it. The description is the primary signal agents use to decide tool relevance.

## Bad

```js
// Too vague — agent can't distinguish from other tools
{ description: "Search" }

// No info about return value
{ description: "Get flights" }

// No guidance on when to use vs. similar tools
{ description: "Process order" }
```

## Good

```js
// Specific about what it does and returns
{
  description: "Search available flights by route and date. Returns array of {id, airline, price, departureTime, arrivalTime}. Does not book — use book-flight to complete a booking."
}

// Includes return format and scope
{
  description: "Get the status of an order by ID. Returns current status, estimated delivery date, and tracking URL."
}

// Distinguishes from related tools
{
  description: "Filter the displayed product list by a natural language query. Updates the page UI to show only matching products. Use search-products to get raw data without UI changes."
}
```

## Best Practices

### 1. Describe What It Does and Returns

```js
description: "Add a stamp to the collection. Returns confirmation with the new total count."
```

### 2. Describe Parameters in Natural Language

Every `inputSchema` property needs a `description`:

```js
properties: {
  query: {
    type: "string",
    description: "Natural language search (e.g. 'red shoes under $50 in size 9')"
  }
}
```

### 3. Distinguish Similar Tools

When you have multiple related tools, make the boundaries clear:

```js
// Tool 1
{ description: "Search for products matching a query. Returns product data as JSON." }

// Tool 2
{ description: "Display specific products on the page by ID. Updates the visible product grid." }
```

### 4. Include Examples for Complex Tools

```js
description: `Edit the current design based on natural language instructions. Examples:
  editDesign("Change the title font color to red")
  editDesign("Add a text field at the bottom reading 'Sale ends Friday'")`
```

### 5. Note Side Effects

If a tool changes visible UI, modifies data, or triggers external calls, say so:

```js
description: "Place an order for the items in the cart. Charges the user's saved payment method and sends a confirmation email."
```
