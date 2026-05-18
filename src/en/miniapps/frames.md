# Frames: Standards and Conventions

> **See also:** [config-json.md](config-json.md) · [self-provisioning.md](self-provisioning.md) · [dashboards.md](dashboards.md) · [js-api.md](js-api.md)
> **← [Home](index.md)**

Every key in `config.json → urls` is a frame (iframe). The frame name determines its architectural role. Follow these conventions — they are checked by the validator.

---

## Standard frame types

### `install` — setup screen

**When to use:** the app requires self-provisioning (creates custom catalogs/fields).

**Purpose:** first-run setup wizard, runs once.

**Required:**

- Checks existence of all needed catalogs via `custom_dbtables` (not via `/db/{catalog}.json`)
- **Always checks the REAL API response status** — see "Checking responses" section below
- Saves a log of each step to `App.storage` under the key `install.log`
- On re-open (catalog already exists): shows the saved log + "Reinstall" / "Close" buttons
- If the app has a `widget` frame — after successful install, **auto-adds the widget to the first dashboard**
- After install completes — navigates to the `main` frame:
  ```js
  App.navigate(`/db/installed_apps/${params.data.token}?frame=main`);
  ```

**In config.json:**
```json
{
    "urls": { "install": "install.html" },
    "urlsConf": { "install": { "method": "get" } }
}
```

---

### `main` — primary interface

**When to use:** the app has a CRM menu item or opens via an "Open" button.

**Purpose:** primary user-facing screen.

**Required:**

- On load, checks the app is ready (catalogs exist, data is accessible)
- If the app is not installed — **redirects to the `install` frame**:
  ```js
  const App = new VMCRMUserApp();
  const params = await App.getRequestParams();

  const isInstalled = await checkCatalogExists('custom_my_catalog');
  if (!isInstalled) {
      App.navigate(`/db/installed_apps/${params.data.token}?frame=install`);
      return;
  }
  // App is ready — render main UI
  ```
- `urls.main` **is required** if the app adds a menu item or should open from the marketplace

**In config.json:**
```json
{
    "urls": { "main": "main.html", "install": "install.html" },
    "urlsConf": { "main": { "method": "get" } },
    "menu": { "tt_tasks": { "frame": "main", "name": "My Module" } }
}
```

---

### `footer` — embedded catalog widget

**When to use:** the app embeds inside catalog pages (below the table, above/below a card).

**Purpose:** additional UI block inside existing CRM pages — not a standalone page.

**Required:**

- Compact layout: no own header/navbar
- `App.setFrameSize()` required after every render/resize
- Receives current catalog context via `params.data.catalog`

**In config.json:**
```json
{
    "catalogs": {
        "ag_clients": {
            "catalog.items.footer": { "name": "My Widget", "frame": "footer", "width": 6 }
        }
    }
}
```

Empty catalog key `""` applies the widget to all platform catalogs.
One html file can serve both `footer` and `widget` — they're visually similar.

---

### `widget` — compact dashboard widget

**When to use:** the app provides a summary for the workspace.

**Purpose:** embedding in the dashboard via the `app-frame` type.

**Required:**

- Compact size: width ≤ 6 columns, minimal height
- Fast loading: just the essentials, no heavy libraries
- `App.setFrameSize()` required
- Works with `data.catalog === 'dashboard_widgets'` — standard dashboard context
- If the app has self-provisioning (`urls.install`): **the install frame must auto-install the widget on the first dashboard**

**In config.json:**
```json
{
    "urls": { "widget": "widget.html" },
    "urlsConf": { "widget": { "method": "get" } },
    "permissions": {
        "catalogs": { "dashboard_widgets": ["read", "write"] }
    }
}
```

---

## Pattern: auto-install widget on setup

Called at the end of `runInstall()` — after creating catalogs and access rights.
A widget install error **does not abort** the main install — log it and continue.

```js
async function installWidgetOnDashboard(appToken, widgetName) {
    try {
        const resp = await App.fetch('/api/db/dashboards?limit=999');
        const boards = (resp.data || []).sort((a, b) => (a.prior || 0) - (b.prior || 0));
        if (!boards.length) {
            logLine('⚠ No dashboards found — widget can be added manually');
            return;
        }
        const board = boards[0];
        const result = await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
            method: 'POST',
            body: {
                'form[alias]': uid(),
                'form[name]': widgetName,
                'form[type]': 'app-frame',
                'form[width]': 6,
                'form[board_id]': board.id,
                'form[options]': JSON.stringify({ app_frame: appToken + ':widget' }),
                'form[from_auth]': currentUserId,
                'form[from_group]': currentUserId,
                submit: 1
            }
        });
        if (!result || result.status === 'error' || result.status === 'no') {
            throw new Error(result?.message || JSON.stringify(result));
        }
        logLine('✓ Widget added to dashboard "' + board.name + '"');
    } catch (e) {
        logLine('⚠ Could not add widget automatically: ' + e.message);
    }
}

// How to get appToken and call:
const params = await App.getRequestParams();
const appToken = params.data.token;   // installed_apps.alias of current install
await installWidgetOnDashboard(appToken, 'Widget Name');
```

> Note: `/api/db/dashboards` (not `/db/`) — returns the full list without server-side filters. More: [dashboards.md](dashboards.md).

---

## Checking API responses: always required

`App.fetch()` **does not throw exceptions** on API errors — the server returns HTTP 200 with `status: 'error'` or `status: 'no'` in the body. This is a common cause of "installation completed but nothing was created".

**Always check the real status:**

```js
const resp = await App.fetch('/db/custom_dbtables/add?edit&ajax=1', { method: 'POST', body });

// WRONG — silently swallows the error:
// Install will complete "successfully" but the table wasn't created
logLine('✓ Table created');  // ← false success

// CORRECT:
if (!resp || resp.status === 'error' || resp.status === 'no') {
    throw new Error(`Table creation error: ${resp?.message || JSON.stringify(resp)}`);
}
logLine(`✓ Table created (alias: ${resp.alias})`);
```

Applies to all mutating requests: creating catalogs, fields, widgets, records.
For reads (GET requests) — check via `resp.data` / `Array.isArray(resp.data)`.

---

## Typical file structure

```
my-app/
├── config.json      # required
├── main.html        # if there's a menu item / "Open" button
├── install.html     # if there's self-provisioning (custom catalogs)
├── widget.html      # if embedding on dashboard
├── footer.html      # if embedding inside catalogs (optionally separate)
├── logo.svg
└── README.md        # required before deploy
```

One file can cover multiple roles (e.g. one `widget.html` as both `footer` and `widget`) — acceptable if the UI is the same. Naming conventions aid navigation.

---

## Validator rules

### Critical (FAIL)

- `main.html` present and `urls.install` exists → `main.html` must have `checkCatalogExists` or `App.navigate(...)` with `frame=install`
- `install.html` has mutating `App.fetch` calls without `resp.status` check

### Must (WARN)

- `urls.widget` declared + `urls.install` exists → `install.html` must have `installWidgetOnDashboard` or equivalent
- `urls.widget` declared → `permissions.catalogs` must contain `dashboard_widgets: ["read", "write"]`

---

**Next:** [self-provisioning.md](self-provisioning.md) · **← [Home](index.md)**
