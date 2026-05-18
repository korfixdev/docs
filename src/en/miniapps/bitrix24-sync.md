# Bitrix24 Synchronization (bitrix24_sync)

> **See also:** [data-api.md](data-api.md) · [storage-and-hooks.md](storage-and-hooks.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [Home](index.md)**

The platform supports two-way synchronization of catalogs with Bitrix24 CRM.
Miniapps can read and manage synchronization settings.

> Internal architecture, REST client, event.php — in [../backend/bitrix24-integration.md](../backend/bitrix24-integration.md)

---

## Catalogs

| Catalog | Description |
|---------|-------------|
| `bitrix24_sync` | Catalog mapping (which local catalog ↔ which Bitrix24 entity) |
| `bitrix24_sync_fieldmap` | Field mapping (which field ↔ which field) |

---

## Reading Sync Settings

```js
// All catalog mappings
const resp = await App.fetch('/db/bitrix24_sync.json');
resp.data.forEach(mapping => {
    console.log(mapping.local_catalog);    // "md_project"
    console.log(mapping.bitrix_catalog);   // "deal"
    console.log(mapping.direction);        // "2" (bidirectional)
    console.log(mapping.event_types);      // "0,1" (ADD, UPDATE)
});

// Field mappings for a specific sync
const fields = await App.fetch(`/db/bitrix24_sync_fieldmap.json?form[sync_id]=${mapping.id}`);
fields.data.forEach(f => {
    console.log(f.local_field);   // "name"
    console.log(f.bitrix_field);  // "TITLE"
    console.log(f.values_map);    // "draft,1\nin_progress,2"
});
```

---

## Sync Directions

| `direction` value | Description |
|-------------------|-------------|
| `0` | From Bitrix24 → local |
| `1` | Local → to Bitrix24 |
| `2` | Bidirectional |

## Event Types

| `event_types` value | Description |
|---------------------|-------------|
| `0` | Add (ADD) |
| `1` | Update (UPDATE) |
| `2` | Delete (DELETE) |

Stored comma-separated: `"0,1"` = ADD + UPDATE.

---

## Bitrix24 Entities

| `bitrix_catalog` value | Entity |
|------------------------|--------|
| `deal` | Deals |
| `lead` | Leads |
| `contact` | Contacts |
| `company` | Companies |
| `product` | Products |
| `productsection` | Product groups |
| `item` | Smart processes |
| `user` | Users |

---

## Creating a Catalog Mapping

```js
// Sync orders → Bitrix24 deals (updates only)
await App.fetch('/db/bitrix24_sync/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Orders → Deals',
        'form[local_catalog]': 'b2b_orders',
        'form[bitrix_catalog]': 'deal',
        'form[direction]': '1',            // to Bitrix
        'form[event_types]': ['0', '1'],   // ADD + UPDATE
        'form[hook_url]': 'https://mycompany.bitrix24.com/rest/1/abc123/',
        submit: 1
    }
});
```

## Creating a Field Mapping

```js
// Map "name" field to "TITLE" in Bitrix24
await App.fetch('/db/bitrix24_sync_fieldmap/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Name → TITLE',
        'form[sync_id]': syncId,           // ID from bitrix24_sync record
        'form[local_field]': 'name',
        'form[bitrix_field]': 'TITLE',
        submit: 1
    }
});

// With value mapping (statuses)
await App.fetch('/db/bitrix24_sync_fieldmap/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Status → STAGE_ID',
        'form[sync_id]': syncId,
        'form[local_field]': 'status',
        'form[bitrix_field]': 'STAGE_ID',
        'form[values_map]': 'new,NEW\nin_progress,EXECUTING\ncompleted,WON',
        submit: 1
    }
});
```

---

## Filtering Events

You can sync only records with specific field values:

```js
// Sync only won deals
await App.fetch('/db/bitrix24_sync/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Won deals only',
        'form[local_catalog]': 'ag_projects',
        'form[bitrix_catalog]': 'deal',
        'form[direction]': '0',               // from Bitrix
        'form[event_types]': ['0', '1'],
        'form[hook_url]': 'https://mycompany.bitrix24.com/rest/1/abc123/',
        'form[event_filter_by]': 'STAGE_ID',
        'form[event_filter_values]': 'WON',
        submit: 1
    }
});
```

---

## Use Cases for Miniapps

### Visualizing sync status

```js
// Check if a catalog is synced with Bitrix24
async function isSyncedWithBitrix(catalog) {
    const resp = await App.fetch(`/db/bitrix24_sync.json?form[local_catalog]=${catalog}`);
    return resp.data.length > 0;
}

// Get all synced catalogs
async function getSyncedCatalogs() {
    const resp = await App.fetchAll('/db/bitrix24_sync.json');
    return resp.data.map(m => ({
        local: m.local_catalog,
        remote: m.bitrix_catalog,
        direction: ['← Bitrix', '→ Bitrix', '↔ Bitrix'][m.direction]
    }));
}
```

### Integration setup wizard

An app can offer a step-by-step wizard: catalog selection,
field mapping, connection testing — simplifying setup for the user.

### Monitoring sync errors

```js
// Integration event logs
const logs = await App.fetch('/db/eventlogs.json?form[catalog]=bitrix24&limit=20');
logs.data.forEach(log => {
    console.log(log.ts, log.action, log.text_message);
});
```

---

## Important Notes

- Synchronization runs via cron — not real-time
- Incoming webhooks are queued (`core_evt`) and processed asynchronously
- Loop protection: an event from Bitrix24 does not trigger reverse sync
- `hook_url` — webhook URL from Bitrix24 settings (Developers → Incoming webhook)
- Access to `bitrix24_sync`: root + administrator
- Value mapping (`values_map`): CSV format, `local_value,bitrix_value` per line

---

*Catalogs: `/db/bitrix24_sync`, `/db/bitrix24_sync_fieldmap` | Backend: [../backend/bitrix24-integration.md](../backend/bitrix24-integration.md)*

**← [Home](index.md)**
