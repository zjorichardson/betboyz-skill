# Bet Boyz — OpenClaw / Agent Skills Plugin

An AI agent skill for the [Bet Boyz](https://betboyz.app) sports betting analytics API.

## What This Skill Does

Gives AI agents (Claude, GPT, etc.) full access to:

- **Sportsbook account management** — real and paper (simulated) accounts
- **Bet tracking** — 13 bet types including straight, parlay, teaser, round robin, props, futures
- **Settlement** — manual bet settlement with P&L calculation
- **Analytics** — performance by sport, bet type, sportsbook; win/loss streaks; cognitive bias detection
- **Live odds** — current odds from 8+ sportsbooks
- **Fuzzy search** — find teams and players by name, abbreviation, or partial match

## Installation

### OpenClaw / ClawHub

```bash
clawhub install betboyz
```

Or publish your own fork:

```bash
clawhub publish ./betboyz --slug betboyz --name "Bet Boyz" --version 1.0.0
```

### Claude Code

Copy the `SKILL.md` into your Claude Code skills directory:

```bash
cp SKILL.md ~/.claude/skills/betboyz/SKILL.md
```

### Any Agent Skills-Compatible Agent

The `SKILL.md` follows the [Agent Skills open standard](https://agentskills.io/specification). Drop it into any agent that supports the standard.

## Required Environment Variables

| Variable | Description |
|----------|-------------|
| `BETBOYZ_API_KEY` | Your Bet Boyz API key |
| `BETBOYZ_BASE_URL` | API base URL (e.g. `https://api.betboyz.app`) |

## Getting an API Key

```bash
# 1. Register
curl -X POST https://api.betboyz.app/v1/customers/register \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "YourSecurePass1"}'

# 2. Login
curl -X POST https://api.betboyz.app/v1/customers/login \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "YourSecurePass1"}'

# 3. Create API key (use the session token from login)
curl -X POST https://api.betboyz.app/v1/keys \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer session_..." \
  -d '{"name": "My Agent Key", "scopes": ["bets:read", "bets:write", "analytics:read", "odds:read", "accounts:write"]}'
```

Save the `data.key` value — it's only shown once.

## License

MIT
