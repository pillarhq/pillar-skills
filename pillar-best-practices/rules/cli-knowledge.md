# cli-knowledge

Use `pillar knowledge` to manage knowledge sources — docs sites, help centers, and other content the copilot draws on when answering questions.

## Commands

```bash
pillar knowledge add <url>           # add a source and start crawling
pillar knowledge list                # list all sources
pillar knowledge status              # health overview of all sources
pillar knowledge status <source-id>  # sync history for a specific source
pillar knowledge sync <source-id>    # re-trigger a crawl
pillar knowledge remove <source-id>  # delete a source
```

## Adding a source

```bash
pillar knowledge add https://docs.myapp.com
```

The CLI:
1. Detects the source type (website, GitHub, Notion) from the URL
2. Creates the source in the backend
3. Triggers an initial crawl
4. Polls until the crawl completes (or times out after ~5 minutes)

URL normalization is automatic — `docs.myapp.com` becomes `https://docs.myapp.com`.

## Source types

| URL pattern | Detected type |
|-------------|---------------|
| `github.com/*` | `github` |
| `notion.so/*` or `notion.site/*` | `notion` |
| Everything else | `website_crawl` |

## Checking health

```bash
pillar knowledge status
```

Shows per-source status: pages crawled, pages indexed, and last sync time.

For detailed sync history of a specific source:

```bash
pillar knowledge status <source-id>
```

## Re-crawling

Trigger a fresh crawl to pick up new or changed content:

```bash
pillar knowledge sync <source-id>
```

## Removing a source

```bash
pillar knowledge remove <source-id>
```

Prompts for confirmation before deleting.

## Prerequisites

- Must be authenticated (`pillar auth login`)
- Must have an agent slug configured (`pillar init` or `PILLAR_SLUG`)

## Automatic setup via init

`pillar init` detects docs URLs in your project (from `package.json` homepage, README links, etc.) and offers to add them as knowledge sources automatically. This is the easiest way to get started.
