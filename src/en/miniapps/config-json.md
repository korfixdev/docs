# Entry Points and config.json

> **See also:** [getting-started.md](getting-started.md) · [js-api.md](js-api.md) · [dashboards.md](dashboards.md) · [deploy.md](deploy.md)
> **← [Home](index.md)**

All frame entry points into the CRM platform and dashboard widget configuration.

---

## Full `config.json` example

All supported fields with comments. In a real app use only what you need.

```json
{
    "name": "App Name",
    "version": "1.0.0",
    "package": "my-app",
    "description": "Short description in 1-2 sentences",
    "about": "## What it does\n...\n\n## Where it appears in CRM\n...\n\n## Features\n...\n\n## How to use\n...\n\n## Setup\n...",
    "tags": "finance, analytics, dashboard",
    "logo": "icon.svg",

    "urls": {
        "main": "index.html",
        "settings": "settings.html",
        "widget": "widget.html",
        "remote": "https://example.com/app/frame.php"
    },
    "urlsConf": {
        "main":     { "method": "get" },
        "settings": { "method": "get" },
        "widget":   { "method": "get" },
        "remote":   { "method": "post" }
    },

    "permissions": {
        "catalogs": {
            "tt_tasks":          ["read", "write"],
            "ag_clients":        ["read"],
            "dashboard_widgets": ["read", "write"],
            "custom_my_catalog": ["*"]
        },
        "storage":  true,
        "navigate": true,
        "modal":    true
    },

    "menu": {
        "tt_tasks":   { "frame": "main", "name": "My App" },
        "ag_clients": { "frame": "settings", "name": "Settings" }
    },

    "catalogs": {
        "ag_clients": {
            "tabs": [
                { "name": "Extra Tab", "frame": "main" }
            ],
            "itemsActions": [
                { "name": "Action on record", "frame": "main" }
            ],
            "menu": [
                { "name": "Catalog submenu item", "frame": "settings" }
            ],
            "catalog.items.footer": {
                "name": "Widget below the table",
                "frame": "widget",
                "width": 6
            },
            "catalog.item.view.header": {
                "name": "Widget above the card",
                "frame": "widget"
            },
            "catalog.item.view.footer": {
                "name": "Widget below the card",
                "frame": "widget"
            },
            "afterSave": "remote"
        },

        "": {
            "itemsActions": [
                { "name": "Universal action", "frame": "main" }
            ]
        }
    }
}
```

### Top-level fields reference

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | App name. Appears on the marketplace card |
| `version` | yes | Version (SemVer). Shown on the card |
| `description` | yes | Short description (1-2 sentences). Goes into `anons` |
| `about` | yes | Detailed markdown description. 5 sections: What it does / Where it appears in CRM / Features / How to use / Setup. All markdown in one string via `\n` |
| `package` | recommended | Package name (app folder name). Unique identifier |
| `tags` | recommended | Comma-separated tags. Used for filtering in the marketplace |
| `logo` | recommended | Logo filename from the zip. SVG preferred |
| `urls` | yes | Frame dictionary. Key → path. For local — relative, for remote — absolute. **`urls.main` is required if the app should open from the menu or the Install button** |
| `urlsConf` | optional | Parameters for `urls`: `method` (`get` for local, `post` by default) |
| `permissions` | recommended | Access rights declaration (see below) |
| `menu` | optional | Main menu items (see below) |
| `catalogs` | optional | Entry points per catalog (see below) |

### `menu` block

Adds an item to the platform's main sidebar.

- **Key** = catalog alias in the menu, **AFTER which** the item is inserted. For example `"tt_tasks"` — right after the Tasks section, `"installed_apps"` — after the Apps section.
- **Value**: `{ "frame": "name_from_urls", "name": "Label" }`.
- Multiple items — multiple keys.
- If the app is a dashboard widget or only embeds into catalogs — the `menu` block is not needed.
- **`urls.main` is required**: without it the platform doesn't know what to open on menu click. An app without `urls.main` doesn't appear in the marketplace UI as "available to install".

### `catalogs` block

Embedding configuration per catalog. Key — catalog alias or `""` (applies to all).

Entry point types — see table below.

---

## Entry points (where the frame appears)

The `"width"` parameter is supported for footer/header/widgets — width in Bootstrap columns (1-12).
For example `"width": 4` = 1/3 screen, `"width": 8` = 2/3. Grid layout (`<div class="row">`)
is enabled only when 2+ frames with width exist on the same page. A single frame is always full width.
With grid layout the frame header is hidden (compact mode).

| Key in config.json | Where it appears | How it opens |
|---------------------|-----------------|--------------|
| `menu.{catalog}` | Main menu, after the specified catalog | Full page |
| `catalogs.{cat}.itemsActions[]` | Item action menu in the list | Popup |
| `catalogs.{cat}.tabs[]` | Tabs on the item view page | Tab |
| `catalogs.{cat}.menu[]` | Third-level catalog menu | Full page |
| `catalogs.{cat}.catalog.items.footer` | Below the catalog list table | Embedded frame |
| `catalogs.{cat}.catalog.item.view.header` | Above the item card | Embedded frame |
| `catalogs.{cat}.catalog.item.view.footer` | Below the item card | Embedded frame |
| `catalogs.{cat}.afterSave` | Called when an item is saved (remote only) | Server webhook |

Empty catalog key `""` — for all catalogs.

---

## Permissions (sandbox)

The `permissions` section in config.json declares which catalogs and operations the app needs access to. The host-side JS checks every `App.fetch()` against the declared permissions.

```json
{
  "permissions": {
    "catalogs": {
      "dashboards": ["read"],
      "dashboard_widgets": ["read", "write"]
    },
    "storage": false,
    "navigate": true,
    "modal": false
  }
}
```

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `catalogs` | `{string: string[]}` | Catalog → operations (`read`, `write`, `delete`, `*`) | `{}` |
| `storage` | `bool` | Access to `App.storage` | `true` |
| `navigate` | `bool` | Access to `App.navigate()` | `true` |
| `modal` | `bool` | Access to `App.modal()` | `true` |

- No `permissions` section → full access (legacy, console.warn)
- `"permissions": {}` → nothing allowed (secure default)
- `"*"` as a catalog key — access to all catalogs
- Schema (`sheme.json`) is automatically allowed for catalogs listed in `catalogs`

When installed, the marketplace page shows the list of requested permissions.

---

## Category

In `config.json` set the `"category": <int>` field — a numeric id from the canonical table:

| id | category  | When to pick                                                              |
|----|-----------|---------------------------------------------------------------------------|
| 1  | AI-agents | AI assistants, chatbots, generative tools, agents                         |
| 2  | Business  | CRM extensions, reports, dashboards, B2B integrations                     |
| 3  | Games     | Games, entertainment miniapps                                             |
| 4  | Tools     | Utilities, converters, widgets, dev tools                                 |
| 5  | Other     | Everything else / no good fit                                             |

The platform writes `category` to the database automatically on the first app install (if the field is still empty). Later edits go through the `/db/marketplace` catalog UI and **are not overwritten** on subsequent deploys.

Example:

```json
{
  "name": "Coin Clicker",
  "category": 3,
  "package": "coin-clicker"
}
```

---

## Dashboard embedding

Apps can be embedded in the dashboard as **full widgets** with drag/drop,
configurable width, repositioning, editing, and deletion.

### How to add

1. Dashboard → Settings → Widgets → Add
2. Type = **"Marketplace App"**
3. In the options — a "App frame" dropdown with all frames from installed apps
4. Select a frame → the widget appears on the dashboard
5. You can change the width (1/3, 1/2, 2/3, 100%), reorder, delete

### What the app receives

A frame inside a dashboard widget receives standard parameters with `catalog = "dashboard_widgets"`.

```js
const App = new VMCRMUserApp();
App.getRequestParams().then(({data}) => {
    // data.catalog === 'dashboard_widgets'
    // The app can detect it's on the dashboard
});
```

---

**Next:** [js-api.md](js-api.md) · **← [Home](index.md)**
