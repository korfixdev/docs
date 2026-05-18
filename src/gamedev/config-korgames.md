# Секция `korgames` в config.json

> **Навигация:** [index](index.md) · [concepts](concepts.md) · **вы здесь** · [client-api](client-api.md) · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · [checklist](checklist.md).
> **Связанное:** [miniapps/config-json.md](../miniapps/config-json.md) — общая структура config.json для любого миниапа.

Чтобы миниап стал **игрой** в экосистеме Korfix, добавь секцию `korgames` в `config.json`. При установке хук платформы прочитает её и создаст записи в `sys_registered_games` и `sys_game_items` (подробнее про хук — в [concepts.md § Игры](concepts.md)).

---

## Минимальный пример

```json
{
    "name": "Coin Clicker",
    "alias": "coin-clicker",
    "package": "coin-clicker",
    "version": "1.0.0",
    "description": "Кликер за Korn",
    "about": "## Что делает\n30-секундный кликер...",
    "logo": "icon.svg",
    "urls": { "main": "index.html" },
    "urlsConf": { "main": { "method": "get" } },
    "permissions": {
        "storage": true,
        "navigate": false,
        "modal": true
    },
    "korgames": {
        "game_id": "coin-clicker",
        "reward_mode": "score_only",
        "items": []
    }
}
```

---

## Верхнеуровневые поля для gamedev

### `package` (обязательно для игр)

Глобальный логический ID миниапа в экосистеме (kebab-case, латиница).

**Конвенция префиксов** (чтобы по namespace сразу понимать что за миниап):

| Префикс | Что | Примеры |
|---|---|---|
| `game-` | Игровой миниап (с секцией `korgames`) | `game-coin-clicker`, `game-tetris`, `game-chess-puzzle` |
| `games-` | Системный сервис для экосистемы игр | `games-hub` (singular-system, не игра сама, а центр) |
| без префикса | Обычный бизнес-миниап | `bdr-report`, `quotes`, `customer-360` |

Твоя игра — **всегда с `game-` префиксом** в package. Это даёт:
- Фильтрацию в marketplace: `/db/marketplace.json?form[package]=game-%` (SQL LIKE-friendly)
- Очевидность для других разработчиков
- Меньше шансов коллизии с бизнес-миниапами

Зачем нужен package вообще: **другие миниапы находят твой по package**, а не по платформенному `marketplace.id` (разный на тесте/проде). Сохраняется в `marketplace.package` (searchable).

Пример поиска Games Hub из сторонней игры:
```js
const r = await App.fetch('/db/marketplace.json?form[package]=games-hub&form[system]=1&limit=1');
const hub = r.data[0];
App.navigate('/db/marketplace/' + hub.alias);
```

Поиск всех игр:
```js
// Через sys_registered_games (правильный путь — там только опубликованные игры)
const r = await App.fetch('/api/korgames/games');

// Либо через marketplace по префиксу (включает и не-установленные)
const r = await App.fetch('/db/marketplace.json?form[package]=game-%&limit=50');
```

### `game_id` vs `package` — различие

- **`package`** — marketplace-level ID с префиксом `game-` (напр. `game-coin-clicker`). Для marketplace-поиска и cross-app discovery.
- **`korgames.game_id`** (ниже) — короткий ID для API и `sys_registered_games.alias` (напр. `coin-clicker`, без префикса). Используется в `/api/korgames/game/score?game_id=X` и т.п.

Держи их согласованными: `package = 'game-' + game_id`.

### `system` (для платформенных/застолблённых приложений)

Нельзя установить через config.json — это платформенный флаг (`marketplace.system tinyint`). Назначается админом платформы вручную (`UPDATE marketplace SET system=1 WHERE package='games-hub'`).

Семантика:
- `system=0` (дефолт) — обычное пользовательское приложение.
- `system=1` — системный/застолблённый миниап. Через фильтр `form[package]=X&form[system]=1` сторонние игры находят именно «настоящий» Hub, а не подделку с тем же package от другого разработчика.

**Защита от подмены** (roadmap): бекенд должен отказываться принимать deploy если `config.package` совпадает с существующим `system=1` миниапом и deployer != автор. Сейчас проверка не реализована — полагаемся на фильтр `system=1` при поиске.

---

## Поля секции `korgames`

### `game_id` (обязательно)

Уникальный логический идентификатор игры. Используется как `alias` в `sys_registered_games`. Формат: `kebab-case`, латиница + `-`.

Если другой разработчик опубликует игру с тем же `game_id` — при установке UPSERT перезатрёт запись. В MVP нет проверки конфликтов на уровне marketplace. Выбирай уникальное имя.

### `reward_mode` (обязательно)

- `"score_only"` — **единственное допустимое значение в MVP.** Запись score + магазин items. Korn не начисляется игрой.
- `"pool"` — не реализовано. Оставлен enum в sys_registered_games для будущего расширения.

### `items` (опционально)

Массив товаров магазина игры. Каждый item:

```json
{
    "key": "gold_cursor",
    "name": "Золотой курсор",
    "description": "Клик даёт 2 очка вместо 1",
    "price_corn": 300,
    "max_per_user": 1
}
```

| Поле | Тип | Смысл |
|------|-----|-------|
| `key` | string | уникален в рамках игры; используется в API `item_key` |
| `name` | string | отображение в магазине |
| `description` | string | что делает item (подсказка юзеру) |
| `price_corn` | int | цена в Korn |
| `max_per_user` | int | `0` = безлимит, `1` = купить только раз, `N` = до N копий |

Будущие поля (в roadmap):
- `price_platinum` — продажа за Платину
- `duration_sec` — временный buff
- `stacking` — можно ли суммировать эффекты при повторной покупке

---

## Что происходит при установке

1. Юзер нажимает «Установить» в маркетплейсе → INSERT в `installed_apps`.
2. Ядро фирит `int_done_run('app.installed', [app_id, user_id, from_group])`.
3. Хук `korgames/hooks/on_app_installed.php`:
   - Читает `crm__marketplace.appconfig` (JSON-кеш, не FS).
   - Если секция `korgames` есть → `Games::registerGameFromConfig`.
4. `registerGameFromConfig`:
   - UPSERT `sys_registered_games` по `alias = game_id`.
   - UPSERT `sys_game_items` для каждого item (по `game_id × item_key`).
5. Доп. квесты: `deploy_app` +1 (автору) и `install_app` +1 (установившему, если отличается).

> Важно: **appconfig в marketplace должен быть наполнен до установки.** Если деплоил через `POST /api/db/marketplace/{id}` — запиши ещё и `POST /api/marketplace/refresh/{id}`, иначе хук логирует warning «appconfig empty» и игру не регистрирует. Правильный быстрый путь — сразу `POST /api/marketplace/deploy/{id}` (= update + refresh).

---

## Обновление игры

При каждом deploy (`POST /api/marketplace/deploy/{id}`):
- `appconfig` перезаливается из нового config.json.
- Новые items добавятся в sys_game_items.
- Изменённые items (name/description/price) обновятся.
- **Удалённые из config items НЕ удаляются** из sys_game_items — остаются is_active=1 (без миграции).

Если хочешь жёстко «срезать» старые items — удали вручную через SQL (и не забудь `sys_game_purchases` там может быть FK через item_id — но это append-only, не затронет).

---

## Ограничения и типичные ошибки

1. **Секция `korgames` обязательна `game_id`.** Если пропустил — хук тихо вернёт без регистрации.
2. **`reward_mode` строго из enum.** `"free"`, `"unlimited"` и т.п. не поддерживаются — хук выдаст SQL-ошибку на INSERT (enum-валидация MySQL).
3. **`price_corn` должен быть ≥ 0.** Отрицательная цена = нежданная эмиссия — проверка в Games::registerGameFromConfig отсечёт.
4. **`max_per_user` > 0 рекомендуется для уникальных improvements.** Иначе юзер может купить 100 «Золотых курсоров» — но эффекта всё равно будет один (гeйм-логика твоя).
5. **Не полагайся на игру как на источник Korn.** Юзер должен где-то наработать Korn (квесты, платформенные действия) и только потом тратить в магазине игры.

---

## Следующий шаг

- [client-api.md](client-api.md) — как вызывать `/api/korgames/game/*` из JS миниапа.
- [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — пройтись по эталонной игре от config.json до score + магазина.
- [checklist.md](checklist.md) — что проверить перед деплоем в маркетплейс.
