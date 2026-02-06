# schema-compatibility

Action `dataSchema` definitions are converted into LLM tool parameter schemas. Pillar routes tool calls to multiple providers (OpenAI, Anthropic, Google Gemini via OpenRouter), and each provider validates schemas differently. Gemini is the strictest. Follow these rules so your schemas work across all providers.

## Rules

### 1. Arrays must have `items`

Every `type: 'array'` property must include an `items` definition specifying the element type. Gemini rejects arrays without `items`.

```tsx
// Bad - Gemini returns "items: missing field"
orderby: { type: 'array' }

// Good
orderby: { type: 'array', items: { type: 'string' } }

// Good - object items
filters: { type: 'array', items: { type: 'object', properties: { col: { type: 'string' } } } }

// Good - loosely typed (when element shape varies)
annotation_layers: { type: 'array', items: { type: 'object' } }
```

### 2. `type` must be a single string

Gemini's `type` field is an enum (`STRING`, `NUMBER`, `OBJECT`, etc.), not an array. JSON Schema union syntax like `type: ['string', 'null']` is invalid.

```tsx
// Bad - Gemini can't parse type arrays
slug: { type: ['string', 'null'], description: 'URL slug' }

// Good - drop the null variant; use description to note it's optional
slug: { type: 'string', description: 'URL slug (optional, can be empty)' }
```

For fields that accept multiple types (string or number), pick the most common one and note alternatives in the description:

```tsx
// Bad
val: { type: ['string', 'number', 'array', 'null'] }

// Good
val: { type: 'string', description: 'Filter value (string, number, or JSON-encoded array)' }
```

### 3. Objects must have `properties`

Gemini rejects `type: 'object'` without a `properties` field. If the shape is truly open-ended, add at least a minimal property.

```tsx
// Bad - Gemini returns "properties: should be non-empty for OBJECT type"
config: { type: 'object' }

// Acceptable - provide at least one known property
config: { type: 'object', properties: { key: { type: 'string' } } }

// Also acceptable for pass-through objects where the model shouldn't guess
config: { type: 'string', description: 'JSON-encoded configuration object' }
```

### 4. Use simple types only

Stick to these types: `string`, `number`, `integer`, `boolean`, `array`, `object`. Gemini does not support `null` as a standalone type or custom format strings beyond `enum` and `date-time`.

### 5. `required` fields must exist in `properties`

Every entry in a `required` array must correspond to a key defined in `properties`. If Gemini can't parse any property definition (e.g., due to a type union), it treats the entire `properties` block as invalid, and all `required` fields become "not defined."

## Quick checklist

Before syncing actions:

- [ ] Every `type: 'array'` has an `items` field
- [ ] Every `type` value is a single string, not an array
- [ ] Every `type: 'object'` has `properties` (or is changed to `type: 'string'` with a description)
- [ ] Every `required` entry matches a key in `properties`

## References

- [Gemini Schema reference](https://ai.google.dev/api/caching#Schema)
- [Gemini function calling docs](https://ai.google.dev/gemini-api/docs/function-calling)
- [Google AI Forum: array/object issues](https://discuss.ai.google.dev/t/function-calling-issues-with-type-object-and-array/34581)
