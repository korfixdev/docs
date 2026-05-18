# Рецепты gamedev — типовые сценарии

> **Навигация:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · [api-reference](api-reference.md) · **вы здесь** · [styling](styling.md) · [checklist](checklist.md).

Готовые сниппеты для типовых задач. Копируй, подставляй свои значения, деплой.

Все примеры предполагают инициализацию:
```js
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
window.App = new VMCRMUserApp();
await App.getRequestParams();  // обязательно ДО storage ops / i18n

// Хелпер — распаковка postMessage-обёртки
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

## SWR-кеш табов (stale-while-revalidate)

Отзывчивый UI: показываем кешированные данные мгновенно → в фоне перезапрашиваем → если изменилось, перерисовываем. Кеш в `sessionStorage` (scope iframe-сессии, пропадает при закрытии таба, переживает переключение вкладок внутри Hub).

```js
// core/swr.js — минимальная реализация
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

// Инвалидация кеша по префиксу (после мутирующих действий)
function swrInvalidate(prefix) {
    const keys = [];
    for (let i = 0; i < sessionStorage.length; i++) {
        const k = sessionStorage.key(i);
        if (k && k.startsWith(prefix)) keys.push(k);
    }
    keys.forEach(k => sessionStorage.removeItem(k));
}
```

Использование:
```js
// В модуле таба
async function renderBalance() {
    await swr('kg:balance', () => kg('/api/korgames/balance'), (r) => {
        // render code
        document.getElementById('balance').textContent = r?.data?.corn ?? '—';
    });
}

// После claim квеста — инвалидировать:
await kg('/api/korgames/quest/claim', { method: 'POST', body: { quest_id: 30 } });
swrInvalidate('kg:balance');
swrInvalidate('kg:quests');
swrInvalidate('kg:lb:activity');
await renderBalance();
```

**Важно:**
- Не кешируй `Map`/`Set` объекты — `JSON.stringify(new Map())` = `{}`, hash всегда одинаковый, diff сломается. Храни массивы/плоские объекты.
- SWR идеален для GET-endpoints. Для POST (claim, buy) — явный fetch + swrInvalidate для связанных кешей.

---

## i18n — переключение языка между фреймами

Три канала сохранения, приоритет: URL `?lang=X` → `localStorage` → `App.storage`. URL даёт мгновенную синхронизацию между фреймами, localStorage — shared между iframe'ами same-origin, App.storage — долгоживущий (на сервере).

```js
// core/i18n.js — точка входа
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

Использование:
```html
<!-- В HTML — помечаем элементы data-i18n -->
<h1 data-i18n="app.title">App</h1>
<button data-i18n="game.start">Start</button>
<input data-i18n-attr="placeholder:form.hint" placeholder="">

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
window.App = new VMCRMUserApp();
await App.getRequestParams();  // важно ДО i18n.init
await i18n.init(App);
i18n.applyToDom(document);
</script>
```

Переключатель:
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

Локали (`locales/en.json` и `locales/ru.json`) — иерархические ключи:
```json
{
    "app": { "title": "My Game" },
    "game": { "start": "Start", "score": "Score: {n}" }
}
```

Параметры — `i18n.t('game.score', {n: 42})` → `"Score: 42"`.

---

## Начисление Korn юзеру


**Нельзя из клиентского JS напрямую.** Игры не печатают Korn — это by design (см. [concepts.md § Эмиссия](concepts.md)).

**Можно:**

1. **Создать квест** в `sys_quests` (через SQL-миграцию или `/korgames/install`), с `condition_type` из whitelist:
   - `login`, `streak` — автоматически триггерятся
   - `create_record` — при INSERT в whitelisted каталоги (accounts, projects, tt_tasks, ...)
   - `game_play`, `game_score` — при `POST /api/korgames/game/score`
   - `deploy_app`, `install_app` — при событиях маркетплейса
   - `referral` — при регистрации приглашённого юзера

2. **Юзер сам забирает**:
```js
const r = await kg('/api/korgames/quest/claim', { method: 'POST', body: { quest_id: 30 } });
if (r.status === 'success') {
    // r.earn_result.earned, r.earn_result.balance
} else if (r.error === 'daily_cap_exceeded') {
    // 500 Korn/день лимит достигнут
}
```

**Для серверной админской эмиссии** (крайний случай, вне миниапа) — прямой `\korgames\Games::earnCorn($authorId, $amount, 'admin', 'reason')` в PHP.

---

## Списание Korn (покупка)

**Через магазин item'ов** (atomic transaction на сервере):

```js
const r = await kg('/api/korgames/game/buy', {
    method: 'POST',
    body: { game_id: 'coin-clicker', item_key: 'gold_cursor' }
});

if (r.status === 'success') {
    // r.price_corn, r.balance_after, r.purchase_id
} else {
    // r.error: insufficient_balance | limit_exceeded | item_not_found | game_not_active
    alert('Ошибка: ' + r.error);
}
```

`max_per_user=1` (в config.korgames.items) предотвращает дубли для уникальных улучшений.

**Прямой spend без item'а** — из PHP: `\korgames\Games::spendCorn($authorId, $amount, 'reason', 'source_id')`. В клиент не выносится намеренно.

---

## Магазин — загрузить items + баланс + инвентарь одним махом

Параллельно три запроса, потом рендер:

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

// Рендер кнопки "Купить"
const canBuy = balance >= item.price_corn && !owned.has(item.item_key);
```

После покупки — **перезапроси** все три (балансы/инвентарь изменились).

---

## Отображение баланса + стрика в хедере

```js
const b = (await kg('/api/korgames/balance'))?.data;
if (b) {
    document.querySelector('#pill-corn').textContent = b.corn + ' Korn';
    document.querySelector('#pill-streak').textContent = b.current_streak > 0 ? '🔥 ' + b.current_streak : '';
    document.querySelector('#daily-left').textContent = b.daily_remaining + ' left today';
}
```

---

## Запись score после раунда + показать результат

```js
async function finishRound(score, durationSec) {
    const r = await kg('/api/korgames/game/score', {
        method: 'POST',
        body: { game_id: 'my-game', score, duration: durationSec }
    });
    if (r.status === 'success') {
        // Обновить личный рекорд (если хочешь локальный кеш)
        // Сервер — source of truth, перезапроси при следующем init
    }
    return r;
}
```

Потом показываешь модалку результата, либо deep-link'ом `?section=leaderboard` открываешь топ.

---

## Топ игроков конкретной игры (per-game leaderboard)

Через `/db/sys_game_scores.json` + схлопывание до max score per user + JOIN с профилями (2 запроса):

```js
async function topScores(gameId, limit = 10) {
    // 1. Последние N scores по убыванию
    const scoreR = await kg(
        '/db/sys_game_scores.json?form[game_id]=' + encodeURIComponent(gameId)
        + '&orderby=score+DESC&limit=200'
    );
    const rows = scoreR?.data || scoreR?.data?.data || [];

    // 2. Свернуть до 1 записи на автора (max score)
    const byAuthor = new Map();
    for (const r of rows) {
        const aid = +r.author_id;
        const sc = +r.score;
        if (!byAuthor.has(aid) || byAuthor.get(aid).score < sc) {
            byAuthor.set(aid, { author_id: aid, score: sc, played_at: r.played_at });
        }
    }
    const top = [...byAuthor.values()].sort((a, b) => b.score - a.score).slice(0, limit);

    // 3. Профили (display_name + avatar) одним запросом
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

**Требует permissions в config.json:**
```json
"permissions": {
    "catalogs": {
        "sys_game_scores":   ["read"],
        "sys_game_profiles": ["read"]
    }
}
```

---

## Игровой профиль — чтение + запись

**Чтение:**
```js
const p = (await kg('/api/korgames/profile'))?.data;
const name = p?.display_name || 'Anonymous';
const avatar = p?.avatar_url ? absUrl(p.avatar_url) : null;
```

**Запись (без аватара):**
```js
await kg('/api/korgames/profile', {
    method: 'POST',
    body: {
        display_name: 'ProGamer42',
        bio: 'Top clicker since 2025',
        avatar_url: p?.avatar_url || ''  // не трогаем если не меняли
    }
});
```

---

## Загрузка аватара (drag-drop или picker)

Клиентский ресайз до 256×256 center-crop → base64 → POST.

```js
async function uploadAvatar(file) {
    if (!file.type.startsWith('image/')) throw new Error('not an image');

    // 1. Читаем файл
    const dataUrl = await new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = () => resolve(reader.result);
        reader.onerror = reject;
        reader.readAsDataURL(file);
    });

    // 2. Ресайзим в canvas 256×256 center-crop
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

**Отображение с resize:**
```html
<img src="https://{domain}/reimg/data/db/f_sys_game_profiles/avatar_3_abc.png?80x80">
```

Обязательно через `absUrl()` — iframe не на CRM-домене.

---

## Deep-link «показать топ игры при открытии»

Games Hub при клике в режиме «По играм» открывает игру с `?section=leaderboard`. Твоя игра может читать URL:

```js
const params = new URLSearchParams(window.location.search);
if (params.get('section') === 'leaderboard') {
    // Сразу открываем свой топ
    TopScoresModal.show();
}
```

Это convention — все игры поддерживают `?section=leaderboard` для единого UX.

---

## Идемпотентный install при первом открытии

Вызывать `/install` на каждый реплей — расточительно. Кешируй:

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
} catch (e) { /* не critical */ }
```

**Backup:** если `App.storage` сбросился — install всё равно идемпотентен, ничего не сломается.

---

## Обработка ошибок — унифицированный паттерн

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

// Использование
const res = await safeCall(kg('/api/korgames/balance'), 'balance');
if (!res.ok) {
    App.alert('Error: ' + res.error);
    return;
}
// res.data.corn ...
```

---

## Cross-app discovery (найти другую игру, Hub, etc.)

По `package` + опционально `system=1`:

```js
async function findAppByPackage(pkg, systemOnly = false) {
    const q = '/db/marketplace.json?form[package]=' + encodeURIComponent(pkg)
            + (systemOnly ? '&form[system]=1' : '') + '&limit=1';
    const r = await kg(q);
    const rows = r?.data || r?.data?.data || [];
    return Array.isArray(rows) ? rows[0] : null;
}

// Найти Games Hub
const hub = await findAppByPackage('games-hub', true);
if (hub) {
    // Проверить установлен ли
    const inst = await kg('/db/installed_apps.json?form[app_id]=' + hub.id + '&limit=1');
    const installed = (inst?.data || [])[0];
    if (installed) {
        App.navigate('/db/installed_apps/' + installed.alias + '?frame=main');
    } else {
        App.navigate('/db/marketplace/' + hub.alias);  // на карточку для установки
    }
}
```

**Permissions:**
```json
"catalogs": { "marketplace": ["read"], "installed_apps": ["read"] }
```

---

## Что читать дальше

- [styling.md](styling.md) — как игра должна выглядеть чтобы вписываться в платформу и гейм-настроение
- [checklist.md](checklist.md) — перед деплоем
- [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — полный эталон с применением всех рецептов
