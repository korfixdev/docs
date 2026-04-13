# Инструкции и справка (account_help, service_help)

> **См. также:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [favorites-menu.md](favorites-menu.md)
> **← [Home](index.md)**

Два каталога для контекстной помощи: инструкции к каталогам
и справочная информация на страницах.

---

## account_help — Инструкции к каталогам

Инструкции привязаны к конкретным каталогам платформы. Отображаются
как кнопка-стикер в заголовке каталога — при клике открывается модалка
с контентом инструкции.

### Структура записи

| Поле | Тип | Описание |
|------|-----|----------|
| `name` | textbox | Заголовок инструкции |
| `aliasdb` | select | Каталог, к которому привязана инструкция (например `tt_tasks`) |
| `is_main` | checkbox | Инструкция по умолчанию (если несколько на каталог — показывается с `is_main=1`) |
| `cont` | textarea (wysiwyg) | Текст инструкции (HTML) |
| `doc` | photo | Иллюстрация |
| `doc1` | file | Прикреплённый файл |
| `youtube` | textbox | Ссылка на YouTube-видео |

Доступ: по группе (`from_group`). Можно создавать несколько инструкций на каталог.

### Как работает в UI

При открытии любого каталога в заголовке появляется иконка:
- **Заполненный стикер** (fas fa-sticky-note) — инструкция есть, клик открывает её
- **Пустой стикер** (far fa-sticky-note) — инструкции нет, клик создаёт новую

### Чтение инструкций

```js
// Получить все инструкции
const resp = await App.fetch('/db/account_help.json');

// Инструкции для конкретного каталога
const resp = await App.fetch('/db/account_help.json?form[aliasdb]=tt_tasks');
const instruction = resp.data[0];
console.log(instruction.name);     // "Как работать с задачами"
console.log(instruction.cont);     // HTML-контент
console.log(instruction.youtube);  // "abcd" или полная ссылка
```

### Создание инструкции из миниапа

```js
// Создать инструкцию для каталога заказов
await App.fetch('/db/account_help/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Как оформить заказ',
        'form[aliasdb]': 'b2b_orders',
        'form[is_main]': 1,
        'form[cont]': '<h3>Шаг 1</h3><p>Откройте каталог заказов...</p>',
        'form[youtube]': 'https://youtu.be/abcd',
        submit: 1
    }
});
```

### Получение списка каталогов

```js
// Доступные каталоги для привязки инструкции
const schema = await App.fetch('/db/account_help/sheme.json');
const catalogs = schema.data.aliasdb.arr;
// { "tt_tasks": "Задачи (tt_tasks)", "b2b_orders": "Заказы (b2b_orders)", ... }
```

---

## service_help — Справочная информация по страницам

Справки привязаны к URL-адресам страниц. Отображаются как кнопка "?"
в заголовке страницы.

### Структура записи

| Поле | Тип | Описание |
|------|-----|----------|
| `name` | textbox | Заголовок справки |
| `show_on_pages` | textarea | URL-адреса страниц, на которых показывать (по одному на строку) |
| `cont` | textarea (tinymce) | Текст справки (HTML с WYSIWYG-редактором) |
| `doc` | photo | Иллюстрация |
| `doc1` | file | Прикреплённый файл |
| `youtube` | textbox | Ссылка на YouTube-видео |

### Как работает в UI

Справки загружаются вызовом `cmd('db/service_help')` в подвале каждого каталога.
Если для текущей страницы есть справка — в заголовке появляется кнопка "?" (fa-question-circle),
клик открывает модалку с контентом, картинкой и видео.

### Чтение справок

```js
// Все справки
const resp = await App.fetch('/db/service_help.json');

// Справки содержащие определённый URL
const resp = await App.fetch('/db/service_help.json?form[show_on_pages]=/db/tt_tasks');
```

### Создание справки

```js
await App.fetch('/db/service_help/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Как пользоваться фильтрами',
        'form[show_on_pages]': '/db/tt_tasks\n/db/b2b_orders',
        'form[cont]': '<p>Используйте панель фильтров...</p>',
        submit: 1
    }
});
```

---

## Сценарии использования в миниапах

### Автоматическое создание инструкции при установке

Приложение может создать инструкцию к каталогу, с которым работает:

```js
async function createAppInstruction(catalog, title, content, videoUrl) {
    // Проверяем, нет ли уже инструкции от нашего приложения
    const resp = await App.fetch(`/db/account_help.json?form[aliasdb]=${catalog}`);
    if (resp.data.length > 0) return; // уже есть

    await App.fetch('/db/account_help/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[name]': title,
            'form[aliasdb]': catalog,
            'form[cont]': content,
            'form[youtube]': videoUrl || '',
            submit: 1
        }
    });
}
```

### Контекстная помощь внутри миниапа

Миниап может читать существующие инструкции и показывать их
в собственном UI, например как тултипы или встроенные подсказки:

```js
async function getHelpForCatalog(catalog) {
    const resp = await App.fetch(`/db/account_help.json?form[aliasdb]=${catalog}`);
    return resp.data; // массив инструкций
}
```

### База знаний

Приложение может построить навигируемую базу знаний из всех инструкций:

```js
const all = await App.fetchAll('/db/account_help.json');
// Группировка по каталогам
const byModule = {};
all.data.forEach(item => {
    if (!byModule[item.aliasdb]) byModule[item.aliasdb] = [];
    byModule[item.aliasdb].push(item);
});
```

---

## Разница между каталогами

| | account_help | service_help |
|---|---|---|
| Привязка | К каталогу (`aliasdb`) | К URL страницы (`show_on_pages`) |
| Иконка | Стикер (fa-sticky-note) | Вопрос (fa-question-circle) |
| Множественность | Несколько на каталог (приоритет `is_main`) | Несколько на страницу |
| Редактор | wysiwyg-simple | tinymce |
| Доступ | По группе | По группе |
| Назначение | Инструкции для пользователей | Справочная информация от администратора |

---

*Каталоги: `/db/account_help`, `/db/service_help`*

**← [Home](index.md)**
