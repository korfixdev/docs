# Pre-Release Game Checklist

> **Navigation:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · **you are here**.

The universal miniapp checklist is at [miniapps/checklist.md](../miniapps/checklist.md). Below — game-specific items.

## `korgames` Section in config.json

> Format and fields — [config-korgames.md](config-korgames.md).

- [ ] `package` with `game-` prefix (e.g. `game-coin-clicker`) — required for game miniapps
- [ ] `game_id` is set, kebab-case, unique. Consistent with package (`package = 'game-' + game_id`)
- [ ] `reward_mode: "score_only"` (the only valid value in MVP)
- [ ] Each item in `items[]` has `key`, `name`, `description`, `price_corn ≥ 0`, `max_per_user ≥ 0`
- [ ] Each item `key` is unique within the game
- [ ] `max_per_user = 1` for unique upgrades (otherwise the user can buy 100 identical ones)

## index.html

- [ ] `<meta charset="UTF-8">` in `<head>`
- [ ] Regular `<script>` BEFORE the module script (to declare globals)
- [ ] Module script imports VMCRMUserApp via absolute path `/templates/def/db/marketplace/vmcrm-user-app.js`
- [ ] `window.App = new VMCRMUserApp()` — on `window`, so globals from regular scripts can see it

## Project Structure

> Pattern — [project-structure.md](project-structure.md).

- [ ] `frames/` contains all HTML (`main.html`, optional `settings.html`, `widget.html`)
- [ ] `core/` — singletons (api.js, i18n.js, storage.js)
- [ ] `modules/` — UI logic by section (one file per tab/modal)
- [ ] `locales/` — en.json + ru.json at minimum, default EN
- [ ] `styles/` — CSS in a separate folder
- [ ] `README.md` in the root with change history and structure
- [ ] **body is transparent** (`background: transparent`) — game theme background **only** inside a container (`.cc-frame` or equivalent). See [project-structure.md § body](project-structure.md).

## JS — API

> Call examples — [client-api.md](client-api.md). Reference — [coin-clicker-walkthrough.md § game.js](coin-clicker-walkthrough.md).

- [ ] All calls via `App.fetch`, not `window.fetch`
- [ ] POST body is an object, not `JSON.stringify` + Content-Type
- [ ] Don't pass `undefined` as second argument → error `Cannot read 'body' of null`. Use `(...rest)` → `App.fetch(url, ...rest)`.
- [ ] Wrapper unpacking: `const r = await App.fetch(...); r = r?.data ?? r;`

## i18n

- [ ] All user-facing strings via `i18n.t('key')`, not hardcoded
- [ ] DOM elements with `data-i18n="key"` for automatic translation on language switch
- [ ] `i18n.applyToDom(document)` called after `i18n.init(App)` and after each `i18n.setLang(...)`
- [ ] If rendering blocks via `innerHTML = \`${i18n.t('...')}...\``, remember to re-render on language change
- [ ] `en.json` contains all keys, `ru.json` — a complete copy (no missing keys)

## Error Handling

- [ ] Check `r.status === 'success'` before using `r.data`
- [ ] `try/catch` around `App.fetch` — to avoid crashing on timeout or permission_denied
- [ ] `App.alert` / inline message to the user on error — not silent

## Deploy

- [ ] Deploy via `POST /api/marketplace/deploy/{id}` — update + refresh in one operation
- [ ] After deploy, verify `crm__marketplace.appconfig` and `cont` are populated (not NULL)
- [ ] Install under a test user → verify a record with your `game_id` appears in `sys_registered_games`
- [ ] Verify `sys_game_items` contains all items from config

## Functional Tests

- [ ] Opening the app throws no errors in console
- [ ] `GET /api/korgames/game/inventory?game_id=X` returns an empty array for a new user
- [ ] Game round → `POST /api/korgames/game/score` returned success, `sys_game_scores` +1 record
- [ ] Shop — "Buy" button is disabled if not enough Korn, enabled if enough
- [ ] Purchase → `sys_game_purchases` +1, `sys_transactions` +1 (type='game_buy'), balance decreased
- [ ] After purchase the effect is applied (inventory re-read in `init()`)
- [ ] `max_per_user` works — second attempt to buy the same item returns `limit_exceeded`

## UX

- [ ] Loading state is visible while APIs are loading
- [ ] Balance/streak displayed somewhere in the UI (if the game is long)
- [ ] After a round, the result (score) is shown with clear next steps
- [ ] "Play again" / "Go to shop" — obvious follow-up actions
- [ ] Shop shows why an item is unavailable (not enough Korn / already purchased / limit reached)

## Security

- [ ] Not trying to award Korn yourself — that's the platform's job
- [ ] Not bypassing the shop — purchases only via `/api/korgames/game/buy` (atomic transaction)
- [ ] Not relying on client-side score for critical rewards (it can be forged via devtools) — no anti-cheat in MVP currently

## Documentation

- [ ] `about` in config.json — markdown string with 5 sections (What it does / Where it appears / Features / How to use / Settings)
- [ ] README.md in game root with change history — helps you when you return in a month
- [ ] If the game is complex — link to wiki/blog post in `about`

---

## Commonly Forgotten Things

1. **`permissions`** — explicitly declare `storage`/`navigate`/`modal`. Without it you'll get warning "full access (legacy)".
2. **`logo: "icon.svg"`** — file must be in the zip. Without a logo the app looks empty in the marketplace.
3. **`tags`** — comma-separated. Helps search in the marketplace.
4. **`version`** — bump on every deploy, otherwise users won't see updates.
5. **`about`** — markdown as **a single JSON string** with `\n`. Object format `{short, full}` breaks `VMCRM\Models\UserApps::buildMetaUpdates` (there's a guard, but don't tempt it).

---

## After Release

- Verify your game is in `sys_registered_games` with `is_active=1`.
- Play it yourself under a test user — does everything work end-to-end?
- Add the game to your developer portfolio (if you have a public listing).
- Monitor `sys_game_scores` — if no one is playing, the cause is usually marketplace/discovery, not the code.
