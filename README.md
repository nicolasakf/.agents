# Agent Skills

This repository is a personal collection of reusable agent skills and supporting assets.
Each skill lives under [`skills/`](skills/) in its own directory and is defined by a
`SKILL.md` file with Agent Skills frontmatter.

## Using the skills

Use [Unity](https://www.npmjs.com/package/@agent-skills/unity) to synchronize this
source directory with supported coding-agent skill locations:

```bash
npx @agent-skills/unity sync
```

For the available commands and configuration, see
[`skills/unity-skill/SKILL.md`](skills/unity-skill/SKILL.md).

## Repository layout

- `skills/` — reusable skill instructions, references, scripts, and assets.
- `subagents/` — notebook definitions for specialized sub-agents.

Runtime state, temporary notebook files, and Unity configuration are excluded via
[`.gitignore`](.gitignore).
