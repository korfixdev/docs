# Права на статусы (access_statuses)

> **См. также:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [catalog-rules.md](catalog-rules.md)
> **← [INDEX](INDEX.md)**

Каталог `access_statuses` управляет тем, какие статусы доступны
для каждой роли пользователя в каждом каталоге.

> Внутренняя механика (PHP, get_status_arr, get_disabled_statuses) — в [../backend/statuses.md](../backend/statuses.md)

---

## Принцип

В каталогах с полем `status` каждый статус — этап процесса.
Через `access_statuses` администратор ограничивает, какие статусы
доступны конкретной роли. Без записи — все статусы доступны.

**Пример:** менеджер переводит заказ в "В работе",
но только руководитель может поставить "Оплачен" или "Отменён".

---

## Структура записи

| Поле | Тип | Описание |
|------|-----|----------|
| `dbmodule` | select | Каталог |
| `role` | select | Роль (тип аккаунта) |
| `statuses` | multiselect | Разрешённые статусы (whitelist) |

Одна запись на пару каталог + роль. Alias: `{dbmodule}_{role}`.

**Доступ:** только root (создатель группы). Остальные роли, включая администраторов, не имеют доступа к управлению правами на статусы.

---

## Чтение настроек

```js
// Все правила
const resp = await App.fetch('/db/access_statuses.json');

// Для конкретного каталога
const resp = await App.fetch('/db/access_statuses.json?form[dbmodule]=b2b_orders');
resp.data.forEach(rule => {
    console.log(rule.role);      // "3"
    console.log(rule.statuses);  // "10,20,30"
});
```

---

## Создание правила

```js
await App.fetch('/db/access_statuses/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[dbmodule]': 'b2b_orders',
        'form[role]': '3',
        'form[statuses]': ['10', '20'],
        submit: 1
    }
});
```

---

## Получение каталогов, ролей и статусов

```js
const schema = await App.fetch('/db/access_statuses/sheme.json');

// Каталоги с поддержкой статусов
const catalogs = schema.data.dbmodule.arr;

// Роли (типы аккаунтов)
const roles = schema.data.role.arr;

// Статусы по каталогам
const statusesByModule = schema.data.dbmodule.group_statuses;
// { "b2b_orders": { "10": "Новый", "20": "В работе" }, ... }

// Или статусы конкретного каталога через его схему
const catSchema = await App.fetch('/db/b2b_orders/sheme.json');
const statuses = catSchema.data.status.arr;
```

---

## Сценарии для миниапов

### Визуализация процесса

```js
async function getStatusMatrix(catalog) {
    const [rulesResp, schemaResp] = await Promise.all([
        App.fetch(`/db/access_statuses.json?form[dbmodule]=${catalog}`),
        App.fetch(`/db/${catalog}/sheme.json`)
    ]);
    const allStatuses = schemaResp.data.status?.arr || {};
    return rulesResp.data.map(rule => ({
        role: rule.role,
        allowed: (rule.statuses || '').split(','),
        denied: Object.keys(allStatuses).filter(
            s => s && !(rule.statuses || '').split(',').includes(s)
        )
    }));
}
```

### Проверка доступа перед сменой статуса

```js
async function canSetStatus(catalog, statusId) {
    const resp = await App.fetch(`/db/access_statuses.json?form[dbmodule]=${catalog}`);
    if (resp.data.length === 0) return true;

    const params = await App.getRequestParams();
    const rule = resp.data.find(r => r.role === params.data.accountType);
    if (!rule) return true;

    return (rule.statuses || '').split(',').includes(String(statusId));
}
```

### Проверка прав перед показом UI

Управление правами на статусы доступно только root (создателю группы).
Перед показом интерфейса настройки стоит проверить:

```js
const params = await App.getRequestParams();
if (params.data.userId !== params.data.groupId) {
    // Не root — показать предупреждение или скрыть секцию
    App.alert('Настройка прав на статусы доступна только владельцу аккаунта');
    return;
}
// Root — показать матрицу прав
```

### Настройка воркфлоу при установке

Приложение может при установке создать правила доступа к статусам,
настраивая бизнес-процесс под роли компании.

---

*Каталог: `/db/access_statuses` | Бэкенд: [../backend/statuses.md](../backend/statuses.md)*

**← [INDEX](INDEX.md)**
