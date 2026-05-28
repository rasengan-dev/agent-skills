# Contributing to Rasengan.js Agent Skills

Thank you for contributing to Rasengan.js agent skills! This guide explains how to add or modify skills.

## Skills Overview

Each skill is a directory under the repo root containing a `SKILL.md` file with YAML frontmatter:

```markdown
---
name: skill-name
description: What this skill does (shown to agents when deciding to load it)
license: MIT
metadata:
  author: your-name
---

# Skill Content
...
```

## Adding a New Skill

1. Create a new directory under `rasengan`: `mkdir rasengan/<skill-name>`
2. Add a `SKILL.md` with frontmatter (`name`, `description`, `license`)
3. The `name` must match the directory name, be 1-64 lowercase chars with hyphens
4. The `description` must be 1-1024 characters
5. Add any supporting rule files in a `rules/` subdirectory

## SKILL.md Guidelines

- Start with a "When to Activate" section describing triggering conditions
- Use concrete code examples
- Include an anti-patterns table
- Keep information specific to the skill domain

## Commit Messages

Use conventional commits: `type(scope): description`

Examples:
- `feat: add rasengan/pages skill`
- `fix: correct metadata merge priority example`
- `docs: improve loader function guide`

## Pull Request Process

1. Create a feature branch from `main`
2. Commit your changes
3. Open a PR targeting `main`
4. A maintainer will review and merge
