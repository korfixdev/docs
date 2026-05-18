# Концепции игровой экономики Korfix

> **Навигация:** [index](index.md) · **вы здесь** · [config-korgames](config-korgames.md) · [client-api](client-api.md) · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · [checklist](checklist.md).

Базовые понятия, которыми оперируют `/api/korgames/*`, Games Hub и игры-миниапы. Перед чтением [client-api.md](client-api.md) и кодированием — прочитать первым.

---

## Валюты

| Валюта | Эмитент | Назначение |
|--------|---------|------------|
| **Korn** | Только платформа (за whitelist-события) | Универсальная игровая валюта. Зарабатывается квестами и активностью, тратится на магазин игр и (планируется) pool-режим. |
| **Платина** | Real-money (покупка за рубли/usd) | Премиум. Покупка премиум-миниапов, подписок. В игры вне магазина не идёт. |
| **Внутренние валюты игр** | Сама игра (в своей storage) | Дело игры — платформа не знает и не участвует. Если хочешь монетку «кристаллы» — храни в `App.storage`. |

Твоя игра работает в основном с **Korn** (единственная межигровая валюта).

## Эмиссия Korn — только whitelist

Платформа начисляет Korn только за:

| Event | Кто начисляет | Источник |
|-------|---------------|----------|
| `login` | Ядро | Первый вход за день (onboarding-дейлик) |
| `streak` | Ядро | Milestone-бонусы 3/7/14/30 дней (+30/100/250/1000 Korn) |
| `create_record` | Ядро | Создание записи в whitelisted каталоге (accounts/projects/...) |
| `deploy_app` | Ядро | Деплой своего миниапа в маркетплейс |
| `install_app` | Ядро | Установка чужого миниапа |
| `referral` | Ядро | Приглашённый юзер зарегился |
| `quest` | Ядро | `claimQuest` после condition_value достигнут |
| `game_pool` | Ядро (планируется) | Приз за победу в pool-раунде |
| `admin` | Ядро | Ручные начисления разработчиком |

**Твоя игра не может вызвать `earnCorn`.** В исходниках Games.class.php есть жёсткий whitelist на source — попытка не-whitelisted source возвращает ошибку и ничего не пишет. Этот контракт защищает Korn от инфляции (реализация — на серверной стороне `\korgames\Games::earnCorn`, см. backend `Docs/korgames/security.md` если нужны детали).

**Почему:** анти-инфляция. Если каждая игра бы сама печатала Korn — валюта превратилась бы в мусор. Игры это рынки/каналы, не монетные дворы.

## Daily cap

Каждый юзер может заработать не более `daily_cap` Korn в сутки (default 500). Превышение режется: `earnCorn` возвращает `capped: true`, начисляет столько, сколько осталось до cap'а.

Читай в `GET /api/korgames/balance`:
```json
{"corn": 120, "today_earned": 480, "daily_remaining": 20, ...}
```

## Истечение Korn

Каждое начисление Korn сдвигает `corn_expires_at` на 90 дней вперёд. Если юзер не активничает — через 90 дней cron `expire_coins` обнуляет его баланс (с записью в sys_transactions как `expire`).

Для игры: не обещай юзеру «копи Korn всю жизнь» — копить дольше 90 дней без активности не получится.

---

## Стрик (streak)

`current_streak` в балансе — сколько дней подряд юзер логинится. Обновляется при первом входе за день.

- Пропустил день → стрик сбрасывается в 0 (через cron `daily_reset`).
- Milestone-бонусы за 3/7/14/30 дней → автоматический `earnCorn` (source='streak').
- `longest_streak` — рекорд (монотонно не убывает).

Используй `current_streak` в UI игры — это социальный сигнал («4-й день подряд заходишь — молодец!»).

---

## Квесты

**Квест** — декларация задачи в справочнике `sys_quests`. **Прогресс юзера** — в `sys_user_quests`.

### Типы (`sys_quests.type`)

| Тип | Периодичность | Пример |
|-----|----------------|--------|
| `onboarding` | `once` (раз на всю жизнь) | «Заполни профиль» |
| `daily` | Сброс ежесуточно (UTC) | «Создай 3 записи сегодня» |
| `weekly` | Сброс еженедельно | «Задеплой миниап за неделю» |
| `achievement` | `once`, но долгоиграющий | «100 часов в игре» |

### condition_type — какое событие считаем

Встроенные типы:
- `login` — каждый session.start (считает page-loads — условие обычно 1, "первый вход за день")
- `streak` — **абсолютное значение** стрика (не инкремент!)
- `create_record` — INSERT в whitelisted каталогах
- `deploy_app` — деплой миниапа
- `install_app` — установка чужого миниапа
- `referral` — успешная регистрация приглашённого
- `game_play` — раунд любой игры (+1 за сессию)
- `game_score` — `value = score` (абсолют, НЕ инкремент — хотя сейчас инкрементально, как и login)
- `profile_fill` — зарезервирован, пока без триггера (см. backend gotchas)

### Жизненный цикл user_quests

```
available → in_progress (при первом прогрессе)
         → completed (progress >= condition_value)
         → claimed (юзер нажал «Забрать» → +reward_corn)
```

`claimed` — финальный для того period_key. На новый период (следующий день для daily) создаётся новая запись.

### Как создать свой квест

Добавь строку в `sys_quests` через SQL (миграция в `public_html/log/`):

```sql
INSERT INTO crm__sys_quests (alias, name, description, type, reward_corn, condition_type, condition_value, period_type, is_active, sort_order, from_auth, from_group)
VALUES ('my_super_quest', 'Супер-квест', 'Сделай что-то крутое', 'daily', 50, 'create_record', 5, 'daily', 1, 100, 0, 0);
```

Если condition_type стандартный (`create_record` и т.п.) — квест заработает сразу. Если нужен кастомный condition_type — нужен серверный хук/вызов `Games::checkQuest('your_type', +value)`. Делается в модуле платформы, не в миниапе — правка бекенда описана в backend `Docs/korgames/hooks-and-cron.md`.

---

## Игры

**Игра** = миниап с секцией `korgames` в config.json. При установке хук `on_app_installed` читает секцию и UPSERT'ит запись в `sys_registered_games` (alias=game_id).

**Package convention:** игровые миниапы обязательно префиксятся `game-` (`game-coin-clicker`, `game-tetris`) — см. [config-korgames.md § package](config-korgames.md). Это упрощает фильтрацию и коэкзистенцию с бизнес-миниапами.

### Режимы игры (`reward_mode`)

- `score_only` (MVP) — только лидерборд, без начисления Korn. Продаёшь items за Korn, который юзер наработал вне игры.
- `pool` (планируется) — раунд с входным фи в Korn, победитель забирает пул минус комиссия платформы. Лимиты в `max_corn_per_session`/`max_corn_per_day`.

### Лидерборд

**Общий лидерборд Korfix** (`/api/korgames/leaderboard`) агрегирует `SUM(earn_korn)` по всем источникам — это не игровой топ, а рейтинг самых активных пользователей.

Для **игрового лидерборда** (топ по `sys_game_scores` для конкретной игры) — сейчас endpoint'а нет. Запланировано: `/api/korgames/game/scores?game_id=X&period=all_time`. Пока строй сам из `App.fetch('/db/sys_game_scores.json?form[game_id]=X')`.

### Магазин items

Товары декларируются в `config.korgames.items` → при установке создаются в `sys_game_items`. Покупка через `POST /api/korgames/game/buy`:

1. Юзер выбирает item.
2. Платформа атомарно списывает Korn, пишет транзакцию и `sys_game_purchases`.
3. Инвентарь юзера = SELECT из `sys_game_purchases WHERE author_id=me`. Игра читает через `GET /api/korgames/game/inventory?game_id=X`.

Эффект от item — твоя ответственность: считаешь у себя на клиенте (гeйм-логика).

---

## Роли пользователей

Фолкнамп: важно понимать что author_id != from_group.

- `author_id` — уникальный ID пользователя (персона).
- `from_group` — ID тенанта (организации, где пользователь работает).

Все записи в sys_* проставляют оба. В лидерборде по умолчанию ранк строится across всех tenants — если нужна изоляция, фильтруй `from_group` в запросе.

---

## Что читать дальше

- [config-korgames.md](config-korgames.md) — как объявить свою игру
- [client-api.md](client-api.md) — JS API с примерами
- [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — эталонная игра
- [checklist.md](checklist.md) — перед релизом
