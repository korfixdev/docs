# Синхронизация с Bitrix24 (bitrix24_sync)

> **См. также:** [data-api.md](data-api.md) · [storage-and-hooks.md](storage-and-hooks.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [INDEX](INDEX.md)**

Платформа поддерживает двустороннюю синхронизацию каталогов с Bitrix24 CRM.
Миниапы могут читать и управлять настройками синхронизации.

> Внутренняя архитектура, REST-клиент, event.php — в [../backend/bitrix24-integration.md](../backend/bitrix24-integration.md)

---

## Каталоги

| Каталог | Описание |
|---------|----------|
| `bitrix24_sync` | Маппинг каталогов (какой локальный ↔ какой в Bitrix24) |
| `bitrix24_sync_fieldmap` | Маппинг полей (какое поле ↔ какое поле) |

---

## Чтение настроек синхронизации

```js
// Все маппинги каталогов
const resp = await App.fetch('/db/bitrix24_sync.json');
resp.data.forEach(mapping => {
    console.log(mapping.local_catalog);    // "md_project"
    console.log(mapping.bitrix_catalog);   // "deal"
    console.log(mapping.direction);        // "2" (двустороннее)
    console.log(mapping.event_types);      // "0,1" (ADD, UPDATE)
});

// Маппинги полей для конкретной синхронизации
const fields = await App.fetch(`/db/bitrix24_sync_fieldmap.json?form[sync_id]=${mapping.id}`);
fields.data.forEach(f => {
    console.log(f.local_field);   // "name"
    console.log(f.bitrix_field);  // "TITLE"
    console.log(f.values_map);    // "draft,1\nin_progress,2"
});
```

---

## Направления синхронизации

| Значение `direction` | Описание |
|----------------------|----------|
| `0` | Из Bitrix24 → локально |
| `1` | Локально → в Bitrix24 |
| `2` | Двустороннее |

## Типы событий

| Значение `event_types` | Описание |
|------------------------|----------|
| `0` | Добавление (ADD) |
| `1` | Обновление (UPDATE) |
| `2` | Удаление (DELETE) |

Хранятся через запятую: `"0,1"` = ADD + UPDATE.

---

## Каталоги Bitrix24

| Значение `bitrix_catalog` | Сущность |
|---------------------------|----------|
| `deal` | Сделки |
| `lead` | Лиды |
| `contact` | Контакты |
| `company` | Компании |
| `product` | Товары |
| `productsection` | Группы товаров |
| `item` | Smart-процессы |
| `user` | Пользователи |

---

## Создание маппинга каталогов

```js
// Синхронизировать заказы → сделки Bitrix24 (только обновления)
await App.fetch('/db/bitrix24_sync/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Заказы → Сделки',
        'form[local_catalog]': 'b2b_orders',
        'form[bitrix_catalog]': 'deal',
        'form[direction]': '1',            // в Bitrix
        'form[event_types]': ['0', '1'],   // ADD + UPDATE
        'form[hook_url]': 'https://mycompany.bitrix24.ru/rest/1/abc123/',
        submit: 1
    }
});
```

## Создание маппинга полей

```js
// Связать поле "name" с "TITLE" в Bitrix24
await App.fetch('/db/bitrix24_sync_fieldmap/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Название → TITLE',
        'form[sync_id]': syncId,           // ID записи из bitrix24_sync
        'form[local_field]': 'name',
        'form[bitrix_field]': 'TITLE',
        submit: 1
    }
});

// С маппингом значений (статусы)
await App.fetch('/db/bitrix24_sync_fieldmap/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Статус → STAGE_ID',
        'form[sync_id]': syncId,
        'form[local_field]': 'status',
        'form[bitrix_field]': 'STAGE_ID',
        'form[values_map]': 'new,NEW\nin_progress,EXECUTING\ncompleted,WON',
        submit: 1
    }
});
```

---

## Фильтрация событий

Можно синхронизировать только записи с определёнными значениями:

```js
// Синхронизировать только выигранные сделки
await App.fetch('/db/bitrix24_sync/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Только выигранные сделки',
        'form[local_catalog]': 'ag_projects',
        'form[bitrix_catalog]': 'deal',
        'form[direction]': '0',               // из Bitrix
        'form[event_types]': ['0', '1'],
        'form[hook_url]': 'https://mycompany.bitrix24.ru/rest/1/abc123/',
        'form[event_filter_by]': 'STAGE_ID',
        'form[event_filter_values]': 'WON',
        submit: 1
    }
});
```

---

## Сценарии для миниапов

### Визуализация статуса синхронизации

```js
// Проверить, синхронизирован ли каталог с Bitrix24
async function isSyncedWithBitrix(catalog) {
    const resp = await App.fetch(`/db/bitrix24_sync.json?form[local_catalog]=${catalog}`);
    return resp.data.length > 0;
}

// Получить все синхронизируемые каталоги
async function getSyncedCatalogs() {
    const resp = await App.fetchAll('/db/bitrix24_sync.json');
    return resp.data.map(m => ({
        local: m.local_catalog,
        remote: m.bitrix_catalog,
        direction: ['← Bitrix', '→ Bitrix', '↔ Bitrix'][m.direction]
    }));
}
```

### Мастер настройки интеграции

Приложение может предложить пошаговый мастер: выбор каталогов,
маппинг полей, тестирование соединения — упрощая настройку для пользователя.

### Мониторинг ошибок синхронизации

```js
// Логи событий интеграции
const logs = await App.fetch('/db/eventlogs.json?form[catalog]=bitrix24&limit=20');
logs.data.forEach(log => {
    console.log(log.ts, log.action, log.text_message);
});
```

---

## Важно

- Синхронизация работает через cron — не в реальном времени
- Входящие вебхуки ставятся в очередь (`core_evt`) и обрабатываются отложенно
- Защита от петли: событие от Bitrix24 не триггерит обратную синхронизацию
- `hook_url` — webhook URL из настроек Bitrix24 (Разработчикам → Входящий вебхук)
- Доступ к `bitrix24_sync`: root + администратор
- Маппинг значений (`values_map`): CSV-формат, `local_value,bitrix_value` на строку

---

*Каталоги: `/db/bitrix24_sync`, `/db/bitrix24_sync_fieldmap` | Бэкенд: [../backend/bitrix24-integration.md](../backend/bitrix24-integration.md)*

**← [INDEX](INDEX.md)**
