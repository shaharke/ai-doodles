# Contributor Guide

## Repository Structure

```
ai-doodles/
├── .claude-plugin/
│   └── marketplace.json      # Marketplace manifest (lists all plugins)
├── plugins/
│   └── <plugin-name>/
│       ├── .claude-plugin/
│       │   └── plugin.json   # Plugin manifest (lists skills)
│       ├── README.md          # Plugin documentation
│       └── skills/
│           └── <skill-name>/
│               └── SKILL.md   # Skill definition
```

## Adding a New Plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` with name, description, version, and skills array.
2. Add skill files under `plugins/<name>/skills/<skill-name>/SKILL.md`.
3. Add a `plugins/<name>/README.md` with install instructions and skill descriptions.
4. Register the plugin in `.claude-plugin/marketplace.json` under the `plugins` array.

## Skill Format

Skills use SKILL.md with YAML frontmatter:

```yaml
---
name: skill-name
description: 'Short description. Used for auto-invocation matching.'
---
```

The `description` field is how Claude decides when to auto-invoke the skill — write it to match natural user intent (e.g., "commit this", "save changes").

## Naming Conventions

- Plugin names: lowercase, hyphenated (e.g., `gitops`, `code-review`)
- Skill names: lowercase, hyphenated (e.g., `commit`, `pre-push-check`)
- Keep names short and descriptive

## Versioning

When updating a plugin (adding/removing/modifying skills, changing plugin metadata, etc.), always bump the `version` field in the plugin's `plugin.json`. Use semver: patch for fixes, minor for new skills or features, major for breaking changes.

## Git Commits

Always use the `/commit` skill when committing changes.
