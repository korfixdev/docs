# Настройки push-уведомлений (systempush_settings)

> **См. также:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [account-help.md](account-help.md)
> **← [Home](index.md)**

Каталог `systempush_settings` — персональные подписки пользователя
на push-уведомления из каталогов платформы.

---

## Как работает

Платформа отправляет push-уведомления при действиях в каталогах (добавление,
редактирование, удаление записей). По умолчанию уведомления **не приходят** —
пользователь сам выбирает, из каких разделов и о каких событиях получать push.

Каждая запись в `systempush_settings` — это подписка на один каталог
с опциональными фильтрами.

---

## Структура записи

| Поле | Тип | Описание |
|------|-----|----------|
| `alias` | hidden | md5(menu + userId) — автоматически |
| `menu` | select | Каталог (раздел), из которого получать уведомления |
| `restrict_entries` | multiselect | Ограничение по записям: `self` — только свои, `members` — где наблюдатель |
| `actions` | multiselect | Фильтр по действиям: `add`, `edit`, `delete` |
| `from_auth` | hidden | ID пользователя |

Доступ: `self` — каждый управляет только своими подписками.
Множественные записи: по одной на каждый каталог, на который подписан пользователь.

---

## Чтение подписок

```js
// Все подписки текущего пользователя
const resp = await App.fetch('/db/systempush_settings.json');
resp.data.forEach(sub => {
    console.log(sub.menu);              // "tt_tasks"
    console.log(sub.restrict_entries);  // "self,members" или ""
    console.log(sub.actions);           // "add,edit" или ""
});
```

---

## Подписка на каталог

```js
// Подписать пользователя на уведомления из каталога задач
// (только свои записи, только добавление и редактирование)
await App.fetch('/db/systempush_settings/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[menu]': 'tt_tasks',
        'form[restrict_entries][]': ['self', 'members'],
        'form[actions][]': ['add', 'edit'],
        submit: 1
    }
});
```

> Alias генерируется автоматически серверной стороной: `md5(menu + userId)`.
> Повторная подписка на тот же каталог обновит существующую запись.

### Подписка без ограничений (все записи, все действия)

```js
await App.fetch('/db/systempush_settings/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[menu]': 'b2b_orders',
        submit: 1
    }
});
```

---

## Отписка

```js
// Найти подписку и удалить
const resp = await App.fetch('/db/systempush_settings.json?form[menu]=tt_tasks');
if (resp.data.length) {
    await App.fetch(`/db/systempush_settings/${resp.data[0].alias}?udel`);
}
```

---

## Получение доступных каталогов для подписки

```js
// Схема содержит список каталогов с activity-поддержкой
const schema = await App.fetch('/db/systempush_settings/sheme.json');
const catalogs = schema.data.menu.arr;
// { "tt_tasks": "Задачи", "b2b_orders": "Заказы", ... }
```

---

## Значения фильтров

### restrict_entries

| Значение | Описание |
|----------|----------|
| *(пусто)* | Все записи каталога, к которым есть доступ |
| `self` | Только записи, где `from_auth` = текущий пользователь |
| `members` | Только записи, где пользователь в `members_id` (наблюдатель) |

Можно комбинировать: `self,members` — свои + где наблюдатель.

### actions

| Значение | Описание |
|----------|----------|
| *(пусто)* | Все действия |
| `add` | Только добавление |
| `edit` | Только редактирование |
| `delete` | Только удаление |

Можно комбинировать: `add,edit` — добавление и редактирование, без удаления.

---

## Сценарии использования в миниапах

### Автоподписка при установке приложения

Приложение, работающее с определённым каталогом, может при установке
автоматически подписать пользователя на уведомления:

```js
async function setupNotifications() {
    // Проверяем, есть ли уже подписка
    const resp = await App.fetch('/db/systempush_settings.json?form[menu]=b2b_orders');
    if (resp.data.length === 0) {
        // Подписываем на новые заказы
        await App.fetch('/db/systempush_settings/add?edit&ajax=1', {
            method: 'POST',
            body: {
                'form[menu]': 'b2b_orders',
                'form[actions][]': ['add'],
                submit: 1
            }
        });
    }
}
```

### Панель управления подписками

Приложение может показать пользователю UI для управления подписками,
сгруппированный по модулям или бизнес-логике приложения.

### Проверка подписки перед действием

```js
// Узнать, подписан ли пользователь на каталог
async function isSubscribed(catalog) {
    const resp = await App.fetch(`/db/systempush_settings.json?form[menu]=${catalog}`);
    return resp.data.length > 0;
}
```

---

## Важно

- Подписки работают в связке с системой activity — уведомления приходят только для каталогов с включённым механизмом activity
- Push-уведомления доставляются в реальном времени через WebSocket (модуль `systempush`)
- Уведомления отображаются в колокольчике в шапке платформы и как браузерные push
- По умолчанию уведомления не приходят — нужна явная подписка

---

*Каталог: `/db/systempush_settings` | Схема: `systempush_settings.sheme.inc.php`*

**← [Home](index.md)**
