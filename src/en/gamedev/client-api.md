# Client API — Calls from the Miniapp

> **Navigation:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · **you are here** · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · [checklist](checklist.md).
> **Related:** [miniapps/js-api.md](../miniapps/js-api.md) — general `VMCRMUserApp` API (methods `App.storage`, `App.modal`, events). Only game endpoints here.

All requests — via `App.fetch` (not `window.fetch` — CORS). Authorization via cookie (parent window session). POST body is passed as an **object**, not a JSON string — VMCRMUserApp serializes it itself. Korn/quests/score concepts — in [concepts.md](concepts.md).

---

## Required Wrapper

`App.fetch` returns `{data: serverResponse, requestId}` — needs unpacking. This is a feature of the postMessage transport (see also [miniapps/data-api.md](../miniapps/data-api.md) § API responses):

```js
async function kg(url, ...rest) {
    const r = await App.fetch(url, ...rest);
    return r?.data ?? r;   // r.data contains what the server returned
}
```

All examples below use `kg(...)`.

---

## Balance and Streak

```js
const r = await kg('/api/korgames/balance');
// r = { status: 'success', data: { corn, platinum, today_earned, daily_remaining,
//       current_streak, longest_streak, total_earned, total_spent,
//       expires_at, last_activity_at, last_login_date } }

const b = r.data;
console.log(`You have ${b.corn} Korn, streak ${b.current_streak} days`);
```

`POST /api/korgames/install` — idempotent, useful to call on miniapp startup:

```js
await kg('/api/korgames/install', { method: 'POST' });
// data: {ok: true, created: true|false, balance: {...}}
```

> Better to cache the `installed` flag in `App.storage` — see `Idempotency via storage` section below.

---

## Quests

### List

```js
const r = await kg('/api/korgames/quests?type=all');
// r.data = [{id, alias, name, description, type, reward_corn,
//           condition_type, condition_value, period_type,
//           status, progress, completed_at, claimed_at}, ...]

for (const q of r.data) {
    const pct = Math.min(100, Math.round(q.progress / q.condition_value * 100));
    console.log(`[${q.status}] ${q.name} — ${pct}%, reward ${q.reward_corn} Korn`);
}
```

Filter `type`: `all` | `daily` | `weekly` | `onboarding` | `achievement`.

### Claim

```js
const r = await kg('/api/korgames/quest/claim', {
    method: 'POST',
    body: { quest_id: 30 }
});
// success: { status: 'success', ok: true, quest_id, quest_alias,
//            reward_corn, earn_result: {ok, earned, capped, balance} }
// error:   { status: 'error', ... }
```

Possible errors: `not_completed`, `already_claimed`, `not_found`.

---

## Leaderboard

Global — aggregates **earn transactions** (not game scores!).

```js
const r = await kg('/api/korgames/leaderboard?period=week');
// r.data = [{author_id, rank, corn_earned, games_played, quests_completed, user_name}, ...]
// period: week | month | all_time
```

Data is materialized in `sys_leaderboard`, updated every 5 minutes by cron.

For a **game-specific leaderboard** (top by score) there's no dedicated endpoint in MVP — read directly:

```js
const r = await kg('/db/sys_game_scores.json?form[game_id]=coin-clicker&orderby=score+DESC&limit=10');
// r.data.data = [{author_id, score, duration_sec, played_at, ...}, ...]
```

---

## Transaction History

```js
const r = await kg('/api/korgames/transactions?limit=50&offset=0');
// r.data = [{id, amount, currency_type, transaction_type, source, source_id,
//           balance_after, description, ts}, ...]
```

---

## Games — List

```js
const r = await kg('/api/korgames/games');
// r.data = [{alias, name, reward_mode, miniapp_id}, ...] — active games
```

---

## Shop — Items

```js
const r = await kg('/api/korgames/game/items?game_id=coin-clicker');
// r.data = [{id, item_key, name, description, price_corn, max_per_user}, ...]
```

---

## Recording Score

```js
const r = await kg('/api/korgames/game/score', {
    method: 'POST',
    body: {
        game_id: 'coin-clicker',
        score: 142,
        duration: 30     // seconds
    }
});
// success: { status:'success', ok:true, session_id, corn_earned (0 in score_only mode) }
```

In `score_only` mode `corn_earned=0` always — Korn is awarded only via quests of type `game_play` / `game_score`, which the server triggers automatically when a score is recorded (if such quests are defined and progress reaches `condition_value` — the user then claims the quest).

---

## Buying an Item

```js
const r = await kg('/api/korgames/game/buy', {
    method: 'POST',
    body: {
        game_id: 'coin-clicker',
        item_key: 'gold_cursor'
    }
});
// success: { status:'success', ok:true, purchase_id, price_corn, balance_after, transaction_id }
// error: { status:'error', error: 'insufficient_balance' | 'limit_exceeded' | 'item_not_found' | ... }
```

Atomic: the platform checks balance, `max_per_user`, deducts Korn, records transaction and `sys_game_purchases`.

---

## Inventory

```js
const r = await kg('/api/korgames/game/inventory?game_id=coin-clicker');
// r.data = [{item_key, price_corn, purchased_at}, ...]

// Which upgrades are purchased
const owned = new Set(r.data.map(p => p.item_key));
if (owned.has('gold_cursor')) multiplier = 2;
```

Without `game_id` parameter — inventory for all games at once.

---

## Game Profile

Separate from the business profile (`auth_pers`). Shown in the leaderboard instead of the system name.

```js
// Get
const r = await kg('/api/korgames/profile');
// r.data = {display_name, avatar_url, bio, exists}
// exists: false if no record yet (first open)

// Update (UPSERT)
const r = await kg('/api/korgames/profile', {
    method: 'POST',
    body: {
        display_name: 'ProGamer42',
        avatar_url:   'https://example.com/me.png',
        bio:          'Top clicker since 2025'
    }
});
// r.data = profile after save, r.created = true/false
```

### Avatar — File Upload

`App.fetch` via postMessage cannot handle `FormData`/`File` (JSON.stringify destroys them). Solution — **base64 data-URL**: the client reads the file, resizes via canvas, sends as a string.

```js
async function uploadAvatar(file) {
    // 1. Read file
    const reader = new FileReader();
    const dataUrl = await new Promise((resolve, reject) => {
        reader.onload = () => resolve(reader.result);
        reader.onerror = () => reject(new Error('read failed'));
        reader.readAsDataURL(file);
    });

    // 2. Resize to 256×256 square (center-crop) to save space
    const img = new Image();
    await new Promise(r => { img.onload = r; img.src = dataUrl; });
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 256;
    const ctx = canvas.getContext('2d');
    const sq = Math.min(img.width, img.height);
    const sx = (img.width - sq) / 2;
    const sy = (img.height - sq) / 2;
    ctx.drawImage(img, sx, sy, sq, sq, 0, 0, 256, 256);

    const outType = file.type === 'image/png' ? 'image/png' : 'image/jpeg';
    const resized = canvas.toDataURL(outType, 0.9);

    // 3. Upload
    const r = await App.fetch('/api/korgames/profile/avatar', {
        method: 'POST',
        body: { avatar_base64: resized }
    });
    const res = r?.data ?? r;
    // res = { status: 'success', avatar_url: '/reimg/data/db/f_sys_game_profiles/avatar_N_xxx.png', mime, size, data: {profile} }
    return res.avatar_url;
}
```

Supported formats: **png, jpeg, webp, gif**. Limit — **512 KB after resize**.

Display with auto-resize via reimg:
```html
<img src="https://{CRM_HOST}/reimg/data/db/f_sys_game_profiles/avatar_3_xxx.png?80x80">
```

### Absolute URLs (avatars, platform resources) {#absolute-urls}

**IMPORTANT — the path must be ABSOLUTE.** The miniapp iframe lives on the store domain (`vmcrm.vnn.ru`), while files in `/reimg/` and `/data/` are on the CRM domain (`vibe.korfix.app`). A relative `src="/reimg/..."` resolves to the store domain → 404.

The CRM host is in `App.requestParams.domain` after `getRequestParams()`. Helper:

```js
function absUrl(path) {
    if (!path || /^https?:\/\//i.test(path)) return path;
    const domain = window.App?.requestParams?.domain || '';
    if (!domain) return path;
    return 'https://' + domain.replace(/\/$/, '') + (path.startsWith('/') ? path : '/' + path);
}

// Usage
img.src = absUrl(profile.avatar_url);  // '/reimg/...' → 'https://vibe.korfix.app/reimg/...'
```

Apply to **any** links to platform resources:
- Game profile avatars (`/reimg/data/db/f_sys_game_profiles/...`)
- Platform avatars (`/reimg/data/auth/...`)
- App icons (`/data/db/f_marketplace/...`)
- Catalog files (`/data/db/f_{catalog}/...`)

Reference: `games-hub/core/api.js` → `KgApi.absUrl()`.

Server-side validation:
- `display_name` — up to 100 chars
- `avatar_url` — up to 500 chars, must start with `http(s)://` if not empty
- `bio` — up to 200 chars

Errors: `display_name_too_long`, `avatar_url_invalid`, `bio_too_long`, `invalid_author`.

In the leaderboard (`/api/korgames/leaderboard`) the `display_name` field is joined automatically via LEFT JOIN — takes priority over the system name from `auth_pers`.

---

## Game-Specific Leaderboard (by score)

No dedicated endpoint in MVP. Read directly:

```js
const r = await kg('/db/sys_game_scores.json?form[game_id]=coin-clicker&orderby=score+DESC&limit=10');
const rows = r?.data?.data || r?.data || [];
// Collapse to 1 record per author (max score per user):
const byAuthor = new Map();
for (const row of rows) {
    const aid = +row.author_id, sc = +row.score;
    if (!byAuthor.has(aid) || byAuthor.get(aid).score < sc) {
        byAuthor.set(aid, { author_id: aid, score: sc, played_at: row.played_at });
    }
}
const top = Array.from(byAuthor.values()).sort((a,b) => b.score - a.score).slice(0, 10);
```

Production example — `coin-clicker/modules/top-scores.js`. See [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md).

---

## Idempotency via Storage

Better not to call install on every open — user was already registered in `sys_user_balances` long ago:

```js
async function ensureInstalled() {
    const s = await App.storage.get('install_status');
    if (s && s.at) return;  // already done
    const r = await kg('/api/korgames/install', { method: 'POST' });
    if (r.status === 'success') {
        await App.storage.set('install_status', {
            at: new Date().toISOString(),
            corn_init: r.data?.balance?.corn ?? 0
        });
    }
}
```

In settings (gear in header) add "Reinstall" — calls install again, updates storage.

---

## Error Handling

```js
try {
    const r = await kg('/api/korgames/quest/claim', { method: 'POST', body: { quest_id: 999 } });
    if (r.status !== 'success') {
        // Business error — not an exception, a valid response
        App.alert('Failed to claim: ' + (r.error || r.message));
        return;
    }
    // r.data — result
} catch (e) {
    // HTTP failure / timeout / permission_denied
    console.error('API call failed:', e);
}
```

HTTP errors (401/403/500) from `App.fetch` usually arrive as a rejected promise or non-success body — handle both paths.

---

## Timeouts

`App.fetch` internally uses postMessage with a 60-second timeout. If the parent didn't respond (e.g., miniapp opened in a detached iframe or host not configured) — you'll get a reject after one minute.

For UX, add your own short timeout:

```js
const withTimeout = (p, ms, label) => Promise.race([
    p,
    new Promise((_, rej) => setTimeout(() => rej(new Error(label + ' timeout')), ms))
]);

await withTimeout(kg('/api/korgames/balance'), 5000, 'balance');
```
