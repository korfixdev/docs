# Modular Game Miniapp Structure

> **Navigation:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · **you are here** · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · [checklist](checklist.md).

As a game grows, a flat `index.html + js/{api,game,shop}.js` structure quickly becomes unmanageable. Below is the pattern used by the reference Games Hub and Coin Clicker.

---

## Directory Pattern

```
my-game/
├── config.json            — metadata + korgames section (required)
├── icon.svg
├── README.md              — project structure, change history
├── frames/                — HTML pages (referenced in config.urls)
│   ├── main.html
│   └── settings.html      — optional, if there's a "gear" settings entry
├── core/                  — reusable singletons
│   ├── api.js             — wrapper over App.fetch to /api/korgames/*
│   ├── i18n.js            — mini-i18n for EN/RU (see below)
│   └── storage.js         — safe-wrappers for App.storage
├── modules/               — UI logic, one file per section/modal
│   ├── game.js
│   ├── shop.js
│   ├── top-scores.js
│   └── profile.js
├── locales/
│   ├── en.json            — default language
│   └── ru.json
└── styles/
    └── style.css
```

In `config.json`:

```json
"urls": {
    "main":     "frames/main.html",
    "settings": "frames/settings.html"
},
"urlsConf": {
    "main":     { "method": "get" },
    "settings": { "method": "get" }
}
```

---

## Why frames ≠ html in Root

1. **Readability** — it's clear that multiple frames are possible, and they're all together.
2. **Scalability** — easy to add `frames/widget.html` and `"widget": "frames/widget.html"` to config.urls.
3. **Paths** — relative `../core/api.js` works without surprises regardless of depth.

---

## Modular Loading Rules

Inside each frame:

```html
<!-- 1. Regular scripts — declare globals -->
<script src="../core/api.js"></script>
<script src="../core/i18n.js"></script>
<script src="../core/storage.js"></script>
<script src="../modules/game.js"></script>
<script src="../modules/shop.js"></script>

<!-- 2. Module script — bootstrap -->
<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
window.App = new VMCRMUserApp();

await i18n.init(App);
i18n.applyToDom(document);

// Event wiring
await CC.init();
document.getElementById('cc-start').addEventListener('click', () => CC.start());
</script>
```

Order matters:
- Regular scripts load **before** the module script is parsed (the latter is deferred).
- Objects from regular scripts must be on `window` (or `const X = {}; window.X = X;`) so the module bootstrap can see them.
- Async initialization (`i18n.init`, `CC.init`) — only in the module script, because regular scripts cannot use top-level `await`.

---

## i18n — Minimal Implementation

Use the same `core/i18n.js` as in Games Hub / Coin Clicker. It:

1. Stores language in `App.storage.get('lang')` (per-user, per-app).
2. Loads `locales/{lang}.json` via `fetch(...)`.
3. Resolves keys by dot: `t('shop.price', {n: 300})` → `'300 Korn'`.
4. `applyToDom(root)` replaces all elements with `data-i18n="key"` and `data-i18n-attr="title:key"`.

### Keys

Hierarchy by UI sections:

```json
{
    "app":     { "title": "...", "loading": "..." },
    "tabs":    { "balance": "...", "quests": "..." },
    "balance": { "title": "...", "corn": "...", "streak": "..." },
    "shop":    { "title": "...", "buy": "...", "price": "{n} Korn" }
}
```

Parameters — `{placeholder}`, passed as second argument: `i18n.t('shop.price', {n: 300})`.

### Markup

```html
<h2 data-i18n="shop.title">Shop</h2>
<button data-i18n="shop.buy">Buy</button>
<input placeholder="" data-i18n-attr="placeholder:profile.name_hint">
```

After `i18n.applyToDom(document)` — `textContent` and attributes are replaced.

### Language Switcher

```js
const langSwitch = document.getElementById('lang-switch');
for (const code of i18n.supported()) {
    const b = document.createElement('button');
    b.textContent = code.toUpperCase();
    if (code === i18n.current()) b.classList.add('active');
    b.addEventListener('click', async () => {
        await i18n.setLang(code);
        i18n.applyToDom(document);
        // Re-render custom blocks (innerHTML with t()):
        await CC.refreshAll();
    });
    langSwitch.appendChild(b);
}
```

Important: `applyToDom` only works with `data-i18n` tags. If you have `root.innerHTML = \`${i18n.t('key')}\``, you must **re-render** that block manually on language change.

### Adding a New Language

1. Copy `locales/en.json` → `locales/de.json`, translate.
2. In `core/i18n.js` add `'de'` to `const SUPPORTED = ['en', 'ru']`.
3. Add language name key in EN/RU files: `settings.lang_de`.

---

## core/api.js — App.fetch Wrapper

```js
const KgApi = {
    async _call(url, ...rest) {
        const r = await App.fetch(url, ...rest);
        return r?.data ?? r;           // unpack postMessage wrapper
    },
    // Relative → absolute URL (iframe on store, resources on CRM)
    absUrl(path) {
        if (!path || /^https?:\/\//i.test(path)) return path;
        const domain = window.App?.requestParams?.domain || '';
        if (!domain) return path;
        return 'https://' + domain.replace(/\/$/, '')
             + (path.startsWith('/') ? path : '/' + path);
    },
    balance:    ()  => KgApi._call('/api/korgames/balance'),
    getItems:   (g) => KgApi._call('/api/korgames/game/items?game_id=' + encodeURIComponent(g)),
    sendScore:  (p) => KgApi._call('/api/korgames/game/score', {method: 'POST', body: p}),
    // ...
};
window.KgApi = KgApi;
```

Important:
- Rest spread (`...rest`) to avoid passing `undefined` as second argument (breaks the host, see [client-api.md](client-api.md)).
- Body — **object**, not `JSON.stringify`. VMCRMUserApp serializes it itself.
- `encodeURIComponent` for values in query strings.
- **absUrl for any platform link in `<img src>`, `<a href>` (not via `App.navigate`), etc.** The iframe lives on the store domain; relative `/reimg/...` resolves to the store → 404. See [client-api.md § Avatar](client-api.md).

---

## core/storage.js — Safe Wrappers

```js
const KgStore = {
    async get(key, defaultValue = null) {
        try {
            const v = await App.storage.get(key);
            return (v === undefined || v === null) ? defaultValue : v;
        } catch (e) { return defaultValue; }
    },
    async set(key, value) {
        try { await App.storage.set(key, value); return true; } catch (e) { return false; }
    }
};
window.KgStore = KgStore;
```

Why: `App.storage.*` sometimes throws on missing values — wrap in try/catch, always return a value.

---

## modules/* — UI Logic

Each module = one object, exported to `window`. Example:

```js
// modules/profile.js
const KgProfile = {
    async render() {
        const root = document.getElementById('kg-profile-content');
        root.innerHTML = '<p>' + i18n.t('profile.loading') + '</p>';

        const r = await KgApi.getProfile();
        const p = (r?.status === 'success' && r.data) || {};

        root.innerHTML = `
            <form id="kg-profile-form">
                <input name="display_name" value="${escapeAttr(p.display_name || '')}">
                <button type="submit">${i18n.t('profile.save')}</button>
            </form>
        `;
        document.getElementById('kg-profile-form').addEventListener('submit', async (e) => {
            e.preventDefault();
            const payload = Object.fromEntries(new FormData(e.target));
            const r = await KgApi.updateProfile(payload);
            // ... handle response
        });
    }
};
window.KgProfile = KgProfile;
```

Separation: one file = one UI section (tab, modal, installer step). If a file approaches 300 lines — it's a sign to split.

---

## Naming Conventions

- **Global prefixes**: `Kg*` for Games Hub, `Cc*` for Coin Clicker, your own prefix for your game. Avoid conflicts — multiple games can be open simultaneously in different iframes, but globals live within one document.
- **CSS classes**: `.kg-*` / `.cc-*` — avoid collisions with platform styles.
- **i18n keys**: `<section>.<key>`, snake_case inside (`balance.daily_left`, `shop.not_enough`).
- **Event handlers**: when re-rendering a block (`.innerHTML = ...`) handlers are also removed — always re-attach after rendering.

---

## REQUIRED: Don't Touch body

The miniapp iframe is inserted into the platform page — background and font are visually inherited. Setting your own `background` on `body` creates a foreign block.

**Rule:** leave `body` **transparent**:

```css
body {
    margin: 0; padding: 12px;
    font: 400 14px/1.5 "Open Sans", sans-serif;
    color: var(--dark);
    background: transparent;  /* ← required */
}
```

If the game requires its own atmosphere (dark background, gold, neon) — **only inside a container**:

```html
<div class="cc-app">
    <!-- common elements outside the themed frame — topbar, profile -->
    <div class="cc-topbar">...</div>
    <div id="cc-profile-strip"></div>

    <!-- game area in a gamified frame -->
    <div class="cc-frame">
        <h1>Coin Clicker</h1>
        <!-- coin, timer, buttons -->
    </div>
</div>
```

```css
.cc-frame {
    background: linear-gradient(160deg, #fffdf4 0%, #fff5d7 100%);
    border: 1px solid rgba(198, 146, 20, 0.35);
    border-radius: 14px;
    padding: 16px;
    box-shadow: 0 4px 16px rgba(196, 146, 20, 0.12),
                inset 0 0 0 1px rgba(255, 255, 255, 0.7);
}
```

Benefits:
- The game looks like a self-contained element on the platform page.
- Modals/popups outside `.cc-frame` render with neutral body — don't "bleed" the atmosphere everywhere.
- Easy to restyle: change one `.cc-frame` — new arena. body untouched.

Anti-pattern:
```css
body {
    background: linear-gradient(160deg, #fff9e6 0%, #fff3cd 100%);  /* ✗ */
}
```
When the miniapp opens in the platform iframe, the edge of the page will be sharply yellow while the rest of the panel is grey-white. Looks out of place.

Reference — Coin Clicker `styles/style.css` (`.cc-frame` + transparent body).

---

## What to Read Next

- [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — the reference game with all the described structure.
- [checklist.md](checklist.md) — what to check before deploy.
- Source files for reference: `etalon-apps/games-hub/` and `etalon-apps/coin-clicker/`.

---

## Cross-App Discovery — Finding Another Miniapp

When one game needs to know "where is Games Hub" or "where is my companion widget" — **don't hardcode** `marketplace.id` or `marketplace.alias`. They are per-instance and change.

Use `config.package` — this is the logical ID (e.g. `games-hub`, `coin-clicker`). Stored in `marketplace.package`, searchable. System miniapps are additionally marked `marketplace.system = 1` — filtering by this protects against fakes with the same package.

```js
// Find Games Hub in the marketplace
const r = await App.fetch('/db/marketplace.json?form[package]=games-hub&form[system]=1&limit=1');
const hub = r?.data?.[0];

// Is it installed by the user?
if (hub) {
    const inst = await App.fetch('/db/installed_apps.json?form[app_id]=' + hub.id + '&limit=1');
    const installed = inst?.data?.[0];
    if (installed) {
        App.navigate('/db/installed_apps/' + installed.alias + '?frame=main');
    } else {
        App.navigate('/db/marketplace/' + hub.alias);  // marketplace card
    }
}
```

Required permissions in your config.json:
```json
"permissions": {
    "catalogs": {
        "marketplace":     ["read"],
        "installed_apps":  ["read"]
    },
    "navigate": true
}
```

See production code in `coin-clicker/modules/profile-strip.js`.
