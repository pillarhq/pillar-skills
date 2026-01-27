# Pillar Skills

Agent skills for integrating the [Pillar SDK](https://trypillar.com) into your applications.

[![skills.sh](https://img.shields.io/badge/skills.sh-compatible-brightgreen)](https://skills.sh)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Overview

Skills are reusable capabilities for AI coding agents that provide procedural knowledge for integrating the Pillar SDK into your applications. Install a skill once, and your AI assistant will automatically follow best practices.

## Compatible AI Assistants

| Assistant | Support |
|-----------|---------|
| [Cursor](https://cursor.com) | Full support |
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) | Full support |
| [Windsurf](https://codeium.com/windsurf) | Full support |
| [GitHub Copilot](https://github.com/features/copilot) | Partial support |

## Documentation

**[View Full Documentation](https://trypillar.com/docs)** | [SDK Quick Start](https://trypillar.com/docs/getting-started/quick-start) | [skills.sh](https://skills.sh)

## Installation

```bash
npx skills add pillarhq/pillar-skills --skill pillar-best-practices
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `pillar-best-practices` | Best practices for integrating Pillar SDK into React/Next.js applications |

## What's Included

After installing, your AI coding assistant will know how to:

- **Set up PillarProvider correctly** — Including Next.js App Router patterns with client components
- **Write effective action definitions** — With descriptions that the AI can accurately match
- **Create centralized action handlers** — With proper cleanup to prevent memory leaks
- **Follow best practices automatically** — TypeScript patterns, error handling, and more

## Skill Contents

The `pillar-best-practices` skill includes:

```
pillar-best-practices/
├── SKILL.md              # Skill metadata and quick reference
├── AGENTS.md             # Complete SDK integration reference
└── rules/
    ├── setup-provider.md     # Provider setup patterns
    ├── setup-nextjs.md       # Next.js App Router specific setup
    ├── action-descriptions.md # Writing effective action descriptions
    └── action-handlers.md    # Handler implementation patterns
```

## Example Usage

Once installed, simply ask your AI assistant:

> "Add Pillar help to my React app"

Your assistant will automatically:
1. Install the correct packages
2. Set up `PillarProvider` at the app root
3. Handle Next.js App Router requirements if applicable
4. Configure actions with proper descriptions
5. Create handlers with correct cleanup patterns

## Related Packages

| Package | Description |
|---------|-------------|
| [@pillar-ai/sdk](https://github.com/pillarhq/sdk) | Core vanilla JavaScript SDK |
| [@pillar-ai/react](https://github.com/pillarhq/sdk-react) | React bindings |
| [@pillar-ai/vue](https://github.com/pillarhq/sdk-vue) | Vue 3 bindings |
| [@pillar-ai/svelte](https://github.com/pillarhq/sdk-svelte) | Svelte bindings |

## Learn More

- [Pillar SDK Documentation](https://trypillar.com/docs)
- [skills.sh](https://skills.sh) — The open agent skills ecosystem

## License

MIT
