# Coin Clicker — Reference Game Walkthrough

> **Navigation:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · **you are here** · [checklist](checklist.md).

A minimal complete game demonstrating all MVP mechanics: game registration, shop, score recording, inventory, purchase. As you read — links to the relevant sections of [config-korgames.md](config-korgames.md) and [client-api.md](client-api.md).

**Marketplace id = 109, alias = `coin-clicker`**.

Source files: `etalon-apps/coin-clicker/` — copy as a template for a new game.

---

## Files

```
coin-clicker/
├── config.json          — metadata + korgames section
├── index.html           — UI + module script with VMCRMUserApp bootstrap
├── icon.svg             — marketplace icon
├── css/
│   └── style.css        — coin + shop modal
└── js/
    ├── game.js          — CC object (gameplay)
    └── shop.js          — Shop object (shop)
```

---

## config.json

```json
{
    "name": "Coin Clicker",
    "alias": "coin-clicker",
    "version": "1.0.1",
    "description": "A clicker for Korn — 30 seconds, leaderboard and upgrade shop",
    "about": "## What it does\n30-second clicker...",
    "tags": "game, clicker, demo, korn",
    "logo": "icon.svg",
    "urls": { "main": "index.html" },
    "urlsConf": { "main": { "method": "get" } },
    "permissions": {
        "storage": false,
        "navigate": false,
        "modal": true
    },
    "korgames": {
        "game_id": "coin-clicker",
        "reward_mode": "score_only",
        "items": [
            {
                "key": "gold_cursor",
                "name": "Gold Cursor",
                "description": "Each click gives 2 points instead of 1",
                "price_corn": 300,
                "max_per_user": 1
            },
            {
                "key": "lucky_shine",
                "name": "Lucky Shine",
                "description": "Every 10th click = +5 points",
                "price_corn": 150,
                "max_per_user": 1
            }
        ]
    }
}
```

What the platform does on install:
- Creates `sys_registered_games.alias='coin-clicker'`, reward_mode='score_only'.
- Creates 2 records in `sys_game_items` — gold_cursor (300), lucky_shine (150).

Full list of section fields — [config-korgames.md](config-korgames.md).

---

## index.html — Bootstrap

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Coin Clicker</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
<div class="cc-app">
    <h1>Coin Clicker</h1>
    <div class="cc-coin-wrap"><div class="cc-coin" id="cc-coin">K</div></div>
    <p class="cc-score">Score: <span id="cc-score">0</span></p>
    <p class="cc-timer">Time: <span id="cc-timer">30</span>s</p>
    <button id="cc-start" class="cc-btn">Start</button>
    <button id="cc-shop" class="cc-btn">Shop</button>
    <div id="cc-shop-modal" class="cc-modal hidden">
        <div class="cc-modal-content">
            <h2>Upgrade Shop</h2>
            <div id="cc-items"></div>
            <button id="cc-close-shop" class="cc-btn">Close</button>
        </div>
    </div>
</div>

<!-- Regular scripts — declare CC and Shop objects -->
<script src="js/game.js"></script>
<script src="js/shop.js"></script>

<!-- Module script — bootstrap VMCRMUserApp and attach handlers -->
<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
window.App = new VMCRMUserApp();

await CC.init();
document.getElementById('cc-coin').addEventListener('click', () => CC.click());
document.getElementById('cc-start').addEventListener('click', () => CC.start());
document.getElementById('cc-shop').addEventListener('click', () => Shop.show());
document.getElementById('cc-close-shop').addEventListener('click', () => Shop.hide());
</script>
</body>
</html>
```

**Key points:**
- Regular `<script>` tags must be BEFORE the module script: they execute synchronously in order, declaring `CC` and `Shop` as globals.
- The `type="module"` script is deferred until after parsing, then imports VMCRMUserApp and attaches handlers.
- `window.App = new VMCRMUserApp()` — global App, used by CC/Shop. See [miniapps/js-api.md](../miniapps/js-api.md).
- **The path `/templates/def/db/marketplace/vmcrm-user-app.js` is always absolute.** Don't change it.

---

## js/game.js — Gameplay

```js
const CC = {
    score: 0,
    running: false,
    timeLeft: 30,
    startTime: 0,
    multiplier: 1,      // gold_cursor effect
    luckyShine: false,  // lucky_shine effect
    clicks: 0,

    // Helper: unpack postMessage wrapper from App.fetch (see client-api.md § wrapper)
    async _api(url, ...rest) {
        const r = await App.fetch(url, ...rest);
        return r?.data ?? r;
    },

    // Read inventory on startup — find out which upgrades are purchased
    async init() {
        const inv = await this._api('/api/korgames/game/inventory?game_id=coin-clicker');
        const keys = (inv.data || []).map(p => p.item_key);
        this.multiplier = keys.includes('gold_cursor') ? 2 : 1;
        this.luckyShine = keys.includes('lucky_shine');
    },

    start() {
        this.score = 0;
        this.clicks = 0;
        this.timeLeft = 30;
        this.startTime = Date.now();
        this.running = true;
        document.getElementById('cc-score').textContent = '0';
        document.getElementById('cc-timer').textContent = '30';
        this.tick();
    },

    tick() {
        if (!this.running) return;
        const elapsed = (Date.now() - this.startTime) / 1000;
        this.timeLeft = Math.max(0, 30 - Math.floor(elapsed));
        document.getElementById('cc-timer').textContent = this.timeLeft;
        if (this.timeLeft <= 0) { this.finish(); return; }
        requestAnimationFrame(() => this.tick());
    },

    click() {
        if (!this.running) return;
        this.clicks++;
        let add = this.multiplier;
        if (this.luckyShine && this.clicks % 10 === 0) add += 5;
        this.score += add;
        document.getElementById('cc-score').textContent = this.score;
        // Animation
        const coin = document.getElementById('cc-coin');
        coin.classList.remove('clicking');
        void coin.offsetWidth;
        coin.classList.add('clicking');
    },

    async finish() {
        this.running = false;
        const duration = Math.round((Date.now() - this.startTime) / 1000);
        await this._api('/api/korgames/game/score', {
            method: 'POST',
            body: { game_id: 'coin-clicker', score: this.score, duration: duration }
        });
        alert(`Game over! Score: ${this.score}. Saved to leaderboard.`);
    }
};
```

**What happens here:**
1. `init()` — on miniapp start, read inventory. If `gold_cursor` is purchased — x2 multiplier. If `lucky_shine` — every 10th click gives +5 bonus.
2. `start()` — 30-second round with a requestAnimationFrame timer.
3. `click()` — add points + animation.
4. `finish()` — send score to server. Result is recorded in `sys_game_scores`.

---

## js/shop.js — Shop

```js
const Shop = {
    async _api(url, ...rest) {
        const r = await App.fetch(url, ...rest);
        return r?.data ?? r;
    },

    async show() {
        const modal = document.getElementById('cc-shop-modal');
        const box = document.getElementById('cc-items');

        // Load items, balance, and inventory in parallel
        const [itemsR, balR, invR] = await Promise.all([
            this._api('/api/korgames/game/items?game_id=coin-clicker'),
            this._api('/api/korgames/balance'),
            this._api('/api/korgames/game/inventory?game_id=coin-clicker'),
        ]);

        const owned = new Set((invR.data || []).map(p => p.item_key));

        box.innerHTML = (itemsR.data || []).map(it => {
            const has = owned.has(it.item_key);
            const canBuy = balR.data && balR.data.corn >= it.price_corn && !has;
            return `<div class="cc-item">
                <strong>${it.name}</strong> — ${it.price_corn} Korn
                <p>${it.description || ''}</p>
                <button class="cc-btn cc-buy" data-k="${it.item_key}" ${canBuy?'':'disabled'}>
                    ${has ? 'Purchased' : 'Buy'}
                </button>
            </div>`;
        }).join('');

        box.querySelectorAll('.cc-buy').forEach(b => b.addEventListener('click', async () => {
            const r = await Shop._api('/api/korgames/game/buy', {
                method: 'POST',
                body: { game_id: 'coin-clicker', item_key: b.dataset.k }
            });
            if (r.status === 'success') {
                await CC.init();  // recalculate effects
                Shop.show();      // re-render shop (will show "Purchased")
            } else {
                alert('Error: ' + (r.error || 'unknown'));
            }
        }));
        modal.classList.remove('hidden');
    },

    hide() { document.getElementById('cc-shop-modal').classList.add('hidden'); }
};
```

**Logic:**
1. When the shop opens, three endpoints are requested in parallel.
2. For each item a button is built: "Buy" if enough Korn and not yet purchased, "Purchased" if already owned, disabled if not enough Korn.
3. After purchase — recalculate effects (`CC.init()`) and re-render the shop.

---

## Player Lifecycle

1. User opens Games Hub → earns Korn via quests (daily_login +10, daily_play_5 +20, etc).
2. Installs Coin Clicker from the marketplace → the `on_app_installed` hook registers the game.
3. Opens Coin Clicker → `CC.init()` reads inventory (empty at first) — multiplier=1.
4. Clicks for 30 sec → `CC.finish()` → score is written to sys_game_scores, quest `daily_play_5` makes progress.
5. Opens the shop → sees gold_cursor (300 Korn) and lucky_shine (150 Korn).
6. If balance ≥ 300 — buys gold_cursor → platform atomically deducts Korn, writes transaction and purchase.
7. `CC.init()` after purchase sees gold_cursor in inventory → multiplier=2.
8. Next game — clicks give 2 points.

---

## Deploy

```bash
# package
cd /tmp/coin-clicker
zip -rq /tmp/coin-clicker.zip config.json index.html icon.svg css/ js/

# deploy (update + refresh in one call)
curl -X POST "https://vibe.korfix.app/api/marketplace/deploy/109" \
  -H "Authorization: Bearer $KORFIX_TOKEN" \
  -F "doc1=@/tmp/coin-clicker.zip;type=application/zip"
```

Deploying a new app (when there's no id yet):

```bash
curl -X POST "https://vibe.korfix.app/api/db/marketplace" \
  -H "Authorization: Bearer $KORFIX_TOKEN" \
  -F 'name=Coin Clicker' \
  -F 'category=games' \
  -F "doc1=@/tmp/coin-clicker.zip;type=application/zip"
# → {"status":"success","id":"N","alias":"..."}
# Save N, use /api/marketplace/deploy/N from now on
```

---

## What to Add Next (Phase 2)

- Top-10 players by score after a round (currently just alert). Endpoint `/db/sys_game_scores.json?form[game_id]=coin-clicker&orderby=score+DESC&limit=10` — see [client-api.md § game leaderboard](client-api.md).
- Save personal best in `App.storage.set('best_score', ...)` and show on start — see [miniapps/storage-and-hooks.md](../miniapps/storage-and-hooks.md).
- More items: ×3 multiplier, auto-click every 5 seconds, +15 seconds round time. Add to `config.korgames.items` — [config-korgames.md](config-korgames.md).
- Sound effects (be careful — autoplay policy inside iframes).
- "Refer a friend" — custom quest with condition_type=referral (see [concepts.md § Creating Your Own Quest](concepts.md)).

The template is ready — take it and build your game. Before deploying — run through [checklist.md](checklist.md).
