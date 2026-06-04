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
| `App.getRequestParams()` | Promise -> `{data: {app_id, domain, catalog, itemId, items, user}}` | Параметры текущего фрейма. Результат кешируется — повторные вызовы мгновенные |
| `App.getUser()` | Promise -> `{data: {name, from_auth, from_group, alias, role, avatar, tarif, tarif_name}}` | Информация о текущем пользователе, включая тариф. Результат кешируется |
| `App.getLocation()` | Promise -> `{data: '/db/projects'}` | URL родительского окна |
| `App.fetch(url, options?)` | Promise -> response | HTTP-запрос от имени пользователя. Дедупликация параллельных запросов с одинаковым URL |
| `App.fetchAll(url, options?)` | Promise -> response | fetch + автосклейка всех страниц пагинации |
| `App.prefetch(url)` | void | Запустить фетч в фоне заранее. Когда `App.fetch(url)` вызовется позже — вернёт готовые данные мгновенно |
| `App.done()` | Promise | Сигнал "установка завершена" из install-фрейма. Setup-экран платформы немедленно переходит к следующему приложению |
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

#### Pattern: reload list after record edit

`modal.closed` fires when the user closes a modal opened via `App.modal()`. `data.url`
contains the same URL that was passed to `App.modal()`. Filter by it so you don't react
to unrelated modals. A 50 ms debounce prevents double-firing.

```js
// Open the edit modal
App.modal('/db/tt_projects/' + alias + '?edit', { title: 'Edit' });

// React to close — data.url matches the URL from App.modal()
let _reloadTimer = 0;
App.on('modal.closed', (data) => {
    if (data?.url?.includes('/tt_projects/')) {
        clearTimeout(_reloadTimer);
        _reloadTimer = setTimeout(() => loadRecords(), 50);
    }
});
```

#### Pattern: background polling (detect external changes)

Track a snapshot: total count + top-5 record IDs sorted by `ts_desc`. Any edit moves
a record to the top — the signature changes and the list reloads.

```js
let _pollSnap = { total: -1, topIds: '' };

// After first load: seed the baseline
_pollSnap = { total: records.length, topIds: records.slice(0,5).map(r=>r.id).join(',') };

// Polling
setInterval(async () => {
    try {
        const r = await App.fetch('/db/MY_CATALOG.json?limit=5&order=ts_desc&not_cache=1');
        const rows = Array.isArray(r?.data) ? r.data : (r?.data?.data ?? []);
        const total = Number(r?.total ?? rows.length);
        const topIds = rows.map(r => r.id).join(',');
        if (_pollSnap.total >= 0 && (total !== _pollSnap.total || topIds !== _pollSnap.topIds)) {
            loadRecords();
        }
        _pollSnap = { total, topIds };
    } catch (_) {}
}, 60000);
```

Both patterns complement each other: `modal.closed` reacts instantly to the user's own
actions; polling catches changes made by another user or outside the miniapp.

### Auto-resize

Фрейм автоматически репортит высоту контента через `ResizeObserver`.
Хост подхватывает и обновляет размер iframe. `App.setFrameSize()` по-прежнему
работает как ручной override.

### Абсолютные пути для ресурсов платформы

Миниап живёт в изолированном iframe. **Относительные пути** (`./../img/photo.jpg`) резолвятся
относительно store-URL самого приложения, а не CRM-домена — это изолированное хранилище zip-архива.

Для ресурсов платформы (аватары, файлы каталогов) используй **абсолютные пути**:

```js
// Аватар пользователя — абсолютный путь через /reimg/
const avatarUrl = `/reimg/data/auth/${user.data.avatar}?80x80`;

// Файл из каталога marketplace (иконка приложения) — поле doc
const iconUrl = `/data/db/f_marketplace/${item.doc}`;

// Вложения из любого каталога — /data/db/f_{catalog}/
const fileUrl = `/data/db/f_tt_tasks/${attachment}`;
```

Ключевые пути:
| Тип ресурса | Путь |
|-------------|------|
| Аватар пользователя | `/reimg/data/auth/{doc}?{size}` |
| Файл каталога `{cat}` | `/data/db/f_{cat}/{doc}` |
| Иконка приложения (marketplace.doc) | `/data/db/f_marketplace/{doc}` |

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

> **Ловушка: не передавай `undefined`/`null` вторым аргументом.** Обёртка вида
> `App.fetch(url, opts)`, где `opts === undefined` (типично для GET), ломается:
> `undefined` сериализуется в `null` через postMessage, на хосте срабатывает
> `typeof null === 'object'` → чтение `null.body` → исключение, и запрос **висит до
> 60-сек таймаута** (молча: данные/профиль не грузятся, явной ошибки нет). В обёртке
> ветви по наличию опций:
> ```js
> async function apiFetch(url, opts) {
>     const r = opts ? await App.fetch(url, opts) : await App.fetch(url);
>     return r?.data ?? r;
> }
> ```

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
  const { name, from_auth, from_group, alias, role, avatar, tarif, tarif_name } = resp.data;
  // name        — ФИО (author_comment)
  // from_auth   — ID пользователя → передавать в form[from_auth] при создании записей
  // from_group  — ID тенанта → передавать в form[from_group] при создании записей
  // alias       — md5(login), идентификатор пользователя в системе приложений
  // role        — тип аккаунта (account_type, числовой: 0=admin, 1=manager, ...)
  // avatar      — имя файла аватара (doc) → /reimg/data/auth/{avatar}?80x80
  // tarif       — ID тарифа (строка с числом, напр. "7")
  // tarif_name  — название тарифа (напр. "Премиум")
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

#### prefetch(url)

Запускает фетч в фоне и сохраняет результат. Когда позже вызывается `App.fetch()` с тем же URL, данные возвращаются мгновенно — без `postMessage`.

Вызывай в начале `init`, параллельно с другими инициализирующими операциями:

```js
async function init() {
    // Запускаем в фоне сразу
    App.prefetch('/db/marketplace.json?limit=200&free_cache=1');
    App.prefetch('/db/installed_apps.json?limit=200&free_cache=1');

    // ... другой код инициализации ...

    // К этому моменту данные уже готовы — возвращается мгновенно
    const market = await App.fetch('/db/marketplace.json?limit=200&free_cache=1');
    const installed = await App.fetch('/db/installed_apps.json?limit=200&free_cache=1');
}
```

> URL в `prefetch` и `fetch` должен совпадать точно (включая query-параметры).

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

**Дедупликация:** если два параллельных вызова `App.fetch(url)` с одинаковым URL запущены одновременно — оба получат один и тот же Promise (один реальный запрос). Полезно при `Promise.all`. Чтобы отключить: добавь `not_cache=1` в URL.

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

#### done()

Сигнал из install-фрейма платформе: "я завершил установку". Используется в `install.html` после self-provisioning.

**Контекст**: при первом входе нового пользователя платформа показывает экран инициализации, который по очереди открывает каждый install-фрейм в скрытом iframe. Когда install-фрейм вызывает `App.done()` — платформа немедленно переходит к следующему приложению, не дожидаясь таймаута (4 секунды по умолчанию).

```js
// install.html — паттерн: авто-установка + сигнал готовности

import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();

async function init() {
    const exists = await checkCatalogExists('custom_myapp');

    if (!exists) {
        showProgress('Создаём структуру данных...');
        await runInstall();     // createCatalog() + configureAccess() + ...
        showProgress('Готово');
    }

    // Вызываем в обоих случаях: установили только что, или уже было установлено.
    // Setup-экран перейдёт к следующему приложению немедленно.
    App.done();
}

init();
```

> **Fallback**: если `App.done()` не вызван (например, приложение ещё не обновлено) — setup-экран ждёт 4 секунды после загрузки iframe, затем всё равно переходит дальше. Незавершённые API-запросы продолжают работать в фоне.

> **За пределами setup-экрана**: вызов `App.done()` безопасен и ничего не делает — хост-обработчик по умолчанию no-op.

#### navigate(url)

SPA-навигация родительского окна без полной перезагрузки страницы.
Только относительные URL (начинающиеся с `/`).

```js
App.navigate('/db/projects');           // перейти к каталогу
App.navigate('/db/orders/ORDER123');    // к конкретному элементу
```

##### Паттерны для навигации между приложениями

```js
// Открыть установленное приложение по его alias из installed_apps
// frame=main — открыть главный фрейм; catalog=marketplace — контекст маркетплейса
App.navigate('/db/installed_apps/MY_APP_ALIAS?frame=main&catalog=marketplace');

// Открыть карточку приложения в маркетплейсе
App.navigate('/db/marketplace/MY_APP_ALIAS');
```

Эти паттерны используются когда одно приложение хочет открыть другое или направить пользователя в маркетплейс. `MY_APP_ALIAS` — alias записи в соответствующем каталоге.

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
