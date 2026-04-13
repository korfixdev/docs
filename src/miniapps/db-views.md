# Представления (custom_dbview)

> **См. также:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [self-provisioning.md](self-provisioning.md)
> **← [Home](index.md)**

Объединение двух каталогов в одно представление через MySQL VIEW + LEFT JOIN.
Результат работает как обычный каталог — с фильтрацией, API, виджетами.

---

## Когда использовать

### Отчёты по связанным данным

Нужен отчёт «заказы + клиенты» или «задачи + проекты + сотрудники»? Вместо того
чтобы загружать два каталога и мёржить в JS — создай view и работай как с одним.

```js
// Без view: два запроса + merge в JS
const orders = asArray(await App.fetchAll('/db/b2b_orders.json'));
const clients = asArray(await App.fetchAll('/db/b2b_clients.json'));
const merged = orders.map(o => ({
    ...o,
    clientName: clients.find(c => c.id === o.client_id)?.name
}));

// С view: один запрос, данные уже объединены
const data = asArray(await App.fetchAll('/db/custom_orders_clients_view.json'));
// data[0].t1_summ, data[0].t2_name — поля из обоих каталогов
```

### Сводные таблицы для дашборда

Виджет `last-created` и `aggr-table` на дашборде работают с одним каталогом.
View позволяет показать данные из двух каталогов в одном виджете без написания
кастомного приложения.

### Фильтрация по полям другого каталога

Обычный каталог фильтруется только по своим полям. View позволяет фильтровать
заказы по имени клиента, задачи по названию проекта и т.д.

### Экспорт объединённых данных

Стандартный экспорт каталога (CSV/API) выгрузит данные из view со всеми
объединёнными полями — без дополнительного кода.

---

## Как создать представление

### Через интерфейс

1. Перейти в `/db/custom_dbview`
2. Добавить запись:
   - **Название** — например, «Заказы с клиентами»
   - **Имя таблицы (dbname)** — `orders_clients` (будет `custom_orders_clients_view`)
   - **Каталог 1** — `b2b_orders`
   - **Поле каталога 1** — `client_id`
   - **Каталог 2** — `b2b_clients`
   - **Поле каталога 2** — `id`
3. Сохранить → MySQL VIEW создаётся автоматически

### Через API (из миниапа)

```js
// Создание view программно
await App.fetch('/db/custom_dbview/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[alias]': Date.now().toString(36) + Math.random().toString(36).substr(2, 8),
        'form[name]': 'Заказы с клиентами',
        'form[dbname]': 'orders_clients',
        'form[catalog1]': 'b2b_orders',
        'form[catalog1_field]': 'client_id',
        'form[catalog2]': 'b2b_clients',
        'form[catalog2_field]': 'id',
        'form[from_auth]': currentUserId,
        'form[from_group]': currentUserId,
        submit: 1
    }
});
```

---

## Структура данных view

### Именование полей

Поля из обоих каталогов получают префиксы для избежания конфликтов:

| Источник | Префикс | Пример |
|----------|---------|--------|
| Каталог 1 | `t1_` | `t1_summ`, `t1_status`, `t1_client_id` |
| Каталог 2 | `t2_` | `t2_name`, `t2_email`, `t2_phone` |

**Исключения:** системные поля `id`, `from_auth`, `from_group`, `hidden` берутся
из каталога 1 без префикса.

### SQL который генерируется

```sql
CREATE OR REPLACE VIEW custom_orders_clients_view AS
SELECT
  c1.id, c1.from_auth, c1.from_group, c1.hidden,
  c1.summ        AS t1_summ,
  c1.status      AS t1_status,
  c1.client_id   AS t1_client_id,
  c2.name        AS t2_name,
  c2.email       AS t2_email,
  c2.phone       AS t2_phone
FROM b2b_orders AS c1
LEFT JOIN b2b_clients AS c2 ON c1.client_id = c2.id
```

LEFT JOIN — все записи каталога 1 сохраняются, даже если нет совпадения
в каталоге 2 (поля `t2_*` будут `NULL`).

---

## Работа с view из миниапа

### Чтение данных

```js
// Как обычный каталог — .json или /api/db/
const data = asArray(await App.fetchAll('/db/custom_orders_clients_view.json'));

// Или через REST API
const data = asArray(await App.fetch('/api/db/custom_orders_clients_view?limit=999'));
```

### Фильтрация

```js
// Фильтр по полю каталога 1 (заказы со статусом "оплачен")
const paid = asArray(await App.fetchAll(
    '/db/custom_orders_clients_view.json?form[t1_status]=paid'
));

// Фильтр по полю каталога 2 (заказы клиента "Иванов")
const ivanov = asArray(await App.fetchAll(
    '/db/custom_orders_clients_view.json?form[t2_name]=Иванов'
));
```

### Получение схемы

```js
const schema = await App.fetch('/db/custom_orders_clients_view/sheme.json');
// schema.data содержит объединённые поля обоих каталогов с префиксами t1_/t2_
```

### Проверка существования view

```js
// Перед использованием — проверить что view существует
try {
    const resp = await App.fetch('/api/db/custom_orders_clients_view?limit=1');
    // View существует, работаем
} catch(e) {
    // View не создан — показать инструкцию пользователю
}
```

---

## Кейсы для миниапов

### Аналитическое приложение

Приложение P&L или BI-аналитика может при установке (self-provisioning) создать
нужные view и строить отчёты на объединённых данных — без множественных fetch:

```js
// При первом запуске: создать view если его нет
const views = asArray(await App.fetch('/api/db/custom_dbview?limit=999'));
const exists = views.some(v => v.dbname === 'cashflows_clients');

if (!exists) {
    await App.fetch('/db/custom_dbview/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[alias]': uid(),
            'form[name]': 'Операции с контрагентами',
            'form[dbname]': 'cashflows_clients',
            'form[catalog1]': 'ag_cashflows',
            'form[catalog1_field]': 'client_id',
            'form[catalog2]': 'ag_clients',
            'form[catalog2_field]': 'id',
            'form[from_auth]': currentUserId,
            'form[from_group]': currentUserId,
            submit: 1
        }
    });
}

// Далее работаем как с обычным каталогом
const data = asArray(await App.fetchAll('/db/custom_cashflows_clients_view.json'));
```

### Виджет дашборда

Виджет `last-created` для view покажет объединённые данные:

```json
{
    "catalog": "custom_orders_clients_view",
    "fields": ["t1_summ", "t1_status", "t2_name"],
    "limit": "10"
}
```

### Экспорт связанных данных

Приложение import-export может работать с view как с обычным каталогом —
экспорт CSV с полями из обоих источников.

---

## Ограничения

| Ограничение | Описание |
|-------------|----------|
| **Read-only** | View по умолчанию только для чтения (`ue=0, ud=0`). Для редактирования — работайте с исходными каталогами |
| **Статическая схема** | Добавление полей в исходный каталог НЕ обновляет view. Нужно пересоздать |
| **Нет CASCADE** | Удаление исходного каталога ломает view без предупреждения |
| **Два каталога** | Только JOIN двух каталогов. Для трёх — создайте view из view + третий каталог |
| **Производительность** | На больших таблицах (100k+ записей) view может работать медленно — MySQL не оптимизирует индексы для VIEW |
| **NULL при отсутствии** | LEFT JOIN: если в каталоге 2 нет совпадения, поля `t2_*` будут `NULL` |

---

## Permissions для приложений

Если миниап создаёт или читает view, добавь в `config.json`:

```json
"permissions": {
    "catalogs": {
        "custom_dbview": ["read", "write"],
        "custom_orders_clients_view": ["read"]
    }
}
```

`custom_dbview` — для создания view. `custom_*_view` — для чтения данных из конкретного view.

---

**← [Home](index.md)**
