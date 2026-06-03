# Gamedev Recipes — Common Scenarios

> **Navigation:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · [api-reference](api-reference.md) · **you are here** · [styling](styling.md) · [checklist](checklist.md).

Ready-made snippets for common tasks. Copy, fill in your values, deploy.

All examples assume initialization:
```js
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
window.App = new VMCRMUserApp();
await App.getRequestParams();  // required BEFORE storage ops / i18n

// Helper — unpack postMessage wrapper
async function kg(url, ...rest) {
    const r = await App.fetch(url, ...rest);
    return r?.data ?? r;
}
function absUrl(path) {
    if (!path || /^https?:\/\//i.test(path)) return path;
    const d = window.App?.requestParams?.domain || '';
    return d ? 'https://' + d + (path.startsWith('/') ? path : '/' + path) : path;
}
```

---

## SWR Cache for Tabs (stale-while-revalidate)

Responsive UI: show cached data instantly → re-fetch in background → if changed, re-render. Cache in `sessionStorage` (scoped to iframe session, cleared on tab close, survives tab switching inside Hub).

```js
// core/swr.js — minimal implementation
function _hash(d) { try { return JSON.stringify(d); } catch (e) { return String(Math.random()); } }
function _read(k)   { try { const r = sessionStorage.getItem(k); return r ? JSON.parse(r) : null; } catch (e) { return null; } }
function _write(k, d) { try { sessionStorage.setItem(k, JSON.stringify(d)); } catch (e) {} }

async function swr(key, fetcher, render) {
    const stale = _read(key);
    if (stale !== null) {
        try { render(stale); } catch (e) { console.warn('[swr] stale render:', e); }
    }
    try {
        const fresh = await fetcher();
        if (_hash(fresh) !== (stale !== null ? _hash(stale) : null)) {
            _write(key, fresh);
            try { render(fresh); } catch (e) { console.warn('[swr] fresh render:', e); }
        }
    } catch (e) { console.warn('[swr] fetcher failed:', key, e); }
}

// Invalidate cache by prefix (after mutating actions)
function swrInvalidate(prefix) {
    const keys = [];
    for (let i = 0; i < sessionStorage.length; i++) {
        const k = sessionStorage.key(i);
        if (k && k.startsWith(prefix)) keys.push(k);
    }
    keys.forEach(k => sessionStorage.removeItem(k));
}
```

Usage:
```js
// In a tab module
async function renderBalance() {
    await swr('kg:balance', () => kg('/api/korgames/balance'), (r) => {
        document.getElementById('balance').textContent = r?.data?.corn ?? '—';
    });
}

// After claiming a quest — invalidate:
await kg('/api/korgames/quest/claim', { method: 'POST', body: { quest_id: 30 } });
swrInvalidate('kg:balance');
swrInvalidate('kg:quests');
swrInvalidate('kg:lb:activity');
await renderBalance();
```

**Important:**
- Don't cache `Map`/`Set` objects — `JSON.stringify(new Map())` = `{}`, hash always the same, diff breaks. Store arrays/plain objects.
- SWR is ideal for GET endpoints. For POST (claim, buy) — explicit fetch + swrInvalidate for related caches.

---

## i18n — Language Switching Across Frames

Three storage channels, priority: URL `?lang=X` → `localStorage` → `App.storage`. URL gives instant synchronization between frames, localStorage — shared between same-origin iframes, App.storage — long-lived (on server).

```js
// core/i18n.js — entry point
const i18n = (function() {
    const SUPPORTED = ['en', 'ru'];
    const DEFAULT = 'en';
    let current = DEFAULT, dict = {}, app = null;

    function resolve(obj, path) {
        return path.split('.').reduce((o, k) => (o && o[k] !== undefined) ? o[k] : null, obj);
    }
    function format(s, p) {
        if (!p || typeof s !== 'string') return s;
        return s.replace(/\{(\w+)\}/g, (_, k) => p[k] !== undefined ? p[k] : '{' + k + '}');
    }
    async function loadDict(lang) {
        try { return await (await fetch('../locales/' + lang + '.json')).json(); }
        catch (e) { return {}; }
    }

    return {
        async init(appInstance) {
            app = appInstance;
            let saved = DEFAULT;
            try {
                const urlLang = new URLSearchParams(window.location.search).get('lang');
                if (urlLang && SUPPORTED.includes(urlLang)) saved = urlLang;
                else {
                    let ls = null;
                    try { ls = localStorage.getItem('kg:lang'); } catch (e) {}
                    if (ls && SUPPORTED.includes(ls)) saved = ls;
                    else {
                        const s = await app.storage.get('lang');
                        if (s && SUPPORTED.includes(s)) saved = s;
                    }
                }
            } catch (e) {}
            current = saved;
            dict = await loadDict(current);
        },
        t(key, params) {
            const v = resolve(dict, key);
            return v === null ? key : format(v, params);
        },
        current() { return current; },
        supported() { return SUPPORTED.slice(); },
        async setLang(lang) {
            if (!SUPPORTED.includes(lang)) return;
            current = lang;
            dict = await loadDict(lang);
            try { localStorage.setItem('kg:lang', lang); } catch (e) {}
            try {
                const u = new URL(window.location.href);
                u.searchParams.set('lang', lang);
                history.replaceState(null, '', u.toString());
            } catch (e) {}
            if (app) { try { await app.storage.set('lang', lang); } catch (e) {} }
        },
        applyToDom(root) {
            root.querySelectorAll('[data-i18n]').forEach(el => {
                el.textContent = this.t(el.getAttribute('data-i18n'));
            });
            root.querySelectorAll('[data-i18n-attr]').forEach(el => {
                el.getAttribute('data-i18n-attr').split(',').forEach(pair => {
                    const [attr, key] = pair.split(':').map(s => s.trim());
                    if (attr && key) el.setAttribute(attr, this.t(key));
                });
            });
        }
    };
})();
window.i18n = i18n;
```

Usage:
```html
<!-- In HTML — mark elements with data-i18n -->
<h1 data-i18n="app.title">App</h1>
<button data-i18n="game.start">Start</button>
<input data-i18n-attr="placeholder:form.hint" placeholder="">

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
window.App = new VMCRMUserApp();
await App.getRequestParams();  // important BEFORE i18n.init
await i18n.init(App);
i18n.applyToDom(document);
</script>
```

Language switcher:
```js
for (const code of i18n.supported()) {
    const btn = document.createElement('button');
    btn.textContent = code.toUpperCase();
    if (code === i18n.current()) btn.classList.add('active');
    btn.addEventListener('click', async () => {
        await i18n.setLang(code);
        i18n.applyToDom(document);
    });
    container.appendChild(btn);
}
```

Locale files (`locales/en.json` and `locales/ru.json`) — hierarchical keys:
```json
{
    "app": { "title": "My Game" },
    "game": { "start": "Start", "score": "Score: {n}" }
}
```

Parameters — `i18n.t('game.score', {n: 42})` → `"Score: 42"`.

---

## Awarding Korn to a User

**Cannot be done from client-side JS directly.** Games do not mint Korn — this is by design (see [concepts.md § Emission](concepts.md)).

**You can:**

1. **Create a quest** in `sys_quests` (via SQL migration or `/korgames/install`), with `condition_type` from whitelist:
   - `login`, `streak` — triggered automatically
   - `create_record` — on INSERT in whitelisted catalogs (accounts, projects, tt_tasks, ...)
   - `game_play`, `game_score` — on `POST /api/korgames/game/score`
   - `deploy_app`, `install_app` — on marketplace events
   - `referral` — on invited user registration

2. **User claims themselves:**
```js
const r = await kg('/api/korgames/quest/claim', { method: 'POST', body: { quest_id: 30 } });
if (r.status === 'success') {
    // r.earn_result.earned, r.earn_result.balance
} else if (r.error === 'daily_cap_exceeded') {
    // 500 Korn/day limit reached
}
```

**For server-side admin emission** (edge case, outside miniapp) — direct `\korgames\Games::earnCorn($authorId, $amount, 'admin', 'reason')` in PHP.

---

## Spending Korn (Purchase)

**Via item shop** (atomic transaction on server):

```js
const r = await kg('/api/korgames/game/buy', {
    method: 'POST',
    body: { game_id: 'coin-clicker', item_key: 'gold_cursor' }
});

if (r.status === 'success') {
    // r.price_corn, r.balance_after, r.purchase_id
} else {
    // r.error: insufficient_balance | limit_exceeded | item_not_found | game_not_active
    alert('Error: ' + r.error);
}
```

`max_per_user=1` (in config.korgames.items) prevents duplicates for unique upgrades.

**Direct spend without item** — from PHP: `\korgames\Games::spendCorn($authorId, $amount, 'reason', 'source_id')`. Intentionally not exposed to client.

---

## Shop — Load Items + Balance + Inventory in One Shot

Three parallel requests, then render:

```js
async function loadShop(gameId) {
    const [items, bal, inv] = await Promise.all([
        kg('/api/korgames/game/items?game_id=' + encodeURIComponent(gameId)),
        kg('/api/korgames/balance'),
        kg('/api/korgames/game/inventory?game_id=' + encodeURIComponent(gameId)),
    ]);
    return {
        items:   items?.data || [],
        balance: bal?.data?.corn ?? 0,
        owned:   new Set((inv?.data || []).map(p => p.item_key))
    };
}

// Render "Buy" button
const canBuy = balance >= item.price_corn && !owned.has(item.item_key);
```

After purchase — **re-fetch** all three (balance/inventory changed).

---

## Displaying Balance + Streak in Header

```js
const b = (await kg('/api/korgames/balance'))?.data;
if (b) {
    document.querySelector('#pill-corn').textContent = b.corn + ' Korn';
    document.querySelector('#pill-streak').textContent = b.current_streak > 0 ? '🔥 ' + b.current_streak : '';
    document.querySelector('#daily-left').textContent = b.daily_remaining + ' left today';
}
```

---

## Recording Score After a Round + Showing Result

```js
async function finishRound(score, durationSec) {
    const r = await kg('/api/korgames/game/score', {
        method: 'POST',
        body: { game_id: 'my-game', score, duration: durationSec }
    });
    if (r.status === 'success') {
        // Update personal record (if you want a local cache)
        // Server is source of truth, re-fetch on next init
    }
    return r;
}
```

Then show a result modal, or deep-link with `?section=leaderboard` to open the top scores.

---

## Top Players for a Specific Game (per-game leaderboard)

Via `/db/sys_game_scores.json` + collapse to max score per user + JOIN with profiles (2 requests):

```js
async function topScores(gameId, limit = 10) {
    // 1. Last N scores descending
    const scoreR = await kg(
        '/db/sys_game_scores.json?form[game_id]=' + encodeURIComponent(gameId)
        + '&orderby=score+DESC&limit=200'
    );
    const rows = scoreR?.data || scoreR?.data?.data || [];

    // 2. Collapse to 1 record per author (max score)
    const byAuthor = new Map();
    for (const r of rows) {
        const aid = +r.author_id;
        const sc = +r.score;
        if (!byAuthor.has(aid) || byAuthor.get(aid).score < sc) {
            byAuthor.set(aid, { author_id: aid, score: sc, played_at: r.played_at });
        }
    }
    const top = [...byAuthor.values()].sort((a, b) => b.score - a.score).slice(0, limit);

    // 3. Profiles (display_name + avatar) in one request
    const profR = await kg('/db/sys_game_profiles.json?limit=500');
    const profiles = new Map();
    const profRows = profR?.data || [];
    for (const p of profRows) {
        profiles.set(+p.author_id, { display_name: p.display_name, avatar_url: p.avatar_url });
    }

    return top.map((u, i) => ({
        rank: i + 1,
        ...u,
        display_name: profiles.get(u.author_id)?.display_name || 'user ' + u.author_id,
        avatar_url:   profiles.get(u.author_id)?.avatar_url || '',
    }));
}
```

**Requires permissions in config.json:**
```json
"permissions": {
    "catalogs": {
        "sys_game_scores":   ["read"],
        "sys_game_profiles": ["read"]
    }
}
```

---

## Game Profile — Read + Write

**Read:**
```js
const p = (await kg('/api/korgames/profile'))?.data;
const name = p?.display_name || 'Anonymous';
const avatar = p?.avatar_url ? absUrl(p.avatar_url) : null;
```

**Write (without avatar):**
```js
await kg('/api/korgames/profile', {
    method: 'POST',
    body: {
        display_name: 'ProGamer42',
        bio: 'Top clicker since 2025',
        avatar_url: p?.avatar_url || ''  // don't change if not updated
    }
});
```

### Header profile strip (canonical)

Name + avatar + "edit profile" link to the Games Hub. **The three spots games
systematically get wrong** — collected in one place:

```js
async function renderProfileStrip() {
    const p = (await kg('/api/korgames/profile'))?.data || {};

    // 1. NAME: display_name only. The response has NO name/username field —
    //    falling back to them yields a permanent "Anonymous".
    const name = p.display_name || 'Anonymous';

    // 2. AVATAR: absUrl is mandatory. avatar_url arrives as '/reimg/...'
    //    relative to the store domain → without absUrl it 404s.
    nameEl.textContent = name;
    if (p.avatar_url) {
        avatarEl.innerHTML = `<img src="${absUrl(p.avatar_url)}" alt="${name}">`;
    } else {
        avatarEl.textContent = name.slice(0, 2).toUpperCase();
    }

    // 3. EDIT LINK: goes to the Games Hub, profile tab.
    //    Correct URL — /db/installed_apps/{alias}?frame=main&tab=profile
    //    (NOT /app/{alias}/profile and NOT /{alias}?tab=profile — those 404).
    editLink.addEventListener('click', (e) => { e.preventDefault(); goToProfileTab(); });
}
```

`absUrl` — from [client-api.md](client-api.md#absolute-urls). `goToProfileTab` — in the
[Cross-app discovery](#cross-app-discovery-find-another-game-hub-etc) section below.

---

## Avatar Upload (drag-drop or picker)

Client resize to 256×256 center-crop → base64 → POST.

```js
async function uploadAvatar(file) {
    if (!file.type.startsWith('image/')) throw new Error('not an image');

    // 1. Read file
    const dataUrl = await new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = () => resolve(reader.result);
        reader.onerror = reject;
        reader.readAsDataURL(file);
    });

    // 2. Resize in canvas 256×256 center-crop
    const img = new Image();
    await new Promise(r => { img.onload = r; img.src = dataUrl; });
    const canvas = document.createElement('canvas');
    canvas.width = canvas.height = 256;
    const ctx = canvas.getContext('2d');
    const sq = Math.min(img.width, img.height);
    ctx.drawImage(img, (img.width - sq) / 2, (img.height - sq) / 2, sq, sq, 0, 0, 256, 256);

    const outType = file.type === 'image/png' ? 'image/png' : 'image/jpeg';
    const resized = canvas.toDataURL(outType, 0.9);

    // 3. Upload
    const r = await kg('/api/korgames/profile/avatar', {
        method: 'POST', body: { avatar_base64: resized }
    });
    return r.avatar_url;  // /reimg/data/db/f_sys_game_profiles/...
}

// Drag-and-drop handler
const zone = document.getElementById('drop-zone');
zone.addEventListener('drop', async (e) => {
    e.preventDefault();
    if (e.dataTransfer.files.length) {
        const url = await uploadAvatar(e.dataTransfer.files[0]);
        document.getElementById('preview').src = absUrl(url);
    }
});
zone.addEventListener('dragover', e => e.preventDefault());
```

**Display with resize:**
```html
<img src="https://{domain}/reimg/data/db/f_sys_game_profiles/avatar_3_abc.png?80x80">
```

Must use `absUrl()` — iframe is not on the CRM domain.

---

## Deep-link "Show Game Top on Open"

Games Hub when clicking in "By Games" mode opens the game with `?section=leaderboard`. Your game can read the URL:

```js
const params = new URLSearchParams(window.location.search);
if (params.get('section') === 'leaderboard') {
    // Immediately show your top scores
    TopScoresModal.show();
}
```

This is a convention — all games support `?section=leaderboard` for consistent UX.

---

## Idempotent Install on First Open

Calling `/install` on every replay is wasteful. Cache it:

```js
try {
    const installed = await App.storage.get('install_status');
    if (!installed || !installed.at) {
        const r = await kg('/api/korgames/install', { method: 'POST' });
        if (r?.status === 'success') {
            await App.storage.set('install_status', {
                at: new Date().toISOString(),
                corn_init: r.data?.balance?.corn ?? 0
            });
        }
    }
} catch (e) { /* not critical */ }
```

**Backup:** if `App.storage` was reset — install is still idempotent, nothing breaks.

---

## Error Handling — Unified Pattern

```js
async function safeCall(promise, label) {
    try {
        const r = await promise;
        if (r?.status !== 'success') {
            console.warn('[' + label + '] not success:', r?.error || r?.message);
            return { ok: false, error: r?.error || r?.message || 'unknown' };
        }
        return { ok: true, data: r.data };
    } catch (e) {
        console.error('[' + label + '] exception:', e);
        return { ok: false, error: 'network_error' };
    }
}

// Usage
const res = await safeCall(kg('/api/korgames/balance'), 'balance');
if (!res.ok) {
    App.alert('Error: ' + res.error);
    return;
}
// res.data.corn ...
```

---

## Cross-App Discovery (find another game, Hub, etc.)

By `package` + optionally `system=1`:

```js
async function findAppByPackage(pkg, systemOnly = false) {
    const q = '/db/marketplace.json?form[package]=' + encodeURIComponent(pkg)
            + (systemOnly ? '&form[system]=1' : '') + '&limit=1';
    const r = await kg(q);
    // r = {status, data: {total, data: [...]}} — the rows are in the nested data
    const rows = r?.data?.data || r?.data || [];
    return Array.isArray(rows) ? rows[0] : null;
}

// Find Games Hub and open its main frame
const hub = await findAppByPackage('games-hub', true);
if (hub) {
    const inst = await kg('/db/installed_apps.json?form[app_id]=' + hub.id + '&limit=1');
    // inst?.data = {total, data: [...]} — use inst?.data?.data
    const installed = (inst?.data?.data || inst?.data || [])[0];
    if (installed?.alias) {
        App.navigate('/db/installed_apps/' + installed.alias + '?frame=main');
    } else {
        App.navigate('/db/marketplace/' + hub.alias);
    }
}

// Open the Games Hub profile tab directly
async function goToProfileTab() {
    const hub = await findAppByPackage('games-hub', true);
    if (!hub) { App.navigate('/db/sys_game_profiles'); return; }
    const inst = await kg('/db/installed_apps.json?form[app_id]=' + hub.id + '&limit=1');
    const installed = (inst?.data?.data || inst?.data || [])[0];
    if (installed?.alias) {
        App.navigate('/db/installed_apps/' + installed.alias + '?frame=main&tab=profile');
    } else {
        App.navigate('/db/marketplace/' + hub.alias);
    }
}
```

**Permissions:**
```json
"catalogs": { "marketplace": ["read"], "installed_apps": ["read"] }
```

---

## What to Read Next

- [styling.md](styling.md) — how a game should look to fit the platform and the gaming mood
- [checklist.md](checklist.md) — before deploy
- [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — full reference applying all recipes
