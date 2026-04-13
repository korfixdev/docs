# Настройки каталогов

> **См. также:** [self-provisioning.md](self-provisioning.md) · [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [storage-and-hooks.md](storage-and-hooks.md)
> **← [INDEX](INDEX.md)**

Пользователь может настраивать отображение каталога через модалку (шестерёнка в хедере списка).
Приложение может читать и сохранять эти настройки, чтобы отображать данные
в том же виде, что и стандартный интерфейс CRM.

---

## Получение настроек

```js
const resp = await App.fetch('/db/tt_tasks/catalog/settings.json');
```

Ответ содержит три блока:

| Поле | Описание |
|------|----------|
| `data.table_fields` | Колонки таблицы: порядок, заголовки, видимость |
| `data.settings` | Пользовательские настройки по типам (interface, table, scheme) |
| `data.search_form` | Поля формы поиска/фильтрации |

---

## Типы настроек

### 1. interface — общие настройки каталога

Хранится в: `settings.interface.{catalog}`

| Поле | Тип | Описание |
|------|-----|----------|
| `settingsInterfaceCatalogname` | string | Пользовательское название каталога (переименование в меню и хедере) |
| `settingsInterfaceImgUrl` | string | Альтернативный путь для изображений (если поле `doc` есть в схеме) |
| `settingsInterfaceCatalogKanban` | string | Поле для канбан-представления (select или select_from_table) |

Пример значения:
```json
{
    "settingsInterfaceCatalogname": "Мои задачи",
    "settingsInterfaceImgUrl": "/data/custom-images"
}
```

**Чтение:**
```js
const resp = await App.fetch('/db/tt_tasks/catalog/settings.json');
const iface = resp.data.settings?.interface;
if (iface) {
    const customName = iface.value.settingsInterfaceCatalogname;
    // customName = 'Мои задачи' (или undefined если не переименован)
}
```

---

### 2. table — порядок и видимость колонок таблицы

Хранится в: `settings.table.{catalog}`

Значение — объект с единственным ключом `settingsFieldsOrder`: массив объектов `{name, value}`.

| Поле | Тип | Описание |
|------|-----|----------|
| `name` | string | Имя поля (field_name из схемы) |
| `value` | boolean | `true` = колонка видна, `false` = скрыта |

Порядок элементов массива = порядок колонок в таблице.

Пример значения:
```json
{
    "settingsFieldsOrder": [
        {"name": "name",      "value": true},
        {"name": "status",    "value": true},
        {"name": "person_id", "value": true},
        {"name": "ts",        "value": false},
        {"name": "complete",  "value": false}
    ]
}
```

**Чтение через table_fields:**

`data.table_fields` — уже обработанный результат: настройки пользователя применены поверх
дефолтных колонок каталога. Порядок ключей = порядок отображения:

```js
const resp = await App.fetch('/db/tt_tasks/catalog/settings.json');
const columns = resp.data.table_fields;

// columns = {
//   name:      { title: 'Название',    sort_by: 'ASC' },            // видно (нет checked)
//   status:    { title: 'Статус',      sort_by: 'ASC' },            // видно
//   person_id: { title: 'Исполнитель', sort_by: 'ASC', checked: false },  // скрыто
// }

// Видимые колонки в порядке пользователя
const visible = Object.entries(columns)
    .filter(([_, col]) => col.checked !== false);
```

---

### 3. scheme — порядок полей формы редактирования

Хранится в: `settings.scheme.{catalog}`

Значение — объект с ключом `settingsSchemeOrder`: массив объектов `{name, value}`.
Определяет порядок полей в форме добавления/редактирования и распределение по вкладкам.

| Поле | Тип | Описание |
|------|-----|----------|
| `name` | string | Имя поля (field_name из схемы). Поля типа `delimeter_line` = заголовки вкладок |
| `value` | string | `"1"` = поле включено |

Пример значения:
```json
{
    "settingsSchemeOrder": [
        {"name": "delimiter1",  "value": "1"},
        {"name": "name",        "value": "1"},
        {"name": "status",      "value": "1"},
        {"name": "person_id",   "value": "1"},
        {"name": "delimiter2",  "value": "1"},
        {"name": "cont",        "value": "1"}
    ]
}
```

Поля `delimeter_line` разделяют вкладки формы. Порядок элементов = порядок отображения.

---

## Сохранение настроек

Все настройки сохраняются через `apps_storage` с `app_id = -1` (системные, не привязаны к приложению маркетплейса):

```js
await App.fetch('/empty/db/apps_storage/add?edit=1&ajax=y', {
    method: 'POST',
    body: {
        'form[app_id]': -1,
        'form[name]': 'settings.table.tt_tasks',   // ключ: settings.{тип}.{каталог}
        'form[value]': JSON.stringify(data),         // значение: JSON
        submit: 'ok'
    }
});
```

### Сохранение для себя vs для всех

| Параметр | Значение | Результат |
|----------|----------|-----------|
| без `from_group` | — | Настройки сохраняются для текущего пользователя |
| `from_group: 1` | true | Настройки применяются ко всем пользователям группы |

Личные настройки имеют приоритет над групповыми.

### Сброс настроек

Для сброса — сохранить с `form[hidden] = 1`:

```js
await App.fetch('/empty/db/apps_storage/add?edit=1&ajax=y', {
    method: 'POST',
    body: {
        'form[app_id]': -1,
        'form[name]': 'settings.table.tt_tasks',
        'form[hidden]': 1,
        submit: 'ok'
    }
});
```

---

## Полный use-case: таблица с настройками пользователя

```js
const App = new VMCRMUserApp();

async function renderTable(catalog) {
    // 1. Получаем настройки + схему + данные параллельно
    const [settingsResp, schemaResp, dataResp] = await Promise.all([
        App.fetch(`/db/${catalog}/catalog/settings.json`),
        App.fetch(`/db/${catalog}/sheme.json`),
        App.fetchAll(`/db/${catalog}.json`)
    ]);

    const columns = settingsResp.data.table_fields;
    const schema  = schemaResp.data;
    const items   = Array.isArray(dataResp?.data) ? dataResp.data : [];

    // 2. Видимые колонки в порядке пользователя
    const visibleCols = Object.entries(columns)
        .filter(([_, col]) => col.checked !== false)
        .map(([name, col]) => ({
            name,
            title: col.title,
            type: schema[name]?.type,
            arr: schema[name]?.arr   // для select — варианты
        }));

    // 3. Рендер заголовков
    let html = '<table><thead><tr>';
    visibleCols.forEach(col => { html += `<th>${col.title}</th>`; });
    html += '</tr></thead><tbody>';

    // 4. Рендер строк с подстановкой значений select
    items.forEach(item => {
        html += '<tr>';
        visibleCols.forEach(col => {
            let val = item[col.name] ?? '';
            // Для select — показываем текст вместо ключа
            if (col.type === 'select' && col.arr && col.arr[val] !== undefined) {
                val = col.arr[val];
            }
            html += `<td>${val}</td>`;
        });
        html += '</tr>';
    });
    html += '</tbody></table>';
    return html;
}
```

---

## Полный use-case: редактор настроек колонок

Приложение может предоставить собственный UI для настройки колонок:

```js
// 1. Загрузить текущие настройки
const resp = await App.fetch(`/db/${catalog}/catalog/settings.json`);
const columns = resp.data.table_fields;

// 2. Показать чекбоксы (пользователь включает/выключает и сортирует)
// ... UI с drag & drop ...

// 3. Сохранить изменённый порядок
const newOrder = [
    { name: 'name',      value: true },
    { name: 'status',    value: true },
    { name: 'person_id', value: false },  // скрыть
];

await App.fetch('/empty/db/apps_storage/add?edit=1&ajax=y', {
    method: 'POST',
    body: {
        'form[app_id]': -1,
        'form[name]': `settings.table.${catalog}`,
        'form[value]': JSON.stringify({ settingsFieldsOrder: newOrder }),
        submit: 'ok'
    }
});

// 4. Перезагрузить страницу чтобы применить
App.reload();
```

---

## Справочник ключей

| Ключ в storage | Тип настроек | Влияет на |
|----------------|-------------|-----------|
| `settings.interface.{catalog}` | Общие | Название каталога в меню, путь изображений |
| `settings.table.{catalog}` | Таблица | Порядок и видимость колонок в списке |
| `settings.scheme.{catalog}` | Форма | Порядок полей и вкладки в форме редактирования |

Все ключи имеют формат `settings.{type}.{catalog}`, где `{catalog}` — alias каталога (например `tt_tasks`, `b2b_orders`, `custom_tickets`).

---

**← [INDEX](INDEX.md)**
