# Gamedev — Building Games and Game Mechanics for Korfix

A section for developers who want to:
- **Build their own game** for the Korfix marketplace with a leaderboard and shop.
- **Embed gamification elements** in a regular miniapp (show balance, quests, streak).
- **Extend the existing Games Hub** with their own widget.

> Regular miniapp development (without gamedev specifics) — [miniapps/index.md](../miniapps/index.md). A game miniapp is a regular miniapp with an additional `korgames` section in config.json and use of `/api/korgames/*` endpoints.
>
> Relevant common sections: [miniapps/config-json.md](../miniapps/config-json.md) (general format), [miniapps/js-api.md](../miniapps/js-api.md) (VMCRMUserApp), [miniapps/data-api.md](../miniapps/data-api.md) (CRUD via App.fetch).

---

## Quick Start

1. [concepts.md](concepts.md) — what is Korn, quests, leaderboards, shop, profile (read first).
2. [config-korgames.md](config-korgames.md) — the `korgames` section in config.json: game_id, reward_mode, items, package convention.
3. [api-reference.md](api-reference.md) — **complete reference for all endpoints** with request/response (structures, error codes).
4. [client-api.md](client-api.md) — JS wrapper around `App.fetch`, postMessage transport details.
5. [recipes.md](recipes.md) — **ready-made recipes** for all common mechanics (earn/spend/shop/leaderboard/profile/avatar).
6. [styling.md](styling.md) — styling for game miniapps (transparent body, game-frame, CSS tokens).
7. [project-structure.md](project-structure.md) — modular structure (frames/core/modules/locales/styles), i18n pattern.
8. [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — line-by-line walkthrough of the reference app.
9. [checklist.md](checklist.md) — pre-release checklist.

**TL;DR for vibe-coding agents:** read in order 3 → 5 → 6 → 7. That's API + Recipes + Styling + Structure — 80% of what you need to build a game from a single prompt.

---

## Key Rules

- **Games do NOT mint Korn.** Emission comes only from whitelisted platform events (login, streak, create_record, deploy_app, install_app, referral, quest). Your game can:
  - Record `score` in the global leaderboard (`POST /api/korgames/game/score`).
  - Sell `items` for Korn (user spends from their wallet earned outside the game).
  - Participate in `pool` mode (enter a round for Korn, winner takes the pool minus platform fee) — planned, not in MVP.

- **All game + shop data** is registered automatically on install — via the `korgames` section in `config.json`. No custom tables needed.

- **User state** — personal flags, settings, save data — store in `App.storage` (KV storage per app_id + user).

- **Emission limits.** The platform has a `daily_cap` (default 500 Korn/day), balances expire after 90 days of inactivity.

---

## What's Already Implemented

| Mechanic | Status | Document |
|----------|--------|----------|
| Korn balance, user wallet | MVP ✅ | [concepts.md](concepts.md) |
| Daily login streak | MVP ✅ | [concepts.md](concepts.md) |
| Quests (daily/weekly/onboarding) | MVP ✅ | [concepts.md](concepts.md) |
| Leaderboard by Korn earned | MVP ✅ | [client-api.md](client-api.md) |
| Game profile (display_name, avatar, bio) | MVP ✅ | [client-api.md](client-api.md) · [project-structure.md](project-structure.md) |
| Game registration from config.json | MVP ✅ | [config-korgames.md](config-korgames.md) |
| Item shop for Korn | MVP ✅ | [client-api.md](client-api.md) |
| Score in game + sys_game_scores | MVP ✅ | [client-api.md](client-api.md) |
| i18n (EN/RU) | MVP ✅ | [project-structure.md](project-structure.md) |
| Local top by game score | MVP ✅ (via `/db/sys_game_scores.json`) | [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) |
| Pool mode (round entry stake) | planned | — |
| Pvp / team-battle | not in MVP scope | — |

---

## Platform Integration

Server-side module documentation — in core docs: `sited_core3php8/Docs/korgames/`:

| File | Contents |
|------|----------|
| `index.md` | overview, dependencies, dev config |
| `architecture.md` | layers, event flow, extension points |
| `catalogs.md` | 9 `sys_*` tables (exact columns for direct queries via `/db/sys_*.json`) |
| `services-and-api.md` | `\korgames\Games::*` methods + `/api/korgames/*` specification |
| `hooks-and-cron.md` | 5 core hooks + 3 cron jobs |
| `security.md` | hard-gate `sys_*`, source whitelist |
| `gotchas.md` | 12 known pitfalls (including `App.fetch` wrapper) |

For game miniapp development you don't need to know the backend: everything is done via `App.fetch('/api/korgames/*')`. But if you're building a server-side integration or adding your own quest condition_type — go to backend docs.
