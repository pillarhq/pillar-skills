# codebase-verification

Tool handlers call real APIs, navigate real routes, and return real data shapes. Every field name, endpoint path, and response type must come from the actual source code -- not from guessing or assumptions.

## The Rule

**Read the code before writing the tool.** The developer's project is the Cursor workspace. Search it directly for route handlers, controllers, models, and DTOs to get exact shapes.

## What To Verify

Before writing a tool handler that calls an API:

1. **Find the endpoint handler.** Search the codebase for the route path (e.g., `/api/users`) to find the controller or handler function.
2. **Read the request shape.** Look at the request DTO, body parser, or validation schema to get exact field names, types, and which fields are required.
3. **Read the response shape.** Look at what the handler returns -- the status codes, response body structure, and field names.
4. **Check validation rules.** Look for required fields, enums, min/max constraints, and format requirements. These map directly to `inputSchema` properties.
5. **Write `inputSchema` to match.** Use the exact field names and types from the source. Don't rename, abbreviate, or paraphrase.

## What To Verify for Navigation Tools

For tools that navigate the user to a page:

1. **Find the route definition.** Search for the path in the router config (e.g., React Router, Next.js pages, or framework-specific routing).
2. **Check required parameters.** If the route has path params (e.g., `/dashboard/:uid`), those become required fields in the tool's `inputSchema`.
3. **Check available query params.** If the page reads query params, those become optional fields.

## When the Source Isn't Available

If the API or route handler isn't in the workspace (e.g., it's in a separate backend repo, a third-party service, or behind an SDK):

- **Ask the developer.** Don't guess the API shape. Ask what the endpoint expects and returns.
- **Check API documentation.** Look for OpenAPI specs, Swagger docs, or README files in the project.
- **Check SDK types.** If the project uses a typed client or SDK, read the type definitions to infer request/response shapes.

Never fabricate API shapes. A tool with wrong field names will silently fail at runtime.

## Common Mistakes

### Guessing field names

```tsx
// Bad -- assumed the field is called "userId"
inputSchema: {
  type: 'object',
  properties: {
    userId: { type: 'string' },
  },
  required: ['userId'],
}

// Good -- read the API handler and found it expects "user_id"
inputSchema: {
  type: 'object',
  properties: {
    user_id: { type: 'string' },
  },
  required: ['user_id'],
}
```

### Assuming response structure

```tsx
// Bad -- assumed the API returns { data: [...] }
const items = response.data;

// Good -- read the handler and found it returns { results: { items: [...] } }
const items = response.results.items;
```

### Hardcoding paths without checking

```tsx
// Bad -- guessed the route
path: '/settings/billing'

// Good -- searched the router config and found the actual path
path: '/account/billing-settings'
```
