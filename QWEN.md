# Global Auth — Qwen Code Extension

## Project Purpose

This repository contains skills for Global Auth (TxGlobalAuth) — a proprietary authentication widget deployed across multiple products and companies.

This repository also serves as the marketplace (`ga-claude-code-marketplace`). GitHub source: `FriendsOfGlobalAuth/cc-ga-plugins`.

## Language

Official language: **English**.

## Structure

- `.claude-plugin/marketplace.json` — Marketplace registry (plugin catalog)
- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `qwen-extension.json` — Qwen Code extension manifest
- `skills/{skill-name}/SKILL.md` — skill definitions
- Each skill = one directory with a `SKILL.md` file

## Namespace

Extension name: `global-auth`. Skills appear as `global-auth:{skill-name}`.

GitHub source: `FriendsOfGlobalAuth/cc-ga-plugins`.

## Adding New Skills

1. Create `skills/{skill-name}/SKILL.md`
2. Include YAML frontmatter: `name`, `description`
3. Update README.md skills table
4. Bump version in `qwen-extension.json`, `.claude-plugin/plugin.json`, and `.claude-plugin/marketplace.json` (both top-level and plugin entry)
5. Update marketplace plugin entry in `.claude-plugin/marketplace.json` if skill name or description changed

## Updating Skills

1. Edit `skills/{skill-name}/SKILL.md`
2. Preserve YAML frontmatter structure
3. Bump patch version in `qwen-extension.json`, `.claude-plugin/plugin.json`, and `.claude-plugin/marketplace.json` (both top-level and plugin entry)
4. If skill name or description changed — update README.md skills table and marketplace plugin entry in `.claude-plugin/marketplace.json`

## Skill Description Quality

The `description` field in SKILL.md frontmatter is the PRIMARY auto-activation signal. Always include: all name variants, technology keywords, task verbs, "Use when..." trigger, and scope markers. See `docs/llm/skill-description-patterns.md` for the full pattern.
