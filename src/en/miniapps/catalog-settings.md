# Catalog Settings

> **See also:** [self-provisioning.md](self-provisioning.md) · [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [storage-and-hooks.md](storage-and-hooks.md)
> **← [Home](index.md)**

Users can configure catalog display via a modal (gear icon in the list header).
An app can read and save these settings to display data
in the same way as the standard CRM interface.

---

## Reading Settings

```js
const resp = await App.fetch('/db/tt_tasks/catalog/settings.json');
```

The response contains three blocks:

| Field | Description |
|-------|-------------|
| `data.table_fields` | Table columns: order, titles, visibility |
| `data.settings` | User settings by type (interface, table, scheme) |
| `data.search_form` | Search/filter form fields |

---

## Setting Types

### 1. interface — general catalog settings

Stored in: `settings.interface.{catalog}`

| Field | Type | Description |
|-------|------|-------------|
| `settingsInterfaceCatalogname` | string | Custom catalog name (renamed in menu and header) |
| `settingsInterfaceImgUrl` | string | Alternative path for images (if `doc` field exists in schema) |
| `settingsInterfaceCatalogKanban` | string | Field for kanban view (select or select_from_table) |

Example value:
```json
{
    "settingsInterfaceCatalogname": "My Tasks",
    "settingsInterfaceImgUrl": "/data/custom-images"
}
```

**Reading:**
```js
const resp = await App.fetch('/db/tt_tasks/catalog/settings.json');
const iface = resp.data.settings?.interface;
if (iface) {
    const customName = iface.value.settingsInterfaceCatalogname;
    // customName = 'My Tasks' (or undefined if not renamed)
}
```

---

### 2. table — column order and visibility

Stored in: `settings.table.{catalog}`

Value — object with single key `settingsFieldsOrder`: array of `{name, value}` objects.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Field name (field_name from schema) |
| `value` | boolean | `true` = column visible, `false` = hidden |

Array element order = column order in the table.

Example value:
```json
{
    "settingsFieldsOrder": [
        {"name": "name",      "value": true},
        {"name": "status",    "value": true},
        {"name": "person_id", "value": true},
        {"name": "ts",        "value": false},
        {"name": "complete",  "value": false}
    ]
}
```

**Reading via table_fields:**

`data.table_fields` — already processed result: user settings applied on top of
default catalog columns. Key order = display order:

```js
const resp = await App.fetch('/db/tt_tasks/catalog/settings.json');
const columns = resp.data.table_fields;

// columns = {
//   name:      { title: 'Name',     sort_by: 'ASC' },            // visible (no checked)
//   status:    { title: 'Status',   sort_by: 'ASC' },            // visible
//   person_id: { title: 'Assignee', sort_by: 'ASC', checked: false },  // hidden
// }

// Visible columns in user's order
const visible = Object.entries(columns)
    .filter(([_, col]) => col.checked !== false);
```

---

### 3. scheme — edit form field order

Stored in: `settings.scheme.{catalog}`

Value — object with key `settingsSchemeOrder`: array of `{name, value}` objects.
Defines field order in the add/edit form and tab assignment.

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Field name (field_name from schema). Fields of type `delimeter_line` = tab headings |
| `value` | string | `"1"` = field enabled |

Example value:
```json
{
    "settingsSchemeOrder": [
        {"name": "delimiter1",  "value": "1"},
        {"name": "name",        "value": "1"},
        {"name": "status",      "value": "1"},
        {"name": "person_id",   "value": "1"},
        {"name": "delimiter2",  "value": "1"},
        {"name": "cont",        "value": "1"}
    ]
}
```

`delimeter_line` fields separate form tabs. Element order = display order.

---

## Saving Settings

All settings are saved via `apps_storage` with `app_id = -1` (system settings, not tied to a marketplace app):

```js
await App.fetch('/empty/db/apps_storage/add?edit=1&ajax=y', {
    method: 'POST',
    body: {
        'form[app_id]': -1,
        'form[name]': 'settings.table.tt_tasks',   // key: settings.{type}.{catalog}
        'form[value]': JSON.stringify(data),         // value: JSON
        submit: 'ok'
    }
});
```

### Saving for yourself vs for everyone

| Parameter | Value | Result |
|-----------|-------|--------|
| without `from_group` | — | Settings saved for the current user |
| `from_group: 1` | true | Settings applied to all group users |

Personal settings take priority over group settings.

### Resetting Settings

To reset — save with `form[hidden] = 1`:

```js
await App.fetch('/empty/db/apps_storage/add?edit=1&ajax=y', {
    method: 'POST',
    body: {
        'form[app_id]': -1,
        'form[name]': 'settings.table.tt_tasks',
        'form[hidden]': 1,
        submit: 'ok'
    }
});
```

---

## Full Use-Case: Table with User Settings

```js
const App = new VMCRMUserApp();

async function renderTable(catalog) {
    // 1. Fetch settings + schema + data in parallel
    const [settingsResp, schemaResp, dataResp] = await Promise.all([
        App.fetch(`/db/${catalog}/catalog/settings.json`),
        App.fetch(`/db/${catalog}/sheme.json`),
        App.fetchAll(`/db/${catalog}.json`)
    ]);

    const columns = settingsResp.data.table_fields;
    const schema  = schemaResp.data;
    const items   = Array.isArray(dataResp?.data) ? dataResp.data : [];

    // 2. Visible columns in user's order
    const visibleCols = Object.entries(columns)
        .filter(([_, col]) => col.checked !== false)
        .map(([name, col]) => ({
            name,
            title: col.title,
            type: schema[name]?.type,
            arr: schema[name]?.arr   // for select — options
        }));

    // 3. Render headers
    let html = '<table><thead><tr>';
    visibleCols.forEach(col => { html += `<th>${col.title}</th>`; });
    html += '</tr></thead><tbody>';

    // 4. Render rows with select value substitution
    items.forEach(item => {
        html += '<tr>';
        visibleCols.forEach(col => {
            let val = item[col.name] ?? '';
            // For select — show text instead of key
            if (col.type === 'select' && col.arr && col.arr[val] !== undefined) {
                val = col.arr[val];
            }
            html += `<td>${val}</td>`;
        });
        html += '</tr>';
    });
    html += '</tbody></table>';
    return html;
}
```

---

## Full Use-Case: Column Settings Editor

An app can provide its own UI for configuring columns:

```js
// 1. Load current settings
const resp = await App.fetch(`/db/${catalog}/catalog/settings.json`);
const columns = resp.data.table_fields;

// 2. Show checkboxes (user enables/disables and sorts)
// ... UI with drag & drop ...

// 3. Save the changed order
const newOrder = [
    { name: 'name',      value: true },
    { name: 'status',    value: true },
    { name: 'person_id', value: false },  // hide
];

await App.fetch('/empty/db/apps_storage/add?edit=1&ajax=y', {
    method: 'POST',
    body: {
        'form[app_id]': -1,
        'form[name]': `settings.table.${catalog}`,
        'form[value]': JSON.stringify({ settingsFieldsOrder: newOrder }),
        submit: 'ok'
    }
});

// 4. Reload to apply
App.reload();
```

---

## Settings Key Reference

| Storage key | Setting type | Affects |
|-------------|-------------|---------|
| `settings.interface.{catalog}` | General | Catalog name in menu, image path |
| `settings.table.{catalog}` | Table | Column order and visibility in list view |
| `settings.scheme.{catalog}` | Form | Field order and tabs in edit form |

All keys follow format `settings.{type}.{catalog}`, where `{catalog}` is the catalog alias (e.g. `tt_tasks`, `b2b_orders`, `custom_tickets`).

---

**← [Home](index.md)**
