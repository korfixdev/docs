# Korfix Game Economy Concepts

> **Navigation:** [index](index.md) · **you are here** · [config-korgames](config-korgames.md) · [client-api](client-api.md) · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · [checklist](checklist.md).

Core concepts used by `/api/korgames/*`, Games Hub, and game miniapps. Read this before [client-api.md](client-api.md) and coding.

---

## Currencies

| Currency | Issuer | Purpose |
|----------|--------|---------|
| **Korn** | Platform only (for whitelisted events) | Universal game currency. Earned through quests and activity, spent in game shops and (planned) pool mode. |
| **Platinum** | Real money (purchased with fiat) | Premium. Buying premium miniapps, subscriptions. Does not go into games outside the shop. |
| **Game-internal currencies** | The game itself (in its storage) | The game's own business — the platform does not know or participate. If you want a "crystals" coin — store in `App.storage`. |

Your game primarily works with **Korn** (the only cross-game currency).

## Korn Emission — Whitelist Only

The platform awards Korn only for:

| Event | Who awards | Source |
|-------|-----------|--------|
| `login` | Core | First session of the day (onboarding daily) |
| `streak` | Core | Milestone bonuses at 3/7/14/30 days (+30/100/250/1000 Korn) |
| `create_record` | Core | Creating a record in whitelisted catalogs (accounts/projects/...) |
| `deploy_app` | Core | Deploying your miniapp to the marketplace |
| `install_app` | Core | Installing another user's miniapp |
| `referral` | Core | An invited user registered |
| `quest` | Core | `claimQuest` after condition_value is reached |
| `game_pool` | Core (planned) | Prize for winning a pool round |
| `admin` | Core | Manual awards by developers |

**Your game cannot call `earnCorn`.** In Games.class.php there's a hard whitelist on source — an attempt from a non-whitelisted source returns an error and writes nothing. This contract protects Korn from inflation (implementation — on the server side `\korgames\Games::earnCorn`, see backend `Docs/korgames/security.md` for details).

**Why:** anti-inflation. If every game minted its own Korn — the currency would become worthless. Games are markets/channels, not mints.

## Daily Cap

Each user can earn no more than `daily_cap` Korn per day (default 500). Excess is cut: `earnCorn` returns `capped: true`, awarding only the amount remaining up to the cap.

Read in `GET /api/korgames/balance`:
```json
{"corn": 120, "today_earned": 480, "daily_remaining": 20, ...}
```

## Korn Expiry

Each Korn award shifts `corn_expires_at` forward by 90 days. If a user is inactive — after 90 days the cron `expire_coins` resets their balance (recorded in sys_transactions as `expire`).

For games: don't promise users "save Korn forever" — saving for more than 90 days without activity is not possible.

---

## Streak

`current_streak` in the balance — how many consecutive days the user has logged in. Updated on the first login of each day.

- Missed a day → streak resets to 0 (via cron `daily_reset`).
- Milestone bonuses at 3/7/14/30 days → automatic `earnCorn` (source='streak').
- `longest_streak` — all-time record (monotonically non-decreasing).

Use `current_streak` in the game UI — it's a social signal ("4th day in a row — nice!").

---

## Quests

A **quest** — a task declaration in the `sys_quests` reference table. **User progress** — in `sys_user_quests`.

### Types (`sys_quests.type`)

| Type | Frequency | Example |
|------|-----------|---------|
| `onboarding` | `once` (once in a lifetime) | "Fill out your profile" |
| `daily` | Resets daily (UTC) | "Create 3 records today" |
| `weekly` | Resets weekly | "Deploy a miniapp this week" |
| `achievement` | `once`, but long-running | "100 hours in the game" |

### condition_type — which event we count

Built-in types:
- `login` — each session.start (counts page-loads — condition is usually 1, "first login of the day")
- `streak` — **absolute streak value** (not increment!)
- `create_record` — INSERT in whitelisted catalogs
- `deploy_app` — miniapp deploy
- `install_app` — installing another user's miniapp
- `referral` — successful referral registration
- `game_play` — a round of any game (+1 per session)
- `game_score` — `value = score` (absolute, NOT increment — though currently incremental, like login)
- `profile_fill` — reserved, no trigger yet (see backend gotchas)

### user_quests Lifecycle

```
available → in_progress (on first progress)
         → completed (progress >= condition_value)
         → claimed (user clicked "Claim" → +reward_corn)
```

`claimed` — final for that period_key. For a new period (next day for daily) a new record is created.

### How to Create Your Own Quest

Add a row to `sys_quests` via SQL (migration in `public_html/log/`):

```sql
INSERT INTO crm__sys_quests (alias, name, description, type, reward_corn, condition_type, condition_value, period_type, is_active, sort_order, from_auth, from_group)
VALUES ('my_super_quest', 'Super Quest', 'Do something cool', 'daily', 50, 'create_record', 5, 'daily', 1, 100, 0, 0);
```

If condition_type is standard (`create_record` etc.) — the quest works immediately. If you need a custom condition_type — you need a server-side hook/call to `Games::checkQuest('your_type', +value)`. This is done in the platform module, not in the miniapp — backend modification described in backend `Docs/korgames/hooks-and-cron.md`.

---

## Games

A **game** = a miniapp with a `korgames` section in config.json. On install, the `on_app_installed` hook reads the section and UPSERTs a record in `sys_registered_games` (alias=game_id).

**Package convention:** game miniapps must be prefixed with `game-` (`game-coin-clicker`, `game-tetris`) — see [config-korgames.md § package](config-korgames.md). This simplifies filtering and coexistence with business miniapps.

### Game Modes (`reward_mode`)

- `score_only` (MVP) — leaderboard only, no Korn awarded. Sell items for Korn that the user earned outside the game.
- `pool` (planned) — a round with an entry fee in Korn, winner takes the pool minus platform commission. Limits in `max_corn_per_session`/`max_corn_per_day`.

### Leaderboard

The **global Korfix leaderboard** (`/api/korgames/leaderboard`) aggregates `SUM(earn_korn)` across all sources — this is not a game-specific ranking, but a rating of the most active users.

For a **game-specific leaderboard** (top by `sys_game_scores` for a specific game) — there's no endpoint yet. Planned: `/api/korgames/game/scores?game_id=X&period=all_time`. For now, build it yourself from `App.fetch('/db/sys_game_scores.json?form[game_id]=X')`.

### Item Shop

Items are declared in `config.korgames.items` → on install they are created in `sys_game_items`. Purchase via `POST /api/korgames/game/buy`:

1. User selects an item.
2. Platform atomically deducts Korn, writes a transaction and `sys_game_purchases`.
3. User inventory = SELECT from `sys_game_purchases WHERE author_id=me`. Game reads via `GET /api/korgames/game/inventory?game_id=X`.

Item effects — your responsibility: you apply them in your client-side game logic.

---

## User Roles

Important distinction: `author_id` != `from_group`.

- `author_id` — unique user ID (person).
- `from_group` — tenant ID (the organization the user works in).

All `sys_*` records store both. The leaderboard by default ranks across all tenants — if you need isolation, filter `from_group` in the request.

---

## What to Read Next

- [config-korgames.md](config-korgames.md) — how to declare your game
- [client-api.md](client-api.md) — JS API with examples
- [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — the reference game
- [checklist.md](checklist.md) — before release
