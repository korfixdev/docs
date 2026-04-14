# Дашборды и виджеты

> **См. также:** [config-json.md](config-json.md) · [styling.md](styling.md) · [data-api.md](data-api.md) · [self-provisioning.md](self-provisioning.md)
> **← [Home](index.md)**

Дашборды — настраиваемые рабочие столы с виджетами. Приложение маркетплейса может
встраивать свои фреймы как виджеты дашборда, а также создавать стандартные виджеты
(графики, таблицы) через API.

---

## Структура данных

### Каталог dashboards

Рабочие столы. Каждый пользователь/группа может иметь несколько дашбордов.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | bigint | ID (используется как `board_id` в виджетах) |
| `alias` | varchar | Уникальный alias |
| `name` | varchar | Название дашборда |
| `prior` | int | Порядок сортировки |
| `from_auth` | int | Автор (0 = системный) |
| `from_group` | bigint | Группа |

### Каталог dashboard_widgets

Виджеты внутри дашборда. Привязка к дашборду через `board_id`.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | bigint | ID |
| `alias` | varchar | Уникальный alias |
| `name` | varchar | Заголовок виджета |
| `type` | varchar | Тип виджета (см. справочник ниже) |
| `width` | int | Ширина в колонках Bootstrap (1-12). 4 = треть, 6 = половина, 12 = полная |
| `aside` | int | Боковой виджет (0/1) |
| `options` | text | JSON с параметрами виджета (зависит от типа) |
| `board_id` | varchar | ID дашборда (связь с `dashboards.id`) |
| `prior` | int | Порядок сортировки (меньше = выше) |
| `from_auth` | int | Автор |
| `from_group` | bigint | Группа |

---

## Типы виджетов

### Графические (данные из каталогов)

| Тип | Описание | Визуализация |
|-----|----------|-------------|
| `bar-chart` | Столбчатая диаграмма | Chart.js Bar |
| `pie-chart` | Круговая диаграмма | Chart.js Pie |
| `doughnut-chart` | Кольцевая диаграмма | Chart.js Doughnut |
| `line-chart` | Линейная диаграмма | Chart.js Line |
| `aggr-table` | Сводная таблица | HTML table |

**Параметры options (JSON):**

```json
{
    "catalog": "ag_cashflows",
    "pre-filter": "39",
    "fieldX": "project_id",
    "fieldY": "summa",
    "groupType": "sum",
    "colors": ["#5F67A8", "#E6576F", "#4E8F98"]
}
```

| Параметр | Описание |
|----------|----------|
| `catalog` | Каталог-источник данных |
| `pre-filter` | ID сохранённого фильтра (из `/db/saved_filters`) |
| `fieldX` | Поле для оси X / группировки |
| `fieldY` | Поле для оси Y / значений |
| `groupType` | `"count"` — количество, `"sum"` — сумма |
| `colors` | Массив цветов (hex). Палитра Korfix по умолчанию |

### Табличные

| Тип | Описание | Параметры options |
|-----|----------|-------------------|
| `last-created` | Последние записи каталога | `catalog`, `fields[]`, `limit` |

```json
{
    "catalog": "b2b_orders",
    "fields": ["id", "status", "client_id", "summ"],
    "limit": "8"
}
```

| Параметр | Описание |
|----------|----------|
| `catalog` | Каталог-источник |
| `fields` | Массив имён полей для отображения в колонках |
| `limit` | Количество строк (по умолчанию 5) |

### Встроенные

| Тип | Описание |
|-----|----------|
| `user-info` | Карточка текущего пользователя |

### Маркетплейс-приложение

| Тип | Описание |
|-----|----------|
| `app-frame` | Iframe с фреймом установленного приложения |

```json
{
    "app_frame": "{token}:{frameName}"
}
```

| Параметр | Описание |
|----------|----------|
| `app_frame` | Формат: `{installed_apps.alias}:{имя_фрейма_из_urls}` |

Фрейм получает стандартные параметры через VMCRMUserApp с `catalog = "dashboard_widgets"`.

---

## Работа с API

### Получить список дашбордов

```js
// Рекомендуется /api/db/ — возвращает полный список без серверных фильтров каталога
const boards = asArray(await App.fetch('/api/db/dashboards?limit=999'));
// [{id, alias, name, prior, from_auth, from_group}, ...]
```

### Получить виджеты дашборда

```js
const widgets = asArray(await App.fetchAll('/db/dashboard_widgets.json?form[board_id]=' + boardId));
// [{id, alias, name, type, width, options, board_id, prior}, ...]
```

### Создать виджет

```js
// Столбчатая диаграмма по заказам
await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Заказы по клиентам',
        'form[type]': 'bar-chart',
        'form[width]': 6,
        'form[board_id]': boardId,
        'form[prior]': 1,
        'form[options]': JSON.stringify({
            catalog: 'b2b_orders',
            fieldX: 'client_id',
            fieldY: 'summ',
            groupType: 'sum'
        }),
        submit: 1
    }
});
```

### Создать виджет-таблицу

```js
// Последние задачи
await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Последние задачи',
        'form[type]': 'last-created',
        'form[width]': 12,
        'form[board_id]': boardId,
        'form[options]': JSON.stringify({
            catalog: 'tt_tasks',
            fields: ['name', 'status', 'person_id', 'complete'],
            limit: '10'
        }),
        submit: 1
    }
});
```

### Создать виджет из приложения (app-frame)

Приложение может программно добавить свой фрейм как виджет дашборда:

```js
// Нужен token инсталляции и имя фрейма из config.json urls
const params = await App.getRequestParams();
const appFrame = params.data.token + ':widget';  // token:frameName

await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Мой виджет',
        'form[type]': 'app-frame',
        'form[width]': 6,
        'form[board_id]': boardId,
        'form[options]': JSON.stringify({ app_frame: appFrame }),
        submit: 1
    }
});
```

### Создать виджет по данным кастомного каталога

Приложение с self-provisioning может создавать виджеты для своих данных:

```js
// Приложение создало каталог custom_tickets (сервис-деск)
// Теперь добавляет виджет на дашборд

// Круговая диаграмма тикетов по статусу
await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Тикеты по статусу',
        'form[type]': 'doughnut-chart',
        'form[width]': 4,
        'form[board_id]': boardId,
        'form[options]': JSON.stringify({
            catalog: 'custom_tickets',
            fieldX: 'custom_status',
            fieldY: 'id',
            groupType: 'count',
            colors: ['#5F67A8', '#4E8F98', '#E6576F', '#D6B075', '#45476A']
        }),
        submit: 1
    }
});

// Таблица последних тикетов
await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Последние тикеты',
        'form[type]': 'last-created',
        'form[width]': 8,
        'form[board_id]': boardId,
        'form[options]': JSON.stringify({
            catalog: 'custom_tickets',
            fields: ['name', 'custom_status', 'custom_priority', 'custom_category'],
            limit: '5'
        }),
        submit: 1
    }
});
```

---

## Use-case: создание дашборда при установке

Приложение может при первом запуске создать дашборд с преднастроенными виджетами:

```js
async function createDashboardWithWidgets() {
    // 1. Создать дашборд
    const resp = await App.fetch('/db/dashboards/add?edit&ajax=1', {
        method: 'POST',
        body: { 'form[name]': 'Сервис-деск', submit: 1 }
    });

    // 2. Получить ID созданного дашборда
    const boards = asArray(await App.fetchAll('/db/dashboards.json'));
    const board = boards.find(b => b.name === 'Сервис-деск');
    if (!board) return;

    // 3. Добавить виджеты
    const widgets = [
        { name: 'Тикеты по статусу', type: 'doughnut-chart', width: 4,
          options: { catalog: 'custom_tickets', fieldX: 'custom_status', fieldY: 'id', groupType: 'count' }},
        { name: 'По приоритету', type: 'bar-chart', width: 4,
          options: { catalog: 'custom_tickets', fieldX: 'custom_priority', fieldY: 'id', groupType: 'count' }},
        { name: 'Последние тикеты', type: 'last-created', width: 12,
          options: { catalog: 'custom_tickets', fields: ['name', 'custom_status', 'custom_priority'], limit: '8' }},
    ];

    for (let i = 0; i < widgets.length; i++) {
        await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
            method: 'POST',
            body: {
                'form[name]': widgets[i].name,
                'form[type]': widgets[i].type,
                'form[width]': widgets[i].width,
                'form[board_id]': board.id,
                'form[prior]': i + 1,
                'form[options]': JSON.stringify(widgets[i].options),
                submit: 1
            }
        });
    }

    App.alert('Дашборд "Сервис-деск" создан с 3 виджетами');
}
```

---

## Справочник цветов

Палитра Korfix для графиков:

```js
const colors = [
    '#5F67A8', '#45476A', '#E6576F', '#5388AF', '#4E8F98',
    '#CBDDA6', '#2C55BF', '#D6B075', '#A25E8B', '#D48474'
];
```

---

## Пример приложения

Полный рабочий пример работы с дашбордами и виджетами — приложение **dashboard-share** из коллекции эталонных миниапов Korfix.

Реализует экспорт/импорт виджетов между дашбордами и демонстрирует:
- Загрузку списка дашбордов через `/api/db/dashboards`
- Чтение виджетов по `board_id`
- Массовое создание виджетов с уникальными `alias`
- Явную передачу `from_auth` / `from_group` для корректной принадлежности
- Точку интеграции `itemsActions` (контекстное меню элемента каталога)

---

**Дальше:** [deploy.md](deploy.md) · **← [Home](index.md)**
