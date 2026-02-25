# Global Auth — Qwen Code Extension

## Project Purpose

This repository contains skills for Global Auth (TxGlobalAuth) — a proprietary authentication widget deployed across multiple products and companies.

Part of [ga-claude-code-marketplace](https://github.com/FriendsOfGlobalAuth/claude-code-marketplace) (`FriendsOfGlobalAuth/claude-code-marketplace`).

## Language

Official language: **English**.

## Structure

- `qwen-extension.json` — Qwen Code extension manifest
- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `skills/{skill-name}/SKILL.md` — skill definitions
- Each skill = one directory with a `SKILL.md` file

## Namespace

Extension name: `global-auth`. Skills appear as `global-auth:{skill-name}`.

GitHub source: `FriendsOfGlobalAuth/cc-ga-plugins`.

## Adding New Skills

1. Create `skills/{skill-name}/SKILL.md`
2. Include YAML frontmatter: `name`, `description`
3. Update README.md skills table
4. Bump version in `qwen-extension.json` and `.claude-plugin/plugin.json`
5. Update marketplace: add skill reference to `marketplace.json` in `FriendsOfGlobalAuth/claude-code-marketplace`

## Updating Skills

1. Edit `skills/{skill-name}/SKILL.md`
2. Preserve YAML frontmatter structure
3. Bump patch version in `qwen-extension.json` and `.claude-plugin/plugin.json`
4. If skill name or description changed — update README.md skills table and marketplace description

## Skill Description Quality

The `description` field in SKILL.md frontmatter is the PRIMARY auto-activation signal. Always include: all name variants, technology keywords, task verbs, "Use when..." trigger, and scope markers. See `docs/llm/skill-description-patterns.md` for the full pattern.
