# Skill Description Patterns for Auto-Activation

## Purpose

Claude Code skills auto-activate based on keyword matching in the `description` field of SKILL.md YAML frontmatter. A well-written description dramatically increases the chance of the skill being triggered when relevant.

## Pattern: Keyword-Dense Description

### Good Example (frontend skill)
```yaml
description: "TxGlobalAuth (GlobalAuth, GA) JavaScript widget integration. Use when working with TxGlobalAuth, auth-widget, Kratos authentication, JWT token management, or embedding authentication UI in frontend projects."
```

**Why it works:**
- Includes all name variants: `TxGlobalAuth`, `GlobalAuth`, `GA`
- Lists technologies: `JavaScript`, `JWT`
- Lists tasks: `integration`, `token management`, `embedding authentication UI`
- Includes related systems: `Kratos`, `auth-widget`
- Explicit trigger phrase: `Use when working with...`
- Scope marker: `frontend projects`

### Bad Example (original backend skill)
```yaml
description: "TxGlobalAuth backend integration — JWT validation, session data extraction, and common misconceptions about the token format."
```

**Why it's weak:**
- "common misconceptions about the token format" — nobody searches for this
- Missing name variants (GlobalAuth, GA)
- Missing technology keywords (Node.js, NestJS, Express, Go)
- Missing task keywords (verify, validate, decode, API endpoint)
- No explicit `Use when...` trigger

### Fixed Example (backend skill)
```yaml
description: "TxGlobalAuth (GlobalAuth, GA) backend integration. Use when validating TxGlobalAuth JWT tokens on the server, decoding session/person data, building auth verify endpoints, or working with Kratos token introspection in Node.js, NestJS, Express, Go, or any backend."
```

## Rules for Skill Descriptions

1. **Include all name variants** — abbreviations, full names, common misspellings
2. **List technologies** — frameworks, languages, protocols the skill covers
3. **List task verbs** — what the developer is trying to DO (validate, integrate, embed, decode)
4. **Add "Use when..."** — explicit activation trigger phrase
5. **Include related systems** — adjacent technologies that would trigger the need
6. **Add scope markers** — frontend/backend/fullstack to disambiguate
7. **Avoid abstract language** — "common misconceptions" is not a search term
8. **Keep under ~300 chars** — long enough for keywords, short enough to parse

## Gotchas

- Description is the PRIMARY auto-activation signal — skill content doesn't matter for triggering
- Duplicate key terms between description and skill name if the name is short/ambiguous
- Test by asking yourself: "If a developer says X, would this description match?"
