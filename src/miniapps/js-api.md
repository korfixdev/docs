# JS API — VMCRMUserApp

> **См. также:** [config-json.md](config-json.md) · [data-api.md](data-api.md) · [storage-and-hooks.md](storage-and-hooks.md) · [styling.md](styling.md)
> **← [Home](index.md)**

Клиентский JS-класс для взаимодействия миниапа с CRM-платформой через `postMessage`.

---

### Подключение

```html
<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();
</script>
```

**Важно**: импорт всегда с абсолютного пути `/templates/def/db/marketplace/vmcrm-user-app.js`.
Это работает потому что `App.fetch()` и другие методы проксируются через `postMessage`
в родительское окно на домене CRM.

### Методы

| Метод | Возвращает | Описание |
|-------|-----------|----------|
| `App.getRequestParams()` | Promise -> `{data: {app_id, domain, catalog, itemId, items, user}}` | Параметры текущего фрейма |
| `App.getUser()` | Promise -> `{data: {name, group, role, avatar, alias, id, tarif, tarif_name}}` | Информация о текущем пользователе, включая тариф |
| `App.getLocation()` | Promise -> `{data: '/db/projects'}` | URL родительского окна |
| `App.fetch(url, options?)` | Promise -> response | HTTP-запрос от имени пользователя (через родительское окно, минуя CORS) |
| `App.fetchAll(url, options?)` | Promise -> response | fetch + автосклейка всех страниц пагинации |
| `App.navigate(url)` | void | SPA-переход родительского окна (без перезагрузки) |
| `App.reload()` | void | Перезагрузка родительского окна |
| `App.modal(content, options?)` | void | Открыть модалку: текст, объект или URL |
| `App.closeModal()` | Promise | Закрыть текущую модалку |
| `App.alert(message, title?)` | void | Всплывающее уведомление (gritter) |
| `App.setFrameSize(width, height)` | void | Размер фрейма. `null` пропускает параметр |
| `App.startLoadingAnimation()` | void | Показать индикатор загрузки платформы |
| `App.stopLoadingAnimation()` | void | Скрыть индикатор загрузки |
| `App.storage.get(key, default?)` | Promise | Чтение из KV-хранилища |
| `App.storage.set(key, value)` | Promise | Запись в KV-хранилище |
| `App.storage.unset(key)` | Promise | Удаление из KV-хранилища |
| `App.on(event, callback)` | this | Подписка на событие платформы. Chainable |
| `App.off(event, callback?)` | this | Отписка. Без callback — удалить все |

### События платформы

Приложение может подписаться на события через `App.on()`:

```js
// Конкретное событие
App.on('page.navigated', (data) => {
    console.log('Navigated:', data.url);
});

// Все события (wildcard)
App.on('*', ({event, data}) => {
    console.log(event, data);
});
```

| Событие | Когда | Данные |
|---------|-------|--------|
| `page.navigated` | SPA-переход между страницами | `{url, title}` |
| `modal.opened` | Открытие модалки редактирования | `{url}` |
| `modal.closed` | Закрытие модалки | `{url}` |
| `catalog.selected` | Выбор чекбоксов в списке каталога | `{catalog, ids}` |

### Auto-resize

Фрейм автоматически репортит высоту контента через `ResizeObserver`.
Хост подхватывает и обновляет размер iframe. `App.setFrameSize()` по-прежнему
работает как ручной override.

### CORS и fetch

**Критически важно**: из iframe нельзя делать `fetch()` или `XMLHttpRequest`
на внешние API напрямую — CORS заблокирует. Всегда используй `App.fetch()`:

```js
// НЕПРАВИЛЬНО — CORS ошибка:
const resp = await fetch('https://api.example.com/data');

// ПРАВИЛЬНО — запрос идёт через родительское окно:
const resp = await App.fetch('/api/db/projects');

// Для данных доступных на CRM:
const resp = await App.fetch('/db/currency_rate.json');
```

### Подробности по методам

#### getRequestParams()

```js
App.getRequestParams().then(resp => {
  const { app_id, domain, catalog, itemId, items, user } = resp.data;
  // items -- алиасы выбранных элементов (через запятую, если список)
});
```

#### getUser()

Информация о текущем пользователе без дополнительных запросов к API.

```js
App.getUser().then(resp => {
  const { name, group, role, avatar, alias, id, tarif, tarif_name } = resp.data;
  // name        — ФИО (author_comment)
  // group       — ID группы (from_group)
  // role        — тип аккаунта (account_type, числовой)
  // avatar      — имя файла аватара (doc)
  // alias       — alias пользователя
  // id          — хеш логина (md5)
  // tarif       — ID тарифа пользователя (строка с числом, напр. "3")
  // tarif_name  — название тарифа (напр. "Стандарт")
});
```

Пример использования — показать имя пользователя в интерфейсе:

```js
const user = await App.getUser();
document.getElementById('userName').textContent = user.data.name;

// Аватар (если есть)
if (user.data.avatar) {
    document.getElementById('userAvatar').src =
        `/reimg/data/auth/${user.data.avatar}?80x80`;
}
```

##### Feature gating по тарифу

Поля `tarif` и `tarif_name` приходят сразу в iframe-параметрах (без дополнительного API-запроса). Используй для условного UI:

```js
const { tarif, tarif_name } = (await App.getUser()).data;

if (tarif === '3' || tarif === '4') {
    // Стандарт/Премиум — показываем расширенный функционал
    showAdvancedFeatures();
} else {
    // Базовый тариф — показываем заглушку с предложением апгрейда
    showUpgradePrompt(tarif_name);
}
```

Для **полной биллинговой информации** (баланс, скидка, дата платежа, прайсы) — используй endpoint `/api/user/tariff` (см. ниже).

##### `/api/user/tariff` — детальная биллинговая инфа

Если нужны не только название тарифа, но и баланс, скидки, даты — отдельный endpoint:

```js
const billing = await App.fetch('/api/user/tariff');
// billing.data:
// {
//   tarif: "3",
//   tarif_name: "Стандарт",
//   balance: "1500.00",
//   discount: "10",
//   discount_date: "2026-06-01",
//   payment_date: "2026-05-01",
//   price: "990.00",
//   discount_3months: "2700.00",
//   discount_12months: "9900.00"
// }
```

Доступен **только по сессии** (через `App.fetch` из миниапа). Для Bearer-токена endpoint вернёт `401 Unauthorized` — это сделано намеренно, биллинг привязан к авторизованной сессии пользователя.

#### fetch(url, options?)

```js
// GET
App.fetch('/db/projects.json').then(resp => {
  console.log(resp.data);
});

// POST (изменение элемента)
App.fetch(`/db/projects/${alias}?edit&ajax=1`, {
  method: 'POST',
  body: {
    'form[name]': 'Новое имя',
    'form[id]': id,
    'form[alias]': alias,
    submit: 1
  }
});
```

URL может быть только относительным (без домена).
Тело запроса преобразуется в `URLSearchParams`.

#### fetchAll(url, options?)

Как `fetch()` но автоматически загружает все страницы пагинации и объединяет `data`.

```js
App.fetchAll('/db/projects.json').then(resp => {
  // resp.data -- все элементы, без пагинации
});
```

#### modal(content, options?)

```js
// Текстовое сообщение
App.modal('Текст сообщения');

// С заголовком
App.modal({ title: 'Заголовок', content: 'Текст' });

// Содержимое по URL (только относительный)
App.modal('/db/todo', { title: 'Заголовок' });
```

#### navigate(url)

SPA-навигация родительского окна без полной перезагрузки страницы.
Только относительные URL (начинающиеся с `/`).

```js
App.navigate('/db/projects');           // перейти к каталогу
App.navigate('/db/orders/ORDER123');    // к конкретному элементу
```

#### on(event, callback) / off(event, callback?)

Подписка/отписка на события платформы. Chainable.

```js
App.on('page.navigated', (data) => console.log(data.url));
App.on('modal.closed', (data) => refreshData());
App.on('catalog.selected', (data) => console.log(data.catalog, data.ids));
App.on('*', ({event, data}) => console.log(event, data)); // wildcard

App.off('page.navigated'); // удалить все обработчики
```

---

**Дальше:** [data-api.md](data-api.md) · **← [Home](index.md)**
