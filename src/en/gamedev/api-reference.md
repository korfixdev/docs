# API Reference — `/api/korgames/*`

> **Navigation:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · **you are here** · [recipes](recipes.md) · [styling](styling.md) · [checklist](checklist.md).

Complete endpoint reference for the korgames module. All examples are real responses from the test instance.

---

## General

**Authorization:** cookie session (miniapp receives from iframe parent) OR Bearer token with apiclass `korgames_*`. Miniapps typically use cookies.

**Response format:**
```json
{"status": "success", "data": { ... }}
{"status": "error", "message": "human msg", "error": "machine_code"}
```

**HTTP codes:** 200 for success/business errors, 401 if no session/token, 403 if token without apiclass, 500 uncaught.

**Client wrapper:** `App.fetch` returns `{data: serverResponse, requestId}` — unpack with `r?.data ?? r` (see [client-api.md](client-api.md)).

---

## Balance & Install

### `POST /api/korgames/install`

Idempotently creates a record in `sys_user_balances` for the current user. Call on first miniapp open (or less often — see pattern in games-hub).

**Request:** empty body.

**Response (success):**
```json
{
    "status": "success",
    "data": {
        "ok": true,
        "created": false,
        "balance": {
            "corn": 130,
            "platinum": 0,
            "today_earned": 130,
            "daily_remaining": 370,
            "current_streak": 1,
            "longest_streak": 1,
            "total_earned": 130,
            "total_spent": 0,
            "expires_at": "2026-07-20 23:53:01",
            "last_activity_at": "2026-04-22 23:53:01",
            "last_login_date": "2026-04-22"
        }
    }
}
```
- `created: false` — record already existed. `true` — created now.

**Errors:** system errors only (500).

### `GET /api/korgames/balance`

Current balance + streak + daily limit.

**Response:**
```json
{
    "status": "success",
    "data": {
        "corn": 130,
        "platinum": 0,
        "today_earned": 130,
        "daily_remaining": 370,
        "current_streak": 1,
        "longest_streak": 1,
        "total_earned": 130,
        "total_spent": 0,
        "expires_at": "2026-07-20 23:53:01",
        "last_activity_at": "2026-04-22 23:53:01",
        "last_login_date": "2026-04-22"
    }
}
```

If user has not installed — `data: null` or error. Check `r.data` before accessing.

---

## Quests

### `GET /api/korgames/quests?type=all|daily|weekly|onboarding|achievement`

List of active quests with current user's progress. `type=all` (default).

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 30,
            "alias": "daily_login",
            "name": "Daily Login",
            "description": "Log into the panel every day",
            "type": "daily",
            "reward_corn": 10,
            "condition_type": "login",
            "condition_value": 1,
            "period_type": "daily",
            "status": "claimed",
            "progress": 1,
            "completed_at": "2026-04-22 01:55:28",
            "claimed_at": "2026-04-22 01:29:16"
        },
        {
            "id": 31,
            "alias": "daily_create_3",
            "name": "Create 3 records",
            "description": "Three new records today",
            "type": "daily",
            "reward_corn": 30,
            "condition_type": "create_record",
            "condition_value": 3,
            "period_type": "daily",
            "status": "available",
            "progress": 0,
            "completed_at": null,
            "claimed_at": null
        }
    ]
}
```

**Statuses:** `available` → `in_progress` → `completed` → `claimed`.

### `POST /api/korgames/quest/claim`

Claim reward from a completed quest. Can only be done once per period_key.

**Request body:**
```json
{"quest_id": 30}
```

**Response (success):**
```json
{
    "status": "success",
    "ok": true,
    "quest_id": 30,
    "quest_alias": "daily_login",
    "reward_corn": 10,
    "earn_result": {
        "ok": true,
        "earned": 10,
        "capped": false,
        "balance": 10
    }
}
```

**Errors:** `not_completed`, `already_claimed`, `not_found`, `daily_cap_exceeded`.

---

## Leaderboard & History

### `GET /api/korgames/leaderboard?period=week|month|all_time`

Top by Korn earned for the period. Updated by cron `leaderboard_refresh` every 5 minutes.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "rank": 1,
            "author_id": 3,
            "display_name": "DemoPlayer",
            "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc123.png",
            "user_name": "Finance",
            "corn_earned": 130,
            "apps_deployed": 0,
            "quests_completed": 3,
            "games_played": 8,
            "prize_corn": 0
        }
    ],
    "period": "week",
    "period_key": "2026-W17"
}
```

- `display_name` — from sys_game_profiles (priority). `user_name` — from auth_pers.author_comment (fallback). `null`/empty if not filled.
- `avatar_url` — relative path. Make absolute via `absUrl()` (see [client-api.md](client-api.md)).

**Empty array** — cron hasn't run yet or no transactions for the period.

### `GET /api/korgames/transactions?limit=100&offset=0`

Transaction history for the current user.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 102,
            "author_id": 3,
            "amount": 100,
            "currency_type": "corn",
            "transaction_type": "earn",
            "source": "quest",
            "source_id": "weekly_streak_7",
            "balance_after": 130,
            "description": "Earn via quest",
            "ts": "2026-04-22 14:34:08"
        }
    ]
}
```

**`transaction_type`:** `earn`, `spend`, `purchase`, `expire`, `admin`, `game_buy`.

**`source`** (whitelist for earn): `login`, `streak`, `referral`, `create_record`, `deploy_app`, `install_app`, `quest`, `game_pool`, `admin`.

---

## Games — Registration & List

### `GET /api/korgames/games`

List of active registered games.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 8,
            "alias": "coin-clicker",
            "name": "coin-clicker",
            "miniapp_id": 109,
            "reward_mode": "score_only",
            "is_active": 1
        }
    ]
}
```

- `alias` == `korgames.game_id` from miniapp config.json
- `name` may equal alias (if hook saved it) or a human-readable name. For display, fetch `marketplace.name` via an additional request.
- `miniapp_id` — FK to `marketplace.id`

### `GET /api/korgames/game/items?game_id=coin-clicker`

Game shop items.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 20,
            "item_key": "gold_cursor",
            "name": "Gold Cursor",
            "description": "Each click gives 2 points instead of 1",
            "price_corn": 300,
            "max_per_user": 1,
            "is_active": 1
        }
    ]
}
```

### `POST /api/korgames/game/score`

Record round result. Triggers quests `game_play` +1 and `game_score` value=score.

**Request body:**
```json
{
    "game_id": "coin-clicker",
    "score": 182,
    "duration": 30
}
```

**Response:**
```json
{
    "status": "success",
    "ok": true,
    "session_id": 14,
    "corn_earned": 0
}
```

- `corn_earned: 0` in `reward_mode=score_only` (MVP) — Korn is not awarded for score, only via quests.

**Errors:** `game_not_found` (game_id not in sys_registered_games), `invalid_score` (< 0).

### `POST /api/korgames/game/buy`

Atomic item purchase. Deducts Korn, writes `sys_transactions` and `sys_game_purchases`.

**Request body:**
```json
{
    "game_id": "coin-clicker",
    "item_key": "gold_cursor"
}
```

**Response (success):**
```json
{
    "status": "success",
    "ok": true,
    "purchase_id": 5,
    "price_corn": 300,
    "balance_after": 830,
    "transaction_id": 103
}
```

**Errors:**
- `insufficient_balance` — not enough Korn
- `limit_exceeded` — already purchased `max_per_user` times
- `item_not_found` — no such item_key in `sys_game_items`
- `game_not_active` — game `is_active=0`

### `GET /api/korgames/game/inventory?game_id=X` (game_id optional)

User inventory.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 5,
            "item_key": "gold_cursor",
            "price_corn": 300,
            "purchased_at": "2026-04-22 14:12:33"
        }
    ]
}
```

Without `game_id` — inventory for all games. Use `new Set(r.data.map(p => p.item_key))` for quick "is X purchased?" check.

---

## Profile (cross-game)

### `GET /api/korgames/profile`

Game profile for current user — cross-game, lives in `sys_game_profiles`.

**Response:**
```json
{
    "status": "success",
    "data": {
        "display_name": "DemoPlayer",
        "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc123.png",
        "bio": "Love clicker games",
        "exists": true
    }
}
```

`exists: false` if record not yet created (first open) — other fields are empty. **Not an error**, just start with `exists=false` and prompt user to fill in.

### `POST /api/korgames/profile`

UPSERT game profile.

**Request body:**
```json
{
    "display_name": "DemoPlayer",
    "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc123.png",
    "bio": "Love clicker games"
}
```

**Validation:**
- `display_name` ≤ 100 chars
- `avatar_url` ≤ 500 chars, must start with `http(s)://`, `/reimg/`, or `/data/` if not empty
- `bio` ≤ 200 chars

**Response:**
```json
{
    "status": "success",
    "data": { "display_name": "...", "avatar_url": "...", "bio": "...", "exists": true },
    "created": true
}
```

**Errors:** `display_name_too_long`, `avatar_url_too_long`, `avatar_url_invalid`, `bio_too_long`, `invalid_author`.

### `POST /api/korgames/profile/avatar`

Upload avatar via base64 data-URL. Client-side resize to 256×256 via canvas is recommended before sending.

**Request body (JSON):**
```json
{"avatar_base64": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUg..."}
```

Supported formats: `image/png`, `image/jpeg`, `image/webp`, `image/gif`. Validated by magic bytes (cannot spoof mime).

**Limit:** 512 KB after decoding.

**Response:**
```json
{
    "status": "success",
    "data": { "display_name": "...", "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc.png", "bio": "...", "exists": true },
    "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc.png",
    "size": 45678,
    "mime": "image/png"
}
```

`avatar_url` — reimg path, supports query `?WxH` for on-the-fly resize.

**Errors:**
- `no_file` — empty avatar_base64
- `bad_format` — not a data-URL or mime not in allowlist
- `decode_failed` — base64 doesn't decode
- `too_large` — > 512 KB
- `not_image` — magic bytes don't match mime
- `fs_error` — no write permission to `/data/db/f_sys_game_profiles/`
- `write_failed` — file_put_contents returned false

**Why not multipart:** `App.fetch` via postMessage does JSON.stringify, File/FormData are lost. Data-URL = transport via string.

---

## Swagger

### `GET /api/korgames/swagger`

OpenAPI 3.0 spec for all endpoints. **Public** (no auth) — convenient for type generation.

---

## What to Read Next

- [recipes.md](recipes.md) — ready-made recipes for all scenarios (earn, spend, shop, leaderboard, profile)
- [styling.md](styling.md) — styling for game miniapps
- [client-api.md](client-api.md) — client wrapper and call patterns
