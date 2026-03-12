# cli-sync

Use `pillar sync` to push tool definitions from your codebase to the Pillar backend. The scanner statically extracts metadata from `defineTool()` and `usePillarTool()` calls using the TypeScript compiler API.

## Commands

```bash
pillar sync --scan ./src              # one-time sync
pillar sync --scan ./src --watch      # watch for changes and re-sync
pillar sync --scan ./src --force      # force sync even if unchanged
pillar sync --scan ./src --local      # print manifest without syncing
pillar sync status --scan ./src       # compare local tools vs remote
```

## Required credentials

Set these as environment variables, in `~/.pillar/config.json` (via `pillar init`), or in `.env.local`:

| Variable | Description |
|----------|-------------|
| `PILLAR_SLUG` | Your product key |
| `PILLAR_SECRET` | Sync secret (created by `pillar init` or in the dashboard) |

## What the scanner extracts

From `defineTool()` and `usePillarTool()` calls:

- `name`, `description`, `guidance`, `type`
- `inputSchema`, `outputSchema`
- `examples`, `autoRun`, `autoComplete`

Fields must be **inline literals**. The scanner cannot resolve variable references, computed values, or `execute` functions.

## AGENT_GUIDANCE.md

Place an `AGENT_GUIDANCE.md` file in the scan directory root. The scanner picks it up automatically and syncs it alongside tools:

```markdown
<!-- src/tools/AGENT_GUIDANCE.md -->
PREFER API TOOLS OVER NAVIGATION:
- When both an API tool and a navigation tool can accomplish a task, prefer the API tool

ORDER FULFILLMENT WORKFLOW:
When a user asks to process an order:
1. Use get_order to fetch order details
2. Use validate_inventory to check stock
3. Use create_shipment to generate shipping label
```

Keep guidance focused on abstract patterns and tool preferences. Per-tool sequencing belongs in the `guidance` field on each tool. See `rules/workflow-patterns.md`.

## CI/CD

Add tool syncing to your deploy pipeline:

```yaml
# GitHub Actions
- name: Sync Pillar tools
  run: npx pillar-cli sync --scan ./src
  env:
    PILLAR_SLUG: ${{ secrets.PILLAR_SLUG }}
    PILLAR_SECRET: ${{ secrets.PILLAR_SECRET }}
```

## Development workflow

Use `--watch` during development for automatic re-sync on save:

```bash
pillar sync --scan ./src --watch
```

Use `status` to see what would change before pushing:

```bash
pillar sync status --scan ./src
```

## Migrating from pillar-sync

The `pillar-sync` command from `@pillar-ai/sdk` is deprecated. Replace in CI:

```bash
# Old
npx -y -p @pillar-ai/sdk@latest pillar-sync --scan ./src

# New
npx pillar-cli sync --scan ./src
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Missing PILLAR_SLUG or PILLAR_SECRET" | Run `pillar init` or set in `.env.local` |
| Scanner finds 0 tools | Ensure `defineTool()` / `usePillarTool()` use inline literal values, not variables |
| Sync returns 403 | Check that `PILLAR_SECRET` matches the product — regenerate with `pillar init` if needed |
| Tools in remote but not local | They were synced previously but removed from code — `pillar sync --force` will delete them |
