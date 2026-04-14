# Работа с данными каталогов

> **См. также:** [js-api.md](js-api.md) · [storage-and-hooks.md](storage-and-hooks.md) · [self-provisioning.md](self-provisioning.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [Home](index.md)**

CRUD-операции над каталогами, форматы запросов, фильтрация и пагинация.

---

## ⚠️ Ключевое правило: какой endpoint откуда использовать

У Korfix **два разных endpoint'а** для работы с данными — часто путают. Правило простое:

| Откуда делается запрос | Какой endpoint | Авторизация | Формат полей |
|---|---|---|---|
| **Внутри iframe миниапа** (`App.fetch`) | `/db/{catalog}.json` | сессия браузера (cookie) | `form[name]=value` |
| **Снаружи**: curl, тесты, скрипты, серверные интеграции, webhook'и, n8n, Bitrix24 | `/api/db/{catalog}` | `Authorization: Bearer <token>` | `name=value` (плоско) |

**НЕ перепутай:** `/db/` снаружи iframe вернёт 302-редирект на логин (нет сессии), `/api/db/` внутри iframe без токена — 401. Типичная ловушка: агент видит `/db/...` в коде приложения, копирует в curl, получает редирект, начинает «чинить авторизацию» — не надо, просто замени `/db/` на `/api/db/` и добавь Bearer.

### Проверка доступа к каталогу перед разработкой

Всегда начинай с `curl` в `/api/db/` с токеном — чтобы убедиться что токен имеет доступ:

```bash
# Проверить что токен видит каталог
curl -sI "https://panel.korfix.ru/api/db/{catalog}?limit=1" \
  -H "Authorization: Bearer {TOKEN}"
# HTTP/2 200 — ок, доступ есть
# HTTP/2 403 — токен не имеет класса db_{catalog}_get (добавить в /db/api)
# HTTP/2 404 — каталога не существует (проверь имя, особенно префикс custom_)

# Получить список всех доступных токену каталогов
curl -s "https://panel.korfix.ru/api/db/getcatalogs" \
  -H "Authorization: Bearer {TOKEN}"
```

Никогда не делай `curl https://panel.korfix.ru/db/{catalog}.json` — это только для внутри iframe, curl получит 302 на логин.

---

## Два способа работы с API

### 1. Через сессию (App.fetch — без токена)

`App.fetch()` проксирует запрос через `postMessage` в родительское окно.
Родительское окно выполняет `fetch(url)` **с куками авторизованного пользователя**.
Токен не нужен — авторизация по сессии.

```js
// Загрузка каталога — работает сразу, без токена
const resp = await App.fetch('/db/projects.json');

// Все страницы
const all = await App.fetchAll('/db/installed_apps.json');
```

**Доступно**: все каталоги, к которым у текущего пользователя есть доступ.
**Ограничение**: работает только внутри iframe маркетплейс-приложения.

### 2. Через Bearer-токен (REST API)

Для серверных интеграций, webhook'ов и внешних приложений.
Токен из `/db/api`, определяет доступные каталоги и методы.

```js
// Из приложения — с токеном в query string
const resp = await App.fetch('/api/db/projects?token=YOUR_TOKEN');

// getcatalogs — список доступных каталогов
const catalogs = await App.fetch('/api/db/getcatalogs?token=YOUR_TOKEN');

// /api/user/tariff — тариф и биллинговая инфа текущего пользователя
const billing = await App.fetch('/api/user/tariff');
// data: { tarif, tarif_name, balance, discount, discount_date, payment_date, price, ... }
```

**Настройка**: `/db/api` -> создать токен -> указать разрешённые классы API.

### Когда какой использовать

| Сценарий | Способ |
|----------|--------|
| Виджет загружает данные для отображения | `App.fetch('/db/catalog.json')` — сессия |
| Виджет создаёт/редактирует элемент | `App.fetch('/db/catalog/add?edit&ajax=1', {method:'POST'})` — сессия |
| Тестирование API с конкретным токеном | `/api/db/catalog?token=XXX` — токен |
| Серверный webhook (afterSave) | `Authorization: Bearer XXX` — токен |
| Внешний сервис (n8n, Bitrix24) | `Authorization: Bearer XXX` — токен |

### Формат полей: form[] vs плоские

| Эндпоинт | Формат полей | Обёртка |
|----------|-------------|---------|
| `/db/catalog/...` | `form[name]=value` | нужна `form[]` |
| `/api/db/catalog` | `name=value` | **без** `form[]` |

Правило простое — по эндпоинту:
- **`/db/...`** — всегда `form[]`
- **`/api/db/...`** — всегда плоские поля

Это единое правило **для любого способа вызова** — App.fetch(), curl, внешний сервис.

```js
// /db/ — с form[]
App.fetch('/db/projects/add?edit&ajax=1', {
    method: 'POST',
    body: { 'form[name]': 'Проект', submit: 1 }
});

// /api/db/ — без form[]
App.fetch('/api/db/custom_dbtables', {
    method: 'POST',
    body: { name: 'My Table', dbname: 'mytable', submit: 1 }
});
```

```bash
# curl к /api/ — тоже без form[]
curl -X POST "https://panel.korfix.ru/api/db/projects" \
  -H "Authorization: Bearer TOKEN" \
  -F "name=Проект" -F "submit=1"
```

---

## Работа с данными каталогов

### Чтение списка

```js
const resp = await App.fetch('/db/projects.json');
// resp.data — массив элементов
// resp.total — общее количество

// С фильтром
const resp = await App.fetch('/db/projects.json?form[status]=active');

// Все страницы сразу
const all = await App.fetchAll('/db/projects.json');
```

> **`.json` vs `/api/db/` — различия при загрузке списков.**
> Эндпоинт `/db/catalog.json` использует серверные фильтры каталога (видимость, права, пользовательские настройки), которые могут ограничивать выборку.
> Эндпоинт `/api/db/catalog?limit=999` выполняет прямую выборку с пагинацией и может возвращать больше записей.
> Если нужно получить **полный список** записей (например, все дашборды для селектора), предпочитайте `/api/db/`:
>
> ```js
> // Может вернуть неполный список из-за фильтров каталога
> const resp = await App.fetchAll('/db/dashboards.json');
>
> // Гарантированно все записи (до limit)
> const resp = await App.fetch('/api/db/dashboards?limit=999');
> ```

**Важно: нормализация ответа.** `resp.data` не всегда массив — может быть
объектом (единичная запись), `null`, или вложенным `resp.data.data`
(при проксировании через `App.fetch`). **Всегда** приводите к массиву:

```js
// Безопасное извлечение массива данных
function asArray(resp) {
    if (Array.isArray(resp?.data)) return resp.data;
    if (Array.isArray(resp?.data?.data)) return resp.data.data;
    return [];
}

const projects = asArray(await App.fetchAll('/db/tt_projects.json'));
const tasks    = asArray(await App.fetchAll('/db/tt_tasks.json'));
// Теперь .sort(), .filter(), .map() безопасны
```

### Чтение одного элемента

```js
const resp = await App.fetch('/db/projects/ALIAS.json');
// или через API:
const resp = await App.fetch('/api/db/projects/ALIAS');
```

### Создание элемента

```js
await App.fetch('/db/projects/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Новый проект',
        'form[status]': 'active',
        submit: 1
    }
});
```

> **Важно: `alias` — уникальный ключ записи.**
> В каждой таблице платформы есть два идентификатора: `id` (числовой timestamp) и `alias` (строка, уникальна в таблице).
> Обращение к элементам через URL и ссылки — **всегда по `alias`**: `/db/projects/ALIAS.json`, `/db/projects/ALIAS?edit`.
> При создании через API `form[alias]` имеет схемный дефолт `uniqid()`, но он вычисляется **один раз при загрузке схемы**.
> Если создаёте несколько записей в одном запросе или в цикле, **явно генерируйте уникальный alias** для каждой:
>
> ```js
> // Безопасная генерация alias при массовом создании
> const alias = Date.now().toString(36) + Math.random().toString(36).substr(2, 8);
> await App.fetch('/db/projects/add?edit&ajax=1', {
>     method: 'POST',
>     body: {
>         'form[alias]': alias,
>         'form[name]': 'Новый проект',
>         submit: 1
>     }
> });
> ```

### Системные поля записей

Каждая запись в каталоге содержит набор системных полей. Эти поля управляют идентификацией, принадлежностью и видимостью записи. При создании через API их нужно учитывать явно.

#### `id` и `alias` — два идентификатора

| Поле | Тип | Назначение |
|------|-----|-----------|
| `id` | int / bigint | Числовой идентификатор (обычно timestamp создания). Используется для связей между таблицами (FK) |
| `alias` | varchar, unique | Строковый уникальный ключ. Используется во всех URL: `/db/catalog/ALIAS`, `/db/catalog/ALIAS.json`, `/db/catalog/ALIAS?edit` |

- URL элемента — **всегда alias**, не id: `/db/projects/6989d171cce41`
- Связи между таблицами (FK) — обычно **id**: `board_id` в `dashboard_widgets` ссылается на `dashboards.id`
- При создании: `alias` по умолчанию генерируется через `uniqid()`, но **дефолт вычисляется один раз при загрузке схемы**. При массовом создании (цикл) — генерируйте явно
- `id` обычно заполняется автоматически (timestamp), передавать не нужно

#### `from_auth` и `from_group` — принадлежность и видимость

Большинство каталогов содержат поля `from_auth` и `from_group`, которые определяют, кому принадлежит и кому видна запись.

| Поле | Значение | Смысл |
|------|----------|-------|
| `from_auth` | `0` | Запись видна **всем пользователям группы** (публичная в рамках группы) |
| `from_auth` | `USER_ID` | Запись видна **только этому пользователю** (персональная) |
| `from_group` | `0` | Запись доступна **всем группам** (глобальная, без возможности удалить) |
| `from_group` | `GROUP_ID` | Запись принадлежит конкретной группе |

**Правила при создании записей через API:**

1. **Для не-админов** схема автоматически выставляет `from_auth = SESSION_USER_ID` (скрытое поле с дефолтом). Явная передача не нужна.

2. **Для администраторов** `from_auth` — это `select` с вариантами `{0: 'Всем', USER_ID: 'Персональный'}`. Если не передать `form[from_auth]`, запись станет публичной (`from_auth = 0`). Чтобы создать персональную запись — передайте явно.

3. **Определение текущего user ID** — из схемы каталога:

```js
// Загружаем схему — from_auth.arr содержит {0: 'Всем', USER_ID: 'Персональный'}
const schema = await App.fetch('/db/dashboard_widgets/sheme.json');
const fromAuthArr = schema?.data?.from_auth?.arr || {};
const currentUserId = Object.keys(fromAuthArr).find(k => k !== '0') || 0;
```

4. **Пример: массовое создание с корректной принадлежностью:**

```js
// Определяем ID текущего пользователя
const schema = await App.fetch('/db/my_catalog/sheme.json');
const arr = schema?.data?.from_auth?.arr || {};
const userId = Object.keys(arr).find(k => k !== '0') || 0;

for (const item of items) {
    const alias = Date.now().toString(36) + Math.random().toString(36).substr(2, 8);
    await App.fetch('/db/my_catalog/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[alias]': alias,
            'form[name]': item.name,
            'form[from_auth]': userId,
            'form[from_group]': userId,
            submit: 1
        }
    });
}
```

> **Итого:** при создании записей всегда передавайте `form[alias]` (уникальный), а если нужна персональная видимость — `form[from_auth]` и `form[from_group]`.

### Редактирование

```js
await App.fetch(`/db/projects/${alias}?edit&ajax=1`, {
    method: 'POST',
    body: {
        'form[name]': 'Обновлённое название',
        'form[id]': id,
        'form[alias]': alias,
        submit: 1
    }
});
```

### Удаление

Удаление — это **не HTTP DELETE**. Запись помечается `hidden=1` и попадает в корзину.
Из корзины пользователь может удалить окончательно.

```js
// Через /db/ — мягкое удаление (в корзину)
await App.fetch(`/db/projects/${alias}?udel&ajax=1`, { method: 'POST' });

// Через /api/db/ — тоже мягкое удаление (hidden=1)
await App.fetch(`/api/db/projects/${id}`, {
    method: 'POST',
    body: { hidden: 1, submit: 1 }
});
```

**Важно**: `hidden` должно быть в схеме каталога. Для кастомных каталогов (`custom_*`)
поле `hidden` присутствует автоматически.

### Кастомные каталоги (`custom_*`) — особенности CRUD

Для кастомных каталогов (созданных через self-provisioning) рекомендуется использовать
**`/api/db/`** вместо `/db/` для операций записи:

```js
// СОЗДАНИЕ — /api/db/ сохраняет alias корректно
await App.fetch(`/api/db/custom_my_catalog`, {
    method: 'POST',
    body: {
        alias: uid(),                    // без form[]
        custom_name: 'Название',
        from_auth: currentUserId,
        from_group: currentUserId,
        submit: 1
    }
});

// РЕДАКТИРОВАНИЕ — /api/db/ по id
await App.fetch(`/api/db/custom_my_catalog/${id}`, {
    method: 'POST',
    body: {
        custom_name: 'Новое название',
        submit: 1
    }
});

// УДАЛЕНИЕ — hidden=1
await App.fetch(`/api/db/custom_my_catalog/${id}`, {
    method: 'POST',
    body: { hidden: 1, submit: 1 }
});
```

> **Почему `/api/db/` а не `/db/`?** Эндпоинт `/db/.../add?edit&ajax=1`
> для кастомных каталогов может **игнорировать `form[alias]`** — запись создаётся без alias.
> Без alias невозможно редактирование через `/db/{catalog}/{alias}?edit`.
> Через `/api/db/` alias сохраняется корректно.
>
> **Адресация в `/api/db/`**: POST-обновление работает по `/{id}` (числовой идентификатор).
> По alias для кастомных каталогов POST может вернуть `item not found`.

### `from_auth` и `from_group` — обязательны

При создании записей **всегда** передавайте `from_auth` и `from_group`:

```js
body: {
    from_auth: currentUserId,   // владелец записи
    from_group: currentUserId,  // группа
    submit: 1
}
```

Записи с пустыми `from_auth`/`from_group` принадлежат суперадмину, видны всем аккаунтам
и **неуправляемы** — их нельзя редактировать или удалять через интерфейс платформы.

### Получение схемы каталога

```js
const schema = await App.fetch('/db/projects/sheme.json');
// schema.data — объект с описаниями полей по ключам
```

Для полей типа `select` — варианты в `arr`:

```js
const statusField = schema.data.status;
// statusField.type = 'select'
// statusField.arr = {0: 'Новый', 10: 'В работе', 40: 'Завершено'}
```

Для полей типа `select_from_table` — варианты в `arr` (до 200 записей), плюс метаданные:

```js
const personField = schema.data.person_id;
// personField.type = 'select_from_table'
// personField.catalog = 'auth_pers'           — имя связанного каталога
// personField.total = 18                      — общее количество записей
// personField.arr = {123: 'Иванов И.И.', 456: 'Петров П.П.'}  — первые 200 вариантов
// personField.ex_table_field = 'author_comment'  — поле для отображения
// personField.id_ex_table = 'author_id'          — поле-ключ
```

Для небольших справочников (total <= 200) — `arr` содержит все варианты:

```js
const schema = await App.fetch('/db/tt_tasks/sheme.json');
const options = schema.data.person_id.arr;
const select = document.getElementById('personSelect');
for (const [id, name] of Object.entries(options)) {
    select.add(new Option(name, id));
}
```

Для больших справочников (total > 200) — пагинация по конкретному полю:

```js
const field = schema.data.client_id;
if (field.total > Object.keys(field.arr).length) {
    // Есть ещё страницы — загружаем вторую
    const page2 = await App.fetch('/db/tt_tasks/sheme.json?field=client_id&p=2');
    const moreOptions = page2.data.client_id.arr;
    // ... добавить в select или реализовать autocomplete
}
```

Параметры пагинации:
- `field=person_id` — загрузить arr только для указанного поля (экономит трафик)
- `p=2` — номер страницы (по 200 записей)

### Пользовательские настройки каталога

Порядок колонок, видимость полей, переименование каталога — пользователь настраивает
через шестерёнку в хедере. Приложение может читать и сохранять эти настройки.

Подробный справочник с примерами и use-case: [catalog-settings.md](catalog-settings.md)

### Формат URL

```
/db/{catalog}.json          -- список элементов (GET)
/db/{catalog}/{alias}       -- конкретный элемент
/db/{catalog}/{alias}.json  -- элемент в JSON
/db/{catalog}/add?edit      -- страница создания
/db/{catalog}/sheme.json    -- схема (поля, типы, варианты)
/db/{catalog}/catalog/settings.json -- пользовательские настройки колонок
/empty/db/{catalog}         -- без шаблона (для модалок)
/api/db/{catalog}           -- REST API (JSON)
```

### Фильтрация

```
/db/projects.json?form[status]=active&form[client_id]=123
```

### Пагинация

```
/db/projects.json?p=2
```

### Параметры запроса (/api/db/)

Через `/api/db/{catalog}` доступны дополнительные параметры управления выдачей:

| Параметр | Описание | Пример |
|----------|----------|--------|
| `filter[field]=value` | Фильтр по полю | `filter[status]=active` |
| `order_by=field` | Сортировка по полю | `order_by=name` |
| `order=ASC\|DESC` | Направление сортировки | `order=DESC` |
| `limit=N` | Количество записей (по умолч. 20) | `limit=50` |
| `offset=N` | Смещение | `offset=20` |
| `select=f1,f2` | Выбор конкретных полей | `select=name,status` |
| `load_values=1` | Подменить ID связанных полей на отображаемые значения | `load_values=1` |

**`load_values`** — для полей типа `select_from_table` вместо числового ID
возвращает текстовое значение (имя сотрудника вместо ID). Удобно для отображения
без дополнительных запросов:

```js
// Без load_values: person_id = "1715761701"
// С load_values:   person_id = "Алексей Григорьев"
const resp = await App.fetch('/api/db/tt_tasks?load_values=1');
```

**`select`** — ограничивает набор полей в ответе. Поля с `_id` суффиксом
и обязательные поля всегда включаются:

```js
const resp = await App.fetch('/api/db/tt_tasks?select=name,status&limit=10');
```

**`order_by` + `order`** — сортировка работает только по полям, присутствующим в схеме:

```js
const resp = await App.fetch('/api/db/tt_tasks?order_by=name&order=ASC');
```

---

## Доступные каталоги

Полный список с примерами: [korfix-catalogs.md](korfix-catalogs.md)

Основные группы: AG (финансы), B2B (торговля), MD (производство),
TT (задачи), WH (склад), VRN (выездные работы), CRM, системные.

---

**Дальше:** [storage-and-hooks.md](storage-and-hooks.md) · **← [Home](index.md)**
