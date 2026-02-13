# workflow-patterns

Design multi-tool workflows using the distributed guidance pattern. Instead of a central workflow file, encode workflow knowledge into per-tool `guidance` fields and abstract patterns in `AGENT_GUIDANCE.md`.

## The Problem

When tools form workflows (e.g., creating a dashboard requires checking datasources, creating the container, adding panels, navigating to the result), the LLM needs to know the order. Two bad approaches:

1. **One monolith tool** -- unreliable because the schema is huge and the LLM guesses wrong on fields
2. **Central workflow file** -- brittle because adding a tool means updating the workflow definition separately

## The Solution: Distributed Guidance

Encode workflow knowledge in two layers:

### Layer 1: Abstract Patterns (AGENT_GUIDANCE.md)

Product-level guidance that describes principles without naming specific tools:

```markdown
## Complex Resource Creation

When the user asks to create something with dependencies:
1. Discover what exists (datasources, existing resources)
2. If a required dependency is missing, ask the user before proceeding
3. If multiple options exist, ask the user to choose
4. Create the container resource first, then add components one at a time
5. Validate each component before committing
6. Navigate to the finished result
```

### Layer 2: Per-Tool Guidance

Each tool declares its own prerequisites and follow-ups:

```tsx
get_available_datasources: {
  guidance: 'Call BEFORE creating dashboards or panels. If zero results, ask user to create one.',
}

create_dashboard: {
  guidance: 'First step when building a new dashboard. Returns dashboard_uid needed by all create_*_panel tools. Call get_available_datasources first.',
}

create_timeseries_panel: {
  guidance: 'Requires dashboard_uid from create_dashboard. MUST call test_datasource_query first to validate the query.',
}

navigate_to_dashboard: {
  guidance: 'Final step after creating a dashboard and adding panels.',
}
```

The LLM combines the abstract pattern ("discover -> create container -> add components -> navigate") with per-tool guidance ("Requires dashboard_uid from create_dashboard") to plan multi-step workflows.

## Why This Works

- **Adding a tool**: just add guidance to the new tool. The abstract pattern still applies.
- **Removing a tool**: remove it. Other tools' guidance still makes sense individually.
- **No sync burden**: each tool is self-contained. No central file to keep in sync.

## Anti-Patterns

### Don't hardcode tool sequences in AGENT_GUIDANCE.md

```markdown
<!-- Bad: brittle, breaks when tools change -->
To create a dashboard:
1. Call get_available_datasources
2. Call create_dashboard
3. Call create_timeseries_panel for each metric
4. Call navigate_to_dashboard
```

### Don't skip guidance on prerequisite tools

```tsx
// Bad: LLM doesn't know this needs to be called first
get_available_datasources: {
  description: 'Get available datasources.',
  // No guidance -- LLM may skip this step
}
```

### Don't use guidance for user-facing information

```tsx
// Bad: guidance is for the agent, not the user
create_dashboard: {
  guidance: 'Creates a beautiful new dashboard for monitoring your infrastructure!',
  // This should be in the description
}
```

## Checklist

Before shipping a new set of tools:

- [ ] Each tool with prerequisites has `guidance` naming those prerequisites
- [ ] Each entry-point tool says it should be called first
- [ ] Each exit-point tool says it's the final step
- [ ] Similar tools have disambiguation in their guidance
- [ ] AGENT_GUIDANCE.md describes abstract patterns, not specific tool sequences
