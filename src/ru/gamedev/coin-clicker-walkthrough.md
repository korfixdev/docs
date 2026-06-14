# Coin Clicker — эталонная игра-пример

> **Навигация:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · **вы здесь** · [checklist](checklist.md).

Минимальная завершённая игра, демонстрирующая все MVP-механики: регистрация игры, магазин, запись score, инвентарь, покупка. По мере чтения — ссылки на соответствующие разделы [config-korgames.md](config-korgames.md) и [client-api.md](client-api.md).

**Marketplace id = 109, alias = `coin-clicker`**.

Исходники: `/tmp/coin-clicker/` (в dev-окружении panel.korfix.ru) — можно скопировать как шаблон для новой игры.

---

## Файлы

```
coin-clicker/
├── config.json          — метаданные + секция korgames
├── index.html           — UI + module-script с bootstrap VMCRMUserApp
├── icon.svg             — иконка для маркетплейса
├── css/
│   └── style.css        — монетка + модалка магазина
└── js/
    ├── game.js          — объект CC (геймплей)
    └── shop.js          — объект Shop (магазин)
```

---

## config.json

```json
{
    "name": "Coin Clicker",
    "alias": "coin-clicker",
    "version": "1.0.1",
    "description": "Кликер за Korn — 30 секунд, рекорды и магазин улучшений",
    "about": "## Что делает\n30-секундный кликер...",
    "tags": "игра, кликер, демо, korn",
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
                "name": "Золотой курсор",
                "description": "Клик даёт 2 очка вместо 1",
                "price_corn": 300,
                "max_per_user": 1
            },
            {
                "key": "lucky_shine",
                "name": "Счастливый блеск",
                "description": "Каждый 10й клик = +5 очков",
                "price_corn": 150,
                "max_per_user": 1
            }
        ]
    }
}
```

Что платформа сделает при установке:
- Создаст `sys_registered_games.alias='coin-clicker'`, reward_mode='score_only'.
- Создаст 2 записи в `sys_game_items` — gold_cursor (300), lucky_shine (150).

Полный список полей секции — [config-korgames.md](config-korgames.md).

---

## index.html — bootstrap

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Coin Clicker</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
<div class="cc-app">
    <h1>Coin Clicker</h1>
    <div class="cc-coin-wrap"><div class="cc-coin" id="cc-coin">K</div></div>
    <p class="cc-score">Очки: <span id="cc-score">0</span></p>
    <p class="cc-timer">Время: <span id="cc-timer">30</span>с</p>
    <button id="cc-start" class="cc-btn">Старт</button>
    <button id="cc-shop" class="cc-btn">Магазин</button>
    <div id="cc-shop-modal" class="cc-modal hidden">
        <div class="cc-modal-content">
            <h2>Магазин улучшений</h2>
            <div id="cc-items"></div>
            <button id="cc-close-shop" class="cc-btn">Закрыть</button>
        </div>
    </div>
</div>

<!-- Регулярные скрипты — просто объявляют объекты CC и Shop -->
<script src="js/game.js"></script>
<script src="js/shop.js"></script>

<!-- Module-скрипт — bootstrap VMCRMUserApp и навешивает хендлеры -->
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

**Ключевые моменты:**
- Regular `<script>` должны быть ДО module-скрипта: они блокируют парс и выполняются в порядке загрузки, объявляя `CC` и `Shop` как globals.
- Module-скрипт type="module" дефферится до конца парса, потом импортирует VMCRMUserApp и вешает обработчики.
- `window.App = new VMCRMUserApp()` — глобальный App, используется CC/Shop. См. [miniapps/js-api.md](../miniapps/js-api.md).
- **Путь `/templates/def/db/marketplace/vmcrm-user-app.js` всегда абсолютный.** Не менять.

---

## js/game.js — геймплей

```js
const CC = {
    score: 0,
    running: false,
    timeLeft: 30,
    startTime: 0,
    multiplier: 1,      // эффект gold_cursor
    luckyShine: false,  // эффект lucky_shine
    clicks: 0,

    // Хелпер: распаковка postMessage-обёртки App.fetch (см. client-api.md § обёртка)
    async _api(url, ...rest) {
        const r = await App.fetch(url, ...rest);
        return r?.data ?? r;
    },

    // Читаем инвентарь при старте — узнаём какие улучшения куплены
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
        // Анимация
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
        alert(`Игра окончена! Очки: ${this.score}. Записано в рейтинг.`);
    }
};
```

**Что здесь происходит:**
1. `init()` — на старте миниапа читаем инвентарь. Если куплен `gold_cursor` — множитель x2. Если `lucky_shine` — каждый 10-й клик +5 бонусом.
2. `start()` — 30-секундный раунд с requestAnimationFrame-таймером.
3. `click()` — добавляем очки + анимация.
4. `finish()` — отправляем score на сервер. Результат записывается в `sys_game_scores`.

---

## js/shop.js — магазин

```js
const Shop = {
    async _api(url, ...rest) {
        const r = await App.fetch(url, ...rest);
        return r?.data ?? r;
    },

    async show() {
        const modal = document.getElementById('cc-shop-modal');
        const box = document.getElementById('cc-items');

        // Параллельно грузим товары, баланс, инвентарь
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
                    ${has ? 'Куплено' : 'Купить'}
                </button>
            </div>`;
        }).join('');

        box.querySelectorAll('.cc-buy').forEach(b => b.addEventListener('click', async () => {
            const r = await Shop._api('/api/korgames/game/buy', {
                method: 'POST',
                body: { game_id: 'coin-clicker', item_key: b.dataset.k }
            });
            if (r.status === 'success') {
                await CC.init();  // пересчитаем эффекты
                Shop.show();      // перерисуем магазин (покажет "Куплено")
            } else {
                alert('Ошибка: ' + (r.error || 'unknown'));
            }
        }));
        modal.classList.remove('hidden');
    },

    hide() { document.getElementById('cc-shop-modal').classList.add('hidden'); }
};
```

**Логика:**
1. При открытии магазина параллельно запрашиваем три endpoint'а.
2. Для каждого item строим кнопку: «Купить» если хватает Korn и не куплен, «Куплено» если уже есть, disabled если не хватает Korn.
3. После покупки — пересчитываем эффекты (`CC.init()`) и перерисовываем магазин.

---

## Цикл жизни игрока

1. Юзер открывает Games Hub → зарабатывает Korn за квесты (daily_login +10, daily_play_5 +20, etc).
2. Устанавливает Coin Clicker из маркетплейса → хук `on_app_installed` регистрирует игру.
3. Открывает Coin Clicker → CC.init() читает инвентарь (пока пустой) — multiplier=1.
4. Кликает 30 сек → CC.finish() → score запишется в sys_game_scores, квест `daily_play_5` прогрессит.
5. Открывает магазин → видит gold_cursor (300 Korn) и lucky_shine (150 Korn).
6. Если баланс ≥ 300 — покупает gold_cursor → платформа атомарно списывает Korn, пишет транзакцию и покупку.
7. CC.init() после покупки видит gold_cursor в инвентаре → multiplier=2.
8. Следующая игра — клики дают 2 очка.

---

## Деплой

```bash
# запаковать
cd /tmp/coin-clicker
zip -rq /tmp/coin-clicker.zip config.json index.html icon.svg css/ js/

# задеплоить (update + refresh в одном вызове)
curl -X POST "https://vibe.korfix.app/api/marketplace/deploy/109" \
  -H "Authorization: Bearer $KORFIX_TOKEN" \
  -F "doc1=@/tmp/coin-clicker.zip;type=application/zip"
```

Деплой нового приложения (когда id ещё нет):

```bash
curl -X POST "https://vibe.korfix.app/api/db/marketplace" \
  -H "Authorization: Bearer $KORFIX_TOKEN" \
  -F 'name=Coin Clicker' \
  -F 'category=3' \
  -F "doc1=@/tmp/coin-clicker.zip;type=application/zip"
# → {"status":"success","id":"N","alias":"..."}
# Сохранить N, дальше использовать /api/marketplace/deploy/N
```

---

## Что можно добавить (этап 2)

- Топ-10 игроков по score после раунда (сейчас только alert). Endpoint `/db/sys_game_scores.json?form[game_id]=coin-clicker&orderby=score+DESC&limit=10` — см. [client-api.md § лидерборд игры](client-api.md).
- Сохранять свой рекорд в `App.storage.set('best_score', ...)` и показывать на старте — см. [miniapps/storage-and-hooks.md](../miniapps/storage-and-hooks.md).
- Больше items: ×3 множитель, автоклик каждые 5 секунд, время раунда +15 сек. Добавлять в `config.korgames.items` — [config-korgames.md](config-korgames.md).
- Звуки (надо аккуратно — в iframe autoplay политика).
- «Призыв друга» — собственный квест на condition_type=referral (см. [concepts.md § Как создать свой квест](concepts.md)).

Шаблон готов — бери и делай свою игру. Перед деплоем — прогоняй [checklist.md](checklist.md).
