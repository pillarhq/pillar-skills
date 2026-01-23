# Pillar AI Skills

Agent skills for the [Pillar SDK](https://pillar.bot).

Skills are reusable capabilities for AI coding agents (Claude Code, Cursor, Windsurf, GitHub Copilot, etc.) that provide procedural knowledge for integrating the Pillar SDK into your applications.

## Install

```bash
npx skills add pillarhq/pillar-skills --skill pillar-best-practices
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `pillar-best-practices` | Best practices for integrating Pillar SDK into React/Next.js applications |

## What's Included

After installing, your AI coding assistant will know how to:

- Set up `PillarProvider` correctly (including Next.js App Router patterns)
- Write action definitions with good descriptions that the AI can match
- Create centralized action handlers with proper cleanup
- Follow Pillar SDK best practices automatically

## Learn More

- [Pillar SDK Documentation](https://help.trypillar.com/docs)
- [skills.sh](https://skills.sh) - The open agent skills ecosystem
