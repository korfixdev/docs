# Favorites Menu (favorites_menu)

> **See also:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [systempush-settings.md](systempush-settings.md)
> **← [Home](index.md)**

The `favorites_menu` catalog — personal navigation settings for a user:
bookmarked sidebar menu items, the start page, and display mode.

---

## Record Structure

One record per user (alias = user ID).

| Field | Type | Description |
|-------|------|-------------|
| `alias` | hidden | User ID (= `from_auth`) |
| `name` | hidden | User name |
| `from_auth` | hidden | Record owner |
| `from_group` | hidden | Owner's group |
| `first_page` | select_from_table | Start page (from menu tree `rutree`) |
| `menu` | multiselect_from_table | IDs of menu items added to favorites |
| `hide_menu` | checkbox | Collapse other menu items (show favorites only) |

Access: `self` — each user sees and edits only their own record.

---

## Reading Favorites

```js
// Get current user's settings
const resp = await App.fetch('/db/favorites_menu.json');
const favorites = resp.data[0]; // one record per user

console.log(favorites.menu);       // "12,45,78" — menu item IDs, comma-separated
console.log(favorites.first_page); // ID of the start page
console.log(favorites.hide_menu);  // "1" or "0"
```

---

## Adding an Item to Favorites

To add an item to the user's favorites:

1. Read the current `menu` list
2. Add the desired ID
3. Save it back

```js
async function addToFavorites(menuItemId) {
    // 1. Read current settings
    const resp = await App.fetch('/db/favorites_menu.json');
    const record = resp.data[0];

    if (!record) {
        console.error('No favorites_menu record for user');
        return;
    }

    // 2. Check if already added
    const currentIds = (record.menu || '').split(',').filter(Boolean);
    if (currentIds.includes(String(menuItemId))) {
        return; // already in favorites
    }

    // 3. Add and save
    currentIds.push(String(menuItemId));

    await App.fetch(`/db/favorites_menu/${record.alias}?edit&ajax=1`, {
        method: 'POST',
        body: {
            'form[menu]': currentIds,  // array of IDs
            submit: 1
        }
    });
}
```

### Removing from Favorites

```js
async function removeFromFavorites(menuItemId) {
    const resp = await App.fetch('/db/favorites_menu.json');
    const record = resp.data[0];
    if (!record) return;

    const currentIds = (record.menu || '').split(',').filter(Boolean);
    const newIds = currentIds.filter(id => id !== String(menuItemId));

    await App.fetch(`/db/favorites_menu/${record.alias}?edit&ajax=1`, {
        method: 'POST',
        body: {
            'form[menu]': newIds,
            submit: 1
        }
    });
}
```

---

## Getting the Menu Tree

To find out which menu items are available to add to favorites:

```js
// Load schema — it contains the full menu tree in `arr`
const schema = await App.fetch('/db/favorites_menu/sheme.json');
const menuTree = schema.data.menu.arr;
// { "12": "Finance > Transactions", "45": "Tasks > All Tasks", ... }
```

---

## Setting the Start Page

```js
async function setStartPage(pageId) {
    const resp = await App.fetch('/db/favorites_menu.json');
    const record = resp.data[0];
    if (!record) return;

    await App.fetch(`/db/favorites_menu/${record.alias}?edit&ajax=1`, {
        method: 'POST',
        body: {
            'form[first_page]': pageId,
            submit: 1
        }
    });
}
```

---

## Use Cases for Miniapps

### Auto-add to favorites on install

On install, an app can automatically add relevant menu items
to favorites so the user sees them in navigation immediately.

### Smart favorites based on activity

An app can analyze `activity` and automatically suggest
adding frequently visited sections to favorites:

```js
// Get user's recent visits
const activity = await App.fetchAll('/db/activity.json?form[action]=viewed&limit=100');

// Count catalog visit frequency
const counts = {};
activity.data.forEach(item => {
    counts[item.catalog] = (counts[item.catalog] || 0) + 1;
});

// Top 5 most visited
const top5 = Object.entries(counts)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 5);
```

### Navigation personalization by role

An app can configure favorites based on the user's role,
department, or active company modules.

---

## Important Notes

- The record is created automatically on first user login (defaults depend on `type_user`)
- The `menu` field is a comma-separated string of IDs (multiselect_from_table)
- When writing via API you can pass `form[menu]` as an array — the platform converts it
- Changes to favorites are visible after a page reload (sidebar menu is rendered server-side)
- Drag-and-drop of menu items to favorites works in the UI — your app gets the already-updated data on read

---

*Catalog: `/db/favorites_menu` | Schema: `favorites_menu.sheme.inc.php`*

**← [Home](index.md)**
