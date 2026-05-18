# Styling Miniapps for the Platform

> **See also:** [js-api.md](js-api.md) · [dashboards.md](dashboards.md) · [getting-started.md](getting-started.md) · [checklist.md](checklist.md)
> **← [Home](index.md)**

Styling, CSS variables, HTML structure, JS and Chart.js integration.

---

## Principles

1. The app renders in an **iframe** — platform styles are not inherited
2. To look native, use **platform CSS variables** (see below)
3. **Don't duplicate** jQuery, Bootstrap, and other libraries — load from the portal or CDN
4. Font — `"Open Sans"`, with system fallback
5. Structural wrappers (`body.widget`, `main.content`, `article.content__main`) — optional, for full-screen apps

---

## Platform CSS variables (design tokens)

Full set of variables from the Korfix design system. Copy into `:root` of your app — use what you need.

### Base colors

```css
:root {
    /* Main */
    --primary: #323C8F;
    --secondary: #343859;
    --success: #33BE2B;
    --info: #17a2b8;
    --warning: #FF4D50;
    --danger: #EF233C;
    --light: #fff;
    --dark: #1D1E32;

    /* Accent */
    --blue: #475CFF;
    --purple: #323C8F;
    --orange: #FF4D50;
    --red: #EF233C;
}
```

### Grays

```css
:root {
    --white: #fff;
    --gray1: #4e4f56;
    --gray2: #78797f;        /* table header text */
    --gray3: #a3a3a7;
    --gray4: #b8b8bb;
    --gray5: #dddde1;
    --gray6: #eaeaee;        /* borders, dividers */
    --gray7: #f6f6f6;
    --gray8: #f7f7f8;        /* alternating table rows */
    --gray9: #f9f9f9;
}
```

### Blue/blue-gray shades

```css
:root {
    --bluegray3: #57596e;
    --bluegray4-6d6f89: #6d6f89;
    --bluegray5: #8a8ca1;    /* secondary text */
    --bluegray6: #b9bdcd;
    --bluegray7: #c5cadc;
    --bluegray8: #dce0ef;    /* element borders */
    --bluegray9: #eceffa;
    --bluegray10: #f4f5fa;   /* card backgrounds */
    --bluegray11: #f8f8fd;
    --bluegray12: #F5F5F8;
    --bluegray13: #FAFAFC;

    --blue3-323c8f: #323c8f;  /* links */
    --blue4: #5a63b4;
    --blue9: #eff1fc;         /* badge backgrounds */
}
```

### Status colors

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

### Typography

```css
:root {
    --font-family-sans-serif: "Open Sans", sans-serif, -apple-system,
        BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial,
        "Noto Sans", "Liberation Sans", sans-serif;
    --font-family-monospace: SFMono-Regular, Menlo, Monaco, Consolas,
        "Liberation Mono", "Courier New", monospace;
}
```

### Breakpoints

```css
:root {
    --breakpoint-sm: 576px;
    --breakpoint-md: 768px;
    --breakpoint-lg: 992px;
    --breakpoint-xl: 1200px;
}
```

---

## Font

The platform uses **Open Sans**. Load via CDN if unsure whether the font is loaded in the parent:

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

## HTML page structure

### Minimal template (widget/popup)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My App</title>
    <style>
        :root { /* needed variables */ }
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

### Full-screen template (menu item)

For full-screen apps use the platform wrappers:

```html
<body class="widget">
<main class="content m-0">
    <article class="content__main">
        <div class="content__common pt-4">
            <!-- your content -->
        </div>
    </article>
</main>
</body>
```

Helper CSS for structural classes:

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

## UI components

### Buttons

```css
.btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 16px;
    border: none;
    border-radius: 3px;       /* matches platform */
    cursor: pointer;
    font: 500 13px/1.4 "Open Sans", sans-serif;
    transition: .15s;
}

/* Primary (blue) */
.btn-primary {
    background: var(--primary);
    color: #fff;
    border-bottom: 3px solid #2A6AA4;
}
.btn-primary:hover { opacity: 0.9; }

/* Success (green) */
.btn-success {
    background: var(--success);
    color: #fff;
    border-bottom: 3px solid #2B9C5B;
}

/* Danger (red) */
.btn-danger {
    background: var(--danger);
    color: #fff;
    border-bottom: 3px solid #B7433F;
}

/* Secondary (outline) */
.btn-outline {
    background: var(--white);
    color: var(--primary);
    border: 1px solid var(--bluegray8);
}
.btn-outline:hover { border-color: var(--primary); }

.btn:disabled { opacity: .5; cursor: not-allowed; }
```

### Cards

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

### Tables

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

### Forms (select, input, textarea)

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

### Status / notifications

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

### Badges

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

### Tabs

> **Important:** use `<a href="javascript:void(0)">` instead of `<div>` for tabs.
> On mobile (especially iOS Safari) `click` on a `<div>` may not fire.

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

### Links

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

## JS: what to load from the portal vs CDN

### From the portal (absolute path in iframe)

```js
// VMCRMUserApp — required, main API
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
```

This is the only JS module to import from the portal. It provides:
`fetch`, `fetchAll`, `modal`, `alert`, `navigate`, `setFrameSize`, `storage`, `on`.

### From CDN (when needed)

```html
<!-- Chart.js (charts) -->
<script src="https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js"></script>

<!-- Mermaid (diagrams) -->
<script type="module">
import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
</script>
```

### What NOT to include

- **jQuery** — not used in miniapps, write in Vanilla JS (ES6+)
- **Bootstrap CSS/JS** — don't include in full; if you need utility classes — define them locally
- **Open Sans font** — load via Google Fonts CDN if needed

---

## Chart.js — platform palette

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

## Responsiveness and mobile

### Required rules

1. **Tables → cards on mobile.** Wide tables don't fit on phone screens. Use media queries to switch to a tile layout:

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

HTML markup for `data-label`:
```html
<td data-label="Status"><span class="badge-type">active</span></td>
<td data-label="Amount">$150.00</td>
```

2. **Input font size — at least 16px.** iOS Safari automatically zooms the page when focusing on an input with `font-size < 16px`:

```css
/* CORRECT — no zoom on iOS */
input, select, textarea {
    font-size: 16px;  /* minimum! */
}

/* WRONG — will trigger zoom */
input { font-size: 13px; }
```

3. **Clickable elements — `<a>` or `<button>`, not `<div>`.** iOS Safari may not fire `click` on `<div>`:

```html
<!-- CORRECT -->
<a class="tab" href="javascript:void(0)" data-tab="export">Export</a>
<button class="btn btn-primary" id="btnExport">Export</button>

<!-- WRONG — may not work on iOS -->
<div class="tab" data-tab="export">Export</div>
```

### Other

```css
@media (max-width: 768px) {
    .tabs { flex-direction: column; }
    .card { padding: 12px; }
    .actions { flex-direction: column; }
}
```

```js
// In JS — for control type selection
const isMobile = window.innerWidth < 768;
```

---

## Settings icon (gear)

Every app's interface should have a ⚙ icon for access to:
- App settings (if any)
- Install screen / self-provisioning (with log and progress)
- Version info

```html
<div class="header">
    <h2>My App</h2>
    <a href="javascript:void(0)" id="btnSettings" style="margin-left:auto;color:var(--bluegray5);">
        <i class="fa fa-cog"></i>
    </a>
</div>
```

The install result (flag `installed: true`) is saved in `App.storage` so the install screen doesn't reappear.

---

## Frame auto-resize (REQUIRED)

> **This is the most common mistake in miniapps.** Audits show 70% of apps
> don't call `setFrameSize` — content gets cut off, users can't see part of the UI.
> **Every** DOM change requires calling `setFrameSize`.

The iframe doesn't auto-adjust — you must call `setFrameSize` after **every** content change: rendering a list, switching tabs, expanding an accordion, showing/hiding blocks, loading data.

### Required setup

```css
/* REQUIRED on body — removes scrollbar inside iframe */
body { overflow: hidden; }
```

```js
// Helper — REQUIRED: create and call after ANY DOM change
function resizeFrame() {
    requestAnimationFrame(() => App.setFrameSize(null, document.body.scrollHeight));
}
```

### Where to call it

```js
// After rendering a list
list.innerHTML = items.map(renderItem).join('');
resizeFrame();

// After loading data
App.fetch('/db/catalog.json').then(resp => {
    renderTable(resp.data);
    resizeFrame();
});

// After switching tabs
tab.addEventListener('click', () => {
    showPanel(tab.dataset.panel);
    resizeFrame();
});

// After expanding/collapsing a block
toggle.addEventListener('click', () => {
    body.classList.toggle('open');
    resizeFrame();
});

// On initialization
App.getRequestParams().then(({data}) => {
    renderWidget(data);
    resizeFrame();
});
```

### Anti-patterns

```js
// WRONG — +20 masks the problem, gives inaccurate height
App.setFrameSize(null, document.body.scrollHeight + 20);

// WRONG — calling without requestAnimationFrame, DOM hasn't updated yet
container.innerHTML = html;
App.setFrameSize(null, document.body.scrollHeight); // measures old height

// WRONG — not calling setFrameSize at all
container.innerHTML = renderBigTable(data); // content will be cut off

// CORRECT
container.innerHTML = html;
requestAnimationFrame(() => App.setFrameSize(null, document.body.scrollHeight));
```

---

## Required rules (common mistakes)

An audit of 37 apps revealed typical mistakes. Follow these rules:

### 1. Always use `App.fetch()`, not native `fetch()`

The miniapp runs in an iframe — native `fetch()` to platform endpoints is blocked by CORS.
`App.fetch()` proxies the request via `postMessage` to the parent window.

```js
// WRONG — CORS error
const resp = await fetch('/db/projects.json');

// CORRECT
const resp = await App.fetch('/db/projects.json');
```

### 2. Use `App.storage`, not `localStorage`

`localStorage` in an iframe is isolated — data will be lost when the app domain changes.

```js
// WRONG
localStorage.setItem('settings', JSON.stringify(data));

// CORRECT
await App.storage.set('settings', JSON.stringify(data));
```

### 3. Declare `permissions` in config.json

Every catalog and operation used in code must be listed in `permissions`.
Without this the app may be blocked by the sandbox.

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

### 4. All files referenced in config.json `urls` must exist

If `config.json` references `client-tab.html` — the file must be in the zip.
Otherwise the iframe won't load and the user sees an error.

### 5. Required config.json fields

```json
{
    "name": "App Name",
    "version": "1.0.0",
    "package": "folder-name",
    "description": "Short description (1-2 sentences)",
    "about": "## What it does\n...\n## Features\n...",
    "tags": "tag1, tag2, tag3",
    "logo": "icon.svg",
    "permissions": { ... },
    "urls": { ... }
}
```

---

## Patterns

### Footer widget (most common)

```js
const App = new VMCRMUserApp();
App.getRequestParams().then(async ({data}) => {
    const resp = await App.fetch(`/db/${data.catalog}.json`);
    renderWidget(resp.data);
    App.setFrameSize(null, document.body.scrollHeight);
});
```

### Loading multiple catalogs

```js
const [orders, clients] = await Promise.all([
    App.fetchAll('/db/b2b_orders.json'),
    App.fetch('/db/b2b_clients.json')
]);
```

### Extending VMCRMUserApp

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

### Dynamic columns from schema

```js
const schema = await App.fetch('/db/todo/sheme.json');
const statusMap = schema.data.fields.status.arr;
// statusMap = {1: 'New', 2: 'In progress', 3: 'Done'}
```

### Storage as settings store

```js
await App.storage.set('ifttt.token', tokenValue);
const token = await App.storage.get('ifttt.token');
```

### Demo data (fallback)

```js
let items = [];
try {
    const resp = await App.fetchAll('/db/projects.json');
    items = resp.data || [];
} catch(e) {
    items = [
        { alias: 'demo1', name: 'Demo Project 1', status: 'active' },
        { alias: 'demo2', name: 'Demo Project 2', status: 'done' }
    ];
}
```

### Stack selection by complexity

| Task | Stack |
|---|---|
| Widget, chart, simple action | Vanilla JS |
| Interactive board, calendar, complex UI | Vue.js + Vuex |
| Settings page without data | Vanilla JS + storage |

---

**Next:** [self-provisioning.md](self-provisioning.md) · **← [Home](index.md)**
