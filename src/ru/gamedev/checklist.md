# Чеклист перед релизом игры

> **Навигация:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · [coin-clicker-walkthrough](coin-clicker-walkthrough.md) · **вы здесь**.

Универсальный miniapp-чеклист — в [miniapps/checklist.md](../miniapps/checklist.md). Ниже — специфика игровой части.

## Секция `korgames` в config.json

> Формат и поля — [config-korgames.md](config-korgames.md).

- [ ] `package` с префиксом `game-` (напр. `game-coin-clicker`) — обязательно для игровых миниапов
- [ ] `game_id` задан, в kebab-case, уникален. Согласован с package (`package = 'game-' + game_id`)
- [ ] `reward_mode: "score_only"` (единственное в MVP)
- [ ] `items[]` — у каждого `key`, `name`, `description`, `price_corn ≥ 0`, `max_per_user ≥ 0`
- [ ] `key` каждого item уникален в рамках игры
- [ ] `max_per_user = 1` для уникальных улучшений (иначе юзер купит 100 одинаковых)

## index.html

- [ ] `<meta charset="UTF-8">` в `<head>`
- [ ] Regular `<script>` ДО module-скрипта (для объявления глобалов)
- [ ] Module-скрипт импортирует VMCRMUserApp по абсолютному пути `/templates/def/db/marketplace/vmcrm-user-app.js`
- [ ] `window.App = new VMCRMUserApp()` — именно на `window`, чтобы глобалы из regular-скриптов видели

## Структура проекта

> Паттерн — [project-structure.md](project-structure.md).

- [ ] `frames/` содержит все HTML (`main.html`, опц. `settings.html`, `widget.html`)
- [ ] `core/` — singletons (api.js, i18n.js, storage.js)
- [ ] `modules/` — UI-логика по разделам (один файл на таб/модалку)
- [ ] `locales/` — en.json + ru.json как минимум, дефолт EN
- [ ] `styles/` — CSS в отдельной папке
- [ ] `README.md` в корне с историей изменений и структурой
- [ ] **body прозрачный** (`background: transparent`) — тематический фон игры **только** внутри контейнера (`.cc-frame` или эквивалент). См. [project-structure.md § body не трогаем](project-structure.md).

## JS — API

> Примеры вызовов — [client-api.md](client-api.md). Эталон — [coin-clicker-walkthrough.md § game.js](coin-clicker-walkthrough.md).

- [ ] Все вызовы через `App.fetch`, не `window.fetch`
- [ ] Body в POST — объект, не `JSON.stringify` + Content-Type
- [ ] Второй аргумент не передавать `undefined` → ошибка `Cannot read 'body' of null`. Используй `(...rest)` → `App.fetch(url, ...rest)`.
- [ ] Распаковка обёртки: `const r = await App.fetch(...); r = r?.data ?? r;`

## i18n

- [ ] Все пользовательские строки через `i18n.t('key')`, не захардкожены
- [ ] DOM-элементы с `data-i18n="key"` для автоматического перевода при смене языка
- [ ] `i18n.applyToDom(document)` вызывается после `i18n.init(App)` и после каждого `i18n.setLang(...)`
- [ ] Если рендеришь блок через `innerHTML = \`${i18n.t('...')}...\``, не забудь перерисовать при смене языка
- [ ] `en.json` содержит все ключи, `ru.json` — их полную копию (без потерь)

## Обработка ошибок

- [ ] Проверка `r.status === 'success'` перед использованием `r.data`
- [ ] `try/catch` вокруг `App.fetch` — чтобы не упасть из-за таймаута или permission_denied
- [ ] `App.alert` / inline-сообщение юзеру при ошибке — не тихо

## Деплой

- [ ] Деплой через `POST /api/marketplace/deploy/{id}` — update + refresh в одной операции
- [ ] После деплоя проверить `crm__marketplace.appconfig` и `cont` заполнены (не NULL)
- [ ] Установить под тестовым юзером → проверить что в `sys_registered_games` появилась запись с твоим `game_id`
- [ ] Проверить `sys_game_items` содержит все items из config

## Функциональные тесты

- [ ] Откровение приложения не бросает ошибок в console
- [ ] `GET /api/korgames/game/inventory?game_id=X` возвращает пустой массив у нового юзера
- [ ] Раунд игры → `POST /api/korgames/game/score` вернул success, `sys_game_scores` +1 запись
- [ ] Магазин — кнопка «Купить» disabled если не хватает Korn, enabled если хватает
- [ ] Покупка → `sys_game_purchases` +1, `sys_transactions` +1 (type='game_buy'), баланс уменьшился
- [ ] После покупки эффект применяется (инвентарь перечитывается в `init()`)
- [ ] `max_per_user` работает — вторая попытка купить тот же item возвращает `limit_exceeded`

## UX

- [ ] Состояние «Загрузка…» видно пока грузятся API
- [ ] Баланс/стрик отображаются где-то в UI (если игра длинная)
- [ ] После раунда показан результат (score) и что дальше делать
- [ ] Кнопка «Играть ещё» / «В магазин» — очевидные дальнейшие действия
- [ ] Магазин показывает почему item недоступен (не хватает Korn / уже куплено / лимит)

## Безопасность

- [ ] Не пытаешься самостоятельно начислять Korn — это задача платформы
- [ ] Не обходишь магазин — покупка только через `/api/korgames/game/buy` (атомарная транзакция)
- [ ] Не полагаешься на клиентский score для критичных наград (его можно подделать через devtools) — сейчас в MVP без анти-чита

## Документация

- [ ] `about` в config.json — markdown-строка с 5 разделами (Что делает / Где появляется / Возможности / Как пользоваться / Настройка)
- [ ] README.md в корне игры с историей изменений — помогает тебе самому через месяц
- [ ] Если игра сложная — ссылка на wiki/блог-пост в `about`

---

## Часто забываемые вещи

1. **`permissions`** — явно указывай `storage`/`navigate`/`modal`. Без этого получишь warning «полный доступ (legacy)».
2. **`logo: "icon.svg"`** — файл должен быть в zip. Без logo приложение в маркетплейсе выглядит пустым.
3. **`tags`** — через запятую. Помогает поиску в маркетплейсе.
4. **`version`** — bump при каждом deploy, иначе пользователи не увидят обновления.
5. **`about`** — markdown **одной JSON-строкой** с `\n`. Объектный формат `{short, full}` ломает `VMCRM\Models\UserApps::buildMetaUpdates` (есть защита, но не надо провоцировать).

---

## После релиза

- Проверить что в `sys_registered_games` твоя игра с `is_active=1`.
- Сыграть самому под тестовым юзером — всё ли работает end-to-end.
- Добавить игру в портфель разработчика (если есть публичный листинг).
- Мониторить `sys_game_scores` — если никто не играет, причина скорее в маркетплейсе/каталогизации, чем в коде.
