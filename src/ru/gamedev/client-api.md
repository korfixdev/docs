# Client API — вызовы из миниапа

> **Навигация:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · **вы здесь** · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · [checklist](checklist.md).
> **Связанное:** [miniapps/js-api.md](../miniapps/js-api.md) — общий `VMCRMUserApp` API (методы `App.storage`, `App.modal`, события). Здесь только игровые endpoints.

Все запросы — через `App.fetch` (не `window.fetch` — CORS). Авторизация через cookie (сессия родительского окна). Body в POST передаётся **объектом**, не JSON-строкой — VMCRMUserApp сам сериализует. Концепции Korn/квестов/score — в [concepts.md](concepts.md).

---

## Обязательная обёртка

`App.fetch` возвращает `{data: serverResponse, requestId}` — нужно распаковывать. Это особенность postMessage-транспорта (см. также [miniapps/data-api.md](../miniapps/data-api.md) § ответы API):

```js
async function kg(url, ...rest) {
    const r = await App.fetch(url, ...rest);
    return r?.data ?? r;   // в r.data лежит то, что вернул сервер
}
```

Далее примеры используют `kg(...)`.

---

## Баланс и стрик

```js
const r = await kg('/api/korgames/balance');
// r = { status: 'success', data: { corn, platinum, today_earned, daily_remaining,
//       current_streak, longest_streak, total_earned, total_spent,
//       expires_at, last_activity_at, last_login_date } }

const b = r.data;
console.log(`У тебя ${b.corn} Korn, стрик ${b.current_streak} дней`);
```

`POST /api/korgames/install` — идемпотентный, полезно дёрнуть на старте миниапа:

```js
await kg('/api/korgames/install', { method: 'POST' });
// data: {ok: true, created: true|false, balance: {...}}
```

> Лучше кешировать признак `installed` в `App.storage` — см. раздел `Идемпотентность через storage` ниже.

---

## Квесты

### Список

```js
const r = await kg('/api/korgames/quests?type=all');
// r.data = [{id, alias, name, description, type, reward_corn,
//           condition_type, condition_value, period_type,
//           status, progress, completed_at, claimed_at}, ...]

for (const q of r.data) {
    const pct = Math.min(100, Math.round(q.progress / q.condition_value * 100));
    console.log(`[${q.status}] ${q.name} — ${pct}%, награда ${q.reward_corn} Korn`);
}
```

Фильтр `type`: `all` | `daily` | `weekly` | `onboarding` | `achievement`.

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

Возможные ошибки: `not_completed`, `already_claimed`, `not_found`.

---

## Лидерборд

Общий — агрегирует **earn-транзакции** (не scoreы игр!).

```js
const r = await kg('/api/korgames/leaderboard?period=week');
// r.data = [{author_id, rank, corn_earned, games_played, quests_completed, user_name}, ...]
// period: week | month | all_time
```

Данные материализованы в `sys_leaderboard`, обновляются каждые 5 мин cron'ом.

Для **лидерборда конкретной игры** (топ по score) в MVP пока endpoint'а нет — читай напрямую:

```js
const r = await kg('/db/sys_game_scores.json?form[game_id]=coin-clicker&orderby=score+DESC&limit=10');
// r.data.data = [{author_id, score, duration_sec, played_at, ...}, ...]
```

---

## История транзакций

```js
const r = await kg('/api/korgames/transactions?limit=50&offset=0');
// r.data = [{id, amount, currency_type, transaction_type, source, source_id,
//           balance_after, description, ts}, ...]
```

---

## Игры — regexp / список

```js
const r = await kg('/api/korgames/games');
// r.data = [{alias, name, reward_mode, miniapp_id}, ...] — активные игры
```

---

## Магазин — items

```js
const r = await kg('/api/korgames/game/items?game_id=coin-clicker');
// r.data = [{id, item_key, name, description, price_corn, max_per_user}, ...]
```

---

## Запись score

```js
const r = await kg('/api/korgames/game/score', {
    method: 'POST',
    body: {
        game_id: 'coin-clicker',
        score: 142,
        duration: 30     // секунд
    }
});
// success: { status:'success', ok:true, session_id, corn_earned (0 в score_only mode) }
```

В `score_only`-режиме `corn_earned=0` всегда — Korn начисляется только за квесты типа `game_play` / `game_score`, которые триггерит сервер автоматически при записи score (если такие квесты определены и прогресс до `condition_value` дошёл — юзер потом claimQuest'ит).

---

## Покупка item

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

Атомарно: платформа проверит баланс, `max_per_user`, спишет Korn, запишет транзакцию и `sys_game_purchases`.

---

## Инвентарь

```js
const r = await kg('/api/korgames/game/inventory?game_id=coin-clicker');
// r.data = [{item_key, price_corn, purchased_at}, ...]

// Какие улучшения куплены
const owned = new Set(r.data.map(p => p.item_key));
if (owned.has('gold_cursor')) multiplier = 2;
```

Без параметра `game_id` — инвентарь всех игр разом.

---

## Игровой профиль

Отдельно от бизнес-профиля (`auth_pers`). Показывается в лидерборде вместо системного имени.

```js
// Получить
const r = await kg('/api/korgames/profile');
// r.data = {display_name, avatar_url, bio, exists}
// exists: false если записи ещё нет (первое открытие)

// Обновить (UPSERT)
const r = await kg('/api/korgames/profile', {
    method: 'POST',
    body: {
        display_name: 'ProGamer42',
        avatar_url:   'https://example.com/me.png',
        bio:          'Top clicker since 2025'
    }
});
// r.data = профиль после сохранения, r.created = true/false
```

### Аватар — загрузка файла

`App.fetch` через postMessage не умеет `FormData`/`File` (JSON.stringify уничтожает). Решение — **base64 data-URL**: клиент читает файл, ресайзит через canvas, шлёт строкой.

```js
async function uploadAvatar(file) {
    // 1. Читаем файл
    const reader = new FileReader();
    const dataUrl = await new Promise((resolve, reject) => {
        reader.onload = () => resolve(reader.result);
        reader.onerror = () => reject(new Error('read failed'));
        reader.readAsDataURL(file);
    });

    // 2. Ресайзим до 256×256 квадрата (center-crop) для экономии
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

Поддерживаемые форматы: **png, jpeg, webp, gif**. Лимит — **512 KB после ресайза**.

Отображение с auto-resize через reimg:
```html
<img src="https://{CRM_HOST}/reimg/data/db/f_sys_game_profiles/avatar_3_xxx.png?80x80">
```

### Absolute URLs (аватары, ресурсы платформы) {#absolute-urls}

**ВАЖНО — путь должен быть АБСОЛЮТНЫМ.** Iframe миниапа живёт на store-домене (`vmcrm.vnn.ru`), а файлы в `/reimg/` и `/data/` — на CRM-домене (`vibe.korfix.app`). Относительный `src="/reimg/..."` резолвится к store-домену → 404.

CRM-хост лежит в `App.requestParams.domain` после `getRequestParams()`. Хелпер:

```js
function absUrl(path) {
    if (!path || /^https?:\/\//i.test(path)) return path;
    const domain = window.App?.requestParams?.domain || '';
    if (!domain) return path;
    return 'https://' + domain.replace(/\/$/, '') + (path.startsWith('/') ? path : '/' + path);
}

// Использование
img.src = absUrl(profile.avatar_url);  // '/reimg/...' → 'https://vibe.korfix.app/reimg/...'
```

Применять к **любым** ссылкам на платформенные ресурсы:
- Аватары игрового профиля (`/reimg/data/db/f_sys_game_profiles/...`)
- Платформенные аватары (`/reimg/data/auth/...`)
- Иконки приложений (`/data/db/f_marketplace/...`)
- Файлы каталогов (`/data/db/f_{catalog}/...`)

Эталон: `games-hub/core/api.js` → `KgApi.absUrl()`.

Валидация на сервере:
- `display_name` — до 100 chars
- `avatar_url` — до 500 chars, должен начинаться с `http(s)://` если непустой
- `bio` — до 200 chars

Ошибки: `display_name_too_long`, `avatar_url_invalid`, `bio_too_long`, `invalid_author`.

В лидерборде (`/api/korgames/leaderboard`) поле `display_name` подтягивается автоматически через LEFT JOIN — приоритет над системным именем из `auth_pers`.

---

## Лидерборд конкретной игры (по score)

Нет специального endpoint'а в MVP. Читай напрямую:

```js
const r = await kg('/db/sys_game_scores.json?form[game_id]=coin-clicker&orderby=score+DESC&limit=10');
const rows = r?.data?.data || r?.data || [];
// Свернуть до 1 записи на автора (max score per user):
const byAuthor = new Map();
for (const row of rows) {
    const aid = +row.author_id, sc = +row.score;
    if (!byAuthor.has(aid) || byAuthor.get(aid).score < sc) {
        byAuthor.set(aid, { author_id: aid, score: sc, played_at: row.played_at });
    }
}
const top = Array.from(byAuthor.values()).sort((a,b) => b.score - a.score).slice(0, 10);
```

Пример в продакшене — `coin-clicker/modules/top-scores.js`. См. [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md).

---

## Идемпотентность через storage

Install лучше не дёргать на каждом открытии — юзер уже давно провисан в `sys_user_balances`:

```js
async function ensureInstalled() {
    const s = await App.storage.get('install_status');
    if (s && s.at) return;  // уже делали
    const r = await kg('/api/korgames/install', { method: 'POST' });
    if (r.status === 'success') {
        await App.storage.set('install_status', {
            at: new Date().toISOString(),
            corn_init: r.data?.balance?.corn ?? 0
        });
    }
}
```

В разделе настроек (gear в заголовке) добавь «Переустановить» — снова дёрнет install, обновит storage.

---

## Обработка ошибок

```js
try {
    const r = await kg('/api/korgames/quest/claim', { method: 'POST', body: { quest_id: 999 } });
    if (r.status !== 'success') {
        // Бизнес-ошибка — не исключение, а валидный ответ
        App.alert('Не удалось забрать: ' + (r.error || r.message));
        return;
    }
    // r.data — результат
} catch (e) {
    // HTTP-сбой / timeout / permission_denied
    console.error('API call failed:', e);
}
```

HTTP-ошибки (401/403/500) у `App.fetch` обычно приходят как rejected promise или не-success в теле — лучше оба пути обработать.

---

## Таймауты

`App.fetch` внутри использует postMessage с 60-секундным таймаутом. Если парент не ответил (например, миниап открыт в отстранённом iframe или host не настроен) — получишь reject через минуту.

Для UX добавь собственный короткий timeout:

```js
const withTimeout = (p, ms, label) => Promise.race([
    p,
    new Promise((_, rej) => setTimeout(() => rej(new Error(label + ' timeout')), ms))
]);

await withTimeout(kg('/api/korgames/balance'), 5000, 'balance');
```
