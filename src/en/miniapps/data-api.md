# Working with Catalog Data

> **See also:** [js-api.md](js-api.md) · [storage-and-hooks.md](storage-and-hooks.md) · [self-provisioning.md](self-provisioning.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [Home](index.md)**

CRUD operations on catalogs, request formats, filtering, and pagination.

---

## ⚠️ Key rule: which endpoint to use from where

Korfix has **two different endpoints** for data access — they're commonly confused. The rule is simple:

| Where the request comes from | Endpoint | Auth | Field format |
|---|---|---|---|
| **Inside the miniapp iframe** (`App.fetch`) | `/db/{catalog}.json` | browser session (cookie) | `form[name]=value` |
| **Outside**: curl, tests, scripts, server integrations, webhooks, n8n, Bitrix24 | `/api/db/{catalog}` | `Authorization: Bearer <token>` | `name=value` (flat) |

**Don't mix them up:** `/db/` from outside the iframe returns a 302 redirect to login (no session). A common trap: an agent sees `/db/...` in app code, copies it to curl, gets a redirect, starts "fixing auth" — don't. Just replace `/db/` with `/api/db/` and add Bearer.

### Verifying catalog access before development

Always start with `curl` to `/api/db/` with a token — to confirm access:

```bash
# Check that the token sees the catalog
curl -sI "https://panel.korfix.ru/api/db/{catalog}?limit=1" \
  -H "Authorization: Bearer {TOKEN}"
# HTTP/2 200 — ok, access granted
# HTTP/2 403 — token lacks db_{catalog}_get class (add it in /db/api)
# HTTP/2 404 — catalog doesn't exist (check name, especially the custom_ prefix)

# Get all catalogs accessible to the token
curl -s "https://panel.korfix.ru/api/db/getcatalogs" \
  -H "Authorization: Bearer {TOKEN}"
```

Never do `curl https://panel.korfix.ru/db/{catalog}.json` — that's iframe-only; curl will get a 302 to login.

---

## Two ways to work with the API

### 1. Via session (App.fetch — no token)

`App.fetch()` proxies the request via `postMessage` to the parent window.
The parent window performs `fetch(url)` **with the authenticated user's cookies**.
No token needed — auth via session.

```js
// Load a catalog — works immediately, no token
const resp = await App.fetch('/db/projects.json');

// All pages
const all = await App.fetchAll('/db/installed_apps.json');
```

**Available**: all catalogs the current user has access to.
**Limitation**: works only inside a marketplace app's iframe.

### 2. Via Bearer token (REST API)

For server integrations, webhooks, and external apps.
Token from `/db/api` — determines accessible catalogs and methods.

```js
// Token auth is for EXTERNAL callers (curl / server / n8n) — see the Bearer/curl examples below.
// Never hard-code a platform token inside a shipped miniapp (marketplace-review failure).

// /api/user/tariff — current user's billing info (from the app this works via session, no token)
const billing = await App.fetch('/api/user/tariff');
// data: { tarif, tarif_name, balance, discount, discount_date, payment_date, price, ... }
```

**Setup**: `/db/api` → create a token → specify allowed API classes.

> ℹ️ **Inside a miniapp, prefer `/db/...` (with `form[]`) and pass no token.** `App.fetch` proxies
> through the logged-in parent window, so requests authenticate via the user **session** — you
> never need a token, and you must never hard-code a platform token in a shipped miniapp
> (marketplace-review failure). The Bearer-token path (`/api/db/...` + `Authorization: Bearer`)
> is for **external** callers — curl / CI / server / n8n.

### When to use which

| Scenario | Method |
|----------|--------|
| Widget loads data for display | `App.fetch('/db/catalog.json')` — session |
| Widget creates/edits an item | `App.fetch('/db/catalog/add?edit&ajax=1', {method:'POST'})` — session |
| Testing API with a specific token | `/api/db/catalog?token=XXX` — token |
| Server webhook (afterSave) | `Authorization: Bearer XXX` — token |
| External service (n8n, Bitrix24) | `Authorization: Bearer XXX` — token |

### Field format: form[] vs flat

| Endpoint | Field format | Wrapper |
|----------|-------------|---------|
| `/db/catalog/...` | `form[name]=value` | needs `form[]` |
| `/api/db/catalog` | `name=value` | **no** `form[]` |

Simple rule — based on the endpoint:
- **`/db/...`** — always `form[]`
- **`/api/db/...`** — always flat fields

This rule applies **regardless of how you call it** — App.fetch(), curl, external service.

```js
// /db/ — with form[]
App.fetch('/db/projects/add?edit&ajax=1', {
    method: 'POST',
    body: { 'form[name]': 'Project', submit: 1 }
});

// /api/db/ — without form[]
App.fetch('/api/db/custom_dbtables', {
    method: 'POST',
    body: { name: 'My Table', dbname: 'mytable', submit: 1 }
});
```

```bash
# curl to /api/ — also without form[]
curl -X POST "https://panel.korfix.ru/api/db/projects" \
  -H "Authorization: Bearer TOKEN" \
  -F "name=Project" -F "submit=1"
```

### ⚠️ Always check `status` on write operations

`App.fetch()` **does not throw an exception** on logical errors — it returns an object with `{status: 'error'}` or `{status: 'no'}` + HTTP 200. If you don't check it, the error is silently swallowed and the app behaves as if everything is fine (until it fails somewhere unexpected).

**Typical mistake:**

```js
// ❌ No check — error is swallowed
await App.fetch('/db/custom_dbtables/add?edit&ajax=1', {
    method: 'POST',
    body: { 'form[dbname]': 'my_catalog', submit: 1 }
});
// Record wasn't created (e.g. scheme wasn't passed), but code continues
```

**Correct:**

```js
const resp = await App.fetch('/db/custom_dbtables/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[dbname]': 'my_catalog',
        'form[scheme]': 'coredb_def_catalog',
        submit: 1
    }
});
if (!resp || resp.status === 'error' || resp.status === 'no') {
    throw new Error(resp?.message || JSON.stringify(resp));
}
```

**Status value meanings:**

| Value | Meaning |
|---|---|
| `'ok'` / `'success'` | Operation succeeded. Has data. |
| `'error'` | Explicit error (validation failed, required field empty, access denied). Message in `message` or `errors`. |
| `'no'` | Operation not performed for a legitimate reason (e.g., record not found during edit). |
| absent/`null` | Network/server error. Treat as error. |

**Helper pattern** to avoid copy-pasting the check everywhere:

```js
async function appFetch(url, options) {
    const resp = await App.fetch(url, options);
    if (!resp || resp.status === 'error' || resp.status === 'no') {
        throw new Error(`${url}: ${resp?.message || JSON.stringify(resp)}`);
    }
    return resp;
}

// Usage — no need to write checks in code:
const resp = await appFetch('/db/custom_dbtables/add?edit&ajax=1', {
    method: 'POST',
    body: { ... }
});
// If we got here — all ok, resp.data contains the result
```

---

### HTTP status codes for `/api/db/` write endpoints

`POST /api/db/{catalog}` and `POST /api/db/{catalog}/{id}` return correct HTTP codes:

| Scenario | HTTP code | `ok` |
|----------|-----------|------|
| Record created successfully | **201 Created** | `true` |
| Record updated successfully | **200 OK** | `true` |
| Validation error (missing required field, etc.) | **422 Unprocessable** | `false` |
| Record not found (update by non-existent id) | **404 Not Found** | `false` |
| Server error | **500** | `false` |

All responses include `ok: boolean` in the body:

```json
{ "ok": true, "status": "success", "id": "123", "alias": "abc" }
{ "ok": false, "status": "error", "message": "field_empty: name" }
```

> **`/db/` session endpoints are unchanged** — they still return HTTP 200 with `status: 'error'` in the body for backward compatibility with the UI layer.

For new miniapp code, use `App.fetchV2()` — it reads the `ok` field and normalizes the response shape for you.

---

## Working with catalog data

### Reading a list

```js
const resp = await App.fetch('/db/projects.json');
// resp.data — array of items
// resp.total — total count

// With a filter
const resp = await App.fetch('/db/projects.json?form[status]=active');

// All pages at once
const all = await App.fetchAll('/db/projects.json');
```

> **`.json` vs `/api/db/` — filters and session cache.**
>
> #### Session filter cache (`/db/`)
>
> Catalogs in AJAX mode remember the user's filters in the session (key — URL path of the request).
>
> **When cache applies to a miniapp request:**
> Only when the request has `ajax=1` AND the HTTP Referer path matches the path where the user saved filters. A standard `App.fetch('/db/tt_tasks.json')` **without** `ajax=1` uses path `/db/tt_tasks.json`, which doesn't match UI path `/db/tt_tasks` — cache not applied.
>
> If the request explicitly includes `ajax=1`, cache may apply. Then:
> - Any passed `form[]` (even empty) makes `$search` non-empty → cache not loaded.
> - **`not_cache=1`** only prevents **writing** to cache, not reading.
> - **`free_cache=1`** — ignore session-cached filters and sorting (don't read from cache). Use for programmatic requests so the user's previous UI search doesn't affect results.
>
> | Parameter | Read from cache | Write to cache |
> |-----------|:-:|:-:|
> | *(no params)* | yes | yes |
> | `not_cache=1` | yes | **no** |
> | `free_cache=1` | **no** | yes |
> | `free_cache=1&not_cache=1` | **no** | **no** |
>
> ```js
> // Safe: without ajax=1, path /db/tt_tasks.json ≠ /db/tt_tasks (UI cache untouched)
> App.fetch('/db/tt_tasks.json?limit=20')
>
> // Full cache bypass — recommended for programmatic miniapp requests:
> App.fetch('/db/tt_tasks.json?free_cache=1&not_cache=1')
>
> // With ajax=1: cache may apply if Referer matches the filter path
> // To bypass — pass explicit form[] or free_cache=1:
> App.fetch('/db/tt_tasks.json?ajax=1&not_cache=1&form[status]=open')
> // not_cache=1 — doesn't overwrite the user's cache with this value
> ```
>
> #### `/api/db/` — no session cache
>
> The `/api/db/catalog` endpoint doesn't use the session and is suitable for full queries without user filters. `/db/` returns more fields — in particular, `from_group`/`from_auth` are returned in `/db/` but not in `/api/db/` by default (hidden schema fields).
>
> To get hidden fields via `/api/db/`, request them explicitly with `select=`:
>
> ```js
> // from_group is absent — hidden field in schema, not selected by default
> App.fetch('/api/db/tt_tasks?limit=100')
>
> // Explicitly include hidden fields via select=:
> App.fetch('/api/db/tt_tasks?limit=100&select=name,status,from_group,from_auth')
>
> // Full list without session filters (but without from_group):
> const resp = await App.fetch('/api/db/dashboards?limit=999');
> ```

**Important: response normalization.** `resp.data` is not always an array — it can be
an object (single record), `null`, or nested `resp.data.data`
(when proxied through `App.fetch`). **Always** normalize to array:

```js
// Safe array extraction
function asArray(resp) {
    if (Array.isArray(resp?.data)) return resp.data;
    if (Array.isArray(resp?.data?.data)) return resp.data.data;
    return [];
}

const projects = asArray(await App.fetchAll('/db/tt_projects.json'));
const tasks    = asArray(await App.fetchAll('/db/tt_tasks.json'));
// Now .sort(), .filter(), .map() are safe
```

### Reading a single item

```js
const resp = await App.fetch('/db/projects/ALIAS.json');
// or via API:
const resp = await App.fetch('/api/db/projects/ALIAS');
```

### Creating an item

```js
await App.fetch('/db/projects/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'New Project',
        'form[status]': 'active',
        submit: 1
    }
});
```

> **Important: `alias` — unique record key.**
> Every platform table has two identifiers: `id` (numeric timestamp) and `alias` (string, unique in the table).
> Accessing items via URL and links — **always by `alias`**: `/db/projects/ALIAS.json`, `/db/projects/ALIAS?edit`.
> When creating via API, `form[alias]` has a schema default of `uniqid()`, but the default is computed **once when the schema loads**.
> When creating multiple records in a loop, **explicitly generate a unique alias** for each:
>
> ```js
> // Safe alias generation for bulk creation
> const alias = Date.now().toString(36) + Math.random().toString(36).substr(2, 8);
> await App.fetch('/db/projects/add?edit&ajax=1', {
>     method: 'POST',
>     body: {
>         'form[alias]': alias,
>         'form[name]': 'New Project',
>         submit: 1
>     }
> });
> ```

### System record fields

Every catalog record contains a set of system fields. These fields manage identification, ownership, and visibility. When creating via API, they must be handled explicitly.

#### `id` and `alias` — two identifiers

| Field | Type | Purpose |
|-------|------|---------|
| `id` | int / bigint | Numeric identifier (usually creation timestamp). Used for table relationships (FK) |
| `alias` | varchar, unique | String unique key. Used in all URLs: `/db/catalog/ALIAS`, `/db/catalog/ALIAS.json`, `/db/catalog/ALIAS?edit` |

- Item URL — **always alias**, not id: `/db/projects/6989d171cce41`
- Table relationships (FK) — usually **id**: `board_id` in `dashboard_widgets` references `dashboards.id`
- On create: `alias` defaults to `uniqid()`, but **the default is computed once when the schema loads**. For bulk creation (loops) — generate explicitly
- `id` is usually filled automatically (timestamp), no need to pass it

#### `from_auth` and `from_group` — ownership and visibility

Most catalogs have `from_auth` and `from_group` fields that determine who owns and who can see a record.

| Field | Value | Meaning |
|-------|-------|---------|
| `from_auth` | `0` | Record visible to **all users in the group** (group-public) |
| `from_auth` | `USER_ID` | Record visible **only to this user** (personal) |
| `from_group` | `0` | Record accessible to **all groups** (global, undeletable) |
| `from_group` | `GROUP_ID` | Record belongs to a specific group |

**Rules when creating records via API:**

1. **For non-admins** the schema automatically sets `from_auth = SESSION_USER_ID` (hidden field with default). Explicit passing not needed.

2. **For admins** `from_auth` is a `select` with options `{0: 'All', USER_ID: 'Personal'}`. If you don't pass `form[from_auth]`, the record becomes public (`from_auth = 0`). To create a personal record — pass explicitly.

3. **Getting the current user ID** — from the catalog schema:

```js
// Load schema — from_auth.arr contains {0: 'All', USER_ID: 'Personal'}
const schema = await App.fetch('/db/dashboard_widgets/sheme.json');
const fromAuthArr = schema?.data?.from_auth?.arr || {};
const currentUserId = Object.keys(fromAuthArr).find(k => k !== '0') || 0;
```

4. **Example: bulk creation with correct ownership:**

```js
// Determine current user ID
const schema = await App.fetch('/db/my_catalog/sheme.json');
const arr = schema?.data?.from_auth?.arr || {};
const userId = Object.keys(arr).find(k => k !== '0') || 0;

for (const item of items) {
    const alias = Date.now().toString(36) + Math.random().toString(36).substr(2, 8);
    await App.fetch('/db/my_catalog/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[alias]': alias,
            'form[name]': item.name,
            'form[from_auth]': userId,
            'form[from_group]': userId,
            submit: 1
        }
    });
}
```

> **Summary:** when creating records, always pass `form[alias]` (unique), and if personal visibility is needed — `form[from_auth]` and `form[from_group]`.

### Editing

```js
await App.fetch(`/db/projects/${alias}?edit&ajax=1`, {
    method: 'POST',
    body: {
        'form[name]': 'Updated name',
        'form[id]': id,
        'form[alias]': alias,
        submit: 1
    }
});
```

### Deleting

Deletion is **not HTTP DELETE**. The record is marked `hidden=1` and moved to trash.
From trash the user can permanently delete.

```js
// Via /db/ — soft delete (to trash)
await App.fetch(`/db/projects/${alias}?udel&ajax=1`, { method: 'POST' });

// Via /api/db/ — also soft delete (hidden=1)
await App.fetch(`/api/db/projects/${id}`, {
    method: 'POST',
    body: { hidden: 1, submit: 1 }
});
```

**Important**: `hidden` must be in the catalog schema. For custom catalogs (`custom_*`)
the `hidden` field is present automatically.

### Custom catalogs (`custom_*`) — CRUD specifics

For custom catalogs (created via self-provisioning), use
**`/api/db/`** instead of `/db/` for write operations:

```js
// CREATE — /api/db/ saves alias correctly
await App.fetch(`/api/db/custom_my_catalog`, {
    method: 'POST',
    body: {
        alias: uid(),                    // no form[]
        custom_name: 'Name',
        from_auth: currentUserId,
        from_group: currentUserId,
        submit: 1
    }
});

// EDIT — /api/db/ by id
await App.fetch(`/api/db/custom_my_catalog/${id}`, {
    method: 'POST',
    body: {
        custom_name: 'New name',
        submit: 1
    }
});

// DELETE — hidden=1
await App.fetch(`/api/db/custom_my_catalog/${id}`, {
    method: 'POST',
    body: { hidden: 1, submit: 1 }
});
```

> **Why `/api/db/` and not `/db/`?** The `/db/.../add?edit&ajax=1` endpoint
> for custom catalogs may **ignore `form[alias]`** — the record is created without an alias.
> Without an alias, editing via `/db/{catalog}/{alias}?edit` is impossible.
> Via `/api/db/` the alias is saved correctly.
>
> **Addressing in `/api/db/`**: POST updates work by `/{id}` (numeric identifier).
> By alias for custom catalogs, POST may return `item not found`.

### `from_auth` and `from_group` — required

When creating records **always** pass `from_auth` and `from_group`:

```js
body: {
    from_auth: currentUserId,   // record owner
    from_group: currentUserId,  // group
    submit: 1
}
```

Records with empty `from_auth`/`from_group` belong to the super-admin, are visible to all accounts
and are **unmanageable** — they can't be edited or deleted through the platform UI.

### Getting the catalog schema

```js
const schema = await App.fetch('/db/projects/sheme.json');
// schema.data — object with field descriptions by key
```

For `select` type fields — options in `arr`:

```js
const statusField = schema.data.status;
// statusField.type = 'select'
// statusField.arr = {0: 'New', 10: 'In progress', 40: 'Done'}
```

For `select_from_table` type fields — options in `arr` (up to 200 records), plus metadata:

```js
const personField = schema.data.person_id;
// personField.type = 'select_from_table'
// personField.catalog = 'auth_pers'              — related catalog name
// personField.total = 18                         — total record count
// personField.arr = {123: 'Ivanov I.', 456: 'Petrov P.'}  — first 200 options
// personField.ex_table_field = 'author_comment'  — display field
// personField.id_ex_table = 'author_id'          — key field
```

For small references (total <= 200) — `arr` contains all options:

```js
const schema = await App.fetch('/db/tt_tasks/sheme.json');
const options = schema.data.person_id.arr;
const select = document.getElementById('personSelect');
for (const [id, name] of Object.entries(options)) {
    select.add(new Option(name, id));
}
```

For large references (total > 200) — paginate on a specific field:

```js
const field = schema.data.client_id;
if (field.total > Object.keys(field.arr).length) {
    // There are more pages — load the second
    const page2 = await App.fetch('/db/tt_tasks/sheme.json?field=client_id&p=2');
    const moreOptions = page2.data.client_id.arr;
    // ... add to select or implement autocomplete
}
```

Pagination parameters:
- `field=person_id` — load arr only for the specified field (saves bandwidth)
- `p=2` — page number (200 records per page)

### User catalog settings

Column order, field visibility, catalog renaming — the user configures
via the gear icon in the header. The app can read and save these settings.

Detailed reference with examples and use-cases: [catalog-settings.md](catalog-settings.md)

### URL format

```
/db/{catalog}.json          -- item list (GET)
/db/{catalog}/{alias}       -- specific item
/db/{catalog}/{alias}.json  -- item as JSON
/db/{catalog}/add?edit      -- create page
/db/{catalog}/sheme.json    -- schema (fields, types, options)
/db/{catalog}/catalog/settings.json -- user column settings
/empty/db/{catalog}         -- without template (for modals)
/api/db/{catalog}           -- REST API (JSON)
```

### Filtering

```
/db/projects.json?form[status]=active&form[client_id]=123
```

### Pagination

```
/db/projects.json?p=2
```

### Query parameters (/api/db/)

Via `/api/db/{catalog}` additional output control parameters are available:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `filter[field]=value` | Filter by field | `filter[status]=active` |
| `order_by=field` | Sort by field | `order_by=name` |
| `order=ASC\|DESC` | Sort direction | `order=DESC` |
| `limit=N` | Record count (default 20) | `limit=50` |
| `offset=N` | Offset | `offset=20` |
| `select=f1,f2` | Select specific fields | `select=name,status` |
| `load_values=1` | Replace linked field IDs with display values | `load_values=1` |

**`load_values`** — for `select_from_table` fields returns the text value instead of numeric ID (employee name instead of ID). Convenient for display without extra requests:

```js
// Without load_values: person_id = "1715761701"
// With load_values:   person_id = "Alexey Grigoriev"
const resp = await App.fetch('/api/db/tt_tasks?load_values=1');
```

**`select`** — limits the field set in the response. Fields with `_id` suffix
and required fields are always included:

```js
const resp = await App.fetch('/api/db/tt_tasks?select=name,status&limit=10');
```

**`order_by` + `order`** — sorting works only on fields present in the schema:

```js
const resp = await App.fetch('/api/db/tt_tasks?order_by=name&order=ASC');
```

---

## Available catalogs

Full list with examples: [korfix-catalogs.md](korfix-catalogs.md)

Main groups: AG (finance), B2B (trade), MD (manufacturing),
TT (tasks), WH (warehouse), VRN (field work), CRM, system.

---

**Next:** [storage-and-hooks.md](storage-and-hooks.md) · **← [Home](index.md)**
