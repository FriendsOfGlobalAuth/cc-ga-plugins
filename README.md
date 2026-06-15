# global-auth — Claude Code, Qwen Code, Codex & Cursor Plugin

Global Auth (TxGlobalAuth) integration skills for Claude Code, Qwen Code, Codex and Cursor.

This repository also serves as the Claude marketplace (`ga-claude-code-marketplace`), Codex marketplace (`ga-codex-marketplace`) and Cursor marketplace (`ga-cursor-marketplace`).

## Available Skills

| Skill | Command | Description |
|-------|---------|-------------|
| frontend | `/global-auth:frontend` | TxGlobalAuth widget integration - initialization, auth flows, token management, UI embedding |
| backend | `/global-auth:backend` | TxGlobalAuth backend integration - JWT validation, session data extraction, TxAuth Agent, HTTP introspect, auth verify endpoints |
| audit | `/global-auth:audit` | TxGlobalAuth full-stack integration audit and code review - auth-widget/API usage, JWT validation, token exchange, sessions, identity keys, PII, CORS, deployment posture |

## Installation

### Claude Code

```bash
claude plugin marketplace add FriendsOfGlobalAuth/cc-ga-plugins
claude plugin install global-auth@ga-claude-code-marketplace --scope user
```

### Qwen Code

```bash
qwen extensions install FriendsOfGlobalAuth/cc-ga-plugins
```

### Codex

```bash
codex plugin marketplace add FriendsOfGlobalAuth/cc-ga-plugins
codex plugin add global-auth@ga-codex-marketplace
```

### Cursor

In Cursor, open an Agent chat and run:

```bash
/add-plugin global-auth@https://github.com/FriendsOfGlobalAuth/cc-ga-plugins
```

## Usage Examples

**Audit existing integration:**
```
Review this project's TxGlobalAuth integration. Check for missing error handling,
deprecated methods, and token management issues.
```

**Add authentication to a page:**
```
Add a login button that requires authentication, agreements, and a verified email
before allowing the user to post comments.
```

**Fix token handling:**
```
Our app sometimes loses the user session after navigating between pages.
Review how we handle subscribeJWT and token state.
```

**Integrate from scratch:**
```
This project needs TxGlobalAuth. Set up initialization, token subscription,
and auth level detection. The appName is "myProduct", env is "prod".
```

**Cross-domain navigation:**
```
Add a "Go to Dashboard" link that preserves the user's L2 authentication
when navigating to dashboard.example.com.
```

## Project Structure

```
cc-ga-plugins/
├── .agents/
│   └── plugins/
│       └── marketplace.json # Codex marketplace registry
├── .claude-plugin/
│   ├── marketplace.json     # Claude Code Marketplace registry
│   └── plugin.json          # Claude Code manifest
├── .codex-plugin/
│   └── plugin.json          # Codex manifest
├── .cursor-plugin/
│   ├── marketplace.json     # Cursor Marketplace registry
│   └── plugin.json          # Cursor manifest
├── qwen-extension.json      # Qwen Code manifest
├── CLAUDE.md                # Claude Code context
├── QWEN.md                  # Qwen Code context
└── skills/
    ├── audit/
    │   └── SKILL.md
    ├── backend/
    │   └── SKILL.md
    └── frontend/
        └── SKILL.md
```

## License

MIT

## Authors

Dmitry Vorobyev — [github.com/VoDmAl](https://github.com/VoDmAl)
