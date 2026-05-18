# Status Permissions (access_statuses)

> **See also:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [catalog-rules.md](catalog-rules.md)
> **← [Home](index.md)**

The `access_statuses` catalog controls which statuses are available
to each user role in each catalog.

> Internal mechanics (PHP, get_status_arr, get_disabled_statuses) — in [../backend/statuses.md](../backend/statuses.md)

---

## Concept

In catalogs with a `status` field, each status is a process stage.
Via `access_statuses`, an admin restricts which statuses are available
to a specific role. Without a record — all statuses are available.

**Example:** a manager moves an order to "In Progress",
but only a supervisor can set "Paid" or "Cancelled".

---

## Record Structure

| Field | Type | Description |
|-------|------|-------------|
| `dbmodule` | select | Catalog |
| `role` | select | Role (account type) |
| `statuses` | multiselect | Allowed statuses (whitelist) |

One record per catalog + role pair. Alias: `{dbmodule}_{role}`.

**Access:** root only (group creator). Other roles, including administrators, cannot manage status permissions.

---

## Reading Settings

```js
// All rules
const resp = await App.fetch('/db/access_statuses.json');

// For a specific catalog
const resp = await App.fetch('/db/access_statuses.json?form[dbmodule]=b2b_orders');
resp.data.forEach(rule => {
    console.log(rule.role);      // "3"
    console.log(rule.statuses);  // "10,20,30"
});
```

---

## Creating a Rule

```js
await App.fetch('/db/access_statuses/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[dbmodule]': 'b2b_orders',
        'form[role]': '3',
        'form[statuses]': ['10', '20'],
        submit: 1
    }
});
```

---

## Getting Catalogs, Roles, and Statuses

```js
const schema = await App.fetch('/db/access_statuses/sheme.json');

// Catalogs with status support
const catalogs = schema.data.dbmodule.arr;

// Roles (account types)
const roles = schema.data.role.arr;

// Statuses by catalog
const statusesByModule = schema.data.dbmodule.group_statuses;
// { "b2b_orders": { "10": "New", "20": "In Progress" }, ... }

// Or statuses for a specific catalog via its schema
const catSchema = await App.fetch('/db/b2b_orders/sheme.json');
const statuses = catSchema.data.status.arr;
```

---

## Use Cases for Miniapps

### Visualizing the process

```js
async function getStatusMatrix(catalog) {
    const [rulesResp, schemaResp] = await Promise.all([
        App.fetch(`/db/access_statuses.json?form[dbmodule]=${catalog}`),
        App.fetch(`/db/${catalog}/sheme.json`)
    ]);
    const allStatuses = schemaResp.data.status?.arr || {};
    return rulesResp.data.map(rule => ({
        role: rule.role,
        allowed: (rule.statuses || '').split(','),
        denied: Object.keys(allStatuses).filter(
            s => s && !(rule.statuses || '').split(',').includes(s)
        )
    }));
}
```

### Checking access before changing a status

```js
async function canSetStatus(catalog, statusId) {
    const resp = await App.fetch(`/db/access_statuses.json?form[dbmodule]=${catalog}`);
    if (resp.data.length === 0) return true;

    const params = await App.getRequestParams();
    const rule = resp.data.find(r => r.role === params.data.accountType);
    if (!rule) return true;

    return (rule.statuses || '').split(',').includes(String(statusId));
}
```

### Checking permissions before showing UI

Status permission management is only available to root (group creator).
Before showing the settings interface:

```js
const params = await App.getRequestParams();
if (params.data.userId !== params.data.groupId) {
    // Not root — show warning or hide section
    App.alert('Status permission settings are only available to the account owner');
    return;
}
// Root — show permission matrix
```

### Configuring workflow on install

An app can create status access rules on install,
setting up business processes tailored to company roles.

---

*Catalog: `/db/access_statuses` | Backend: [../backend/statuses.md](../backend/statuses.md)*

**← [Home](index.md)**
