# API Reference — `/api/korgames/*`

> **Навигация:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · **вы здесь** · [recipes](recipes.md) · [styling](styling.md) · [checklist](checklist.md).

Полный справочник endpoints модуля korgames. Все примеры — реальные ответы тест-инстанса.

---

## Общее

**Авторизация:** cookie-сессия (миниап получает из iframe-парента) ИЛИ Bearer-токен с apiclass `korgames_*`. Миниапы обычно используют cookie.

**Формат ответа:**
```json
{"status": "success", "data": { ... }}
{"status": "error", "message": "human msg", "error": "machine_code"}
```

**HTTP коды:** 200 для успех/бизнес-ошибок, 401 если нет сессии/токена, 403 если токен без apiclass, 500 uncaught.

**Клиентская обёртка:** `App.fetch` возвращает `{data: serverResponse, requestId}` — распаковывай `r?.data ?? r` (см. [client-api.md](client-api.md)).

---

## Balance & Install

### `POST /api/korgames/install`

Идемпотентно создаёт запись в `sys_user_balances` для текущего юзера. Вызывай при первом открытии миниапа (или реже — см. паттерн в games-hub).

**Request:** body пустой.

**Response (success):**
```json
{
    "status": "success",
    "data": {
        "ok": true,
        "created": false,
        "balance": {
            "corn": 130,
            "platinum": 0,
            "today_earned": 130,
            "daily_remaining": 370,
            "current_streak": 1,
            "longest_streak": 1,
            "total_earned": 130,
            "total_spent": 0,
            "expires_at": "2026-07-20 23:53:01",
            "last_activity_at": "2026-04-22 23:53:01",
            "last_login_date": "2026-04-22"
        }
    }
}
```
- `created: false` — запись уже была. `true` — создана сейчас.

**Errors:** только system errors (500).

### `GET /api/korgames/balance`

Текущий баланс + стрик + дневной лимит.

**Response:**
```json
{
    "status": "success",
    "data": {
        "corn": 130,
        "platinum": 0,
        "today_earned": 130,
        "daily_remaining": 370,
        "current_streak": 1,
        "longest_streak": 1,
        "total_earned": 130,
        "total_spent": 0,
        "expires_at": "2026-07-20 23:53:01",
        "last_activity_at": "2026-04-22 23:53:01",
        "last_login_date": "2026-04-22"
    }
}
```

Если юзер не install'ен — `data: null` или ошибка. Проверяй `r.data` перед доступом.

---

## Quests

### `GET /api/korgames/quests?type=all|daily|weekly|onboarding|achievement`

Список активных квестов с прогрессом текущего юзера. `type=all` (default).

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 30,
            "alias": "daily_login",
            "name": "Ежедневный вход",
            "description": "Заходи в панель каждый день",
            "type": "daily",
            "reward_corn": 10,
            "condition_type": "login",
            "condition_value": 1,
            "period_type": "daily",
            "status": "claimed",
            "progress": 1,
            "completed_at": "2026-04-22 01:55:28",
            "claimed_at": "2026-04-22 01:29:16"
        },
        {
            "id": 31,
            "alias": "daily_create_3",
            "name": "Создай 3 записи",
            "description": "Три новые записи за сегодня",
            "type": "daily",
            "reward_corn": 30,
            "condition_type": "create_record",
            "condition_value": 3,
            "period_type": "daily",
            "status": "available",
            "progress": 0,
            "completed_at": null,
            "claimed_at": null
        }
    ]
}
```

**Статусы:** `available` → `in_progress` → `completed` → `claimed`.

### `POST /api/korgames/quest/claim`

Забрать награду у completed-квеста. Можно только один раз за period_key.

**Request body:**
```json
{"quest_id": 30}
```

**Response (success):**
```json
{
    "status": "success",
    "ok": true,
    "quest_id": 30,
    "quest_alias": "daily_login",
    "reward_corn": 10,
    "earn_result": {
        "ok": true,
        "earned": 10,
        "capped": false,
        "balance": 10
    }
}
```

**Errors:** `not_completed`, `already_claimed`, `not_found`, `daily_cap_exceeded`.

---

## Leaderboard & History

### `GET /api/korgames/leaderboard?period=week|month|all_time`

Топ по Korn earned за период. Обновляется cron'ом `leaderboard_refresh` каждые 5 минут.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "rank": 1,
            "author_id": 3,
            "display_name": "DemoPlayer",
            "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc123.png",
            "user_name": "Финансы",
            "corn_earned": 130,
            "apps_deployed": 0,
            "quests_completed": 3,
            "games_played": 8,
            "prize_corn": 0
        }
    ],
    "period": "week",
    "period_key": "2026-W17"
}
```

- `display_name` — из sys_game_profiles (приоритет). `user_name` — из auth_pers.author_comment (fallback). `null`/пусто если не заполнено.
- `avatar_url` — относительный путь. Абсолютизируй через `absUrl()` (см. [client-api.md](client-api.md)).

**Пустой массив** — cron ещё не отработал или транзакций за период нет.

### `GET /api/korgames/transactions?limit=100&offset=0`

История операций текущего юзера.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 102,
            "author_id": 3,
            "amount": 100,
            "currency_type": "corn",
            "transaction_type": "earn",
            "source": "quest",
            "source_id": "weekly_streak_7",
            "balance_after": 130,
            "description": "Earn via quest",
            "ts": "2026-04-22 14:34:08"
        }
    ]
}
```

**`transaction_type`:** `earn`, `spend`, `purchase`, `expire`, `admin`, `game_buy`.

**`source`** (whitelist для earn): `login`, `streak`, `referral`, `create_record`, `deploy_app`, `install_app`, `quest`, `game_pool`, `admin`.

---

## Games — registration & list

### `GET /api/korgames/games`

Список активных зарегистрированных игр.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 8,
            "alias": "coin-clicker",
            "name": "coin-clicker",
            "miniapp_id": 109,
            "reward_mode": "score_only",
            "is_active": 1
        }
    ]
}
```

- `alias` == `korgames.game_id` из config.json миниапа
- `name` может быть = alias (если хук сохранил) или человеческое имя. Для отображения берёшь `marketplace.name` через доп. запрос.
- `miniapp_id` — FK на `marketplace.id`

### `GET /api/korgames/game/items?game_id=coin-clicker`

Товары магазина игры.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 20,
            "item_key": "gold_cursor",
            "name": "Золотой курсор",
            "description": "Клик даёт 2 очка вместо 1",
            "price_corn": 300,
            "max_per_user": 1,
            "is_active": 1
        }
    ]
}
```

### `POST /api/korgames/game/score`

Записать результат раунда. Триггерит квесты `game_play` +1 и `game_score` value=score.

**Request body:**
```json
{
    "game_id": "coin-clicker",
    "score": 182,
    "duration": 30
}
```

**Response:**
```json
{
    "status": "success",
    "ok": true,
    "session_id": 14,
    "corn_earned": 0
}
```

- `corn_earned: 0` в `reward_mode=score_only` (MVP) — Korn за score не начисляется, только через квесты.

**Errors:** `game_not_found` (game_id нет в sys_registered_games), `invalid_score` (< 0).

### `POST /api/korgames/game/buy`

Покупка item атомарно. Списывает Korn, пишет `sys_transactions` и `sys_game_purchases`.

**Request body:**
```json
{
    "game_id": "coin-clicker",
    "item_key": "gold_cursor"
}
```

**Response (success):**
```json
{
    "status": "success",
    "ok": true,
    "purchase_id": 5,
    "price_corn": 300,
    "balance_after": 830,
    "transaction_id": 103
}
```

**Errors:**
- `insufficient_balance` — не хватает Korn
- `limit_exceeded` — уже куплено `max_per_user` раз
- `item_not_found` — нет такого item_key в `sys_game_items`
- `game_not_active` — игра `is_active=0`

### `GET /api/korgames/game/inventory?game_id=X` (game_id опциональный)

Инвентарь юзера.

**Response:**
```json
{
    "status": "success",
    "data": [
        {
            "id": 5,
            "item_key": "gold_cursor",
            "price_corn": 300,
            "purchased_at": "2026-04-22 14:12:33"
        }
    ]
}
```

Без `game_id` — инвентарь всех игр. Используй `new Set(r.data.map(p => p.item_key))` для быстрого проверки "куплено ли X".

---

## Profile (cross-game)

### `GET /api/korgames/profile`

Игровой профиль текущего юзера — кросс-игровой, живёт в `sys_game_profiles`.

**Response:**
```json
{
    "status": "success",
    "data": {
        "display_name": "DemoPlayer",
        "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc123.png",
        "bio": "Люблю кликеры",
        "exists": true
    }
}
```

`exists: false` если запись не создана (первое открытие) — остальные поля пустые. **Не ошибка**, просто начни с `exists=false` и предложи заполнить.

### `POST /api/korgames/profile`

UPSERT игрового профиля.

**Request body:**
```json
{
    "display_name": "DemoPlayer",
    "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc123.png",
    "bio": "Люблю кликеры"
}
```

**Валидация:**
- `display_name` ≤ 100 chars
- `avatar_url` ≤ 500 chars, должен начинаться с `http(s)://`, `/reimg/`, или `/data/` если не пустой
- `bio` ≤ 200 chars

**Response:**
```json
{
    "status": "success",
    "data": { "display_name": "...", "avatar_url": "...", "bio": "...", "exists": true },
    "created": true
}
```

**Errors:** `display_name_too_long`, `avatar_url_too_long`, `avatar_url_invalid`, `bio_too_long`, `invalid_author`.

### `POST /api/korgames/profile/avatar`

Загрузка аватара через base64 data-URL. Рекомендуется клиентский ресайз до 256×256 через canvas перед отправкой.

**Request body (JSON):**
```json
{"avatar_base64": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUg..."}
```

Поддерживаемые форматы: `image/png`, `image/jpeg`, `image/webp`, `image/gif`. Валидация по magic-bytes (нельзя обмануть mime).

**Лимит:** 512 KB после декодирования.

**Response:**
```json
{
    "status": "success",
    "data": { "display_name": "...", "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc.png", "bio": "...", "exists": true },
    "avatar_url": "/reimg/data/db/f_sys_game_profiles/avatar_3_abc.png",
    "size": 45678,
    "mime": "image/png"
}
```

`avatar_url` — reimg-путь, поддерживает query `?WxH` для ресайза на лету.

**Errors:**
- `no_file` — пустой avatar_base64
- `bad_format` — не data-URL или mime не в allowlist
- `decode_failed` — base64 не декодируется
- `too_large` — > 512 KB
- `not_image` — magic-bytes не совпадают с mime
- `fs_error` — нет прав записи в `/data/db/f_sys_game_profiles/`
- `write_failed` — file_put_contents вернул false

**Почему не multipart:** `App.fetch` через postMessage делает JSON.stringify, File/FormData теряются. Data-URL = транспорт через string.

---

## Swagger

### `GET /api/korgames/swagger`

OpenAPI 3.0 спека всех endpoints. **Публичный** (без auth) — удобно для генерации типов.

---

## Что читать дальше

- [recipes.md](recipes.md) — готовые рецепты на все случаи (начислить, списать, магазин, лидерборд, профиль)
- [styling.md](styling.md) — стилистика игровых миниапов
- [client-api.md](client-api.md) — клиентская обёртка и паттерны вызова
