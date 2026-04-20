# web-dev-skills

A collection of Claude Code skills for web application and backend development.

## What are Skills?

Skills are SKILL.md files that teach Claude specific domain knowledge, patterns, and checklists. They are invoked as slash commands inside Claude Code (e.g., `/twelve-factor-app`).

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [Twelve-Factor App](.claude/skills/twelve-factor-app/SKILL.md) | `/twelve-factor-app` | Audit a backend service against the 12-factor methodology — flags violations and suggests fixes |

## Usage

1. Clone this repo into your project or add it as a Claude Code plugin
2. Invoke a skill using its slash command in Claude Code
3. Point it at your service directory to get an audit

## Adding Skills

Each skill lives in `.claude/skills/<skill-name>/SKILL.md` with this frontmatter:

```markdown
---
name: skill-name
description: One-line description
origin: local
---
```
