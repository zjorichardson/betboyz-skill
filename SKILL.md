---
name: Bet Boyz
description: >
  Track, analyze, and optimize sports betting activity across sportsbooks.
  Manage accounts, log bets of all types (straight, parlay, teaser, round robin, futures, props),
  settle outcomes, surface cognitive biases, and query live odds — all through a REST API
  designed for AI agents.
version: 1.0.0
metadata:
  openclaw:
    slug: betboyz
    category: finance
    tags:
      - sports-betting
      - analytics
      - odds
      - portfolio-tracking
    requires:
      env:
        - name: BETBOYZ_API_KEY
          description: "Your Bet Boyz API key (starts with a random prefix). Create one via POST /v1/keys after registering."
          required: true
        - name: BETBOYZ_BASE_URL
          description: "Base URL of the Bet Boyz API (e.g. https://api.betboyz.app)"
          required: true
    homepage: https://betboyz.app
    repository: https://github.com/zjorichardson/bet-boyz-api
---

# Bet Boyz — AI Agent Skill

You are an AI agent with access to the **Bet Boyz API**, a sports betting analytics platform. This skill gives you full control over a customer's betting data: sportsbook accounts, bet tracking across 13 bet types, settlement, performance analytics, bias detection, and live odds.

## Quick Start

**Base URL:** Use the `BETBOYZ_BASE_URL` environment variable.

**Authentication:** Every request (except health checks) requires an `Authorization: Bearer <key>` header. Use the `BETBOYZ_API_KEY` environment variable as the key value.

**Content-Type:** All request bodies must be `application/json`.

**Example request:**
```
POST {BETBOYZ_BASE_URL}/v1/bets
Authorization: Bearer {BETBOYZ_API_KEY}
Content-Type: application/json

{
  "bet_type": "straight",
  "stake": 50.00,
  "odds_american": -150,
  "legs": [{
    "bet_type": "moneyline",
    "event_description": "Chiefs vs Ravens",
    "team_name": "Kansas City Chiefs"
  }]
}
```

---

## Response Format

All responses follow a consistent envelope.

**Success (single object):**
```json
{
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-03-01T12:00:00Z"
  }
}
```

**Success (list):**
```json
{
  "data": [ ... ],
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-03-01T12:00:00Z",
    "total": 142,
    "cursor": "eyJ...",
    "has_more": true
  }
}
```

**Error:**
```json
{
  "error": {
    "code": "not_found",
    "message": "No bet found with ID ...",
    "status": 404
  },
  "meta": {
    "request_id": "uuid"
  }
}
```

Common error codes: `invalid_credentials`, `unauthorized`, `insufficient_scope`, `not_found`, `duplicate_email`, `duplicate_bookmaker`, `duplicate_paper_name`, `validation_error`, `invalid_uuid`, `rate_limit_exceeded`.

---

## Authentication & API Keys

The API uses two auth mechanisms. **As an agent, you will almost always use an API key.**

| Mechanism | Header Value | Purpose |
|-----------|-------------|---------|
| Session token | `Bearer session_...` | Account management (creating keys, viewing profile) |
| API key | `Bearer <key>` | All /v1/* operations — the primary auth for agents |

API keys are scoped. The scopes on your key determine what you can access:

| Scope | Grants Access To |
|-------|-----------------|
| `bets:read` | List and get bets |
| `bets:write` | Create, update, delete, settle bets |
| `analytics:read` | All analytics endpoints |
| `odds:read` | Sports, events, odds, team/player search |
| `accounts:write` | Sportsbook account CRUD (read and write) |

If your key lacks a required scope, you get `403` with `error.code = "insufficient_scope"`.

### Rate Limits

Every response includes rate limit headers:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 58
X-RateLimit-Reset: 1709312400
```

When exceeded, you get `429` with a `Retry-After` header (seconds). Back off and retry after that duration.

| Tier | Per Minute | Per Day |
|------|-----------|---------|
| trial | 10 | 100 |
| starter | 60 | 10,000 |
| growth | 120 | 100,000 |
| scale | 300 | 1,000,000 |

---

## Endpoints

### Health

#### GET /health
Liveness probe. No auth required.

**Response 200:**
```json
{ "status": "healthy" }
```

#### GET /health/ready
Readiness probe — confirms database and Redis are connected. No auth required.

**Response 200:**
```json
{
  "status": "ready",
  "checks": { "database": "ok", "redis": "ok" }
}
```

---

### Customer Management

These endpoints manage the customer account itself. Typically used during initial setup.

#### POST /v1/customers/register
Create a new customer account. No auth required.

**Body:**
```json
{
  "email": "user@example.com",        // required, valid email
  "password": "SecurePass1",           // required, min 8 chars, uppercase + lowercase + digit
  "company_name": "My Company"         // optional
}
```

**Response 201:** Returns customer profile with `id`, `email`, `tier` (starts as `trial`), `trial_expires_at`.

**Errors:** `409 duplicate_email`, `422 validation_error`.

#### POST /v1/customers/login
Get a session token. No auth required.

**Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass1"
}
```

**Response 200:** Returns `data.token` (session token with `session_` prefix) and `data.customer` profile.

**Errors:** `401 invalid_credentials`.

#### GET /v1/customers/me
Get the authenticated customer's profile.

**Response 200:** Returns customer profile.

---

### API Keys

Manage API keys for programmatic access. These endpoints work with both session tokens and API keys.

#### POST /v1/keys
Create a new API key.

**Body:**
```json
{
  "name": "Production Key",                                    // required
  "scopes": ["bets:read", "bets:write", "analytics:read"],    // optional (defaults to all)
  "expires_at": "2027-01-01T00:00:00Z"                         // optional
}
```

Valid scopes: `bets:read`, `bets:write`, `analytics:read`, `odds:read`, `accounts:write`.

**Response 201:** Returns the full key value in `data.key`. **This is the only time the full key is returned.** Store it immediately.

#### GET /v1/keys
List all API keys for the customer. Returns `key_prefix` and `key_last4` — never the full key.

#### DELETE /v1/keys/{key_id}
Revoke an API key. Returns `204`. The key becomes immediately unusable.

---

### Sportsbook Accounts

Accounts represent the customer's sportsbook accounts (real or paper/simulated). Every bet is linked to an account.

**Requires scope:** `accounts:write`

#### POST /v1/accounts
Create a sportsbook account.

**For a real sportsbook:**
```json
{
  "bookmaker_key": "fanduel",         // required for real accounts
  "display_name": "FanDuel Primary",  // optional
  "initial_balance": 500.00           // optional
}
```

**For a paper (simulated) account:**
```json
{
  "is_paper": true,                   // required
  "paper_name": "NFL Strategy Test",  // required for paper accounts
  "display_name": "Paper NFL",        // optional
  "initial_balance": 1000.00          // optional
}
```

**Response 201:** Returns the account with `id`, `bookmaker_key`, `is_paper`, `current_balance`.

**Errors:** `409 duplicate_bookmaker` (same real bookmaker twice), `409 duplicate_paper_name` (same paper name twice), `422` missing required fields.

Available bookmaker keys: `fanduel`, `draftkings`, `betmgm`, `caesars`, `pointsbet`, `fanatics`, `hard_rock`, `espnbet`.

#### GET /v1/accounts
List all sportsbook accounts for the customer. Returns both real and paper accounts.

#### GET /v1/accounts/{account_id}
Get a single account by UUID.

#### PATCH /v1/accounts/{account_id}
Update account. Allowed fields: `display_name`, `current_balance` (paper accounts only).

#### DELETE /v1/accounts/{account_id}
Delete an account. Returns `204`.

---

### Bets

The core of the platform. Create, query, update, and settle bets of any type.

#### POST /v1/bets
**Requires scope:** `bets:write`

Create a single bet. The `legs` array defines the individual selections.

**Straight bet (moneyline):**
```json
{
  "account_id": "uuid",               // optional, links bet to a sportsbook account
  "bet_type": "straight",
  "stake": 50.00,
  "odds_american": -150,
  "sport_key": "americanfootball_nfl",
  "user_notes": "Confident in Chiefs defense",
  "legs": [{
    "bet_type": "moneyline",
    "event_id": "evt_abc123",
    "event_description": "Chiefs vs Ravens",
    "team_name": "Kansas City Chiefs",
    "odds_american": -150,
    "odds_decimal": 1.67
  }]
}
```

**Spread bet:**
```json
{
  "bet_type": "straight",
  "stake": 100.00,
  "odds_american": -110,
  "legs": [{
    "bet_type": "spread",
    "event_description": "Packers vs Bears",
    "team_name": "Green Bay Packers",
    "spread_value": -3.5,
    "odds_american": -110
  }]
}
```

**Total (over/under):**
```json
{
  "bet_type": "straight",
  "stake": 50.00,
  "odds_american": -110,
  "legs": [{
    "bet_type": "total",
    "event_description": "Packers vs Bears",
    "total_value": 47.5,
    "total_direction": "over",
    "odds_american": -110
  }]
}
```

**Player prop:**
```json
{
  "bet_type": "straight",
  "stake": 25.00,
  "odds_american": -120,
  "legs": [{
    "bet_type": "player_prop",
    "event_description": "Chiefs vs Ravens",
    "player_name": "Patrick Mahomes",
    "prop_stat": "passing_yards",
    "prop_value": 299.5,
    "prop_direction": "over",
    "odds_american": -120
  }]
}
```

**Futures:**
```json
{
  "bet_type": "straight",
  "stake": 25.00,
  "odds_american": 800,
  "legs": [{
    "bet_type": "futures",
    "future_market": "super_bowl_winner",
    "future_outcome": "Kansas City Chiefs",
    "odds_american": 800
  }]
}
```

**Parlay (2+ legs required):**
```json
{
  "bet_type": "parlay",
  "stake": 20.00,
  "odds_american": 600,
  "legs": [
    {
      "leg_number": 1,
      "bet_type": "moneyline",
      "event_description": "Chiefs vs Ravens",
      "team_name": "Kansas City Chiefs",
      "odds_american": -150
    },
    {
      "leg_number": 2,
      "bet_type": "spread",
      "event_description": "Packers vs Bears",
      "team_name": "Green Bay Packers",
      "spread_value": -3.5,
      "odds_american": -110
    }
  ]
}
```

**Same-game parlay:**
```json
{
  "bet_type": "same_game_parlay",
  "stake": 10.00,
  "odds_american": 450,
  "legs": [
    {
      "leg_number": 1,
      "bet_type": "spread",
      "event_description": "Chiefs vs Ravens",
      "team_name": "Kansas City Chiefs",
      "spread_value": -6.5,
      "odds_american": -110
    },
    {
      "leg_number": 2,
      "bet_type": "player_prop",
      "event_description": "Chiefs vs Ravens",
      "player_name": "Patrick Mahomes",
      "prop_stat": "passing_yards",
      "prop_value": 274.5,
      "prop_direction": "over",
      "odds_american": -115
    }
  ]
}
```

**Teaser:**
```json
{
  "bet_type": "teaser",
  "stake": 50.00,
  "odds_american": -120,
  "teaser_points": 6.0,
  "legs": [
    {
      "leg_number": 1,
      "bet_type": "spread",
      "team_name": "Kansas City Chiefs",
      "spread_value": -1.0,
      "odds_american": -110
    },
    {
      "leg_number": 2,
      "bet_type": "spread",
      "team_name": "Green Bay Packers",
      "spread_value": 9.5,
      "odds_american": -110
    }
  ]
}
```

**Round robin:**
```json
{
  "bet_type": "round_robin",
  "stake": 10.00,
  "round_robin_size": 2,
  "legs": [
    { "leg_number": 1, "bet_type": "moneyline", "team_name": "Chiefs", "odds_american": -150 },
    { "leg_number": 2, "bet_type": "moneyline", "team_name": "Packers", "odds_american": -120 },
    { "leg_number": 3, "bet_type": "moneyline", "team_name": "Eagles", "odds_american": +130 }
  ]
}
```

**Live bet (placed during a game):**
```json
{
  "bet_type": "straight",
  "stake": 30.00,
  "odds_american": +150,
  "live_game_time": "Q3 8:42",
  "legs": [{
    "bet_type": "moneyline",
    "event_description": "Chiefs vs Ravens",
    "team_name": "Kansas City Chiefs",
    "odds_american": +150
  }]
}
```

**All supported bet_type values:** `straight`, `parlay`, `same_game_parlay`, `teaser`, `if_bet`, `reverse`, `round_robin`.

**All supported leg bet_type values:** `moneyline`, `spread`, `total`, `player_prop`, `game_prop`, `futures`.

**Validation rules:**
- `stake` must be > 0
- `odds_american` cannot be 0
- Parlays and same-game parlays require at least 2 legs
- `legs` array must have at least 1 element

**Response 201:** Returns the full bet with all legs populated and `status: "pending"`.

#### POST /v1/bets/batch
**Requires scope:** `bets:write`

Create up to 100 bets in one request.

**Body:**
```json
{
  "bets": [
    { /* same shape as single bet create */ },
    { /* ... */ }
  ]
}
```

**Response 201:**
```json
{
  "data": {
    "created": 5,
    "bets": ["uuid1", "uuid2", "uuid3", "uuid4", "uuid5"]
  }
}
```

**Error:** `422` if more than 100 bets in the array.

#### GET /v1/bets
**Requires scope:** `bets:read`

List bets with optional filters and cursor pagination.

**Query parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `status` | string | Filter by status: `pending`, `won`, `lost`, `push`, `void`, `cashed_out` |
| `bet_type` | string | Filter by bet type: `straight`, `parlay`, etc. |
| `account_id` | UUID | Filter by sportsbook account |
| `sport_key` | string | Filter by sport (e.g. `americanfootball_nfl`) |
| `cursor` | string | Pagination cursor from previous response |
| `limit` | int | Page size (default 50, max 100) |

**Response 200:** Returns array of bets with legs. Uses cursor-based pagination — pass `meta.cursor` to get the next page. Check `meta.has_more` to know if more pages exist. `meta.total` gives total matching count.

#### GET /v1/bets/{bet_id}
**Requires scope:** `bets:read`

Get a single bet with all legs.

#### PATCH /v1/bets/{bet_id}
**Requires scope:** `bets:write`

Update a bet. Currently only `user_notes` is editable.

**Body:**
```json
{
  "user_notes": "Updated analysis after watching film"
}
```

#### DELETE /v1/bets/{bet_id}
**Requires scope:** `bets:write`

Delete a bet. Returns `204`.

#### POST /v1/bets/{bet_id}/settle
**Requires scope:** `bets:write`

Manually settle a bet. Provide the outcome for each leg.

**Body:**
```json
{
  "status": "won",
  "actual_payout": 83.33,
  "legs": [
    {
      "leg_number": 1,
      "status": "won",
      "result_value": null
    }
  ]
}
```

Leg status values: `won`, `lost`, `push`, `void`, `cashed_out`.

For spread/total/prop bets, `result_value` is the actual numeric outcome (e.g., final score, actual passing yards).

**Settlement rules:**
- **Parlay:** All legs must win for the parlay to win. One loss = entire parlay lost. A pushed leg reduces the parlay (lower payout). A voided leg is removed from the parlay.
- **Teaser:** Same as parlay but with adjusted spreads.
- **Round robin:** Each combination settles independently.
- **Profit/loss** is computed as `actual_payout - stake` (won) or `-stake` (lost).

---

### Analytics

Performance analytics computed from the customer's betting history. All endpoints require `analytics:read` scope.

All analytics endpoints accept these query parameters:
- `account_id` (string, optional) — restrict to a specific sportsbook account
- `include_paper` (bool, optional, default: `false`) — include paper account bets in calculations

#### GET /v1/analytics/summary
Overall performance summary.

**Response fields:**
- `total_bets`, `total_won`, `total_lost`, `total_push`, `total_void`, `pending` — bet counts
- `win_rate` — decimal (e.g. 0.55 = 55%)
- `total_staked`, `total_payout`, `profit_loss` — financial totals
- `roi_pct` — return on investment percentage
- `avg_stake`, `avg_odds_decimal` — averages
- `largest_win`, `largest_loss` — extremes
- `best_day`, `worst_day` — best/worst single-day P&L

#### GET /v1/analytics/by-sport
Performance broken down by sport. Each entry includes `sport_key`, win rate, ROI, P&L.

#### GET /v1/analytics/by-type
Performance broken down by bet type. Each entry includes `bet_type`, win rate, ROI, P&L.

#### GET /v1/analytics/by-book
Performance broken down by sportsbook account. Each entry includes `account_id`, `bookmaker_key`, `is_paper`, win rate, ROI, P&L. Useful for comparing real vs paper accounts.

#### GET /v1/analytics/streaks
Current and longest win/loss streaks.

**Response fields:**
- `current_type` — `"win"` or `"loss"`
- `current_length` — how many consecutive
- `longest_win_streak`, `longest_loss_streak` — all-time records

#### GET /v1/analytics/biases
Cognitive bias detection — the platform's core differentiator. Analyzes betting patterns to flag common biases.

**Query parameters:**
- `period_days` (int, optional, default: 30) — lookback window

**Response fields:**
- `bias_score` — overall score (0 = disciplined, higher = more biased)
- `indicators` — object with named indicators, each containing:
  - `is_flagged` — whether this bias is detected
  - `description` — human-readable explanation
  - Indicator-specific fields (varies by type)

**Known bias indicators:**
| Indicator | What It Detects | Key Fields |
|-----------|----------------|------------|
| `chasing_losses` | Increasing stakes after losses | `avg_stake_after_loss`, `avg_stake_after_win`, `ratio` |
| `parlay_heavy` | >50% of bets are parlays | `parlay_pct` |
| `favorite_heavy` | >70% bets on favorites | `favorite_pct` |
| `on_tilt` | Rapid betting after big loss | `rapid_bets_after_loss` |

#### GET /v1/analytics/time-patterns
Performance patterns by day of week and hour of day. Helps identify when the user bets best/worst.

**Response fields:**
- `by_day_of_week` — array of 7 entries (0=Monday through 6=Sunday), each with `day_name`, `bets_placed`, `win_rate`, `profit_loss`, `roi_pct`
- `by_hour_of_day` — array of 24 entries (0-23), each with `hour`, `bets_placed`, `win_rate`, `profit_loss`, `roi_pct`

---

### Odds & Events

Live odds data from multiple sportsbooks. All endpoints require `odds:read` scope.

#### GET /v1/sports
List all active sports with available odds data.

**Response fields per sport:** `key`, `group_name`, `title`, `active`, `has_outrights`.

#### GET /v1/sports/{sport_key}/events
List upcoming (not completed) events for a sport, ordered by commence time.

**Response fields per event:** `id`, `sport_key`, `commence_time`, `home_team`, `away_team`, `completed`, `scores_json`.

#### GET /v1/events/{event_id}
Get details for a single event.

#### GET /v1/events/{event_id}/odds
Get odds from all bookmakers for an event.

**Query parameters:**
- `bookmaker` (string, optional) — filter to a specific bookmaker key

**Response:**
```json
{
  "data": {
    "event_id": "string",
    "odds": [
      {
        "bookmaker_key": "fanduel",
        "bookmaker_title": "FanDuel",
        "markets": [
          {
            "key": "h2h",
            "outcomes": [
              { "name": "Kansas City Chiefs", "price": -150 },
              { "name": "Baltimore Ravens", "price": +130 }
            ],
            "last_update": "2026-03-01T12:00:00Z"
          }
        ]
      }
    ]
  }
}
```

Market keys: `h2h` (moneyline), `spreads`, `totals`.

---

### Search

Fuzzy search for teams and players. Uses PostgreSQL trigram matching. Requires `odds:read` scope.

#### GET /v1/teams/search?q={query}
Search teams by name, abbreviation, or alias. Minimum 2 characters.

Handles partial matches ("Pack" → Packers), abbreviations ("GB" → Green Bay Packers), and typos ("Grean Bay" → Green Bay Packers).

**Response fields per result:** `id`, `name`, `short_name`, `abbreviation`, `league` (object with `id`, `name`, `sport_key`), `similarity` (0-1 relevance score).

#### GET /v1/players/search?q={query}
Search players by name. Minimum 2 characters. Handles partial and fuzzy matches.

**Response fields per result:** `id`, `first_name`, `last_name`, `full_name`, `position`, `team` (object with `id`, `name` or null), `similarity`.

---

### Usage

#### GET /v1/usage
Get current API usage and rate limit information for the authenticated customer.

**Response:**
```json
{
  "data": {
    "customer_id": "uuid",
    "tier": "starter",
    "rate_limit": {
      "requests_per_minute": 60,
      "requests_per_day": 10000
    },
    "usage_today": {
      "requests": 142,
      "limit": 10000
    },
    "usage_this_month": {
      "requests": 2847
    },
    "trial_expires_at": null,
    "trial_days_remaining": null
  }
}
```

---

## Common Workflows

### 1. First-Time Setup (Register → Key → Account → Ready)

```
1. POST /v1/customers/register  → get customer profile
2. POST /v1/customers/login     → get session token
3. POST /v1/keys                → create API key with desired scopes (SAVE THE KEY)
4. POST /v1/accounts            → create sportsbook account(s)
```

After step 3, use the API key for all subsequent requests.

### 2. Log a Bet

```
1. GET /v1/sports/{sport}/events          → find the event
2. GET /v1/events/{id}/odds               → check current odds
3. POST /v1/bets                          → create the bet
```

### 3. Settle and Analyze

```
1. POST /v1/bets/{id}/settle              → settle with outcome
2. GET /v1/analytics/summary              → check overall P&L
3. GET /v1/analytics/biases               → check for cognitive biases
4. GET /v1/analytics/streaks              → check win/loss streaks
```

### 4. Paper Trading Strategy Test

```
1. POST /v1/accounts (is_paper: true)     → create paper account
2. POST /v1/bets (account_id: paper_id)   → place paper bets
3. POST /v1/bets/{id}/settle              → settle paper bets
4. GET /v1/analytics/summary?account_id=X → analyze paper performance
5. GET /v1/analytics/summary?account_id=Y → compare against real account
```

### 5. Multi-Book Comparison

```
1. GET /v1/analytics/by-book              → P&L by sportsbook
2. Compare win rates, ROI across books
3. Identify which sportsbook is most profitable
```

### 6. Bias Check After a Losing Streak

```
1. GET /v1/analytics/streaks              → confirm streak length
2. GET /v1/analytics/biases               → check if chasing losses or on tilt
3. GET /v1/analytics/time-patterns        → identify if losing at specific times
4. GET /v1/analytics/by-type              → check if overweighting parlays
```

---

## State Management

**Critical:** The API key is shown exactly once when created. If you lose it, the only recovery is to revoke and create a new one. You **must** persist credentials after the setup flow.

### What to Save After Setup

After completing the First-Time Setup workflow, immediately save these values to your environment or config file:

| Value | Where It Comes From | Why It Matters |
|-------|---------------------|----------------|
| `BETBOYZ_BASE_URL` | Provided by user or defaults to `https://api.betboyz.app` | Every request needs this |
| `BETBOYZ_API_KEY` | `POST /v1/keys` → `data.key` | **Shown once, never again.** This is your primary auth. |
| `BETBOYZ_CUSTOMER_ID` | `POST /v1/customers/register` → `data.id` | Useful for debugging, not required for API calls |
| `BETBOYZ_ACCOUNT_IDS` | `POST /v1/accounts` → `data.id` (one per account) | Every bet links to an account. Save all account IDs with labels. |

### How to Save (by platform)

**Claude Code / OpenClaw agents:**
Write credentials to your `.env` file or equivalent config immediately after each setup step:
```
BETBOYZ_BASE_URL=https://api.betboyz.app
BETBOYZ_API_KEY=bb_live_04bbe965aa1debeb0038e057ebd82144
BETBOYZ_CUSTOMER_ID=878bfc8c-0219-4400-b036-121fe03d842c
BETBOYZ_ACCOUNT_FANDUEL=<uuid>
BETBOYZ_ACCOUNT_DRAFTKINGS=<uuid>
BETBOYZ_ACCOUNT_PAPER_NFL=<uuid>
```

**Other agents:**
Use whatever persistence mechanism your platform provides (memory, config files, key-value store). The API key is the most critical — without it, you cannot authenticate.

### On Subsequent Sessions

1. Check if `BETBOYZ_API_KEY` is already set in your environment
2. If yes → skip setup, start making API calls
3. If no → run the First-Time Setup workflow and save credentials
4. To discover existing accounts: `GET /v1/accounts` (requires a valid API key)
5. To verify your key still works: `GET /v1/usage` (returns tier, limits, and trial status)

### If You Lose Your API Key

1. Log in: `POST /v1/customers/login` (email + password still works)
2. List keys: `GET /v1/keys` (shows prefix + last4 to identify which key is which)
3. Revoke the lost key: `DELETE /v1/keys/{key_id}`
4. Create a new key: `POST /v1/keys`
5. Save the new key immediately

---

## Tips for AI Agents

1. **Always check `meta.has_more`** when listing bets. Don't assume a single page contains all results.

2. **Store the API key securely.** It is only returned once on creation. If lost, revoke the old key and create a new one.

3. **Use paper accounts for backtesting.** Create a paper account, log hypothetical bets, settle them, then compare analytics against real accounts using `GET /v1/analytics/by-book`.

4. **Check biases regularly.** After every 10-20 settled bets, call `GET /v1/analytics/biases` to flag patterns early.

5. **Use `account_id` filters** on analytics to isolate performance by sportsbook or paper vs real.

6. **Batch create bets** when logging historical data. `POST /v1/bets/batch` handles up to 100 bets per request.

7. **Settlement is manual in Phase 1.** The API does not auto-settle from live scores. After a game ends, call `POST /v1/bets/{id}/settle` with the outcome.

8. **Respect rate limits.** Check `X-RateLimit-Remaining` headers. If you hit 429, wait for `Retry-After` seconds before retrying.

9. **UUID validation matters.** All IDs in path parameters must be valid UUIDs. Invalid formats return `422 invalid_uuid`.

10. **Trial accounts expire after 14 days.** Check `GET /v1/usage` for `trial_days_remaining`. After expiry, all keys become invalid.
