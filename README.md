# global-auth — Claude Code Plugins

Claude Code plugins for Global Auth (TxGlobalAuth) integration across projects.

Part of [ga-claude-code-marketplace](https://github.com/FriendsOfGlobalAuth/claude-code-marketplace).

## Available Skills

| Skill | Command | Description |
|-------|---------|-------------|
| frontend | `/global-auth:frontend` | TxGlobalAuth widget integration — initialization, auth flows, token management, UI embedding |
| backend | `/global-auth:backend` | JWT validation, session data extraction, and common misconceptions about the token format |

## Installation

Via marketplace (recommended):

```bash
claude plugin marketplace add FriendsOfGlobalAuth/claude-code-marketplace
claude plugin install global-auth@ga-claude-code-marketplace --scope user
```

Direct install:

```bash
claude plugin install global-auth --scope user
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
├── .claude-plugin/
│   └── plugin.json
└── skills/
    ├── frontend/
    │   └── SKILL.md
    └── backend/
        └── SKILL.md
```

## License

MIT

## Authors

Dmitry Vorobyev — [github.com/VoDmAl](https://github.com/VoDmAl)
