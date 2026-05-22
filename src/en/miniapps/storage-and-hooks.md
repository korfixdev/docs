# Storage API — Key-Value Store

> **See also:** [data-api.md](data-api.md) · [catalog-rules.md](catalog-rules.md) · [js-api.md](js-api.md) · [self-provisioning.md](self-provisioning.md)
> **← [Home](index.md)**

An isolated key-value store for each installed app, plus webhooks and afterSave.

---

Each installed app has an isolated storage (bound by token — the `alias` of the `installed_apps` record).

> ⚠️ **Reinstalling an app destroys its storage.**
> When you uninstall and reinstall, a new `installed_apps` record is created with a new `token`.
> The old `apps_storage` records are left orphaned and inaccessible to the new `token`.
> If the data matters — export or migrate it before uninstall, or use a
> **custom catalog** (`custom_dbtables`) instead: it is bound to the group and survives reinstallation.

> **Important: storage is isolated per user, not just per app.**
> Each record stores `from_auth` (user ID). Manager A and Manager B
> in the same group will get **different** values for the same key.
> The group owner (admin) is an exception: their records are stored with `from_auth = 0`
> and effectively become "group-wide" (shared across the instance).
>
> For truly shared storage use a **custom catalog**
> (`custom_dbtables`) instead of `App.storage`.

```js
// Save a setting
await App.storage.set('my.setting', 'value');

// Read one record
const record = await App.storage.get('my.setting');
// record = {name: 'my.setting', value: 'hello', alias: '...', app_id: '...'}
const val = record?.value;  // 'hello'

// All app keys
const all = await App.storage.get('');
// all = [{name: 'key1', value: 'val1'}, {name: 'key2', value: 'val2'}, ...]

// Delete
await App.storage.unset('my.setting');
```

> ⚠️ **`get()` returns a RECORD (object), not a value.** Common mistake:
>
> ```js
> // ❌ WRONG — typecasting to string gives "[object Object]"
> const val = await App.storage.get('my.setting');
> console.log(val);                     // [object Object]
> element.textContent = val;            // "[object Object]" in UI
>
> // ✅ CORRECT — take the .value field
> const rec = await App.storage.get('my.setting');
> const val = rec?.value ?? '';
> console.log(val);                     // 'hello'
> ```
>
> If you need to read as an object (stored via `JSON.stringify`):
> ```js
> const rec = await App.storage.get('my.config');
> const cfg = rec?.value ? JSON.parse(rec.value) : {};
> ```

> **Important: `value` is always a string.** `App.storage.set(key, value)` sends value as
> a string via POST. If you pass an object or array — `[object Object]` will be saved.
> For complex structures use `JSON.stringify()` / `JSON.parse()`:
>
> ```js
> // Save an object/array
> const config = { theme: 'dark', columns: ['name', 'status'] };
> await App.storage.set('my.config', JSON.stringify(config));
>
> // Read an object/array
> const record = await App.storage.get('my.config');
> const config = record?.value ? JSON.parse(record.value) : {};
> ```
>
> **`get()` response format:**
> - Single record: returns object `{name, value, alias, app_id, ...}` — value in the `.value` field
> - All records (`get('')`): returns array of objects `[{name, value, ...}, ...]`
> - Not found: returns `undefined` or the second argument (default)

### Webhooks via storage

An app can subscribe to webhooks via storage:

```js
// Hook on edits to a specific catalog
await App.storage.set(
    'event.hook.activity.projects.отредактировал.json',
    'https://example.com/api/webhook'
);

// Hook on all events in all catalogs
await App.storage.set(
    'event.hook.activity.*.*.json',
    'https://example.com/api/webhook'
);
```

Key format: `event.hook.activity.{catalog}.{event}[.json]`
- `{catalog}` — catalog name or `*`
- `{event}` — `добавил` (added), `отредактировал` (edited), `удалил` (deleted) or `*`
- `.json` at the end — send as `application/json`

### afterSave — webhook on catalog data changes

Called when a catalog item is saved or deleted. POST request to the handler URL.

#### Two subscription methods

**1. Static (in config.json)** — for remote apps:

```json
{
  "catalogs": {
    "tt_tasks": { "afterSave": "webhook-frame" },
    "": { "afterSave": "webhook-frame" }
  }
}
```

`""` — subscribe to all catalogs. `"webhook-frame"` — key from `urls`.

**2. Dynamic (via storage)** — from any app at runtime:

```js
// Specific catalog
await App.storage.set('event.hook.after.save.tt_tasks.json', 'https://example.com/webhook');
// All catalogs
await App.storage.set('event.hook.after.save.json', 'https://example.com/webhook');
```

`.json` suffix = `Content-Type: application/json`. Without suffix = `application/x-www-form-urlencoded`.

#### Webhook data (payload)

| Field | Type | Description |
|-------|------|-------------|
| `catalog` | string | Catalog name (`tt_tasks`, `b2b_orders`, ...) |
| `action` | string | Operation type (see table below) |
| `cmd` | string | Item alias |
| `form` | object | **All item fields after the operation** |
| `prev_form` | object\|null | **Previous state before the operation.** `null` when creating a new item |
| `token` | string | App installation alias |
| `app_id` | string | App alias in the marketplace |
| `domain` | string | CRM domain |
| `user` | string | MD5 hash of the user's login |

#### Operation types (action)

| action | When | form | prev_form |
|--------|------|------|-----------|
| `save` | Creating or editing an item | Data after save | Data before change (`null` on create) |
| `delete` | Deleting an item (to trash) | Data of deleted item | Not sent |

#### Determining the change type

```php
// Handler side (PHP)
$form = $_POST['form'] ?? [];
$prev = $_POST['prev_form'] ?? [];

// New item created
if (empty($prev)) {
    // New item was created
}

// Edit — compare fields
if (!empty($prev) && $form['status'] !== $prev['status']) {
    // Status changed
}

// Delete
if ($_POST['action'] === 'delete') {
    // Item deleted
}
```

```js
// Handler side (Node.js / n8n)
const { action, form, prev_form, catalog } = req.body;

if (action === 'save' && !prev_form) {
    console.log('New item created:', form.name);
}

if (action === 'save' && prev_form && form.status !== prev_form.status) {
    console.log(`Status changed: ${prev_form.status} → ${form.status}`);
}

if (action === 'delete') {
    console.log('Deleted:', form.name);
}
```

#### Use-case examples

**Notification on task status change:**

```js
// On app install — subscribe to tt_tasks
await App.storage.set(
    'event.hook.after.save.tt_tasks.json',
    'https://myapp.example.com/api/task-status-changed'
);
```

Handler compares `form.status` and `prev_form.status`:
- If changed → send Telegram notification
- If unchanged → ignore

**Auto-create record on new order:**

```js
await App.storage.set(
    'event.hook.after.save.b2b_orders.json',
    'https://myapp.example.com/api/new-order'
);
```

Handler checks `prev_form === null` (new order) and creates a task in the TT module.

### event.hook.activity — webhook on user actions

Alternative hook type — fires on activity log entries (user actions).

```js
// Format: event.hook.activity.{catalog}.{action}[.json]
// {action} = добавил, отредактировал, удалил or * for all

await App.storage.set(
    'event.hook.activity.b2b_orders.добавил.json',
    'https://myapp.example.com/api/order-added'
);

// All actions on all catalogs
await App.storage.set(
    'event.hook.activity.*.*.json',
    'https://myapp.example.com/api/all-activity'
);
```

Payload contains the same fields (`catalog`, `token`, `app_id`, `domain`, `user`) plus activity data (`activity`, `activity_text`).

---

### App install and uninstall hooks

The platform fires events when an app is installed or uninstalled.
A remote app can use these for initialization (creating data structures,
registering webhooks) or cleanup on uninstall.

| Event | When | Data |
|-------|------|------|
| `app.installed` | After app installation | `app_id`, `token`, `form`, `user_id`, `from_group` |
| `app.uninstalled` | After app uninstall | `app_id`, `token`, `form`, `user_id`, `from_group` |

For **remote apps** (with `install_url`) — the platform POSTs to the frame URL
with this data. The app can perform initialization and respond.

For **local apps** (zip) — hooks are called server-side via `int_done_run()`.
A local app cannot subscribe to them directly, but can use a storage webhook:

```js
// On first run — subscribe to the uninstall event
// (to clean up data if the app is uninstalled)
await App.storage.set(
    'event.hook.activity.installed_apps.удалил.json',
    'https://example.com/api/app-cleanup'
);
```

---

**Next:** [self-provisioning.md](self-provisioning.md) · **← [Home](index.md)**
