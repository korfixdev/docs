# Push Notification Settings (systempush_settings)

> **See also:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [account-help.md](account-help.md)
> **← [Home](index.md)**

The `systempush_settings` catalog — personal subscriptions for a user
to receive push notifications from platform catalogs.

---

## How It Works

The platform sends push notifications on catalog actions (add, edit, delete records).
By default, notifications **are not sent** — the user explicitly selects which
sections and events to receive pushes for.

Each record in `systempush_settings` is a subscription to one catalog
with optional filters.

---

## Record Structure

| Field | Type | Description |
|-------|------|-------------|
| `alias` | hidden | md5(menu + userId) — set automatically |
| `menu` | select | Catalog (section) to receive notifications from |
| `restrict_entries` | multiselect | Entry filter: `self` — own records only, `members` — where user is a watcher |
| `actions` | multiselect | Action filter: `add`, `edit`, `delete` |
| `from_auth` | hidden | User ID |

Access: `self` — each user manages only their own subscriptions.
Multiple records: one per subscribed catalog.

---

## Reading Subscriptions

```js
// All subscriptions for the current user
const resp = await App.fetch('/db/systempush_settings.json');
resp.data.forEach(sub => {
    console.log(sub.menu);              // "tt_tasks"
    console.log(sub.restrict_entries);  // "self,members" or ""
    console.log(sub.actions);           // "add,edit" or ""
});
```

---

## Subscribing to a Catalog

```js
// Subscribe user to task notifications
// (own records only, add and edit events only)
await App.fetch('/db/systempush_settings/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[menu]': 'tt_tasks',
        'form[restrict_entries][]': ['self', 'members'],
        'form[actions][]': ['add', 'edit'],
        submit: 1
    }
});
```

> Alias is generated automatically server-side: `md5(menu + userId)`.
> Re-subscribing to the same catalog updates the existing record.

### Subscribe with no restrictions (all records, all actions)

```js
await App.fetch('/db/systempush_settings/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[menu]': 'b2b_orders',
        submit: 1
    }
});
```

---

## Unsubscribing

```js
// Find the subscription and delete it
const resp = await App.fetch('/db/systempush_settings.json?form[menu]=tt_tasks');
if (resp.data.length) {
    await App.fetch(`/db/systempush_settings/${resp.data[0].alias}?udel`);
}
```

---

## Getting Available Catalogs for Subscription

```js
// Schema contains list of catalogs with activity support
const schema = await App.fetch('/db/systempush_settings/sheme.json');
const catalogs = schema.data.menu.arr;
// { "tt_tasks": "Tasks", "b2b_orders": "Orders", ... }
```

---

## Filter Values

### restrict_entries

| Value | Description |
|-------|-------------|
| *(empty)* | All catalog records the user has access to |
| `self` | Only records where `from_auth` = current user |
| `members` | Only records where user is in `members_id` (watcher) |

Can be combined: `self,members` — own records + where user is a watcher.

### actions

| Value | Description |
|-------|-------------|
| *(empty)* | All actions |
| `add` | Add only |
| `edit` | Edit only |
| `delete` | Delete only |

Can be combined: `add,edit` — add and edit, but not delete.

---

## Use Cases for Miniapps

### Auto-subscribe on app install

An app working with a specific catalog can automatically
subscribe the user to notifications on install:

```js
async function setupNotifications() {
    // Check if already subscribed
    const resp = await App.fetch('/db/systempush_settings.json?form[menu]=b2b_orders');
    if (resp.data.length === 0) {
        // Subscribe to new orders
        await App.fetch('/db/systempush_settings/add?edit&ajax=1', {
            method: 'POST',
            body: {
                'form[menu]': 'b2b_orders',
                'form[actions][]': ['add'],
                submit: 1
            }
        });
    }
}
```

### Subscription management panel

An app can show the user a UI for managing subscriptions,
grouped by modules or app business logic.

### Check subscription before action

```js
// Check if user is subscribed to a catalog
async function isSubscribed(catalog) {
    const resp = await App.fetch(`/db/systempush_settings.json?form[menu]=${catalog}`);
    return resp.data.length > 0;
}
```

---

## Important Notes

- Subscriptions work in conjunction with the activity system — notifications are only sent for catalogs with activity enabled
- Push notifications are delivered in real time via WebSocket (the `systempush` module)
- Notifications appear in the bell icon in the platform header and as browser push notifications
- By default no notifications are sent — explicit subscription is required

---

*Catalog: `/db/systempush_settings` | Schema: `systempush_settings.sheme.inc.php`*

**← [Home](index.md)**
