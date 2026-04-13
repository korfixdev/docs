# Стилизация миниапов под платформу

> **См. также:** [js-api.md](js-api.md) · [dashboards.md](dashboards.md) · [getting-started.md](getting-started.md) · [checklist.md](checklist.md)
> **← [INDEX](INDEX.md)**

Оформление, CSS-переменные, структура HTML, подключение JS и Chart.js.

---

## Принципы

1. Приложение рендерится в **iframe** — стили платформы не наследуются
2. Чтобы выглядеть нативно, используй **CSS-переменные платформы** (см. ниже)
3. **Не дублируй** jQuery, Bootstrap и другие библиотеки — бери с портала или CDN
4. Шрифт — `"Open Sans"`, с системным fallback
5. Структурные обёртки (`body.widget`, `main.content`, `article.content__main`) — по желанию, для полноэкранных приложений

---

## CSS-переменные платформы (дизайн-токены)

Полный набор переменных из дизайн-системы korfix. Копируй в `:root` своего приложения — используй нужные.

### Базовые цвета

```css
:root {
    /* Основные */
    --primary: #323C8F;
    --secondary: #343859;
    --success: #33BE2B;
    --info: #17a2b8;
    --warning: #FF4D50;
    --danger: #EF233C;
    --light: #fff;
    --dark: #1D1E32;

    /* Акцентные */
    --blue: #475CFF;
    --purple: #323C8F;
    --orange: #FF4D50;
    --red: #EF233C;
}
```

### Оттенки серого

```css
:root {
    --white: #fff;
    --gray1: #4e4f56;
    --gray2: #78797f;        /* текст заголовков таблиц */
    --gray3: #a3a3a7;
    --gray4: #b8b8bb;
    --gray5: #dddde1;
    --gray6: #eaeaee;        /* границы, разделители */
    --gray7: #f6f6f6;
    --gray8: #f7f7f8;        /* чередование строк таблиц */
    --gray9: #f9f9f9;
}
```

### Оттенки синего/серо-голубого

```css
:root {
    --bluegray3: #57596e;
    --bluegray4-6d6f89: #6d6f89;
    --bluegray5: #8a8ca1;    /* вторичный текст */
    --bluegray6: #b9bdcd;
    --bluegray7: #c5cadc;
    --bluegray8: #dce0ef;    /* границы элементов */
    --bluegray9: #eceffa;
    --bluegray10: #f4f5fa;   /* фон карточек */
    --bluegray11: #f8f8fd;
    --bluegray12: #F5F5F8;
    --bluegray13: #FAFAFC;

    --blue3-323c8f: #323c8f;  /* ссылки */
    --blue4: #5a63b4;
    --blue9: #eff1fc;         /* фон бейджей */
}
```

### Статусные цвета

```css
:root {
    --red-warning1: #b52929;
    --red-warning2: #fbe0e0;
    --yellow-warning1: #e5ab0e;
    --yellow-warning2: #fefbe2;
    --green-positive1: #388651;
    --green-positive2: #dcf5e2;
}
```

### Типографика

```css
:root {
    --font-family-sans-serif: "Open Sans", sans-serif, -apple-system,
        BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial,
        "Noto Sans", "Liberation Sans", sans-serif;
    --font-family-monospace: SFMono-Regular, Menlo, Monaco, Consolas,
        "Liberation Mono", "Courier New", monospace;
}
```

### Брейкпоинты

```css
:root {
    --breakpoint-sm: 576px;
    --breakpoint-md: 768px;
    --breakpoint-lg: 992px;
    --breakpoint-xl: 1200px;
}
```

---

## Шрифт

Платформа использует **Open Sans**. Подключай через CDN, если не уверен, что шрифт загружен в родителе:

```html
<link href="https://fonts.googleapis.com/css2?family=Open+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">
```

```css
body {
    font: 400 14px/1.5 var(--font-family-sans-serif);
    color: #212529;
    margin: 0;
}
```

---

## Структура HTML-страницы

### Минимальный шаблон (виджет/попап)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Моё приложение</title>
    <style>
        :root { /* нужные переменные */ }
        *, ::after, ::before { box-sizing: border-box; }
        body {
            font: 400 14px/1.5 "Open Sans", sans-serif, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background: transparent;
            margin: 0;
            color: #212529;
        }
    </style>
</head>
<body>
    <div id="app">Loading...</div>
    <script type="module">
        import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
        const App = new VMCRMUserApp();
        // ...
    </script>
</body>
</html>
```

### Полноэкранный шаблон (пункт меню)

Для полноэкранных приложений используй обёртки платформы:

```html
<body class="widget">
<main class="content m-0">
    <article class="content__main">
        <div class="content__common pt-4">
            <!-- ваш контент -->
        </div>
    </article>
</main>
</body>
```

Вспомогательные CSS для структурных классов:

```css
body.widget {
    overflow-x: hidden !important;
    background-color: transparent;
}
article.content__main {
    min-height: auto;
    padding: 0;
}
```

---

## Компоненты UI

### Кнопки

```css
.btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 16px;
    border: none;
    border-radius: 3px;       /* как на платформе */
    cursor: pointer;
    font: 500 13px/1.4 "Open Sans", sans-serif;
    transition: .15s;
}

/* Основная (синяя) */
.btn-primary {
    background: var(--primary);
    color: #fff;
    border-bottom: 3px solid #2A6AA4;
}
.btn-primary:hover { opacity: 0.9; }

/* Успешная (зелёная) */
.btn-success {
    background: var(--success);
    color: #fff;
    border-bottom: 3px solid #2B9C5B;
}

/* Опасная (красная) */
.btn-danger {
    background: var(--danger);
    color: #fff;
    border-bottom: 3px solid #B7433F;
}

/* Вторичная (контурная) */
.btn-outline {
    background: var(--white);
    color: var(--primary);
    border: 1px solid var(--bluegray8);
}
.btn-outline:hover { border-color: var(--primary); }

.btn:disabled { opacity: .5; cursor: not-allowed; }
```

### Карточки

```css
.card {
    background: var(--white);
    border-radius: 8px;
    box-shadow: 0 1px 3px rgba(0,0,0,.08);
    padding: 16px;
    margin-bottom: 12px;
}
.card h3 {
    font: 600 14px/1.4 "Open Sans", sans-serif;
    margin-bottom: 12px;
}
```

### Таблицы

```css
table {
    border-collapse: separate;
    border-spacing: 0;
    border: 1px solid var(--gray6);
    width: 100%;
}
thead th {
    font: 500 13px/15px "Open Sans", sans-serif;
    background: var(--white);
    color: var(--gray2);
    padding: 16px 8px;
    border-bottom: 1px solid var(--gray6);
    text-align: left;
}
tbody tr:nth-child(odd) { background: var(--white); }
tbody tr:nth-child(even) { background: var(--gray8); }
tbody td {
    padding: 8px;
    border-bottom: 1px solid var(--gray6);
}
```

### Формы (select, input, textarea)

```css
select, input[type="text"], textarea {
    width: 100%;
    padding: 8px 12px;
    border: 1px solid var(--gray6);
    border-radius: 3px;
    font: 400 13px/1.4 "Open Sans", sans-serif;
    background: var(--white);
    color: #212529;
    transition: border-color .15s;
}
select:focus, input:focus, textarea:focus {
    outline: none;
    border-color: var(--primary);
}
label {
    display: block;
    font: 400 12px/1.4 "Open Sans", sans-serif;
    color: var(--bluegray5);
    margin-bottom: 4px;
}
```

### Статусы/уведомления

```css
.status {
    padding: 10px 14px;
    border-radius: 6px;
    font-size: 13px;
    margin-top: 12px;
}
.status.ok {
    background: var(--green-positive2);
    color: var(--green-positive1);
}
.status.err {
    background: var(--red-warning2);
    color: var(--red-warning1);
}
.status.warn {
    background: var(--yellow-warning2);
    color: var(--yellow-warning1);
}
```

### Бейджи

```css
.badge {
    display: inline-block;
    padding: 3px 10px;
    border-radius: 12px;
    font-size: 12px;
    font-weight: 600;
    background: var(--blue9);
    color: var(--primary);
}
```

### Табы

> **Важно:** используйте `<a href="javascript:void(0)">` вместо `<div>` для табов.
> На мобильных (особенно iOS Safari) `click` на `<div>` может не срабатывать.

```css
.tabs {
    display: flex;
    border-bottom: 2px solid var(--gray6);
    margin-bottom: 20px;
}
.tab {
    padding: 10px 20px;
    cursor: pointer;
    font-weight: 500;
    color: var(--bluegray5);
    border-bottom: 2px solid transparent;
    margin-bottom: -2px;
    text-decoration: none;
}
.tab:hover { color: var(--primary); }
.tab.active {
    color: var(--primary);
    border-bottom-color: var(--primary);
}
```

### Ссылки

```css
a {
    color: var(--blue3-323c8f);
    text-decoration: underline;
    text-decoration-style: dotted;
    cursor: pointer;
    transition: .2s;
}
a:hover { text-decoration-style: solid; }
```

---

## JS: что брать с портала, что с CDN

### С портала (через абсолютный путь в iframe)

```js
// VMCRMUserApp — обязательный, основной API
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
```

Это единственный JS-модуль, который нужно импортировать с портала. Он даёт:
`fetch`, `fetchAll`, `modal`, `alert`, `navigate`, `setFrameSize`, `storage`, `on`.

### С CDN (когда нужно)

```html
<!-- Chart.js (графики) -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js"></script>

<!-- Mermaid (диаграммы) -->
<script type="module">
import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
</script>
```

### Чего НЕ нужно подключать

- **jQuery** — не используется в миниапах, пиши на Vanilla JS (ES6+)
- **Bootstrap CSS/JS** — не подключай целиком; если нужны утилити-классы — опиши их локально
- **Шрифт Open Sans** — подключай через Google Fonts CDN, если нужен

---

## Chart.js — палитра платформы

```js
const colors = [
    '#5F67A8', '#45476A', '#E6576F', '#5388AF', '#4E8F98',
    '#CBDDA6', '#2C55BF', '#D6B075', '#A25E8B', '#D48474'
];
const backgrounds = [
    '#EFF0F7', '#EDEDF1', '#FDEFF1', '#EEF4F7', '#EEF4F5',
    '#FAFCF7', '#EAEFF9', '#FBF8F2', '#F6EFF4', '#FBF3F2'
];
```

---

## Адаптивность и мобильные

### Обязательные правила

1. **Таблицы → карточки на мобильных.** Широкие таблицы не помещаются на экран телефона. Используйте media query для переключения на плиточную раскладку:

```css
@media (max-width: 768px) {
    table, thead, tbody, tr, td, th { display: block; }
    thead { display: none; }
    tr {
        background: var(--white);
        border: 1px solid var(--gray6);
        border-radius: 6px;
        padding: 10px;
        margin-bottom: 8px;
    }
    td {
        display: flex;
        justify-content: space-between;
        padding: 4px 0;
        border: none;
    }
    td::before {
        content: attr(data-label);
        font-weight: 600;
        color: var(--bluegray5);
        font-size: 11px;
    }
}
```

HTML-разметка для `data-label`:
```html
<td data-label="Статус"><span class="badge-type">active</span></td>
<td data-label="Сумма">15 000 ₽</td>
```

2. **Размер шрифта в полях ввода — не менее 16px.** iOS Safari автоматически зумит страницу при фокусе на input с `font-size < 16px`:

```css
/* ПРАВИЛЬНО — без зума на iOS */
input, select, textarea {
    font-size: 16px;  /* минимум! */
}

/* НЕПРАВИЛЬНО — вызовет зум */
input { font-size: 13px; }
```

3. **Кликабельные элементы — `<a>` или `<button>`, не `<div>`.** iOS Safari может не обрабатывать `click` на `<div>`:

```html
<!-- ПРАВИЛЬНО -->
<a class="tab" href="javascript:void(0)" data-tab="export">Экспорт</a>
<button class="btn btn-primary" id="btnExport">Экспорт</button>

<!-- НЕПРАВИЛЬНО — может не работать на iOS -->
<div class="tab" data-tab="export">Экспорт</div>
```

### Прочее

```css
@media (max-width: 768px) {
    .tabs { flex-direction: column; }
    .card { padding: 12px; }
    .actions { flex-direction: column; }
}
```

```js
// В JS — для выбора типа контрола
const isMobile = window.innerWidth < 768;
```

---

## Иконка настроек (шестерёнка)

В интерфейсе каждого приложения должна быть иконка ⚙ для доступа к:
- Настройкам приложения (если есть)
- Экрану установки / self-provisioning (с логом и прогрессом)
- Информации о версии

```html
<div class="header">
    <h2>Моё приложение</h2>
    <a href="javascript:void(0)" id="btnSettings" style="margin-left:auto;color:var(--bluegray5);">
        <i class="fa fa-cog"></i>
    </a>
</div>
```

Результат установки (флаг `installed: true`) сохраняется в `App.storage`, чтобы экран установки не показывался повторно.

---

## Авторесайз фрейма (ОБЯЗАТЕЛЬНО)

> **Это самая частая ошибка в миниапах.** Аудит показал, что 70% приложений
> не вызывают `setFrameSize` — контент обрезается, пользователь не видит часть UI.
> **Каждое** изменение DOM требует вызова `setFrameSize`.

Iframe не подстраивается автоматически — нужно вызывать `setFrameSize` после **каждого** изменения контента: рендер списка, переключение табов, раскрытие аккордеона, показ/скрытие блоков, загрузка данных.

### Обязательная настройка

```css
/* ОБЯЗАТЕЛЬНО на body — убирает скроллбар внутри iframe */
body { overflow: hidden; }
```

```js
// Хелпер — ОБЯЗАТЕЛЬНО создать и вызывать после ЛЮБОГО изменения DOM
function resizeFrame() {
    requestAnimationFrame(() => App.setFrameSize(null, document.body.scrollHeight));
}
```

### Где вызывать

```js
// После рендера списка
list.innerHTML = items.map(renderItem).join('');
resizeFrame();

// После загрузки данных
App.fetch('/db/catalog.json').then(resp => {
    renderTable(resp.data);
    resizeFrame();
});

// После переключения табов
tab.addEventListener('click', () => {
    showPanel(tab.dataset.panel);
    resizeFrame();
});

// После раскрытия/сворачивания блока
toggle.addEventListener('click', () => {
    body.classList.toggle('open');
    resizeFrame();
});

// При инициализации
App.getRequestParams().then(({data}) => {
    renderWidget(data);
    resizeFrame();
});
```

### Антипаттерны

```js
// НЕПРАВИЛЬНО — +20 маскирует проблему, даёт неточную высоту
App.setFrameSize(null, document.body.scrollHeight + 20);

// НЕПРАВИЛЬНО — вызов без requestAnimationFrame, DOM не успел обновиться
container.innerHTML = html;
App.setFrameSize(null, document.body.scrollHeight); // замерит старую высоту

// НЕПРАВИЛЬНО — не вызывать setFrameSize совсем
container.innerHTML = renderBigTable(data); // контент обрежется

// ПРАВИЛЬНО
container.innerHTML = html;
requestAnimationFrame(() => App.setFrameSize(null, document.body.scrollHeight));
```

---

## Обязательные правила (частые ошибки)

Аудит 37 приложений выявил типовые ошибки. Следуйте этим правилам:

### 1. Всегда используйте `App.fetch()`, не нативный `fetch()`

Миниап работает в iframe — нативный `fetch()` к endpoints платформы заблокирован CORS.
`App.fetch()` проксирует запрос через `postMessage` в родительское окно.

```js
// НЕПРАВИЛЬНО — CORS ошибка
const resp = await fetch('/db/projects.json');

// ПРАВИЛЬНО
const resp = await App.fetch('/db/projects.json');
```

### 2. Используйте `App.storage`, не `localStorage`

`localStorage` в iframe изолирован — данные потеряются при смене домена приложения.

```js
// НЕПРАВИЛЬНО
localStorage.setItem('settings', JSON.stringify(data));

// ПРАВИЛЬНО
await App.storage.set('settings', JSON.stringify(data));
```

### 3. Объявляйте `permissions` в config.json

Каждый каталог и операция, используемые в коде, должны быть перечислены в `permissions`.
Без этого приложение может быть заблокировано песочницей.

```json
{
    "permissions": {
        "catalogs": {
            "tt_tasks": ["read", "write"],
            "eventlogs": ["read"]
        },
        "storage": true,
        "navigate": true,
        "modal": false
    }
}
```

### 4. Все файлы из config.json `urls` должны существовать

Если `config.json` ссылается на `client-tab.html` — файл должен быть в zip.
Иначе iframe не загрузится и пользователь увидит ошибку.

### 5. Обязательные поля config.json

```json
{
    "name": "Название приложения",
    "version": "1.0.0",
    "package": "имя-папки",
    "description": "Краткое описание (1-2 предложения)",
    "about": "## Что делает\n...\n## Возможности\n...",
    "tags": "тег1, тег2, тег3",
    "logo": "icon.svg",
    "permissions": { ... },
    "urls": { ... }
}
```

---

## Паттерны

### Footer-виджет (самый частый)

```js
const App = new VMCRMUserApp();
App.getRequestParams().then(async ({data}) => {
    const resp = await App.fetch(`/db/${data.catalog}.json`);
    renderWidget(resp.data);
    App.setFrameSize(null, document.body.scrollHeight);
});
```

### Загрузка нескольких каталогов

```js
const [orders, clients] = await Promise.all([
    App.fetchAll('/db/b2b_orders.json'),
    App.fetch('/db/b2b_clients.json')
]);
```

### Наследование VMCRMUserApp

```js
// app.js
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
export default class MyApp extends VMCRMUserApp {
    async run() {
        const params = await this.getRequestParams();
        const data = await this.fetchAll(`/db/${params.data.catalog}.json`);
        this.render(data);
    }
    render(data) { /* ... */ }
}

// widget.html
import MyApp from './app.js';
new MyApp().run();
```

### Динамические колонки из схемы

```js
const schema = await App.fetch('/db/todo/sheme.json');
const statusMap = schema.data.fields.status.arr;
// statusMap = {1: 'Новая', 2: 'В работе', 3: 'Выполнена'}
```

### Хранилище как база настроек

```js
await App.storage.set('ifttt.token', tokenValue);
const token = await App.storage.get('ifttt.token');
```

### Демо-данные (fallback)

```js
let items = [];
try {
    const resp = await App.fetchAll('/db/projects.json');
    items = resp.data || [];
} catch(e) {
    items = [
        { alias: 'demo1', name: 'Демо проект 1', status: 'active' },
        { alias: 'demo2', name: 'Демо проект 2', status: 'done' }
    ];
}
```

### Выбор стека по сложности

| Задача | Стек |
|---|---|
| Виджет, диаграмма, простое действие | Vanilla JS |
| Интерактивная доска, календарь, сложный UI | Vue.js + Vuex |
| Страница настроек без данных | Vanilla JS + storage |

---

**Дальше:** [dashboards.md](dashboards.md) · **← [INDEX](INDEX.md)**
