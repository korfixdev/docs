# Модульная структура игры-миниапа

> **Навигация:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · **вы здесь** · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · [checklist](checklist.md).

С ростом игры плоская структура `index.html + js/{api,game,shop}.js` быстро превращается в простыню. Ниже — паттерн, по которому построены эталонные Games Hub и Coin Clicker.

---

## Паттерн каталогов

```
my-game/
├── config.json            — метаданные + секция korgames (обязательно)
├── icon.svg
├── README.md              — структура проекта, история изменений
├── frames/                — HTML-страницы (то, что в config.urls)
│   ├── main.html
│   └── settings.html      — опц., если есть "шестерёнка"
├── core/                  — переиспользуемые singletons
│   ├── api.js             — обёртка над App.fetch к /api/korgames/*
│   ├── i18n.js            — мини-i18n для EN/RU (см. ниже)
│   └── storage.js         — safe-wrappers для App.storage
├── modules/               — UI-логика, один файл на раздел/модалку
│   ├── game.js
│   ├── shop.js
│   ├── top-scores.js
│   └── profile.js
├── locales/
│   ├── en.json            — дефолтный язык
│   └── ru.json
└── styles/
    └── style.css
```

В `config.json`:

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

## Почему frames ≠ html в корне

1. **Читаемость** — видно что фреймов может быть несколько, они все вместе.
2. **Расширяемость** — легко добавить `frames/widget.html` и в config.urls `"widget": "frames/widget.html"`.
3. **Пути** — относительные `../core/api.js` работают без сюрпризов независимо от глубины.

---

## Правила modular-загрузки

Внутри каждого фрейма:

```html
<!-- 1. Regular scripts — объявляют globals -->
<script src="../core/api.js"></script>
<script src="../core/i18n.js"></script>
<script src="../core/storage.js"></script>
<script src="../modules/game.js"></script>
<script src="../modules/shop.js"></script>

<!-- 2. Module-script — bootstrap -->
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

Порядок важен:
- Regular scripts грузятся **до** парсинга module-script (последний дефферится).
- Объекты из regular-скриптов должны быть на `window` (или `const X = {}; window.X = X;`) чтобы module-bootstrap их видел.
- Async-инициализация (`i18n.init`, `CC.init`) — только в module-script, потому что regular-скрипты не могут использовать `await` на top-level.

---

## i18n — минимальная реализация

Используй тот же `core/i18n.js` что и в Games Hub / Coin Clicker. Он:

1. Хранит язык в `App.storage.get('lang')` (per-user, per-app).
2. Грузит `locales/{lang}.json` через `fetch(...)`.
3. Резолвит ключи по точке: `t('shop.price', {n: 300})` → `'300 Korn'`.
4. `applyToDom(root)` заменяет все элементы с `data-i18n="key"` и `data-i18n-attr="title:key"`.

### Ключи

Иерархия по разделам UI:

```json
{
    "app":     { "title": "...", "loading": "..." },
    "tabs":    { "balance": "...", "quests": "..." },
    "balance": { "title": "...", "corn": "...", "streak": "..." },
    "shop":    { "title": "...", "buy": "...", "price": "{n} Korn" }
}
```

Параметры — `{placeholder}`, передаются вторым аргументом: `i18n.t('shop.price', {n: 300})`.

### Разметка

```html
<h2 data-i18n="shop.title">Shop</h2>
<button data-i18n="shop.buy">Buy</button>
<input placeholder="" data-i18n-attr="placeholder:profile.name_hint">
```

После `i18n.applyToDom(document)` — `textContent` и атрибуты заменятся.

### Переключатель

```js
const langSwitch = document.getElementById('lang-switch');
for (const code of i18n.supported()) {
    const b = document.createElement('button');
    b.textContent = code.toUpperCase();
    if (code === i18n.current()) b.classList.add('active');
    b.addEventListener('click', async () => {
        await i18n.setLang(code);
        i18n.applyToDom(document);
        // повторно отрендерить свои кастомные куски (innerHTML после t()):
        await CC.refreshAll();
    });
    langSwitch.appendChild(b);
}
```

Важно: `applyToDom` работает только с `data-i18n` тегами. Если у тебя `root.innerHTML = \`${i18n.t('key')}\``, при смене языка нужно **перерендерить** этот блок самостоятельно.

### Добавить новый язык

1. Скопировать `locales/en.json` → `locales/de.json`, перевести.
2. В `core/i18n.js` добавить `'de'` в `const SUPPORTED = ['en', 'ru']`.
3. Ключ названия языка в EN/RU файлах: `settings.lang_de`.

---

## core/api.js — обёртка App.fetch

```js
const KgApi = {
    async _call(url, ...rest) {
        const r = await App.fetch(url, ...rest);
        return r?.data ?? r;           // распаковка postMessage-обёртки
    },
    // Релативный → абсолютный URL (iframe на store, ресурсы на CRM)
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

Важно:
- Rest-спред (`...rest`) чтобы не передавать `undefined` как второй аргумент (ломает хост, см. [gotchas в backend docs](https://docs.korfix.info/gamedev/client-api)).
- Body — **объект**, не `JSON.stringify`. VMCRMUserApp сам превратит в URLSearchParams.
- `encodeURIComponent` для значений в query string.
- **absUrl для любой платформенной ссылки в `<img src>`, `<a href>` (не через `App.navigate`) и прочем.** Iframe на store-домене, относительный `/reimg/...` идёт на store → 404. См. [client-api.md § Аватар](https://docs.korfix.info/gamedev/client-api) и skill `korfix-js-api` (пункт «Ресурсы платформы — абсолютные пути»).

---

## core/storage.js — safe-wrappers

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

Зачем: `App.storage.*` иногда кидает на миссинге — оборачиваем в try/catch, всегда возвращаем значение.

---

## modules/* — UI-логика

Каждый модуль = один объект, экспортируется на `window`. Пример:

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
            // ... обработка
        });
    }
};
window.KgProfile = KgProfile;
```

Разделение: один файл — одна UI-секция (таб, модалка, шаг installer'а). Если файл подбирается к 300 строкам — признак что пора разбить.

---

## Конвенции именования

- **Префиксы globals**: `Kg*` для Games Hub, `Cc*` для Coin Clicker, свой префикс для своей игры. Избегай конфликтов — может быть несколько игр открыты одновременно в разных iframes, но globals живут в рамках одного document.
- **CSS-классы**: `.kg-*` / `.cc-*` — избегаем коллизий со стилями платформы.
- **i18n-ключи**: `<section>.<key>`, snake_case внутри (`balance.daily_left`, `shop.not_enough`).
- **Event handlers**: при повторном рендере блока (`.innerHTML = ...`) handler'ы тоже удалятся — всегда перевешивать после рендера.

---

## ОБЯЗАТЕЛЬНО: body не трогаем

iframe миниапа вставляется в страницу платформы — фон и шрифт наследуются визуально. Если задать свой `background` на `body`, получается инородный блок.

**Правило:** `body` оставляем **прозрачным**:

```css
body {
    margin: 0; padding: 12px;
    font: 400 14px/1.5 "Open Sans", sans-serif;
    color: var(--dark);
    background: transparent;  /* ← обязательно */
}
```

Если игра требует собственной атмосферы (тёмный фон, золото, неон) — **только внутри контейнера**:

```html
<div class="cc-app">
    <!-- общие элементы вне тематической рамки — топбар, профиль -->
    <div class="cc-topbar">...</div>
    <div id="cc-profile-strip"></div>

    <!-- игровая область в gamified-frame -->
    <div class="cc-frame">
        <h1>Coin Clicker</h1>
        <!-- монетка, таймер, кнопки -->
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

Преимущества:
- Игра выглядит как самостоятельный элемент на странице платформы.
- Модалки/попапы вне `.cc-frame` рендерятся с нейтральным body — не «вытягивают» атмосферу на всё.
- Легко переключить стиль: поменял один `.cc-frame` — новая арена. Body не затронут.

Анти-паттерн:
```css
body {
    background: linear-gradient(160deg, #fff9e6 0%, #fff3cd 100%);  /* ✗ */
}
```
При открытии миниапа в iframe платформы край страницы будет резко жёлтым, остальная панель — серо-белая. Выглядит инородно.

Эталон — Coin Clicker `styles/style.css` (`.cc-frame` + прозрачный body).

---

## Что дальше

- [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — эталонная игра со всей описанной структурой.
- [checklist.md](checklist.md) — что проверить перед деплоем.
- Исходники для reference: `/tmp/games-hub-v2/` и `/tmp/coin-clicker-v2/` (в dev-окружении panel.korfix.ru).

---

## Cross-app discovery — поиск другого миниапа

Когда одной игре нужно знать «где Games Hub» или «где мой спутник-виджет» — **не хардкодь** `marketplace.id` или `marketplace.alias`. Они per-instance и меняются.

Используй поле `config.package` — это логический ID (напр. `games-hub`, `coin-clicker`). Сохраняется в `marketplace.package`, searchable. Системные миниапы дополнительно помечены `marketplace.system = 1` — фильтр по этому защищает от подделок с тем же package.

```js
// Найти Games Hub в маркетплейсе
const r = await App.fetch('/db/marketplace.json?form[package]=games-hub&form[system]=1&limit=1');
const hub = r?.data?.[0];

// Установлен ли у юзера?
if (hub) {
    const inst = await App.fetch('/db/installed_apps.json?form[app_id]=' + hub.id + '&limit=1');
    const installed = inst?.data?.[0];
    if (installed) {
        App.navigate('/db/installed_apps/' + installed.alias + '?frame=main');
    } else {
        App.navigate('/db/marketplace/' + hub.alias);  // карточка в маркетплейсе
    }
}
```

Требования к permissions в твоём config.json:
```json
"permissions": {
    "catalogs": {
        "marketplace":     ["read"],
        "installed_apps":  ["read"]
    },
    "navigate": true
}
```

См. боевой код в `coin-clicker/modules/profile-strip.js`.
