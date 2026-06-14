# Views (custom_dbview)

> **See also:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [self-provisioning.md](self-provisioning.md)
> **← [Home](index.md)**

Combine two catalogs into a single view via MySQL VIEW + LEFT JOIN.
The result works as a regular catalog — with filtering, API, and widgets.

> **Reading data.** Single request → `App.fetchV2()` (`(await App.fetchV2(url)).data ?? []`). Paginated
> reads use `App.fetchAll()` which has no V2 variant — normalize its legacy shape with `asArray`:
> ```js
> function asArray(resp) {
>     if (Array.isArray(resp?.data)) return resp.data;
>     if (Array.isArray(resp?.data?.data)) return resp.data.data;
>     return [];
> }
> ```
> The `asArray(await App.fetchAll(...))` snippets below assume this helper is defined.

---

## When to Use

### Reports on related data

Need an "orders + clients" or "tasks + projects + employees" report? Instead of
loading two catalogs and merging in JS — create a view and work with it as one.

```js
// Without view: two requests + merge in JS
const orders = asArray(await App.fetchAll('/db/b2b_orders.json'));
const clients = asArray(await App.fetchAll('/db/b2b_clients.json'));
const merged = orders.map(o => ({
    ...o,
    clientName: clients.find(c => c.id === o.client_id)?.name
}));

// With view: one request, data already joined
const data = asArray(await App.fetchAll('/db/custom_orders_clients_view.json'));
// data[0].t1_summ, data[0].t2_name — fields from both catalogs
```

### Summary tables for the dashboard

The `last-created` and `aggr-table` dashboard widgets work with a single catalog.
A view lets you show data from two catalogs in one widget without writing a
custom app.

### Filter by fields from another catalog

A regular catalog can only be filtered by its own fields. A view lets you filter
orders by client name, tasks by project title, etc.

### Export of joined data

The standard catalog export (CSV/API) will export view data with all
joined fields — without extra code.

---

## How to Create a View

### Via the interface

1. Go to `/db/custom_dbview`
2. Add a record:
   - **Name** — e.g., "Orders with Clients"
   - **Table name (dbname)** — `orders_clients` (becomes `custom_orders_clients_view`)
   - **Catalog 1** — `b2b_orders`
   - **Catalog 1 field** — `client_id`
   - **Catalog 2** — `b2b_clients`
   - **Catalog 2 field** — `id`
3. Save → MySQL VIEW is created automatically

### Via API (from a miniapp)

```js
// Create a view programmatically
await App.fetch('/db/custom_dbview/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[alias]': Date.now().toString(36) + Math.random().toString(36).substr(2, 8),
        'form[name]': 'Orders with Clients',
        'form[dbname]': 'orders_clients',
        'form[catalog1]': 'b2b_orders',
        'form[catalog1_field]': 'client_id',
        'form[catalog2]': 'b2b_clients',
        'form[catalog2_field]': 'id',
        // from_group omitted — server forces it; from_auth omitted → personal (pass 0 for group-shared)
        submit: 1
    }
});
```

---

## View Data Structure

### Field naming

Fields from both catalogs get prefixes to avoid conflicts:

| Source | Prefix | Example |
|--------|--------|---------|
| Catalog 1 | `t1_` | `t1_summ`, `t1_status`, `t1_client_id` |
| Catalog 2 | `t2_` | `t2_name`, `t2_email`, `t2_phone` |

**Exceptions:** system fields `id`, `from_auth`, `from_group`, `hidden` are taken
from catalog 1 without prefix.

### Generated SQL

```sql
CREATE OR REPLACE VIEW custom_orders_clients_view AS
SELECT
  c1.id, c1.from_auth, c1.from_group, c1.hidden,
  c1.summ        AS t1_summ,
  c1.status      AS t1_status,
  c1.client_id   AS t1_client_id,
  c2.name        AS t2_name,
  c2.email       AS t2_email,
  c2.phone       AS t2_phone
FROM b2b_orders AS c1
LEFT JOIN b2b_clients AS c2 ON c1.client_id = c2.id
```

LEFT JOIN — all catalog 1 records are kept even if there is no match
in catalog 2 (`t2_*` fields will be `NULL`).

---

## Working with Views from a Miniapp

### Reading data

```js
// Like a regular catalog — .json or /api/db/
const data = asArray(await App.fetchAll('/db/custom_orders_clients_view.json'));

// Or via REST API
const data = asArray(await App.fetch('/api/db/custom_orders_clients_view?limit=999'));
```

### Filtering

```js
// Filter by catalog 1 field (orders with status "paid")
const paid = asArray(await App.fetchAll(
    '/db/custom_orders_clients_view.json?form[t1_status]=paid'
));

// Filter by catalog 2 field (orders by client "Smith")
const smith = asArray(await App.fetchAll(
    '/db/custom_orders_clients_view.json?form[t2_name]=Smith'
));
```

### Getting the schema

```js
const schema = await App.fetch('/db/custom_orders_clients_view/sheme.json');
// schema.data contains joined fields from both catalogs with t1_/t2_ prefixes
```

### Checking if a view exists

```js
// Before using — check the view exists
try {
    const resp = await App.fetch('/api/db/custom_orders_clients_view?limit=1');
    // View exists, proceed
} catch(e) {
    // View not created — show instruction to user
}
```

---

## Use Cases for Miniapps

### Analytical application

A P&L or BI analytics app can create the needed views at install time (self-provisioning)
and build reports on joined data — without multiple fetches:

```js
// On first run: create view if it doesn't exist
const views = asArray(await App.fetch('/api/db/custom_dbview?limit=999'));
const exists = views.some(v => v.dbname === 'cashflows_clients');

if (!exists) {
    await App.fetch('/db/custom_dbview/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[alias]': uid(),
            'form[name]': 'Transactions with Counterparties',
            'form[dbname]': 'cashflows_clients',
            'form[catalog1]': 'ag_cashflows',
            'form[catalog1_field]': 'client_id',
            'form[catalog2]': 'ag_clients',
            'form[catalog2_field]': 'id',
            // from_group omitted — server forces it; from_auth omitted → personal (pass 0 for group-shared)
            submit: 1
        }
    });
}

// Then work with it as a regular catalog
const data = asArray(await App.fetchAll('/db/custom_cashflows_clients_view.json'));
```

### Dashboard widget

A `last-created` widget for a view shows joined data:

```json
{
    "catalog": "custom_orders_clients_view",
    "fields": ["t1_summ", "t1_status", "t2_name"],
    "limit": "10"
}
```

### Export of related data

An import-export app can work with a view as a regular catalog —
CSV export with fields from both sources.

---

## Limitations

| Limitation | Description |
|------------|-------------|
| **Read-only** | Views are read-only by default (`ue=0, ud=0`). To edit — work with the source catalogs |
| **Static schema** | Adding fields to a source catalog does NOT update the view. Recreate it |
| **No CASCADE** | Deleting a source catalog breaks the view without warning |
| **Two catalogs** | Only JOIN of two catalogs. For three — create a view from a view + third catalog |
| **Performance** | On large tables (100k+ records) views can be slow — MySQL does not optimize indexes for VIEWs |
| **NULL on missing** | LEFT JOIN: if no match in catalog 2, `t2_*` fields will be `NULL` |

---

## Permissions for Apps

If a miniapp creates or reads views, add to `config.json`:

```json
"permissions": {
    "catalogs": {
        "custom_dbview": ["read", "write"],
        "custom_orders_clients_view": ["read"]
    }
}
```

`custom_dbview` — for creating views. `custom_*_view` — for reading data from a specific view.

---

**← [Home](index.md)**
