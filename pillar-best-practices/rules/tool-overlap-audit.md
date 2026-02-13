# tool-overlap-audit

Before creating or substantially modifying a tool, audit the existing tools in the codebase for semantic overlap. Two tools that respond to the same user intent cause unpredictable tool selection and confuse the LLM.

## The Rule

**Search existing tools before writing a new one.** Find all tool definition files in the project, read their names, descriptions, and examples, and compare them against the new tool's intended purpose.

## How to Audit

1. **Find all tool definitions.** Search the codebase for tool definition patterns -- objects with `description`, `type`, and optionally `examples`, `dataSchema`, or `guidance`. Common locations: `actions/`, `tools/`, `lib/pillar/`.
2. **Compare descriptions.** Look for tools whose descriptions cover similar intent. Two tools that both mention "create a dashboard" or "invite a user" are candidates for overlap.
3. **Compare examples.** Check `examples` arrays for phrases that would match both the existing and proposed tool. If "add a panel" appears in both `create_visualization` and `add_panel_to_dashboard`, one of them will be picked arbitrarily.
4. **Compare schemas.** If two tools accept similar parameters (same field names, same types), they likely serve the same intent.
5. **Decide: extend, disambiguate, or replace.**

## Decision Matrix

| Situation | Action |
|-----------|--------|
| New tool covers a strict subset of an existing tool | Don't create it. Extend the existing tool's schema if needed. |
| New tool overlaps partially but serves a distinct intent | Create it, but add `guidance` to both tools explaining when to use each. |
| New tool replaces an existing tool entirely | Remove the old tool. Don't leave both registered. |
| New tool is genuinely novel | Proceed. No overlap to resolve. |

## What Overlap Looks Like

```tsx
// These two tools overlap -- "create a dashboard" matches both
save_dashboard: {
  description: 'Create a new dashboard or update the current one with panels.',
  examples: ['create a dashboard', 'new dashboard', 'build a dashboard'],
}

create_empty_dashboard: {
  description: 'Create a new empty dashboard.',
  examples: ['create a dashboard', 'new dashboard', 'start a dashboard'],
}
```

The LLM sees both tools and picks one semi-randomly. Fix by removing `create_empty_dashboard` (since `save_dashboard` with an empty panels array does the same thing) or by adding disambiguation guidance.

## What Disambiguation Looks Like

When two tools must coexist, use `guidance` to draw a clear boundary:

```tsx
save_dashboard: {
  description: 'Create or update a dashboard with visualization panels.',
  guidance: 'Use when the user wants to populate a dashboard with panels. Do NOT use for creating an empty shell -- use create_empty_dashboard instead.',
}

create_empty_dashboard: {
  description: 'Create a new empty dashboard with no panels.',
  guidance: 'Use ONLY when the user explicitly wants a blank dashboard. If they mention panels, charts, or visualizations, use save_dashboard instead.',
}
```

## Checklist

Before registering a new tool:

- [ ] Searched the codebase for all existing tool definitions
- [ ] Compared descriptions for semantic overlap
- [ ] Compared examples for phrase collisions
- [ ] Compared schemas for parameter overlap
- [ ] If overlap found: extended, disambiguated, or replaced -- not duplicated
- [ ] If disambiguated: both tools have `guidance` explaining the boundary
