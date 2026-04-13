# Storage API — KV-хранилище

> **См. также:** [data-api.md](data-api.md) · [catalog-rules.md](catalog-rules.md) · [js-api.md](js-api.md) · [self-provisioning.md](self-provisioning.md)
> **← [INDEX](INDEX.md)**

Изолированное хранилище ключ-значение для каждого установленного приложения, а также вебхуки и afterSave.

---

Каждое установленное приложение имеет изолированное хранилище
(привязка по token).

```js
// Сохранить настройки
await App.storage.set('my.setting', 'value');

// Прочитать одну запись
const record = await App.storage.get('my.setting');
// record = {name: 'my.setting', value: 'hello', alias: '...', app_id: '...'}
const val = record?.value;  // 'hello'

// Все ключи приложения
const all = await App.storage.get('');
// all = [{name: 'key1', value: 'val1'}, {name: 'key2', value: 'val2'}, ...]

// Удалить
await App.storage.unset('my.setting');
```

> **Важно: `value` — всегда строка.** `App.storage.set(key, value)` передаёт value как
> строку через POST. Если передать объект или массив — сохранится `[object Object]`.
> Для хранения сложных структур используйте `JSON.stringify()` / `JSON.parse()`:
>
> ```js
> // Сохранить объект/массив
> const config = { theme: 'dark', columns: ['name', 'status'] };
> await App.storage.set('my.config', JSON.stringify(config));
>
> // Прочитать объект/массив
> const record = await App.storage.get('my.config');
> const config = record?.value ? JSON.parse(record.value) : {};
> ```
>
> **Формат ответа `get()`:**
> - Одна запись: возвращает объект `{name, value, alias, app_id, ...}` — значение в поле `.value`
> - Все записи (`get('')`): возвращает массив объектов `[{name, value, ...}, ...]`
> - Не найдено: возвращает `undefined` или второй аргумент (default)

### Вебхуки через storage

Приложение может подписаться на вебхуки через storage:

```js
// Хук на редактирование конкретного каталога
await App.storage.set(
    'event.hook.activity.projects.отредактировал.json',
    'https://example.com/api/webhook'
);

// Хук на все события всех каталогов
await App.storage.set(
    'event.hook.activity.*.*.json',
    'https://example.com/api/webhook'
);
```

Формат ключа: `event.hook.activity.{catalog}.{event}[.json]`
- `{catalog}` — имя каталога или `*`
- `{event}` — `добавил`, `отредактировал`, `удалил` или `*`
- `.json` в конце — отправлять как `application/json`

### afterSave — вебхук при изменении данных каталога

Вызывается при сохранении, удалении элемента каталога. POST-запрос на URL обработчика.

#### Два способа подписки

**1. Статический (в config.json)** — для удалённых приложений:

```json
{
  "catalogs": {
    "tt_tasks": { "afterSave": "webhook-frame" },
    "": { "afterSave": "webhook-frame" }
  }
}
```

`""` — подписка на все каталоги. `"webhook-frame"` — ключ из `urls`.

**2. Динамический (через storage)** — из любого приложения в рантайме:

```js
// Конкретный каталог
await App.storage.set('event.hook.after.save.tt_tasks.json', 'https://example.com/webhook');
// Все каталоги
await App.storage.set('event.hook.after.save.json', 'https://example.com/webhook');
```

Суффикс `.json` = `Content-Type: application/json`. Без суффикса = `application/x-www-form-urlencoded`.

#### Данные вебхука (payload)

| Поле | Тип | Описание |
|------|-----|----------|
| `catalog` | string | Имя каталога (`tt_tasks`, `b2b_orders`, ...) |
| `action` | string | Тип операции (см. таблицу ниже) |
| `cmd` | string | Alias элемента |
| `form` | object | **Все поля элемента после операции** |
| `prev_form` | object\|null | **Предыдущее состояние до операции.** `null` при создании нового элемента |
| `token` | string | Alias инсталляции приложения |
| `app_id` | string | Alias приложения в маркетплейсе |
| `domain` | string | Домен CRM |
| `user` | string | MD5-хеш логина пользователя |

#### Типы операций (action)

| action | Когда | form | prev_form |
|--------|-------|------|-----------|
| `save` | Создание или редактирование элемента | Данные после сохранения | Данные до изменения (`null` при создании) |
| `delete` | Удаление элемента (в корзину) | Данные удалённого элемента | Не передаётся |

#### Определение типа изменения

```php
// На стороне обработчика (PHP)
$form = $_POST['form'] ?? [];
$prev = $_POST['prev_form'] ?? [];

// Создание нового элемента
if (empty($prev)) {
    // Новый элемент создан
}

// Редактирование — сравнение полей
if (!empty($prev) && $form['status'] !== $prev['status']) {
    // Статус изменился
}

// Удаление
if ($_POST['action'] === 'delete') {
    // Элемент удалён
}
```

```js
// На стороне обработчика (Node.js / n8n)
const { action, form, prev_form, catalog } = req.body;

if (action === 'save' && !prev_form) {
    console.log('Создан новый элемент:', form.name);
}

if (action === 'save' && prev_form && form.status !== prev_form.status) {
    console.log(`Статус изменён: ${prev_form.status} → ${form.status}`);
}

if (action === 'delete') {
    console.log('Удалён:', form.name);
}
```

#### Примеры use-case

**Уведомление при смене статуса задачи:**

```js
// При установке приложения — подписаться на tt_tasks
await App.storage.set(
    'event.hook.after.save.tt_tasks.json',
    'https://myapp.example.com/api/task-status-changed'
);
```

Обработчик сравнивает `form.status` и `prev_form.status`:
- Если изменился → отправить уведомление в Telegram
- Если не изменился → игнорировать

**Автоматическое создание записи при новом заказе:**

```js
await App.storage.set(
    'event.hook.after.save.b2b_orders.json',
    'https://myapp.example.com/api/new-order'
);
```

Обработчик проверяет `prev_form === null` (новый заказ) и создаёт задачу в TT-модуле.

### event.hook.activity — вебхук на действия пользователя

Альтернативный тип хуков — срабатывает на записи в лог активности (действия пользователя).

```js
// Формат: event.hook.activity.{catalog}.{действие}[.json]
// {действие} = добавил, отредактировал, удалил или * для всех

await App.storage.set(
    'event.hook.activity.b2b_orders.добавил.json',
    'https://myapp.example.com/api/order-added'
);

// Все действия со всеми каталогами
await App.storage.set(
    'event.hook.activity.*.*.json',
    'https://myapp.example.com/api/all-activity'
);
```

Payload содержит те же поля (`catalog`, `token`, `app_id`, `domain`, `user`) плюс данные активности (`activity`, `activity_text`).

---

### Хуки установки и удаления приложения

Платформа вызывает события при установке и удалении приложения.
Удалённое приложение может использовать их для инициализации (создание структур данных,
регистрация вебхуков) или очистки при удалении.

| Событие | Когда | Данные |
|---------|-------|--------|
| `app.installed` | После установки приложения | `app_id`, `token`, `form`, `user_id`, `from_group` |
| `app.uninstalled` | После удаления приложения | `app_id`, `token`, `form`, `user_id`, `from_group` |

Для **remote-приложений** (с `install_url`) — платформа делает POST на URL фрейма
с этими данными. Приложение может выполнить инициализацию и ответить.

Для **локальных приложений** (zip) — хуки вызываются серверно через `int_done_run()`.
Локальное приложение не может на них подписаться напрямую, но может использовать
вебхук через storage:

```js
// При первом запуске — подписаться на событие удаления
// (чтобы очистить данные если приложение будет удалено)
await App.storage.set(
    'event.hook.activity.installed_apps.удалил.json',
    'https://example.com/api/app-cleanup'
);
```

---

**Дальше:** [self-provisioning.md](self-provisioning.md) · **← [INDEX](INDEX.md)**
