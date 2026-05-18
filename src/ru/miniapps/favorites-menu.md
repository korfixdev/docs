# Избранное меню (favorites_menu)

> **См. также:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [systempush-settings.md](systempush-settings.md)
> **← [Home](index.md)**

Каталог `favorites_menu` — персональные настройки навигации пользователя:
избранные пункты бокового меню, стартовая страница, режим отображения.

---

## Структура записи

Одна запись на пользователя (alias = ID пользователя).

| Поле | Тип | Описание |
|------|-----|----------|
| `alias` | hidden | ID пользователя (= `from_auth`) |
| `name` | hidden | Имя пользователя |
| `from_auth` | hidden | Владелец записи |
| `from_group` | hidden | Группа владельца |
| `first_page` | select_from_table | Стартовая страница (из дерева меню `rutree`) |
| `menu` | multiselect_from_table | ID пунктов меню, добавленных в избранное |
| `hide_menu` | checkbox | Свернуть остальные пункты меню (показывать только избранное) |

Доступ: `self` — каждый пользователь видит и редактирует только свою запись.

---

## Чтение избранного

```js
// Получить настройки текущего пользователя
const resp = await App.fetch('/db/favorites_menu.json');
const favorites = resp.data[0]; // одна запись на пользователя

console.log(favorites.menu);       // "12,45,78" — ID пунктов меню через запятую
console.log(favorites.first_page); // ID стартовой страницы
console.log(favorites.hide_menu);  // "1" или "0"
```

---

## Добавление пункта в избранное

Чтобы добавить пункт в избранное пользователя, нужно:

1. Прочитать текущий список `menu`
2. Добавить нужный ID
3. Сохранить обратно

```js
async function addToFavorites(menuItemId) {
    // 1. Читаем текущие настройки
    const resp = await App.fetch('/db/favorites_menu.json');
    const record = resp.data[0];

    if (!record) {
        console.error('Нет записи favorites_menu для пользователя');
        return;
    }

    // 2. Проверяем, не добавлен ли уже
    const currentIds = (record.menu || '').split(',').filter(Boolean);
    if (currentIds.includes(String(menuItemId))) {
        return; // уже в избранном
    }

    // 3. Добавляем и сохраняем
    currentIds.push(String(menuItemId));

    await App.fetch(`/db/favorites_menu/${record.alias}?edit&ajax=1`, {
        method: 'POST',
        body: {
            'form[menu]': currentIds,  // массив ID
            submit: 1
        }
    });
}
```

### Удаление из избранного

```js
async function removeFromFavorites(menuItemId) {
    const resp = await App.fetch('/db/favorites_menu.json');
    const record = resp.data[0];
    if (!record) return;

    const currentIds = (record.menu || '').split(',').filter(Boolean);
    const newIds = currentIds.filter(id => id !== String(menuItemId));

    await App.fetch(`/db/favorites_menu/${record.alias}?edit&ajax=1`, {
        method: 'POST',
        body: {
            'form[menu]': newIds,
            submit: 1
        }
    });
}
```

---

## Получение дерева меню

Чтобы узнать какие пункты меню доступны для добавления в избранное:

```js
// Загрузить схему — в ней arr содержит полное дерево меню
const schema = await App.fetch('/db/favorites_menu/sheme.json');
const menuTree = schema.data.menu.arr;
// { "12": "Финансы > Операции", "45": "Задачи > Все задачи", ... }
```

---

## Установка стартовой страницы

```js
async function setStartPage(pageId) {
    const resp = await App.fetch('/db/favorites_menu.json');
    const record = resp.data[0];
    if (!record) return;

    await App.fetch(`/db/favorites_menu/${record.alias}?edit&ajax=1`, {
        method: 'POST',
        body: {
            'form[first_page]': pageId,
            submit: 1
        }
    });
}
```

---

## Сценарии использования в миниапах

### Автодобавление в избранное при установке

При установке приложения можно автоматически добавить нужные пункты меню
в избранное, чтобы пользователь сразу видел их в навигации.

### Умное избранное на основе активности

Приложение может анализировать `activity` и автоматически предлагать
добавить в избранное часто посещаемые разделы:

```js
// Получить последние визиты пользователя
const activity = await App.fetchAll('/db/activity.json?form[action]=просмотрел&limit=100');

// Подсчитать частоту посещения каталогов
const counts = {};
activity.data.forEach(item => {
    counts[item.catalog] = (counts[item.catalog] || 0) + 1;
});

// Топ-5 самых посещаемых
const top5 = Object.entries(counts)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 5);
```

### Персонализация навигации по роли

Приложение может настраивать избранное в зависимости от роли пользователя,
отдела или активных модулей компании.

---

## Важно

- Запись создаётся автоматически при первом входе пользователя (дефолт зависит от `type_user`)
- Поле `menu` — строка с ID через запятую (multiselect_from_table)
- При записи через API можно передать `form[menu]` как массив — платформа сконвертирует
- Изменения в избранном видны после перезагрузки страницы (боковое меню рендерится серверно)
- Drag-and-drop пунктов меню в избранное работает в UI — приложение получит уже обновлённые данные при чтении

---

*Каталог: `/db/favorites_menu` | Схема: `favorites_menu.sheme.inc.php`*

**← [Home](index.md)**
