# Game Miniapp Styling

> **Navigation:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · [api-reference](api-reference.md) · [recipes](recipes.md) · **you are here** · [checklist](checklist.md).

Brief rules: how to look native in the platform AND keep the game atmosphere.

> **Base miniapp styling** — in [miniapps/styling.md](../miniapps/styling.md) (Korfix CSS variables, buttons, cards). Here — gamedev-specific additions only.

---

## Rule #1 — `body { background: transparent }`

**Required.** The miniapp iframe is embedded inside the platform panel. Setting `background: #fff9e6` (yellow) on `body` gives you a yellow square inside a grey panel — it looks foreign.

All thematic styling goes **inside a container**, not on body. Pattern:

```html
<body>
    <div class="app">
        <!-- Neutral elements: header, topbar, profile strip -->
        <div class="topbar">...</div>
        <div class="profile-strip">...</div>

        <!-- Game arena — all game atmosphere goes HERE -->
        <div class="game-frame">
            <h1>Coin Clicker</h1>
            <div class="coin">...</div>
        </div>
    </div>
</body>
```

```css
body {
    margin: 0; padding: 12px;
    font: 400 14px/1.5 "Open Sans", sans-serif;
    color: var(--dark, #1D1E32);
    background: transparent;  /* ← */
}

/* All atmosphere lives here */
.game-frame {
    background: linear-gradient(160deg, #fffdf4 0%, #fff5d7 100%);
    border: 1px solid rgba(198, 146, 20, 0.35);
    border-radius: 14px;
    padding: 16px;
    box-shadow: 0 4px 16px rgba(196, 146, 20, 0.12),
                inset 0 0 0 1px rgba(255, 255, 255, 0.7);
}
```

Reference — `etalon-apps/coin-clicker/styles/style.css` (`.cc-frame`).

---

## Rule #2 — Korfix CSS Variables as the Base

Colors, typography, buttons — use platform tokens:

```css
:root {
    --primary:   #323C8F;   /* buttons, links */
    --primary-h: #2a327a;   /* hover/active border-bottom */
    --success:   #33BE2B;   /* ok/claim */
    --danger:    #EF233C;   /* errors, negatives */
    --dark:      #1D1E32;
    --gray2:     #78797f;   /* subtitle */
    --gray6:     #eaeaee;   /* dividers */
    --bluegray8: #dce0ef;   /* input border */
    --bluegray10: #f4f5fa;  /* card background */
    --bluegray11: #f8f8fd;  /* backing */
}
```

Game-specific accents on top (gold for Korn, streak-orange):

```css
:root {
    --gold:     #f4c542;
    --gold-d:   #c69214;
    --gold-bg:  #fff3cd;
    --gold-txt: #856404;
    --streak:   #ff6b35;
}
```

**Full token list** — [miniapps/styling.md § CSS Variables](../miniapps/styling.md).

---

## Rule #3 — Containers Must Not Be Empty Squares

Every block needs a soft shadow or border. A plain white card on a plain grey background looks unfinished.

```css
/* Platform card — the minimum */
.card {
    background: #fff;
    border-radius: 8px;
    padding: 16px;
    box-shadow: 0 1px 3px rgba(0,0,0,.08);
}

/* Game card — add accent */
.game-card {
    background: #fff;
    border-radius: 10px;
    padding: 16px;
    border: 1px solid rgba(198, 146, 20, 0.2);
    box-shadow: 0 2px 8px rgba(196, 146, 20, 0.08);
}

/* "Pill" container (for balance, streak) */
.pill {
    display: inline-flex; align-items: center; gap: 8px;
    padding: 8px 14px; border-radius: 24px;
    background: linear-gradient(135deg, var(--gold-bg), #ffe28a);
    color: var(--gold-txt); font-weight: 600;
    border: 1px solid rgba(198, 146, 20, 0.25);
    box-shadow: 0 2px 6px rgba(196, 146, 20, 0.15);
}
```

---

## Rule #4 — Buttons With the Korfix Pattern

Rounded corners of **3px** (not 6/8), `border-bottom: 3px solid darker-color` for a visual "press" effect:

```css
.btn {
    display: inline-flex; align-items: center; gap: 6px;
    padding: 8px 16px;
    border: 0; border-radius: 3px;
    font: 500 13px/1.4 "Open Sans", sans-serif;
    cursor: pointer; transition: .15s;

    background: var(--primary);
    color: #fff;
    border-bottom: 3px solid var(--primary-h);
}
.btn:hover { opacity: .92; }
.btn:active { transform: translateY(1px); border-bottom-width: 2px; }
.btn:disabled { opacity: .5; cursor: not-allowed; }

/* Success (claim, save) */
.btn-success {
    background: var(--success);
    border-bottom-color: #2B9C5B;
}

/* Secondary (back) */
.btn-secondary {
    background: #fff; color: var(--primary);
    border: 1px solid var(--bluegray8);
    border-bottom: 3px solid var(--bluegray8);
}
.btn-secondary:hover { border-color: var(--primary); }

/* Gold (accent for Korn-related actions) */
.btn-gold {
    background: var(--gold); color: #5a3f00;
    border-bottom: 3px solid var(--gold-d);
}
```

---

## Rule #5 — Interactive Feedback

Every clickable element must give visual feedback on press:

```css
.pill-button {
    cursor: pointer;
    transition: transform .08s, box-shadow .12s;
}
.pill-button:hover {
    box-shadow: 0 4px 10px rgba(196, 146, 20, 0.25);
}
.pill-button:active {
    transform: translateY(1px);
    box-shadow: 0 1px 3px rgba(196, 146, 20, 0.2) inset;
}

/* Game coin/clickable object */
.clickable-object { transition: transform .1s; }
.clickable-object:active { transform: scale(0.92); }
.clickable-object.bumped { animation: bump .1s; }
@keyframes bump { 50% { transform: scale(1.2) rotate(10deg); } }
```

Without this the game feels "dead".

---

## Rule #6 — Monospace Font for Numbers

Score, timer, amounts — use `"SF Mono"`, `Menlo`, `monospace`. They won't jitter on change.

```css
.score, .timer, .balance-amount, .transaction-amt {
    font: 700 16px/1 "SF Mono", Menlo, Monaco, "Liberation Mono", monospace;
    color: var(--gold-txt);
}
```

---

## Rule #7 — Modals With Blur

For overlay modals (game over, shop, top scores) — semi-transparent background + blur:

```css
.modal {
    position: fixed; inset: 0;
    background: rgba(0, 0, 0, 0.55);
    backdrop-filter: blur(3px);
    display: flex; align-items: center; justify-content: center;
    padding: 20px; z-index: 100;
}
.modal.hidden { display: none; }
.modal-content {
    background: #fff; padding: 22px; border-radius: 12px;
    max-width: 400px; width: 100%; max-height: 80vh; overflow-y: auto;
    box-shadow: 0 10px 40px rgba(0, 0, 0, 0.25);
}
```

---

## Rule #8 — Avatars Are Round, With a Border

```css
.avatar {
    width: 44px; height: 44px; border-radius: 50%;
    object-fit: cover; background: var(--bluegray10);
    border: 2px solid var(--gold-d);  /* or --bluegray8 for non-gamedev */
    flex-shrink: 0;
}

/* Fallback for missing avatar */
.avatar.avatar-fallback {
    background: radial-gradient(circle at 30% 30%, var(--gold), var(--gold-d));
    position: relative;
}
.avatar.avatar-fallback::after {
    content: '?';
    position: absolute; inset: 0;
    display: flex; align-items: center; justify-content: center;
    color: #5a3f00; font: 700 22px/1 "Open Sans", sans-serif;
}
```

---

## Rule #9 — Drag-Drop Zone for Upload

Visual feedback: dashed border → hover → dragover glow:

```css
.drop-zone {
    width: 120px; height: 120px; border-radius: 50%;
    background: var(--bluegray10);
    border: 2px dashed var(--bluegray8);
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; transition: .15s;
}
.drop-zone:hover {
    border-color: var(--primary); background: var(--bluegray11);
}
.drop-zone.dragover {
    border-color: var(--primary);
    background: #eef0fc;
    box-shadow: 0 0 0 4px rgba(50, 60, 143, 0.1);  /* glow */
}
```

JS:
```js
['dragenter','dragover'].forEach(ev => zone.addEventListener(ev, e => {
    e.preventDefault(); zone.classList.add('dragover');
}));
['dragleave','drop'].forEach(ev => zone.addEventListener(ev, e => {
    e.preventDefault(); zone.classList.remove('dragover');
}));
```

---

## Rule #10 — i18n-Ready Texts

All user-facing strings via `i18n.t('key')`. Even if only EN is supported today — tomorrow they'll ask for RU, and it'll be cheaper to do it right.

```html
<button data-i18n="game.start">Start</button>
<h1 data-i18n="app.title">My Game</h1>
<input placeholder="" data-i18n-attr="placeholder:form.hint">
```

```js
i18n.applyToDom(document);  // after init and after setLang
```

See [project-structure.md § i18n](project-structure.md).

---

## Style Checklist Before Deploy

- [ ] `body { background: transparent }`
- [ ] All game atmosphere inside `.*-frame` or equivalent container
- [ ] Korfix CSS variables in `:root`
- [ ] Buttons follow Korfix pattern (`border-radius: 3px`, `border-bottom: 3px solid darker`)
- [ ] All buttons have `:hover` and `:active` states
- [ ] Score/timer/numbers use monospace font
- [ ] Modals use `backdrop-filter: blur`
- [ ] Avatars are round with fallback
- [ ] Upload zone has dashed border + dragover glow
- [ ] Containers have `box-shadow`, not flat
- [ ] i18n ready — `data-i18n` attributes present

---

## Links

- [miniapps/styling.md](../miniapps/styling.md) — base platform styling (what the platform provides and expects)
- [etalon-apps/coin-clicker/styles/style.css](../../etalon-apps/coin-clicker/styles/style.css) — reference game style (`.cc-frame` etc.)
- [etalon-apps/games-hub/styles/style.css](../../etalon-apps/games-hub/styles/style.css) — reference hub style (tabs, leaderboard, profile)
