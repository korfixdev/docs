# The `korgames` Section in config.json

> **Navigation:** [index](index.md) · [concepts](concepts.md) · **you are here** · [client-api](client-api.md) · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · [checklist](checklist.md).
> **Related:** [miniapps/config-json.md](../miniapps/config-json.md) — general config.json structure for any miniapp.

To make a miniapp a **game** in the Korfix ecosystem, add a `korgames` section to `config.json`. On install, a platform hook reads it and creates records in `sys_registered_games` and `sys_game_items` (more on the hook — in [concepts.md § Games](concepts.md)).

---

## Minimal Example

```json
{
    "name": "Coin Clicker",
    "alias": "coin-clicker",
    "package": "game-coin-clicker",
    "version": "1.0.0",
    "description": "A clicker game for Korn",
    "about": "## What it does\n30-second clicker...",
    "logo": "icon.svg",
    "urls": { "main": "index.html" },
    "urlsConf": { "main": { "method": "get" } },
    "permissions": {
        "storage": true,
        "navigate": false,
        "modal": true
    },
    "korgames": {
        "game_id": "coin-clicker",
        "reward_mode": "score_only",
        "items": []
    }
}
```

---

## Top-Level Fields for Gamedev

### `package` (required for games)

Global logical miniapp ID in the ecosystem (kebab-case, latin characters).

**Prefix conventions** (to immediately identify the miniapp type by namespace):

| Prefix | What | Examples |
|--------|------|---------|
| `game-` | Game miniapp (with `korgames` section) | `game-coin-clicker`, `game-tetris`, `game-chess-puzzle` |
| `games-` | System service for the game ecosystem | `games-hub` (singular-system, not a game itself but a hub) |
| no prefix | Regular business miniapp | `bdr-report`, `quotes`, `customer-360` |

Your game — **always with `game-` prefix** in package. This gives:
- Marketplace filtering: `/db/marketplace.json?form[package]=game-%` (SQL LIKE-friendly)
- Clarity for other developers
- Lower chance of collision with business miniapps

Why package matters: **other miniapps find yours by package**, not by the platform `marketplace.id` (different on test/prod). Stored in `marketplace.package` (searchable).

Example — finding Games Hub from another game:
```js
const r = await App.fetch('/db/marketplace.json?form[package]=games-hub&form[system]=1&limit=1');
const hub = r.data[0];
App.navigate('/db/marketplace/' + hub.alias);
```

Finding all games:
```js
// Via sys_registered_games (correct way — only published games)
const r = await App.fetch('/api/korgames/games');

// Or via marketplace by prefix (includes not-installed ones)
const r = await App.fetch('/db/marketplace.json?form[package]=game-%&limit=50');
```

### `game_id` vs `package` — the difference

- **`package`** — marketplace-level ID with `game-` prefix (e.g. `game-coin-clicker`). For marketplace search and cross-app discovery.
- **`korgames.game_id`** (below) — short ID for the API and `sys_registered_games.alias` (e.g. `coin-clicker`, no prefix). Used in `/api/korgames/game/score?game_id=X` etc.

Keep them consistent: `package = 'game-' + game_id`.

### `system` (for platform/reserved applications)

Cannot be set via config.json — this is a platform flag (`marketplace.system tinyint`). Assigned by the platform admin manually (`UPDATE marketplace SET system=1 WHERE package='games-hub'`).

Semantics:
- `system=0` (default) — regular user application.
- `system=1` — system/reserved miniapp. Via filter `form[package]=X&form[system]=1`, other games find the "real" Hub, not a fake with the same package from another developer.

**Protection against substitution** (roadmap): backend should refuse deploy if `config.package` matches an existing `system=1` miniapp and the deployer is not the author. Currently not implemented — relies on `system=1` filter when searching.

---

## `korgames` Section Fields

### `game_id` (required)

Unique logical game identifier. Used as `alias` in `sys_registered_games`. Format: `kebab-case`, latin + `-`.

If another developer publishes a game with the same `game_id` — on install UPSERT will overwrite the record. MVP has no conflict checking at the marketplace level. Choose a unique name.

### `reward_mode` (required)

- `"score_only"` — **the only valid value in MVP.** Score recording + item shop. No Korn is awarded by the game.
- `"pool"` — not implemented. Enum reserved in sys_registered_games for future extension.

### `items` (optional)

Array of game shop items. Each item:

```json
{
    "key": "gold_cursor",
    "name": "Gold Cursor",
    "description": "Each click gives 2 points instead of 1",
    "price_corn": 300,
    "max_per_user": 1
}
```

| Field | Type | Meaning |
|-------|------|---------|
| `key` | string | unique within the game; used in API `item_key` |
| `name` | string | display in shop |
| `description` | string | what the item does (hint for user) |
| `price_corn` | int | price in Korn |
| `max_per_user` | int | `0` = unlimited, `1` = buy only once, `N` = up to N copies |

Future fields (in roadmap):
- `price_platinum` — sell for Platinum
- `duration_sec` — temporary buff
- `stacking` — whether effects can be stacked on repeated purchase

---

## What Happens on Install

1. User clicks "Install" in the marketplace → INSERT in `installed_apps`.
2. Core fires `int_done_run('app.installed', [app_id, user_id, from_group])`.
3. Hook `korgames/hooks/on_app_installed.php`:
   - Reads `crm__marketplace.appconfig` (JSON cache, not FS).
   - If `korgames` section exists → `Games::registerGameFromConfig`.
4. `registerGameFromConfig`:
   - UPSERT `sys_registered_games` by `alias = game_id`.
   - UPSERT `sys_game_items` for each item (by `game_id × item_key`).
5. Additional quests: `deploy_app` +1 (for author) and `install_app` +1 (for installer, if different).

> Important: **appconfig in marketplace must be populated before install.** If you deployed via `POST /api/db/marketplace/{id}` — also call `POST /api/marketplace/refresh/{id}`, otherwise the hook logs a warning "appconfig empty" and doesn't register the game. The correct fast path — use `POST /api/marketplace/deploy/{id}` directly (= update + refresh).

---

## Updating a Game

On each deploy (`POST /api/marketplace/deploy/{id}`):
- `appconfig` is re-uploaded from the new config.json.
- New items are added to sys_game_items.
- Changed items (name/description/price) are updated.
- **Items removed from config are NOT deleted** from sys_game_items — they remain is_active=1 (without migration).

If you want to hard-remove old items — delete manually via SQL (and don't forget `sys_game_purchases` may have FK via item_id — but it's append-only, won't be affected).

---

## Limitations and Common Mistakes

1. **`korgames` section requires `game_id`.** If omitted — the hook silently returns without registering.
2. **`reward_mode` strictly from enum.** `"free"`, `"unlimited"` etc. are not supported — hook will throw a SQL error on INSERT (MySQL enum validation).
3. **`price_corn` must be ≥ 0.** Negative price = unexpected emission — check in Games::registerGameFromConfig will reject it.
4. **`max_per_user` > 0 recommended for unique improvements.** Otherwise the user can buy 100 "Gold Cursors" — but the effect will still be applied only once (your game logic).
5. **Don't rely on the game as a Korn source.** The user must earn Korn somewhere (quests, platform actions) and only then spend it in the game shop.

---

## Next Steps

- [client-api.md](client-api.md) — how to call `/api/korgames/game/*` from a JS miniapp.
- [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — walk through the reference game from config.json to score + shop.
- [checklist.md](checklist.md) — what to check before deploying to the marketplace.
