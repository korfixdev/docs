# Gamedev — разработка игр и игровых механик для Korfix

Раздел для разработчиков, которые хотят:
- **Сделать свою игру** для маркетплейса Korfix с лидербордом и магазином.
- **Встроить элементы геймификации** в обычный миниап (показать баланс, квесты, стрик).
- **Расширить существующий Games Hub** своим виджетом.

> Разработка обычных миниапов (без gamedev-специфики) — [miniapps/index.md](../miniapps/index.md). Игровой миниап — это обычный миниап с доп. секцией `korgames` в config.json и использованием `/api/korgames/*` endpoints.
>
> Релевантные общие разделы: [miniapps/config-json.md](../miniapps/config-json.md) (общий формат), [miniapps/js-api.md](../miniapps/js-api.md) (VMCRMUserApp), [miniapps/data-api.md](../miniapps/data-api.md) (CRUD через App.fetch).

---

## Быстрый старт

1. [concepts.md](concepts.md) — что такое Korn, квесты, лидерборды, магазин, профиль (прочитать первым).
2. [config-korgames.md](config-korgames.md) — секция `korgames` в config.json: game_id, reward_mode, items, package convention.
3. [api-reference.md](api-reference.md) — **полный справочник всех endpoints** с request/response (структуры, коды ошибок).
4. [client-api.md](client-api.md) — JS-обёртка вокруг `App.fetch`, особенности postMessage-транспорта.
5. [recipes.md](recipes.md) — **готовые рецепты** всех типовых механик (начислить/списать/магазин/топ/профиль/аватар).
6. [styling.md](styling.md) — стилистика игровых миниапов (body transparent, game-frame, CSS tokens).
7. [project-structure.md](project-structure.md) — модульная структура (frames/core/modules/locales/styles), i18n-паттерн.
8. [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) — построчный разбор эталона.
9. [checklist.md](checklist.md) — проверка перед релизом.

**TL;DR для агентов vibe-coding:** читай в порядке 3 → 5 → 6 → 7. Это API + Recipes + Styling + Structure — 80% того, что нужно чтобы делать игру с одного промпта.

---

## Ключевые правила

- **Игры НЕ печатают Korn.** Эмиссия только из whitelisted событий платформы (login, streak, create_record, deploy_app, install_app, referral, quest). Твоя игра может:
  - Записать `score` в общий лидерборд (`POST /api/korgames/game/score`).
  - Продать `items` за Korn (юзер тратит из кошелька, который он наработал вне игры).
  - Участвовать в `pool`-режиме (вход в раунд за Korn, победитель забирает пул − комиссия платформы) — запланировано, в MVP не реализовано.

- **Все данные игры + магазина** регистрируются автоматически при установке — через секцию `korgames` в `config.json`. Не нужны собственные таблицы.

- **Состояние пользователя** — персональные флаги, настройки, сохранки — храни в `App.storage` (KV-хранилище на app_id + user).

- **Лимиты эмиссии.** Платформа имеет `daily_cap` (default 500 Korn/сутки), балансы «сгорают» за 90 дней неактивности.

---

## Что уже реализовано

| Механика | Статус | Документ |
|----------|--------|----------|
| Korn-баланс, кошелёк юзера | MVP ✅ | [concepts.md](concepts.md) |
| Стрик ежедневного входа | MVP ✅ | [concepts.md#стрик](concepts.md) |
| Квесты (daily/weekly/onboarding) | MVP ✅ | [concepts.md#квесты](concepts.md) |
| Лидерборд по earn Korn | MVP ✅ | [client-api.md](client-api.md) |
| Игровой профиль (display_name, avatar, bio) | MVP ✅ | [client-api.md](client-api.md) · [project-structure.md](project-structure.md) |
| Регистрация игры из config.json | MVP ✅ | [config-korgames.md](config-korgames.md) |
| Магазин items за Korn | MVP ✅ | [client-api.md](client-api.md) |
| Score в игре + sys_game_scores | MVP ✅ | [client-api.md](client-api.md) |
| i18n (EN/RU) | MVP ✅ | [project-structure.md#i18n](project-structure.md) |
| Локальный топ по game score | MVP ✅ (через `/db/sys_game_scores.json`) | [coin-clicker-walkthrough.md](coin-clicker-walkthrough.md) |
| Pool-режим (ставка в раунд) | планируется | — |
| Pvp / team-battle | не в scope MVP | — |

---

## Связь с платформой

Серверная документация модуля — в core docs: `sited_core3php8/Docs/korgames/`:

| Файл | Что внутри |
|------|-----------|
| `index.md` | обзор, зависимости, dev-конфиг |
| `architecture.md` | слои, flow событий, точки расширения |
| `catalogs.md` | 9 таблиц `sys_*` (точные колонки для прямых запросов через `/db/sys_*.json`) |
| `services-and-api.md` | `\korgames\Games::*` методы + спецификация `/api/korgames/*` |
| `hooks-and-cron.md` | 5 хуков ядра + 3 cron |
| `security.md` | hard-gate `sys_*`, source whitelist |
| `gotchas.md` | 12 известных ловушек (включая `App.fetch` obёртку) |

Для разработки game-миниапа знать бекенд не обязательно: всё реализуется через `App.fetch('/api/korgames/*')`. Но если строишь серверную интеграцию или добавляешь свой condition_type квеста — идёшь в backend docs.
