# Dashboards and Widgets

> **See also:** [config-json.md](config-json.md) · [styling.md](styling.md) · [data-api.md](data-api.md) · [self-provisioning.md](self-provisioning.md)
> **← [Home](index.md)**

Dashboards are configurable workspaces with widgets. A marketplace app can
embed its frames as dashboard widgets, and can also create standard widgets
(charts, tables) via the API.

---

## Data structure

### The `dashboards` catalog

Workspaces. Each user/group can have multiple dashboards.

| Field | Type | Description |
|-------|------|-------------|
| `id` | bigint | ID (used as `board_id` in widgets) |
| `alias` | varchar | Unique alias |
| `name` | varchar | Dashboard name |
| `prior` | int | Sort order |
| `from_auth` | int | Author (0 = system) |
| `from_group` | bigint | Group |

### The `dashboard_widgets` catalog

Widgets inside a dashboard. Linked to the dashboard via `board_id`.

| Field | Type | Description |
|-------|------|-------------|
| `id` | bigint | ID |
| `alias` | varchar | Unique alias |
| `name` | varchar | Widget title |
| `type` | varchar | Widget type (see reference below) |
| `width` | int | Width in Bootstrap columns (1-12). 4 = third, 6 = half, 12 = full |
| `aside` | int | Sidebar widget (0/1) |
| `options` | text | JSON with widget parameters (depends on type) |
| `board_id` | varchar | Dashboard ID (links to `dashboards.id`) |
| `prior` | int | Sort order (lower = higher) |
| `from_auth` | int | Author |
| `from_group` | bigint | Group |

---

## Widget types

### Chart widgets (data from catalogs)

| Type | Description | Visualization |
|------|-------------|---------------|
| `bar-chart` | Bar chart | Chart.js Bar |
| `pie-chart` | Pie chart | Chart.js Pie |
| `doughnut-chart` | Doughnut chart | Chart.js Doughnut |
| `line-chart` | Line chart | Chart.js Line |
| `aggr-table` | Pivot table | HTML table |

**options parameters (JSON):**

```json
{
    "catalog": "ag_cashflows",
    "pre-filter": "39",
    "fieldX": "project_id",
    "fieldY": "summa",
    "groupType": "sum",
    "colors": ["#5F67A8", "#E6576F", "#4E8F98"]
}
```

| Parameter | Description |
|-----------|-------------|
| `catalog` | Data source catalog |
| `pre-filter` | Saved filter ID (from `/db/saved_filters`) |
| `fieldX` | Field for X axis / grouping |
| `fieldY` | Field for Y axis / values |
| `groupType` | `"count"` — count, `"sum"` — sum |
| `colors` | Color array (hex). Korfix default palette |

### Table widgets

| Type | Description | options parameters |
|------|-------------|-------------------|
| `last-created` | Latest catalog records | `catalog`, `fields[]`, `limit` |

```json
{
    "catalog": "b2b_orders",
    "fields": ["id", "status", "client_id", "summ"],
    "limit": "8"
}
```

| Parameter | Description |
|-----------|-------------|
| `catalog` | Data source catalog |
| `fields` | Array of field names to display as columns |
| `limit` | Row count (default 5) |

### Built-in widgets

| Type | Description |
|------|-------------|
| `user-info` | Current user card |

### Marketplace app widget

| Type | Description |
|------|-------------|
| `app-frame` | Iframe with an installed app's frame |

```json
{
    "app_frame": "{token}:{frameName}"
}
```

| Parameter | Description |
|-----------|-------------|
| `app_frame` | Format: `{installed_apps.alias}:{frame_name_from_urls}` |

The frame receives standard parameters via VMCRMUserApp with `catalog = "dashboard_widgets"`.

---

## Working with the API

### Get dashboard list

```js
// Recommended /api/db/ — returns full list without server catalog filters
const boards = asArray(await App.fetch('/api/db/dashboards?limit=999'));
// [{id, alias, name, prior, from_auth, from_group}, ...]
```

### Get dashboard widgets

```js
const widgets = asArray(await App.fetchAll('/db/dashboard_widgets.json?form[board_id]=' + boardId));
// [{id, alias, name, type, width, options, board_id, prior}, ...]
```

### Create a chart widget

```js
// Bar chart of orders by client
await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Orders by client',
        'form[type]': 'bar-chart',
        'form[width]': 6,
        'form[board_id]': boardId,
        'form[prior]': 1,
        'form[options]': JSON.stringify({
            catalog: 'b2b_orders',
            fieldX: 'client_id',
            fieldY: 'summ',
            groupType: 'sum'
        }),
        submit: 1
    }
});
```

### Create a table widget

```js
// Latest tasks
await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Latest tasks',
        'form[type]': 'last-created',
        'form[width]': 12,
        'form[board_id]': boardId,
        'form[options]': JSON.stringify({
            catalog: 'tt_tasks',
            fields: ['name', 'status', 'person_id', 'complete'],
            limit: '10'
        }),
        submit: 1
    }
});
```

### Create an app-frame widget

An app can programmatically add its frame as a dashboard widget:

```js
// Need the install token and frame name from config.json urls
const params = await App.getRequestParams();
const appFrame = params.data.token + ':widget';  // token:frameName

await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'My Widget',
        'form[type]': 'app-frame',
        'form[width]': 6,
        'form[board_id]': boardId,
        'form[options]': JSON.stringify({ app_frame: appFrame }),
        submit: 1
    }
});
```

### Create a widget for custom catalog data

An app with self-provisioning can create widgets for its own data:

```js
// App created catalog custom_tickets (service desk)
// Now adding widgets to the dashboard

// Doughnut chart of tickets by status
await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Tickets by status',
        'form[type]': 'doughnut-chart',
        'form[width]': 4,
        'form[board_id]': boardId,
        'form[options]': JSON.stringify({
            catalog: 'custom_tickets',
            fieldX: 'custom_status',
            fieldY: 'id',
            groupType: 'count',
            colors: ['#5F67A8', '#4E8F98', '#E6576F', '#D6B075', '#45476A']
        }),
        submit: 1
    }
});

// Table of latest tickets
await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Latest tickets',
        'form[type]': 'last-created',
        'form[width]': 8,
        'form[board_id]': boardId,
        'form[options]': JSON.stringify({
            catalog: 'custom_tickets',
            fields: ['name', 'custom_status', 'custom_priority', 'custom_category'],
            limit: '5'
        }),
        submit: 1
    }
});
```

---

## Use-case: creating a dashboard on install

An app can create a dashboard with pre-configured widgets on first run:

```js
async function createDashboardWithWidgets() {
    // 1. Create dashboard
    const resp = await App.fetch('/db/dashboards/add?edit&ajax=1', {
        method: 'POST',
        body: { 'form[name]': 'Service Desk', submit: 1 }
    });

    // 2. Get the created dashboard ID
    const boards = asArray(await App.fetchAll('/db/dashboards.json'));
    const board = boards.find(b => b.name === 'Service Desk');
    if (!board) return;

    // 3. Add widgets
    const widgets = [
        { name: 'Tickets by status', type: 'doughnut-chart', width: 4,
          options: { catalog: 'custom_tickets', fieldX: 'custom_status', fieldY: 'id', groupType: 'count' }},
        { name: 'By priority', type: 'bar-chart', width: 4,
          options: { catalog: 'custom_tickets', fieldX: 'custom_priority', fieldY: 'id', groupType: 'count' }},
        { name: 'Latest tickets', type: 'last-created', width: 12,
          options: { catalog: 'custom_tickets', fields: ['name', 'custom_status', 'custom_priority'], limit: '8' }},
    ];

    for (let i = 0; i < widgets.length; i++) {
        await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
            method: 'POST',
            body: {
                'form[name]': widgets[i].name,
                'form[type]': widgets[i].type,
                'form[width]': widgets[i].width,
                'form[board_id]': board.id,
                'form[prior]': i + 1,
                'form[options]': JSON.stringify(widgets[i].options),
                submit: 1
            }
        });
    }

    App.alert('Dashboard "Service Desk" created with 3 widgets');
}
```

---

## Color reference

Korfix color palette for charts:

```js
const colors = [
    '#5F67A8', '#45476A', '#E6576F', '#5388AF', '#4E8F98',
    '#CBDDA6', '#2C55BF', '#D6B075', '#A25E8B', '#D48474'
];
```

---

## Sample app

A complete working example of dashboards and widgets — the **dashboard-share** app from the Korfix reference miniapps collection.

It implements widget export/import between dashboards and demonstrates:
- Loading dashboard list via `/api/db/dashboards`
- Reading widgets by `board_id`
- Bulk widget creation with unique `alias`
- Explicit `from_auth` / `from_group` for correct ownership
- The `itemsActions` integration point (catalog item context menu)

---

**Next:** [deploy.md](deploy.md) · **← [Home](index.md)**
